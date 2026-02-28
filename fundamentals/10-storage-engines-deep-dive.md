# Storage Engines Deep Dive — B-Trees vs LSM-Trees

> The choice between B-trees and LSM-trees fundamentally shapes your database's performance characteristics. This chapter provides an exhaustive analysis with production math, decision frameworks, and real-world implementations.

---

## 1. B-Trees: The Universal Default

### 1.1 Physical Structure

A B-tree organizes data in **fixed-size pages** (typically 4KB-16KB) arranged in a balanced tree. Every non-leaf page contains keys and pointers to child pages. Leaf pages contain the actual data values.

```
                    ┌─────────────────────────────────────┐
                    │         ROOT PAGE (4KB)              │
                    │  [ │ 10 │ │ 20 │ │ 30 │ │ 40 │ ]     │
                    │   │    │   │    │   │    │          │
                    └───┼────┼────┼────┼────┼────┼──────────┘
                        │    │    │    │    │    │
        ┌───────────────┼────┼────┼────┼────┼───────────┐
        │               │    │    │    │    │           │
        ▼               ▼    ▼    ▼    ▼    ▼           ▼
┌───────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  LEVEL 1     │ │  LEVEL 1     │ │  LEVEL 1     │ │  LEVEL 1     │
│ [3|5|7|9]    │ │ [12|15|18]   │ │ [22|25|28]   │ │ [32|35|38]   │
│ (Internal)   │ │ (Internal)   │ │ (Internal)   │ │ (Internal)   │
└───────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
        │               │             │             │
        ▼               ▼             ▼             ▼
┌───────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  LEAF PAGE   │ │  LEAF PAGE   │ │  LEAF PAGE   │ │  LEAF PAGE   │
│ [k1=v1]      │ │ [k10=v10]    │ │ [k20=v20]    │ │ [k30=v30]    │
│ [k2=v2]      │ │ [k11=v11]    │ │ [k21=v21]    │ │ [k31=v31]    │
│ [k3=v3]      │ │ [k12=v12]    │ │ [k22=v22]    │ │ [k32=v32]    │
│ [k4=v4]      │ │ [k13=v13]    │ │ [k23=v23]    │ │ [k33=v33]    │
│ ...          │ │ ...          │ │ ...          │ │ ...          │
└───────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### 1.2 Read Path

```python
# B-Tree Read Algorithm
def btree_read(tree, key):
    page = tree.root
    
    while not page.is_leaf:
        # Binary search within page to find the correct child
        child_index = page.binary_search(key)
        page = page.get_child(child_index)
    
    # Now at leaf level - binary search for exact key
    index = page.binary_search(key)
    if index >= 0:
        return page.values[index]
    return None  # Key not found
```

**Time Complexity**: O(log n) where n is the number of keys. For 1 billion keys, this is only ~30 page accesses.

### 1.3 Write Path (Update-in-Place)

```python
# B-Tree Write Algorithm
def btree_write(tree, key, value):
    # Find the leaf page containing the key
    page = tree.find_leaf(key)
    
    if key in page:
        # Update existing key - in-place
        page.update(key, value)
    else:
        # Insert new key
        if page.has_space():
            page.insert_sorted(key, value)
        else:
            # Page is full - split it
            left, right = page.split()
            tree.insert_page_into_parent(left, right)
    
    # Write-Ahead Log (WAL) - ALWAYS happens first
    tree.wal.append(LogEntry(key, value))
    tree.wal.flush_to_disk()
```

### 1.4 Page Splits and Merges

When a leaf page overflows:

```
BEFORE SPLIT:
┌─────────────────────────────────────────┐
│ Page: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]  │
│ (full - 10 keys)                       │
└─────────────────────────────────────────┘

