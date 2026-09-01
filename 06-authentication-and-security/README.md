# Module 06: Authentication & Security

Master enterprise backend security: JWT signing and JWKS key rotation, OAuth 2.1 Authorization Code with PKCE, Refresh Token Rotation with reuse detection, Redis session management with CSRF defense, RBAC/ABAC authorization, OWASP API Top 10 defenses (BOLA/IDOR), and Envelope Encryption with AES-256-GCM.

---

## 🎯 Learning Objectives
- Architect stateless authentication with **JWT RS256/ES256 and dynamic JWKS key rotation**.
- Eliminate token theft vulnerabilities using **OAuth 2.1 PKCE and Refresh Token Rotation with automatic revocation**.
- Defend against CSRF and Session Hijacking using **`HttpOnly`, `SameSite=Strict`, and Synchronizer Tokens**.
- Implement dynamic **Attribute-Based Access Control (ABAC)** and SpEL expression evaluators in Spring Security.
- Harden APIs against **OWASP API Top 10 (BOLA/IDOR, Mass Assignment, SSRF)**.
- Secure data at rest using **Envelope Encryption (DEK/KEK)** with AES-256-GCM and HashiCorp Vault.

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**JWT Architecture & Cryptographic Signing**](./01-jwt-architecture-and-cryptographic-signing.md) | RS256 vs HS256, JWKS rotation, Redis Token Revocation Blocklist | Mermaid, Java 21 |
| **02** | [**OAuth 2.1 & OIDC Flows**](./02-oauth2-and-oidc-flows.md) | Authorization Code + PKCE, Client Credentials, Refresh Token Reuse Detection | Mermaid, Java 21 |
| **03** | [**Session Management & Cookie Security**](./03-session-management-and-cookie-security.md) | Redis Sessions, `SameSite=Strict`, `HttpOnly`, CSRF Double-Submit / Synchronizer | Mermaid, Java 21 |
| **04** | [**RBAC & ABAC Access Control**](./04-role-based-and-attribute-based-access-control.md) | Roles vs Permissions vs Attributes, SpEL Security, PEP / PDP Architecture | Mermaid, Java 21 |
| **05** | [**OWASP API Top 10 & Vulnerabilities**](./05-api-security-and-top-vulnerabilities.md) | BOLA / IDOR Defense, Mass Assignment, SSRF, Input Sanitization | Mermaid, Java 21 |
| **06** | [**Data Encryption & Secrets Management**](./06-data-encryption-and-secrets-management.md) | Envelope Encryption, AES-256-GCM, DEK / KEK, Argon2id & BCrypt Hashing | Mermaid, Java 21 |

---

## 🛠️ Verification & Drills
- Test JWT revocation latency using Redis blocklist integration tests.
- Simulate OAuth 2.1 PKCE verification and malicious Refresh Token replay detection.
