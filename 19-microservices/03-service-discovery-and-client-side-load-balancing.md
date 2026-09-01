# Lesson 03: Service Discovery & Client Load Balancing

Master dynamic service registry architectures (Eureka, Consul, Kubernetes CoreDNS), client-side load balancing algorithms, health check heartbeats, and zone-aware routing.

---

## 1. What Is It?
- **Service Discovery**: An automated mechanism that allows microservices to locate the dynamic IP addresses and ports of other ephemeral service instances without hardcoding hostnames.
- **Client-Side Load Balancing**: An architecture where the calling microservice queries the registry, caches the list of healthy backend instances, and executes load-balancing algorithms locally before making the network request.

---

## 2. Why Does It Exist?
In modern cloud and Kubernetes environments, container pods scale up and down dynamically, crash, and change IP addresses constantly. Hardcoding IPs in configuration files is impossible.

---

## 3. Mental Model: Server-Side vs Client-Side Load Balancing

```mermaid
flowchart TD
    subgraph ServerSide["Server-Side Load Balancing (Hardware / NLB)"]
        C1["Service A"] --> LB["Central Load Balancer (Proxy Hop)"]
        LB --> I1["Instance 1"]
        LB --> I2["Instance 2"]
    end

    subgraph ClientSide["Client-Side Load Balancing (Spring Cloud LoadBalancer)"]
        Reg["Service Registry (Consul / Eureka)"]
        C2["Service A
(Maintains Cached Server List)"]
        C2 -. 1. Query Instances .-> Reg
        C2 -- 2. Direct Round-Robin Call ⚡ --> I3["Instance 1"]
        C2 -- 3. Direct Round-Robin Call ⚡ --> I4["Instance 2"]
    end
```

---

## 4. How Does It Work: Comparing Service Discovery Approaches

| Approach | Technology | Pros | Cons |
|---|---|---|---|
| **Application-Level Registry**| Netflix Eureka / HashiCorp Consul | Zone-aware routing, rich metadata | Requires language-specific SDK |
| **Kubernetes DNS / CoreDNS** | K8s ClusterIP Services | Language-agnostic, built into platform | Simple round-robin only (no custom algorithms) |
| **Service Mesh (Envoy)** | Istio / Linkerd | Language-agnostic, mTLS, traffic splitting | High memory and CPU sidecar overhead |

---

## 5. Internal Working: Client-Side Load Balancing Algorithms

1. **Round Robin**: Cycles sequentially through the instance list ($1, 2, 3, 1, 2, 3$).
2. **Weighted Response Time**: Measures p99 latency of each pod; routes more traffic to pods with faster response times.
3. **Zone-Aware / Locality Routing**: Prioritizes instances located within the **same AWS Availability Zone (AZ)** to eliminate inter-AZ cross-zone data transfer latency and cloud egress fees.

---

## 6. Example & Production Java 21 Code

Custom Round-Robin Client Load Balancer with Heartbeat Filtering:

```java
package com.backend.microservices.discovery;

import java.net.URI;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class ClientSideLoadBalancer {

    public record ServiceInstance(String instanceId, String host, int port, boolean healthy, String zone) {
        public URI toUri() {
            return URI.create("http://" + host + ":" + port);
        }
    }

    private final AtomicInteger position = new AtomicInteger(0);

    public ServiceInstance chooseInstance(List<ServiceInstance> allInstances, String currentZone) {
        // 1. Filter healthy instances
        List<ServiceInstance> healthyInstances = allInstances.stream()
            .filter(ServiceInstance::healthy)
            .toList();

        if (healthyInstances.isEmpty()) {
            throw new IllegalStateException("No healthy service instances available!");
        }

        // 2. Prefer instances in the same Availability Zone (Zone-Aware)
        List<ServiceInstance> zoneInstances = healthyInstances.stream()
            .filter(inst -> inst.zone().equalsIgnoreCase(currentZone))
            .toList();

        List<ServiceInstance> candidates = zoneInstances.isEmpty() ? healthyInstances : zoneInstances;

        // 3. Atomically select next instance via Round Robin
        int index = Math.abs(position.getAndIncrement() % candidates.size());
        return candidates.get(index);
    }
}
```

