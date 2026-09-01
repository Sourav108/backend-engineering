# System Design Mathematical Formulas & Estimation Cheat Sheet

---

## ⚡ 1. Capacity & Traffic Estimation Math

### Seconds in a Day:
$$1\text{ Day} = 86,400\text{ seconds} \approx \mathbf{10^5\text{ seconds}}$$

---

### QPS & TPS Conversions:
$$\text{Average QPS} = \frac{\text{Daily Active Users (DAU)} \times \text{Requests per User}}{86,400}$$

$$\text{Peak QPS} = \text{Average QPS} \times 2 \quad (\text{or } 3\times)$$

- *Example*: $100\text{M DAU}$ making $10\text{ reqs/day}$:
  $$\text{Average QPS} = \frac{10^8 \times 10}{10^5} = \mathbf{10,000\text{ QPS}} \implies \text{Peak QPS} \approx \mathbf{20,000 - 30,000\text{ QPS}}$$

---

## ⚡ 2. Storage Estimation Math

$$\text{Storage per Year} = \text{Daily Writes} \times \text{Average Payload Size} \times 365$$

- *Example*: $10\text{M photos/day} \times 2\text{MB}$:
  $$\text{Storage/Day} = 20\text{TB/day} \implies \text{Storage/Year} = 20\text{TB} \times 365 \approx \mathbf{7.3\text{ PB/year}}$$

---

## ⚡ 3. Memory Caching Estimation (80/20 Pareto Rule)

$$\text{Cache RAM Required} = \text{Daily Read Volume} \times 20\%$$

- *Example*: $5\text{TB of daily reads} \times 0.20 = \mathbf{1\text{TB RAM Redis Cluster}}$.

---

## ⚡ 4. Concurrency Laws

### Little's Law (Queue Sizing):
$$L = \lambda \times W$$
*($L$ = Number of concurrent requests in system, $\lambda$ = Arrival rate QPS, $W$ = Average latency)*
- *Example*: At $10,000\text{ QPS}$ with $50\text{ms}$ ($0.05\text{s}$) latency:
  $$L = 10,000 \times 0.05 = \mathbf{500\text{ Concurrent In-Flight Connections}}$$

---

### Tail Latency Amplification Law (Jeff Dean):
$$P(\text{Slow User Request}) = 1 - (1 - p)^N$$
*($p$ = Probability of single service being slow, $N$ = Number of parallel microservice calls)*
- *Example*: $N = 100\text{ microservices}$, $p = 0.01$ ($1\%$ tail):
  $$P(\text{Slow}) = 1 - (0.99)^{100} = \mathbf{63.4\% \text{ of User Requests Experience Slow Tail!}}$$
