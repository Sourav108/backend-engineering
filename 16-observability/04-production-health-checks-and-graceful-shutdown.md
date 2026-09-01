# Health Checks, Kubernetes Probes, and Graceful Shutdown Mechanics

---

## 1. What Is It?
**Production Health Checks** are HTTP endpoints exposed by a backend application to report its operational availability to load balancers and orchestrators.

In Kubernetes and cloud platforms, health checks are divided into:
1. **Startup Probes**: Verifies whether the JVM has finished initialization.
2. **Liveness Probes**: Verifies whether the application process is alive or deadlocked (triggers container restart).
3. **Readiness Probes**: Verifies whether the application is ready to accept user traffic (regulates load balancer ingress routing).
4. **Graceful Shutdown**: The orderly draining and completion of active in-flight requests when a `SIGTERM` signal is received, preventing dropped user connections during deployments.

---

## 2. Why Does It Exist?
Without granular probes and graceful shutdown:
- **Deployment Downtime**: Rolling updates terminate old pods while they are actively processing credit card transactions, throwing `502 Bad Gateway` and `Connection Reset` errors to thousands of active users.
- **Deadlock Cascades**: A deadlocked application thread pool continues returning HTTP 200 on basic root checks, trapping user traffic in hanging sockets.
- **Premature Traffic Inundation**: Kubernetes routes live customer traffic to a pod while the JVM JIT compiler and database connection pool are still warming up, causing an initial wave of 500 errors.

---

## 3. Mental Model: Kubernetes Probe Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Starting: Pod Container Created
    Starting --> StartupProbe: JVM Booting up
    
    StartupProbe --> StartupProbe: /health/startup -> 503 (Keep waiting)
    StartupProbe --> Running: /health/startup -> 200 (App Ready!)
    
    state Running {
        [*] --> CheckingProbes
        
        state CheckingProbes {
            Liveness: Liveness Probe (/health/liveness)
            Readiness: Readiness Probe (/health/readiness)
        }
        
        Readiness --> RemoveFromService: Returns 503 (DB Slow/Disconnected)
        RemoveFromService --> Readiness: Returns 200 (Reconnected -> Traffic Resumed!)
        
        Liveness --> RestartContainer: Returns 503 (Deadlocked -> Kube kills Pod!)
    }

    Running --> Terminating: SIGTERM Received (Rolling Update)
    Terminating --> Draining: Readiness fails -> Traffic halted -> In-flight requests drain (30s)
    Draining --> [*]: Pod cleanly stops
```

---

## 4. How Does It Work?

### Liveness vs Readiness Probes (The Critical Distinction)

| Attribute | Liveness Probe (`/actuator/health/liveness`) | Readiness Probe (`/actuator/health/readiness`) |
|---|---|---|
| **Question Answered** | "Is the JVM process alive and un-deadlocked?" | "Is the service ready to handle user requests *right now*?" |
| **Failure Action** | **Kills and Restarts the Pod (`SIGKILL`)** | **Removes Pod from Load Balancer Endpoints** (Zero Restarts!) |
| **What It Checks** | Internal JVM liveness (Zero external DB checks!) | Downstream databases, Redis caches, Kafka brokers, warmup state |

$$\textbf{Critical Rule: } \text{NEVER check external dependencies (Database, Redis) inside a Liveness Probe!}$$

*Why?* If your PostgreSQL database suffers a 10-second blip, checking the database in a Liveness Probe will cause Kubernetes to **simultaneously kill and restart all 100 microservice pods at once**, turning a brief database slowdown into a catastrophic total platform crash!

---

## 5. Graceful Shutdown Mechanics (Handling `SIGTERM`)

When Kubernetes deploys a new version or terminates a pod:
1. Kubernetes sends a **`SIGTERM`** signal to the container process.
2. The Spring Boot application immediately flips its **Readiness State to `REFUSING_TRAFFIC`** (returning 503).
3. Kubernetes removes the pod IP from the Service endpoint routing table (Ingress stops sending new requests).
4. The embedded Tomcat server stops accepting new TCP connections, but grants active in-flight requests a grace period (e.g. $30\text{ seconds}$) to finish processing and flush database transactions.
5. Once in-flight requests hit 0, Spring closes HikariCP connection pools, shuts down Kafka listeners, and terminates cleanly with exit code 0.

---

## 6. Implementation: Spring Boot 3 Configuration & Custom Health Indicator

### 1. `application.yml`
```yaml
server:
  shutdown: graceful # Enables orderly in-flight draining

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s # Maximum time allowed for in-flight requests to finish

