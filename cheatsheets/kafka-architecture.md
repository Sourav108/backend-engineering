# Apache Kafka Architecture & Tuning Cheat Sheet

---

## ⚡ 1. Production Producer Settings

```properties
# Durability & Exactly-Once Semantics (EOS)
acks=all
min.insync.replicas=2
enable.idempotence=true
retries=2147483647
max.in.flight.requests.per.connection=5

# High-Throughput Batching & Compression
compression.type=zstd
linger.ms=20
batch.size=65536 # 64 KB buffer
buffer.memory=33554432 # 32 MB total accumulator
```

---

## ⚡ 2. Production Consumer Settings

```properties
# Consumer Group & Rebalance Tuning
group.id=order-processing-group
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
enable.auto.commit=false
auto.offset.reset=earliest
max.poll.interval.ms=300000 # 5 minutes
max.poll.records=500
session.timeout.ms=45000
heartbeat.interval.ms=15000
```

---

## ⚡ 3. Spring Kafka Virtual Thread Listener & Non-Blocking Retries

```java
@Configuration
public class KafkaListenerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            ConsumerFactory<String, Object> consumerFactory) {
        var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        // Bind consumer listener threads to Java 21 Virtual Threads!
        factory.getContainerProperties().setListenerTaskExecutor(new SimpleAsyncTaskExecutor(Thread.ofVirtual().factory()));
        return factory;
    }
}
```

```java
// Non-blocking retry with Dead Letter Topic (DLT)
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
    dltStrategy = DltStrategy.FAIL_ON_ERROR
)
@KafkaListener(topics = "orders")
public void handleOrder(OrderEvent event, Acknowledgment ack) {
    processOrder(event);
    ack.acknowledge();
}
```
