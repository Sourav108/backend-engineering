# Incident 10: Split-Brain in Distributed Cluster (Even-Node Quorum Violation)

---

## 1. Symptoms & Alert
- **Alert**: `ClusterLeaderElectionSplitBrainWarning` in distributed consensus coordinator.
- **Customer Impact**: Conflicting state writes accepted simultaneously in two separate partitions; financial balances desynchronized.

---

## 2. Metric & Telemetry Anomalies
- **Cluster State**: Cluster monitoring reported **Two Active Master Nodes** simultaneously: Node 1 in Datacenter A and Node 3 in Datacenter B.
- **Data Inconsistency**: Diverging transaction logs recorded in Datacenter A and Datacenter B for the exact same entity keys.

---

## 3. Diagnostic Steps & Root Cause
### Cluster Architecture Review:
The cluster was configured with **4 nodes** (2 nodes in Datacenter A, 2 nodes in Datacenter B).

### The Network Split:
1. The transatlantic fiber link between Datacenter A and Datacenter B severed.
2. Datacenter A nodes (2 nodes) could not communicate with Datacenter B nodes (2 nodes).
3. Because the cluster lacked an **Odd Number of Nodes** and a **Tie-Breaker Witness Node**, both partitions calculated $\text{Active Nodes} = 2$.
4. A misconfigured custom consensus quorum rule permitted both sides to elect a local Leader, creating a **Classic Split-Brain Catastrophe**.

---

## 4. Immediate Mitigation
1. **Force Shutdown of Leader in Datacenter B**:
   ```bash
   ssh node-3.dc-b "systemctl stop consensus-cluster"
   ```
2. **Execute Emergency Database Reconciliation Script** to resolve conflicting transactions between DC-A and DC-B.

---

## 5. Permanent Fix
1. **Enforce Odd Number of Consensus Nodes with Strict Quorum ($\lfloor N/2 \rfloor + 1$)**:
   - Reconfigure cluster with **5 nodes** (e.g. 2 in DC-A, 2 in DC-B, 1 lightweight witness node in Cloud Region C).
   - In any network split, only the side containing the 3-node majority can achieve Quorum ($3 > 5/2$) and elect a leader; the 2-node minority **strictly halts all writes**.

---

## 6. Postmortem Action Items
- [x] Configure Raft-based consensus protocol (replacing legacy home-grown leader election).
- [x] Schedule Chaos GameDay simulating inter-datacenter network partitions.