AFTER SPLIT:
┌─────────────────────┐   ┌─────────────────────┐
│ Left Page           │   │ Right Page          │
│ [1, 2, 3, 4, 5]    │   │ [6, 7, 8, 9, 10]    │
│ (5 keys)            │   │ (5 keys)           │
└─────────────────────┘   └─────────────────────┘
         │                         │
         └──────────┬──────────────┘
                    ▼
         ┌─────────────────────┐
         │ New Parent          │
         │ [5, 10]            │
         │ (promoted keys)    │
         └─────────────────────┘
```

### 1.5 Write-Ahead Log (WAL)

Every B-tree implementation uses a WAL for crash recovery:

```
┌─────────────────────────────────────────────────────────────┐
│                     WRITE PATH                              │
│                                                             │
│  1. User requests: UPDATE SET balance=100 WHERE id=42     │
│                         ↓                                   │
│  2. Write to WAL: "id=42, balance=100"                    │
│     ┌─────────────────────────────────────────────┐         │
│     │ WAL (append-only, sequential I/O)          │         │
│     │ [txn1] [txn2] [txn3] [txn4] [txn5] ...    │         │
│     └─────────────────────────────────────────────┘         │
│                         ↓ (durable after flush)            │
│  3. Update B-tree pages in memory                         │
│                         ↓                                   │
│  4. Return "success" to user                              │
└─────────────────────────────────────────────────────────────┘
```

**Recovery**: After a crash, replay the WAL to restore the B-tree to a consistent state.

### 1.6 Production Characteristics

| Metric | B-Tree Value | Notes |
|--------|-------------|-------|
| Read Latency | 0.5-2ms | Single key lookup |
| Write Latency | 1-5ms | Includes WAL fsync |
| Space Amplification | 10-30% | Page fragmentation |
| Write Amplification | 1x | Single write to WAL + B-tree |
| Optimal Workload | Read-heavy (>70% reads) | Point queries, range scans |

### 1.7 When to Choose B-Trees

- **Primary database storage** for most applications
- **OLTP workloads** with point lookups and range queries
- **When query latency predictability matters** (consistent O(log n))
- **When storage is relatively cheap** (some space overhead)
- **PostgreSQL, MySQL, Oracle, SQL Server** use B-trees by default

---

## 2. LSM-Trees: Write-Optimized Alternative

### 2.1 Physical Structure

LSM-trees (Log-Structured Merge-trees) use a **multi-level sorted structure** with append-only writes:

```
┌─────────────────────────────────────────────────────────────────┐
│                     LSM-TREE ARCHITECTURE                       │
│                                                                  │
│  MEMTABLE (RAM)                                                 │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │ Sorted Map: { "user:1" → {...}, "user:2" → {...}, ... } │   │
│  │ Size: ~128MB - 1GB before flush                         │   │
│  └───────────────────────────────────────────────────────────┘   │
│                            ↓ flush when full                     │
│  ════════════════════════════════════════════════════════════    │
│                                                                  │
│  SSTABLE FILES (Disk)                                           │
│                                                                  │
│  Level 0 (L0): Most recent, may overlap                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ L0_001   │ │ L0_002   │ │ L0_003   │                        │
│  │ (sorted) │ │ (sorted) │ │ (sorted) │                        │
│  └──────────┘ └──────────┘ └──────────┘                        │
│         ↓           ↓           ↓                               │
│  Level 1 (L1): Older, non-overlapping                          │
│  ┌──────────┐ ┌──────────┐                                      │
│  │ L1_001   │ │ L1_002   │                                      │
│  └──────────┘ └──────────┘                                      │
│         ↓                                                       │
│  Level 2 (L2): Even older...                                   │
│  ┌──────────┐                                                   │
│  │ L2_001   │                                                   │
│  └──────────┘                                                   │
│                                                                  │
│  COMPACTION (background): L0 → L1 → L2 → ...                   │
│  Merges sorted runs, removes deleted/tombstoned entries        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Write Path

