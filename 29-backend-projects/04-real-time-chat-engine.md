# Project 04: Real-Time Distributed Chat Platform

Build a horizontally scalable real-time chat platform supporting 1-on-1 direct messages, group chat channels, ephemeral user presence heartbeats, and Redis Pub/Sub inter-node broadcasting.

---

## 🗺️ System Architecture

```mermaid
flowchart TD
    ClientA["User Alice (Connected to Pod 1)"] -->|WebSocket TLS| Pod1["Chat Pod 1"]
    ClientB["User Bob (Connected to Pod 2)"] -->|WebSocket TLS| Pod2["Chat Pod 2"]

    Pod1 -->|1. Publish Message to Channel| RedisPubSub[("Redis Pub/Sub Backplane ('chat:room:42')")]
    RedisPubSub -->|2. Broadcast Message| Pod1 & Pod2

    Pod2 -->|3. Push over WebSocket Socket| ClientB

    Pod1 -->|4. Async Buffer Message| Kafka["Kafka Topic: 'chat-messages'"]
    Kafka --> MessageArchiver["Archival Worker"]
    MessageArchiver --> PostgresDB[("PostgreSQL / Cassandra Archive")]
```

---

## ⚡ Implementation: Spring Boot STOMP WebSocket & Redis Backplane

```java
package com.backend.engineering.projects.chat;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.listener.PatternTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

@Service
public class ChatBroadcastingService {

    private final StringRedisTemplate redisTemplate;
    private final SimpMessagingTemplate simpMessagingTemplate;

    public ChatBroadcastingService(StringRedisTemplate redisTemplate, SimpMessagingTemplate simpMessagingTemplate) {
        this.redisTemplate = redisTemplate;
        this.simpMessagingTemplate = simpMessagingTemplate;
    }

    // Publish message from local client to global Redis cluster
    public void publishMessageToRoom(String roomId, String messageJson) {
        redisTemplate.convertAndSend("chat:room:" + roomId, messageJson);
    }

    // Called when Redis Pub/Sub receives a broadcasted message from ANY pod
    public void onRedisMessageReceived(String messageJson, String topicChannel) {
        String roomId = topicChannel.replace("chat:room:", "");
        // Broadcast over local WebSocket connections connected to THIS pod
        simpMessagingTemplate.convertAndSend("/topic/rooms/" + roomId, messageJson);
    }
}
```
