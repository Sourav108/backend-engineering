# Incident 12: Cross-AZ Network Partition (Availability Zone Outage)

---

## 1. Symptoms & Alert
- **Alert**: `PostgreSQLReplicationLagHigh > 300s` and `ELB TargetGroupUnhealthyHostCount > 33%`.
- **Customer Impact**: $\approx 33\%$ of user requests experiencing network connection timeouts and HTTP 502 errors.

---

## 2. Metric & Telemetry Anomalies
- **AWS CloudWatch**: Packet loss between Availability Zone `us-east-1a` and `us-east-1b` jumped to $100\%$.
- **Database Metrics**: PostgreSQL Synchronous Standby in AZ-1b lost connection to Primary in AZ-1a, stalling all primary write transactions.

---

## 3. Diagnostic Steps & Root Cause
1. AWS experienced a physical fiber link severance between AZ-1a and AZ-1b.
2. The PostgreSQL Primary in AZ-1a was configured with `synchronous_commit = on` and `synchronous_standby_names = 'replica_az_1b'`.
3. Because the Primary could not receive WAL replication ACKs from the severed Standby in AZ-1b, **all incoming database write transactions hung indefinitely**, waiting for replication confirmation.
4. Concurrently, the Application Load Balancer continued routing $33\%$ of public traffic to pods in AZ-1b, which could not reach the primary database.

---

## 4. Immediate Mitigation
1. **Deregister Severed AZ-1b from Load Balancer Target Groups**:
   - Force all ingress traffic to route exclusively to healthy AZ-1a and AZ-1c.
2. **Dynamically Demote Standby to Asynchronous**:
   ```sql
   ALTER SYSTEM SET synchronous_commit = 'local';
   SELECT pg_reload_conf();
   ```

---

## 5. Permanent Fix
1. Configure **Multi-Standby Quorum Replication** across 3 distinct Availability Zones:
   ```text
   synchronous_commit = on
   synchronous_standby_names = 'ANY 1 (replica_az_1b, replica_az_1c)'
   ```
   *(If any single AZ fails, the remaining healthy AZ satisfies Quorum with zero write stalls!)*
2. Configure **Route 53 AZ-Level Health Checks** to automatically evict failing AZs from DNS routing.

---

## 6. Postmortem Action Items
- [x] Ensure all microservices and databases are deployed across a minimum of 3 Availability Zones.
- [x] Run automated GameDay validating seamless multi-AZ failover during synthetic AZ severance.
