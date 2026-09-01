# Incident 18: Rate Limiter Cascading Failure (Fail-Closed Redis Outage)

---

## 1. Symptoms & Alert
- **Alert**: `Global HTTP 429 Too Many Requests Spike > 90%`.
- **Customer Impact**: 100% of legitimate users locked out of the platform; API Gateway rejecting all requests with HTTP 429.

---

## 2. Metric & Telemetry Anomalies
- **Ingress Traffic**: Normal traffic volume ($1,200\text{ QPS}$); no DDoS attack occurring.
- **Rate Limiting Redis Cluster**: Redis node in AZ-1 crashed due to a hardware node failure.

---

## 3. Diagnostic Steps & Root Cause
### Inspecting API Gateway Rate Limiting Filter Code:
```java
// DISASTER: FAIL-CLOSED RATE LIMITING DESIGN!
public boolean allowRequest(String apiKey) {
    try {
        return redisRateLimiter.tryAcquire(apiKey);
    } catch (RedisConnectionException ex) {
        log.error("Redis is down!");
        return false; // FAIL-CLOSED: BLOCKS 100% OF LEGITIMATE TRAFFIC ON REDIS OUTAGE!
    }
}
```

### Root Cause:
The API Gateway rate limiting filter was implemented with a **Fail-Closed Architecture**:
When the underlying Redis cluster experienced a transient connection dropout, the exception handler returned `false`, rejecting $100\%$ of legitimate customer requests with `HTTP 429 Too Many Requests`.

---

## 4. Immediate Mitigation
Deploy emergency configuration bypass disabling the rate limiting filter on the API Gateway:
```yaml
rate-limiter:
  enabled: false
```

---

## 5. Permanent Fix
Implement a **Fail-Open Architecture with Local In-Memory Fallback**:

```java
public boolean allowRequestWithFailOpen(String apiKey) {
    try {
        // Step 1: Attempt Distributed Redis Rate Limiting
        return redisRateLimiter.tryAcquire(apiKey);
    } catch (Exception ex) {
        // FAIL-OPEN: Log alert, fallback to local in-memory token bucket, and ALLOW traffic!
        log.warn("Distributed Redis rate limiter failed. Falling back to local in-memory limiter: {}", ex.getMessage());
        return localCaffeineRateLimiter.tryAcquire(apiKey);
    }
}
```

---

## 6. Postmortem Action Items
- [x] Convert non-security rate limiters to Fail-Open architecture across all ingress proxies.
- [x] Configure automated Chaos tests verifying API Gateway continues serving traffic during Redis outages.