management:
  endpoints:
    web:
      exposure:
        include: "health,metrics,prometheus"
  endpoint:
    health:
      probes:
        enabled: true # Automatically activates /actuator/health/liveness and readiness
      show-details: when_authorized
```

---

### 2. Custom Dependency Health Indicator in Java 21

```java
package com.backend.engineering.observability.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.data.redis.connection.RedisConnection;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.stereotype.Component;

@Component
public class RedisCustomHealthIndicator implements HealthIndicator {

    private final RedisConnectionFactory redisConnectionFactory;

    public RedisCustomHealthIndicator(RedisConnectionFactory redisConnectionFactory) {
        this.redisConnectionFactory = redisConnectionFactory;
    }

    @Override
    public Health health() {
        try (RedisConnection connection = redisConnectionFactory.getConnection()) {
            String pingResult = connection.ping();
            if ("PONG".equalsIgnoreCase(pingResult)) {
                return Health.up()
                        .withDetail("redis", "Available")
                        .withDetail("response", pingResult)
                        .build();
            } else {
                return Health.down()
                        .withDetail("redis", "Unexpected PING response: " + pingResult)
                        .build();
            }
        } catch (Exception ex) {
            // Contributes to Readiness failure, removing pod from traffic without killing it!
            return Health.down(ex)
                    .withDetail("redis", "Connection unreachable")
                    .build();
        }
    }
}
```

---

## 7. Performance

| Shutdown Mode | Deployment Success Rate | Active Transaction Dropped Connections |
|---|---|---|
| Abrupt `SIGKILL` / Default | $\approx 92 - 97\%$ | Hundreds of 502/504 errors per rolling deploy |
| **Graceful Shutdown (`30s` Drain)** | **$\mathbf{100\%}$ Zero Downtime** | **$\mathbf{0}$ Dropped Connections** |

---

## 8. Interview Questions

### Q1: Why is checking database availability in a Kubernetes Liveness Probe a dangerous anti-pattern?
<details>
<summary>Reveal Answer</summary>

**Answer**:
The purpose of a **Liveness Probe** is to detect if the container is in an unrecoverable internal deadlock or infinite loop, where restarting the process is the only remediation.
If a Liveness probe includes an external database check:
1. When the database experiences a momentary overload or brief network hiccup, all application pods will fail their liveness checks simultaneously.
2. Kubernetes will immediately **kill and restart every single pod across the entire cluster**.
3. When hundreds of new pods restart at the exact same moment, they will all attempt to initialize new database connection pools simultaneously (**Thundering Herd**), hammering the struggling database and causing a **Cascading Cluster Crash Loop**.
**Rule**: Check databases strictly in **Readiness Probes**. When the DB fails, readiness marks the pods as unready (pausing traffic), but leaves the pods alive and waiting for database recovery.
</details>

---

## 9. Quick Revision
- **Startup Probe**: Protects slow-starting JVMs from premature liveness kills.
- **Liveness Probe**: Checks internal process health; restarts container on failure.
- **Readiness Probe**: Checks dependencies; stops traffic routing without restarting pod.
- **Graceful Shutdown**: `server.shutdown: graceful` allows in-flight requests to complete during rolling updates.
- **`SIGTERM` Drain**: Always configure `timeout-per-shutdown-phase: 30s`.
