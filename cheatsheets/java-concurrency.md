# Java Concurrency & JVM Cheat Sheet (Java 21 LTS)

---

## ⚡ 1. JMM & Memory Visibility

```java
// Volatile: Happens-before write visibility; prevents instruction reordering
private volatile boolean running = true;

// CAS (Compare-And-Swap): Hardware-level lock-free atomic update (LOCK CMPXCHG)
AtomicLong balance = new AtomicLong(1000L);
boolean success = balance.compareAndSet(1000L, 1050L);

// LongAdder: Cell striping (@Contended) for ultra-high multi-threaded increments
LongAdder metricCounter = new LongAdder();
metricCounter.increment();
long total = metricCounter.sum();
```

---

## ⚡ 2. Virtual Threads (JEP 444)

```java
// 1. Launch Virtual Thread per task (Unmounts on blocking I/O)
Thread.startVirtualThread(() -> {
    String res = restClient.get().retrieve().body(String.class); // Non-blocking carrier unmount!
});

// 2. Structured Concurrency (Java 21 Preview)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Supplier<User> user = scope.fork(() -> fetchUser(id));
    Supplier<Order> order = scope.fork(() -> fetchOrder(id));

    scope.join();           // Join both subtasks
    scope.throwIfFailed();  // Propagate errors

    return new Response(user.get(), order.get());
}
```

> [!WARNING]
> **Virtual Thread Pinning Hazard**: Never call blocking I/O inside a `synchronized` block. Use `ReentrantLock` instead.

---

## ⚡ 3. Advanced Locks: `StampedLock`

```java
StampedLock lock = new StampedLock();

// Optimistic Read (Zero CPU mutex locks!)
long stamp = lock.tryOptimisticRead();
double currentX = x, currentY = y;
if (!lock.validate(stamp)) { // If written concurrently, upgrade to pessimistic read
    stamp = lock.readLock();
    try {
        currentX = x;
        currentY = y;
    } finally {
        lock.unlockRead(stamp);
    }
}
```

---

## ⚡ 4. `ThreadPoolExecutor` Sizing & Rejection Policies

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    8,                      // Core Pool Size (CPUs)
    32,                     // Max Pool Size
    60L, TimeUnit.SECONDS,  // KeepAlive Time
    new ArrayBlockingQueue<>(1000), // Bounded Queue (Prevents OOM!)
    new ThreadPoolExecutor.CallerRunsPolicy() // Backpressure: Calling thread executes task!
);
```
