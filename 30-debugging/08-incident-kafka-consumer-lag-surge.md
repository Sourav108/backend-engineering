# Incident 08: Kafka Consumer Lag Surge (Poison Pill Crash Loop)

---

## 1. Symptoms & Alert
- **Alert**: `KafkaConsumerGroupLag > 100,000 records on topic 'payment-notifications'`.
- **Customer Impact**: Order confirmation emails and push notifications delayed by over 4 hours.

---

## 2. Metric & Telemetry Anomalies
- **Kafka Lag**: Lag on Partition 4 exploded from 10 to **125,000 records**, while all other partitions had 0 lag.
- **Consumer Rebalances**: Consumer group was undergoing continuous **Rebalance Storms** every 5 minutes (`CommitFailedException`).

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Describe Kafka Consumer Group
```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group notification-workers
```
- *Output*: Partition 4 offset was frozen at `Offset: 458920` with zero movement.

### Step 2: Inspect Worker Logs on Partition 4
```bash
kubectl logs -n production -l app=notification-worker | grep -C 5 "Exception"
```
- *Log Output*:
```text
java.lang.NullPointerException: Cannot invoke "String.toLowerCase()" because "email" is null
	at com.backend.notification.TemplateEngine.render(TemplateEngine.java:42)
org.apache.kafka.clients.consumer.CommitFailedException: 
Offset commit cannot be completed since the consumer exceeded max.poll.interval.ms (300000ms)
```

### Root Cause:
1. Message `Offset: 458920` contained a malformed JSON payload with `email: null` (**A Poison Pill**).
2. The consumer threw an unhandled `NullPointerException` and crashed.
3. The consumer failed to call `commitSync()`, exceeded `max.poll.interval.ms (5 mins)`, was evicted from the group, triggered a cluster-wide rebalance, and the new assigned consumer picked up the **exact same poison pill** and crashed again in an infinite loop!

---

## 4. Immediate Mitigation
Seek consumer group offset past the single poison pill message:
```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group notification-workers \
  --reset-offsets --to-offset 458921 \
  --topic payment-notifications:4 \
  --execute
```

---

## 5. Permanent Fix
Configure Spring Kafka with `@RetryableTopic` and an automated **Dead Letter Topic (DLT)**:

```java
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2.0),
    dltStrategy = DltStrategy.FAIL_ON_ERROR, // Routes to 'payment-notifications-dlt' on 3rd failure!
    include = { Exception.class }
)
@KafkaListener(topics = "payment-notifications")
public void handleNotification(NotificationEvent event, Acknowledgment ack) {
    notificationService.sendNotification(event);
    ack.acknowledge();
}
```

---

## 6. Postmortem Action Items
- [x] Configure schema registry validation on Kafka producers to reject null emails at ingress.
- [x] Deploy Dead Letter Topic consumer to alert on poisoned records.
