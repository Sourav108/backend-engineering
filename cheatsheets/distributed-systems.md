# Distributed Systems & Consensus Cheat Sheet

---

## ⚡ 1. The CAP & PACELC Theorems

$$\textbf{PACELC Theorem: } \text{If Partition (P) } \longrightarrow \text{Choose (A) or (C)}; \quad \text{Else (E) } \longrightarrow \text{Choose (L) or (C)}$$

| System | Classification | Behavior in Partition | Normal Behavior |
|---|---|---|---|
| **PostgreSQL / Spanner** | **PC / EC** | Consistency over Availability | Consistency over Latency |
| **Cassandra / DynamoDB** | **PA / EL** | Availability over Consistency | Sub-ms Latency over Consistency |
| **MongoDB** | **PC / EC** | Consistency over Availability | Consistency over Latency |

---

## ⚡ 2. Quorum Calculus & Raft Consensus

$$\textbf{Quorum Rule: } W + R > N \quad \text{and} \quad W > \frac{N}{2}$$

- **Raft Election**: Leader elected by strict majority ($\lfloor N/2 \rfloor + 1$).
- **Randomized Timeouts**: $150 - 300\text{ms}$ election timeout prevents vote splitting.
- **Log Replication**: Committed once replicated to a quorum of nodes.

---

## ⚡ 3. Twitter Snowflake 64-Bit ID Layout

```text
+-----------------------------------------------------------------------------------+
| 1 Bit: Sign (0) | 41 Bits: Timestamp (ms) | 10 Bits: Machine ID | 12 Bits: Seq ID |
+-----------------------------------------------------------------------------------+
```
- **Throughput**: $4,096\text{ IDs/ms/node}$ ($4.09\text{M IDs/sec}$).
- **Sortability**: Chronologically sortable via leading timestamp bits.

---

## ⚡ 4. Consistent Hashing with VNodes

$$\text{Migrated Keys on Cluster Resize } (N \to N+1) = \frac{K}{N+1}$$

- **VNodes**: 150–256 virtual nodes per physical server guarantee $< 2\%$ variance across cluster partitions.
