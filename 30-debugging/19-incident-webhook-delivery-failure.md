# Incident 19: Webhook Delivery Failure (Slow Consumer Head-of-Line Blocking)

---

## 1. Symptoms & Alert
- **Alert**: `WebhookDeliveryLatencyP99 > 3600s (1 Hour Lag)`.
- **Customer Impact**: Third-party merchant webhook events (payment confirmations, refund updates) delayed by hours.

---

## 2. Metric & Telemetry Anomalies
- **Worker Metrics**: Webhook delivery worker thread pool utilization at $100\%$.
- **Queue Metrics**: Inbound webhook dispatch queue growing exponentially ($> 250,000$ queued events).

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Analyze Destination Host Latency Distribution
```bash
# Check worker logs for slow response times
kubectl logs -n production -l app=webhook-worker --tail=500 | grep "Delivered" | awk '{print $NF}' | sort -n | tail -n 10
# Output:
# Multiple webhooks to 'https://slow-merchant-endpoint.com' taking 30,000ms (30s HTTP timeout)!
```

### Root Cause:
1. One high-volume merchant misconfigured their webhook receiving server, causing it to hang for **$30\text{ seconds}$** before responding.
2. Because the webhook engine processed events on a **single shared queue without tenant-level fair queueing or per-host concurrency limits**:
3. All worker threads were hijacked by the single slow merchant's hanging requests, causing severe **Head-of-Line Blocking** that starved all other 5,000 healthy merchants on the platform.

---

## 4. Immediate Mitigation
1. **Temporarily Disable Webhook Delivery to the Slow Merchant**:
   ```sql
   UPDATE merchant_webhook_configs SET is_enabled = false WHERE merchant_id = 'merchant_slow_99';
   ```
2. **Scale Out Webhook Worker Pods**:
   ```bash
   kubectl scale deployment webhook-worker -n production --replicas=20
   ```

---

## 5. Permanent Fix
1. **Enforce Strict 5-Second HTTP Client Timeouts**:
   ```java
   HttpClient client = HttpClient.newBuilder()
           .connectTimeout(Duration.ofSeconds(2))
           .build();
   ```
2. **Implement Per-Destination Domain Bulkheads & Separate Priority Queues**:
   - Limit concurrent in-flight HTTP requests to any single destination host (`maxConcurrentPerHost = 5`).
   - If a host exceeds its concurrency limit, immediately defer its webhooks to a dedicated **Slow/Retry Queue**, preventing interference with fast merchants.

---

## 6. Postmortem Action Items
- [x] Configure Weighted Fair Queueing (WFQ) across merchant webhook topics.
- [x] Implement automated merchant circuit breaker (tripping after 10 consecutive timeouts).
