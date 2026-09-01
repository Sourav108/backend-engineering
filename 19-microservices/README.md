# Module 07: Microservices Architecture & Patterns

Master enterprise microservices architecture: Domain-Driven Design (DDD) bounded contexts, Strangler Fig decomposition, API Gateway & BFF (Backend-For-Frontend), Service Discovery with client-side load balancing, Saga Pattern for distributed transactions, CQRS with Event Sourcing, and Transactional Outbox with Debezium Change Data Capture (CDC).

---

## 🎯 Learning Objectives
- Decompose monolithic codebases into autonomous bounded contexts using the **Strangler Fig Pattern**.
- Architect edge routing and aggregation with **API Gateways and Backend-For-Frontend (BFF)**.
- Eliminate distributed transaction deadlocks using **Saga Orchestration & Choreography with Compensating Transactions**.
- Decouple reads and writes at scale using **CQRS and Event Sourcing**.
- Solve the dual-write problem using the **Transactional Outbox Pattern with Debezium CDC**.

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**Microservices Decomposition & DDD**](./01-microservices-decomposition-and-domain-driven-design.md) | Bounded Contexts, Aggregates, Strangler Fig, Branch by Abstraction | Mermaid, Java 21 |
| **02** | [**API Gateway Pattern & BFF**](./02-api-gateway-pattern-and-bff.md) | Spring Cloud Gateway, BFF Architecture, Request Aggregation, SSL Offload | Mermaid, Java 21 |
| **03** | [**Service Discovery & Client Load Balancing**](./03-service-discovery-and-client-side-load-balancing.md) | Eureka / Consul / K8s DNS, Client-Side LB, Least Connection, Health Probing | Mermaid, Java 21 |
| **04** | [**Distributed Transactions & Saga Pattern**](./04-distributed-transactions-and-saga-pattern.md) | 2PC Failure, Choreography vs Orchestration, Compensating Actions | Mermaid, Java 21 |
| **05** | [**CQRS & Event Sourcing**](./05-cqrs-and-event-sourcing.md) | Command/Query Separation, Event Store, Snapshots, Read Projections | Mermaid, Java 21 |
| **06** | [**Event-Driven Architecture & Transactional Outbox**](./06-event-driven-architecture-and-transactional-outbox.md) | Dual-Write Hazard, Outbox Table, Debezium CDC, At-Least-Once Delivery | Mermaid, Java 21 |

---

## 🛠️ Verification & Drills
- Implement Saga Orchestration with compensating rollback transactions in Java 21.
- Write a Transactional Outbox publisher and verify CDC event streaming into Kafka.
