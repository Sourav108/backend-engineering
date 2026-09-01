# Optimistic vs Pessimistic Locking and SKIP LOCKED Queues

---

## 1. What Is It?
**Concurrency Locking Strategies** prevent concurrent write transactions from causing the **Lost Update Anomaly** and corrupting shared domain state:
- **Pessimistic Locking**: Assumes conflicts *will* happen. Explicitly acquires exclusive row-level database locks (`SELECT ... FOR UPDATE`) at the start of a transaction, blocking all other transactions until the lock holder commits or rolls back.
- **Optimistic Locking**: Assumes conflicts are *rare*. Allows concurrent transactions to read and modify entities in memory without database locks. Upon commit, it verifies that no other transaction has modified the entity's version number (`WHERE version = ?`). If a conflict is detected, the transaction aborts and retries.

---

## 2. Why Does It Exist?
In modern multi-threaded application servers handling thousands of concurrent requests:
- Two users attempt to book the last concert seat simultaneously.
- Two background workers attempt to process the exact same payment job.
- A user submits a form twice in rapid succession.

Without locking, both threads read the same initial state and overwrite each other, causing double spending or duplicate processing.

---

## 3. Mental Model

```mermaid
sequenceDiagram
    autonumber
    actor ClientA as Thread A (Pessimistic)
    actor ClientB as Thread B (Pessimistic)
    participant DB as Database (Row Lock)

    ClientA->>DB: SELECT * FROM seats WHERE id = 1 FOR UPDATE
    DB-->>ClientA: Row returned (Exclusive Lock Held by A)
    
    ClientB->>DB: SELECT * FROM seats WHERE id = 1 FOR UPDATE
    Note over ClientB,DB: Thread B is BLOCKED and waits in DB Lock Manager queue!
    
    ClientA->>DB: UPDATE seats SET booked = true WHERE id = 1; COMMIT;
    DB-->>ClientA: Success (Lock Released)
    
    DB-->>ClientB: Row returned to Thread B (Sees booked = true!)
    ClientB->>ClientB: Rejects booking (Seat already taken)
```

---

## 4. How Does It Work?

### Pessimistic Locking Modes
1. **`SELECT ... FOR UPDATE`**: Acquires an Exclusive (`X`) row lock. Blocks all other `FOR UPDATE`, `FOR SHARE`, `UPDATE`, and `DELETE` operations on those rows.
2. **`SELECT ... FOR UPDATE NOWAIT`**: Attempts to acquire the lock; if another transaction holds it, **fails immediately** with `ERROR: could not obtain lock on row` instead of blocking.
3. **`SELECT ... FOR UPDATE SKIP LOCKED`**: **Skips all locked rows** and returns only available unlocked rows. **The ultimate building block for high-throughput distributed task queues in relational databases!**

---

### Optimistic Locking Mechanics (`@Version`)
1. Table includes a dedicated `version BIGINT NOT NULL DEFAULT 0` column.
2. Thread reads entity ($id = 42, version = 5$).
3. Thread modifies entity in memory.
4. During commit, Hibernate/JDBC executes:
   ```sql
   UPDATE products 
   SET stock = stock - 1, version = version + 1 
   WHERE id = 42 AND version = 5;
   ```
5. If another transaction modified the row first ($version$ is now $6$), the `UPDATE` affects **0 rows**.
6. The application detects `0 rows updated` and throws `OptimisticLockingFailureException`.

---

## 5. Implementation: Distributed Job Worker with `SKIP LOCKED`

```mermaid
flowchart TD
    subgraph JobTable["Database Job Table (1,000 PENDING Jobs)"]
        J1["Job 1 (Locked by Worker A)"]
        J2["Job 2 (Locked by Worker B)"]
        J3["Job 3 (Available)"]
        J4["Job 4 (Available)"]
    end

    WorkerA["Worker Thread A"] -->|1. SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1| J1
    WorkerB["Worker Thread B"] -->|2. SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1| J2
    WorkerC["Worker Thread C"] -->|3. SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1| J3

    Note over WorkerC: Worker C skips J1 & J2 instantly without blocking!
```

