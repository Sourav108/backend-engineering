# Module 02: HTTP and APIs

Master the communication protocols, transport evolution, RESTful design principles, and client connection management powering the modern web.

---

## 🎯 Learning Objectives
- Deconstruct the evolution of HTTP: **HTTP/1.0 $\to$ HTTP/1.1 $\to$ HTTP/2 $\to$ HTTP/3 (QUIC)**.
- Analyze **Head-of-Line (HOL) Blocking** at both application and transport (TCP) layers.
- Master HTTP Methods, the **Safe vs Idempotent Matrix**, and **RFC 9457 Problem Details**.
- Implement Conditional Requests with **ETags, `If-None-Match`, and `Cache-Control`**.
- Configure Enterprise Cookie Security (`SameSite`, `HttpOnly`, `Secure`) and CORS.
- Evaluate the **Richardson Maturity Model** (Level 0 through Level 3 HATEOAS).
- Compare **Offset vs Keyset/Cursor Pagination** on database B+Tree indexes.
- Tune HTTP Client Connection Pools (`maxTotal`, `defaultMaxPerRoute`) to prevent pool starvation.

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**HTTP Evolution: 1.1 vs 2 vs 3 (QUIC)**](./01-http-evolution-and-protocols.md) | HOL blocking, Binary framing, Streams, Multiplexing, UDP QUIC, 0-RTT | Mermaid, Java 21 |
| **02** | [**HTTP Methods, Idempotency & Status Codes**](./02-http-methods-idempotency-and-status-codes.md) | Safe vs Idempotent, 2xx-5xx semantics, RFC 9457 Problem Details | Mermaid, Java 21 |
| **03** | [**Headers, Content Negotiation & HTTP Caching**](./03-headers-content-negotiation-and-caching.md) | ETags, 304 Not Modified, Cache-Control, stale-while-revalidate, CDN caching | Mermaid, Java 21 |
| **04** | [**Cookies, Sessions & Browser Security Headers**](./04-cookies-sessions-and-security-headers.md) | SameSite (Strict/Lax/None), HttpOnly, CORS Preflight, HSTS, CSP | Mermaid, Java 21 |
| **05** | [**REST & The Richardson Maturity Model**](./05-rest-and-richardson-maturity-model.md) | Levels 0-3 HATEOAS, Resource URI design, Pragmatic REST API rules | Mermaid, Java 21 |
| **06** | [**API Pagination, Filtering & Sorting**](./06-pagination-filtering-and-sorting.md) | Offset vs Keyset/Cursor pagination, B+Tree seek vs scan, Deep pagination trap | Mermaid, Java 21 |
| **07** | [**HTTP Client Connection Pools & Tuning**](./07-http-client-connection-pools.md) | Apache HttpClient / RestClient, maxPerRoute, Keep-Alive, Pool Starvation | Mermaid, Java 21 |

---

## 🛠️ Verification & Drills
- Run unit drills using `mvn test` in the parent module.
- Validate Keyset pagination SQL generation against mock datasets.
