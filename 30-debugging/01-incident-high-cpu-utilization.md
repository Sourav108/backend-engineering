# Incident 01: High CPU Utilization in Production (Regex Catastrophic Backtracking)

---

## 1. Symptoms & Alert
- **PagerDuty Alert**: `ContainerCPUUtilization > 95% for 3 consecutive evaluation periods`.
- **Customer Impact**: API $p99$ response times surged from $25\text{ms}$ to $> 15,000\text{ms}$; HTTP 504 Gateway Timeouts on `/api/v1/auth/validate`.

---

## 2. Metric & Telemetry Anomalies
- **CloudWatch / Prometheus**: CPU utilization pinned at $100\%$ across all 6 authentication pods.
- **JVM Metrics**: Heap memory usage normal ($35\%$), GC pause times $< 1\text{ms}$ (ZGC healthy), but `jvm.threads.busy` at $100\%$.

---

## 3. Diagnostic Steps

### Step 1: Identify High-CPU Thread IDs inside Container
```bash
# Get inside the running Kubernetes pod
kubectl exec -it auth-service-784f697b8-x2q9w -n production -- top -H -b -n 1 | head -n 20
```
- *Output*: Thread PID `42` consuming $99.8\%$ CPU in user-space (`us`).

### Step 2: Convert Thread PID to Hexadecimal
```bash
printf "0x%x\n" 42
# Output: 0x2a
```

### Step 3: Capture On-Demand JVM Thread Dump
```bash
kubectl exec -it auth-service-784f697b8-x2q9w -n production -- jcmd 1 Thread.print | grep -A 25 "nid=0x2a"
```
- *Thread Dump Trace*:
```text
"http-nio-8080-exec-4" #42 daemon prio=5 os_prio=0 cpu=45120.50ms nid=0x2a runnable
   java.lang.Thread.State: RUNNABLE
	at java.util.regex.Pattern$Curly.match(java.base@21.0.4/Pattern.java:4521)
	at java.util.regex.Pattern$GroupCurly.match(java.base@21.0.4/Pattern.java:4810)
	at java.util.regex.Matcher.match(java.base@21.0.4/Matcher.java:1774)
	at com.backend.engineering.auth.EmailValidator.isValid(EmailValidator.java:18)
```

---

## 4. Root Cause Analysis
An unanchored regular expression containing nested quantifiers (`^([a-zA-Z0-9_.-]+)+@([a-zA-Z0-9-]+.)+([a-zA-Z]{2,4})+$`) was used to validate email inputs.
A malicious or malformed input payload (`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!@domain.com`) triggered **Catastrophic Backtracking (Exponential $O(2^N)$ Complexity)** in the Java Regex NFA engine, trapping CPU worker threads in an infinite calculation loop.

---

## 5. Immediate Mitigation
1. **Scale Out Pods to Absorb Legitimate Traffic**:
   ```bash
   kubectl scale deployment auth-service -n production --replicas=15
   ```
2. **Apply WAF / API Gateway Rate Limiting Rule** on `/api/v1/auth/validate` to block the malformed user agent pattern.

---

## 6. Permanent Fix
Replace the complex backtracking Regex with Apache Commons Validator or a deterministic linear-time state machine:

```java
// BEFORE (VULNERABLE TO CATASTROPHIC BACKTRACKING)
Pattern p = Pattern.compile("^([a-zA-Z0-9_.-]+)+@([a-zA-Z0-9-]+.)+([a-zA-Z]{2,4})+$");

// AFTER (SAFE LINEAR-TIME VALIDATION)
public static boolean isValidEmail(String email) {
    if (email == null || email.length() > 254) return false;
    return EmailValidator.getInstance().isValid(email);
}
```

---

## 7. Postmortem Action Items
- [x] Add CI static analysis rule (SonarQube) flagging regexes with nested quantifiers (`(a+)+`).
- [x] Enforce timeout boundaries on all regex evaluations via custom `CharSequence` wrappers.
