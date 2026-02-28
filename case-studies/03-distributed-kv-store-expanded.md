# System Design Case Study: Distributed Key-Value Store (Like DynamoDB)

> A comprehensive walkthrough of designing a Dynamo-style distributed key-value store with consistent hashing, quorum reads/writes, and failure handling.

---

## 1. Requirements Analysis

### 1.1 Functional Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FUNCTIONAL REQUIREMENTS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  F1: Basic Operations                                                  │
│      • Get(key) → value                                                │
│      • Put(key, value) → success/failure                               │
│      • Delete(key) → success/failure                                   │
│                                                                          │
│  F2: Consistency Options                                               │
│      • Strong consistency (default)                                    │
│      • Eventual consistency (optional)                                 │
│      • Read-your-writes consistency                                    │
│                                                                          │
│  F3: Data Types                                                        │
│      • String, List, Set, Hash, Sorted Set                              │
│      • TTL (expiration) support                                         │
│                                                                          │
│  F4: Conditional Operations                                            │
│      • Compare-and-set (only if unchanged)                             │
│      • Atomic increment/decrement                                      │
│                                                                          │
│  F5: Query Operations                                                  │
│      • Range queries (within a partition)                              │
│      • Scan operations (full table scan)                               │
│                                                                          │
│  F6: Administrative                                                    │
│      • Create/describe/drop tables                                     │
│      • Backup and restore                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NON-FUNCTIONAL REQUIREMENTS                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SCALABILITY:                                                          │
│  • Handle trillions of items                                           │
│  • Petabytes of data                                                   │
│  • Millions of requests per second                                     │
│                                                                          │
│  LATENCY:                                                              │
│  • Single-digit millisecond p99 for Get/Put                          │
│  • Consistent low latency under load                                   │
│                                                                          │
│  AVAILABILITY:                                                         │
│  • 99.99% availability (multi-AZ)                                      │
│  • Survive entire region failure                                       │
│  • No single point of failure                                          │
│                                                                          │
│  CONSISTENCY:                                                          │
│  • Tunable: strong to eventual                                         │
│  • Eventual consistency: < 1 second                                   │
│                                                                          │
│  DURABILITY:                                                           │
│  • Write acknowledged only after N replicas store                     │
│  • Survive disk failures                                               │
│                                                                          │
│  OPERATIONS:                                                           │
│  • Auto-scaling (add/remove nodes)                                     │
│  • Online schema changes                                               │
│  • Transparent rebalancing                                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Capacity Estimation

### 2.1 System Capacity

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CAPACITY ESTIMATION                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STORAGE:                                                              │
│  ────────                                                             │
│  • 1 trillion items × 1KB average = 1PB data                          │
│  • With replication (3x): 3PB physical storage                        │
│  • 100 nodes × 10TB each = 1PB                                         │
│                                                                          │
│  QPS:                                                                 │
│  ────                                                                 │
│  • Read: 10M QPS                                                       │
│  • Write: 1M QPS                                                       │
│  • Per node: 100K QPS                                                  │
│                                                                          │
│  NETWORK:                                                             │
│  ────────                                                             │
│  • Write path: 1M × 1KB = 1GB/s inbound                              │
│  • Cross-replica: 3x write = 3GB/s                                    │
│                                                                          │
│  MEMORY (for caching):                                                │
│  ─────────────────                                                    │
│  • Hot data: 1% of items in memory                                    │
│  • 10M items × 1KB = 10GB per node                                   │
│                                                                          │
│  PARTITIONS:                                                           │
│  ───────────                                                           │
│  • 1000 partitions across 100 nodes                                   │
│  • 10 partitions per node                                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Core Architecture

