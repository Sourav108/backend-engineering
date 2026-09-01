# Backend Engineering: Master Curriculum & Source of Truth

This document serves as the **SINGLE SOURCE OF TRUTH** for the entire backend engineering curriculum, tracking all 32 modules, core mechanics, runnable Java 21 projects, production debugging scenarios, and interview drills.

---

## 📊 Global Status Legend
- `TODO`: Planned curriculum module.
- `IN PROGRESS`: Currently being written, coded, and verified.
- `COMPLETE` ✅: Fully authored, diagrammed with Mermaid, coded in Java 21 LTS, tested, and verified.

---

## 📚 32 Modules Overview

| ID | Topic | Path | Status | Dependencies | Code | Project | Interview |
|---|---|---|:---:|---|:---:|:---:|:---:|
 ✅| `MOD-01` | **Backend Fundamentals** | [`01-backend-fundamentals/`](./01-backend-fundamentals/) | `COMPLETE` ✅ | None | ✅ | — | ✅ |
| `MOD-02` | **HTTP and APIs** | [`02-http-and-apis/`](./02-http-and-apis/) | `COMPLETE` ✅ | MOD-01 | ✅ | — | ✅ |
 ✅| `MOD-03` | **Request Lifecycle (Flagship)**| [`03-request-lifecycle/`](./03-request-lifecycle/) | `TODO` | MOD-01, MOD-02 | ✅ | — | ✅ |
| `MOD-04` | **Networking for Backend** | [`04-networking-for-backend/`](./04-networking-for-backend/) | `TODO` | MOD-02 | ✅ | — | ✅ |
 ✅| `MOD-05` | **API Design & Idempotency** | [`05-api-design/`](./05-api-design/) | `TODO` | MOD-02, MOD-03 | ✅ | — | ✅ |
| `MOD-06` | **Authentication & Security** | [`06-authentication-and-security/`](./06-authentication-and-security/) | `TODO` | MOD-05 | ✅ | — | ✅ |
 ✅| `MOD-07` | **Databases (SQL & NoSQL)** | [`07-databases/`](./07-databases/) | `TODO` | MOD-01 | ✅ | — | ✅ |
| `MOD-08` | **Database Internals & Storage**| [`08-database-internals/`](./08-database-internals/) | `TODO` | MOD-07 | ✅ | — | ✅ |
 ✅| `MOD-09` | **Transactions & Consistency** | [`09-transactions-and-consistency/`](./09-transactions-and-consistency/) | `TODO` | MOD-07, MOD-08 | ✅ | — | ✅ |
| `MOD-10` | **Caching & Redis Mechanics** | [`10-caching/`](./10-caching/) | `TODO` | MOD-07 | ✅ | — | ✅ |
 ✅| `MOD-11` | **Messaging & Apache Kafka** | [`11-messaging/`](./11-messaging/) | `TODO` | MOD-01, MOD-09 | ✅ | — | ✅ |
| `MOD-12` | **Event-Driven Architecture** | [`12-event-driven-architecture/`](./12-event-driven-architecture/) | `TODO` | MOD-11 | ✅ | — | ✅ |
 ✅| `MOD-13` | **Distributed Systems Core** | [`13-distributed-systems/`](./13-distributed-systems/) | `TODO` | MOD-09, MOD-11 | ✅ | — | ✅ |
| `MOD-14` | **Resilience & Fault Tolerance**| [`14-resilience/`](./14-resilience/) | `TODO` | MOD-05, MOD-13 | ✅ | — | ✅ |
 ✅| `MOD-15` | **Concurrency & Backpressure** | [`15-concurrency-and-backpressure/`](./15-concurrency-and-backpressure/) | `TODO` | MOD-01, MOD-03 | ✅ | — | ✅ |
| `MOD-16` | **Observability & Tracing** | [`16-observability/`](./16-observability/) | `TODO` | MOD-03, MOD-14 | ✅ | — | ✅ |
 ✅| `MOD-17` | **Performance & p99 Tuning** | [`17-performance/`](./17-performance/) | `TODO` | MOD-08, MOD-15 | ✅ | — | ✅ |
| `MOD-18` | **Scalability & Sharding** | [`18-scalability/`](./18-scalability/) | `TODO` | MOD-08, MOD-10 | ✅ | — | ✅ |
 ✅| `MOD-19` | **Microservices Architecture** | [`19-microservices/`](./19-microservices/) | `TODO` | MOD-14, MOD-16 | ✅ | — | ✅ |
| `MOD-20` | **Service-to-Service Comm** | [`20-service-to-service-communication/`](./20-service-to-service-communication/) | `TODO` | MOD-19 | ✅ | — | ✅ |
 ✅| `MOD-21` | **Background Jobs & Workers** | [`21-background-jobs/`](./21-background-jobs/) | `TODO` | MOD-11, MOD-15 | ✅ | — | ✅ |
| `MOD-22` | **File & Object Storage** | [`22-file-and-object-storage/`](./22-file-and-object-storage/) | `TODO` | MOD-05 | ✅ | — | ✅ |
 ✅| `MOD-23` | **Search & Inverted Indexes** | [`23-search/`](./23-search/) | `TODO` | MOD-07, MOD-12 | ✅ | — | ✅ |
| `MOD-24` | **Real-Time Systems** | [`24-real-time-systems/`](./24-real-time-systems/) | `TODO` | MOD-02, MOD-15 | ✅ | — | ✅ |
 ✅| `MOD-25` | **Cloud Backend (AWS)** | [`25-cloud-backend/`](./25-cloud-backend/) | `TODO` | MOD-19, MOD-22 | ✅ | — | ✅ |
| `MOD-26` | **Containers & Kubernetes** | [`26-containers-and-kubernetes/`](./26-containers-and-kubernetes/) | `TODO` | MOD-19 | ✅ | — | ✅ |
 ✅| `MOD-27` | **Production Engineering** | [`27-production-engineering/`](./27-production-engineering/) | `TODO` | MOD-16, MOD-26 | ✅ | — | ✅ |
| `MOD-28` | **Testing Backend Systems** | [`28-testing/`](./28-testing/) | `TODO` | MOD-07, MOD-11 | ✅ | — | ✅ |
 ✅| `MOD-29` | **Backend Projects (15 Apps)** | [`29-backend-projects/`](./29-backend-projects/) | `TODO` | MOD-01 to MOD-28 | ✅ | ✅ | ✅ |
| `MOD-30` | **Production Debugging (20 Incidents)**| [`30-debugging/`](./30-debugging/) | `TODO` | MOD-08, MOD-16 | ✅ | — | ✅ |
 ✅| `MOD-31` | **System Design Bridge** | [`31-system-design-bridge/`](./31-system-design-bridge/) | `TODO` | MOD-13, MOD-18 | ✅ | — | ✅ |
| `MOD-32` | **Interview Preparation (300+ Qs)**| [`32-interview/`](./32-interview/) | `TODO` | All Modules | — | — | ✅ |
