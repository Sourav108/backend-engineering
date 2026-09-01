# Docker & Kubernetes for Backend Engineers Cheat Sheet

---

## ⚡ 1. Production Multi-Stage Dockerfile Template

```dockerfile
FROM maven:3.9.8-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests -B
WORKDIR /app/target
RUN java -Djarmode=layertools -jar *.jar extract

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup
COPY --from=builder --chown=appuser:appgroup /app/target/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /app/target/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup /app/target/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /app/target/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-XX:+UseZGC", "-XX:+ZGenerational", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## ⚡ 2. Kubernetes Pod Health Probes & Graceful Shutdown

```yaml
spec:
  terminationGracePeriodSeconds: 45
  containers:
    - name: backend-app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"] # Wait for iptables endpoint propagation
      resources:
        requests:
          cpu: "1000m"
          memory: "2Gi"
        limits:
          cpu: "2000m"
          memory: "2Gi" # Guaranteed QoS!
      startupProbe:
        httpGet: { path: /actuator/health/liveness, port: 8080 }
        failureThreshold: 30
        periodSeconds: 2
      livenessProbe:
        httpGet: { path: /actuator/health/liveness, port: 8080 }
        periodSeconds: 10
      readinessProbe:
        httpGet: { path: /actuator/health/readiness, port: 8080 }
        periodSeconds: 5
```

---

## ⚡ 3. Pod Disruption Budget (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend-pdb
spec:
  minAvailable: "80%"
  selector:
    matchLabels:
      app: backend-app
```