### High-Throughput Java 21 Task Processor with `SKIP LOCKED`
```java
package com.backend.engineering.transactions.queue;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.Optional;

@Service
public class DistributedTaskQueue {

    private final JdbcTemplate jdbcTemplate;

    public DistributedTaskQueue(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public record TaskRecord(Long id, String taskType, String payload) {}

    // Polled by 50 parallel worker threads across multiple Kubernetes pods
    @Transactional
    public Optional<TaskRecord> acquireNextJob() {
        String sql = """
            SELECT id, task_type, payload
            FROM background_tasks
            WHERE status = 'PENDING'
            ORDER BY priority DESC, created_at ASC
            LIMIT 1
            FOR UPDATE SKIP LOCKED
            """;

        return jdbcTemplate.query(sql, (rs, rowNum) -> {
            long id = rs.getLong("id");
            String type = rs.getString("task_type");
            String payload = rs.getString("payload");

            // Mark job in-flight within the same transaction lock boundary
            jdbcTemplate.update(
                "UPDATE background_tasks SET status = 'PROCESSING', started_at = ? WHERE id = ?",
                Instant.now(), id
            );

            return new TaskRecord(id, type, payload);
        }).stream().findFirst();
    }
}
```

---

## 6. Implementation: Optimistic Locking with Retry Loop

### Spring Data JPA Entity with `@Version`
```java
package com.backend.engineering.transactions.model;

import jakarta.persistence.*;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Integer stockCount;

    @Version // JPA Automated Optimistic Locking Field
    @Column(nullable = false)
    private Long version;

    public void decrementStock(int quantity) {
        if (this.stockCount < quantity) {
            throw new IllegalStateException("Insufficient stock for product: " + id);
        }
        this.stockCount -= quantity;
    }

    // Getters and Constructors
    protected Product() {}
    public Product(String name, Integer stockCount) {
        this.name = name;
        this.stockCount = stockCount;
    }

    public Long getId() { return id; }
    public Integer getStockCount() { return stockCount; }
    public Long getVersion() { return version; }
}
```

---

## 7. Performance & Contention Comparison

| Concurrency Level | Workload Characteristic | Best Strategy | Reason |
|---|---|---|---|
| **Low Contention ($< 1\%$ conflict)** | User profile edits, blog posts | **Optimistic Locking** | Zero DB lock manager overhead; zero thread blocking |
| **High Contention ($> 20\%$ conflict)** | Flash sale inventory, seat booking | **Pessimistic Locking** | Optimistic retry loops thrash CPU and fail repeatedly |
| **Queue Processing / Task Claiming** | 100 workers claiming jobs from 1 table | **`FOR UPDATE SKIP LOCKED`** | Eliminates lock contention; 100% linear worker scaling |

---

## 8. Failure Scenarios

1. **Optimistic Retry Storms under High Contention**:
   - *Failure*: 1,000 concurrent users try to buy the last 5 iPhones in a flash sale. Optimistic locking causes 995 transactions to fail on commit, trigger retries simultaneously, and repeatedly fail, pinning application CPU at $100\%$ with zero progress.
   - *Mitigation*: Switch to **Pessimistic Locking** or **Atomic SQL Updates** (`UPDATE products SET stock = stock - 1 WHERE id = 1 AND stock > 0`).

2. **Deadlocks via Unordered `SELECT FOR UPDATE`**:
   - *Failure*: Thread A locks Product 1 then Product 2; Thread B locks Product 2 then Product 1.
   - *Mitigation*: Always sort entity IDs before acquiring pessimistic locks.

---

## 9. Observability

- **Metrics**:
  - `hikaricp_pending_threads`: Spike indicates threads blocked waiting on pessimistic database locks.
  - `application_optimistic_lock_retries_total`: Counter tracking optimistic collision frequency.

---

## 10. Debugging

### Inspecting Blocked Rows in PostgreSQL
```sql
SELECT 
    l.pid,
    sa.query,
    l.mode,
    l.granted,
    now() - sa.query_start AS waiting_duration
FROM pg_locks l
JOIN pg_stat_activity sa ON sa.pid = l.pid
WHERE NOT l.granted;
```

---

## 11. Scaling

