# Module 03: The Complete Life of a Backend Request (Flagship)

Master the comprehensive, 15-hop end-to-end journey of an enterprise backend request from the user's click across DNS, TCP/TLS, Load Balancers, API Gateways, Application Servers, Framework Dispatch, Database Storage Engines, Distributed Messaging, and Observability Spans.

---

## 🎯 Learning Objectives
- Trace the complete 15-step execution pipeline of a production request.
- Deconstruct the multi-layer DNS resolution hierarchy (Browser $\to$ OS $\to$ Recursive $\to$ Authoritative $\to$ Anycast BGP).
- Analyze kernel-level **TCP 3-Way Handshakes** and **TLS 1.2 vs 1.3 Handshake** latency.
- Evaluate **L4 vs L7 Load Balancing** and Layer 7 Reverse Proxy routing in Envoy / NGINX.
- Implement **API Gateway security validation and distributed Rate Limiting**.
- Contrast **Netty Event Loops (Epoll)** with **Tomcat Thread-per-Request** and **Java 21 Virtual Threads**.
- Navigate framework dispatch across **Spring Filters, Interceptors, Controllers, and `@Transactional` Boundaries**.
- Follow database execution down to **HikariCP pool leasing, B+Tree page seeks, and WAL fsyncs**.
- Implement fault-tolerant outbound integrations with **Resilience4j Circuit Breakers and Transactional Outbox**.
- Master **W3C `traceparent` OpenTelemetry Context Propagation** and Structured Observability.

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**DNS Resolution & Edge Routing**](./01-dns-resolution-and-edge-routing.md) | Browser cache, OS resolver, Recursive DNS, Root/TLD/Authoritative, Anycast BGP | Mermaid, Java 21 |
| **02** | [**TCP & TLS 1.3 Handshake Internals**](./02-tcp-and-tls-handshake-internals.md) | SYN/ACK, SYN Flood, TLS 1.3 1-RTT, SNI, ALPN, Session Tickets, 0-RTT | Mermaid, Java 21 |
| **03** | [**Load Balancing & Reverse Proxies (L4 vs L7)**](./03-load-balancing-and-reverse-proxies.md) | Layer 4 vs Layer 7, TLS termination, Consistent Hashing, Health checks | Mermaid, Java 21 |
| **04** | [**API Gateway, Auth & Distributed Rate Limiting**](./04-api-gateway-auth-and-rate-limiting.md) | Stateless JWT validation, Token Bucket, Sliding Window Redis, CORS | Mermaid, Java 21 |
| **05** | [**App Server Execution & Framework Dispatch**](./05-application-server-and-framework-dispatch.md) | Netty Epoll vs Tomcat Workers, Virtual Threads, DispatcherServlet, Filters | Mermaid, Java 21 |
| **06** | [**Domain Logic, Transactions & Database Persistence**](./06-business-logic-transactions-and-persistence.md) | HikariCP leasing, JPA entity states, B+Tree page seeks, Redis Cache-Aside | Mermaid, Java 21 |
| **07** | [**Outbound Integrations, Circuit Breakers & Outbox**](./07-outbound-integrations-and-fault-tolerance.md) | Resilience4j Circuit Breakers, Bulkheads, Transactional Outbox, Kafka CDC | Mermaid, Java 21 |
| **08** | [**Serialization, Compression & Observability**](./08-response-serialization-compression-and-observability.md) | Jackson zero-copy, Brotli compression, W3C traceparent, Prometheus metrics | Mermaid, Java 21 |
| **09** | [**Master End-to-End Request Lifecycle Trace**](./09-end-to-end-request-lifecycle-master-trace.md) | Complete unified 15-step trace from click to disk to wire with full diagram | Mermaid, Java 21 |

---

## 🛠️ Verification & Drills
- Run unit drills using `mvn test` in the parent module.
- Inspect the complete 15-step sequence diagram in Lesson 09.
