# Backend Engineering Curriculum Dependency Graph

Visualizing the prerequisite relationships, architectural bridges, and progressive mastery paths across all 32 curriculum modules.

---

## 🗺️ Architectural Mastery Graph

```mermaid
flowchart TD
    subgraph Foundation["Tier 1: Network & Protocol Foundations"]
        MOD01["01: Backend Fundamentals"] --> MOD02["02: HTTP & APIs"]
        MOD02 --> MOD03["03: Flagship Request Lifecycle"]
        MOD02 --> MOD04["04: Networking (DNS/TCP/TLS)"]
        MOD02 --> MOD05["05: API Design & Idempotency"]
        MOD05 --> MOD06["06: Auth & Security (JWT/OAuth2)"]
    end

    subgraph DataTier["Tier 2: Persistence, Transactions & Caching"]
        MOD05 --> MOD07["07: Databases (SQL/NoSQL)"]
        MOD07 --> MOD08["08: Database Internals (B+Tree/WAL/MVCC)"]
        MOD08 --> MOD09["09: Transactions & Consistency (ACID/Locks)"]
        MOD07 --> MOD10["10: Caching & Redis Mechanics"]
    end

    subgraph AsyncTier["Tier 3: Asynchronous & Distributed Systems"]
        MOD09 & MOD10 --> MOD11["11: Messaging & Apache Kafka"]
        MOD11 --> MOD12["12: Event-Driven Architecture (Outbox/CDC)"]
        MOD11 --> MOD13["13: Distributed Systems (CAP/PACELC/Clocks)"]
    end

    subgraph ConcurrencyTier["Tier 4: Concurrency & Resilience"]
        MOD13 --> MOD14["14: Resilience (Circuit Breaker/Bulkheads)"]
        MOD03 --> MOD15["15: Concurrency & Backpressure (Virtual Threads)"]
        MOD14 & MOD15 --> MOD19["19: Microservices Architecture"]
        MOD19 --> MOD20["20: Service-to-Service Communication (gRPC/REST)"]
        MOD11 --> MOD21["21: Background Jobs & Workers"]
    end

    subgraph OperationsTier["Tier 5: Observability, Scale & Production"]
        MOD19 --> MOD16["16: Observability (Logs/Metrics/Traces)"]
        MOD15 --> MOD17["17: Performance & p99 Optimization"]
        MOD17 --> MOD18["18: Scalability (Sharding/Replication)"]
        MOD19 --> MOD25["25: Cloud Backend (AWS Core)"]
        MOD25 --> MOD26["26: Containers & Kubernetes"]
        MOD26 --> MOD27["27: Production Engineering & Zero-Downtime"]
        MOD19 --> MOD28["28: Testing (Unit & Testcontainers)"]
    end

    subgraph CapstoneTier["Tier 6: Specialized Capabilities, Projects & Interviews"]
        MOD21 --> MOD22["22: File & Object Storage (S3)"]
        MOD07 --> MOD23["23: Search & Inverted Indexes"]
        MOD02 & MOD15 --> MOD24["24: Real-Time Systems (WebSocket/SSE)"]

        OperationsTier --> MOD29["29: Runnable Backend Projects"]
        OperationsTier --> MOD30["30: Production Debugging Scenarios"]
        OperationsTier --> MOD31["31: System Design Bridge"]
        OperationsTier --> MOD32["32: SDE2 Interview Preparation"]
    end
```