### 3.1 Ring Architecture (Consistent Hashing)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENT HASHING RING                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  THE PROBLEM:                                                          │
│  When adding/removing nodes, minimal data should move                   │
│                                                                          │
│  TRADITIONAL HASHING:                                                  │
│  hash(key) % N = node                                                 │
│  When N changes, ALL keys remap!                                       │
│                                                                          │
│  CONSISTENT HASHING:                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │          Node A          Node B          Node C                │   │
│  │             │               │               │                  │   │
│  │             ▼               ▼               ▼                  │   │
│  │    ┌────────────────────────────────────────────────────┐    │   │
│  │    │                                                    │    │   │
│  │    │          HASH RING (0 to 2^160)                   │    │   │
│  │    │                                                    │    │   │
│  │    │   0 ──────────► A ◄──────────► B ◄──────────► C   │    │   │
│  │    │        │         │              │              │    │   │
│  │    │        │         │              │              │    │   │
│  │    │        ▼         ▼              ▼              ▼    │   │
│  │    │    Range A    Range B        Range C       Range D  │   │
│  │    │                                                    │    │   │
│  │    └────────────────────────────────────────────────────┘    │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  LOOKUP:                                                              │
│  hash("user:123") = 0x4F3A...                                         │
│  Find first node clockwise from this position → Node B                 │
│                                                                          │
│  ADD NODE:                                                             │
│  • New node D inserted between B and C                                  │
│  • Only keys in range (B,D) move to D                                  │
│  • ~1/N keys affected                                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  VIRTUAL NODES:                                                       │
│  • Each physical node has 100 virtual nodes                             │
│  • Better load distribution                                            │
│  • Node A: vnode-1, vnode-2, ... vnode-100                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Replication Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REPLICATION WITH QUORUM                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CONFIGURATION:                                                        │
│  • N = 3 replicas                                                     │
│  • W = 2 (write quorum)                                                │
│  • R = 2 (read quorum)                                                 │
│                                                                          │
│  QUORUM GUARANTEE:                                                     │
│  W + R > N = 2 + 2 > 3 = 5 > 3 ✓                                      │
│  At least one node has latest write!                                   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WRITE PATH:                                                           │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Client writes key="user:123", value={name: "alice"}             │   │
│  │                                                                  │   │
│  │ 1. Route to coordinator (based on hash ring)                    │   │
│  │                                                                  │   │
│  │ 2. Coordinator sends to N replicas in parallel                  │   │
│  │    • Primary: node1                                            │   │
│  │    • Replica 1: node2                                          │   │
│  │    • Replica 2: node3                                          │   │
│  │                                                                  │   │
│  │ 3. Wait for W=2 acknowledgments                                │   │
│  │    • node1: ✓ ACK                                             │   │
│  │    • node2: ✓ ACK                                             │   │
│  │    • node3: (waiting...)                                       │   │
│  │                                                                  │   │
│  │ 4. Return success to client                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  READ PATH:                                                            │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Client reads key="user:123"                                     │   │
│  │                                                                  │   │
│  │ 1. Route to coordinator                                        │   │
│  │                                                                  │   │
│  │ 2. Coordinator sends to N replicas in parallel                 │   │
│  │                                                                  │   │
│  │ 3. Wait for R=2 responses                                      │   │
│  │    • node1: version 5, value={name: "alice"}                  │   │
│  │    • node2: version 5, value={name: "alice"}                  │   │
│  │                                                                  │   │
│  │ 4. If versions differ:                                         │   │
│  │    • Use latest version (version 5)                            │   │
│  │    • Trigger async repair (write latest to laggard)           │   │
│  │                                                                  │   │
│  │ 5. Return value to client                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  CONSISTENCY TUNING:                                                   │
│                                                                          │
│  │ Consistency │ W │ R │ Latency │ Use Case                        │  │
│  │-------------|---|---|---------|--------------------------------|  │
│  │ Strong      │ 3 │ 3 │ Highest │ Financial, inventory         │  │
│  │ Default     │ 2 │ 2 │ Medium  │ Most applications             │  │
│  │ Eventual    │ 1 │ 1 │ Lowest  │ Analytics, caching            │  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 High-Level Architecture

```mermaid
graph TB
    subgraph "Client"
        Client[SDK Client]
    end

    subgraph "Routing Layer"
        Router[Request Router]
        PartitionMap[Partition Map Cache]
    end

    subgraph "Storage Nodes"
        Node1[Node 1<br/>Partitions: A, B]
        Node2[Node 2<br/>Partitions: C, D]
        Node3[Node 3<br/>Partitions: E, F]
    end

    subgraph "Coordination"
        Gossip[Gossip Protocol]
        FailureDetector[Failure Detector]
    end

    Client --> Router
    Router --> PartitionMap
    
    Router --> Node1
    Router --> Node2
    Router --> Node3
    
    Node1 <--> Gossip
    Node2 <--> Gossip
    Node3 <--> Gossip
    
    Gossip --> FailureDetector
```

---

## 4. Data Model

