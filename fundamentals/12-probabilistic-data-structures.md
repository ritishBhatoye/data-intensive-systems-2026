# Probabilistic Data Structures

> Bloom filters, HyperLogLog, Count-Min Sketch, and more — the secret weapons for working with massive-scale data with minimal memory.

---

## 1. The Problem: Exact vs. Approximate

When you're processing billions of items, exact data structures become prohibitively expensive:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    EXACT vs APPROXIMATE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  EXACT DATA STRUCTURES:                                                │
│  ─────────────────────                                                  │
│  • HashSet: Store every unique item                                    │
│    → 1B items × 8 bytes = 8GB just for keys!                          │
│  • HashMap: Count exact occurrences                                    │
│    → Even more memory                                                   │
│  • Sorted list: Find order statistics                                   │
│    → O(n) memory, O(log n) query                                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  APPROXIMATE DATA STRUCTURES:                                          │
│  ──────────────────────────                                            │
│  • Answer "is this item in the set?" → 1 bit per item possible        │
│  • Answer "how many unique items?" → ~1KB for billions                 │
│  • Answer "what's the frequency?" → ~10KB for millions                │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  THE TRADE-OFF:                                                         │
│  • Use less memory (10x-1000x improvement)                            │
│  • Answers are probabilistic (tunable accuracy)                        │
│  • Great for large-scale data processing, monitoring, caching          │
│                                                                          │
│  KEY INSIGHT: At scale, "good enough" beats "perfect but impossible"  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bloom Filters

### 2.1 The Problem Bloom Filters Solve

**Question**: Given a set of 1 billion keys, is a specific key present?

**Exact Solution**: HashSet
- Memory: ~40 bytes per key (Java) = 40GB

**Bloom Filter Solution**:
- Memory: ~1.2 bytes per key = 1.2GB (97% reduction!)
- Question: "Is key definitely NOT in set?" → DEFINITE YES
- Question: "Is key possibly in set?" → PROBABLY YES (with false positive rate)

### 2.2 How Bloom Filters Work

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BLOOM FILTER MECHANICS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  COMPONENTS:                                                            │
│  1. Bit array of m bits, initially all 0                              │
│  2. k hash functions, each mapping to [0, m-1]                        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ADD OPERATION: add("alice")                                            │
│  ────────────────────────                                               │
│                                                                          │
│  h1("alice") = 3  → set bit 3 to 1                                     │
│  h2("alice") = 7  → set bit 7 to 1                                     │
│  h3("alice") = 12 → set bit 12 to 1                                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │  Bit Array (m=16):                                          │        │
│  │  [0,0,0,1,0,0,0,1,0,0,0,0,1,0,0,0]                        │        │
│  │               ↑      ↑      ↑                               │        │
│  │               3      7      12                              │        │
│  └─────────────────────────────────────────────────────────────┘        │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  QUERY OPERATION: might_contain("alice")                               │
│  ─────────────────────────────────────                                  │
│                                                                          │
│  h1("alice") = 3  → bit[3] = 1?                                        │
│  h2("alice") = 7  → bit[7] = 1?                                        │
│  h3("alice") = 12 → bit[12] = 1?                                       │
│                                                                          │
│  If ALL bits are 1 → "PROBABLY YES" (might be false positive)        │
│  If ANY bit is 0  → "DEFINITELY NO"                                    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  KEY PROPERTY: False negatives are IMPOSSIBLE                         │
│  If we added it, all bits are 1 → query returns possibly               │
│  If we never added it, some bit might be 0 → query returns definitely not│
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Mathematical Analysis

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BLOOM FILTER MATH                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  VARIABLES:                                                             │
│  • n = number of items to insert                                       │
│  • m = number of bits in array                                          │
│  • k = number of hash functions                                         │
│  • p = false positive rate                                              │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  OPTIMAL NUMBER OF HASH FUNCTIONS:                                      │
│                                                                          │
│       k = (m/n) × ln(2)                                                 │
│                                                                          │
│  Example: m/n = 10 bits per item                                       │
│            k = 10 × 0.693 = 6.9 ≈ 7 hash functions                     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  FALSE POSITIVE PROBABILITY:                                           │
│                                                                          │
│       p ≈ (1 - e^(-kn/m))^k                                             │
│                                                                          │
│  Common configurations:                                                 │
│                                                                          │
│  │ Bits/Item │ Hash Funcs │ False Positive Rate │ Memory for 1M items │ │
│  │-----------|------------|---------------------|----------------------│  │
│  │ 5         | 3          | 14%                 | 625 KB               │  │
│  │ 10        | 7          | 1%                  │ 1.25 MB              │  │
│  │ 15        | 10         │ 0.1%                | 1.9 MB               │  │
│  │ 20        | 14         │ 0.01%               | 2.5 MB               │  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  MEMORY FORMULA:                                                        │
│                                                                          │
│       m = -n × ln(p) / (ln(2)^2)                                        │
│                                                                          │
│  Example: n=10M items, p=1%                                            │
│           m = -10M × ln(0.01) / (0.693^2)                              │
│           m = -10M × (-4.605) / 0.48                                   │
│           m = 96M bits = 12 MB                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Code Example

