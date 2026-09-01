# Incident 20: Outbox Event Publishing Lag (Uncommitted Batch Transaction WAL Stall)

---

## 1. Symptoms & Alert
- **Alert**: `DebeziumCDCLagHigh > 300s` and `OutboxUnpublishedEventCount > 500,000`.
- **Customer Impact**: Asynchronous search index updates, notifications, and analytics pipelines stalled by 45 minutes.

---

## 2. Metric & Telemetry Anomalies
- **Debezium Metrics**: `debezium_postgres_wal_offset_lag` continuously increasing; zero events emitted to Kafka topic `outbox.events` for 45 minutes.
- **PostgreSQL Replication Slots**: `pg_replication_slots.confirmed_flush_lsn` remained completely frozen.

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Check Active Long-Running PostgreSQL Transactions
```sql
SELECT pid, age(clock_timestamp(), xact_start), usename, state, query 
FROM pg_stat_activity 
WHERE state != 'idle' 
ORDER BY xact_start ASC 
LIMIT 5;
```
- *Output*:
```text
PID: 8412
Age: 48 minutes 12 seconds
Query: INSERT INTO legacy_data_migration SELECT ... (Batch transaction writing 20M rows in a single TX!)
```

### Root Cause:
1. A DBA started a manual data migration script that executed inside a **Single Massive Uncommitted Transaction** lasting 48 minutes.
2. In PostgreSQL logical replication, the Write-Ahead Log (WAL) engine **cannot advance the logical replication slot's LSN past an open, uncommitted transaction's `xmin`**.
3. Consequently, the **Debezium CDC connector was completely blocked from publishing any outbox events to Kafka**, stalling the entire event-driven messaging architecture.

---

## 4. Immediate Mitigation
Terminate the uncommitted batch migration query:
```sql
SELECT pg_terminate_backend(8412);
```
*(Debezium immediately resumed streaming WAL events and drained the 500,000 queued outbox events in 90 seconds).*

---

## 5. Permanent Fix
1. **Enforce Chunked Batching for Data Migrations**:
   - Rewrite migration scripts to commit every 1,000 rows in separate transactions, never holding an open transaction for more than $500\text{ms}$.
2. **Configure PostgreSQL Transaction Ceilings**:
   ```text
   idle_in_transaction_session_timeout = '15s'
   statement_timeout = '30s'
   ```

---

## 6. Postmortem Action Items
- [x] Configure PagerDuty alert when any single database transaction duration exceeds 60 seconds.
- [x] Audit all internal scripts to mandate batching and automatic chunk commits.
