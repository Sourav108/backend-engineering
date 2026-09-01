# Incident 09: Cascading Microservice Failure (Unbounded RPC Chain)

---

## 1. Symptoms & Alert
- **Alert**: `Global HTTP 504 Gateway Timeout Rate > 40%`.
- **Customer Impact**: Total platform paralysis across API Gateway, Order Service, User Service, and Payment Service.

---

## 2. Metric & Telemetry Anomalies
- **OpenTelemetry Distributed Tracing**: Traces showed a 4-tier synchronous RPC dependency chain:
  $$\text{API Gateway} \longrightarrow \text{Order Service} \longrightarrow \text{User Service} \longrightarrow \text{Loyalty Points Service}$$
- **Thread Pools**: Tomcat request threads across all 4 services pinned at $100\%$ ($200/200$ busy threads).

---

## 3. Diagnostic Steps & Root Cause
1. The leaf service (`Loyalty Points Service`) experienced a database lock wait and began taking **15 seconds** per request.
2. The `User Service` called `Loyalty Points Service` using default unconfigured HTTP client settings with **infinite socket read timeouts**.
3. All 200 Tomcat worker threads in `User Service` blocked waiting for `Loyalty Points Service`.
4. Once `User Service` ran out of threads, all 200 Tomcat threads in `Order Service` blocked waiting for `User Service`.
5. The failure cascaded upstream all the way to the API Gateway (**Cascading Failure Wave**), bringing down the entire platform.

---

## 4. Immediate Mitigation
1. **Enable Circuit Breaker Fallback on API Gateway**:
   - Return cached/empty loyalty points response immediately (`HTTP 200` with fallback).
2. **Restart Upstream Microservice Pods** to clear saturated thread pools.

---

## 5. Permanent Fix
1. **Enforce Strict Timeout Budgets on All HTTP/gRPC Clients**:
   ```java
   ConnectionConfig connectionConfig = ConnectionConfig.custom()
           .setConnectTimeout(Timeout.ofMilliseconds(500))
           .setSocketTimeout(Timeout.ofMilliseconds(1000)) // 1-second hard ceiling!
           .build();
   ```
2. **Apply Resilience4j Circuit Breakers & Bulkheads**:
   ```yaml
   resilience4j.circuitbreaker:
     instances:
       loyaltyService:
         slidingWindowSize: 20
         failureRateThreshold: 50
         waitDurationInOpenState: 10s
   ```

---

## 6. Postmortem Action Items
- [x] Eliminate synchronous RPC chains: Convert loyalty point updates to asynchronous Kafka events.
- [x] Enforce automated CI lint rules prohibiting unconfigured `RestTemplate` / `HttpClient` beans.
