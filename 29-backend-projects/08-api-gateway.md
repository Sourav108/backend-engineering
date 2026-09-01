# Project 08: High-Performance API Gateway

Build a high-performance API Gateway supporting dynamic routing, JWT token validation, rate limiting, header transformation, and Resilience4j circuit breakers using Spring Cloud Gateway and Netty non-blocking event loops.

---

## 🗺️ System Architecture

```mermaid
flowchart LR
    Client["Public Client"] -->|HTTPS / TLS| Gateway["API Gateway (Port 443)"]
    
    subgraph GatewayFilters["Non-Blocking Filter Pipeline"]
        F1["1. RateLimiterFilter (Redis Token Bucket)"]
        F2["2. JwtAuthenticationFilter (Cryptographic Signature Check)"]
        F3["3. HeaderTransformationFilter (Inject X-User-Id)"]
        F4["4. CircuitBreakerFilter (Resilience4j Fallback)"]
        F1 --> F2 --> F3 --> F4
    end

    Gateway --> GatewayFilters
    F4 -->|HTTP/2 Routing| OrderSvc["Order Service (:8081)"]
    F4 -->|HTTP/2 Routing| UserSvc["User Service (:8082)"]
```

---

## ⚡ Implementation: Spring Cloud Gateway Route Configuration

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Route 1: Orders Microservice
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
            - name: CircuitBreaker
              args:
                name: orderCircuitBreaker
                fallbackUri: forward:/fallback/orders
            - AddRequestHeader=X-Gateway-Processed, true

        # Route 2: Users Microservice
        - id: user-service-route
          uri: lb://user-service
          predicates:
            - Path=/api/v1/users/**
```
