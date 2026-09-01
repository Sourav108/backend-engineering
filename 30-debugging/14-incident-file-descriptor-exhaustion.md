# Incident 14: File Descriptor Exhaustion (`java.io.IOException: Too many open files`)

---

## 1. Symptoms & Alert
- **Alert**: `java.io.IOException: Too many open files` thrown across all HTTP clients and database connection pools.
- **Customer Impact**: Application unable to open new TCP network connections, accept incoming HTTP requests, or read files from disk.

---

## 2. Metric & Telemetry Anomalies
- **Linux OS Metrics**: Open file descriptors count (`process_open_fds`) reached the hard container ulimit ceiling ($1,024 / 1,024$).
- **Socket Metrics**: `ss -s` showed thousands of sockets stuck in `CLOSE_WAIT` state.

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Inspect Open File Descriptors on Container Process
```bash
kubectl exec -it document-service-784f-x2q -n production -- ls -l /proc/1/fd | wc -l
# Output: 1024

kubectl exec -it document-service-784f-x2q -n production -- ls -l /proc/1/fd | head -n 20
# Output: Hundreds of open socket handles pointing to S3 / Internal REST endpoints:
# lrwx------ 1 app app 64 Sep 1 14:20 42 -> socket:[12489012]
```

### Root Cause:
A new PDF generation service read input streams from S3 using raw `InputStream` without a **`try-with-resources` block**:

```java
// DISASTER: UNCLOSED INPUT STREAM LEAKS SOCKET FILE DESCRIPTOR!
public byte[] downloadUserDocument(String s3Key) {
    ResponseInputStream<GetObjectResponse> stream = s3Client.getObject(r -> r.bucket("docs").key(s3Key));
    return stream.readAllBytes(); // Stream never closed! Socket descriptor leaked forever!
}
```

---

## 4. Immediate Mitigation
Restart container pods to release leaked Linux file descriptors:
```bash
kubectl rollout restart deployment document-service -n production
```

---

## 5. Permanent Fix
1. Wrap all stream and I/O reads in strict **`try-with-resources` blocks**:
   ```java
   public byte[] downloadUserDocument(String s3Key) throws IOException {
       try (ResponseInputStream<GetObjectResponse> stream = s3Client.getObject(r -> r.bucket("docs").key(s3Key))) {
           return stream.readAllBytes(); // Guaranteed socket closure on block exit!
       }
   }
   ```
2. Increase Linux container file descriptor ulimits in Kubernetes pod security context:
   ```yaml
   securityContext:
     sysctls:
       - name: fs.file-max
         value: "65536"
   ```

---

## 6. Postmortem Action Items
- [x] Configure SonarQube linter rule enforcing `try-with-resources` on all `AutoCloseable` types.
- [x] Add Prometheus alert triggering at $75\%$ open file descriptor capacity.