```python
import mmh3  # MurmurHash3
import math

class BloomFilter:
    def __init__(self, n, p):
        """Initialize bloom filter for n items with false positive rate p"""
        self.n = n
        self.p = p
        
        # Calculate optimal size and hash functions
        self.m = int(-n * math.log(p) / (math.log(2) ** 2))
        self.k = int((self.m / n) * math.log(2))
        
        self.bit_array = [0] * self.m
    
    def _hashes(self, item):
        """Generate k hash values for an item"""
        # Use double hashing technique: h(i) = h1 + i*h2
        h1 = mmh3.hash(str(item)) % self.m
        h2 = mmh3.hash(str(item), seed=42) % self.m
        
        for i in range(self.k):
            yield (h1 + i * h2) % self.m
    
    def add(self, item):
        """Add item to bloom filter"""
        for idx in self._hashes(item):
            self.bit_array[idx] = 1
    
    def might_contain(self, item):
        """Check if item might be in the set"""
        for idx in self._hashes(item):
            if self.bit_array[idx] == 0:
                return False
        return True
    
    def get_memory_bytes(self):
        """Return memory usage in bytes"""
        return self.m // 8


# Usage example
bloom = BloomFilter(n=10_000_000, p=0.01)  # 10M items, 1% FPR
bloom.add("alice")
bloom.add("bob")

print(bloom.might_contain("alice"))  # True
print(bloom.might_contain("charlie"))  # False (probably)
print(f"Memory: {bloom.get_memory_bytes() / 1024 / 1024:.2f} MB")  # ~1.2MB
```

### 2.5 Real-World Use Cases

| Company | Use Case | Memory Saved |
|---------|----------|--------------|
| **Google Bigtable** | Check if row/col exists before disk lookup | ~90% vs. storing keys |
| **Cassandra** | Avoid unnecessary disk reads for missing keys | Significant |
| **Bitcoin** | Store unspent transaction outputs | Early blockchain sync |
| **Medium** | Track articles a user has already seen | Personalized feeds |
| **Netflix** | Content caching decisions | Avoid caching duplicates |
| **Waze** | Filter out already-processed road segments | Real-time processing |

### 2.6 Bloom Filter Variations

| Variant | Description | Use Case |
|---------|-------------|----------|
| **Scalable Bloom Filter** | Dynamically grows to maintain FPR | Unknown dataset size |
| **Counting Bloom Filter** | Supports deletions (count instead of bit) | Set intersection |
| **cBloom Filter** | Uses compressed bits | Memory constrained |
| **Xor Filter** | Newer, faster, smaller | Replacement for Bloom |

---

## 3. HyperLogLog

### 3.1 The Problem HyperLogLog Solves

**Question**: How many unique visitors came to my website today?

**Exact Solution**: HashSet
- 100 million unique visitors × 8 bytes = 800MB
- Too expensive for real-time monitoring!

**HyperLogLog Solution**:
- 12KB memory for 100 million unique visitors!
- Accuracy: ~2% standard error

