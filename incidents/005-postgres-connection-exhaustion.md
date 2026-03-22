# Incident 005: PostgreSQL Connection Exhaustion

**Date:** February 2024  
**Severity:** P1 (Full Outage)  
**Duration:** 8 minutes  
**Impact:** All API requests failing; indexer stalled; frontend showing "Service Unavailable"  

---

## Summary

The `suidex-postgres` instance hit its maximum connection limit (100), causing all services to fail with `FATAL: too many connections for role "suidex"`. The API, indexer, and worker backends all went down simultaneously. Root cause was a connection leak in the indexer during error handling combined with no connection pooling.

## Detection

- **09:12 UTC** — Grafana alert: `PostgresConnectionsUsed > 90%` (93 of 100)
- **09:13 UTC** — API health checks start failing: `Error: Connection terminated unexpectedly`
- **09:13 UTC** — All pods CrashLoopBackOff — can't establish DB connection on startup
- **09:14 UTC** — PagerDuty alert: "suidex-api-backend Pod Restarts > 5"

## Timeline

```
09:00  Indexer processes large batch of swap events (on-chain spike)
09:08  Indexer errors on malformed event → catch block doesn't release connection
09:10  Leaked connections accumulate (indexer retries each batch with new connection)
09:12  100/100 connections used; Grafana alert fires
09:13  API and worker pods fail to get connections; health checks fail
09:13  Kubernetes restarts all failing pods (CrashLoopBackOff)
09:14  Engineer investigates; confirms connection saturation
09:16  Immediate fix: kill idle connections via psql
09:18  Pods recover; services resume
09:20  All clear; monitoring confirms normal connection count (~25)
```

## Root Cause

**1. Connection leak in indexer error handling**

```javascript
// suidex-indexer-backend — BEFORE (buggy)
async function processSwapEvents(events) {
  const client = await pool.connect();    // Acquires connection
  try {
    await client.query('BEGIN');
    for (const event of events) {
      await insertSwapEvent(client, event);  // Throws on malformed data
    }
    await client.query('COMMIT');
    client.release();                      // Only releases on success path
  } catch (err) {
    await client.query('ROLLBACK');
    logger.error('Batch failed:', err);
    // BUG: client.release() missing in catch block!
    // Connection leaked on every error
    throw err;                             // Re-thrown → retry loop acquires new connection
  }
}
```

**2. No connection pool limits per service**

Every service connected with its own pool, no max limit:
```javascript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  // No max setting → defaults to 10 per pool
  // 4 services × 5 replicas × 10 = 200 possible connections
  // PostgreSQL max_connections = 100
});
```

**3. No PgBouncer or connection proxy**

Direct connections from every pod to PostgreSQL with no multiplexing.

## Resolution

### Immediate (during incident)
```sql
-- Kill idle connections to restore service
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND pid <> pg_backend_pid()
  AND state_change < NOW() - INTERVAL '30 seconds';
```

### Fix 1: Fixed connection leak with `finally` block
```javascript
async function processSwapEvents(events) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    for (const event of events) {
      await insertSwapEvent(client, event);
    }
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    logger.error('Batch failed:', err);
    throw err;
  } finally {
    client.release();    // ALWAYS release, regardless of success/failure
  }
}
```

### Fix 2: Pool configuration per service
```javascript
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 5,                 // Hard limit per pool
  min: 1,                 // Keep 1 warm connection
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,  // Fail fast instead of hanging
});

// Log pool errors
pool.on('error', (err) => {
  logger.error('Unexpected pool error:', err);
});
```

Connection budget:
```
api-backend:     3 pods × 5 connections = 15
indexer-backend: 2 pods × 3 connections = 6
worker-backend:  2 pods × 3 connections = 6
event-backend:   2 pods × 3 connections = 6
Reserved for admin/monitoring:           = 5
                                  Total: 38 of 100
```

### Fix 3: Deployed PgBouncer as sidecar
```yaml
# suidex-postgres StatefulSet — pgbouncer sidecar
- name: pgbouncer
  image: pgbouncer/pgbouncer:1.22.0
  ports:
    - containerPort: 6432
  env:
    - name: DATABASES_HOST
      value: "127.0.0.1"
    - name: DATABASES_PORT
      value: "5432"
    - name: DATABASES_DBNAME
      value: "suidex"
    - name: PGBOUNCER_POOL_MODE
      value: "transaction"
    - name: PGBOUNCER_DEFAULT_POOL_SIZE
      value: "20"
    - name: PGBOUNCER_MAX_CLIENT_CONN
      value: "200"
    - name: PGBOUNCER_MIN_POOL_SIZE
      value: "5"
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi
```

Services now connect to PgBouncer (port 6432) instead of PostgreSQL directly.

### Fix 4: PostgreSQL connection monitoring
```yaml
- alert: PostgresConnectionsHigh
  expr: |
    pg_stat_activity_count{datname="suidex"}
    / pg_settings_max_connections > 0.7
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "PostgreSQL connections at {{ $value | humanizePercentage }}"

- alert: PostgresConnectionsCritical
  expr: |
    pg_stat_activity_count{datname="suidex"}
    / pg_settings_max_connections > 0.9
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "PostgreSQL connections CRITICAL: {{ $value | humanizePercentage }}"
```

## Preventive Measures

1. **Code review checklist** — verify `client.release()` in `finally` blocks for all database operations
2. **Linting rule** — ESLint custom rule: flag `pool.connect()` without corresponding `finally { client.release() }`
3. **Connection pool dashboards** — Grafana panel showing per-service connection counts
4. **PgBouncer** — transaction-mode pooling handles connection multiplexing; services can't exhaust PostgreSQL
5. **Idle connection reaper** — PostgreSQL config:
   ```
   idle_in_transaction_session_timeout = 30s
   statement_timeout = 30s
   ```
6. **Load testing** — connection exhaustion scenario added to pre-deploy tests

## Lessons Learned

- `finally` blocks for resource cleanup are non-negotiable. Node.js `pg` library requires explicit `client.release()`.
- Connection pool defaults (10 per pool) multiply across replicas and services. Always set explicit `max` per service.
- PgBouncer in transaction mode is essential for any Kubernetes deployment with multiple services sharing PostgreSQL. It absorbed a 3x traffic spike without issues after deployment.
- Monitoring connection usage percentage, not just count, catches trends before they become outages.
- The 8-minute outage could have been avoided with a single `finally` block and a 70% connection threshold alert.
