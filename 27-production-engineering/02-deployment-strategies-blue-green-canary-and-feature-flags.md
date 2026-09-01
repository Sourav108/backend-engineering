# Deployment Strategies: Blue-Green, Canary Releases, and Feature Flags

---

## 1. What Is It?
A **Production Deployment Strategy** is the architectural methodology by which new versions of backend application software are introduced into live environments while minimizing user-facing downtime, service degradation, and operational blast radius.

The 4 core production deployment strategies are:
1. **Rolling Update**: Incremental pod-by-pod replacement in Kubernetes.
2. **Blue-Green Deployment**: Running two identical production environments (Blue = Live, Green = Stage) and atomically switching the load balancer router.
3. **Canary Deployment**: Routing a tiny percentage ($1\% - 5\%$) of live user traffic to the new version to evaluate real-world metrics before full rollout.
4. **Feature Flags**: Dynamic runtime toggles that decouple **Code Deployment** from **Feature Activation**.

---

## 2. Why Does It Exist?
Deploying new code directly to $100\%$ of users simultaneously (**Big-Bang Deployment**) is the #1 cause of catastrophic SEV-1 production outages.

- **The Hidden Edge Bug**: A memory leak or concurrency deadlock that never appeared in staging tests will crash all production servers within 5 minutes if released to $100\%$ of users at once.
- **Canary & Feature Flag Defenses**: Exposing new code to only $1\%$ of users isolates the blast radius to a tiny fraction of requests, and automated Prometheus monitoring triggers an **instant rollback before 99% of customers are ever exposed**.

---

## 3. Mental Model: The 4 Deployment Strategies

```mermaid
flowchart TD
    subgraph Rolling["1. Rolling Update (Progressive Pod Replacement)"]
        OldPods["[Pod v1] [Pod v1]"] --> Trans["[Pod v1] [Pod v2]"] --> NewPods["[Pod v2] [Pod v2]"]
    end

    subgraph BlueGreen["2. Blue-Green (Atomic Router Switch)"]
        ALB["Load Balancer"] -->|Active Route (100%)| Blue[("Blue Env: v1.0 (Live)")]
        ALB -.->|Switch Route in 1ms| Green[("Green Env: v2.0 (Idle -> Live)")]
    end

    subgraph Canary["3. Canary Release (Progressive Metric Analysis)"]
        Router["Traffic Splitter"] -->|95% Live Traffic| Stable["Stable Tier: v1.0"]
        Router -->|5% Live Traffic| CanaryPod["Canary Pod: v2.0 (Prometheus Monitored)"]
    end

    subgraph Flags["4. Feature Flags (Runtime Decoupled)"]
        Code["Deployed Code v2.0"] --> Toggle{"Feature Flag: 'new_checkout'"}
        Toggle -- Enabled for User 42 --> NewFlow["Execute New Checkout Flow"]
        Toggle -- Disabled (Default) --> OldFlow["Execute Legacy Checkout Flow"]
    end
```

---

## 4. Comprehensive Strategy Comparison

| Strategy | Zero Downtime? | Rollback Speed | Extra Infrastructure Cost | Blast Radius on Bug |
|---|---|---|---|:---:|
| **Rolling Update** | ✅ Yes | Moderate ($1 - 3\text{ minutes}$) | **Minimal ($+25\%$ surge)** | Moderate (Partial cluster) |
| **Blue-Green** | ✅ Yes | **Instant ($< 1\text{ second}$)** | **High ($+100\%$ duplicate capacity)**| Moderate (All or nothing) |
| **Canary Release** | ✅ Yes | **Automated ($< 10\text{ seconds}$)** | Low ($+5\%$ extra pods) | **Minimal ($1\% - 5\%$ of users)** |
| **Feature Flags** | ✅ Yes | **Instant ($0\text{ deploy time}$)** | Near-Zero | **Configurable (Per-user / Target %)** |

---

## 5. Implementation: Automated Canary Deployment with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service-rollout
  namespace: production
