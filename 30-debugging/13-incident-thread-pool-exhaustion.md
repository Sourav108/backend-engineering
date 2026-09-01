# Incident 13: Thread Pool Exhaustion (Queue-First CallerRuns Saturation)

---

## 1. Symptoms & Alert
- **Alert**: `TomcatWorkerThreadsExhausted: 200/200 active threads`.
- **Customer Impact**: Ingress API Gateway timing out; health check endpoints `/actuator/health` failing to respond, causing Kubernetes to restart healthy pods.

---

## 2. Metric & Telemetry Anomalies
- **Micrometer Metrics**: `executor.active` and `tomcat.threads.busy` flatlined at $100\%$.
- **Response Latency**: Baseline $15\text{ms}$ latency spiked to $> 30,000\text{ms}$.

---

## 3. Diagnostic Steps & Root Cause
### Inspecting Custom Async Executor Configuration:
```java
@Bean
public Executor asyncTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(32);
    executor.setQueueCapacity(50); // Small queue!
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy()); // DANGER!
    return executor;
}
```

### The CallerRuns Saturation Cascade:
1. Under a traffic burst, the background async queue filled up ($> 50$ tasks).
2. The `CallerRunsPolicy` kicked in: **The calling Tomcat HTTP request thread was forced to execute the heavy 5-second background report task directly**.
3. As more incoming HTTP requests arrived, more Tomcat request threads were hijacked to run background tasks.
4. Within 30 seconds, all 200 Tomcat worker threads were trapped running heavy background computations, **preventing the server from processing lightweight `/health` probes or new API requests**.

---

## 4. Immediate Mitigation
Scale out application deployment to 10 pods to absorb load:
```bash
kubectl scale deployment reporting-service -n production --replicas=10
```

---

## 5. Permanent Fix
1. Replace `CallerRunsPolicy` on HTTP-adjacent pools with **`AbortPolicy`** (throwing a rejected exception to return `HTTP 429 Too Many Requests` or `HTTP 503` immediately):
   ```java
   executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
   ```
2. Offload heavy background tasks to an external asynchronous **Queue-Worker Architecture (Kafka/SQS)** with dedicated worker pods.

---

## 6. Postmortem Action Items
- [x] Separate internal async thread pools from external HTTP request processing pools.
- [x] Configure KEDA to auto-scale worker pods based on queue depth.
