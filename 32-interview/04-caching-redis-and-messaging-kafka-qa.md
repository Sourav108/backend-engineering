# Senior Backend Interview Questions: Caching, Redis, and Kafka Messaging

Comprehensive bank of senior-level Caching, Redis Architecture, and Apache Kafka Distributed Streaming interview questions with mechanical, production-grade model answers.

---

### Q1: Why is Redis single-threaded for core command execution, and how does it process 100,000+ requests/sec?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **The In-Memory Bottleneck**:
   - In-memory databases are bottlenecked by **Network I/O and Memory Bandwidth**, not CPU computation. Executing an in-memory `GET` or `SET` takes only a few nanoseconds.
2. **Eliminating Concurrency Overhead**:
   - A single-threaded execution model eliminates all thread synchronization mutexes, read-write locks, lock contention, and OS thread context switching.
3. **I/O Multiplexing via Linux `epoll`**:
   - Redis uses an asynchronous event loop based on Linux **`epoll` / `kqueue`**. A single thread monitors thousands of connected client TCP sockets simultaneously, processing commands sequentially as socket buffers receive data.
   - In Redis 6.0+, **Multi-Threaded I/O** is used exclusively for reading and writing network socket bytes, while command execution remains strictly single-threaded and lock-free.
</details>

---

### Q2: How do you differentiate and mitigate Cache Penetration, Cache Breakdown (Dogpiling), and Cache Avalanche?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Cache Penetration**:
   - **Problem**: Queries request non-existent keys (e.g. `userId = -9999`) that exist in neither Redis nor the database, causing every request to bypass cache and hit the database.
   - **Mitigation**:
     - **Bloom Filter**: Ingress layer checks Bloom filter before querying cache/DB.
     - **Null Value Caching**: Cache `null` with a short 60s TTL.
2. **Cache Breakdown (Hot Key Expiration / Dogpiling)**:
   - **Problem**: A single viral hot key (e.g. Flash sale product) expires, and 50,000 concurrent requests simultaneously miss the cache and hammer the database.
   - **Mitigation**: **Distributed Mutex (Singleflight / Redlock)**: Only 1 thread is permitted to query the database and repopulate the cache, while the other 49,999 threads wait.
3. **Cache Avalanche**:
   - **Problem**: Millions of keys are initialized with the exact same 1-hour TTL, expiring simultaneously at $T = 3600\text{s}$, dumping massive traffic onto the primary database.
   - **Mitigation**: **Randomized TTL Jitter**: Set $\text{TTL} = \text{BaseTTL} + \text{Random}(0, 300\text{s})$ to spread expirations smoothly across time.
</details>

---

### Q3: How does Apache Kafka achieve massive write and read throughput using Zero-Copy `sendfile()` and Sequential Append-Only Logs?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Sequential Disk I/O**:
   - Kafka writes messages strictly to the end of an **Append-Only Commit Log segment file** on disk.
   - Sequential disk writes achieve $> 600\text{MB/sec}$ on standard disks (matching raw memory speed), completely avoiding slow random disk head seeks.
2. **OS Page Cache Utilization**:
   - Kafka does not maintain large heap caches; it delegates caching directly to the Linux kernel **Page Cache**. All unallocated server RAM acts as a massive message cache.
3. **Zero-Copy Network Transfer (`sendfile()` Syscall)**:
   - In standard Java I/O: Data is copied: Disk $\to$ OS Page Cache $\to$ JVM Heap Buffer $\to$ OS Socket Buffer $\to$ Network NIC (**4 context switches, 4 data copies**).
   - In Kafka: Uses the Linux `sendfile()` system call (via Java `FileChannel.transferTo()`). Data is DMA-transferred directly from **OS Page Cache to the Network Card NIC buffer** with **zero CPU copies and zero JVM memory pollution**.
</details>

---

### Q4: How does Kafka achieve Exactly-Once Semantics (EOS) across producer publishing and consumer processing?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Idempotent Producer (`enable.idempotence=true`)**:
   - The broker assigns each producer a unique 64-bit **Producer ID (PID)**.
   - The producer stamps each message with a strictly incrementing **Sequence Number**.
   - If a network ACK drops and the producer retries, the broker detects the duplicate sequence number and discards it with zero duplicates saved to the log.
2. **Transactional Coordinator & 2-Phase Commit over Kafka Logs**:
   - When consuming from Topic A, processing in Spring Boot, and producing to Topic B:
   - The producer coordinates with Kafka's **Transaction Coordinator**.
   - The transaction state is written to a special internal topic `__transaction_state`.
   - When `commitTransaction()` is called, Kafka writes a two-phase commit marker to the output topic partition, and consumer offsets are committed atomically as part of the same transaction.
</details>

---

### Q5: What is the difference between `EagerRebalanceProtocol` and `CooperativeStickyAssignor` in Kafka Consumer Groups?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- **Eager Rebalance Protocol (Legacy / `RangeAssignor`)**:
  - When a consumer joins or leaves the group, **all consumers in the group revoke all assigned partitions simultaneously (Stop-The-World pause)**.
  - No messages are processed across the entire group until the coordinator recalculates partition assignments, causing lag spikes.
- **Cooperative Sticky Assignor (Modern / `CooperativeStickyAssignor`)**:
  - Implements **Incremental Cooperative Rebalancing**.
  - Consumers continue processing unaffected partitions without interruption.
  - Only the specific partitions migrating between consumers are temporarily revoked, eliminating cluster-wide Stop-The-World consumer pauses.
</details>
