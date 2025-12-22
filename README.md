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
        user_id INTEGER REFERENCES users(id),
        product VARCHAR(200) NOT NULL,
        amount DECIMAL(10,2) NOT NULL,
        status VARCHAR(50) DEFAULT 'pending',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    -- Insert sample data
    INSERT INTO users (name, email) VALUES 
        ('Alice Johnson', 'alice@example.com'),
        ('Bob Smith', 'bob@example.com'),
        ('Charlie Brown', 'charlie@example.com');
    
    INSERT INTO orders (user_id, product, amount, status) VALUES
        (1, 'Laptop', 999.99, 'completed'),
        (1, 'Mouse', 29.99, 'completed'),
        (2, 'Keyboard', 79.99, 'pending'),
        (3, 'Monitor', 349.99, 'shipped');
    
    -- Create indexes for query optimization demo
    CREATE INDEX idx_orders_user_id ON orders(user_id);
    CREATE INDEX idx_orders_status ON orders(status);
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
    tags.datadoghq.com/version: "15"
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
        tags.datadoghq.com/version: "15"
      annotations:
        ad.datadoghq.com/postgres.checks: |
          {
            "postgres": {
              "instances": [{
                "dbm": true,
                "host": "%%host%%",
                "port": 5432,
                "username": "datadog",
                "password": "datadog_password",
                "dbname": "demo_app"
              }]
            }
          }
        ad.datadoghq.com/postgres.logs: '[{"source":"postgresql","service":"postgres-demo"}]'
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
            - "pg_stat_statements.max=10000"
            - "-c"
            - "track_activity_query_size=4096"
          volumeMounts:
            - name: postgres-init
              mountPath: /docker-entrypoint-initdb.d
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: postgres-init
          configMap:
            name: postgres-init
        - name: postgres-data
          emptyDir: {}
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
  type: ClusterIP
