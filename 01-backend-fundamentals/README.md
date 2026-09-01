# Module 01: Backend Engineering Fundamentals

Master the foundational architectural principles, execution models, and system properties that govern modern production backend systems.

---

## 🎯 Learning Objectives
- Deconstruct the Client/Server and Request/Response paradigms across network boundaries.
- Distinguish Stateless vs Stateful architectures and their implications on horizontal scalability.
- Differentiate Synchronous vs Asynchronous and Blocking vs Non-Blocking execution models.
- Analyze CPU-Bound vs I/O-Bound workloads and calculate optimal thread pool sizing formulas.
- Master the mathematical relationships governing **Latency, Throughput, and Concurrency** (Little's Law).
- Evaluate System Properties: Availability, Reliability (MTBF/MTTR), Durability (fsync/WAL), and Consistency.

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**What is a Backend & Client-Server Model**](./01-what-is-a-backend-and-client-server-model.md) | Request/Response, Service Boundaries, Transport boundaries, API contracts | Mermaid, Java 21 |
| **02** | [**Stateless vs Stateful Services**](./02-stateless-vs-stateful-services.md) | Horizontal vs Vertical Scaling, Session state, Sticky routing, Shared-nothing | Mermaid, Java 21 |
| **03** | [**Synchronous vs Asynchronous & Blocking vs Non-Blocking**](./03-synchronous-vs-asynchronous-and-blocking-vs-nonblocking.md) | Event loops, Thread-per-request, Reactive streams, Virtual threads | Mermaid, Java 21 |
| **04** | [**CPU-Bound vs I/O-Bound Workloads**](./04-cpu-bound-vs-io-bound-workloads.md) | Context switching, Thread pool sizing formulas, OS scheduler runqueues | Mermaid, Java 21 |
| **05** | [**Latency, Throughput, and Concurrency**](./05-latency-throughput-and-concurrency.md) | Little's Law ($L = \lambda W$), Saturation curves, p99 tail latency, Coordinated Omission | Mermaid, Java 21 |
| **06** | [**Availability, Reliability, Durability & Consistency**](./06-system-properties-availability-reliability-durability-consistency.md) | Error budgets, Nines of availability, MTBF/MTTR, fsync persistence, ACID | Mermaid, Java 21 |

---

## 🛠️ Verification & Drills
- Run unit drills using `mvn test` in the parent module.
- Validate Little's Law throughput calculations against simulated concurrency queues.