spec:
  replicas: 10
  strategy:
    canary:
      # Automated Prometheus Metric Analysis
      analysis:
        templates:
          - templateName: success-rate-prometheus
        args:
          - name: service-name
            value: payment-service
      steps:
        # Step 1: Route 5% traffic to canary and wait 10 minutes
        - setWeight: 5
        - pause: { duration: 10m }

        # Step 2: Route 20% traffic and pause for automated analysis
        - setWeight: 20
        - pause: { duration: 30m }

        # Step 3: Route 50% traffic
        - setWeight: 50
        - pause: { duration: 15m }

        # Step 4: Promote to 100% (Complete Rollout)
        - setWeight: 100
```

---

## 6. Implementation: Java 21 Feature Flag / Kill Switch Pattern

```java
package com.backend.engineering.production.flags;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class CheckoutService {

    private static final Logger log = LoggerFactory.getLogger(CheckoutService.class);
    private final FeatureFlagProvider flagProvider;

    public CheckoutService(FeatureFlagProvider flagProvider) {
        this.flagProvider = flagProvider;
    }

    public CheckoutResult processOrderCheckout(Long userId, OrderRequest request) {
        // Dynamic Runtime Evaluation (Sub-Millisecond in-memory evaluation)
        if (flagProvider.isFlagEnabled("enable_one_click_checkout", userId)) {
            try {
                return executeOptimizedCheckout(userId, request);
            } catch (Exception ex) {
                // Instant Graceful Fallback / Emergency Kill Switch!
                log.error("Optimized checkout failed. Gracefully falling back to legacy checkout: {}", ex.getMessage());
                return executeLegacyCheckout(userId, request);
            }
        }

        return executeLegacyCheckout(userId, request);
    }

    private CheckoutResult executeOptimizedCheckout(Long userId, OrderRequest request) {
        // High-speed V2 checkout pipeline
        return new CheckoutResult(true, "V2_OPTIMIZED");
    }

    private CheckoutResult executeLegacyCheckout(Long userId, OrderRequest request) {
        // Battle-tested V1 legacy pipeline
        return new CheckoutResult(true, "V1_LEGACY");
    }

    public interface FeatureFlagProvider {
        boolean isFlagEnabled(String flagKey, Long userId);
    }

    public record OrderRequest(Long amountCents) {}
    public record CheckoutResult(boolean success, String pipelineVersion) {}
}
```

---

## 7. Performance

| Operational Scenario | Big-Bang Deployment | Blue-Green Deployment | Canary Release with Argo |
|---|---|---|---|
| Critical Regression Bug in V2 | **$100\%$ of Users Crashed** | $100\%$ crashed for 30s until rollback | **$5\%$ of Users Impacted; Auto-rolled back in 10s** |
| Rollback Execution | Full CI/CD rebuild ($15\text{ mins}$) | **Instant ALB DNS swap ($1\text{s}$)** | **Automated ($0\text{ human intervention}$)** |

---

## 8. Interview Questions

### Q1: What is the fundamental difference between Code Deployment and Feature Release?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- **Code Deployment (Technical Operation)**:
  - The physical act of transferring compiled container images into production Kubernetes clusters and executing processes on servers.
  - In a mature organization, code deployment happens **dozens of times per day** via automated CI/CD pipelines.
- **Feature Release (Business Operation)**:
  - The business decision to expose a newly deployed capability to end users.
  - By using **Feature Flags (e.g. LaunchDarkly / Unleash)**, new code is deployed to production completely dormant (`flag = false`).
  - Product managers and engineers can then release the feature progressively:
    1. First to internal employees (*Dogfooding*).
    2. Then to $1\%$ of beta customers.
    3. Then to $100\%$ globally.
- **Why this matters**: Decoupling the two operations allows engineering to ship code continuously without business coordination, and provides an **instant kill switch** to deactivate broken features in 1 second without redeploying code.
</details>

---

## 9. Quick Revision
- **Rolling Update**: Replaces pods incrementally; low cost; standard K8s strategy.
- **Blue-Green**: Runs 2 identical full environments; instant atomic rollback.
- **Canary (Argo)**: Routes $1-5\%$ traffic; monitors Prometheus metrics; auto-rolls back.
- **Feature Flags**: Decouples code deployment from feature release; provides instant runtime kill switches.
- **Blast Radius**: Canary releases limit failure blast radius to $< 5\%$ of users.
