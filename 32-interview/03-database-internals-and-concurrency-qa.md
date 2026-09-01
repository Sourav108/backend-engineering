# Senior Backend Interview Questions: Database Internals, SQL, and Concurrency

Comprehensive bank of senior-level Database Internals, PostgreSQL MVCC, Indexing, and Transaction Isolation interview questions with mechanical, production-grade model answers.

---

### Q1: Why are B+Trees preferred over B-Trees and Binary Search Trees for relational database indexing?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **High Fan-Out & Shallow Tree Depth**:
   - A B+Tree has a large branching factor (e.g. 100 to 1,000 keys per 8KB page).
   - A tree indexing 100 million rows has a depth of only **3 or 4 levels**, requiring at most 3 or 4 disk I/O page reads to locate any arbitrary record. Binary Search Trees would require $> 26$ random disk I/O reads.
2. **Leaf Node Data Storage**:
   - In a B+Tree, internal routing nodes store **only search keys and page pointers**, leaving maximum space to maximize fan-out.
   - All actual data pointers and row payloads reside exclusively in the **Leaf Nodes**.
3. **Sequential Range Scans via Leaf Linked Lists**:
   - All leaf nodes are chained together in a doubly linked list on disk.
   - Executing range queries (`WHERE age BETWEEN 20 AND 30`) locates the first leaf node in $O(\log N)$ time and then performs fast sequential disk reads across the linked leaves without re-traversing the root.
</details>

---

### Q2: How does Multi-Version Concurrency Control (MVCC) work in PostgreSQL, and why does updating a row create dead tuple bloat?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **PostgreSQL MVCC Mechanics**:
   - PostgreSQL implements MVCC using two hidden system columns on every row: **`xmin`** (Transaction ID that created the row) and **`xmax`** (Transaction ID that deleted/superseded the row).
   - Readers never block writers, and writers never block readers. A query's snapshot sees a tuple only if `xmin` committed before the snapshot started and `xmax` has not committed.
2. **Dead Tuple Bloat on `UPDATE`**:
   - In PostgreSQL, an `UPDATE` **does not modify the row in-place**.
   - It performs an `INSERT` of a brand new row version with the updated data (setting new `xmin`), and sets the `xmax` of the old row to the current transaction ID.
   - The old row becomes a **Dead Tuple**.
   - If `VACUUM` is disabled or falls behind, dead tuples accumulate, bloating table disk size, saturating the RAM buffer pool with dead pages, and degrading sequential scan performance.
</details>

---

### Q3: What is the difference between Repeatable Read and Serializable isolation levels, and what is a "Write Skew" anomaly?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- **Repeatable Read**:
  - Guarantees that any row read during a transaction will return the exact same data throughout the transaction lifecycle.
  - Prevents Dirty Reads and Non-Repeatable Reads.
  - **Vulnerable to Write Skew**: Two concurrent transactions read intersecting state, decide to write to non-overlapping rows, and violate a global business invariant (e.g. Hospital on-call constraint: *"At least one doctor must be on call"*. Doctor A and Doctor B both read that 2 doctors are on call, and both concurrently update their own status to inactive. Both transactions commit, leaving zero doctors on call!).
- **Serializable**:
  - The strictest ACID isolation level.
  - Enforces **Serializable Snapshot Isolation (SSI)** by tracking read-write dependency locks (SIREAD locks) in memory.
  - If a potential Write Skew or serial order cycle is detected, one transaction is aborted with a `40001 serialization_failure` error, forcing the client to retry.
</details>

---

### Q4: How does an LSM-Tree (Log-Structured Merge-Tree) compare to a B+Tree in terms of write amplification and read latency?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- **LSM-Tree (Cassandra / RocksDB / DynamoDB)**:
  - **Write Path**: Writes are appended sequentially to an in-memory **MemTable** (SkipList) and Write-Ahead Log (WAL), then flushed to disk as immutable **SSTables**.
  - **Write Amplification**: Ultra-low write latency; sequentially writes data to disk without random seeks ($100,000+\text{ writes/sec}$).
  - **Read Amplification**: Higher read latency; reading requires searching MemTable, Bloom Filters, and multiple SSTable levels, resolved via background **Compaction**.
- **B+Tree (PostgreSQL / MySQL InnoDB)**:
  - **Write Path**: Updates modify 8KB pages in-place, requiring random disk page writes and write-ahead logging.
  - **Read Path**: Fast, deterministic point lookups in $O(\log N)$ with bounded page cache reads.
- **Rule**: Use B+Trees for read-heavy relational workloads; use LSM-Trees for ultra-high-throughput append-heavy workloads (logs, time-series, telemetry).
</details>

---

### Q5: What is the mathematical formula for sizing a database connection pool (HikariCP) and why does adding more connections reduce performance?
<details>
<summary>Reveal Answer</summary>

**Answer**:
$$\text{Optimal Pool Size} = (\text{CPU Cores} \times 2) + \text{Effective Spindle Count}$$
*(PostgreSQL / HikariCP benchmark standard developed by Oracle & PostgreSQL)*.
- **Why adding 500 connections degrades performance**:
  - A server with 16 CPU cores can physically execute only **16 threads simultaneously in hardware**.
  - If you configure 500 database connections, the Linux kernel scheduler spends $> 60\%$ of its CPU cycles executing **OS Context Switches** and managing page table invalidations, rather than executing actual SQL queries.
  - Sizing the pool tightly to $(\text{Cores} \times 2)$ ensures CPU cores operate at maximum cache locality with zero context-switching thrashing, delivering maximum transactions per second.
</details>

---

### Q6: How does `EXPLAIN (ANALYZE, BUFFERS)` help diagnose slow PostgreSQL queries?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- `EXPLAIN ANALYZE` physically executes the query and compares the query planner's **Estimated Costs** against **Actual Execution Times**.
- Adding `BUFFERS` reveals exact hardware I/O interaction:
  - `Shared Hit Blocks`: Pages read directly from RAM Buffer Pool (Microseconds).
  - `Shared Read Blocks`: Pages fetched from physical NVMe disk (Milliseconds).
  - `Shared Dirtied Blocks`: Pages modified in memory.
- **Diagnostic Clues**:
  - `Seq Scan on large_table` with `Shared Read: 45000` $\longrightarrow$ Missing index; table scan reading 360MB from disk.
  - High difference between `Rows Removed by Filter` vs `Rows Returned` $\longrightarrow$ Inefficient composite index ordering.
</details>
