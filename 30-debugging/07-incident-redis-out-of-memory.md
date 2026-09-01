# Incident 07: Redis Out of Memory (OOM Command Rejection)

---

## 1. Symptoms & Alert
- **Alert**: `RedisException: OOM command not allowed when used memory > 'maxmemory'`.
- **Customer Impact**: User logins and session creations failing with HTTP 500 across all microservices; Redis refusing all write commands (`SET`, `HSET`, `LPUSH`).

---

## 2. Metric & Telemetry Anomalies
- **Redis Memory**: `used_memory` hit $100\%$ of allocated `maxmemory` ($8.0\text{GB}$).
- **Eviction Metrics**: `evicted_keys` metric sat flat at `0` despite memory being full.

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Check Redis Configuration
```bash
redis-cli CONFIG GET maxmemory-policy
# Output:
# 1) "maxmemory-policy"
# 2) "noeviction"   <--- DEFAULT DANGEROUS POLICY!
```

### Step 2: Analyze Big Keys in Redis
```bash
redis-cli --bigkeys
# Output: Found unbounded queue 'jobs:audit:events' holding 4,200,000 items (3.8 GB)!
```

### Root Cause:
1. Redis was running with `maxmemory-policy: noeviction`. Under this policy, when memory reaches `maxmemory`, Redis **does not evict any keys**; instead, it immediately returns `OOM` errors on all write commands.
2. An unmonitored audit log queue had accumulated 4.2 million unconsumed elements, consuming the entire 8GB memory footprint.

---

## 4. Immediate Mitigation
1. **Switch Eviction Policy Dynamically without Restart**:
   ```bash
   redis-cli CONFIG SET maxmemory-policy allkeys-lru
   ```
2. **Purge the Orphaned Unbounded Queue**:
   ```bash
   redis-cli UNLINK jobs:audit:events
   ```

---

## 5. Permanent Fix
1. Update `redis.conf` with production eviction policy:
   ```text
   maxmemory 8gb
   maxmemory-policy allkeys-lru
   ```
2. Move persistent queues from Redis to Apache Kafka (which stores messages on NVMe disk rather than RAM).

---

## 6. Postmortem Action Items
- [x] Configure CloudWatch alert at $80\%$ Redis memory utilization.
- [x] Audit all Redis keyspaces to guarantee explicit TTLs on ephemeral data.
