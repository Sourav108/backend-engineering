# Spring Boot 3.3.4 & Microservices Cheat Sheet

---

## ⚡ 1. Transaction Boundaries (`@Transactional`)

```java
@Transactional(
    isolation = Isolation.READ_COMMITTED,
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class, // Rollback on Checked & Unchecked exceptions!
    timeout = 5                    // 5-second transaction ceiling
)
public void executeFinancialTransfer(Long fromId, Long toId, Long amount) { ... }
```

> [!CAUTION]
> **Self-Invocation Bug**: Calling `@Transactional` method within the same class bypasses CGLIB proxy. Move to separate bean.

---

## ⚡ 2. Modern HTTP Client (`RestClient`)

```java
RestClient restClient = RestClient.builder()
    .baseUrl("https://api.payment.internal")
    .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient5))
    .build();

PaymentDto response = restClient.post()
    .uri("/v1/charge")
    .contentType(MediaType.APPLICATION_JSON)
    .body(new ChargeRequest(101L, 5000L))
    .retrieve()
    .onStatus(HttpStatusCode::is5xxServerError, (req, res) -> {
        throw new PaymentGatewayException("Payment gateway unavailable");
    })
    .body(PaymentDto.class);
```

---

## ⚡ 3. Spring Security 6 Filter Chain (Stateless JWT)

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health/**").permitAll()
            .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

---

## ⚡ 4. Actuator Health Probe Grouping

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState # Internal JVM ONLY! Never include DB here!
        readiness:
          include: readinessState, db, redis, kafka
```
