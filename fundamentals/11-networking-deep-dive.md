# Networking Deep Dive for Distributed Systems

> Understanding networking at the systems level is essential for diagnosing latency issues, designing reliable communication patterns, and passing FAANG system design interviews.

---

## 1. The OSI Model in Practice

### 1.1 Layers That Matter for Distributed Systems

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OSI MODEL RELEVANCE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Layer 7: Application    ←── HTTP, gRPC, DNS, TLS                      │
│         (Your code)           Your API calls, service communication    │
│                                                                          │
│  Layer 4: Transport      ←── TCP, UDP                                   │
│         (Reliability)         Connection handling, reliability          │
│                                                                          │
│  Layer 3: Network       ←── IP, ICMP                                    │
│         (Routing)           How packets get from A to B                │
│                                                                          │
│  Layer 2: Data Link     ←── Ethernet, WiFi                              │
│         (Local)             Frames on local network                    │
│                                                                          │
│  Layer 1: Physical      ←── Cables, switches                            │
│         (Bits)              Fiber, copper, hardware                    │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│  KEY INSIGHT: Distributed systems engineers work primarily at          │
│  Layers 4 (TCP/UDP) and 7 (HTTP/gRPC). Understanding these is          │
│  essential for diagnosing issues and designing reliable systems.        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. TCP Deep Dive

### 2.1 TCP vs UDP: When to Use Each

| Feature | TCP | UDP |
|---------|-----|-----|
| **Reliability** | Guaranteed delivery | Best effort |
| **Ordering** | Guaranteed | Not guaranteed |
| **Connection** | Stateful (handshake) | Stateless |
| **Overhead** | ~20 bytes + ACK overhead | ~8 bytes |
| **Latency** | Higher (retransmission) | Lower (no retransmission) |
| **Use Cases** | HTTP, gRPC, Database | DNS, VoIP, Gaming, Video |

