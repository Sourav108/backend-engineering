# Incident Postmortem: [SEV-X] [Incident Name]

- **Date**: YYYY-MM-DD
- **Severity**: SEV-1 / SEV-2
- **Duration**: XX minutes (MTTD: Xm, MTTR: Ym)
- **Authors**: Backend Core Team

---

## 1. Executive Summary
[Brief description of outage, user impact, and immediate resolution]

## 2. Customer Impact
- Total failed requests: X,XXX (Y% error rate)
- Affected services: [List of microservices]

## 3. Timeline (UTC)
- `18:00` - Alert triggered on Grafana error rate spike.
- `18:04` - Incident commander assembled war room.
- `18:12` - Root cause identified as downstream latency cascade.
- `18:18` - Circuit breaker tripped to fallback response; error rate normalized.
- `18:35` - Permanent hotfix deployed; incident closed.

## 4. Root Cause (5 Whys)
1. Why:
2. Why:
3. Why:
4. Why:
5. Why:

## 5. Action Items & Prevention
| Action Item | Type | Owner | Target Date |
|---|---|---|---|
| Add Circuit Breaker fallback | Remediation | Alice | YYYY-MM-DD |
| Configure PagerDuty SLO alert | Detection | Bob | YYYY-MM-DD |