---

## 7. Performance Characteristics
- Eliminating the middle proxy hop via client-side load balancing reduces end-to-end service call latency by $\sim 1.5\text{ms}$ to $3\text{ms}$.

---

## 8. Failure Scenarios & Edge Cases
- **Stale Registry Cache During Sudden Crash**: A backend pod crashes; the registry takes 30 seconds to detect missed heartbeats. In that 30-second window, the client-side load balancer keeps sending requests to the dead pod.
  - **Mitigation**: Combine client load balancing with **Resilience4j Circuit Breakers** to immediately drop failing nodes from the local active pool on the first connection failure.

---

## 9. Observability (Logs, Metrics, Traces)
```text
# Load Balancer Metrics
loadbalancer_active_instances{service="order-service",zone="us-east-1a"} 8
loadbalancer_active_instances{service="order-service",zone="us-east-1b"} 8
loadbalancer_routing_total{strategy="zone_aware"} 184000
```

---

## 10. Debugging & Troubleshooting
1. **Check Kubernetes CoreDNS Resolution**:
   ```bash
   dig @10.96.0.10 order-service.default.svc.cluster.local
   ```

---

## 11. Scaling Considerations
- For massive clusters ($> 5,000$ pods), use **Kubernetes EndpointsSlice API** to avoid massive network multicast storms caused by updating large monolithic Endpoint resources.

---

## 12. Architectural Trade-offs
| Architecture | Network Latency | Central Bottleneck | Operational Complexity |
|---|---|---|---|
| **Server-Side Proxy (NLB)** | $+2\text{ms}$ (Extra Hop) | Yes (LB can bottleneck) | Low |
| **Client-Side Load Balancer**| **Zero Extra Hops** | **None** | Moderate (App library) |

---

## 13. When to Use
- Use **Client-Side Load Balancing** for internal microservice-to-microservice RPC/REST calls to minimize latency.

---

## 14. When NOT to Use
- Do not use client-side load balancing for external public clients (browsers/mobile) — use public Edge Load Balancers.

---

## 15. SDE2 / Senior Interview Questions & Answers
### Q1: What is Zone-Aware Load Balancing, and why is it critical in multi-region/multi-AZ cloud deployments?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- **What it is**: Cloud providers (AWS, GCP, Azure) deploy infrastructure across multiple independent physical data centers called Availability Zones (e.g., `us-east-1a`, `us-east-1b`, `us-east-1c`). Zone-aware load balancing routes requests from Service A in `us-east-1a` to an instance of Service B in `us-east-1a`.
- **Why it is critical**:
  1. **Latency Reduction**: Crossing physical availability zones adds $\sim 1\text{ms}$ to $2\text{ms}$ round-trip latency. Same-AZ communication stays within the local data center switch fabric ($< 0.2\text{ms}$).
  2. **Cloud Cost Elimination**: Major cloud providers charge **\$0.01 per GB** for data transferred across AZ boundaries. At scale (petabytes of microservice traffic), inter-AZ data transfer fees cost tens of thousands of dollars per month. Same-AZ traffic is completely free.
  3. **Fault Isolation**: If a network cable between AZ `1a` and `1b` is damaged, services in `1a` continue functioning autonomously.
</details>

---

## 16. Practical Exercise
1. Implement a Weighted Round-Robin load balancing algorithm in Java that routes $70\%$ of traffic to 8-core instances and $30\%$ to 4-core instances.

---

## 17. Quick Revision Summary
- **Client-Side Load Balancing** eliminates intermediary proxy bottlenecks.
- Use **Zone-Aware routing** to cut inter-AZ latency and cloud transfer costs.
- Pair client load balancers with **Circuit Breakers** to handle stale registry caches.
