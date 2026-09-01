# Incident 15: Disk Full and Container Root Volume Bloat (`KubeletHasDiskPressure`)

---

## 1. Symptoms & Alert
- **Alert**: `KubeletHasDiskPressure` on Kubernetes worker nodes; pods transitioning to `Evicted` status.
- **Customer Impact**: Application pods randomly evicted; new pods unable to schedule onto nodes.

---

## 2. Metric & Telemetry Anomalies
- **Node Storage**: Host root disk partition `/` hit $100\%$ capacity ($0\text{ bytes free}$).
- **Disk I/O**: High write latency on NVMe disk.

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Check Largest Disk Space Consumers on Host Node
```bash
du -sh /var/log/pods/* | sort -hr | head -n 10
# Output:
# 84G  /var/log/pods/production_payment-service-89f4b-2q9w/
```

### Step 2: Inspect Container Logging Output
```bash
tail -n 20 /var/log/pods/production_payment-service-89f4b-2q9w/payment-app/0.log
# Output:
# DEBUG [http-nio-8080-exec-12] c.b.e.p.SecurityFilter: Full Raw Request Body: {"card_number":"4111...", ...}
```

### Root Cause:
1. A developer accidentally left `logging.level.root = DEBUG` enabled in production.
2. The payment service was logging every inbound raw payload to `stdout` at 5,000 QPS, generating **84 Gigabytes of plaintext log files in 6 hours**, filling the entire host root disk.

---

## 4. Immediate Mitigation
1. **Truncate Oversized Log Files to Clear Disk Pressure**:
   ```bash
   truncate -s 0 /var/log/pods/production_payment-service-*/*/*.log
   ```
2. **Dynamically Change Logging Level to `INFO` via Spring Boot Actuator**:
   ```bash
   curl -X POST http://payment-service:8080/actuator/loggers/ROOT \
     -H "Content-Type: application/json" \
     -d '{"configuredLevel": "INFO"}'
   ```

---

## 5. Permanent Fix
1. Configure container runtime log rotation in Kubernetes `kubelet` configuration:
   ```yaml
   containerLogMaxSize: "50Mi"
   containerLogMaxFiles: 5
   ```
2. Mask sensitive PII/PCI fields in logback configuration and enforce `INFO` default in `application-prod.yml`.

---

## 6. Postmortem Action Items
- [x] Configure CloudWatch Logs agent to ship logs to S3 and purge local container logs.
- [x] Add CI/CD check failing builds if `DEBUG` logging is enabled in production profiles.