### 2.2 TCP Handshake and Latency Cost

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TCP THREE-WAY HANDSHAKE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Client                                                          Server │
│     │                                                                │   │
│     │  ┌─────────────────────────────────────────────────────────┐   │   │
│     │  │ SYN (seq=x)                                           │   │   │
│     │  │  • Client picks initial sequence number                │   │   │
│     │  │  • Client enters SYN_SENT state                       │   │   │
│     │  └─────────────────────────────────────────────────────────┘   │   │
│     │ ──────────────────────────────────────────────────────────────→  │
│     │                                                                │   │
│     │  ┌─────────────────────────────────────────────────────────┐   │   │
│     │  │ SYN-ACK (seq=y, ack=x+1)                               │   │   │
│     │  │  • Server picks initial sequence number                │   │   │
│     │  │  • Acknowledges client's sequence                      │   │   │
│     │  └─────────────────────────────────────────────────────────┘   │   │
│     │ ←──────────────────────────────────────────────────────────────  │
│     │                                                                │   │
│     │  ┌─────────────────────────────────────────────────────────┐   │   │
│     │  │ ACK (ack=y+1)                                          │   │   │
│     │  │  • Client confirms server's sequence                   │   │   │
│     │  │  • Connection established                              │   │   │
│     │  └─────────────────────────────────────────────────────────┘   │   │
│     │ ──────────────────────────────────────────────────────────────→  │
│     │                                                                │   │
│   ════════════════════════════════════════════════════════════════════   │
│                                                                          │
│   TOTAL LATENCY: 1.5 RTT (Round Trip Time)                              │
│   • Example: 10ms between client and server → 15ms just to connect!   │
│                                                                          │
│   WHY THIS MATTERS:                                                     │
│   • Every new TCP connection costs 1.5 RTT before data flows           │
│   • For short-lived requests, this is significant overhead             │
│   • Solution: Connection pooling, HTTP/2 multiplexing, gRPC keepalive  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 TCP Sliding Window and Flow Control

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      TCP SLIDING WINDOW                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  WINDOW SIZE: How much data can be sent before requiring ACK           │
│                                                                          │
│  Example: Window = 4 segments                                           │
│                                                                          │
│  Send:    [1][2][3][4][5][6][7][8][9]...  (waiting for ACKs)          │
│           ├───┘                                 │                       │
│           Window: 4 unacked                   │                       │
│                                                  │                       │
│           ACKs received for 1,2 → window slides: │                       │
│                                                                          │
│  Send:    [1][2][3][4][5][6][7][8][9]...                                │
│               ├───┤   │                                                 │
│               │   │   └── Can send 5,6 now                              │
│               │   └───── Waiting for 3,4                                │
│               └───────── Already ACKed                                  │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│  FLOW CONTROL: Receiver advertises available buffer space             │
│  ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│  Receiver: "My receive buffer has 16KB free" → Window = 16KB           │
│  Sender: Won't send more than 16KB until ACK updates window             │
│                                                                          │
│  PROBLEM: Slow receiver can throttle fast sender (head-of-line blocking)│
│  SOLUTION: TCP_NODELAY to disable Nagle's algorithm for low-latency    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Congestion Control Algorithms

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  TCP CONGESTION CONTROL ALGORITHMS                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CONGESTION WINDOW (cwnd): How much the network can handle             │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     SLOW START                                    │  │
│  │                                                                    │  │
│  │   cwnd = 1 MSS → 2 → 4 → 8 → 16 → ...                          │  │
│  │   Exponential growth until packet loss or ssthresh               │  │
│  │                                                                    │  │
│  │   Example: Start at 1KB, grow to 1MB in ~10 RTTs                │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                               ↓                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                  CONGESTION AVOIDANCE                             │  │
│  │                                                                    │  │
│  │   cwnd = cwnd + MSS²/cwnd (additive increase)                   │  │
│  │   Linear growth: 1MB → 1.01MB → 1.02MB → ...                    │  │
│  │                                                                    │  │
│  │   On loss: cwnd = cwnd / 2 (multiplicative decrease)            │  │
│  │                                                                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│  ALGORITHMS IN PRODUCTION:                                              │
│                                                                          │
│  | Algorithm    | Description                          | Use Case         |   │
│  |--------------|--------------------------------------|------------------|   │
│  | Cubic        | Default Linux, fast recovery        | General purpose  |   │
│  | BBR          | Bottleneck bandwidth & RTT           | High bandwidth   |   │
│  | Reno         | Classic, single loss detection       | Legacy systems   |   │
│  | Westwood+    | Estimate bandwidth on loss           | Wireless networks |   │
│                                                                          │
│  ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│  BBR (Bottleneck Bandwidth and RTT):                                    │
│  • Measures actual bandwidth and RTT                                     │
│  • More aggressive than Cubic on high-latency networks                  │
│  • Used by Google, YouTube                                              │
│  • Can achieve 10-20% throughput improvement                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. DNS Deep Dive

### 3.1 DNS Resolution Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DNS RESOLUTION FLOW                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Client                                         DNS Server              │
│      │                                               │                    │
│      │  1. Check local cache                         │                    │
│      │     • OS hosts file                           │                    │
│      │     • Browser cache                           │                    │
│      │     • Application cache                       │                    │
│      │                                               │                    │
│      │ ───────────────────────────────────────────→ │                    │
│      │                                               │                    │
│      │  2. Query Recursive Resolver                  │                    │
│      │     (e.g., 8.8.8.8, 1.1.1.1)                │                    │
│      │ ───────────────────────────────────────────→ │                    │
│      │                                               │                    │
│      │  3. Root Server (.com, .org, etc.)           │                    │
│      │ ←─────────────────────────────────────────── │                    │
│      │     "Ask .com TLD server"                    │                    │
│      │                                               │                    │
│      │  4. TLD Server (.com)                        │                    │
│      │ ───────────────────────────────────────────→ │                    │
│      │     "Ask example.com authoritative"          │                    │
│      │ ←─────────────────────────────────────────── │                    │
│      │                                               │                    │
│      │  5. Authoritative Server                     │                    │
│      │ ───────────────────────────────────────────→ │                    │
│      │     "example.com = 93.184.216.34"            │                    │
│      │ ←─────────────────────────────────────────── │                    │
│      │                                               │                    │
│      │  6. Return to client + cache                 │                    │
│      │ ←─────────────────────────────────────────── │                    │
│      │                                               │                    │
│   ═══════════════════════════════════════════════════════════════════   │
│                                                                          │
│   TOTAL LATENCY: 20-100ms (can be cached for hours)                    │
│                                                                          │
│   PRODUCTION TIP: DNS caching at multiple levels is critical            │
│   • Browser: ~1 minute                                                   │
│   • OS: ~1 hour                                                         │
│   • Recursive resolver: TTL from record (typically 1-24 hours)           │
│                                                                          │
│   WHY THIS MATTERS FOR SYSTEM DESIGN:                                   │
│   • DNS is often a single point of failure                              │
│   • DNS TTL affects failover speed                                      │
│   • DDoS on DNS can take down entire services                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | `example.com → 93.184.216.34` |
| AAAA | IPv6 address | `example.com → 2606:2800:220:1` |
| CNAME | Canonical name | `www.example.com → example.com` |
| MX | Mail server | `example.com → mail.example.com` |
| TXT | Text records | `example.com → "v=spf1 include:_spf.google.com ~all"` |
| SRV | Service location | `_http._tcp.example.com → 10 5 80 www.example.com` |
| NS | Name server | `example.com → ns1.example.com` |
| SOA | Zone authority | Primary NS, serial, refresh timers |

