# Incident 06: Cache Stampede and Hot Key Dogpiling

---

## 1. Symptoms & Alert
- **Alert**: `PostgreSQL Connection Count Saturation > 95%`.
- **Customer Impact**: Homepage and product catalog API latency exploded from $10\text{ms}$ to $> 8,000\text{ms}$; database connection queue backed up.

---

## 2. Metric & Telemetry Anomalies
- **Redis Cache Hit Ratio**: Dropped from $99.5\%$ to $0\%$ on key `product:black_friday_deal_101`.
- **Database QPS**: Single query `SELECT * FROM products WHERE id = 101` surged from $0.5\text{ QPS}$ to **$32,000\text{ QPS}$** in less than 2 seconds.

---

## 3. Diagnostic Steps & Root Cause
- The marketing team launched a flash sale on Product 101.
- The Redis cache key `product:black_friday_deal_101` was stored with a fixed 1-hour TTL.
- At $T = 3600\text{s}$, the key expired.
- Because 32,000 concurrent user requests were in flight at that exact millisecond, **all 32,000 requests suffered a simultaneous cache miss**.
- Every application thread attempted to query the database and recompute the cache simultaneously (**Cache Stampede / Dogpiling**), instantly saturating the primary database connection pool.

---

## 4. Immediate Mitigation
1. **Manually Pre-Warm Hot Cache Key with High TTL**:
   ```bash
   redis-cli -h redis.internal SET product:black_friday_deal_101 '{"id":101,"name":"Deal"}' EX 86400
   ```
2. **Apply Read-Your-Own-Writes Rate Limiting on Ingress**.

---

## 5. Permanent Fix
Implement the **Singleflight / Distributed Mutex Pattern** with Probabilistic Early Expiration (XFetch algorithm) in Spring Boot:

```java
public ProductDto getProductWithStampedeDefense(Long productId) {
    String cacheKey = "product:" + productId;
    String cachedJson = redisTemplate.opsForValue().get(cacheKey);

    if (cachedJson != null) {
        return deserialize(cachedJson);
    }

    // CACHE MISS: Acquire Distributed Mutex (Singleflight)
    String lockKey = "lock:product:" + productId;
    Boolean acquired = redisTemplate.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

    if (Boolean.TRUE.equals(acquired)) {
        try {
            // ONLY ONE THREAD QUERIES THE DATABASE!
            ProductDto product = productRepository.findDtoById(productId);
            redisTemplate.opsForValue().set(cacheKey, serialize(product), Duration.ofHours(2));
            return product;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // Other 31,999 threads sleep briefly and read from populated cache!
        try { Thread.sleep(50); } catch (InterruptedException ignored) {}
        return getProductWithStampedeDefense(productId);
    }
}
```

---

## 6. Postmortem Action Items
- [x] Configure Caffeine L1 in-memory caching to absorb hot key misses before hitting Redis.
- [x] Apply randomized jitter to all cache TTLs ($\text{TTL} \pm 10\%$).
