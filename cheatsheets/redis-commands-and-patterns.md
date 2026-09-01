# Redis Commands & Caching Patterns Cheat Sheet

---

## ⚡ 1. Essential Data Structures

| Structure | Common Commands | Complexity | Best Use Case |
|---|---|:---:|---|
| **String** | `SET`, `GET`, `INCRBY`, `SETNX` | $O(1)$ | Session tokens, counters, distributed locks |
| **Hash** | `HSET`, `HGETALL`, `HINCRBY` | $O(1) / O(N)$ | User profiles, cart objects |
| **List** | `LPUSH`, `RPOP`, `BRPOPLPUSH` | $O(1)$ | Reliable queue workers, event logs |
| **Set** | `SADD`, `SISMEMBER`, `SINTER` | $O(1) / O(N)$ | Unique tags, friend relationships |
| **Sorted Set** | `ZADD`, `ZRANGEBYSCORE`, `ZREVRANK` | $O(\log N)$ | Leaderboards, delayed task queues, rate limiters |
| **Bitmap** | `SETBIT`, `GETBIT`, `BITCOUNT` | $O(1)$ | Daily active user (DAU) presence ($1\text{M users} = 125\text{KB}$) |
| **HyperLogLog** | `PFADD`, `PFCOUNT` | $O(1)$ | Cardinality estimation ($12\text{KB}$ fixed memory) |

---

## ⚡ 2. Production Distributed Lock (Redlock / Lua)

```lua
-- Atomic Release Lock (Only release if token matches owner!)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

```bash
# Acquire Lock with 10-Second Expiry
SET lock:order:101 "unique-uuid-token" NX PX 10000
```

---

## ⚡ 3. Sliding Window Rate Limiter (Lua Script)

```lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local current = redis.call('ZCARD', key)

if current < limit then
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, math.ceil(window / 1000))
    return 1 -- Allowed
else
    return 0 -- Throttled
end
```

---

## ⚡ 4. Eviction Policies (`maxmemory-policy`)

- `allkeys-lru`: Evicts least recently used keys across all keys (Best for general caching).
- `volatile-lru`: Evicts LRU keys among keys with an explicit `EXPIRE` set.
- `allkeys-lfu`: Evicts least frequently used keys (Best for frequency-based hot spots).