### 3.3 DNS for High Availability

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HIGH-AVAILABILITY DNS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   STRATEGY 1: Multiple A Records (Simple Round Robin)                   │
│   ─────────────────────────────────────────────────────                 │
│                                                                          │
│   example.com → [1.2.3.4, 5.6.7.8, 9.10.11.12]                         │
│                                                                          │
│   • Client picks one randomly                                           │
│   • If one IP fails, client retries another                             │
│   • Simple, no extra infrastructure                                     │
│   • Problem: No health checking                                         │
│                                                                          │
│   STRATEGY 2: Geographic DNS (Route 53, CloudFlare)                    │
│   ─────────────────────────────────────────────────────                 │
│                                                                          │
│   us-east.example.com → [1.2.3.4]                                        │
│   us-west.example.com → [5.6.7.8]                                        │
│   eu.example.com → [9.10.11.12]                                         │
│                                                                          │
│   • Routes users to nearest region                                      │
│   • Health checks per IP/region                                         │
│   • Automatic failover                                                  │
│                                                                          │
│   STRATEGY 3: Anycast                                                  │
│   ─────────────────────────────────────────────────────                 │
│                                                                          │
│   Multiple servers announce same IP via BGP                             │
│   • Packets route to nearest healthy server                           │
│   • Built-in load balancing and failover                                │
│   • Used by CloudFlare, Google's DNS (8.8.8.8)                          │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│   PRODUCTION PATTERN:                                                   │
│                                                                          │
│   1. Use multiple DNS providers (avoid single point of failure)        │
│   2. Set low TTL for critical records (60-300 seconds)                │
│   3. Monitor DNS resolution latency                                     │
│   4. Use health checks for automatic failover                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. TLS/SSL Deep Dive

