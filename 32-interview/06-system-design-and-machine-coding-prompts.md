# System Design Scenarios and Machine Coding Interview Blueprint

Comprehensive architectural blueprints for the top 10 backend system design scenarios with exact database schemas, mathematical sizing, and machine coding evaluation rubrics.

---

## 🗺️ The 10 Master System Design Scenarios

---

### 1. Distributed Rate Limiter (Token Bucket / Sliding Window)
- **Problem**: Protect ingress APIs from DDoS and enforce rate limits ($100\text{ req/min}$ per API key).
- **Core Architecture**:
  - **Redis Lua Script**: Executes sliding window log count atomically:
    ```lua
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local limit = tonumber(ARGV[3])
    local clearBefore = now - window
    redis.call('ZREMRANGEBYSCORE', key, 0, clearBefore)
    local currentRequests = redis.call('ZCARD', key)
    if currentRequests < limit then
        redis.call('ZADD', key, now, now)
        redis.call('EXPIRE', key, math.ceil(window / 1000))
        return 1 -- ALLOWED
    else
        return 0 -- REJECTED (HTTP 429 Too Many Requests)
    end
    ```
- **Trade-Off**: Sliding Window Log uses more RAM than Token Bucket, but completely eliminates boundary-burst attacks.

---

### 2. Scalable URL Shortener (TinyURL)
- **Problem**: Shorten 100M URLs/month; redirect $50,000\text{ req/sec}$ with $< 10\text{ms}$ latency.
- **Core Architecture**:
  - **ID Generation**: 64-bit Twitter Snowflake ID generator (or pre-generated ZooKeeper counter range).
  - **Base62 Encoding**: Encode 64-bit integer into 7-character alphanumeric string ($62^7 = 3.5\text{ Trillion}$ combinations).
  - **HTTP Redirection**: Return **`HTTP 302 Found` (Temporary Redirect)** so that the backend can track click telemetry (or `HTTP 301 Moved Permanently` to maximize browser CDN caching).
  - **Data Tier**: PostgreSQL with B+Tree index on `short_code` + Redis L2 Cache with 7-day LRU eviction.

---

### 3. Flash Sale / High-Concurrency Inventory System
- **Problem**: 100,000 users attempting to purchase 1,000 iPhone units simultaneously without overselling.
- **Core Architecture**:
  ```mermaid
  flowchart LR
      User["100,000 Users"] --> ALB["Application Load Balancer"]
      ALB --> Spring["Spring Boot Pods"]
      Spring --> RedisLua["Redis Atomic Decrement (DECRBY)"]
      RedisLua -- Stock > 0 --> Kafka["Kafka Topic: 'orders-placed'"]
      RedisLua -- Stock == 0 --> SoldOut["Return 'Sold Out' (< 2ms)"]
      Kafka --> Worker["Worker Pool (Batch SQL Insert into Postgres)"]
  ```
- **Key Invariant**: **Never query the relational database on the read/checkout path during a flash sale**. Decrement stock atomically in Redis memory, and buffer order generation into Kafka to write to PostgreSQL at a steady, sustainable pace.

---

### 4. Distributed Job Scheduler (Cron & Delayed Tasks)
- **Problem**: Schedule millions of one-off and recurring tasks with guaranteed execution and automatic worker failover.
- **Core Architecture**:
  - **Storage**: Redis Sorted Set (`jobs:delayed`) with target epoch millisecond timestamp as the `score`.
  - **Poller**: Worker threads poll `ZRANGEBYSCORE jobs:delayed 0 <now> LIMIT 0 50` and claim via atomic `ZREM`.
  - **Deduplication & Locking**: ShedLock distributed lock on recurring crons; optimistic locking on task state updates (`WHERE version = expectedVersion`).

---

### 5. Real-Time Distributed Chat Platform (WhatsApp / Discord)
- **Problem**: Support 10M concurrent users chatting in 1-on-1 and group channels with sub-50ms message latency.
- **Core Architecture**:
  - **Transport**: Persistent WebSockets with STOMP framing over TLS.
  - **Inter-Server Backplane**: Redis Pub/Sub channel per room (`chat:room:<roomId>`) to broadcast messages across 100 independent WebSocket server pods.
  - **Persistence**: Messages buffered in Kafka and written asynchronously to Apache Cassandra / ScyllaDB partitioned by `(room_id, bucket_week)` with `message_id` clustering keys.
  - **Presence**: Ephemeral Redis keys with 60s TTL (`SET presence:user_101 "ONLINE" EX 60`) renewed via 20s client heartbeat pings.

---

### 6. High-Throughput Notification & Webhook Engine
- **Problem**: Ingest 10,000 notifications/sec; deliver emails, SMS, mobile push, and external HTTP webhooks with retries.
- **Core Architecture**:
  - **Multi-Bucket Priority Queues**: `queue:critical` (Password resets), `queue:default` (Order updates), `queue:low` (Marketing).
  - **Exponential Backoff with Full Jitter**:
    $$\text{Delay} = \text{UniformRandom}(0, \min(60\text{s}, 1\text{s} \times 2^{\text{attempt}}))$$
  - **Webhook Security**: Sign payloads with HMAC-SHA256 in header `X-Signature-SHA256` using customer's secret key.
  - **Poison Pill Isolation**: Isolate failing webhooks to Dead Letter Queues (DLQ) after 5 failed attempts.

---

### 7. Real-Time Gaming Live Leaderboard
- **Problem**: Real-time ranking for 50 million players with instantaneous score updates and top-100 queries.
- **Core Architecture**:
  - **Data Structure**: Redis Sorted Set (`ZSet`) using **Skiplist + Hash Table** encoding in RAM ($O(\log N)$ updates).
  - **Operations**:
    - Update Score: `ZINCRBY leaderboard:weekly 500 "player_42"`
    - Get Rank: `ZREVRANK leaderboard:weekly "player_42"` ($< 1\text{ms}$)
    - Fetch Top 100: `ZREVRANGE leaderboard:weekly 0 99 WITHSCORES` ($< 2\text{ms}$)

---

## 📋 Machine Coding Evaluation Rubric

When evaluating machine coding / low-level design (LLD) interviews:

| Criteria | Weight | What Top-Tier Interviewers Look For |
|---|:---:|---|
| **Clean Architecture & Separation of Concerns** | $30\%$ | Clear separation: Controller $\to$ Service $\to$ Domain Model $\to$ Repository. Zero business logic inside controllers or database entities. |
| **Concurrency & Thread Safety** | $25\%$ | Correct usage of `ConcurrentHashMap`, `ReentrantLock`, `AtomicLong`, or `@Transactional` isolation. Zero race conditions under concurrent test threads. |
| **Extensibility & Design Patterns** | $20\%$ | Strategy pattern for payment gateways/notification channels; Factory pattern for parsers; Observer pattern for event dispatching. |
| **Edge Cases & Error Handling** | $15\%$ | Handling nulls, negative amounts, out-of-stock conditions, invalid state transitions, custom domain exceptions. |
| **Unit & Integration Test Coverage** | $10\%$ | Deterministic JUnit 5 unit tests with Mockito and AssertJ covering happy path and edge failure branches. |
