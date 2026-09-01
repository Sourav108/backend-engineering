# Contributing & Engineering Guidelines

This repository establishes rigorous engineering, pedagogical, and code quality standards for SDE2 and Senior Backend Engineering preparation.

---

## 🎯 Target Technical Baseline
- **Primary Language**: Java 21 LTS (`Amazon Corretto` / `Eclipse Temurin`)
- **Build System**: Apache Maven 3.9+
- **Spring Boot Baseline**: **`3.3.4`** (Pinned stable release for Java 21 compatibility)
- **Testing**: JUnit 5, Mockito 5, Testcontainers 1.19+
- **Databases & Messaging**: PostgreSQL 16, MySQL 8, Redis 7.2, Apache Kafka 3.7+
- **Observability**: OpenTelemetry, Micrometer, Prometheus, W3C Trace Context

---

## 🧪 Test Execution Tiers

To balance developer velocity and rigorous integration verification, all projects and code implementations must adhere to two distinct execution tiers:

### 1. Fast Tier (`mvn test`)
- Executes **pure unit tests**, Mockito unit tests, and lightweight in-memory logic.
- **Strict Rule**: Zero Docker dependencies, zero container startups, runs in $< 5$ seconds.

### 2. Integration Tier (`mvn verify -Pintegration`)
- Executes container-backed integration tests via **Testcontainers** (Real PostgreSQL, Redis, Kafka, MySQL).
- Named using `*IT.java` convention and executed via `maven-failsafe-plugin`.

---

## 📐 Educational Complexity Progression (V1 -> V2 -> V3)
Never over-engineer educational projects from day one. Follow this strict progression:
1. **V1 — Simple Correct Implementation**: In-memory, baseline semantics, clean code, tests pass.
2. **V2 — Production Improvements**: Connection pools, timeouts, retries, metrics, logging, database integration.
3. **V3 — Distributed / Scaled Version**: Distributed caching, messaging, idempotency, backpressure, circuit breaking.

---

## 📊 Diagram Standard (Mermaid)
All diagrams must use valid Mermaid syntax:
- `flowchart TD` or `flowchart LR` for request lifecycles and architectural flows.
- `sequenceDiagram` for distributed protocol interactions and RPC lifecycles.
- `stateDiagram-v2` for state machines (Circuit breakers, connection pool states).
- `classDiagram` for domain models and design patterns.