### 3.2 How HyperLogLog Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HYPERLOGLOG MECHANICS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CORE INSIGHT: If you see a hash value with k leading zeros,           │
│  you're likely seeing roughly 2^k items                                 │
│                                                                          │
│  Example:                                                                │
│  ─────────                                                               │
│  Hash("alice") = 00001...  (5 leading zeros) → ~2^5 = 32 unique items  │
│  Hash("bob")   = 00101...  (2 leading zeros) → ~2^2 = 4 unique items    │
│  Hash("carol") = 000001... (6 leading zeros) → ~2^6 = 64 unique items  │
│                                                                          │
│  The maximum leading zeros you've seen is a power-of-2 estimate!       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  THE PROBLEM: One hash can be way off                                   │
│                                                                          │
│  THE SOLUTION: Use multiple "registers" and average                      │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Hash space: 2^32                                              │    │
│  │  Split into 2^14 = 16,384 buckets (registers)                  │    │
│  │                                                                  │    │
│  │  For each item:                                                │    │
│  │  1. hash(item) → 32-bit value                                  │    │
│  │  2. bucket = top 14 bits                                       │    │
│  │  3. trailing_zeros = count zeros from bit 15 onwards          │    │
│  │  4. register[bucket] = max(register[bucket], trailing_zeros)  │    │
│  │                                                                  │    │
│  │  Final estimate:                                               │    │
│  │  m × 2^(average of all registers)                             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  WHY IT WORKS: Law of large numbers - random noise averages out!       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Mathematical Analysis

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HYPERLOGLOG MATH                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  VARIABLES:                                                             │
│  • m = number of registers (typically 2^p where p=10-16)               │
│  • p = log2(m)                                                          │
│  • registers[] = max trailing zeros seen in each bucket                 │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ESTIMATE FORMULA:                                                      │
│                                                                          │
│       α × m^2 / Σ(2^(-register[i]))                                      │
│                                                                          │
│  Where α is a correction factor:                                        │
│       α = 0.7213 / (1 + 1.079/m)   (for m ≥ 128)                       │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ACCURACY vs MEMORY:                                                   │
│                                                                          │
│  │ Registers │ Memory   │ Standard Error │ Can Count Up To            │  │
│  │-----------|----------|----------------|---------------------------|  │
│  │ 2^10=1024 | 2 KB     | 3.6%           | ~10^7                     │  │
│  │ 2^14=16384| 32 KB    │ 0.9%           | ~10^9                     │  │
│  │ 2^16=65536| 128 KB   │ 0.4%           | ~10^10                    │  │
│  │ 2^20=1M   | 2 MB     │ 0.09%          | ~10^12                    │  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HASH SIZE: Use 64-bit hash for counting up to 2^64 items               │
│                                                                          │
│  LINEAR COUNTING FOR SMALL COUNTS:                                      │
│  • Below threshold (~m × 10), use exact counting                        │
│  • HyperLogLog switches to "sparse" representation                      │
│  • Much more accurate for small cardinalities                          │
│                                                                          │
│  SPARSE vs DENSE:                                                       │
│  • Sparse: store as list of (register_idx, value) for near-zero regs   │
│  • Dense: full array of 6-bit counters                                 │
│  • HyperLogLog++ switches between them automatically                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Code Example

```python
import mmh3
import math

class HyperLogLog:
    def __init__(self, p=14):
        """Initialize with p bits for register count (default 14 = 16384)"""
        self.p = p
        self.m = 1 << p
        self.registers = [0] * self.m
        
        # Correction factor
        self.alpha = 0.7213 / (1 + 1.079 / self.m)
    
    def _hash_and_bucket(self, item):
        """Get hash and bucket index"""
        h = mmh3.hash64(str(item)) & 0xFFFFFFFFFFFFFFFF
        bucket = h & (self.m - 1)
        return bucket, h >> self.p
    
    def _count_trailing_zeros(self, x):
        """Count trailing zeros in binary representation"""
        if x == 0:
            return 64
        count = 0
        while (x & 1) == 0:
            count += 1
            x >>= 1
        return count
    
    def add(self, item):
        """Add item to counter"""
        bucket, hash_value = self._hash_and_bucket(item)
        trailing_zeros = self._count_trailing_zeros(hash_value) + 1
        self.registers[bucket] = max(self.registers[bucket], trailing_zeros)
    
    def count(self):
        """Estimate cardinality"""
        # Harmonic mean
        sum_inverse = sum(2 ** -r for r in self.registers if r > 0)
        
        if sum_inverse == 0:
            return 0
        
        estimate = self.alpha * (self.m ** 2) / sum_inverse
        
        # Apply corrections
        if estimate <= 2.5 * self.m:
            # Use linear counting for small counts
            zero_registers = self.registers.count(0)
            if zero_registers > 0:
                return int(self.m * math.log(self.m / zero_registers))
        
        # Cap at 2^64
        return min(estimate, 2**64)
    
    def get_memory_bytes(self):
        """Return memory usage"""
        return self.m  # Each register is 1 byte (can store up to 64)


# Usage
hll = HyperLogLog(p=14)  # 16384 registers = 16KB

# Simulate adding 1 million items
for i in range(1_000_000):
    hll.add(f"user_{i}")

print(f"Estimated count: {hll.count():,.0f}")  # ~1,000,000 ± 2%
print(f"Memory used: {hll.get_memory_bytes() / 1024:.1f} KB")  # ~16KB
```

