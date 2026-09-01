# Unit and Web Slice Testing: MockMvc vs WebTestClient in Spring Boot 3.3.4

---

## 1. What Is It?
In Spring Boot testing architecture:
- **Unit Testing**: Testing pure domain aggregates, business entities, and calculation services in complete isolation using JUnit 5, AssertJ, and Mockito without bootstrapping the Spring application context.
- **Web Slice Testing (`@WebMvcTest`)**: An optimized test slice that loads **only the web layer** (Controllers, JSON Serialization, Filters, and `@ControllerAdvice`), mocking all downstream service beans.
- **`MockMvc`**: A mock HTTP execution engine that dispatches requests directly into the Spring DispatcherServlet in memory without opening a physical TCP network socket.
- **`WebTestClient`**: A non-blocking HTTP client that executes real HTTP network calls against a live Spring Boot server listening on a dynamic random port (`@SpringBootTest(webEnvironment = RANDOM_PORT)`).

---

## 2. Why Does It Exist?
Bootstrapping the full `@SpringBootTest` context for every single controller test:
- Loads the entire application context, initializes Hibernate entity managers, connects to caches, and takes **$8 - 15\text{ seconds}$ per test class**.
- Running 50 controller tests takes **10 minutes**.
- **`@WebMvcTest` starts up in $< 400\text{ms}$**, allowing developers to test HTTP status codes, JSON validation errors (RFC 7807 `ProblemDetail`), and security headers at lightning speed.

---

## 3. Mental Model: `MockMvc` vs `WebTestClient`

```mermaid
flowchart TD
    subgraph MockMvcModel["1. @WebMvcTest + MockMvc (In-Memory Servlet Dispatcher)"]
        Test1["JUnit Test"] --> MockMvcEngine["MockMvc In-Memory Dispatcher (0 Network Sockets)"]
        MockMvcEngine --> Controller1["PaymentController (Spring MVC Slice)"]
        Controller1 --> MockSvc["@MockBean PaymentService"]
        Note over MockMvcModel: Startup Time: < 400ms! Lightning fast!
    end

    subgraph LiveClientModel["2. @SpringBootTest(RANDOM_PORT) + WebTestClient (Real TCP)"]
        Test2["JUnit Test"] --> RealSocket["Real TCP Socket (http://localhost:54321)"]
        RealSocket --> NettyTomcat["Embedded Tomcat Server"]
        NettyTomcat --> FullApp["Full Spring Boot Context + DB Connection"]
        Note over LiveClientModel: Startup Time: 8-15 seconds. High end-to-end fidelity!
    end
```

---

## 4. Implementation: Fast Web Slice Test with `MockMvc` (`@WebMvcTest`)

```java
package com.backend.engineering.testing.web;

import com.backend.engineering.controller.PaymentController;
import com.backend.engineering.dto.PaymentRequestDto;
import com.backend.engineering.dto.PaymentResponseDto;
import com.backend.engineering.service.PaymentService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(PaymentController.class)
public class PaymentControllerWebMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private PaymentService paymentService; // Mocks the business layer!

    @Test
    @DisplayName("POST /api/v1/payments - Should return 200 OK and JSON response")
    void shouldAuthorizePaymentSuccessfully() throws Exception {
        // Given
        PaymentRequestDto request = new PaymentRequestDto(101L, 5000L, "USD");
        PaymentResponseDto expectedResponse = new PaymentResponseDto("TX-9988", "AUTHORIZED", 5000L);

        given(paymentService.processPayment(any())).willReturn(expectedResponse);

        // When & Then
        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.transactionId").value("TX-9988"))
                .andExpect(jsonPath("$.status").value("AUTHORIZED"))
                .andExpect(jsonPath("$.amountCents").value(5000));
    }

    @Test
    @DisplayName("POST /api/v1/payments - Should return 400 Bad Request on invalid input (RFC 7807)")
    void shouldReturn400WhenAmountIsNegative() throws Exception {
        // Given invalid negative amount
        PaymentRequestDto invalidRequest = new PaymentRequestDto(101L, -500L, "USD");

        // When & Then
        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidRequest)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.title").value("Bad Request"))
                .andExpect(jsonPath("$.detail").exists());
    }
}
```

---

## 5. Implementation: Live Network Integration Test with `WebTestClient`

```java
package com.backend.engineering.testing.web;

import com.backend.engineering.dto.PaymentRequestDto;
import com.backend.engineering.dto.PaymentResponseDto;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.web.reactive.server.WebTestClient;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class PaymentLiveNetworkIT {

    @Autowired
    private WebTestClient webTestClient;

    @LocalServerPort
    private int port;

    @Test
    void shouldExecuteOverRealTcpSocket() {
        PaymentRequestDto request = new PaymentRequestDto(101L, 2500L, "USD");

        PaymentResponseDto response = webTestClient.post()
                .uri("/api/v1/payments")
                .bodyValue(request)
                .exchange()
                .expectStatus().isOk()
                .expectBody(PaymentResponseDto.class)
                .returnResult()
                .getResponseBody();

        assertThat(response).isNotNull();
        assertThat(response.status()).isEqualTo("AUTHORIZED");
    }
}
```

---

## 6. Performance & Testing Strategy Matrix

| Test Style | Scope | Startup Overhead | Real TCP Socket? | Best Use Case |
|---|---|---|:---:|---|
| **Pure JUnit 5 Unit Test** | Single Class / Aggregate | **$0\text{ms}$** | ❌ No | Domain logic, math, algorithms |
| **`@WebMvcTest` (MockMvc)** | Web Controller Slice | **$\approx 350\text{ms}$** | ❌ No | Input validation, HTTP status codes, security |
| **`@SpringBootTest(RANDOM_PORT)`**| Full Live Container | $\approx 8 - 15\text{s}$ | ✅ **Yes** | Full end-to-end integration flows |

---

## 7. Interview Questions

### Q1: What is the main architectural advantage of `@WebMvcTest` over `@SpringBootTest` when testing Spring Boot REST APIs?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Targeted Context Loading (Speed)**:
   - `@SpringBootTest` instantiates the entire Spring ApplicationContext (all `@Service`, `@Repository`, `@Component`, JPA entity managers, database connection pools, and message listeners).
   - `@WebMvcTest` loads **only the Web Layer components** (`@Controller`, `@ControllerAdvice`, `@JsonComponent`, `Filter`, `WebMvcConfigurer`), mocking all backend services via `@MockBean`.
2. **Execution Velocity**:
   - Because it skips initializing databases, caches, and heavy infrastructure beans, `@WebMvcTest` executes in **less than 400 milliseconds**, enabling fast feedback loops for testing HTTP routing, JSON serialization/deserialization, input validation annotations (`@Valid`, `@NotNull`), and error response formatting.
</details>

---

## 8. Quick Revision
- **Unit Tests**: Pure JUnit 5 + Mockito without Spring context; instant execution ($< 1\text{ms}$).
- **`@WebMvcTest`**: Fast web-slice test loading only controllers and serializers.
- **`MockMvc`**: In-memory dispatcher servlet mock (0 physical network ports).
- **`WebTestClient`**: Real HTTP client against live random TCP port.
- **RFC 7807 `ProblemDetail`**: Standardized JSON error response format in Spring Boot 3.
