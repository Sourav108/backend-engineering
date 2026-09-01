# Service-to-Service Authentication: mTLS, Signed JWTs, and Zero-Trust Identity

---

## 1. What Is It?
**Service-to-Service Authentication (Machine-to-Machine / M2M Auth)** is the cryptographic process by which autonomous backend microservices verify the identity and permissions of peer calling services before granting access to internal APIs.

The 3 primary internal authentication models are:
1. **Shared API Keys / Tokens**: Static secrets passed in HTTP headers.
2. **Signed Asymmetric JWTs (OAuth 2.0 Client Credentials Grant)**: Cryptographically signed bearer tokens verified via public keys.
3. **Mutual TLS (mTLS) & SPIFFE/SPIRE**: Bi-directional cryptographic X.509 certificate validation at the TCP/TLS transport layer, forming the foundation of **Zero-Trust Service Meshes**.

---

## 2. Why Does It Exist?
Historically, legacy networks relied on **Perimeter Security ("Castle-and-Moat")**:
- Everything inside the private corporate VPC was assumed to be trusted.
- All internal microservices communicated over plain-text HTTP (`http://order-service:8080`) with **zero authentication**.

**Why Castle-and-Moat Failed**:
- A single compromised container or SSRF vulnerability allows an attacker to pivot internally and query any private microservice or database without credentials.
- **Zero-Trust Principle**: *"Never Trust, Always Verify."* Every single internal network request must be cryptographically authenticated, authorized, and encrypted in transit.

---

## 3. Mental Model: Standard TLS vs Mutual TLS (mTLS)

```mermaid
sequenceDiagram
    autonumber
    participant Client as Order Service (Client)
    participant Server as Payment Service (Server)

    Note over Client,Server: Standard One-Way TLS (Client verifies Server ONLY)
    Client->>Server: ClientHello
    Server-->>Client: ServerHello + Server X.509 Certificate
    Client->>Client: Verify Server Certificate against Trusted CA Root
    Note over Client,Server: Client knows Server is genuine. Server has NO IDEA who Client is!

    Note over Client,Server: Mutual TLS (mTLS - Both Sides Cryptographically Authenticated)
    Server->>Client: CertificateRequest (Prove your identity!)
    Client-->>Server: Client X.509 Certificate (Signed by Internal Root CA)
    Server->>Server: Verify Client Certificate & Extract Service Identity (SPIFFE ID)
    Note over Client,Server: Both sides prove identity; 100% encrypted & authenticated!
```

---

## 4. How Does It Work?

### 1. SPIFFE IDs & Service Mesh Workload Identity
In modern cloud-native architectures (Istio / Linkerd / SPIRE):
- Each microservice is issued an X.509 certificate containing a **SPIFFE ID (Secure Production Identity Framework for Everyone)** in its Subject Alternative Name (SAN):
  $$\texttt{spiffe://cluster.internal/ns/production/sa/payment-service}$$
- When Order Service connects to Payment Service over mTLS, Payment Service extracts the SPIFFE ID and validates:
  - Is caller from the `production` namespace?
  - Is caller authorized to invoke `/api/v1/charge`?

---

### 2. OAuth 2.0 Client Credentials Grant (Signed JWTs)
For cross-datacenter or non-mesh architectures:
1. Calling service authenticates against an Identity Provider (Keycloak / Okta / Auth0) using its `client_id` and `client_secret`.
2. Identity Provider returns an **Asymmetrically Signed JWT (RS256 / ES256)** with a 15-minute TTL.
3. Receiving service verifies the cryptographic signature using the Identity Provider's public keys (**JWKS endpoint: `/.well-known/jwks.json`**) with **zero database lookups**.

```mermaid
flowchart LR
    Caller["Order Service"] -->|1. Authenticate with Client Credentials| IdP["Identity Provider (Keycloak)"]
    IdP -->|2. Return Signed JWT (RS256)| Caller
    Caller -->|3. Call with Bearer JWT| Receiver["Payment Service"]
    Receiver <-->|4. Cached Public Keys (JWKS)| IdP
```

---

## 5. Implementation: Spring Security OAuth2 Resource Server Configuration

```java
package com.backend.engineering.communication.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class InternalServiceSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // Stateless M2M APIs do not require CSRF
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll() // Public health checks
                .requestMatchers("/api/v1/internal/admin/**").hasAuthority("SCOPE_internal.admin")
                .requestMatchers("/api/v1/internal/payments/**").hasAuthority("SCOPE_payment.write")
                .anyRequest().authenticated()
            )
            // Cryptographic JWT Signature Verification using JWKS Public Keys
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }
}
```

---

## 6. Performance & Security Comparison

| Auth Mechanism | Transport Encryption | Verification Latency | Certificate Rotation Complexity | Security Grade |
|---|---|---|---|:---:|
| Shared API Key | Optional | $< 0.1\text{ms}$ | Complex (Hardcoded secrets leak) | ⚠️ Low |
| **Signed JWT (OAuth 2.0)**| Requires HTTPS | **$< 0.5\text{ms}$ (Local public key)** | Automatic (Keycloak key rotation) | **High** |
| **mTLS (X.509)** | **Mandatory (TLS 1.3)** | **$0\text{ms}$ per request (At handshake)** | Automated via Service Mesh Sidecars | **Maximum (Zero-Trust)** |

---

## 7. Interview Questions

### Q1: What is the primary difference between JWT-based service authentication and Mutual TLS (mTLS)?
<details>
<summary>Reveal Answer</summary>

**Answer**:
1. **Layer of Enforcement**:
   - **JWT Authentication** operates at the **Application Layer (L7)**. The client establishes a standard TLS connection, sends an HTTP request, and passes a signed JWT string in the `Authorization: Bearer <token>` header. The receiving application framework (e.g. Spring Security) parses and cryptographically validates the token.
   - **Mutual TLS (mTLS)** operates at the **Transport Layer (L4/L7)** during the initial TLS handshake. Both client and server exchange and cryptographically verify each other's X.509 digital certificates *before* any HTTP bytes are ever transmitted.
2. **Operational Characteristics**:
   - JWTs allow fine-grained user delegation (e.g. carrying end-user context alongside service identity), but require token refresh lifecycles.
   - mTLS provides non-forgeable hardware/workload identity with zero application code overhead (when offloaded to Envoy service mesh sidecars), guaranteeing 100% wire encryption and strict Zero-Trust compliance.
</details>

---

## 8. Quick Revision
- **Zero-Trust**: Never trust internal network perimeters; verify every request.
- **mTLS**: Dual-ended X.509 certificate validation during TLS handshake.
- **SPIFFE ID**: Standardized uniform workload URI embedded in certificate SAN.
- **Signed JWT (RS256)**: Verified locally using cached JWKS public keys without network roundtrips.
- **Offload to Mesh**: Service meshes (Istio/Linkerd) manage mTLS certificates automatically with zero application code changes.
