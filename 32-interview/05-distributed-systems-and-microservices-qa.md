# Senior Backend Interview Questions: Distributed Systems and Microservices

Comprehensive bank of senior-level Distributed Systems, Consensus Algorithms, and Microservice Architecture interview questions with mechanical, production-grade model answers.

---

### Q1: What is the PACELC theorem and how does it extend the classical CAP theorem?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **The CAP Theorem Limitation**:
   - CAP states that during a Network Partition (**P**), a distributed system must choose between Consistency (**C**) or Availability (**A**).
   - **The Flaw**: Networks are not partitioned $99.99\%$ of the time. What trade-offs does a system make during **Normal Steady-State Operation**?
2. **The PACELC Theorem Formulation**:
   - **If Partition (P)**: Choose between **Availability (A)** or **Consistency (C)**.
   - **Else (E - Normal Operation)**: Choose between **Latency (L)** or **Consistency (C)**.
3. **Real-World Examples**:
   - **PC/EC (e.g. Google Spanner / PostgreSQL)**: In a partition, chooses Consistency; in normal operation, pays network latency roundtrips to enforce Strict Consistency.
   - **PA/EL (e.g. Apache Cassandra / DynamoDB with eventual consistency)**: In a partition, chooses Availability; in normal operation, returns cached local replica reads immediately to deliver sub-millisecond Latency.
</details>

---

### Q2: How does the Raft consensus algorithm ensure that two leaders cannot be elected simultaneously (Split-Brain Prevention)?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Randomized Election Timeouts**:
   - Each follower node sets a randomized election timeout ($150\text{ms} - 300\text{ms}$).
   - This ensures one node times out first, increments the Term number, and broadcasts `RequestVote` RPCs before competitors, preventing vote splitting.
2. **Strict Quorum Calculus ($\lfloor N/2 \rfloor + 1$)**:
   - A candidate can only become Leader if it receives positive votes from a **Strict Majority (Quorum)** of cluster nodes.
   - On a 5-node cluster, Quorum is 3. It is mathematically impossible for two candidates to receive 3 votes each from 5 nodes.
3. **Log Completeness Rule (Election Restriction)**:
   - A follower will vote for a candidate **only if the candidate's log is at least as up-to-date as the follower's own log** (comparing last log term and index).
   - This guarantees that the elected leader is mathematically guaranteed to contain all committed log entries from prior terms.
</details>

---

### Q3: Why is physical NTP clock synchronization dangerous in distributed databases, and how do Google TrueTime and Vector Clocks solve clock skew?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **The NTP Clock Skew Danger**:
   - Network Time Protocol (NTP) clocks synchronize over network packets and frequently drift by $50 - 250\text{ms}$ or jump backwards during leap seconds.
   - If a database uses Last-Write-Wins (LWW) with physical timestamps, Server A's fast clock ($T=105$) will silently overwrite Server B's later causal write ($T=100$), causing **Silent Data Loss**.
2. **Google TrueTime (Hardware GPS & Atomic Clocks)**:
   - TrueTime exposes an API returning an uncertainty interval: $[t_{\text{earliest}}, t_{\text{latest}}]$ where uncertainty $\epsilon < 7\text{ms}$.
   - When committing a transaction, Spanner **waits for $2\epsilon$ time ($14\text{ms}$)** before releasing locks, guaranteeing that every subsequent transaction receives a strictly higher timestamp globally without coordination.
3. **Vector Clocks (Logical Causal Clocks)**:
   - Track causal event ordering via an array of monotonic integers: $V = [c_1, c_2, \dots, c_N]$.
   - Updates increment local counters, allowing systems (like Dynamo) to mathematically detect concurrent conflicting writes and trigger application reconciliation.
</details>

---

### Q4: How is a Twitter Snowflake 64-bit ID structured, and how does it guarantee sortability without central database coordination?
<details>
<summary>Reveal Answer</summary>

**Answer**:
```text
+-------------------------------------------------------------------------------+
| 1 Bit: Sign (0) | 41 Bits: Timestamp (ms) | 10 Bits: Machine ID | 12 Bits: Seq |
+-------------------------------------------------------------------------------+
```
1. **Bit Layout**:
   - **1 Bit**: Unused sign bit (always `0`).
   - **41 Bits**: Millisecond timestamp epoch offset ($\approx 69\text{ years}$ lifetime).
   - **10 Bits**: Machine / Worker ID (supports up to $1,024$ independent worker nodes).
   - **12 Bits**: Sequence counter ($0 - 4,095$ IDs per millisecond per worker).
2. **Key Properties**:
   - **Roughly Monotonically Sortable**: Because the leading 41 bits represent time, IDs naturally sort chronologically when indexed in B+Trees.
   - **High Throughput**: Generates over **4 million unique IDs/sec per node** purely in memory with zero network database lookups.
</details>

---

### Q5: What is Jeff Dean's "Hedged Requests" pattern and how does it solve Tail Latency Amplification in microservice fan-outs?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Tail Latency Amplification ($P(\text{Slow}) = 1 - (1-p)^N$)**:
   - When an API gateway fans out to 100 backend services in parallel, even if each service has a $p99 = 10\text{ms}$ and $1\%$ slow outliers ($1000\text{ms}$), the overall user request has a **$63.4\%$ probability of experiencing the slow tail latency**.
2. **Hedged Requests Mechanics**:
   - Send the primary request to Replica 1.
   - If no response is received by the **$p95$ latency threshold (e.g. $15\text{ms}$)**, immediately dispatch a **speculative second request** to Replica 2.
   - Process whichever response returns first, and send a cancellation to the slower replica.
   - **Result**: Cuts $p99$ tail latency from $1000\text{ms}$ to **$< 20\text{ms}$** while adding only $\approx 5\%$ extra network load!
</details>
