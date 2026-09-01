# Module 04: Networking for Backend Engineers

Master the low-level network protocols and kernel mechanics that power backend infrastructure: TCP/IP packet encapsulation, sliding window flow control, congestion control (Cubic vs BBR), TIME_WAIT socket states, QUIC/HTTP/3, and Mutual TLS (mTLS).

---

## 🎯 Learning Objectives
- Deconstruct the **OSI 7-Layer and TCP/IP 4-Layer Models** with byte-level packet encapsulation.
- Master **TCP Sliding Window, Flow Control, and Congestion Control** algorithms (Reno, Cubic, BBR).
- Diagnose and tune kernel socket states (**TIME_WAIT, CLOSE_WAIT leaks, `SO_REUSEADDR`, `TCP_NODELAY`**).
- Understand **UDP, QUIC packet framing, and HTTP/3 connection migration**.
- Implement **Mutual TLS (mTLS)** for zero-trust microservice communication.
- Debug live production network outages using **`tcpdump`, `ss`, `mtr`, and Wireshark**.

---

## 📚 Lessons Catalog

| # | Lesson | Key Concepts | Code / Diagrams |
|:---:|---|---|:---:|
| **01** | [**OSI Model & Packet Encapsulation**](./01-osi-model-and-packet-encapsulation.md) | OSI vs TCP/IP, MTU 1500B, IP Fragmentation, MSS, Ethernet Frame | Mermaid, Java 21 |
| **02** | [**TCP Mechanics & Congestion Control**](./02-tcp-ip-mechanics-and-congestion-control.md) | Sliding Window, Flow vs Congestion Control, Reno vs Cubic vs BBR, Nagle's Algorithm | Mermaid, Java 21 |
| **03** | [**TCP Connection States & Kernel Tuning**](./03-tcp-connection-states-and-kernel-tuning.md) | 11 TCP States, TIME_WAIT, CLOSE_WAIT leaks, `tcp_tw_reuse`, Buffer tuning | Mermaid, Java 21 |
| **04** | [**UDP, QUIC & HTTP/3 Architecture**](./04-udp-quic-and-http3-architecture.md) | UDP Datagrams, Head-of-Line Blocking, QUIC Crypto, Connection IDs | Mermaid, Java 21 |
| **05** | [**TLS 1.3 & Mutual TLS (mTLS)**](./05-tls-and-mtls-cryptography.md) | Symmetric vs Asymmetric, ECC/ECDSA, CA Chains, mTLS Service Mesh | Mermaid, Java 21 |
| **06** | [**Network Diagnostics & Troubleshooting**](./06-network-troubleshooting-and-diagnostics.md) | `tcpdump`, `ss`, Wireshark, MTU Blackholes, Packet Drops | Shell, Java 21 |

---

## 🛠️ Verification & Drills
- Run socket test drills with Java 21 `java.net.Socket` and `SocketOptions`.
- Inspect TCP socket states using `ss -s` on local terminals.
