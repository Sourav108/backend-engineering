# Real-Time Protocols: Short Polling vs Long Polling vs SSE vs WebSockets

---

## 1. What Is It?
**Real-Time Backend Communication** is the set of network protocols and server architectures that enable servers to push data to connected clients (web browsers, mobile apps) with near-zero latency ($< 50\text{ms}$) the instant a state change occurs.

The 4 core real-time communication protocols are:
1. **Short Polling**: Client repeatedly fires regular HTTP requests on a fixed timer.
2. **Long Polling (Comet)**: Client sends an HTTP request; the server **holds the request open** until new data arrives or a timeout occurs.
3. **Server-Sent Events (SSE)**: Unidirectional, persistent HTTP/2 text stream where the server pushes events to the client.
4. **WebSockets**: Full-duplex, bidirectional, persistent binary TCP connection established via an HTTP Upgrade handshake (`101 Switching Protocols`).

---

## 2. Why Does It Exist?
Standard HTTP/1.1 is strictly **Unidirectional and Client-Initiated** (the server cannot initiate communication to a client without a prior request).

- **Short Polling Disaster**: 100,000 active mobile apps polling every $2\text{ seconds}$ generate **$50,000\text{ req/sec}$ of empty HTTP traffic**, wasting gigabytes of bandwidth in redundant HTTP headers (`Authorization`, `Cookie`, `User-Agent`) and saturating database connections.

---

## 3. Mental Model: The 4 Real-Time Protocols

```mermaid
flowchart TD
    subgraph ShortPoll["1. Short Polling (HTTP/1.1)"]
        C1["Client"] -->|GET /updates (Every 2s)| S1["Server (99% Empty 200 OK)"]
    end

    subgraph LongPoll["2. Long Polling (Hanging GET)"]
        C2["Client"] -->|GET /updates| S2["Server holds connection open (30s) until event occurs!"]
    end

    subgraph SSE["3. Server-Sent Events (Unidirectional HTTP/2)"]
        C3["Client"] -->|GET /events (Accept: text/event-stream)| S3["Server streams continuous SSE chunks over 1 socket"]
    end

    subgraph WS["4. WebSockets (Full-Duplex Bidirectional TCP)"]
        C4["Client"] <-->|Bidirectional Frames (0 HTTP Overhead)| S4["Server (ws://)"]
    end
```

---

## 4. Comprehensive Protocol Comparison

| Dimension | Short Polling | Long Polling | Server-Sent Events (SSE) | WebSockets |
|---|---|---|---|---|
| **Directionality** | Client $\to$ Server | Client $\to$ Server | **Server $\to$ Client (Unidirectional)** | **Full-Duplex (Bidirectional)** |
| **Transport Protocol** | HTTP/1.1 or HTTP/2 | HTTP/1.1 | **HTTP/2 (Standard HTTP)** | **Raw TCP (WebSocket Frame RFC 6455)** |
| **Header Overhead** | Extreme ($1\text{KB}$ per poll) | High (Re-connects on event) | **Zero (Single HTTP handshake)** | **Zero ($\approx 2 - 6\text{ bytes/frame}$)** |
| **Firewall / Proxy Traversal** | Trivial (Standard HTTP) | Trivial | **Native (Works over standard port 443 HTTPS)** | Requires proxy WebSocket Upgrade support |
| **Binary Data Support** | Base64 encoded | Base64 encoded | ❌ Text/UTF-8 only | ✅ **Native Binary (ArrayBuffer/Blob)** |
| **Best Use Case** | Low-frequency status checks | Legacy fallback | **Stock tickers, Live dashboards, AI chat streaming** | **Chat apps, Multiplayer gaming, Collaborative whiteboards** |

---

## 5. Implementation: Server-Sent Events (SSE) in Spring Boot 3.3.4

For unidirectional real-time data streams (like LLM token generation or real-time trading price feeds), **SSE is vastly simpler to implement, operate, and scale than WebSockets**:

```java
package com.backend.engineering.realtime.sse;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;

@RestController
public class LiveCryptoPriceSseController {

    // Store active SSE client emitters
    private final Map<String, SseEmitter> activeEmitters = new ConcurrentHashMap<>();

    @GetMapping(value = "/api/v1/stream/prices/{symbol}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamPrices(@PathVariable("symbol") String symbol) {
        // Create emitter with 30-minute timeout
        SseEmitter emitter = new SseEmitter(1800_000L);
        String clientId = symbol + ":" + System.nanoTime();
        activeEmitters.put(clientId, emitter);

        // Lifecycle Cleanups
        emitter.onCompletion(() -> activeEmitters.remove(clientId));
        emitter.onTimeout(() -> activeEmitters.remove(clientId));
        emitter.onError(ex -> activeEmitters.remove(clientId));

        return emitter;
    }

    // Broadcast price update to all connected SSE clients
    public void broadcastPrice(String symbol, double newPrice) {
        activeEmitters.forEach((clientId, emitter) -> {
            if (clientId.startsWith(symbol)) {
                try {
                    emitter.send(SseEmitter.event()
                            .name("price-update")
                            .data(Map.of("symbol", symbol, "price", newPrice, "timestamp", System.currentTimeMillis())));
                } catch (IOException e) {
                    activeEmitters.remove(clientId);
                }
            }
        });
    }
}
```

---

## 6. Performance

| Protocol | Server Bandwidth ($100\text{k Active Users}$) | Client Battery Impact | Server Memory Overhead |
|---|---|---|---|
| Short Polling ($2\text{s Interval}$) | **$\mathbf{\approx 50\text{MB/sec}}$ (Header Waste)** | **High (Constant radio wakeups)** | Low |
| Long Polling | $\approx 15\text{MB/sec}$ | Moderate | Moderate (Hanging connections) |
| **Server-Sent Events (SSE)** | **$< 0.2\text{MB/sec}$** | **Ultra-Low** | **$< 50\text{MB}$ total** |
| **WebSockets** | **$< 0.1\text{MB/sec}$** | **Ultra-Low** | **$< 35\text{MB}$ total** |

---

## 7. Interview Questions

### Q1: When should an architect choose Server-Sent Events (SSE) over WebSockets?
<details>
<summary>Reveal Answer</summary>

**Answer**:
**Choose Server-Sent Events (SSE) When**:
1. **Unidirectional Server-to-Client Streaming**: Data flows exclusively from server to client (e.g. LLM ChatGPT token streaming, live stock ticker prices, sports scoreboards, background job progress notifications).
2. **Standard HTTP/2 Infrastructure**: SSE runs over standard HTTP/2, seamlessly bypassing corporate firewalls, API gateways, load balancers, and CDN edge proxies without requiring special WebSocket protocol upgrade configurations.
3. **Built-in Auto-Reconnection & Event IDs**: The browser's native `EventSource` API automatically reconnects on network dropouts and sends `Last-Event-ID` headers to resume missed events seamlessly.
**Choose WebSockets When**:
- **True Bidirectional Full-Duplex Communication**: Both client and server continuously exchange messages with low latency (e.g. multiplayer online games, bi-directional collaboration tools like Google Docs, or real-time P2P chat).
</details>

---

## 8. Quick Revision
- **Short Polling**: Wasteful HTTP header overhead; avoid in production.
- **Long Polling**: Hanging GET; legacy fallback.
- **Server-Sent Events (SSE)**: Unidirectional server-to-client streaming over HTTP/2.
- **WebSockets**: Full-duplex bidirectional framing over persistent TCP.
- **SSE Sweet Spot**: AI chat streams, live prices, and progress bars.
