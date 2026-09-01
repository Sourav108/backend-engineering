# 6-Week Structured Backend Engineering Roadmap

A realistic, high-impact 6-week curriculum designed for **1–2 hours/day** of hands-on study, coding, profiling, and interview preparation.

---

## 🗓️ Daily Learning Cycle
$$\textbf{UNDERSTAND} \longrightarrow \textbf{CODE} \longrightarrow \textbf{DEBUG} \longrightarrow \textbf{INTERVIEW} \longrightarrow \textbf{REVISE}$$

---

## 📅 Week-by-Week Breakdown

### 🌟 Week 1: Foundations, Protocols & API Architecture
- **Focus**: Backend fundamentals, Client-to-Server request lifecycle, HTTP/1.1 vs HTTP/2 vs HTTP/3, RESTful resource design, idempotency, rate limiting, and API versioning.
- **Key Modules**:
  - `01-backend-fundamentals`
  - `02-http-and-apis`
  - `03-request-lifecycle`
  - `04-networking-for-backend`
  - `05-api-design`
  - `06-authentication-and-security`

### 🌟 Week 2: Storage Engines, Transactions & Caching
- **Focus**: Relational database storage internals (B+Tree page layout, WAL, MVCC), ACID transaction isolation anomalies, database connection pool sizing (HikariCP), Redis data structures, distributed locking, and cache stampede defenses.
- **Key Modules**:
  - `07-databases`
  - `08-database-internals`
  - `09-transactions-and-consistency`
  - `10-caching`

### 🌟 Week 3: Messaging, Event-Driven Systems & Distributed Fundamentals
- **Focus**: Synchronous vs Asynchronous architectures, Apache Kafka deep dive (KRaft, partitions, consumer groups, `CooperativeStickyAssignor`), Transactional Outbox pattern with Debezium CDC, CAP/PACELC trade-offs, and distributed unique ID generators.
- **Key Modules**:
  - `11-messaging`
  - `12-event-driven-architecture`
  - `13-distributed-systems`

### 🌟 Week 4: Concurrency, Resilience & Microservices Communication
- **Focus**: Java 21 Virtual Threads vs Platform Thread pools, backpressure mechanics, Resilience4j Circuit Breakers, Bulkheads, Rate Limiters, gRPC vs REST vs Kafka, and service boundaries.
- **Key Modules**:
  - `14-resilience`
  - `15-concurrency-and-backpressure`
  - `19-microservices`
  - `20-service-to-service-communication`
  - `21-background-jobs`

### 🌟 Week 5: Observability, Performance & Production Engineering
- **Focus**: Structured logging with correlation IDs, Prometheus RED/USE metrics, W3C distributed tracing with OpenTelemetry, p99 latency profiling, JVM GC tuning, zero-downtime database migrations (Expand/Contract), and Kubernetes workload management.
- **Key Modules**:
  - `16-observability`
  - `17-performance`
  - `18-scalability`
  - `25-cloud-backend`
  - `26-containers-and-kubernetes`
  - `27-production-engineering`
  - `28-testing`

### 🌟 Week 6: Runnable Projects, Production Debugging & Interview Mastery
- **Focus**: Complete runnable backend projects, live production debugging incident scenarios (100% CPU, OOM, connection exhaustion, Kafka lag), System Design bridge, and 300+ SDE2 interview drills.
- **Key Modules**:
  - `22-file-and-object-storage`
  - `23-search`
  - `24-real-time-systems`
  - `29-backend-projects`
  - `30-debugging`
  - `31-system-design-bridge`
  - `32-interview`
  - `cheatsheets/`
