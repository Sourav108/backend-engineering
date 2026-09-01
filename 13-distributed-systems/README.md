# Module 13: Distributed Systems for Backend Engineers

Master the foundational theory and service-level engineering patterns of distributed systems: CAP and PACELC trade-offs, Raft consensus mechanics, quorum calculus, physical clock skew hazards, Vector Clocks, Twitter Snowflake 64-bit ID generation, and SWIM gossip failure detection.

---

## 🗺️ Master Distributed Consensus & Reliability Architecture

```mermaid
flowchart TD
    subgraph ConsensusTier["1. Consensus & Leadership (Raft Mode)"]
        Leader["Elected Leader Node (Term 2)"] <-->|AppendEntries (50ms)| F1["Follower 1"]
        Leader <-->|AppendEntries (50ms)| F2["Follower 2"]
        Note over Leader,F2: Majority Quorum Q = floor(N/2) + 1 prevents Split-Brain
    end

    subgraph ClockAndIDTier["2. Time, Ordering & Unique IDs"]
        Snowflake["Twitter Snowflake Generator (64-bit long)"]
        HLC["Hybrid Logical Clock (HLC / Vector Clocks)"]
    end

    subgraph ClusterMembership["3. Decentralized Health (SWIM Gossip)"]
        NodeA["Node A"] <-->|Random UDP Gossip (O(log N))| NodeB["Node B"]
        NodeB <-->|Indirect Ping-Req| NodeC["Node C"]
    end
```

---

## 📚 Curriculum Lessons

| # | Lesson | Core Focus & Mechanics |
|:---:|---|---|
| **01** | [`01-cap-theorem-and-pacelc-trade-offs.md`](./01-cap-theorem-and-pacelc-trade-offs.md) | CAP theorem (CP vs AP), the PACELC model (Latency vs Consistency in steady state), and Quorum calculus ($W + R > N$). |
| **02** | [`02-consensus-protocols-raft-and-leader-election.md`](./02-consensus-protocols-raft-and-leader-election.md) | Raft state transitions (Follower, Candidate, Leader), randomized election timeouts ($150-300\text{ms}$), and odd-node quorum math. |
| **03** | [`03-clocks-clock-skew-and-ordering.md`](./03-clocks-clock-skew-and-ordering.md) | Physical NTP clock drift, Last-Write-Wins (LWW) data loss, `System.nanoTime()`, Vector Clocks, and Google TrueTime commit-wait. |
| **04** | [`04-distributed-unique-id-generators.md`](./04-distributed-unique-id-generators.md) | Twitter Snowflake (41-bit timestamp + 10-bit worker + 12-bit sequence), UUIDv7, avoiding B+Tree page split fragmentation, and clock skew defense. |
| **05** | [`05-failure-detection-heartbeats-and-gossip.md`](./05-failure-detection-heartbeats-and-gossip.md) | $\Phi$ Accrual Failure Detector, decentralized SWIM gossip protocol, and indirect peer probing (`Ping-Req`). |

---

## ⚡ Key Production Takeaways

1. **Quorum Rule**: Always maintain $W + R > N$ when linearizable consistency is required in distributed datastores.
2. **Odd Node Rule**: Deploy consensus clusters with 3, 5, or 7 nodes; even node counts increase communication without adding fault tolerance.
3. **Monotonic Clocks**: Never use `System.currentTimeMillis()` to measure execution duration; always use `System.nanoTime()`.
4. **Time-Ordered IDs**: Use Snowflake or UUIDv7 instead of UUIDv4 to eliminate B+Tree index page split thrashing.
5. **SWIM Indirect Probes**: Use peer proxy pings to prevent false failure declarations caused by single-link network packet drops.