```

#### app/demo-app.yaml

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app-code
  namespace: postgres-demo
data:
  main.py: |
    import os
    import time
    import psycopg2
    from flask import Flask, jsonify

    app = Flask(__name__)

    def get_db_connection():
        return psycopg2.connect(
            host=os.environ.get('POSTGRES_HOST', 'postgres'),
            port=os.environ.get('POSTGRES_PORT', 5432),
            dbname=os.environ.get('POSTGRES_DB', 'demo_app'),
            user=os.environ.get('POSTGRES_USER', 'postgres'),
            password=os.environ.get('POSTGRES_PASSWORD', 'datadog123')
        )

    @app.route('/health')
    def health():
        return jsonify({"status": "healthy"})

    @app.route('/users')
    def get_users():
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT id, name, email, created_at FROM users")
        users = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"users": [{"id": u[0], "name": u[1], "email": u[2]} for u in users]})

    @app.route('/orders')
    def get_orders():
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT o.id, u.name, o.product, o.amount, o.status, o.created_at
            FROM orders o
            JOIN users u ON o.user_id = u.id
            ORDER BY o.created_at DESC
        """)
        orders = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"orders": [
            {"id": o[0], "user": o[1], "product": o[2], "amount": float(o[3]), "status": o[4]}
            for o in orders
        ]})

    @app.route('/orders/stats')
    def get_order_stats():
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT 
                u.name,
                COUNT(o.id) as order_count,
                SUM(o.amount) as total_amount,
                AVG(o.amount) as avg_amount
            FROM users u
            LEFT JOIN orders o ON u.id = o.user_id
            GROUP BY u.id, u.name
            HAVING COUNT(o.id) > 0
            ORDER BY total_amount DESC
        """)
        stats = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"stats": [
            {"user": s[0], "order_count": s[1], "total": float(s[2]), "average": float(s[3])}
            for s in stats
        ]})

    @app.route('/slow-query')
    def slow_query():
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT pg_sleep(0.5), * FROM orders WHERE status IN ('pending', 'shipped')")
        orders = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"message": "Slow query completed", "count": len(orders)})

    @app.route('/generate-traffic')
    def generate_traffic():
        conn = get_db_connection()
        cur = conn.cursor()
        queries = [
            "SELECT COUNT(*) FROM users",
            "SELECT * FROM orders WHERE status = 'completed'",
            "SELECT u.name, COUNT(o.id) FROM users u LEFT JOIN orders o ON u.id = o.user_id GROUP BY u.name",
            "SELECT * FROM orders WHERE amount > 100 ORDER BY amount DESC",
            "SELECT DISTINCT status FROM orders",
        ]
        results = []
        for q in queries:
            cur.execute(q)
            results.append({"query": q[:50] + "...", "rows": cur.rowcount})
        cur.close()
        conn.close()
        return jsonify({"executed_queries": len(queries), "results": results})

    @app.route('/restricted')
    def get_restricted():
        """
        Query restricted_schema.secret_table using psycopg3 (extended query protocol).
        This generates parameterized queries with $1 placeholders.
        Used to reproduce 'failed_to_explain_with_prepared_statement' error.
        """
        import random
        try:
            import psycopg
            conn = psycopg.connect(
                host=os.environ.get('POSTGRES_HOST', 'postgres'),
                port=int(os.environ.get('POSTGRES_PORT', 5432)),
                dbname=os.environ.get('POSTGRES_DB', 'demo_app'),
                user=os.environ.get('POSTGRES_USER', 'postgres'),
                password=os.environ.get('POSTGRES_PASSWORD', 'datadog123')
            )
            cur = conn.cursor()
            # Parameterized query - will have $1 placeholder in pg_stat_statements
            random_id = random.randint(1, 5)
            cur.execute("SELECT * FROM restricted_schema.secret_table WHERE id = %s", (random_id,))
            rows = cur.fetchall()
            cur.close()
            conn.close()
            return jsonify({"restricted_data": [{"id": r[0], "data": r[1]} for r in rows]})
        except Exception as e:
            return jsonify({"error": str(e)}), 500

    if __name__ == '__main__':
        print("Starting demo app on port 8080...")
        app.run(host='0.0.0.0', port=8080)
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
      annotations:
        ad.datadoghq.com/demo-app.logs: '[{"source":"python","service":"demo-app"}]'
    spec:
      containers:
        - name: demo-app
          image: python:3.11-slim
          command: ["/bin/bash", "-c"]
          args:
            - |
              pip install --quiet psycopg2-binary 'psycopg[binary]' flask
              echo "Waiting for PostgreSQL..."
              sleep 5
              python /app/main.py
          ports:
            - containerPort: 8080
          env:
            - name: POSTGRES_HOST
              value: "postgres"
            - name: POSTGRES_PORT
              value: "5432"
            - name: POSTGRES_DB
              value: "demo_app"
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
          volumeMounts:
            - name: app-code
              mountPath: /app
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
      volumes:
        - name: app-code
          configMap:
            name: demo-app-code
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
  clusterName: "postgres-lab"
  kubelet:
    tlsVerify: false
  logs:
    enabled: true
    containerCollectAll: true

clusterAgent:
  enabled: true

agents:
  enabled: true
```

### Step 3: Deploy everything

```bash
# Create directory structure
mkdir -p postgres app datadog

# Apply manifests
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f app/demo-app.yaml

# Create Datadog secret (replace with your API key)
kubectl create secret generic datadog-secret -n datadog --from-literal=api-key=YOUR_API_KEY

# Deploy Datadog Agent
helm repo add datadog https://helm.datadoghq.com
helm repo update
helm install datadog-agent datadog/datadog -n datadog --create-namespace -f datadog/values.yaml
```

### Step 4: Verify deployment

```bash
# Check pods are running
kubectl get pods -n postgres-demo
kubectl get pods -n datadog

# Check Datadog Agent postgres integration
kubectl exec -n datadog $(kubectl get pods -n datadog -l app=datadog-agent -o jsonpath='{.items[0].metadata.name}') -- agent status | grep -A 10 "postgres"
```

## DBM Configuration

The PostgreSQL integration is configured via Kubernetes annotations:

```yaml
ad.datadoghq.com/postgres.checks: |
  {
    "postgres": {
      "instances": [{
        "dbm": true,
        "host": "%%host%%",
        "port": 5432,
        "username": "datadog",
        "password": "datadog_password",
        "dbname": "demo_app"
      }]
    }
  }
```