### Atomic SQL Decrements (The Zero-Lock Hybrid)
For simple counter and inventory updates, avoid both ORM optimistic and pessimistic locks:
```sql
UPDATE products 
SET stock_count = stock_count - 1 
WHERE id = :productId AND stock_count >= 1;
```
- Row is locked **only for the microseconds of statement execution**.
- If the statement returns `0 rows affected`, the stock was exhausted. Highly scalable ($> 50,000\text{ updates/sec}$).

---

## 12. Trade-offs

| Dimension | Optimistic Locking | Pessimistic Locking (`FOR UPDATE`) | `FOR UPDATE SKIP LOCKED` |
|---|---|---|---|
| **DB Lock Overhead** | Zero | High (Holds Lock Manager entries) | Moderate |
| **Thread Blocking** | Never blocks (Fails on commit) | Blocks until lock holder commits | Never blocks (Skips locked rows) |
| **Best Scenario** | High read, low write conflict | High contention single-resource lock | Distributed background workers |

---

## 13. When to Use
- **Optimistic Locking**: Standard web applications, user profile updates, shopping cart additions.
- **Pessimistic Locking**: High-contention balance transfers, strict FIFO reservations.
- **`SKIP LOCKED`**: Distributed task workers claiming jobs from relational database tables.

---

## 14. When NOT to Use
- Do not use Pessimistic Locking across human interaction boundaries (e.g. holding a database lock while waiting for a user to enter an SMS OTP).

---

## 15. Interview Questions

### Q1: How does `SELECT ... FOR UPDATE SKIP LOCKED` enable building high-performance task queues inside PostgreSQL/MySQL?
<details>
<summary>Reveal Answer</summary>

**Answer**:
In traditional relational queue implementations:
Multiple worker threads executing `SELECT ... FOR UPDATE LIMIT 1` all attempt to acquire a lock on the exact same top pending row (Job 1). Worker 1 acquires the lock, while Workers 2 through 10 **block and wait**. When Worker 1 commits, Worker 2 acquires the lock, realizes the job is already processed, and has to query again. This causes severe lock convoying and zero parallel processing.
With **`SKIP LOCKED`**:
Worker 1 locks Job 1. When Worker 2 executes `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1`, the database engine **instantly skips Job 1** (because it is locked) and immediately locks and returns **Job 2** with zero waiting. 50 worker threads can claim 50 distinct jobs in parallel in $O(1)$ time with zero blocking.
</details>

### Q2: What is the ABA problem in Optimistic Locking and how is it prevented?
<details>
<summary>Reveal Answer</summary>

**Answer**:
The **ABA problem** occurs if an optimistic locking scheme relies on data values rather than a dedicated monotonic version counter:
1. Thread 1 reads Account balance = $\$100$.
2. Thread 2 modifies balance to $\$200$ and commits.
3. Thread 3 modifies balance back to $\$100$ and commits.
4. Thread 1 attempts to commit with `WHERE balance = 100`. The check passes even though the account was modified twice in the interim!
**Prevention**:
Use a dedicated monotonically increasing **`version BIGINT`** counter or unique UUID token. Even if data values revert to their previous states, the `version` field advances monotonically ($5 \to 6 \to 7$), guaranteeing that intermediate mutations are always detected.
</details>

---

## 16. Practical Exercise
1. Create a `products` table with an integer `version` column.
2. In Spring Boot, launch 20 concurrent threads trying to decrement stock from a product starting with 10 units using `@Version`.
3. Observe the `OptimisticLockingFailureException` thrown and implement a retry mechanism.
4. Build a 5-worker background job consumer using `SELECT ... FOR UPDATE SKIP LOCKED` and verify that all workers process distinct jobs simultaneously with zero lock contention.

---

## 17. Quick Revision
- **Pessimistic**: Locks upfront via `SELECT FOR UPDATE`; prevents conflicts by blocking competitors.
- **Optimistic**: Detects conflicts on commit via `version` column; best for low-contention reads.
- **`NOWAIT`**: Fails fast if lock is held.
- **`SKIP LOCKED`**: Skips locked rows; standard pattern for relational DB task queues.
- **Atomic SQL Updates**: `UPDATE ... WHERE balance >= amount` provides highest throughput for simple numeric deltas.
