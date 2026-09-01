# PostgreSQL Performance & Optimization Cheat Sheet

---

## ⚡ 1. Indexing Cheat Sheet

```sql
-- 1. Standard B+Tree Index (Point lookups & range scans)
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);

-- 2. Partial Index (Index active records only; shrinks index size by 90%)
CREATE INDEX idx_orders_unprocessed ON orders (created_at) WHERE status = 'PENDING';

-- 3. GIN Index (JSONB documents & array containment queries)
CREATE INDEX idx_products_metadata_gin ON products USING GIN (metadata jsonb_path_ops);

-- 4. BRIN Index (Block Range Index for massive append-only time-series tables)
CREATE INDEX idx_audit_logs_brin ON audit_logs USING BRIN (created_at);
```

---

## ⚡ 2. Query Diagnostics (`EXPLAIN ANALYZE`)

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT * FROM orders WHERE user_id = 42 AND status = 'COMPLETED';

-- Diagnostic Red Flags:
-- 1. "Seq Scan on large_table" -> Missing index!
-- 2. "Rows Removed by Filter: 500000" -> Inefficient composite index column order!
-- 3. "Shared Read Blocks > 10000" -> Reading heavily from physical disk rather than RAM buffer pool!
```

---

## ⚡ 3. Connection Pool Sizing (HikariCP)

$$\textbf{Max Pool Size} = (\text{CPU Cores} \times 2) + \text{Disk Spindles}$$

```properties
# HikariCP Production Settings
spring.datasource.hikari.maximum-pool-size=32
spring.datasource.hikari.minimum-idle=32       # Eliminate runtime connection creation latency!
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000 # 30 mins (Must be < DB wait_timeout)
spring.datasource.hikari.leak-detection-threshold=5000 # Detects unclosed connections in 5s
```

---

## ⚡ 4. PostgreSQL Maintenance Queries

```sql
-- Check active locking queries & connection states
SELECT pid, age(clock_timestamp(), query_start), usename, state, query 
FROM pg_stat_activity 
WHERE state != 'idle' 
ORDER BY query_start ASC;

-- Terminate rogue locking query
SELECT pg_terminate_backend(<pid>);

-- Check table dead tuple bloat
SELECT relname, n_live_tup, n_dead_tup, 
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_ratio
FROM pg_stat_user_tables 
ORDER BY n_dead_tup DESC;
```
