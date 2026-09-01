# Incident 05: Slow Database Queries and Table Scans

---

## 1. Symptoms & Alert
- **Alert**: `RDS Database CPU > 95% for 10 minutes`.
- **Customer Impact**: Application $p99$ response times degraded from $40\text{ms}$ to $> 4,500\text{ms}$; database disk read IOPS spiked to max hardware burst limits.

---

## 2. Metric & Telemetry Anomalies
- **PostgreSQL RDS Metrics**: Read IOPS peaked at $15,000\text{ IOPS}$; CPU pinned at $98\%$.
- **Buffer Pool**: `Buffer Cache Hit Ratio` plummeted from $99.8\%$ to **$42.1\%$** (severe page cache thrashing).

---

## 3. Diagnostic Steps

### Step 1: Identify Slowest Queries via `pg_stat_statements`
```sql
SELECT query, calls, total_exec_time / calls AS avg_time_ms, rows, shared_blks_read
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;
```
- *Output*:
```text
Query: SELECT * FROM audit_logs WHERE tenant_id = $1 AND event_type = $2 ORDER BY created_at DESC LIMIT 20;
Calls: 45,000
Avg Time: 850.4ms
Shared Blocks Read: 120,400 per query!
```

### Step 2: Run `EXPLAIN (ANALYZE, BUFFERS)` on Query
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM audit_logs WHERE tenant_id = 42 AND event_type = 'LOGIN' ORDER BY created_at DESC LIMIT 20;
```
- *Plan Output*:
```text
->  Sort (cost=450210.12..450215.12 rows=20 width=142)
      ->  Seq Scan on audit_logs (cost=0.00..420100.00 rows=15000 width=142)
            Filter: ((tenant_id = 42) AND (event_type = 'LOGIN'::text))
            Rows Removed by Filter: 14,850,210  <--- SEQUENTIAL SCAN OF 15M ROWS!
            Buffers: shared read=124500
```

---

## 4. Root Cause Analysis
The `audit_logs` table had grown to 15 million rows. A new dashboard feature queried logs by `(tenant_id, event_type, created_at)`. Because no matching composite index existed, PostgreSQL was forced to execute a **Full Table Scan (Sequential Scan)** across 15M rows on disk for every single dashboard page load ($45,000\text{ times/hour}$).

---

## 5. Immediate Mitigation
Create the missing compound B+Tree index **concurrently** (without locking the live production table):

```sql
-- Zero downtime index creation!
CREATE INDEX CONCURRENTLY idx_audit_logs_tenant_event_created 
ON audit_logs (tenant_id, event_type, created_at DESC);
```

---

## 6. Verification & Performance After Fix
Re-running `EXPLAIN (ANALYZE, BUFFERS)` after index creation:
```text
->  Limit (cost=0.56..12.40 rows=20 width=142)
      ->  Index Scan using idx_audit_logs_tenant_event_created on audit_logs
            Index Cond: ((tenant_id = 42) AND (event_type = 'LOGIN'::text))
            Buffers: shared hit=4 read=0
Execution Time: 0.42 ms  (DOWN FROM 850ms - 2,000x FASTER!)
```

---

## 7. Postmortem Action Items
- [x] Configure automated CI check rejecting SQL migrations without accompanying index analysis.
- [x] Enable PostgreSQL `auto_explain` extension for queries exceeding $500\text{ms}$.
