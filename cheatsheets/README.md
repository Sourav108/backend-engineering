# Backend Engineering Quick-Reference Cheat Sheets

High-density, production-tested quick reference guides and mathematical cheat sheets for senior backend engineers, distributed systems architects, and interview preparation.

---

## 📚 Master Cheat Sheet Index

| Cheat Sheet | File | Core Content & Formulas |
|---|---|---|
| **Java Concurrency & JVM** | [`java-concurrency.md`](./java-concurrency.md) | JMM happens-before, `volatile`, CAS atomics, `LongAdder`, Virtual Threads (JEP 444), and `StampedLock`. |
| **Spring Boot & Microservices** | [`spring-boot.md`](./spring-boot.md) | `@Transactional` propagation, `RestClient`, Spring Security 6 filter chain, and Actuator probe grouping. |
| **PostgreSQL Performance** | [`postgresql-performance.md`](./postgresql-performance.md) | B+Trees, GIN/BRIN indexes, `EXPLAIN BUFFERS`, HikariCP sizing math, and dead tuple bloat. |
| **Redis Commands & Patterns** | [`redis-commands-and-patterns.md`](./redis-commands-and-patterns.md) | Redis data structures, Bitmaps, HyperLogLog, Lua scripting, Redlock, and LRU/LFU eviction policies. |
| **Kafka Architecture & Tuning** | [`kafka-architecture.md`](./kafka-architecture.md) | KRaft metadata, Zero-copy `sendfile()`, Idempotent producers, `CooperativeStickyAssignor`, and `@RetryableTopic`. |
| **Docker & Kubernetes** | [`docker-and-k8s.md`](./docker-and-k8s.md) | Multi-stage Dockerfile, `-XX:MaxRAMPercentage=75.0`, K8s health probes, and Pod Disruption Budgets (PDB). |
| **Distributed Systems** | [`distributed-systems.md`](./distributed-systems.md) | PACELC theorem, Raft quorum election, Snowflake 64-bit ID layout, and Consistent Hashing with VNodes. |
| **Linux Troubleshooting** | [`linux-troubleshooting.md`](./linux-troubleshooting.md) | `top -H`, `jcmd`, `asprof` FlameGraphs, `vmstat`, `iostat`, `ss -tulnp`, and `tcpdump`. |
| **System Design Formulas** | [`system-design-formulas.md`](./system-design-formulas.md) | QPS/TPS math ($10^5\text{s/day}$), 80/20 RAM sizing, Little's Law ($L = \lambda W$), and Tail Latency Amplification ($1-(1-p)^N$). |
