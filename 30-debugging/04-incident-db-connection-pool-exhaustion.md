# Incident 04: Database Connection Pool Exhaustion (HikariCP Starvation)

---

## 1. Symptoms & Alert
- **Alert**: `HikariPool-1 - Connection is not available, request timed out after 30000ms.`
- **Customer Impact**: Total outage across checkout and inventory endpoints; $100\%$ of new API requests failing with HTTP 500 / 504 errors.

---

## 2. Metric & Telemetry Anomalies
- **HikariCP Metrics**: `hikaricp.connections.active` flatlined at $32/32$ (max capacity); `hikaricp.connections.pending` spiked to $> 450$ waiting threads.
- **Database Metrics**: PostgreSQL CPU sat at $< 15\%$ (the database was not CPU-bound; connections were simply held open idling inside long transactions).

---

## 3. Diagnostic Steps

### Step 1: Query Active PostgreSQL Connections & Long-Running Queries
```sql
SELECT pid, age(clock_timestamp(), query_start), state, query 
FROM pg_stat_activity 
WHERE state != 'idle' 
ORDER BY query_start ASC 
LIMIT 10;
```
- *Output*: 32 connections stuck in state `idle in transaction` for over 15 minutes!

### Step 2: Correlate Database PID with Java Stack Traces
Inspecting the application code revealed an external third-party HTTP call executing **inside an open Spring `@Transactional` boundary**:

```java
@Transactional
public void processCheckout(OrderRequest request) {
    Order order = orderRepository.save(new Order(request)); // Acquires DB connection lease!
    
    // DISASTER: 30-SECOND BLOCKING THIRD-PARTY HTTP CALL INSIDE DATABASE TRANSACTION!
    FraudCheckResult fraud = fraudApiClient.checkFraud(request.getUserId()); 
    
    order.setApproved(fraud.isApproved());
    orderRepository.save(order);
}
```

---

## 4. Root Cause Analysis
- The third-party Fraud API experienced an outage and began hanging for $30\text{ seconds}$ per call.
- Because `fraudApiClient.checkFraud()` was placed inside `@Transactional`, the application thread **held the physical PostgreSQL JDBC connection lease open and idle for 30 seconds**.
- With just 32 concurrent checkouts, all 32 HikariCP connections were completely exhausted, blocking all other application endpoints from acquiring database connections.

---

## 5. Immediate Mitigation
1. **Terminate Idle Database Transactions**:
   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle in transaction' AND query_start < NOW() - INTERVAL '2 minutes';
   ```
2. **Apply Circuit Breaker to Third-Party Fraud API** to fail fast in $< 500\text{ms}$.

---

## 6. Permanent Fix
Move all network and third-party API calls **OUTSIDE of `@Transactional` database boundaries**:

```java
// STEP 1: Execute external network I/O FIRST (0 DB connections held!)
FraudCheckResult fraud = fraudApiClient.checkFraud(request.getUserId());

// STEP 2: Execute short, sub-millisecond database transaction SECOND
orderService.saveApprovedOrder(request, fraud.isApproved());
```

---

## 7. Postmortem Action Items
- [x] Configure HikariCP `leakDetectionThreshold: 3000` to automatically log stack traces of any connection held $> 3\text{ seconds}$.
- [x] Set PostgreSQL `idle_in_transaction_session_timeout = '10s'` to automatically terminate hung client transactions.
