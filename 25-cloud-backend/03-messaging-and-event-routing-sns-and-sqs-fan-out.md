# Cloud Messaging: SNS-to-SQS Fan-Out, FIFO Queues, and DLQs

---

## 1. What Is It?
In AWS cloud architectures:
- **Amazon SNS (Simple Notification Service)** is a fully managed, high-throughput **Pub/Sub (Publish/Subscribe)** messaging service designed for message broadcasting.
- **Amazon SQS (Simple Queue Service)** is a fully managed, distributed **Message Queue** service designed for point-to-point task consumption and asynchronous worker buffering.
- **The SNS-to-SQS Fan-Out Pattern** connects a single SNS topic to multiple independent SQS queues, delivering a copy of every published message to multiple autonomous microservices concurrently.

---

## 2. Why Does It Exist?
Without the Fan-Out pattern:
- When an `OrderPlaced` event occurs, the Order Service must synchronously invoke 5 separate microservice APIs (Fraud Service, Email Service, Analytics Service, Inventory Service, Fulfillment Service).
- If the Analytics Service is down, the entire order creation fails or times out (**Tight Coupling**).
- By publishing **1 event to an SNS topic**, SNS automatically replicates and pushes the message to 5 separate SQS queues, achieving **$100\%$ decoupled event distribution**.

---

## 3. Mental Model: The SNS-to-SQS Fan-Out Pattern

```mermaid
flowchart TD
    OrderService["Order Service (Publisher)"] -->|1. Publish 'OrderPlaced' Event| SNSTopic["AWS SNS Topic: 'orders-topic'"]
    
    SNSTopic -->|Fan-Out Copy| SQS_Fraud["SQS Queue: 'fraud-verification'"]
    SNSTopic -->|Fan-Out Copy| SQS_Inventory["SQS Queue: 'inventory-reserve'"]
    SNSTopic -->|Fan-Out Copy (Filtered)| SQS_VIP["SQS Queue: 'vip-concierge' (Filter: tier == 'VIP')"]

    SQS_Fraud --> FraudWorker["Fraud Service Workers"]
    SQS_Inventory --> InventoryWorker["Inventory Service Workers"]
    SQS_VIP --> VIPWorker["VIP Concierge Workers"]
```

---

## 4. How Does It Work?

### 1. SQS Standard vs FIFO Queues

| Feature | SQS Standard Queue | SQS FIFO Queue (`.fifo`) |
|---|---|---|
| **Ordering Guarantee** | Best-effort ordering (Messages may arrive out of order) | **Strict FIFO (First-In, First-Out Guaranteed)** |
| **Delivery Guarantee** | **At-Least-Once** (Occasional duplicates possible) | **Exactly-Once Processing** (via Deduplication ID) |
| **Max Throughput** | **Virtually Unlimited TPS** | $3,000\text{ TPS}$ with batching ($300\text{ TPS}$ without) |
| **Partitioning Key** | None | `MessageGroupId` (Guarantees ordering per user/order) |

---

### 2. SQS Long Polling vs Short Polling
- **Short Polling (`WaitTimeSeconds = 0`)**: SQS queries a random subset of servers and returns immediately, often returning empty responses and driving up AWS API billing costs.
- **Long Polling (`WaitTimeSeconds = 20`)**: **Production Standard**. SQS waits up to $20\text{ seconds}$ for a message to arrive before returning, reducing empty receives by **$99\%$** and cutting AWS billing costs.

---

## 5. Implementation: SNS-to-SQS Fan-Out & SQS Listener in Java 21

### 1. Spring Cloud AWS SQS Listener (Spring Boot 3.3.4)
```java
package com.backend.engineering.cloud.messaging;

import io.awspring.cloud.sqs.annotation.SqsListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class InventoryOrderEventListener {

    private static final Logger log = LoggerFactory.getLogger(InventoryOrderEventListener.class);

    // Listens to SQS queue with Long Polling enabled by default
    @SqsListener(value = "inventory-order-processing-queue", maxConcurrentMessages = "10")
    public void processOrderEvent(OrderPlacedPayload payload) {
        log.info("Received OrderPlaced event from SQS. OrderID: {}, UserID: {}", 
                 payload.orderId(), payload.userId());

        // Process inventory allocation logic
        allocateInventory(payload.orderId(), payload.items());
    }

    private void allocateInventory(Long orderId, Object items) {
        // Business logic
    }

    public record OrderPlacedPayload(Long orderId, Long userId, String tier, Object items) {}
}
```

---

### 2. Publishing to SNS with Message Attributes (Subscription Filtering)
```java
package com.backend.engineering.cloud.messaging;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.sns.SnsClient;
import software.amazon.awssdk.services.sns.model.MessageAttributeValue;
import software.amazon.awssdk.services.sns.model.PublishRequest;

import java.util.Map;

@Service
public class SnsOrderPublisherService {

    private static final Logger log = LoggerFactory.getLogger(SnsOrderPublisherService.class);
    private static final String TOPIC_ARN = "arn:aws:sns:us-east-1:123456789012:orders-topic";
    private final SnsClient snsClient;

    public SnsOrderPublisherService(SnsClient snsClient) {
        this.snsClient = snsClient;
    }

    public void publishOrder(String orderJson, String customerTier) {
        PublishRequest request = PublishRequest.builder()
                .topicArn(TOPIC_ARN)
                .message(orderJson)
                .messageAttributes(Map.of(
                    "CustomerTier", MessageAttributeValue.builder()
                            .dataType("String")
                            .stringValue(customerTier)
                            .build()
                ))
                .build();

        snsClient.publish(request);
        log.info("Published order event to SNS with CustomerTier attribute: {}", customerTier);
    }
}
```

---

## 6. Performance

| Pattern | Max Subscribed Consumers | Impact of 1 Slow Consumer | Delivery Latency |
|---|---|---|:---:|
| Direct Sync HTTP Calls | Hard to scale ($> 10$ services) | Cascading timeout failure | $50 - 500\text{ms}$ |
| **SNS-to-SQS Fan-Out** | **$12,500,000$ SQS Subscriptions** | **Zero (Buffered in SQS)** | **$< 25\text{ms}$** |

---

## 7. Interview Questions

### Q1: How does the SNS-to-SQS Fan-Out pattern handle slow or offline consumer microservices without losing data?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Queue Isolation**: In the SNS-to-SQS Fan-Out pattern, each downstream microservice has its own dedicated, private SQS queue subscribed to the shared SNS topic.
2. **Asynchronous Buffering**: When the publisher emits an event to SNS, SNS delivers a discrete copy to each subscribed SQS queue in parallel.
3. **Fault Tolerance**: If the Email Notification service is completely offline or taking 5 minutes to process an email, messages safely accumulate in its dedicated SQS queue (which can hold millions of messages for up to 14 days).
4. **Zero Blast Radius**: The slow Email Service's backlog has **zero impact** on the Fraud Service or Inventory Service, which continue pulling and processing messages from their own independent queues with zero delay.
</details>

---

## 8. Quick Revision
- **SNS**: Pub/Sub topic; broadcasts events to multiple subscribers.
- **SQS**: Message queue; buffers tasks for worker pools.
- **Fan-Out Pattern**: 1 SNS Topic $\longrightarrow$ Many SQS Queues.
- **SQS FIFO**: Exactly-once processing and strict ordering via `MessageGroupId`.
- **Long Polling (`20s`)**: Drastically reduces empty API responses and AWS billing costs.
