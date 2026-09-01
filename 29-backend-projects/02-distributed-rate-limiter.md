# Project 02: Distributed Multi-Algorithm Rate Limiter

Build a production-grade distributed rate limiting system supporting 4 distinct rate limiting algorithms (Token Bucket, Leaky Bucket, Sliding Window Log, Sliding Window Counter) implemented via atomic Redis Lua scripts with Spring Boot 3.3.4.

---

## 🗺️ Rate Limiting Algorithms Matrix

| Algorithm | Burst Handling | Memory Footprint | Accuracy | Best Use Case |
|---|---|:---:|:---:|---|
| **Token Bucket** | ✅ Allows controlled burst | **$O(1)$ (Tiny)** | High | General API Gateway rate limiting |
| **Leaky Bucket** | ❌ Smooths traffic strictly | $O(1)$ | High | Downstream database write smoothing |
| **Sliding Window Log** | ❌ Strictly bounded | $O(N)$ (Higher) | **$100\%$ Exact** | High-security financial endpoints |
| **Sliding Window Counter**| ✅ Approximate | **$O(1)$** | $\approx 99.5\%$ | High-scale multi-tenant ingress |

---

## ⚡ Implementation: Sliding Window Log in Redis Lua

```lua
-- sliding_window_log.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local windowMs = tonumber(ARGV[2])
local maxRequests = tonumber(ARGV[3])

local clearBefore = now - windowMs

-- 1. Remove all requests older than the sliding window
redis.call('ZREMRANGEBYSCORE', key, 0, clearBefore)

-- 2. Count requests remaining in current window
local currentCount = redis.call('ZCARD', key)

if currentCount < maxRequests then
    -- 3. Add current timestamp as both score and member
    redis.call('ZADD', key, now, now .. '-' .. redis.call('INCR', key .. ':seq'))
    redis.call('EXPIRE', key, math.ceil(windowMs / 1000))
    return 1 -- ALLOWED
else
    return 0 -- REJECTED (HTTP 429)
end
```

---

## ⚡ Implementation: Spring Boot Rate Limiting Interceptor

```java
package com.backend.engineering.projects.ratelimiter;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import java.util.Collections;
import java.util.List;

@Component
public class DistributedRateLimiterInterceptor implements HandlerInterceptor {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> slidingWindowScript;

    public DistributedRateLimiterInterceptor(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.slidingWindowScript = new DefaultRedisScript<>();
        this.slidingWindowScript.setLocation(new ClassPathResource("scripts/sliding_window_log.lua"));
        this.slidingWindowScript.setResultType(Long.class);
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String apiKey = request.getHeader("X-API-KEY");
        if (apiKey == null) {
            apiKey = request.getRemoteAddr();
        }

        String rateLimitKey = "ratelimit:" + apiKey;
        long now = System.currentTimeMillis();
        long windowMs = 60_000L; // 1-minute window
        long limit = 100L;       // 100 requests per minute

        List<String> keys = Collections.singletonList(rateLimitKey);
        Long allowed = redisTemplate.execute(slidingWindowScript, keys, String.valueOf(now), String.valueOf(windowMs), String.valueOf(limit));

        if (allowed != null && allowed == 1L) {
            response.setHeader("X-RateLimit-Limit", String.valueOf(limit));
            return true;
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("Retry-After", "60");
            response.setContentType("application/json");
            response.getWriter().write("{\"error\": \"Too Many Requests\", \"message\": \"Rate limit exceeded. Try again in 60s.\"}");
            return false;
        }
    }
}
```
