# Module 05: API Design & Idempotency

Master production-grade API architecture: Richardson Maturity Model, Stripe-style idempotency keys with distributed locking, cursor-based deep pagination, RFC 7807 Problem Details, API versioning strategies, and IETF rate limit headers.

---

## 🎯 Learning Objectives
- Design clean RESTful APIs according to the **Richardson Maturity Model (Levels 0–3)**.
- Implement rock-solid **Stripe-style Idempotency Keys** using Redis distributed locks and DB unique constraints.
- Eliminate database performance cliffs by transitioning from **Offset Pagination to Keyset/Cursor Pagination**.
- Implement **RFC 7807 Problem Details** for consistent, secure error responses.
- Safely evolve APIs using **URI vs Header Versioning** and `Sunset` / `Deprecation` HTTP headers.
- Publish standard IETF **RateLimit Headers** (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`).

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**REST Principles & Richardson Maturity**](./01-rest-api-design-principles-and-maturity-model.md) | Levels 0–3, HATEOAS, Resource Naming, HTTP Verbs & Status Codes | Mermaid, Java 21 |
| **02** | [**Idempotency Keys & Distributed Locking**](./02-idempotency-keys-and-distributed-locking.md) | Stripe `Idempotency-Key`, SHA-256 Fingerprinting, Redis Lock + DB Unique Key | Mermaid, Java 21 |
| **03** | [**Pagination, Filtering & Sorting**](./03-pagination-filtering-and-sorting.md) | Offset ($O(N)$) vs Cursor ($O(1)$) Pagination, Seek Method, Sorting Indexes | Mermaid, Java 21 |
| **04** | [**API Versioning Strategies & Evolution**](./04-api-versioning-strategies-and-evolution.md) | URI vs Header vs Media Type, `Sunset` RFC 8594, Breaking Changes | Mermaid, Java 21 |
| **05** | [**Error Handling & RFC 7807 Problem Details**](./05-error-handling-and-rfc-7807-problem-details.md) | `application/problem+json`, Global Exception Handlers, Trace ID injection | Mermaid, Java 21 |
| **06** | [**Rate Limiting Design & IETF Headers**](./06-rate-limiting-design-and-headers.md) | `RateLimit-Limit`, `RateLimit-Reset`, `429 Too Many Requests`, Quotas | Mermaid, Java 21 |

---

## 🛠️ Verification & Drills
- Run idempotency integration tests simulating concurrent duplicate HTTP POST requests.
- Benchmark Offset vs Keyset pagination queries on $1,000,000$ row PostgreSQL tables.