### 3.5 Real-World Use Cases

| Company | Use Case | Impact |
|---------|----------|--------|
| **Redis** | PFADD, PFCOUNT commands | 12KB for billions of users |
| **Google** | Network traffic analysis | Per-router cardinality |
| **Netflix** | Unique viewer counts | Real-time dashboards |
| **Instagram** | Daily active users | Multi-region aggregation |
| **FinTech** | Unique transactions/day | Fraud detection |

---

## 4. Count-Min Sketch

### 4.1 The Problem Count-Min Sketch Solves

**Question**: How many times did each URL appear in access logs?

**Exact Solution**: HashMap<URL, count>
- 100 million unique URLs × (overhead) = terabytes

**Count-Min Sketch Solution**:
- ~10KB for millions of URLs
- Query any URL's frequency with guaranteed error bound

### 4.2 How Count-Min Sketch Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COUNT-MIN SKETCH MECHANICS                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STRUCTURE: d hash tables, each with w counters                          │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Hash Table 1          Hash Table 2          Hash Table 3      │    │
│  │  ┌───┬───┬───┬───┐    ┌───┬───┬───┬───┐    ┌───┬───┬───┬───┐   │    │
│  │  │ 0 │ 1 │ 2 │ 3 │    │ 0 │ 1 │ 2 │ 3 │    │ 0 │ 1 │ 2 │ 3 │   │    │
│  │  └───┴───┴───┴───┘    └───┴───┴───┴───┘    └───┴───┴───┴───┘   │    │
│  │    ↑     ↑                ↑     ↑               ↑     ↑           │    │
│  │    └─────┴────────────────┴─────┴───────────────┴─────┘           │    │
│  │     h1(item)            h2(item)            h3(item)              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ADD OPERATION: add("google.com")                                       │
│  ─────────────────────────────────────                                  │
│                                                                          │
│  1. h1("google.com") = 0  → increment table[0][0]                      │
│  2. h2("google.com") = 2  → increment table[1][2]                      │
│  3. h3("google.com") = 1  → increment table[2][1]                     │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  Hash Table 1          Hash Table 2          Hash Table 3      │    │
│  │  ┌───┬───┬───┬───┐    ┌───┬───┬───┬───┐    ┌───┬───┬───┬───┐   │    │
│  │  │ 1 │ 0 │ 0 │ 0 │    │ 0 │ 0 │ 1 │ 0 │    │ 0 │ 1 │ 0 │ 0 │   │    │
│  │  └───┴───┴───┴───┘    └───┴───┴───┴───┘    └───┴───┴───┴───┘   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  QUERY OPERATION: count("google.com")                                   │
│  ────────────────────────────────────────────                           │
│                                                                          │
│  1. h1 → table[0][0] = 1                                               │
│  2. h2 → table[1][2] = 1                                               │
│  3. h3 → table[2][1] = 1                                               │
│                                                                          │
│  Return MIN(1, 1, 1) = 1                                               │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  WHY MIN?                                                               │
│  • Hash collisions can cause overcount                                  │
│  • Taking MIN gives us an upper bound                                   │
│  • Actual count ≤ estimated count                                       │
│  • Error = O(log n) with high probability                               │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Mathematical Analysis

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COUNT-MIN SKETCH MATH                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  VARIABLES:                                                             │
│  • d = number of hash tables (typically 5-10)                         │
│  • w = number of counters per table (typically 2^p where p=10-20)     │
│  • ε = error bound (w = 2/ε)                                           │
│  • δ = failure probability (d = ln(1/δ))                                │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  ERROR BOUND:                                                           │
│                                                                          │
│       With probability (1 - δ):                                         │
│                                                                          │
│       estimated_count ≤ true_count + ε × N                             │
│                                                                          │
│  Where N = total number of items added                                  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  MEMORY: d × w counters                                                  │
│                                                                          │
│  Common configurations:                                                 │
│                                                                          │
│  │ ε     │ δ     │ d  │ w    │ Memory   │ Error at N=10M              │  │
│  │-------|-------|-----|-------|----------|----------------------------|  │
│  │ 0.01  │ 0.01  │ 5  │ 200  │ 1 KB     │ ≤ 100,000 (1%)             │  │
│  │ 0.001 │ 0.01  │ 5  │ 2000 │ 10 KB    │ ≤ 10,000 (0.1%)            │  │
│  │ 0.0001│ 0.001 │ 7  │ 20000│ 140 KB   │ ≤ 1,000 (0.01%)            │  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  KEY INSIGHT: Error is proportional to TOTAL count, not unique items! │
│                                                                          │
│  If you add 1 billion items, even 0.1% error = 1 million!              │
│  Solution: Use in sliding windows, decay old counts                    │
│                                                                          │
│  COUNT-MIN with DECAY:                                                   │
│  • Periodically divide all counters by 2                               │
│  • Recent items have more weight                                       │
│  • Error bounded by recent activity                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Code Example

