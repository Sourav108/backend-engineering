# Backpressure, Load Shedding, and Adaptive Concurrency Limits

---

## 1. What Is It?
**Backpressure** is a flow-control mechanism where a downstream consumer or service signals to upstream producers to reduce their transmission rate when incoming traffic exceeds local processing capacity.

**Load Shedding** is the defensive practice of deliberately and proactively dropping or rejecting lower-priority incoming requests (returning HTTP `429 Too Many Requests` or `503 Service Unavailable`) to protect core system stability and preserve sub-millisecond latencies for active in-flight transactions.

---

## 2. Why Unlimited Concurrency Is Dangerous

### 1. Little's Law & The Latency Explosion
According to **Little's Law**:

$$L = \lambda \times W$$

- $L$: Average number of concurrent requests in the system.
- $\lambda$: Arrival rate of incoming requests (requests/second).
- $W$: Average response time (Latency).

If a backend service has a capacity to process $1,000\text{ req/sec}$ at $50\text{ms}$ latency:
- Concurrency $L = 1000 \times 0.05 = 50\text{ concurrent requests}$.
- If incoming traffic spikes to $10,000\text{ req/sec}$ without concurrency limits:
  $$W = \frac{L}{\lambda} \longrightarrow \text{Latency explodes to } 5,000\text{ms} \dots 30,000\text{ms}!$$

```mermaid
flowchart TD
    subgraph UnlimitedConcurrencyCollapse["The Unlimited Concurrency Death Spiral"]
        Surge["Traffic Surge (10x Normal)"] --> Buffer["Queue Fills with 50,000 Requests (Bufferbloat)"]
        Buffer --> DBExhaust["Database Connection Pool Exhausted (HikariCP Timeout)"]
        DBExhaust --> GC_Spike["JVM Garbage Collection Spikes (STW Pauses)"]
        GC_Spike --> ClientTimeout["Clients Timeout at 5s & Retry Automatically!"]
        ClientTimeout --> Surge
        Note over Surge,ClientTimeout: Total System Freeze: 100% CPU, 0% Successful Requests!
    end
```

---

## 3. Mental Model: Load Shedding vs Queue Bloat

```mermaid
flowchart LR
    subgraph BadArchitecture["Queue Bloat (Accept Everything)"]
        In1["10,000 req/s"] --> Q1["Queue (50k items)"] --> Svc1["Service (1k req/s)"]
        Note over BadArchitecture: Result: Every single user experiences 30s timeouts!
    end

    subgraph GoodArchitecture["Load Shedding (Bounded Capacity)"]
        In2["10,000 req/s"] --> Filter{"Limit: Max 50 In-Flight"}
        Filter -- 1,000 req/s Accepted --> Svc2["Service (Processes in 20ms!)"]
        Filter -- 9,000 req/s Shed --> Fast503["Instant HTTP 429/503 (< 1ms)"]
        Note over GoodArchitecture: Result: 1,000 users succeed instantly; system never crashes!
    end
```

$$\textbf{"It is better to serve 1,000 users with 20ms latency and reject the rest, than to fail 10,000 users with 30-second timeouts!"}$$

---

## 4. How Does It Work?

### Adaptive Concurrency Limits (The Netflix Gradient Algorithm)
Rather than configuring static, hardcoded concurrency limits (which become inaccurate as database load fluctuates), Netflix developed **Adaptive Concurrency Limits** based on TCP congestion algorithms (Vegas / Gradient):

$$\text{Gradient} = \frac{\text{RTT}_{\text{no-load}} (\text{Minimum Observed Latency})}{\text{RTT}_{\text{actual}} (\text{Current Moving Average Latency})}$$

$$\text{New Concurrency Limit} = \text{Current Limit} \times \text{Gradient} + \text{Headroom}$$