> **Note:** `dbm: true` automatically enables query samples, query metrics, and `explain_parameterized_queries`.

---

## Test Cases

### 0. ✅ Working Explain Plans (Baseline)

**Expected Behavior:** Explain plans are collected successfully for all queries.

**Prerequisites:**
- `datadog.explain_statement` function exists with `SECURITY DEFINER`
- `datadog` user has `EXECUTE` permission on the function
- `datadog` user has `pg_monitor` role

#### Step 1: Verify explain function exists

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
SELECT routine_name FROM information_schema.routines WHERE routine_schema = 'datadog';
"
```

Expected output:
```
   routine_name    
-------------------
 explain_statement
(1 row)
```

#### Step 2: Test function as datadog user

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U datadog -d demo_app -c "
SELECT datadog.explain_statement('SELECT * FROM users WHERE id = 1');
"
```

Expected output: A JSON explain plan showing the query execution details.

#### Step 3: Generate traffic via the demo-app API

```bash
kubectl port-forward -n postgres-demo svc/demo-app 8080:8080 &
sleep 2

# Generate traffic for 60 seconds (every 0.5 sec)
for i in {1..120}; do
  curl -s http://localhost:8080/generate-traffic > /dev/null &
  curl -s http://localhost:8080/users > /dev/null &
  curl -s http://localhost:8080/orders > /dev/null &
  curl -s http://localhost:8080/orders/stats > /dev/null &
  sleep 0.5
done

pkill -f "port-forward.*8080"
echo "Traffic generated - check DBM UI for explain plans"
```

#### Expected Result in Datadog DBM UI

- Query samples appear for queries like:
  - `SELECT * FROM users`
  - `SELECT * FROM orders`
  - `SELECT COUNT(*) FROM users`
- Each query sample has an **Explain Plan** tab with execution details
- No error messages on explain plan collection

<!-- Add your screenshot here -->
<!-- ![Working Explain Plan](./screenshots/working-explain-plan.png) -->

---

## Reproduced Error Cases

### 1. ❌ Undefined Explain Function

**Error Code:** `undefined_function` / `undefined-explain-function`

**UI Message:**
> Unable to collect explain plan due to an undefined function error
> The explain plan could not be collected because the query contains a function with a parameter whose datatype could not be determined.

**How to Reproduce:**

```bash
# Drop the explain function
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
DROP FUNCTION IF EXISTS datadog.explain_statement(TEXT);
"

# Generate traffic via curl
kubectl port-forward -n postgres-demo svc/demo-app 8080:8080 &
sleep 2

# Generate traffic for 60 seconds (every 0.5 sec)
for i in {1..120}; do
  curl -s http://localhost:8080/generate-traffic > /dev/null &
  curl -s http://localhost:8080/users > /dev/null &
  curl -s http://localhost:8080/orders > /dev/null &
  sleep 0.5
done

pkill -f "port-forward.*8080"
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

### 2. ❌ Failed Function (Permission Issues)

**Error Code:** `failed_function`

**UI Message:**
> Unable to collect explain plan
> There could be a problem with the function used to collect Explain Plans. This may be because the function is missing, the Agent has insufficient permissions, or an incorrect function definition.

**How to Reproduce:**

```bash
# Create function WITHOUT SECURITY DEFINER and revoke permissions
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
DROP FUNCTION IF EXISTS datadog.explain_statement(TEXT);

CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
\$\$
DECLARE
curs REFCURSOR;
plan JSON;
BEGIN
   OPEN curs FOR EXECUTE pg_catalog.concat('EXPLAIN (FORMAT JSON) ', l_query);
   FETCH curs INTO plan;
   CLOSE curs;
   RETURN QUERY SELECT plan;
END;
\$\$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT;

-- Revoke execute permission from PUBLIC and datadog
REVOKE EXECUTE ON FUNCTION datadog.explain_statement(TEXT) FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION datadog.explain_statement(TEXT) FROM datadog;
"

# Generate traffic for 60 seconds (every 0.5 sec)
kubectl port-forward -n postgres-demo svc/demo-app 8080:8080 &
sleep 2

for i in {1..120}; do
  curl -s http://localhost:8080/generate-traffic > /dev/null &
  curl -s http://localhost:8080/users > /dev/null &
  curl -s http://localhost:8080/orders > /dev/null &
  sleep 0.5
