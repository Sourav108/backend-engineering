# Incident 02: JVM Memory Leak and Heap Dump Analysis

---

## 1. Symptoms & Alert
- **Alert**: `Kubernetes Pod OOMKilled (Exit Code 137)` repeating in `payment-service`.
- **Customer Impact**: Periodic $30\text{-second}$ checkout freezes followed by `502 Bad Gateway` errors as pods crash-restarted in a loop.

---

## 2. Metric & Telemetry Anomalies
- **Grafana JVM Dashboard**: JVM Old Generation memory showed a classic **Sawtooth Memory Leak Pattern** (each GC cycle reclaimed less and less memory until Old Gen sat permanently at $98\%$).
- **GC Telemetry**: GC pause frequency spiked from once per 10 minutes to **10 times per second** (GC Thrashing) immediately prior to container death.

---

## 3. Diagnostic Steps

### Step 1: Capture Heap Dump on OutOfMemory
The Kubernetes deployment was configured with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/oom_dump.hprof`.

### Step 2: Download and Open in Eclipse Memory Analyzer (MAT)
```bash
kubectl cp payment-service-589f89-8w2q9:/tmp/oom_dump.hprof ./oom_dump.hprof
```

### Step 3: Run MAT "Leak Suspects Report"
```text
Problem Suspect 1:
The class "com.backend.engineering.metrics.UserSessionTracker", loaded by "app", 
occupies 1,482,104,240 (84.12%) bytes.

The memory is accumulated in one instance of "java.util.concurrent.ConcurrentHashMap" 
holding 1,840,210 objects of "com.backend.engineering.dto.UserSessionContext".
```

---

## 4. Root Cause Analysis
A developer implemented an in-memory tracking map (`private static final Map<String, UserSessionContext> sessionMap = new ConcurrentHashMap<>()`) to track active user clicks.
- When users logged in, `sessionMap.put(sessionId, context)` was invoked.
- However, when sessions expired or users logged out, **no cleanup or eviction logic was ever executed**.
- Over 7 days in production, the static map accumulated 1.8 million dead session objects, consuming 1.5GB of heap RAM until the JVM exhausted its memory limit and was terminated by the Linux OOM Killer.

---

## 5. Immediate Mitigation
1. **Trigger Rolling Restart to Free Heap**:
   ```bash
   kubectl rollout restart deployment payment-service -n production
   ```
2. **Temporarily Bump Container Memory Limit** from `2Gi` to `4Gi` to extend mean time between failures while the fix is deployed.

---

## 6. Permanent Fix
Replace the unbounded `ConcurrentHashMap` with a bounded **Caffeine L1 Cache** enforcing strict size ceilings and time-based TTL eviction:

```java
// BEFORE (UNBOUNDED MEMORY LEAK)
private static final Map<String, UserSessionContext> sessionMap = new ConcurrentHashMap<>();

// AFTER (BOUNDED CAFFEINE CACHE WITH AUTOMATIC EXPIRATION)
private final Cache<String, UserSessionContext> sessionCache = Caffeine.newBuilder()
        .maximumSize(50_000)                        // Strict memory bound!
        .expireAfterWrite(Duration.ofMinutes(30))   // Auto-evict stale sessions
        .recordStats()
        .build();
```

---

## 7. Postmortem Action Items
- [x] Configure automated leak detection alerts when JVM Old Gen remains $> 85\%$ after major GC.
- [x] Enforce architectural lint rule prohibiting raw unbounded `static Map` collections for caching.
