# Module 29: 15 Production-Grade Backend Architecture Projects

Master the complete end-to-end design, implementation, and scaling of 15 production-grade backend engineering systems using Java 21, Spring Boot 3.3.4, PostgreSQL, Redis, Apache Kafka, and AWS cloud patterns.

---

## 🗺️ Master 15 Projects Architecture Catalog

| # | Project Name | Document | Core Technologies & Patterns |
|:---:|---|---|---|
| **01** | **High-Throughput URL Shortener** | [`01-high-throughput-url-shortener.md`](./01-high-throughput-url-shortener.md) | Snowflake 64-bit ID generator, Base62 encoding, Redis L2 caching, and Kafka click analytics. |
| **02** | **Distributed Rate Limiter** | [`02-distributed-rate-limiter.md`](./02-distributed-rate-limiter.md) | Token Bucket, Leaky Bucket, Sliding Window Log in Redis Lua scripts, and Spring Boot filters. |
| **03** | **Distributed Job Scheduler** | [`03-distributed-job-scheduler.md`](./03-distributed-job-scheduler.md) | Redis Sorted Set delayed queues, atomic task claiming, Virtual Threads, and DLQ retries. |
| **04** | **Real-Time Distributed Chat** | [`04-real-time-chat-engine.md`](./04-real-time-chat-engine.md) | STOMP over WebSockets, Redis Pub/Sub inter-node backplane, and Kafka persistence. |
| **05** | **Double-Entry Financial Ledger** | [`05-financial-ledger-system.md`](./05-financial-ledger-system.md) | Double-entry bookkeeping ($\sum\text{Debits}=\sum\text{Credits}$), idempotency keys, and optimistic locking. |
| **06** | **Notification Engine** | [`06-event-driven-notification-engine.md`](./06-event-driven-notification-engine.md) | Multi-channel routing (Email/SMS/Push), priority SQS queues, and user opt-out preferences. |
| **07** | **E-Commerce Flash Sale** | [`07-ecommerce-flash-sale-system.md`](./07-ecommerce-flash-sale-system.md) | Atomic inventory reservation in Redis Lua, sub-2ms fast-fail, and Kafka write buffering. |
| **08** | **High-Performance API Gateway** | [`08-api-gateway.md`](./08-api-gateway.md) | Dynamic routing, non-blocking Netty filters, JWT validation, and Resilience4j circuit breakers. |
| **09** | **Distributed File Storage** | [`09-distributed-file-storage-service.md`](./09-distributed-file-storage-service.md) | S3 Presigned URLs, SHA-256 deduplication, multipart chunked uploads, and Glacier tiering. |
| **10** | **Search & Autocomplete** | [`10-search-and-autocomplete-service.md`](./10-search-and-autocomplete-service.md) | In-memory Trie data structure with Min-Heap Top-K suggestions and BM25 relevance scoring. |
| **11** | **Webhook Delivery System** | [`11-webhook-delivery-system.md`](./11-webhook-delivery-system.md) | HMAC-SHA256 request signing, exponential backoff with full jitter, and per-host bulkheads. |
| **12** | **Distributed Cache Engine** | [`12-distributed-cache-engine.md`](./12-distributed-cache-engine.md) | 32-bit Consistent Hash Ring with VNodes, $O(1)$ LRU memory cache, and singleflight mutexes. |
| **13** | **Real-Time Analytics Pipeline** | [`13-real-time-analytics-pipeline.md`](./13-real-time-analytics-pipeline.md) | Kafka Streams 1-minute tumbling windows, high-throughput ingestion, and ClickHouse storage. |
| **14** | **OAuth2 / OIDC Identity Server** | [`14-auth-and-oidc-identity-server.md`](./14-auth-and-oidc-identity-server.md) | PKCE Authorization Code flow, RS256 asymmetric JWT signing, and RBAC security. |
| **15** | **Distributed Saga Orchestrator** | [`15-workflow-and-saga-orchestrator.md`](./15-workflow-and-saga-orchestrator.md) | Multi-step distributed state machines, automated compensating rollbacks, and audit trails. |