done

pkill -f "port-forward.*8080"
```

**Fix:**

Recreate the function with `SECURITY DEFINER` and grant execute permission:

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

Or run:

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
\\\$\\\$
DECLARE
curs REFCURSOR;
plan JSON;
BEGIN
   OPEN curs FOR EXECUTE pg_catalog.concat('EXPLAIN (FORMAT JSON) ', l_query);
   FETCH curs INTO plan;
   CLOSE curs;
   RETURN QUERY SELECT plan;
END;
\\\$\\\$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
"
```

---

### 3. ❌ Failed to Explain with Prepared Statement (Lack of Permissions)

**Error Code:** `failed_to_explain_with_prepared_statement`

**UI Message:**
> The Datadog agent user lacks permission to execute the parameterized query
> Execution of the EXPLAIN command failed because the datadog user does not have sufficient permissions to explain the specified query on the target table. Please ensure the user has the necessary permissions on the table involved in the query. Granting the necessary COMMAND permission on the relevant table(s) using the GRANT SQL command should resolve this issue.

**How to Reproduce:**

```bash
# Step 1: Ensure explain function exists with SECURITY DEFINER
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
\$\$
DECLARE
curs REFCURSOR;
plan JSON;
BEGIN
   OPEN curs FOR EXECUTE pg_catalog.concat('EXPLAIN (FORMAT JSON) ', l_query);
   FETCH curs INTO plan;
   CLOSE curs;
   RETURN QUERY SELECT plan;
END;
\$\$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
"

# Step 2: Create a restricted schema that datadog cannot access
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
CREATE SCHEMA IF NOT EXISTS restricted_schema;
REVOKE ALL ON SCHEMA restricted_schema FROM datadog;
REVOKE ALL ON SCHEMA restricted_schema FROM PUBLIC;

DROP TABLE IF EXISTS restricted_schema.secret_table;
CREATE TABLE restricted_schema.secret_table (
    id SERIAL PRIMARY KEY,
    data TEXT
);
INSERT INTO restricted_schema.secret_table (data) VALUES ('secret1'), ('secret2'), ('secret3'), ('secret4'), ('secret5');
"

# Step 3: Generate traffic via curl to /restricted endpoint (uses psycopg3 extended query protocol)
kubectl port-forward -n postgres-demo svc/demo-app 8080:8080 &
sleep 2

# Generate traffic for 60 seconds (every 0.5 sec)
for i in {1..120}; do
  curl -s http://localhost:8080/users > /dev/null &
  curl -s http://localhost:8080/orders > /dev/null &
  curl -s http://localhost:8080/restricted > /dev/null &
  sleep 0.5
done

pkill -f "port-forward.*8080"
```

**Why it works:**
1. Demo-app `/restricted` endpoint uses **psycopg3** (extended query protocol)
2. App runs `SELECT * FROM restricted_schema.secret_table WHERE id = $1`
3. Agent sees query in `pg_stat_statements` with `$1` placeholder
4. Agent tries to PREPARE: `PREPARE dd_xxx AS SELECT * FROM restricted_schema.secret_table WHERE id = $1`
5. PREPARE fails: `permission denied for schema restricted_schema`
6. Agent emits `failed_to_explain_with_prepared_statement`

> **Note:** The `/restricted` endpoint uses `psycopg` (psycopg3) instead of `psycopg2` because psycopg3 uses the extended query protocol which sends parameterized queries with `$1` placeholders.

**Fix:**

Grant the `datadog` user access to the restricted schema and table:

```sql
-- Grant schema access
GRANT USAGE ON SCHEMA restricted_schema TO datadog;

-- Grant table access
GRANT SELECT ON restricted_schema.secret_table TO datadog;
```

Or run:

```bash
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
GRANT USAGE ON SCHEMA restricted_schema TO datadog;
GRANT SELECT ON restricted_schema.secret_table TO datadog;
"
```

---

### 4. ❌ Database Error (Configuration Error)

**Error Code:** `database_error`

**UI Message:**
> Unable to collect explain plan
> This may be due to a configuration error.

**How to Reproduce:**

