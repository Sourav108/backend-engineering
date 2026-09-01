# Spring Boot STOMP WebSockets, Broker Relays, and Message Ordering

---

## 1. What Is It?
**STOMP (Simple Text Oriented Messaging Protocol)** is an application-level sub-protocol running on top of raw WebSocket frames that introduces structured message semantics, including **Verbs** (`CONNECT`, `SUBSCRIBE`, `SEND`, `MESSAGE`), **Headers**, and **Destination Routing (`/topic/...`, `/queue/...`)**.

Spring Boot provides first-class support for STOMP via `spring-websocket` and `spring-messaging`, allowing developers to write declarative message-driven controllers (`@MessageMapping`).

---

## 2. Why Does It Exist?
Raw WebSockets provide only raw byte/text frames without application-level routing:
- With raw WebSockets, every development team must invent their own custom JSON protocol for subscribing to rooms, acknowledging messages, and handling routing prefixes.
- **STOMP standardizes framing**: It allows frontend clients (e.g. `@stomp/stompjs`) to subscribe directly to destination topics (`/topic/room.42`) and allows the backend to route messages cleanly.

---

## 3. Mental Model: Spring STOMP Internal Broker vs External Broker Relay

```mermaid
flowchart TD
    subgraph SpringInternal["1. Simple In-Memory STOMP Broker (Single Pod Only)"]
        ClientA["Browser A"] -->|SUBSCRIBE /topic/news| InMem["Spring SimpleBroker (RAM)"]
        InMem -->|Push| ClientA
        Note over SpringInternal: In-memory only! Cannot broadcast across multiple pods!
    end

    subgraph ExternalRelay["2. Full External Broker Relay (Production Multi-Pod Cluster)"]
        ClientB["Browser B (Pod 1)"] --> STOMP_Relay["Spring StompBrokerRelay"]
        STOMP_Relay <-->|TCP Port 61613 (STOMP)| RabbitMQ[("Clustered RabbitMQ / ActiveMQ Broker")]
        RabbitMQ <--> STOMP_Relay2["Spring StompBrokerRelay (Pod 2)"]
        STOMP_Relay2 --> ClientC["Browser C (Pod 2)"]
        Note over ExternalRelay: Enterprise-grade message distribution, clustering, and persistence!
    end
```

---

## 4. How Does It Work?

### 1. The STOMP Frame Structure
A STOMP frame mirrors HTTP syntax over a WebSocket connection:

```text
SEND
destination:/app/chat.send
content-type:application/json
receipt:msg-101

{"roomId":"room_42","text":"Hello everyone!"}
^@
```
- `SEND`: The STOMP command.
- `destination`: The routing target.
- `^@`: Null byte marking the end of the frame.

---

### 2. Guaranteed In-Order Message Delivery
In high-concurrency real-time chat:
- If a user sends 3 messages rapidly, network packet reordering can cause Message 3 to arrive before Message 2.
- **Ordering Guarantee Protocol**:
  1. Each conversation room maintains a **Monotonic Sequence Counter** in Redis (`INCR room:42:seq`).
  2. Every outgoing message is stamped with a strict monotonic `sequence_id` (e.g. `101, 102, 103`).
  3. The client browser buffers and sorts incoming messages by `sequence_id`, guaranteeing **$100\%$ perfectly ordered conversation rendering**.

---

## 5. Implementation: Spring Boot 3 STOMP Broker Relay Configuration

```java
package com.backend.engineering.realtime.stomp;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketBrokerConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // 1. Application Ingress Prefix: Routes to @MessageMapping controllers
        registry.setApplicationDestinationPrefixes("/app");

        // 2. Production Clustered External Broker Relay (RabbitMQ STOMP Plugin)
        registry.enableStompBrokerRelay("/topic", "/queue")
                .setRelayHost("rabbitmq.internal")
                .setRelayPort(61613) // Standard STOMP port
                .setClientLogin("guest")
                .setClientPasscode("guest")
                .setSystemLogin("guest")
                .setSystemPasscode("guest");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // WebSocket Ingress Endpoint with SockJS Fallback for legacy browsers
        registry.addEndpoint("/ws-connect")
                .setAllowedOrigins("https://app.company.com")
                .withSockJS();
    }
}
```

---

### Declarative Message Controller

```java
package com.backend.engineering.realtime.stomp;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.DestinationVariable;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class StompChatController {

    private static final Logger log = LoggerFactory.getLogger(StompChatController.class);
    private final SimpMessagingTemplate messagingTemplate;

    public StompChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/chat.send.{roomId}")
    public void handleIncomingChatMessage(@DestinationVariable("roomId") String roomId, IncomingMessageDto message) {
        log.info("Received STOMP chat message for Room {}: {}", roomId, message.text());

        OutboundMessageDto outbound = new OutboundMessageDto(
                roomId,
                message.senderId(),
                message.text(),
                System.currentTimeMillis()
        );

        // Broadcast to all clients subscribed to /topic/room.<roomId> across the entire cluster!
        messagingTemplate.convertAndSend("/topic/room." + roomId, outbound);
    }

    public record IncomingMessageDto(String senderId, String text) {}
    public record OutboundMessageDto(String roomId, String senderId, String text, long timestamp) {}
}
```

---

## 6. Performance

| Broker Architecture | Multi-Pod Clustering | Max Message Throughput | Persistence / Dead Lettering |
|---|---|---|:---:|
| Spring `SimpleBroker` (In-Memory) | ❌ Broken on Multi-Pod | $\approx 10,000\text{ msg/s}$ | ❌ No |
| **RabbitMQ STOMP Broker Relay** | ✅ **Clustered Scale** | **$> 250,000\text{ msg/s}$** | ✅ **Full Durability & DLQ** |

---

## 7. Interview Questions

### Q1: Why should you replace Spring's built-in `SimpleBroker` with an external STOMP broker relay (e.g. RabbitMQ) in production?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Multi-Node Clustering & Inter-Pod Communication**:
   - The default `SimpleBroker` is a purely in-memory message broker running inside the single JVM heap of the Spring Boot process.
   - If your application scales to 10 pods behind a load balancer, a message sent by User A on Pod 1 to `/topic/announcements` will **only be delivered to clients connected to Pod 1**. Clients connected to Pods 2 through 10 will **never receive the message**.
2. **Dedicated External Broker Relay (RabbitMQ / ActiveMQ)**:
   - Spring connects as a client to an external RabbitMQ cluster.
   - When any pod broadcasts to `/topic/...`, RabbitMQ handles the distributed pub/sub routing across all 10 Spring Boot pods, guaranteeing cluster-wide delivery.
   - External brokers provide production features including message persistence, flow control, rate limiting, and Dead Letter Queues (DLQs).
</details>

---

## 8. Quick Revision
- **STOMP**: Standardized messaging sub-protocol over WebSockets with `SEND`, `SUBSCRIBE`, and destinations.
- **`SimpleBroker` Flaw**: Works only on a single node; cannot broadcast across multi-pod clusters.
- **External Broker Relay**: Delegates pub/sub routing to clustered RabbitMQ / ActiveMQ.
- **`SimpMessagingTemplate`**: Spring utility for broadcasting messages to STOMP destinations.
- **Message Ordering**: Stamp messages with monotonic sequence IDs in Redis to guarantee client-side ordering.