### 4.1 Partitioning

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PARTITIONING STRATEGY                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PARTITION KEY:                                                        │
│  • Hash of partition key → partition number                            │
│  • Example: hash("user:123") % 1000 = 432                            │
│                                                                          │
│  COMPOUND KEYS:                                                        │
│  • Partition key + Sort key                                           │
│  • Example: user_id#timestamp                                          │
│  • Enables efficient range queries within partition                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  EXAMPLE TABLE: Orders                                                 │
│                                                                          │
│  Partition Key: customer_id (for ordering by customer)                │
│  Sort Key: order_timestamp#order_id                                   │
│                                                                          │
│  │ customer_id │ order_timestamp#order_id │ content         │         │
│  │-------------|------------------------|-----------------|         │
│  │ user:123   │ 2024-01-01#ord_001     │ Order 1         │         │
│  │ user:123   │ 2024-01-02#ord_002     │ Order 2         │         │
│  │ user:123   │ 2024-01-05#ord_005     │ Order 5         │         │
│  │ user:456   │ 2024-01-01#ord_100     │ Order 100       │         │
│                                                                          │
│  QUERIES:                                                              │
│  • Get orders for user:123 → Scan partition for user:123             │
│  • Range query: user:123 orders on Jan 2 → range scan                │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HOT KEYS:                                                            │
│  • Problem: One key gets millions of requests                         │
│  • Solution: Split hot keys into multiple partitions                  │
│  • Example: hot_user_123 → hot_user_123#shard_0, shard_1, etc.       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Failure Handling

### 5.1 Failure Detection and Recovery

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FAILURE HANDLING                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  FAILURE DETECTION:                                                    │
│  ───────────────────                                                   │
│                                                                          │
│  Gossip Protocol:                                                      │
│  • Each node periodically shares state with random nodes              │
│  • Eventually all nodes agree on cluster state                         │
│  • Uses Phi Accrual Failure Detector                                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Node A                                        Node B           │   │
│  │    │                                            │              │   │
│  │    │  Gossip: "I think B is down"            │              │   │
│  │    │ ───────────────────────────────────────→  │              │   │
│  │    │                                            │              │   │
│  │    │  Gossip: "I talked to B 10ms ago"        │              │   │
│  │    │ ←──────────────────────────────────────── │              │   │
│  │    │                                            │              │   │
│  │    │  Consensus: B is ALIVE                     │              │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HINTED HANDOFF:                                                       │
│  ─────────────────                                                      │
│                                                                          │
│  When a replica is temporarily down:                                   │
│  • Another node accepts the write "on behalf"                         │
│  • Stores with "hint" about where it should go                        │
│  • When failed node recovers, replays hints                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Normal: Write to A, B, C                                       │   │
│  │                                                                  │   │
│  │  B is down:                                                     │   │
│  │    • Write to A                                                 │   │
│  │    • Write to C (stores hint: "send to B when up")            │   │
│  │    • A returns success                                         │   │
│  │                                                                  │   │
│  │  B recovers:                                                    │   │
│  │    • C sends hinted data to B                                   │   │
│  │    • B now has all data                                        │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ANTI-ENTROPY (Merkle Trees):                                         │
│  ─────────────────────────────────                                    │
│                                                                          │
│  Problem: Replicas can diverge                                         │
│  Solution: Periodic checksum comparison                               │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Merkle Tree:                                                   │   │
│  │                                                                  │   │
│  │              [Root Hash]                                        │   │
│  │             /        \                                         │   │
│  │         [Hash AB]    [Hash CD]                                 │   │
│  │         /    \        /    \                                   │   │
│  │       [A]    [B]    [C]    [D]  (key-value buckets)           │   │
│  │                                                                  │   │
│  │  Compare root hashes → if different, traverse to find         │   │
│  │  differences → sync only divergent data                         │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Conflict Resolution

### 6.1 Version Vectors

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFLICT RESOLUTION                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  THE PROBLEM:                                                          │
│  Network partition + concurrent writes = conflict                      │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Time │ Node A (US-East)      │ Node B (US-West)              │   │
│  │ ──────┼───────────────────────┼─────────────────────────────── │   │
│  │  T1   │ Read user:123        │                              │   │
│  │       │ name="Alice"          │                              │   │
│  │       │                       │                              │   │
│  │  T2   │ [NETWORK PARTITION]  │                              │   │
│  │       │                       │                              │   │
│  │  T3   │ Write: name="Bob"    │ Write: name="Charlie"        │   │
│  │       │                       │                              │   │
│  │  T4   │ [PARTITION HEALS]    │                              │   │
│  │       │                       │                              │   │
│  │  T5   │ Which value wins?    │                              │   │
│  │       │                       │                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  RESOLUTION STRATEGIES:                                                │
│  ─────────────────────────                                              │
│                                                                          │
│  1. LAST-WRITE-WINS (LWW):                                             │
│     • Use timestamp or vector clock                                    │
│     • Simple but can lose updates                                      │
│     • Used by DynamoDB default                                         │
│                                                                          │
│  2. VERSION VECTORS:                                                   │
│     • Each object has vector clock {node: version}                    │
│     • Track causal relationship between versions                       │
│                                                                          │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │                                                             │    │
│     │  Initial: {node1: 1}  → Alice                             │    │
│     │                                                             │    │
│     │  After write at node1: {node1: 2}  → Bob                  │    │
│     │                                                             │    │
│     │  After write at node2: {node2: 1}  → Charlie             │    │
│     │                                                             │    │
│     │  Conflict detected: {node1: 2, node2: 1}                 │    │
│     │  Two versions, neither dominates → CONFLICT               │    │
│     │                                                             │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  3. CRDTs (Conflict-free Replicated Data Types):                      │
│     • Mathematically guaranteed convergence                           │
│     • Examples: G-Counter, LWW-Register, OR-Set                      │
│     • Good for: counters, sets                                        │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  CRDT Example: Grow-only Counter                            │    │
│  │                                                             │    │
│  │  Node A: counter = 5                                      │    │
│  │  Node B: counter = 3                                      │    │
│  │                                                             │    │
│  │  Merge: counter = max(5, 3) = 5  ✓ No conflict!          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  4. APPLICATION-RESOLVED:                                             │
│     • Store all conflicting versions                                  │
│     • Return to application for resolution                             │
│     • Example: Git merge conflict                                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  DYNAMODB APPROACH:                                                   │
│  • Default: LWW with timestamp                                        │
│  • Optional: conditional writes                                       │
│  • Optional: compare-and-swap                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Monitoring and SLOs