```python
import mmh3
import math

class CountMinSketch:
    def __init__(self, epsilon=0.001, delta=0.01):
        """Initialize with error bound epsilon and failure probability delta"""
        self.epsilon = epsilon
        self.delta = delta
        
        # Calculate dimensions
        self.w = int(2 / epsilon)  # width
        self.d = int(math.log(1 / delta))  # depth
        
        # Create hash table
        self.table = [[0] * self.w for _ in range(self.d)]
        
        # Pre-generate random seeds for hash functions
        self.seeds = [i * 1000 for i in range(self.d)]
    
    def _hash(self, item, table_idx):
        """Hash item for specific table"""
        h = mmh3.hash(str(item), seed=self.seeds[table_idx])
        return h % self.w
    
    def add(self, item, count=1):
        """Add count to item"""
        for i in range(self.d):
            idx = self._hash(item, i)
            self.table[i][idx] += count
    
    def estimate(self, item):
        """Get estimated count (upper bound)"""
        min_count = float('inf')
        for i in range(self.d):
            idx = self._hash(item, i)
            min_count = min(min_count, self.table[i][idx])
        return min_count
    
    def get_memory_bytes(self):
        """Return memory usage"""
        return self.d * self.w * 4  # 4 bytes per int


# Usage
cms = CountMinSketch(epsilon=0.001, delta=0.01)  # ~10KB

# Add some data
for url in ["google.com", "google.com", "google.com", 
            "facebook.com", "facebook.com", "amazon.com"]:
    cms.add(url)

print(f"google.com: {cms.estimate('google.com')} (actual: 3)")
print(f"facebook.com: {cms.estimate('facebook.com')} (actual: 2)")
print(f"amazon.com: {cms.estimate('amazon.com')} (actual: 1)")
print(f"Memory: {cms.get_memory_bytes() / 1024:.1f} KB")
```

### 4.5 Real-World Use Cases

| Company | Use Case | Impact |
|---------|----------|--------|
| **DataDog** | APM top endpoints | Real-time latency percentiles |
| **Netflix** | Popular content ranking | Hot content identification |
| **Akamai** | Traffic analysis | DDoS detection |
| **Twitter** | Trending topics | Hashtag frequency |
| **Google** | Network flow analysis | Heavy hitter detection |

---

## 5. Comparison and Decision Framework

