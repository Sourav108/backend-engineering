# Module 08: Fault Tolerance & Distributed Resilience Patterns

## Overview
In distributed backend systems, failures are not anomalies—they are inevitable mathematical certainties. Hardware crashes, transient network partitions, packet loss, downstream GC pauses, database connection exhaustion, and cascading thundering herds threaten system stability. This module covers the theoretical principles, architectural patterns, and production Java 21 / Resilience4j / Spring Cloud implementations required to design, build, and operate self-healing, fault-tolerant backend architectures.

---

## Module Roadmap

| Lesson | Title | Core Focus Areas | Key Patterns / Technologies |
| :--- | :--- | :--- | :--- |
| **01** | [Failure Modes & Distributed Fault Models](./01-failure-modes-and-fault-models.md) | Byzantine vs Crash-Stop, Split-Brain, Network Partitions, Cascading Outages, CAP & PACELC Trade-offs | Formal Fault Taxonomies, Quorum Mechanics, Partition Tolerance |
| **02** | [Circuit Breakers, Bulkheads & Timeouts](./02-circuit-breakers-bulkheads-timeouts.md) | State Machine (Closed/Open/Half-Open), Thread Pool vs Semaphore Isolation, Latency Budget Propagation | Resilience4j, Spring Cloud, Sliding Window Metrics, Micrometer |
| **03** | [Retries, Exponential Backoff & Jitter](./03-retries-backoff-and-jitter.md) | Full Jitter, Equal Jitter, Decorrelated Jitter, Retry Amplification Cascades, Idempotency Keys | Mathematical Backoff Formulations, Resilience4j Retry, Idempotent APIs |
| **04** | [Rate Limiting, Throttling & Shedding](./04-rate-limiting-and-load-shedding.md) | Token Bucket, Leaky Bucket, Sliding Window Log, CoDel, Little's Law, CPU/Queue Adaptive Shedding | Redis Lua, Resilience4j RateLimiter, Concurrency Limits |
| **05** | [Graceful Degradation & Fallbacks](./05-graceful-degradation-and-fallbacks.md) | Tiered Feature Flags, Static & Cache Fallbacks, Stale-While-Revalidate, Read-Only Failover | Unlaunch/Togglz, SWR Caching, Read-Only Replicas, Disaster Recovery |
| **06** | [Chaos Engineering & Fault Injection](./06-chaos-engineering-and-fault-injection.md) | Steady-State Hypothesis, Latency/Packet-Drop Injection, Toxiproxy, Chaos Mesh, GameDay Drills | Chaos Mesh, Toxiproxy, Testcontainers, Automated Fault Drills |

---

## Key Learning Objectives
1. **Architectural Isolation**: Master thread-pool and semaphore bulkhead isolation to guarantee that a failure in one dependency cannot exhaust shared container resources.
2. **Cascading Failure Prevention**: Implement state-of-the-art circuit breakers, adaptive load shedding via Little's Law, and decorrelated jittered retries to eliminate retry storms and thundering herds.
3. **Graceful Degradation Under Load**: Design degradation pathways using static defaults, stale-while-revalidate cached responses, and read-only failover states to maintain high availability for core user journeys.
4. **Resilience Verification**: Operationalize Chaos Engineering methodologies using Toxiproxy and Chaos Mesh to validate system steady-state hypotheses prior to production incidents.