### 7.1 Key Metrics

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MONITORING & SLOS                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CRITICAL METRICS:                                                     │
│  ────────────────                                                      │
│                                                                          │
│  | Metric              | SLO Target     | Alert Threshold              |  |
│  |---------------------|----------------|------------------------------|  |
│  | Read latency p99    | < 10ms        | > 25ms for 5 min           |  |
│  | Write latency p99   | < 15ms        | > 30ms for 5 min           |  |
|  | Availability        | 99.99%        | < 99.9% for 2 min          |  |
│  | Replication lag     | < 100ms       | > 500ms for 1 min          |  |
│  | Throttled requests  | < 0.1%        | > 1% for 5 min             |  |
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  PARTITION HEALTH:                                                     │
│  ─────────────────                                                     │
│  • Partitions per node: should be balanced (within 10%)               │
│  • Hot partitions: alert if any partition > 2x average                │
│  • Partition count: monitor for approaching limits                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  NODE HEALTH:                                                          │
│  ────────────                                                         │
│  • CPU utilization: alert > 80% sustained                              │
│  • Memory: alert > 85%                                                 │
│  • Disk: alert > 90%                                                   │
│  • Network: alert if any node unreachable                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  CONSISTENCY MONITORING:                                               │
│  ─────────────────────────                                             │
│  • Read repair operations: should be rare                             │
│  • Merkle tree sync: should complete within 1 hour                    │
│  • Version conflicts: should be < 0.01%                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Interview Follow-Up Questions

### Q1: "How does consistent hashing help with scaling?"

**Expected Answer:**
1. When adding/removing nodes, only ~1/N keys move
2. Virtual nodes improve distribution
3. Each node knows its responsible key ranges
4. No central coordinator needed (decentralized)
5. Enables incremental scaling

### Q2: "What happens when a node fails?"

**Expected Answer:**
1. Gossip protocol detects failure (within seconds)
2. Partition map updated
3. Requests routed to remaining replicas
4. Hinted handoff stores writes for failed node
5. When node recovers, hinted data is replayed
6. Anti-entropy (Merkle trees) sync any divergence

### Q3: "How do you handle read-after-write consistency?"

**Expected Answer:**
1. Route reads to same partition as write
2. Use version vectors to ensure latest read
3. Leader for partition handles reads after writes
4. DynamoDB: can use strongly consistent reads
5. Trade-off: higher latency for strong consistency

### Q4: "Why use quorum reads/writes?"

**Expected Answer:**
1. W + R > N ensures overlap between read and write sets
2. Guarantees at least one node has latest version
3. Tunable: higher W = stronger consistency, higher latency
4. Lower W/R = higher availability, lower latency
5. Default DynamoDB: N=3, W=2, R=2

---

## 9. Summary

| Component | Implementation | Purpose |
|-----------|---------------|---------|
| **Hash Ring** | Consistent hashing | Partition mapping |
| **Replication** | N=3, W=2, R=2 | Durability + availability |
| **Failure Detection** | Gossip protocol | Node health |
| **Hinted Handoff** | Temporary replica | Write durability |
| **Anti-Entropy** | Merkle trees | Consistency repair |
| **Conflict Resolution** | LWW or CRDTs | Eventual convergence |

**Key Insights:**
1. Consistent hashing enables incremental scaling
2. Quorum provides tunable consistency/availability
3. Failure handling is async and transparent
4. Conflicts are inevitable - choose resolution strategy based on use case
5. No single point of failure - fully decentralized