This is a catch-all error for various database-level issues. One way to trigger it:

```bash
# Create function that returns invalid result
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -d demo_app -c "
DROP FUNCTION IF EXISTS datadog.explain_statement(TEXT);

CREATE OR REPLACE FUNCTION datadog.explain_statement(
   l_query TEXT,
   OUT explain JSON
)
RETURNS SETOF JSON AS
\$\$
BEGIN
   -- Return NULL instead of a valid plan
   RETURN QUERY SELECT NULL::JSON;
END;
\$\$
LANGUAGE 'plpgsql'
RETURNS NULL ON NULL INPUT
SECURITY DEFINER;

GRANT EXECUTE ON FUNCTION datadog.explain_statement(TEXT) TO datadog;
"

# Generate traffic for 60 seconds (every 0.5 sec)
kubectl port-forward -n postgres-demo svc/demo-app 8080:8080 &
sleep 2

for i in {1..120}; do
  curl -s http://localhost:8080/generate-traffic > /dev/null &
  curl -s http://localhost:8080/users > /dev/null &
  curl -s http://localhost:8080/orders > /dev/null &
  sleep 0.5
done

pkill -f "port-forward.*8080"
```

---

### 5. ❌ Query Truncated

**Error Code:** `query_truncated`

**UI Message:**
> Explain plan unavailable due to query truncation
> Explain plan was not collected for this query as it exceeded your `track_activity_query_size` limit.

**How to Reproduce:**

```bash
# Set very low track_activity_query_size and restart PostgreSQL
kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -c "
ALTER SYSTEM SET track_activity_query_size = 100;
"
# Restart postgres to apply the setting
kubectl rollout restart deployment/postgres -n postgres-demo
kubectl rollout status deployment/postgres -n postgres-demo

# Wait for postgres to be ready
sleep 10

# Generate long queries for 60 seconds (every 0.5 sec)
kubectl exec -n postgres-demo deployment/demo-app -- pip install 'psycopg[binary]' -q

for i in {1..120}; do
  kubectl exec -n postgres-demo deployment/demo-app -- python3 -c "
import psycopg
conn = psycopg.connect(host='postgres', port=5432, dbname='demo_app', user='postgres', password='datadog123')
cur = conn.cursor()
# Very long query that will be truncated
long_condition = ' OR '.join([f'name = \\'test{j}\\'' for j in range(50)])
cur.execute(f'SELECT * FROM users WHERE {long_condition}')
cur.close()
conn.close()
" 2>/dev/null &
  sleep 0.5
done

echo "Traffic generated - check DBM UI for 'query truncation' error"

# Don't forget to restore track_activity_query_size after testing!
# kubectl exec -n postgres-demo deployment/postgres -- psql -U postgres -c "ALTER SYSTEM RESET track_activity_query_size;"
# kubectl rollout restart deployment/postgres -n postgres-demo
```

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
- [`DBExplainError` enum](https://github.com/DataDog/integrations-core/blob/467275409f6a9367f22155a872f949d26f7eca2c/postgres/datadog_checks/postgres/util.py#L46)

| Error Code | UI Message | Cause |
|------------|------------|-------|
| `database_error` | Unable to collect explain plan (configuration error) | Generic database-level error or misconfiguration |
| `datatype_mismatch` | - | Return type is not JSON (e.g., multiple queries explained) |
| `invalid_schema` | Missing function in the datadog schema | Schema or function missing |
| `invalid_result` | - | Value retrieved from EXPLAIN function is invalid |
| `no_plans_possible` | - | Statement cannot be explained (e.g., AUTOVACUUM) |
| `failed_function` | Unable to collect explain plan (function problem) | Function exists but has issues (permissions, definition) |
| `query_truncated` | Explain plan unavailable due to query truncation | Query exceeds `track_activity_query_size` |
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
kubectl delete -f app/demo-app.yaml
kubectl delete -f postgres/postgres-deployment.yaml
helm uninstall datadog-agent -n datadog
kubectl delete namespace postgres-demo
kubectl delete namespace datadog
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

### psycopg module not found

The demo-app pod may have restarted. Reinstall:

```bash
kubectl exec -n postgres-demo deployment/demo-app -- pip install 'psycopg[binary]' -q
```
