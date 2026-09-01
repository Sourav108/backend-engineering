# Incident 17: DNS Resolution Failure in Kubernetes (`UnknownHostException`)

---

## 1. Symptoms & Alert
- **Alert**: `java.net.UnknownHostException: postgres.production.svc.cluster.local` spiking across all application pods.
- **Customer Impact**: Microservices unable to connect to PostgreSQL, Redis, or peer services; sporadic intermittent 500 errors across cluster.

---

## 2. Metric & Telemetry Anomalies
- **CoreDNS Metrics**: `coredns_dns_request_duration_seconds` jumped from $< 1\text{ms}$ to $> 5,000\text{ms}$; `coredns_dns_responses_total{rcode="SERVFAIL"}` spiked.
- **CoreDNS Pods**: CPU utilization on the 2 CoreDNS pods reached $100\%$.

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Inspect CoreDNS Deployment & Logs
```bash
kubectl get deployment coredns -n kube-system
# Output: Replicas: 2/2 (Under-provisioned for a 500-pod cluster!)

kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
# Output: [WARNING] plugin/loop: Seen "HINFO" loop for "."
```

### Root Cause:
1. The Kubernetes cluster had scaled from 50 pods to 500 pods, but CoreDNS remained at **2 static replicas**.
2. Because applications queried DNS on every single HTTP and database call (due to Java JVM DNS caching TTL set to 0), CoreDNS was bombarded with **$85,000\text{ DNS queries/sec}$**, saturating CoreDNS CPU and dropping DNS lookup packets.

---

## 4. Immediate Mitigation
Scale CoreDNS replicas immediately to absorb DNS query volume:
```bash
kubectl scale deployment coredns -n kube-system --replicas=10
```

---

## 5. Permanent Fix
1. **Deploy NodeLocal DNSCache**:
   - Runs a lightweight DNS cache daemon on **every single Kubernetes worker node**, absorbing $99\%$ of DNS queries locally without hitting CoreDNS pods.
2. **Configure Positive JVM DNS Caching in Java 21**:
   ```java
   // Cache successful DNS resolutions for 30 seconds
   java.security.Security.setProperty("networkaddress.cache.ttl", "30");
   ```

---

## 6. Postmortem Action Items
- [x] Enable Kubernetes Cluster Proportional Autoscaler for CoreDNS.
- [x] Deploy NodeLocal DNSCache across all EKS node groups.