### 4.1 TLS Handshake

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TLS 1.2 HANDSHAKE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Client                                                            Server │
│      │                                                                │   │
│      │  ┌─────────────────────────────────────────────────────────┐   │   │
│      │  │ ClientHello                                            │   │   │
│      │  │  • TLS version                                         │   │   │
│      │  │  • Cipher suites (e.g., AES256-GCM, CHACHA20)         │   │   │
│      │  │  • Client random                                       │   │   │
│      │  │  • Server Name Indication (SNI)                        │   │   │
│      │  └─────────────────────────────────────────────────────────┘   │   │
│      │ ──────────────────────────────────────────────────────────────→  │
│      │                                                                │   │
│      │  ┌─────────────────────────────────────────────────────────┐   │   │
│      │  │ ServerHello                                            │   │   │
│      │  │  • Selected cipher suite                               │   │   │
│      │  │  • Server random                                       │   │   │
│      │  │  • Certificate (public key)                            │   │   │
│      │  └─────────────────────────────────────────────────────────┘   │   │
│      │ ←──────────────────────────────────────────────────────────────  │
│      │                                                                │   │
│      │  ┌─────────────────────────────────────────────────────────┐   │   │
│      │  │ [Optional] CertificateRequest                          │   │   │
│      │  │  • Request client certificate (mTLS)                   │   │   │
│      │  └─────────────────────────────────────────────────────────┘   │   │
│      │ ←──────────────────────────────────────────────────────────────  │
│      │                                                                │   │
│      │  ┌─────────────────────────────────────────────────────────┐   │   │
│      │  │ ClientKeyExchange                                      │   │   │
│      │  │  • Pre-master secret (encrypted with server's key)    │   │   │
│      │  │  • Certificate (if requested)                          │   │   │
│      │  │  • CertificateVerify (prove client has private key)   │   │   │
│      │  └─────────────────────────────────────────────────────────┘   │   │
│      │ ──────────────────────────────────────────────────────────────→  │
│      │                                                                │   │
│      │  ┌─────────────────────────────────────────────────────────┐   │   │
│      │  │ ChangeCipherSpec + Finished                            │   │   │
│      │  │  • Switch to encrypted communication                  │   │   │
│      │  │  • Hash of all handshake messages                      │   │   │
│      │  └─────────────────────────────────────────────────────────┘   │   │
│      │ ──────────────────────────────────────────────────────────────→  │
│      │                                                                │   │
│      │ ←──────────────────────────────────────────────────────────────  │
│      │  ┌─────────────────────────────────────────────────────────┐   │   │
│      │  │ ChangeCipherSpec + Finished                            │   │   │
│      │  │  • Server confirms encryption                         │   │   │
│      │  └─────────────────────────────────────────────────────────┘   │   │
│      │ ←──────────────────────────────────────────────────────────────  │
│      │                                                                │   │
│   ════════════════════════════════════════════════════════════════════   │
│                                                                          │
│   TOTAL LATENCY: 2-3 RTT (3-30ms depending on network)                  │
│                                                                          │
│   TLS 1.3 IMPROVEMENT: 1-RTT handshake (or 0-RTT for resumption)        │
│                                                                          │
│   PERFORMANCE TIPS:                                                      │
│   • Use TLS session resumption (session tickets)                       │
│   • Enable OCSP stapling (avoid extra CRL lookups)                      │
│   • Use HTTP/2 multiplexing (reuse connections)                        │
│   • Consider TLS 1.3 for faster handshakes                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 TLS Certificate Management

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CERTIFICATE CHAIN                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Browser Trust Store                                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Root CA (e.g., DigiCert, Let's Encrypt, Comodo)               │   │
│   │   • Pre-installed in OS/browser                                │   │
│   │   • Signs intermediate CAs                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│        │                                                               │
│        │ signs                                                        │
│        ▼                                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Intermediate CA                                               │   │
│   │   • Signs server certificates                                  │   │
│   │   • Valid for 1-5 years                                       │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│        │                                                               │
│        │ signs                                                        │
│        ▼                                                               │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Server Certificate (leaf)                                     │   │
│   │   • example.com → public key                                   │   │
│   │   • Valid for 90 days (Let's Encrypt) to 1 year               │   │
│   │   • Contains: domain, public key, expiration, signatures      │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│   COMMON ISSUES:                                                        │
│                                                                          │
│   1. EXPIRED CERTIFICATES                                              │
│      • Let's Encrypt: 90 days → automate renewal!                      │
│      • Production incident: widespread outages from expired certs       │
│                                                                          │
│   2. CHAIN INCOMPLETE                                                  │
│      • Server must serve full chain (leaf + intermediates)             │
│      • Missing intermediate → browser trust error                      │
│                                                                          │
│   3. WRONG DOMAIN                                                      │
│      • Certificate must match requested domain (including subdomains)  │
│      • Wildcards: *.example.com covers a.example.com, NOT a.b.example.com│
│                                                                          │
│   4. SELF-SIGNED CERTIFICATES                                           │
│      • Fine for internal services                                       │
│      • Must be added to trust store                                     │
│      • For Kubernetes: use cert-manager with internal CA                │
│                                                                          │
│   5. WEAK CIPHERS                                                       │
│      • Disable: SSLv3, TLS 1.0, TLS 1.1, 3DES, RC4                    │
│      • Require: TLS 1.2+ with AEAD ciphers (AES-GCM, ChaCha20-Poly1305)│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Load Balancing

### 5.1 Load Balancing Algorithms

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   LOAD BALANCING ALGORITHMS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   1. ROUND ROBIN                                                        │
│   ─────────────────                                                     │
│   Request 1 → Server A                                                  │
│   Request 2 → Server B                                                  │
│   Request 3 → Server C                                                 │
│   Request 4 → Server A (repeat)                                         │
│                                                                          │
│   Pros: Simple, no state needed                                         │
│   Cons: Doesn't consider current load                                   │
│                                                                          │
│   ──────────────────────────────────────────────────────────────────    │
│                                                                          │
│   2. LEAST CONNECTIONS                                                 │
│   ─────────────────────                                                 │
│   Server A: 5 active connections                                         │
│   Server B: 3 active connections  ← Request goes here                 │
│   Server C: 7 active connections                                         │
│                                                                          │
│   Pros: Balances by actual load                                         │
│   Cons: Requires tracking connection counts                              │
│                                                                          │
│   ──────────────────────────────────────────────────────────────────    │
│                                                                          │
│   3. LEAST REQUEST (Least Latency)                                     │
│   ────────────────────────────                                          │
│   Routes to server with lowest recent latency                            │
│   Uses: Kubernetes default (P2C - Power of Two Choices)                │
│                                                                          │
│   Pros: Adapts to server performance                                    │
│   Cons: More complex implementation                                     │
│                                                                          │
│   ──────────────────────────────────────────────────────────────────    │
│                                                                          │
│   4. WEIGHTED                                                           │
│   ──────────                                                            │
│   Server A: weight=3 (gets 3x traffic)                                  │
│   Server B: weight=2 (gets 2X traffic)                                  │
│   Server C: weight=1 (gets 1X traffic)                                 │
│                                                                          │
│   Use: Different server capacities, gradual rollouts                    │
│                                                                          │
│   ──────────────────────────────────────────────────────────────────    │
│                                                                          │
│   5. IP HASH                                                           │
│   ───────────                                                           │
│   hash(client_ip) % num_servers → server                                │
│                                                                          │
│   Use: Session affinity, same-user requests to same server              │
│   Cons: Uneven distribution if hash not uniform                         │
│                                                                          │
│   ──────────────────────────────────────────────────────────────────    │
│                                                                          │
│   6. CONSISTENT HASHING                                                │
│   ─────────────────────                                                │
│   Hash ring: Server A (0-100), B (101-200), C (201-300)                │
│   Add server D: Takes ~33% from each existing server                   │
│   Remove server B: Traffic redistributes to A, C                        │
│                                                                          │
│   Use: Cache routing, minimal reshuffling on changes                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Layer 4 vs Layer 7 Load Balancing

| Aspect | Layer 4 (TCP) | Layer 7 (HTTP) |
|--------|---------------|----------------|
| ** OSI Layer** | Transport | Application |
| **Inspection** | IP/Port only | Full HTTP request |
| **Routing** | By IP/Port | By URL, headers, cookies |
| **Performance** | Faster (less parsing) | Slower (full parse) |
| **TLS Termination** | At backend servers | At load balancer |
| **Protocol** | TCP, UDP | HTTP, HTTP/2, gRPC |
| **Examples** | HAProxy (TCP), AWS NLB | HAProxy (HTTP), AWS ALB, NGINX |

---

## 6. Network Latency Math

### 6.1 Latency Numbers Every Engineer Should Know

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  NETWORK LATENCY NUMBERS (2026)                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Operation                                    Time         Relative    │
│   ────────────────────────────────────────────────────────            │
│                                                                          │
│   L1 cache reference                             0.5 ns      1 sec      │
│   L2 cache reference                             7 ns         14 sec      │
│   Main memory reference (RAM)                   100 ns       3 min      │
│   Compress 1KB with Snappy                     3 μs         10 hours   │
│   Send 1KB over 1 Gbps LAN                     10 μs        12 days     │
│   Read 1MB from memory                         250 μs       8 months    │
│   Round trip within same datacenter            500 μs       1.5 years   │
│   Read 1MB from SSD                            1 ms         3 years     │
│   Disk seek                                     10 ms        30 years    │
│   Read 1MB from HDD                            20 ms        60 years     │
│   Round trip CA → Netherlands                  150 ms       450 years   │
│   Round trip CA → Australia                    200 ms       600 years   │
│                                                                          │
│   ─────────────────────────────────────────────────────────────────────   │
│                                                                          │
│   KEY INSIGHT:                                                          │
│   • Network within datacenter: 0.5-1ms                                 │
│   • Cross-region in same continent: 20-50ms                            │
│   • Cross-continental: 100-200ms                                        │
│   • Satellite: 600+ ms                                                  │
│                                                                          │
│   FOR SYSTEM DESIGN:                                                    │
│   • 1ms = 1,000,000 ns                                                  │
│   • Design for local calls first                                        │
│   • Cache aggressively to avoid network                                 │
│   • Async communication to hide latency                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Capacity Planning Examples

**Example 1: Internal Service Communication**

```
Requirements:
- 10,000 requests/second
- Average request: 10KB
- 3 services in call chain

Calculation:
- Per request bandwidth: 10KB × 3 = 30KB
- Total bandwidth: 10,000 × 30KB = 300MB/s = 2.4Gbps
- Choose: 10Gbps network interface

Latency budget:
- 10ms p99 target
- Per-hop budget: 3ms
- Include: serialization, network, deserialization
```

**Example 2: Database Connection Pool**

```
Requirements:
- 5,000 requests/second  
- Average query time: 5ms
- Connection overhead: 1ms

Calculation:
- Max queries per connection: 1000/5 = 200 queries/sec
- Queries per second: 5,000
- Minimum connections needed: 5,000 / 200 = 25
- Add buffer: 25 × 2 = 50 connections

Formula:
connections = (queries/sec) / (1/avg_query_time) × safety_factor
```

---

## 7. Interview Questions

### Q1: "Why is TCP faster than UDP for some applications?"

**Expected Answer**:
- TCP provides reliability and ordering, which UDP lacks
- For applications needing reliability (HTTP, databases), TCP is faster because it handles retransmission and ordering in the OS kernel
- UDP requires implementing reliability in application code
- TCP's congestion control optimizes network usage
- However, for very latency-sensitive apps (gaming, VoIP), UDP is better because head-of-line blocking in TCP causes variable latency

### Q2: "How does HTTP/2 improve on HTTP/1.1?"

**Expected Answer**:
1. **Multiplexing**: Multiple requests over single TCP connection
2. **Header compression**: HPACK reduces header overhead
3. **Server push**: Server can proactively send resources
4. **Binary framing**: More efficient than text-based HTTP/1.1
5. **Stream prioritization**: Client can prioritize resources

### Q3: "What happens when you type google.com?"

**Expected Answer**:
1. Browser checks cache (DNS, assets)
2. OS resolves `google.com` via DNS (recursive resolver → root → TLD → authoritative)
3. TCP SYN → SYN-ACK → ACK (3-way handshake)
4. TLS handshake (ClientHello → ServerHello → certificates → keys)
5. HTTP request: GET / HTTP/1.1
6. Server processes, returns HTTP response
7. Browser renders page, loads subresources (parallelized via HTTP/2 or domain sharding)

### Q4: "How would you reduce latency for a global API?"

**Expected Answer**:
1. **Edge deployment**: Deploy to multiple regions, route users to nearest
2. **CDN**: Cache static assets at edge
3. **Connection pooling**: Reuse TCP/TLS connections
4. **HTTP/2 or HTTP/3**: Multiplexing, header compression
5. **gRPC**: Binary serialization, streaming
6. **Async**: Use message queues for non-critical paths
7. **Optimize payload**: Compression, pagination, field selection
8. **Database co-location**: Database in same region as API

---

## 8. Summary

| Topic | Key Takeaway |
|-------|--------------|
| **TCP** | 1.5 RTT handshake cost is significant; use connection pooling |
| **UDP** | Lower latency but requires app-level reliability |
| **DNS** | Cache aggressively, use multiple providers, consider Anycast |
| **TLS** | 2-3 RTT for full handshake; use session resumption |
| **Load Balancing** | Layer 4 for performance, Layer 7 for routing flexibility |
| **Latency** | 1ms local, 20-50ms cross-region, 100-200ms cross-continental |

**Production Checklist**:
- [ ] Connection pooling enabled
- [ ] HTTP/2 or HTTP/3 for external APIs
- [ ] gRPC for internal services
- [ ] TLS 1.2+ with strong ciphers
- [ ] DNS monitoring and failover
- [ ] Load balancer health checks configured
- [ ] Client-side timeouts set appropriately
