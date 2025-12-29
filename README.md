# PostgreSQL DBM Sandbox - Explain Plan Error Reproduction

This sandbox environment reproduces various Datadog Database Monitoring (DBM) explain plan collection errors for PostgreSQL.

## Quick Start

### Step 1: Start minikube

```bash
minikube start
```

### Step 2: Create the manifests

Create the following directory structure:

```
sandbox-postgres-dbm/
├── postgres/
│   └── postgres-deployment.yaml
├── app/
│   └── demo-app.yaml
└── datadog/
    └── values.yaml
```

#### postgres/postgres-deployment.yaml

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: postgres-demo
  labels:
    tags.datadoghq.com/env: sandbox
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: postgres-demo
type: Opaque
stringData:
  POSTGRES_USER: "postgres"
  POSTGRES_PASSWORD: "datadog123"
  POSTGRES_DB: "demo_app"
  DD_POSTGRES_USER: "datadog"
  DD_POSTGRES_PASSWORD: "datadog_password"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
  namespace: postgres-demo
data:
  init.sql: |
    -- Create Datadog user for monitoring
    CREATE USER datadog WITH PASSWORD 'datadog_password';
    GRANT pg_monitor TO datadog;
    GRANT SELECT ON pg_stat_database TO datadog;
    
    -- Enable pg_stat_statements for query metrics
    CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
    
    -- Create datadog schema for explain plans
    CREATE SCHEMA IF NOT EXISTS datadog;
    GRANT USAGE ON SCHEMA datadog TO datadog;
    GRANT USAGE ON SCHEMA public TO datadog;
    GRANT pg_monitor TO datadog;
    
    -- Create explain_statement function for DBM explain plans (MANDATORY)
    -- See: https://docs.datadoghq.com/database_monitoring/setup_postgres/selfhosted/
    CREATE OR REPLACE FUNCTION datadog.explain_statement(
       l_query TEXT,
       OUT explain JSON
    )
    RETURNS SETOF JSON AS
    $$
    DECLARE
    curs REFCURSOR;
    plan JSON;

    BEGIN
       OPEN curs FOR EXECUTE pg_catalog.concat('EXPLAIN (FORMAT JSON) ', l_query);
       FETCH curs INTO plan;
       CLOSE curs;
       RETURN QUERY SELECT plan;
    END;
    $$
    LANGUAGE 'plpgsql'
    RETURNS NULL ON NULL INPUT
    SECURITY DEFINER;
    
    GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
    
    -- Create sample tables for the demo app
    CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE TABLE IF NOT EXISTS orders (
        id SERIAL PRIMARY KEY,
        user_id INT REFERENCES users(id),
        amount DECIMAL(10,2) NOT NULL,
        status VARCHAR(20) DEFAULT 'pending',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    -- Create a restricted schema for testing
    CREATE SCHEMA IF NOT EXISTS restricted_schema;
    CREATE TABLE restricted_schema.sensitive_data (
        id SERIAL PRIMARY KEY,
        data TEXT
    );
    -- Note: datadog user does NOT have access to restricted_schema
    
    -- Insert sample data
    INSERT INTO users (name, email) VALUES
        ('Alice', 'alice@example.com'),
        ('Bob', 'bob@example.com'),
        ('Charlie', 'charlie@example.com');
    
    INSERT INTO orders (user_id, amount, status) VALUES
        (1, 99.99, 'completed'),
        (1, 149.99, 'pending'),
        (2, 29.99, 'completed'),
        (3, 199.99, 'pending');
    
    -- Grant SELECT on public tables to datadog
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO datadog;
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: postgres-demo
  labels:
    app: postgres
    tags.datadoghq.com/env: sandbox
    tags.datadoghq.com/service: postgres-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tags.datadoghq.com/env: sandbox
        tags.datadoghq.com/service: postgres-demo
      annotations:
        ad.datadoghq.com/postgres.checks: |
          {
            "postgres": {
              "init_config": {},
              "instances": [{
                "host": "%%host%%",
                "port": 5432,
                "username": "datadog",
                "password": "datadog_password",
                "dbname": "demo_app",
                "dbm": true,
                "query_samples": {"enabled": true},
                "query_metrics": {"enabled": true},
                "collect_settings": {"enabled": true}
              }]
            }
          }
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: POSTGRES_DB
          args:
            - "-c"
            - "shared_preload_libraries=pg_stat_statements"
            - "-c"
            - "pg_stat_statements.track=all"
            - "-c"
            - "track_activity_query_size=4096"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
            - name: postgres-init
              mountPath: /docker-entrypoint-initdb.d
      volumes:
        - name: postgres-data
          emptyDir: {}
        - name: postgres-init
          configMap:
            name: postgres-init
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres-demo
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

#### app/demo-app.yaml

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: postgres-demo
  labels:
    app: demo-app
    tags.datadoghq.com/env: sandbox
    tags.datadoghq.com/service: demo-app
    tags.datadoghq.com/version: "1.0"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
        tags.datadoghq.com/env: sandbox
        tags.datadoghq.com/service: demo-app
        tags.datadoghq.com/version: "1.0"
    spec:
      containers:
        - name: demo-app
          image: python:3.11-slim
          command:
            - /bin/bash
            - -c
            - |
              pip install flask psycopg2-binary
              
              cat > /app.py << 'EOF'
              from flask import Flask, jsonify
              import psycopg2
              
              app = Flask(__name__)
              
              def get_connection(user='postgres', password='datadog123'):
                  return psycopg2.connect(
                      host='postgres',
                      port=5432,
                      dbname='demo_app',
                      user=user,
                      password=password
                  )
              
              @app.route('/health')
              def health():
                  return jsonify({"status": "ok"})
              
              @app.route('/query/users/<int:user_id>')
              def query_user(user_id):
                  conn = get_connection()
                  cur = conn.cursor()
                  cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
                  result = cur.fetchone()
                  cur.close()
                  conn.close()
                  return jsonify({"user": result})
              
              @app.route('/query/users')
              def query_all_users():
                  conn = get_connection()
                  cur = conn.cursor()
                  cur.execute("SELECT * FROM users")
                  results = cur.fetchall()
                  cur.close()
                  conn.close()
                  return jsonify({"users": results})
              
              @app.route('/query/users/<int:user_id>/lock')
              def query_user_lock(user_id):
                  conn = get_connection()
                  cur = conn.cursor()
                  cur.execute("SELECT * FROM users WHERE id = %s FOR UPDATE", (user_id,))
                  result = cur.fetchone()
                  conn.commit()
                  cur.close()
                  conn.close()
                  return jsonify({"user_locked": result})
              
              @app.route('/generate/<int:count>')
              def generate_queries(count):
                  conn = get_connection()
                  cur = conn.cursor()
                  for i in range(count):
                      cur.execute("SELECT * FROM users WHERE id = %s", ((i % 3) + 1,))
                      cur.fetchall()
                  cur.close()
                  conn.close()
                  return jsonify({"generated": count})
              
              if __name__ == '__main__':
                  app.run(host='0.0.0.0', port=8080)
              EOF
              
              python /app.py
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: postgres-demo
spec:
  selector:
    app: demo-app
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

#### datadog/values.yaml

```yaml
datadog:
  site: "datadoghq.com"
  apiKeyExistingSecret: "datadog-secret"
  clusterName: "postgres-dbm-sandbox"
  kubelet:
    tlsVerify: false

clusterAgent:
  enabled: true

agents:
  enabled: true
```

### Step 3: Deploy

```bash
# Deploy PostgreSQL
kubectl apply -f postgres/postgres-deployment.yaml

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n postgres-demo --timeout=120s

# Deploy Datadog Agent
kubectl create namespace datadog
kubectl create secret generic datadog-secret -n datadog --from-literal=api-key=YOUR_API_KEY
helm upgrade --install datadog-agent datadog/datadog -n datadog -f datadog/values.yaml
```

---

## Case 0: Working State (Baseline)

This is the expected working state where explain plans are collected successfully.

**Verification:**

```bash
# Test function as datadog user
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
SELECT datadog.explain_statement('SELECT * FROM users WHERE id = 1');
"
```

**Expected output:** JSON explain plan is returned successfully.

---

## Case 1: Missing Schema / Function (`invalid_schema`)

**UI Message:** "Missing function in the datadog schema"

**Description:**

This error appears when the `datadog.explain_statement` function doesn't exist or the datadog schema is missing.

**How to Reproduce:**

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
DROP FUNCTION IF EXISTS datadog.explain_statement(TEXT);
"
```

**Verify the issue:**

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
SELECT datadog.explain_statement('SELECT * FROM users WHERE id = 1');
"
```

**Expected error:**

```
ERROR:  function datadog.explain_statement(unknown) does not exist
```

**Fix:**

```sql
CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
$$
DECLARE
curs REFCURSOR;
plan JSON;
BEGIN
   OPEN curs FOR EXECUTE pg_catalog.concat('EXPLAIN (FORMAT JSON) ', l_query);
   FETCH curs INTO plan;
   CLOSE curs;
   RETURN QUERY SELECT plan;
END;
$$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
```

---

## Case 2: Table Not in Search Path (`undefined_table`)

**UI Message:** "The Agent can't find one or more tables"

**Description:**

This error occurs when a query references a table in a schema that's not in the search path, and the table name is not schema-qualified.

**How to Reproduce:**

```bash
# Create a table in a non-public schema
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
CREATE SCHEMA IF NOT EXISTS custom_schema;
CREATE TABLE IF NOT EXISTS custom_schema.custom_table (id INT PRIMARY KEY, data TEXT);
GRANT USAGE ON SCHEMA custom_schema TO datadog;
GRANT SELECT ON custom_schema.custom_table TO datadog;
"
```

**Verify the issue:**

```bash
# Query uses unqualified table name - will fail
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
SELECT datadog.explain_statement('SELECT * FROM custom_table WHERE id = 1');
"
```

**Expected error:**

```
ERROR:  relation "custom_table" does not exist
```

**Fix:**

Use schema-qualified table names in queries, or add the schema to search_path:

```sql
-- Option 1: Schema-qualified name works
SELECT datadog.explain_statement('SELECT * FROM custom_schema.custom_table WHERE id = 1');

-- Option 2: Add schema to search_path
ALTER ROLE datadog SET search_path TO public, custom_schema;
```

---

## Case 3: Query Truncation (`query_truncated`)

**UI Message:** "Explain plan unavailable due to query truncation"

**Description:**

This error occurs when the query text is truncated because it exceeds the `track_activity_query_size` PostgreSQL setting.

**How to Reproduce:**

```bash
# Set a very small track_activity_query_size
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
ALTER SYSTEM SET track_activity_query_size = 100;
SELECT pg_reload_conf();
"

# Restart PostgreSQL to apply
kubectl rollout restart deployment/postgres -n postgres-demo
kubectl wait --for=condition=ready pod -l app=postgres -n postgres-demo --timeout=120s
```

**Verify the issue:**

Long queries will be truncated and cannot be explained.

**Fix:**

```sql
ALTER SYSTEM SET track_activity_query_size = 4096;  -- or higher
SELECT pg_reload_conf();
```

---

## Case 4: Configuration Error (`database_error`)

**UI Message:** "Unable to collect explain plan (configuration error)"

**Description:**

This is a **catch-all error** that appears when the agent encounters an unexpected issue while collecting explain plans. Common causes include:

* Incorrect function definition
* Function throws an exception
* Unexpected database response
* Connection issues during explain collection

**How to Reproduce:**

One way to trigger this error is to create an `explain_statement` function that throws an exception:

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
DROP FUNCTION IF EXISTS datadog.explain_statement(TEXT);

CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
\$\$
BEGIN
   -- Throw an exception instead of returning a valid plan
   RAISE EXCEPTION 'Simulated configuration error';
END;
\$\$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
"
```

**Verify the issue:**

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
SELECT datadog.explain_statement('SELECT * FROM users WHERE id = 1');
"
```

**Expected error:**

```
ERROR:  Simulated configuration error
```

**Fix:**

Restore the correct `explain_statement` function:

```sql
CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
$$
DECLARE
curs REFCURSOR;
plan JSON;
BEGIN
   OPEN curs FOR EXECUTE pg_catalog.concat('EXPLAIN (FORMAT JSON) ', l_query);
   FETCH curs INTO plan;
   CLOSE curs;
   RETURN QUERY SELECT plan;
END;
$$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
```

---

## Case 5: SELECT ... FOR UPDATE Requires UPDATE Privilege

**UI Message:** "Datadog agent user lacks permission" (`failed_to_explain_with_prepared_statement`)

**Description:**

When a query uses `SELECT ... FOR UPDATE`, PostgreSQL requires the **UPDATE privilege** on the target table to acquire row-level locks. This applies even to `EXPLAIN` because PostgreSQL validates privileges during query planning.

If the Datadog user only has `SELECT` privilege (which is common), explain plans will fail for `SELECT ... FOR UPDATE` queries.

| Query Type | Required Privileges |
|------------|---------------------|
| `SELECT * FROM users WHERE id = 1` | SELECT |
| `SELECT * FROM users WHERE id = 1 FOR UPDATE` | SELECT + **UPDATE** |

**How to Reproduce:**

First, verify that datadog user only has SELECT privilege:

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
SELECT privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'datadog'
  AND table_schema = 'public'
  AND table_name = 'users'
ORDER BY privilege_type;
"
```

**Expected output:**

```
 privilege_type 
----------------
 SELECT
(1 row)
```

**Verify regular SELECT works:**

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
EXPLAIN (FORMAT JSON)
SELECT * FROM users WHERE id = 1;
"
```

**Expected output:** ✅ Returns JSON explain plan

**Verify SELECT ... FOR UPDATE fails:**

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
EXPLAIN (FORMAT JSON)
SELECT * FROM users WHERE id = 1 FOR UPDATE;
"
```

**Expected error:**

```
ERROR:  permission denied for table users
```

**Fix:**

Grant UPDATE privilege on the table to the Datadog user:

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
GRANT UPDATE ON TABLE users TO datadog;
"
```

**Verify Fix:**

```bash
# Check privileges now include UPDATE
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
SELECT privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'datadog'
  AND table_schema = 'public'
  AND table_name = 'users'
ORDER BY privilege_type;
"
```

**Expected output after fix:**

```
 privilege_type 
----------------
 SELECT
 UPDATE
(2 rows)
```

**Verify EXPLAIN now works:**

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
EXPLAIN (FORMAT JSON)
SELECT * FROM users WHERE id = 1 FOR UPDATE;
"
```

**Expected output:** ✅ Returns JSON explain plan with `LockRows` node

---

## Verification Commands

### Check Agent Status

```bash
# Equivalent kubectl command: kubectl exec -n datadog <pod> -- agent status
kubectl exec -n datadog $(kubectl get pods -n datadog -l app=datadog-agent -o jsonpath='{.items[0].metadata.name}') -- agent status | grep -A 15 "postgres ("
```

### Check Agent Configuration

```bash
# Equivalent kubectl command: kubectl exec -n datadog <pod> -- agent configcheck
kubectl exec -n datadog $(kubectl get pods -n datadog -l app=datadog-agent -o jsonpath='{.items[0].metadata.name}') -- agent configcheck | grep -A 20 "postgres:"
```

### Check Agent Logs for Explain Errors

```bash
kubectl logs -n datadog $(kubectl get pods -n datadog -l app=datadog-agent -o jsonpath='{.items[0].metadata.name}') -c agent | grep -i explain
```

### Verify Explain Function

```bash
# Test function as datadog user
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
SELECT datadog.explain_statement('SELECT * FROM users WHERE id = 1');
"
```

### Check Queries in pg_stat_statements

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
SELECT LEFT(query, 80) as query, calls 
FROM pg_stat_statements 
WHERE query NOT LIKE '%datadog-agent%'
ORDER BY calls DESC 
LIMIT 10;
"
```

### Check Datadog User Permissions

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
-- Check schema access
SELECT has_schema_privilege('datadog', 'restricted_schema', 'USAGE') as can_access_restricted;

-- Check function exists
SELECT routine_name FROM information_schema.routines WHERE routine_schema = 'datadog';

-- Check function permissions
SELECT grantee, privilege_type 
FROM information_schema.routine_privileges 
WHERE routine_name = 'explain_statement';
"
```

---

## Error Codes Reference

These error codes are defined in the Datadog Agent source code:

| Error Code | UI Message | Cause |
|------------|------------|-------|
| `database_error` | Unable to collect explain plan (configuration error) | Generic database-level error or misconfiguration |
| `datatype_mismatch` | - | Return type is not JSON (e.g., multiple queries explained) |
| `invalid_schema` | Missing function in the datadog schema | Schema or function missing |
| `invalid_result` | - | Value retrieved from EXPLAIN function is invalid |
| `no_plans_possible` | - | Statement cannot be explained (e.g., AUTOVACUUM) |
| `failed_function` | Unable to collect explain plan (function problem) | Function exists but has issues (permissions, definition) |
| `query_truncated` | Explain plan unavailable due to query truncation | Query exceeds track_activity_query_size |
| `connection_error` | - | Connection error, possibly misconfiguration |
| `parameterized_query` | - | Extended query protocol/prepared statements can't be explained directly |
| `undefined_table` | The Agent can't find one or more tables | Table not in search path |
| `undefined_function` | - | Cannot create prepared statement with given parameters (likely obfuscated) |
| `explained_with_prepared_statement` | ✅ (success) | Statement was explained using the prepared statement workaround |
| `failed_to_explain_with_prepared_statement` | Datadog agent user lacks permission | PREPARE fails for parameterized query due to permission issues |
| `no_plan_returned_with_prepared_statement` | - | Prepared statement workaround succeeded but no plan returned |
| `indeterminate_datatype` | Unable to collect explain plan due to indeterminate parameter type | PostgreSQL cannot determine parameter data type |
| `unknown_error` | - | Catch-all for unclassified errors |

---

## Cleanup

```bash
# Delete all resources
kubectl delete namespace postgres-demo
kubectl delete namespace datadog
helm uninstall datadog-agent -n datadog
minikube delete
```

---

## Troubleshooting

### No explain plans showing in UI

1. **Check agent is running:**
   ```bash
   kubectl get pods -n datadog -l app=datadog-agent
   ```

2. **Check postgres check is OK:**
   ```bash
   kubectl exec -n datadog <agent-pod> -- agent status | grep postgres
   ```

3. **Check for errors in agent logs:**
   ```bash
   kubectl logs -n datadog <agent-pod> -c agent | grep -E "(ERROR|WARN)" | grep postgres
   ```

4. **Verify function exists and is callable:**
   ```bash
   kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "SELECT datadog.explain_statement('SELECT 1');"
   ```

5. **Restart the agent:**
   ```bash
   kubectl rollout restart daemonset/datadog-agent -n datadog
   ```

---

## References

- [Datadog PostgreSQL Database Monitoring Setup (Self-Hosted)](https://docs.datadoghq.com/database_monitoring/setup_postgres/selfhosted/)
- [Datadog PostgreSQL Integration](https://docs.datadoghq.com/integrations/postgres/)
- [Datadog Database Monitoring Troubleshooting](https://docs.datadoghq.com/database_monitoring/troubleshooting/)
- [Datadog Kubernetes Integration](https://docs.datadoghq.com/containers/kubernetes/)
- [Datadog Agent Autodiscovery](https://docs.datadoghq.com/containers/docker/integrations/)
- [DBExplainError Enum (Agent Source Code)](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/corechecks/database_monitoring/dbm_common/explain_errors.go)
- [PostgreSQL SELECT ... FOR UPDATE Documentation](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)