```python
# LSM-Tree Write Algorithm
def lsm_write(tree, key, value):
    # Step 1: Append to WAL (for recovery)
    tree.wal.append(key, value)
    tree.wal.flush_to_disk()
    
    # Step 2: Insert into MemTable (in-memory)
    tree.memtable.put(key, value)
    
    # Step 3: Check if memtable is full
    if tree.memtable.size > tree.memtable.max_size:
        # Flush to disk as SSTable
        tree.flush_memtable_to_l0()
    
    return True  # Very fast - just memory operations
```

**Key insight**: LSM writes are O(1) - just appending to memory. No random disk I/O, no tree traversal.

### 2.3 Read Path

```python
# LSM-Tree Read Algorithm - Check multiple levels
def lsm_read(tree, key):
    # Step 1: Check memtable (most recent)
    if key in tree.memtable:
        return tree.memtable.get(key)
    
    # Step 2: Check L0 SSTables (most recent on disk)
    for sstable in tree.l0_files:
        if key in sstable:
            return sstable.get(key)
    
    # Step 3: Check L1, L2, ... (older files)
    for level in [tree.l1, tree.l2, ...]:
        # Use bloom filter to skip files that definitely don't contain key
        if not sstable.bloom_filter.might_contain(key):
            continue
        if key in sstable:
            return sstable.get(key)
    
    return None  # Key not found
```

### 2.4 Bloom Filters

Bloom filters provide **probabilistic membership checking** with O(1) space:

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLOOM FILTER OPERATION                        │
│                                                                  │
│  SET OPERATION: add("user:123")                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Hash Functions:                                        │    │
│  │  h1("user:123") = 3  → set bit 3 to 1                  │    │
│  │  h2("user:123") = 7  → set bit 7 to 1                  │    │
│  │  h3("user:123") = 12 → set bit 12 to 1                 │    │
│  │                                                          │    │
│  │  Bit Array: [0,0,0,1,0,0,0,1,0,0,0,0,1,0,0,0,...]     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  TEST OPERATION: might_contain("user:456")                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  h1("user:456") = 3  → bit[3] = 1? YES                 │    │
│  │  h2("user:456") = 7  → bit[7] = 1? YES                 │    │
│  │  h3("user:456") = 15 → bit[15] = 0? NO  → DEFINITELY   │    │
│  │                                       NOT CONTAINS       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  FALSE POSITIVES POSSIBLE: "might contain" but doesn't         │
│  FALSE NEGATIVES IMPOSSIBLE: "definitely not" is always true    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 Compaction Strategies

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Tiered** | Each level Nx size of N-1 | Good write throughput | More space, worse reads |
| **Leveled** | Each level has fixed size | Bounded space, good reads | More write amplification |
| **Tiered+Leveled** | L0 is tiered, deeper levels leveled | Balanced | Complex tuning |

**Leveled Compaction (most common)**:
```
Level 0: 10 files × 128MB = 1.28GB total
    ↓ compaction
Level 1: 10 files × 1GB = 10GB total  
    ↓ compaction
Level 2: 10 files × 10GB = 100GB total
    ↓ compaction
Level 3: 10 files × 100GB = 1TB total
```

### 2.6 Production Characteristics

| Metric | LSM-Tree Value | Notes |
|--------|---------------|-------|
| Read Latency | 5-20ms | Variable, depends on levels checked |
| Write Latency | 0.1-0.5ms | Append-only, blazing fast |
| Space Amplification | 1.5-2x | Due to compaction overhead |
| Write Amplification | 10-100x | Each write causes multiple compactions |
| Optimal Workload | Write-heavy (>70% writes) | Append-only, no random I/O |

### 2.7 When to Choose LSM-Trees

- **Write-heavy workloads** (logging, IoT, time-series, analytics)
- **When write throughput matters more than read latency**
- **When storage efficiency matters** (better compression)
- **Cassandra, RocksDB, LevelDB, ScyllaDB** use LSM-trees

---

## 3. Quantitative Comparison

### 3.1 Mathematical Model