- **When Latency is Low ($\text{Gradient} \approx 1.0$)**: The algorithm smoothly increases the allowed concurrency limit.
- **When Latency Begins Spiking ($\text{Gradient} < 1.0$)**: The algorithm immediately contracts the allowed concurrency limit, shedding excess requests **before the database crashes!**

---

## 5. Implementation: Production Load Shedding Filter in Java 21

```java
package com.backend.engineering.concurrency.loadshedding;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.concurrent.atomic.AtomicInteger;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class AdaptiveLoadSheddingFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(AdaptiveLoadSheddingFilter.class);

    // Hard ceiling on max concurrent in-flight requests per pod
    private static final int MAX_IN_FLIGHT_REQUESTS = 100;
    private final AtomicInteger currentInFlight = new AtomicInteger(0);

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // 1. Health checks and internal metrics bypass load shedding
        String path = httpRequest.getRequestURI();
        if (path.startsWith("/actuator") || path.startsWith("/health")) {
            chain.doFilter(request, response);
            return;
        }

        // 2. Evaluate concurrency limit
        int inFlight = currentInFlight.incrementAndGet();

        if (inFlight > MAX_IN_FLIGHT_REQUESTS) {
            currentInFlight.decrementAndGet(); // Release slot
            log.warn("LOAD SHED: In-flight requests ({}) exceeded ceiling ({}). Rejecting with HTTP 503.",
                    inFlight, MAX_IN_FLIGHT_REQUESTS);

            httpResponse.setStatus(HttpServletResponse.SC_SERVICE_UNAVAILABLE); // HTTP 503
            httpResponse.setHeader("Retry-After", "2"); // Ask client to back off for 2 seconds
            httpResponse.setContentType("application/json");
            httpResponse.getWriter().write("{\"error\": \"Server is overloaded. Load shedding active. Please retry shortly.\"}");
            return;
        }

        try {
            // 3. Process valid request within safety bounds
            chain.doFilter(request, response);
        } finally {
            currentInFlight.decrementAndGet(); // Always release slot on exit!
        }
    }
}
```

---

## 6. Performance

| Traffic Condition | System Without Load Shedding | System With Adaptive Load Shedding |
|---|---|---|
| Steady State ($1,000\text{ req/s}$) | Latency: $20\text{ms}$, Success: $100\%$ | Latency: $20\text{ms}$, Success: $100\%$ |
| Traffic Spike ($10,000\text{ req/s}$) | **Latency: $> 30\text{s}$, Success: $0\%$ (Crash)** | **Latency: $22\text{ms}$, Success: $1,000\text{ req/s}$ ($100\%$ of capacity)** |

---

## 7. Interview Questions

### Q1: What is "Bufferbloat" and why do oversized task queues harm distributed backend systems?
<details>
<summary>Reveal Answer</summary>

**Answer**:
**Bufferbloat** occurs when systems configure massive or unbounded queues between components to "prevent dropping requests."
When a sudden traffic spike occurs:
1. Requests sit waiting in the queue for 10–30 seconds before worker threads even pick them up.
2. Meanwhile, upstream HTTP clients and API Gateways reach their client timeout (e.g. 5 seconds) and abort the connection.
3. When the worker thread finally picks up the request from the queue 15 seconds later, **it spends expensive CPU and database resources executing work for a client that has already disconnected and abandoned the request!**
Oversized queues convert transient traffic surges into permanent latency degradation. Best practice: **Keep in-memory queues small and tightly bounded, shedding excess traffic immediately with fast 503/429 errors**.
</details>

---

## 8. Quick Revision
- **Little's Law**: $L = \lambda W$; unbounded concurrency causes latency explosion.
- **Load Shedding**: Fast rejection of excess traffic to preserve capacity for accepted requests.
- **Adaptive Concurrency Limits**: Dynamically scales allowed in-flight requests based on moving-average RTT latency.
- **Queue Bounds**: Never allow deep queues; drop or shed fast rather than buffering for seconds.
- **HTTP 503 + `Retry-After`**: Informs clients to back off gracefully during load shedding.
