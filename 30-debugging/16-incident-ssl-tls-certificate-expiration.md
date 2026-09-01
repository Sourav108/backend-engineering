# Incident 16: SSL/TLS Certificate Expiration (`PKIX path building failed`)

---

## 1. Symptoms & Alert
- **Alert**: `javax.net.ssl.SSLHandshakeException: PKIX path building failed: validator.ValidatorException: Certificate expired`.
- **Customer Impact**: Total failure of internal microservice communication; Order Service unable to invoke Payment Service or Kafka broker over mTLS.

---

## 2. Metric & Telemetry Anomalies
- **Error Logs**: Spiking to $100\%$ SSL handshake failure on all outgoing HTTPS and gRPC calls.
- **Service Mesh**: Istio Envoy sidecars dropping connections with `SSL_ERROR_CERT_EXPIRED`.

---

## 3. Diagnostic Steps & Root Cause
### Step 1: Inspect Remote TLS Certificate Expiration Date
```bash
echo | openssl s_client -servername payment.internal -connect payment.internal:443 2>/dev/null | openssl x509 -noout -dates
# Output:
# notBefore=Sep  1 00:00:00 2025 GMT
# notAfter=Sep  1 00:00:00 2026 GMT   <--- EXPIRED 2 HOURS AGO!
```

### Root Cause:
The internal X.509 server certificate had a 1-year validity period. The automated rotation cron failed silently due to an expired IAM role permission 14 days ago, and zero alerts were configured to monitor certificate expiration dates.

---

## 4. Immediate Mitigation
Issue emergency certificate via `cert-manager` / AWS ACM:
```bash
kubectl delete secret payment-service-tls-secret -n production
# cert-manager immediately re-issues a new certificate from the Root CA!
```

---

## 5. Permanent Fix
1. Configure `cert-manager` with automated renewal **30 days prior to expiration** (`renewBefore: 720h`).
2. Deploy Prometheus `blackbox_exporter` to monitor certificate expiration dates across all endpoints:
   ```yaml
   - alert: SSLCertificateExpiringSoon
     expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 14 # Alert 14 days in advance!
     for: 10m
     labels:
       severity: warning
   ```

---

## 6. Postmortem Action Items
- [x] Configure PagerDuty alert on all internal and external TLS certificate expiry metrics.
- [x] Migrate all internal service mesh certificates to automated SPIRE / Istio automated rotation.