### 5.1 When to Use Which

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DECISION FRAMEWORK                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  QUESTION: What are you trying to do?                                   │
│         │                                                              │
│         ├── "Is this item in the set?"                                 │
│         │     │                                                          │
│         │     └── Use BLOOM FILTER                                      │
│         │         (definitely not / probably yes)                       │
│         │                                                              │
│         ├── "How many unique items?"                                    │
│         │     │                                                          │
│         │     └── Use HYPERLOGLOG                                      │
│         │         (2% error, 12KB for billions)                         │
│         │                                                              │
│         ├── "How frequently does each item appear?"                     │
│         │     │                                                          │
│         │     └── Use COUNT-MIN SKETCH                                 │
│         │         (upper bound on frequency)                           │
│         │                                                              │
│         └── "What's the most frequent item?"                            │
│               │                                                          │
│               └── Use COUNT-MIN + heap, or HEAVY HITTERS algorithm     │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  COMBINATIONS:                                                          │
│                                                                          │
│  Bloom + HLL: Check uniqueness first, then count                       │
│  CMS + HLL: Get both frequencies and cardinality                       │
│  Multiple CMS: Track different time windows                             │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  HYBRID APPROACH:                                                       │
│                                                                          │
│  Use exact counting for small datasets, switch to prob. for scale      │
│  Redis does this: HLL switches to sparse at low counts                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Memory Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MEMORY COMPARISON                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Task: Track 100 million items                                          │
│                                                                          │
│  ┌─────────────────────┬──────────────┬─────────────────────────────┐  │
│  │ Data Structure      │ Memory       │ Accuracy                    │  │
│  ├─────────────────────┼──────────────┼─────────────────────────────┤  │
│  │ HashSet             │ ~4-8 GB      │ Exact                       │  │
│  │ Bloom Filter        │ ~120 MB      │ 1% false positives          │  │
│  │ HyperLogLog         │ ~12 KB       │ ~2% standard error          │  │
│  │ Count-Min Sketch    │ ~1 MB        │ ~1% relative error          │  │
│  └─────────────────────┴──────────────┴─────────────────────────────┘  │
│                                                                          │
│  ───────────────────────────────────────────────────────────────────    │
│                                                                          │
│  Task: Track frequencies of 1 million URLs                              │
│                                                                          │
│  ┌─────────────────────┬──────────────┬─────────────────────────────┐  │
│  │ Data Structure      │ Memory       │ Accuracy                    │  │
│  ├─────────────────────┼──────────────┼─────────────────────────────┤  │
│  │ HashMap             │ ~100-200 MB  │ Exact                       │  │
│  │ Count-Min Sketch    │ ~10 MB       │ 0.1% relative error         │  │
│  │ CMS + Sliding Window│ ~20 MB       │ Last 1M events accurate      │  │
│  └─────────────────────┴──────────────┴─────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Interview Questions

### Q1: "What are the trade-offs of using a Bloom filter?"

**Expected Answer**:
- **Pros**: Space-efficient (1-2 bytes per item), O(k) time for add/query
- **Cons**: False positives possible (but not false negatives), can't delete items (without Counting Bloom Filter)
- **Use when**: Memory is constrained, occasional false positives are acceptable
- **Don't use when**: You need exact counts or deletions

### Q2: "How would you implement a "recent unique users" counter?"

**Expected Answer**:
1. Use sliding window approach
2. Maintain two HyperLogLogs: current window and previous window
3. Subtract to get "last hour unique"
4. Alternatively, use timestamped item storage with TTL and periodic cleanup

### Q3: "How would you find the top-K most frequent items?"

**Expected Answer**:
1. **Basic**: Count-Min Sketch gives frequencies, but finding top-K requires scanning all
2. **Efficient**: Count-Min Sketch + heap (maintain top-K in heap, update on each add)
3. **Advanced**: Count-Min with heavy hitters algorithm (detect and track "elephants")
4. **Alternative**: Misra-Gries algorithm (approximate frequent items with single pass)

### Q4: "What's the difference between HyperLogLog and Bloom Filter?"

**Expected Answer**:
- **Bloom Filter**: Answers membership questions (is item in set?)
- **HyperLogLog**: Answers cardinality questions (how many unique items?)
- Both are probabilistic but answer fundamentally different questions
- Can combine both: bloom for membership, HLL for uniqueness

---

## 7. Summary

| Structure | What It Answers | Memory | Error Type |
|-----------|----------------|--------|------------|
| **Bloom Filter** | "Is item in set?" | 1-2 bytes/item | False positives |
| **HyperLogLog** | "How many unique?" | ~12 KB | ~2% standard error |
| **Count-Min Sketch** | "How frequent?" | ~1 MB for 1M items | Upper bound, ~εN |
| **Misra-Gries** | "What are the top-K?" | O(K) space | Under-counts |

**Production Tips**:
- Use existing implementations (Redis, Java libraries)
- Tune parameters based on actual error tolerance
- Consider combining structures for complex queries
- Monitor accuracy with periodic ground-truth sampling
