# Technical Diagram Standards (Mermaid)

All technical documentation in this repository must use clean, unambiguous, and renderer-safe Mermaid syntax.

---

## 📐 Supported Diagram Types

### 1. Request Lifecycles & Architecture (`flowchart TD` / `flowchart LR`)
```mermaid
flowchart LR
    Client["Client / Mobile App"] --> DNS["DNS Resolution"]
    DNS --> LB["ALB / Reverse Proxy"]
    LB --> Gateway["API Gateway"]
    Gateway --> App["Spring Boot 3 (Java 21)"]
    App --> Cache[("Redis Cache")]
    App --> DB[("PostgreSQL Master")]
```

### 2. Service Protocol & Handshake Sequences (`sequenceDiagram`)
```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant Gateway as API Gateway
    participant Auth as Auth Service
    participant Backend as Core Backend
    participant DB as PostgreSQL

    Client->>Gateway: POST /api/v1/orders (Bearer JWT)
    Gateway->>Auth: Validate Token Signature
    Auth-->>Gateway: 200 OK (Claims Verified)
    Gateway->>Backend: Forward Request (X-User-Id: 101)
    Backend->>DB: INSERT order INTO orders
    DB-->>Backend: 1 row affected
    Backend-->>Client: 201 Created (OrderDTO)
```

### 3. State Machines (`stateDiagram-v2`)
```mermaid
stateDiagram-v2
    [*] --> Closed: Normal Operation
    Closed --> Open: Failure Rate > 50%
    Open --> HalfOpen: Sleep Window Elapsed (10s)
    HalfOpen --> Closed: Probe Requests Succeed
    HalfOpen --> Open: Probe Request Fails
```
