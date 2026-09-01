# Module 30: Production Debugging Incidents & Runbooks (20 Scenarios)

Master real-world production incident response and diagnostic engineering: 20 comprehensive production outage scenarios covering root-cause mechanics, telemetry anomalies, diagnostic commands, immediate mitigations, permanent architectural fixes, and blameless postmortems.

---

## 🗺️ Master Production Incidents Catalog

| # | Incident Document | Domain / Subsystem | Failure Mechanism & Root Cause |
|:---:|---|---|---|
| **01** | [`01-incident-high-cpu-utilization.md`](./01-incident-high-cpu-utilization.md) | JVM / CPU | Unanchored nested regex triggering Catastrophic Backtracking ($O(2^N)$). |
| **02** | [`02-incident-jvm-memory-leak-heap-dump-analysis.md`](./02-incident-jvm-memory-leak-heap-dump-analysis.md) | JVM Memory | Unbounded `ConcurrentHashMap` leaking 1.8M dead session objects; OOM 137. |
| **03** | [`03-incident-thread-deadlocks.md`](./03-incident-thread-deadlocks.md) | Java Concurrency | Inconsistent lock acquisition ordering causing circular Coffman deadlock. |
| **04** | [`04-incident-db-connection-pool-exhaustion.md`](./04-incident-db-connection-pool-exhaustion.md) | Database / HikariCP | 30-second blocking third-party HTTP call held inside `@Transactional`. |
| **05** | [`05-incident-slow-db-queries-and-table-scans.md`](./05-incident-slow-db-queries-and-table-scans.md) | PostgreSQL / Indexing | Missing composite B+Tree index causing 15M row sequential scan under load. |
| **06** | [`06-incident-cache-stampede-and-dogpiling.md`](./06-incident-cache-stampede-and-dogpiling.md) | Redis / Caching | Expired hot key triggering simultaneous 32k QPS cache miss flood on DB. |
| **07** | [`07-incident-redis-out-of-memory.md`](./07-incident-redis-out-of-memory.md) | Redis Architecture | Default `maxmemory-policy: noeviction` rejecting writes on full memory. |
| **08** | [`08-incident-kafka-consumer-lag-surge.md`](./08-incident-kafka-consumer-lag-surge.md) | Apache Kafka | Poison pill message triggering infinite crash-rebalance loop. |
| **09** | [`09-incident-cascading-microservice-failure.md`](./09-incident-cascading-microservice-failure.md) | Microservices IPC | Unbounded HTTP socket timeouts propagating upstream thread starvation. |
| **10** | [`10-incident-split-brain-in-distributed-cluster.md`](./10-incident-split-brain-in-distributed-cluster.md) | Distributed Consensus | 4-node cluster network partition with invalid even-quorum rule. |
| **11** | [`11-incident-silent-data-corruption.md`](./11-incident-silent-data-corruption.md) | Relational Integrity | Non-atomic read-modify-write causing silent Lost Update anomalies. |
| **12** | [`12-incident-cross-az-network-partition.md`](./12-incident-cross-az-network-partition.md) | Cloud Networking | Single-standby synchronous replication stalling DB writes on AZ severance. |
| **13** | [`13-incident-thread-pool-exhaustion.md`](./13-incident-thread-pool-exhaustion.md) | Thread Pools | `CallerRunsPolicy` hijacking Tomcat request threads to run heavy background tasks. |
| **14** | [`14-incident-file-descriptor-exhaustion.md`](./14-incident-file-descriptor-exhaustion.md) | Linux OS / Sockets | Unclosed S3 `InputStream` leaking Linux OS file descriptors (`EMFILE`). |
| **15** | [`15-incident-disk-full-and-log-bloat.md`](./15-incident-disk-full-and-log-bloat.md) | Kubernetes / Storage | `DEBUG` logging generating 84GB plaintext logs, triggering `DiskPressure`. |
| **16** | [`16-incident-ssl-tls-certificate-expiration.md`](./16-incident-ssl-tls-certificate-expiration.md) | Security / TLS | Expired X.509 mTLS certificate breaking internal service communication. |
| **17** | [`17-incident-dns-resolution-failure.md`](./17-incident-dns-resolution-failure.md) | Kubernetes Networking | Under-provisioned CoreDNS pods overwhelmed by 85k DNS queries/sec. |
| **18** | [`18-incident-rate-limit-cascading-failure.md`](./18-incident-rate-limit-cascading-failure.md) | API Gateway | Fail-Closed rate limiting filter locking out 100% of traffic on Redis blip. |
| **19** | [`19-incident-webhook-delivery-failure.md`](./19-incident-webhook-delivery-failure.md) | Webhook Engine | Single slow merchant endpoint blocking shared worker thread pool. |
| **20** | [`20-incident-outbox-event-publishing-lag.md`](./20-incident-outbox-event-publishing-lag.md) | Event Architecture | 48-minute uncommitted batch transaction freezing Debezium CDC LSN. |
