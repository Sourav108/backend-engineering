# Senior Backend Interview Questions: Spring Boot Internals and Enterprise Architecture

Comprehensive bank of senior-level Spring Boot 3.3.4 internals interview questions with mechanical, production-grade model answers.

---

### Q1: How does Spring's `@Transactional` annotation work under the hood, and why does self-invocation break transaction boundaries?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Underlying Proxy Mechanism**:
   - Spring uses Dynamic Proxies (CGLIB or JDK Dynamic Proxy) to wrap `@Transactional` annotated beans with a `TransactionInterceptor`.
   - When an external caller invokes a method on the proxy:
     1. The proxy calls `PlatformTransactionManager.getTransaction()`.
     2. It opens a physical JDBC connection, binds it to the current thread via `TransactionSynchronizationManager`, and sets `connection.setAutoCommit(false)`.
     3. It delegates execution to the target bean.
     4. If an unhandled `RuntimeException` is thrown, the proxy calls `connection.rollback()`; otherwise, it calls `connection.commit()`.
2. **The Self-Invocation Trap**:
   - If `methodA()` (non-transactional) calls `this.methodB()` (annotated with `@Transactional`) within the **same class**:
   - The invocation bypasses the CGLIB proxy wrapper and executes directly on the underlying `this` instance.
   - **Result**: Zero transaction interceptor is triggered, no JDBC connection is bound, and `methodB()` executes **with zero transactional guarantees**.
   - **Remedies**: Move `methodB()` to a separate `@Service` bean, or inject the self-proxy via `@Lazy private MyService self;`.
</details>

---

### Q2: How does Spring Boot's Auto-Configuration work and what is the evaluation order of `@Conditional` annotations?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Auto-Configuration Mechanics**:
   - When Spring Boot boots (`@SpringBootApplication` $\to$ `@EnableAutoConfiguration`), it scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
   - It reads hundreds of pre-packaged auto-configuration classes (e.g. `DataSourceAutoConfiguration`, `RedisAutoConfiguration`).
2. **Conditional Bean Evaluation**:
   - Each auto-configuration class is guarded by conditional annotations:
     - `@ConditionalOnClass(DataSource.class)`: Evaluates if specific classes exist on the runtime classpath.
     - `@ConditionalOnMissingBean(DataSource.class)`: Evaluates whether the application developer has already declared their own custom `@Bean DataSource`.
     - `@ConditionalOnProperty(prefix = "spring.redis", name = "host")`: Evaluates if specific properties are defined in `application.yml`.
   - This ensures developer-defined beans always override Spring Boot defaults cleanly.
</details>

---

### Q3: What is the Spring Security Filter Chain architecture and how does authentication flow from the HTTP request to the `SecurityContext`?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **DelegatingFilterProxy & FilterChainProxy**:
   - The Servlet container passes incoming HTTP requests to Spring's `DelegatingFilterProxy`, which delegates to the `FilterChainProxy` containing the configured `SecurityFilterChain`.
2. **Authentication Pipeline**:
   - Request passes through ordered filters (e.g. `BearerTokenAuthenticationFilter`, `UsernamePasswordAuthenticationFilter`).
   - The filter extracts credentials (e.g. `Authorization: Bearer <jwt>`) and constructs an unauthenticated `AuthenticationToken`.
   - It delegates to the `AuthenticationManager` (which delegates to an `AuthenticationProvider`, e.g. `JwtAuthenticationProvider`).
   - The provider cryptographically validates the token, queries user roles/authorities, and returns a fully authenticated `Authentication` object.
   - The filter stores the authenticated object in the `SecurityContextHolder` (`ThreadLocal`).
</details>

---

### Q4: What is the difference between Spring Bean scopes: `singleton`, `prototype`, `request`, and `session`?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- **`singleton` (Default)**: Exactly one instance per Spring `ApplicationContext`. Shared across all application threads. Must be strictly stateless.
- **`prototype`**: A new instance is created every time the bean is requested (`getBean()` or injected). Spring does not manage its destruction lifecycle.
- **`request`**: Exactly one instance created per HTTP request lifecycle; destroyed when request finishes.
- **`session`**: Exactly one instance created per HTTP Session; stored in Redis or session storage.
</details>

---

### Q5: How does Spring Boot 3 manage Graceful Shutdown and what happens to active HTTP connections and scheduled background tasks?
<details>
<summary>Reveal Answer</summary>

**Answer**:
When `server.shutdown = graceful` is enabled:
1. Upon receiving `SIGTERM`, the embedded web server (Tomcat/Jetty) **stops accepting new incoming TCP connections**.
2. Active, in-flight HTTP requests are allowed a configurable grace period (`spring.lifecycle.timeout-per-shutdown-phase = 30s`) to complete their execution and return responses.
3. Thread pools (`ThreadPoolTaskExecutor`) and scheduled tasks finish in-flight jobs.
4. Once all active requests complete (or the 30s timeout expires), Spring invokes `@PreDestroy` methods and terminates the JVM cleanly with Exit Code 0.
</details>

---

### Q6: Why was `RestTemplate` deprecated in favor of `RestClient` in Spring Boot 3.2+?
<details>
<summary>Reveal Answer</summary>

**Answer**:
- `RestTemplate` is a legacy, synchronous, template-based client that suffered from bloated method overloading (over 40 overloaded methods for `exchange`, `getForObject`, `postForEntity`).
- **Spring Boot 3.2+ introduced `RestClient`**:
  - Provides a modern, fluent, declarative API (mirroring `WebClient`, but synchronous without reactive WebFlux dependencies).
  - Integrates natively with HTTP interfaces (`@HttpExchange`).
  - Provides built-in support for RFC 7807 `ProblemDetail` error response parsing and modern Apache HttpClient 5 / JDK `HttpClient` request factories.
</details>

---

### Q7: How do you prevent circular dependency errors in Spring Boot 3 without setting `spring.main.allow-circular-references = true`?
<details>
<summary>Reveal Answer</summary>

**Answer**:
In Spring Boot 3, circular references are **prohibited by default**. Setting `allow-circular-references = true` is a code smell.
**Production Solutions**:
1. **Refactor Design (Extract Common Service)**: Break the circular relationship between Service A and Service B by extracting the shared logic into a new Service C.
2. **Event-Driven Decoupling**: Replace direct synchronous method calls with Spring Application Events (`ApplicationEventPublisher` and `@EventListener`).
3. **Use `@Lazy` Injection**: Inject one of the dependencies with `@Lazy private ServiceB serviceB;` so that Spring generates a CGLIB proxy that instantiates the bean only upon first method invocation.
</details>

---

### Q8: What is the purpose of Spring Boot Actuator `/actuator/health` probe grouping for Kubernetes?
<details>
<summary>Reveal Answer</summary>

**Answer**:
In Kubernetes, Liveness and Readiness probes require different component health checks:
- **`management.endpoint.health.group.liveness`**: Must include **only internal JVM state** (`livenessState`). If an internal deadlock occurs, it fails and Kubernetes restarts the pod.
- **`management.endpoint.health.group.readiness`**: Includes external dependencies (`readinessState`, `db`, `redis`, `kafka`). If the database blips, the readiness probe fails and Kubernetes temporarily removes the pod from the Service load balancer without crash-restarting the container.
</details>