Let's analyze a concrete scenario:

**Assumptions:**
- 10,000 writes/second
- 1KB per write
- 10GB total data
- 3-year retention

**B-Tree Performance:**
```
Writes per second: 10,000
Daily write volume: 10,000 × 86,400 × 1KB = 864GB/day
Annual write volume: 864GB × 365 = 315TB/year

Disk I/O per write: ~2-4 (WAL + B-tree update)
Total disk writes/day: 864GB × 3 = 2.5PB/year!
```

**LSM-Tree Performance:**
```
Writes per second: 10,000
Daily write volume: 864GB/day
Annual write volume: 315TB/year

Due to compaction, actual disk writes: ~3-10x = 945TB - 3PB/year
But: sequential writes are MUCH faster than random
```

### 3.2 Capacity Math

```
B-Tree Capacity (single node):
- Page size: 4KB
- Fanout: 200 (typical)
- Tree depth for 1M rows: log_200(1M) ≈ 3
- Tree depth for 1B rows: log_200(1B) ≈ 5
- Storage: 1B rows × 1KB/row = 1TB

LSM-Tree Capacity:
- L0: 128MB files
- L1-L6: 10 files each, 10× size per level
- Total: 128MB × (1 + 10 + 100 + 1000 + 10000 + 100000) ≈ 1.3TB
```

---

## 4. Decision Framework

### 4.1 Choose B-Tree When:

```
┌─────────────────────────────────────────────────────────────────┐
│                    B-TREE DECISION TREE                         │
│                                                                  │
│  What is your primary workload?                                 │
│         │                                                        │
│         ├── Read-heavy (≥70% reads)                              │
│         │     │                                                  │
│         │     ├── Need predictable latency                      │
│         │     │     └── YES → B-Tree (PostgreSQL, MySQL)        │
│         │     │                                                  │
│         │     └── Point queries + range scans                   │
│         │         └── YES → B-Tree                              │
│         │                                                        │
│         ├── Mixed read/write                                    │
│         │     │                                                  │
│         │     ├── ACID transactions required                    │
│         │     │     └── YES → B-Tree (PostgreSQL, MySQL)        │
│         │     │                                                  │
│         │     └── Need complex queries                           │
│         │         └── YES → B-Tree                              │
│         │                                                        │
│         └── Write-heavy (≥70% writes)                           │
│               │                                                  │
│               ├── Time-series / IoT                             │
│               │     └── YES → LSM (Cassandra, Timescale)        │
│               │                                                  │
│               ├── Append-only logging                           │
│               │     └── YES → LSM                               │
│               │                                                  │
│               └── General workload                              │
│                     └── Consider hybrid or LSM                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Decision Matrix

| Factor | B-Tree | LSM-Tree |
|--------|--------|----------|
| **Point read latency** | ✅ Better (0.5-2ms) | Worse (5-20ms) |
| **Range scan throughput** | ✅ Better | Worse |
| **Write throughput** | Worse (random I/O) | ✅ Better (append-only) |
| **Write latency** | Worse (1-5ms) | ✅ Better (0.1-0.5ms) |
| **Space efficiency** | Worse (fragmentation) | ✅ Better (compression) |
| **Write amplification** | ✅ Better (1x) | Worse (10-100x) |
| **Crash recovery** | ✅ Faster (WAL replay) | Slower (replay multiple levels) |
| **Complexity** | ✅ Simpler | Complex (compaction tuning) |
| **Tiered storage** | Poor | ✅ Excellent |

### 4.3 Hybrid Approaches

Modern databases combine both:

| Database | Hybrid Strategy |
|----------|----------------|
| **RocksDB** | Primary LSM, bloom filters for fast reads |
| **Cassandra** | Primary LSM, compaction strategies |
| **PostgreSQL** | B-tree primary, BRIN indexes (LSM-like for time-series) |
| **MongoDB** | B-tree primary, WiredTiger storage engine |
| **TimescaleDB** | PostgreSQL + automatic partitioning + chunk recycling |

---

## 5. Interview Deep-Dive Questions

### Q1: "Why can't we just use B-trees for everything?"

**Expected Answer Structure:**
1. B-trees require random I/O for writes → poor write throughput
2. LSM-trees are append-only → excellent write throughput
3. B-tree write amplification is 1x (ignoring fragmentation), LSM can be 10-100x
4. For write-heavy workloads (IoT, logging, analytics), LSM wins
5. The tradeoff is read performance and complexity

**Follow-up**: "What's the read latency difference?"
- B-tree: O(log n) with ~3-5 page accesses = 0.5-2ms
- LSM: May need to check multiple levels = 5-20ms worst case
- Bloom filters help but are probabilistic

### Q2: "How does compaction affect LSM performance?"

**Expected Answer:**
1. Compaction is the background process that merges SSTables
2. During compaction, disk I/O increases → "compaction pause"
3. Write amplification: every key-value pair is written 10-100 times over its lifetime
4. Tiered compaction: less write amplification, more space
5. Leveled compaction: better read performance, more write amplification
6. Production: tune compaction to match workload (size, thread count, timing)

### Q3: "How would you optimize an LSM-tree for read-heavy workloads?"

**Expected Answer:**
1. Increase bloom filter accuracy (more memory, more hash functions)
2. Use leveled compaction instead of tiered
3. Reduce number of levels by increasing file sizes
4. Add read cache (LRU cache for hot SSTables)
5. Consider hybrid: hot data in B-tree, cold in LSM
6. Leveled compaction + larger L1+ sizes reduces read amplification

### Q4: "What's the maximum data size for a single B-tree node?"

**Expected Answer:**
- Fanout factor ~200-400 depending on key size
- For 4KB pages: ~200 keys per internal page
- For 1M rows: depth = log_200(1M) ≈ 3
- For 1B rows: depth = log_200(1B) ≈ 5
- For 1T rows: depth = log_200(1T) ≈ 7 (impractical)
- Solution: Partition/shard the data across multiple B-trees

---

## 6. Real-World Production Examples

### Example 1: Discord (Message Storage)

```
Problem: Trillions of messages, write-heavy workload
Solution: ScyllaDB (Cassandra-compatible, LSM-based)
Results: 
- 10M writes/second across cluster
- <10ms p99 read latency
- Automatic compaction handling petabytes
```

### Example 2: Instagram (User Data)

```
Problem: User profiles, write-heavy with growth
Solution: PostgreSQL with BRIN indexes (block range indexes)
- BRIN is LSM-like: indexes small summaries per block
- Good for time-series data within tables
- Avoids full B-tree overhead for append-only patterns
```

### Example 3: Uber (Trip Data)

```
Problem: Billions of trips, mixed workload
Solution: PostgreSQL (primary) + Elasticsearch (search) + ClickHouse (analytics)
- PostgreSQL B-tree for primary storage (ACID needed)
- Separate systems for different access patterns
- This is the polyglot persistence pattern
```

---

## 7. Summary

| Aspect | B-Tree | LSM-Tree |
|--------|--------|----------|
| **Primary Use** | General-purpose OLTP | Write-heavy, time-series |
| **Read Latency** | ✅ Predictable, fast | Variable, slower |
| **Write Latency** | Slower (random I/O) | ✅ Fast (append) |
| **Space** | 10-30% overhead | 1.5-2x amplification |
| **Writes Amplification** | ✅ 1x | 10-100x |
| **Complexity** | Lower | Higher (compaction) |
| **Products** | PostgreSQL, MySQL | Cassandra, RocksDB |

**The Final Answer**: Start with B-trees (PostgreSQL/MySQL). If you have a specific write-heavy use case (IoT, logging, time-series), consider LSM-based databases (Cassandra, RocksDB, TimescaleDB). For most applications, the managed database handles this choice for you.
