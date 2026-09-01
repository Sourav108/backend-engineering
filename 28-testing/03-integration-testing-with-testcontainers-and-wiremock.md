# Integration Testing with Testcontainers and WireMock

---

## 1. What Is It?
In production backend engineering:
- **Testcontainers**: A Java testing framework that orchestrates lightweight, ephemeral Docker containers (PostgreSQL, Redis, Apache Kafka, Elasticsearch) directly within JUnit 5 integration tests.
- **WireMock**: An HTTP mock server that stubs external third-party HTTP/REST APIs (e.g. Stripe, Twilio, OpenAI), enabling deterministic testing of edge failure scenarios (timeouts, 503 errors, malformed payloads).

---

## 2. Why Does It Exist?
- **Shared Staging Database Collisions**: Testing against a persistent staging database causes test data collisions (Test A deletes a user row that Test B is actively reading), causing flaky test builds.
- **Third-Party API Outages in CI**: Integration tests calling real external sandbox APIs (e.g. Stripe Sandbox) fail whenever third-party sandboxes undergo maintenance or rate limit CI runners.
- **Testcontainers & WireMock**: Provide **$100\%$ isolated, hermetic, repeatable integration test environments** that spin up and tear down cleanly inside Docker.

---

## 3. Mental Model: The Hermetic Integration Testing Environment

```mermaid
flowchart TD
    subgraph JUnit5Runner["JUnit 5 Integration Test (mvn verify -Pintegration)"]
        Test["PaymentProcessingIT"]
    end

    subgraph TestcontainersCluster["Ephemeral Docker Containers (Dynamic Mapped Ports)"]
        PostgresContainer[("PostgreSQL 16.4 Container (Dynamic Port: 54321)")]
        KafkaContainer[("Apache Kafka KRaft Container (Dynamic Port: 9093)")]
    end

    subgraph WireMockServer["WireMock HTTP Mock Server"]
        StripeMock["WireMock Stripe Stub (http://localhost:8089/v1/charges)"]
    end

    Test -->|@DynamicPropertySource (JDBC URL)| PostgresContainer
    Test -->|@DynamicPropertySource (Bootstrap Servers)| KafkaContainer
    Test -->|HTTP POST (Injected BaseUrl)| StripeMock
```

---

## 4. How Does It Work?

### 1. The Singleton Container Pattern (Performance Optimization)
Starting a new PostgreSQL and Kafka Docker container for every single test class adds $15\text{ seconds}$ per class ($10\text{ classes} = 2.5\text{ minutes}$).
- **The Singleton Pattern**: A static base class starts the containers **once per JVM test run**. All integration test classes share the exact same running containers, reducing total suite runtime from minutes to **25 seconds**!

---

### 2. `@DynamicPropertySource` Configuration Injection
Because Testcontainers binds containers to random, conflict-free host ports (e.g. PostgreSQL port 5432 maps to host port `32789`), the Spring Boot `Environment` properties must be updated dynamically *after* the container starts:

```java
@DynamicPropertySource
static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
}
```

---

## 5. Implementation: Full Integration Test with Testcontainers & WireMock

### 1. Abstract Base Integration Test Class
```java
package com.backend.engineering.testing.integration;

import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import org.junit.jupiter.api.extension.RegisterExtension;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public abstract class AbstractIntegrationTest {

    // 1. Singleton PostgreSQL 16 Container
    @Container
    protected static final PostgreSQLContainer<?> postgres = 
            new PostgreSQLContainer<>("postgres:16.4-alpine")
                    .withDatabaseName("testdb")
                    .withUsername("testuser")
                    .withPassword("testpass");

    // 2. WireMock Server for Third-Party API Stubbing
    @RegisterExtension
    protected static WireMockExtension wiremock = WireMockExtension.newInstance()
            .options(wireMockConfig().dynamicPort())
            .build();

    // 3. Dynamic Spring Property Binding
    @DynamicPropertySource
    static void dynamicProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        
        // Point third-party client to WireMock URL!
        registry.add("stripe.api.base-url", wiremock::baseUrl);
    }
}
```

---

### 2. Concrete Integration Test with Failure Scenario Simulation
```java
package com.backend.engineering.testing.integration;

import com.backend.engineering.repository.PaymentRepository;
import com.backend.engineering.service.PaymentService;
import com.github.tomakehurst.wiremock.client.WireMock;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

public class PaymentServiceIT extends AbstractIntegrationTest {

    @Autowired
    private PaymentService paymentService;

    @Autowired
    private PaymentRepository paymentRepository;

    @Test
    @DisplayName("Should successfully authorize payment and persist in real PostgreSQL")
    void shouldProcessPaymentWithStripe() {
        // 1. Stub WireMock Stripe API Response
        wiremock.stubFor(post(urlEqualTo("/v1/charges"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("{\"id\": \"ch_3M456\", \"status\": \"succeeded\"}")));

        // 2. Execute Business Logic
        var result = paymentService.chargeCustomer(101L, 5000L);

        // 3. Assertions
        assertThat(result.isSuccess()).isTrue();
        assertThat(paymentRepository.findByUserId(101L)).isPresent();
    }

    @Test
    @DisplayName("Should handle 500 Service Unavailable from Stripe and trigger retry")
    void shouldHandleStripe500Error() {
        // 1. Simulate Stripe 500 Internal Server Error
        wiremock.stubFor(post(urlEqualTo("/v1/charges"))
                .willReturn(aResponse()
                        .withStatus(500)
                        .withBody("{\"error\": \"Internal server error\"}")));

        // 2. Verify application resilience exception handling
        assertThatThrownBy(() -> paymentService.chargeCustomer(101L, 5000L))
                .isInstanceOf(PaymentGatewayException.class);
    }
}
```

---

## 6. Performance

| Integration Strategy | Test Isolation | Setup Friction | Reliability Confidence |
|---|---|---|:---:|
| Shared Remote Staging DB | ❌ Flaky (Data collisions) | High (Requires DB setup) | Moderate |
| **Testcontainers + WireMock** | ✅ **100% Hermetic & Isolated** | **Zero (Managed by Docker)** | **Maximum (100% Production Parity)** |

---

## 7. Interview Questions

### Q1: Why should you use the Singleton Container pattern in Testcontainers instead of restarting containers before each test class?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Container Startup Latency Penalty**:
   - Starting a PostgreSQL container, executing entrypoint scripts, running Flyway/Liquibase migrations, and starting a Kafka broker takes **$5 - 15\text{ seconds}$ per startup**.
   - If a test suite contains 20 integration test classes and restarts containers for each class, the test suite spends **$3 - 5\text{ minutes}$ purely starting and stopping Docker containers**.
2. **The Singleton Container Solution**:
   - By declaring container instances as static fields in a shared abstract base class, Testcontainers starts the containers **once when the JVM launches**.
   - All 20 test classes share the running container instances. Test isolation is maintained by truncating tables or using transactional rollback annotations (`@Transactional`), reducing total test suite execution time from 5 minutes to **under 30 seconds**.
</details>

---

## 8. Quick Revision
- **Testcontainers**: Orchestrates real Docker containers in JUnit 5 integration tests.
- **WireMock**: Stubs external HTTP APIs to test error codes and timeouts deterministically.
- **`@DynamicPropertySource`**: Injects dynamic container host/ports into Spring configuration.
- **Singleton Container Pattern**: Starts containers once per test run to save minutes of CI build time.
- **No Staging Pollution**: Tests run in hermetic, isolated, self-contained Docker environments.
