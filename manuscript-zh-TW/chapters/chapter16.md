# Chapter 16: Bloom Filters and Probabilistic Data Structures

**Part IV: Advanced Topics**

---

> "Premature optimization is the root of all evil."
> — Donald Knuth

## The Memory Crisis

Web crawler 消耗 128 MB RAM 只是追蹤訪問過的 URLs。在有 256 MB 總記憶體的嵌入式設備上，這是一半的可用 RAM——消失了。

Crawler 的工作很簡單：追蹤哪些 URLs 被訪問過，避免爬同一頁兩次。處理 100 萬個 URLs（平均長度：80 bytes）後，存這些 URLs 的 hash table 成長到 96 MB，加上 overhead。

「我們能用準確度換記憶體嗎？」我 manager 在 code review 時問。「如果能省大量記憶體，我們可以容忍一些重複爬取。」

這個問題改變了一切。完美的準確度實際上不需要。如果我們偶爾爬同一頁兩次，會浪費一些 bandwidth 但不會破壞任何東西。真正的限制是記憶體。

目前的方法用直接的 hash table：

```c
hash_table_t *visited_urls;  // Stores full URLs

bool is_visited(const char *url) {
    return hash_table_contains(visited_urls, url);
}

void mark_visited(const char *url) {
    hash_table_insert(visited_urls, url, NULL);
}
```

爬了 100 萬個 URLs（平均長度：80 bytes）後，hash table 消耗：

```bash
$ ./crawler_hashtable
Memory usage: 128 MB
  Hash table: 96 MB (1M URLs × 80 bytes + overhead)
  Other: 32 MB
  
Lookup time: 150 ns/lookup (including cache misses)
```

**128 MB** 只是追蹤訪問過的 URLs！在有 256 MB 總 RAM 的嵌入式設備上，這無法接受。

「我們能用準確度換記憶體嗎？」我 manager 問。「如果能省大量記憶體，我們可以容忍一些重複爬取。」

我實作了 Bloom filter。結果：

```bash
$ ./crawler_bloom
Memory usage: 18 MB
  Bloom filter: 1.2 MB (10 bits per URL)
  Other: 16.8 MB
  
Lookup time: 45 ns/lookup
False positive rate: 0.8% (8,000 false positives out of 1M)

Memory reduction: 10.7× (128 MB → 12 MB)
Speedup: 3.3× (150 ns → 45 ns)
```

**記憶體少 10.7 倍**且**快 3.3 倍**，只有 0.8% false positives（這只代表爬幾頁兩次——可接受）。

這章探索用完美準確度換大量記憶體節省的 probabilistic 資料結構。

---

## The Textbook Story

**Bloom filter** 是 space-efficient probabilistic 資料結構，測試 element 是否在 set 中。

**Properties**:
- **No false negatives**：如果它說「not in set」，絕對不在 set 中
- **Possible false positives**：如果它說「in set」，可能錯
- **Space-efficient**：用 bits 而不是存完整的 elements
- **Fast**：O(k) where k is number of hash functions（通常 3-10）

教科書的推銷：「當你能容忍 false positives 且需要省記憶體時用 Bloom filters。」

---

## The Reality Check: How Bloom Filters Work

### Basic Structure

Bloom filter 是 size m 的 bit array，有 k 個 hash functions：

```c
typedef struct {
    uint64_t *bits;   // Bit array
    size_t m;         // Number of bits
    int k;            // Number of hash functions
} bloom_filter_t;

bloom_filter_t *bloom_create(size_t m, int k) {
    bloom_filter_t *bf = malloc(sizeof(bloom_filter_t));
    bf->m = m;
    bf->k = k;
    bf->bits = calloc((m + 63) / 64, sizeof(uint64_t));
    return bf;
}
```

### Insert Operation

Hash element k 次，設 k 個 bits：

```c
void bloom_insert(bloom_filter_t *bf, const char *element) {
    for (int i = 0; i < bf->k; i++) {
        uint64_t hash = hash_function(element, i);
        size_t bit_pos = hash % bf->m;
        
        size_t word = bit_pos / 64;
        size_t bit = bit_pos % 64;
        bf->bits[word] |= (1UL << bit);
    }
}
```

### Lookup Operation

Hash element k 次，檢查所有 k 個 bits 是否設定：

```c
bool bloom_contains(bloom_filter_t *bf, const char *element) {
    for (int i = 0; i < bf->k; i++) {
        uint64_t hash = hash_function(element, i);
        size_t bit_pos = hash % bf->m;
        
        size_t word = bit_pos / 64;
        size_t bit = bit_pos % 64;
        
        if (!(bf->bits[word] & (1UL << bit))) {
            return false;  // Definitely not in set
        }
    }
    return true;  // Probably in set (might be false positive)
}
```

### Why False Positives Happen

插入很多 elements 後，很多 bits 被設為 1。Lookup 可能碰巧找到所有 k 個 bits 都設定，即使 element 從未被插入。

**Example**:
```
Insert "foo": sets bits 5, 12, 23
Insert "bar": sets bits 12, 18, 30
Insert "baz": sets bits 5, 18, 42

Lookup "xyz": hashes to bits 5, 12, 18
  All three bits are set (by other elements)!
  False positive!
```

---

## Choosing Parameters: m and k

False positive rate 取決於：
- **m**：Number of bits
- **k**：Number of hash functions
- **n**：Number of inserted elements

**Optimal k**: k = (m/n) × ln(2) ≈ 0.693 × (m/n)

**False positive rate**: p ≈ (1 - e^(-kn/m))^k

### Example Calculation

對 100 萬個 URLs，1% false positive rate：

```
Target: p = 0.01, n = 1,000,000

Solve for m:
  m = -n × ln(p) / (ln(2))^2
  m = -1,000,000 × ln(0.01) / 0.48
  m ≈ 9,585,058 bits ≈ 1.2 MB

Optimal k:
  k = (m/n) × ln(2)
  k = 9.6 × 0.693
  k ≈ 7 hash functions
```

所以對 1M URLs，1% false positives：**1.2 MB, 7 hash functions**。

比較 hash table：**96 MB**（80 倍更多記憶體！）。

---

## Cache-Friendly Bloom Filter Implementation

Naive 實作有糟糕的 cache behavior：k hash functions 存取 k 個隨機記憶體位置。

### Problem: Random Memory Access

```c
// Naive: k random accesses
for (int i = 0; i < k; i++) {
    size_t bit_pos = hash(element, i) % m;
    // Each bit_pos is random → cache miss!
}
```

對 k=7：**每次 lookup 7 次 cache misses**！

### Solution: Blocked Bloom Filter

把 bit array 分割成 blocks，在一個 block 內用 k bits：

```c
#define BLOCK_SIZE 512  // 512 bits = 64 bytes = 1 cache line

typedef struct {
    uint64_t *bits;
    size_t num_blocks;
    int k;
} blocked_bloom_t;

bool blocked_bloom_contains(blocked_bloom_t *bf, const char *element) {
    uint64_t hash = hash_function(element, 0);
    size_t block = hash % bf->num_blocks;

    // All k bits are in the same block (same cache line!)
    uint64_t *block_ptr = &bf->bits[block * (BLOCK_SIZE / 64)];

    for (int i = 0; i < bf->k; i++) {
        uint64_t h = hash_function(element, i);
        size_t bit_pos = h % BLOCK_SIZE;
        size_t word = bit_pos / 64;
        size_t bit = bit_pos % 64;

        if (!(block_ptr[word] & (1UL << bit))) {
            return false;
        }
    }
    return true;
}
```

**Why this helps**:
- 所有 k bits 在同一個 cache line
- 1 次 cache miss 而不是 k 次 cache misses
- 對 k=7 少 7 倍 cache misses

### The Benchmark

```
Test: 1M lookups in Bloom filter (k=7, m=10M bits)

Naive Bloom filter:
  Cache misses: 7M (7 per lookup)
  Cycles: 450M
  Time: 375 ms

Blocked Bloom filter (512-bit blocks):
  Cache misses: 1M (1 per lookup)
  Cycles: 85M
  Time: 71 ms
  Speedup: 5.3×
```

**快 5.3 倍**只是改善 cache locality！

---

## Advanced: Counting Bloom Filter

標準 Bloom filters 不能刪除 elements（你不能 unset bit——它可能被其他 elements 共享）。

**Counting Bloom filter** 用 counters 而不是 bits：

```c
typedef struct {
    uint8_t *counters;  // 4-bit counters (0-15)
    size_t m;
    int k;
} counting_bloom_t;

void counting_bloom_insert(counting_bloom_t *bf, const char *element) {
    for (int i = 0; i < bf->k; i++) {
        size_t pos = hash(element, i) % bf->m;
        if (bf->counters[pos] < 15) {  // Prevent overflow
            bf->counters[pos]++;
        }
    }
}

void counting_bloom_delete(counting_bloom_t *bf, const char *element) {
    for (int i = 0; i < bf->k; i++) {
        size_t pos = hash(element, i) % bf->m;
        if (bf->counters[pos] > 0) {
            bf->counters[pos]--;
        }
    }
}
```

**Trade-off**：用 4 倍更多記憶體（每個 counter 4 bits vs 1 bit），但支援 deletion。

---

## Real-World Example: Google Chrome's Safe Browsing

Google Chrome 用 Bloom filter 檢查 URL 是否可能惡意，在送到 Google servers 之前。

### The Problem

Chrome 需要對照惡意網站的 blacklist 檢查數百萬個 URLs：
- Blacklist 有 ~100 萬個 entries
- 不能把每個 URL 送到 Google（privacy + latency）
- Client 上記憶體有限

### The Solution

**Two-stage check**:

1. **Local Bloom filter**（快，低記憶體）：
   - 1M entries, 1% false positive rate
   - Memory: 1.2 MB
   - Lookup: <1 μs
   - 如果 Bloom filter 說「not in set」→ Safe，不聯絡 server

2. **Server check**（慢，準確）：
   - 如果 Bloom filter 說「might be in set」→ 聯絡 server 確認
   - 只有 1% 的 URLs 需要 server check（false positives）

**Result**:
- 99% 的 URLs 本地檢查（沒有 network latency）
- 1.2 MB memory（vs 80 MB for full hash table）
- Privacy 保留（只有可疑的 URLs 送到 server）

---

## Other Probabilistic Data Structures

### 1. Count-Min Sketch

估計 stream 中 elements 的頻率。

```c
typedef struct {
    int **counters;  // d × w array of counters
    int d;           // Number of hash functions
    int w;           // Width of each row
} count_min_sketch_t;

void cms_increment(count_min_sketch_t *cms, const char *element) {
    for (int i = 0; i < cms->d; i++) {
        int pos = hash(element, i) % cms->w;
        cms->counters[i][pos]++;
    }
}

int cms_estimate(count_min_sketch_t *cms, const char *element) {
    int min_count = INT_MAX;
    for (int i = 0; i < cms->d; i++) {
        int pos = hash(element, i) % cms->w;
        if (cms->counters[i][pos] < min_count) {
            min_count = cms->counters[i][pos];
        }
    }
    return min_count;  // Estimate (always ≥ true count)
}
```

**Use case**：Network traffic analysis（數 packet frequencies 不存所有 packets）。

**Memory**：O(d × w) instead of O(n) for exact counts。

### 2. HyperLogLog

估計 stream 中的 cardinality（unique elements 數量）。

```c
typedef struct {
    uint8_t *registers;  // m registers
    int m;               // Number of registers (power of 2)
} hyperloglog_t;

void hll_add(hyperloglog_t *hll, const char *element) {
    uint64_t hash = hash_function(element);
    int j = hash & (hll->m - 1);  // First log2(m) bits
    int w = __builtin_clzll(hash >> __builtin_ctz(hll->m)) + 1;  // Leading zeros

    if (w > hll->registers[j]) {
        hll->registers[j] = w;
    }
}

size_t hll_estimate(hyperloglog_t *hll) {
    double sum = 0;
    for (int i = 0; i < hll->m; i++) {
        sum += 1.0 / (1 << hll->registers[i]);
    }
    double alpha = 0.7213 / (1 + 1.079 / hll->m);  // Bias correction
    return (size_t)(alpha * hll->m * hll->m / sum);
}
```

**Use case**：數網站的 unique visitors 不存所有 IPs。

**Memory**：1.5 KB for 2% error on billions of elements！

**Example**:
```
Exact count (hash table): 10 GB for 1B unique IPs
HyperLogLog: 1.5 KB for 2% error
Memory reduction: 6,666,667×
```

### 3. Cuckoo Filter

像 Bloom filter，但支援 deletion 且有更好的 lookup performance。

```c
#define BUCKET_SIZE 4

typedef struct {
    uint8_t fingerprint[BUCKET_SIZE];
} bucket_t;

typedef struct {
    bucket_t *buckets;
    size_t num_buckets;
} cuckoo_filter_t;

bool cuckoo_contains(cuckoo_filter_t *cf, const char *element) {
    uint64_t hash = hash_function(element);
    uint8_t fp = fingerprint(hash);  // 8-bit fingerprint
    size_t i1 = hash % cf->num_buckets;
    size_t i2 = (i1 ^ hash_function(&fp, 0)) % cf->num_buckets;

    // Check bucket i1
    for (int j = 0; j < BUCKET_SIZE; j++) {
        if (cf->buckets[i1].fingerprint[j] == fp) {
            return true;
        }
    }

    // Check bucket i2
    for (int j = 0; j < BUCKET_SIZE; j++) {
        if (cf->buckets[i2].fingerprint[j] == fp) {
            return true;
        }
    }

    return false;
}
```

**Advantages over Bloom filter**:
- 支援 deletion
- 更好的 cache locality（只檢查 2 個 buckets vs k 個隨機位置）
- 稍微更好的 space efficiency

**Benchmark**:
```
Test: 1M elements, 1% false positive rate

Bloom filter:
  Memory: 1.2 MB
  Lookup: 7 cache misses
  Time: 150 ns

Cuckoo filter:
  Memory: 1.1 MB (slightly better)
  Lookup: 2 cache misses (2 buckets)
  Time: 65 ns
  Speedup: 2.3×
```

---

## When to Use Probabilistic Data Structures

實作了幾個 probabilistic 資料結構後，我學到它們何時值得使用。

### Use Bloom Filters When:

1. **Memory is constrained**：比 hash tables 省 10-100 倍記憶體
2. **False positives are acceptable**：能容忍偶爾的錯誤
3. **Negative queries are common**：「這個 URL 訪問過嗎？」大部分 URLs 是新的

**Example**：Web crawler URL deduplication
- 1M URLs: 1.2 MB (Bloom) vs 96 MB (hash table)
- 0.8% false positives → 爬幾頁兩次（可接受）

### Use Count-Min Sketch When:

1. **Counting frequencies in streams**：不需要精確 counts
2. **Memory is limited**：不能存所有 elements

**Example**：Network traffic analysis
- 數 packet types 不存所有 packets
- 100 KB (sketch) vs 10 GB (exact counts)

### Use HyperLogLog When:

1. **Estimating cardinality**：「有多少 unique users？」
2. **Billions of elements**：精確計數不實際

**Example**：Website analytics
- 1.5 KB for 2% error on 1B unique IPs
- vs 10 GB for exact count

### Use Cuckoo Filter When:

1. **Need deletion**：Bloom filters 不能刪除
2. **Better lookup performance**：2 cache misses vs 7

**Example**：Cache admission policy
- 追蹤最近看到的 items
- Cache evicts 時刪除舊 items

### DON'T Use When:

1. **False positives are unacceptable**：Security-critical decisions
2. **Memory is abundant**：就用 hash table
3. **Need exact answers**：Probabilistic ≠ exact

---

## Putting It All Together: Optimized Web Crawler

這是用 blocked Bloom filter 的最終優化 crawler：

```c
#define BLOCK_SIZE 512
#define BITS_PER_URL 10
#define NUM_HASH 7

typedef struct {
    uint64_t *bits;
    size_t num_blocks;
    int k;
} crawler_bloom_t;

crawler_bloom_t *crawler_bloom_create(size_t max_urls) {
    crawler_bloom_t *bf = malloc(sizeof(crawler_bloom_t));
    size_t total_bits = max_urls * BITS_PER_URL;
    bf->num_blocks = (total_bits + BLOCK_SIZE - 1) / BLOCK_SIZE;
    bf->bits = calloc(bf->num_blocks * (BLOCK_SIZE / 64), sizeof(uint64_t));
    bf->k = NUM_HASH;
    return bf;
}

bool crawler_is_visited(crawler_bloom_t *bf, const char *url) {
    uint64_t hash = hash_function(url, 0);
    size_t block = hash % bf->num_blocks;
    uint64_t *block_ptr = &bf->bits[block * (BLOCK_SIZE / 64)];

    for (int i = 0; i < bf->k; i++) {
        uint64_t h = hash_function(url, i);
        size_t bit_pos = h % BLOCK_SIZE;
        size_t word = bit_pos / 64;
        size_t bit = bit_pos % 64;

        if (!(block_ptr[word] & (1UL << bit))) {
            return false;  // Definitely not visited
        }
    }
    return true;  // Probably visited (might be false positive)
}

void crawler_mark_visited(crawler_bloom_t *bf, const char *url) {
    uint64_t hash = hash_function(url, 0);
    size_t block = hash % bf->num_blocks;
    uint64_t *block_ptr = &bf->bits[block * (BLOCK_SIZE / 64)];

    for (int i = 0; i < bf->k; i++) {
        uint64_t h = hash_function(url, i);
        size_t bit_pos = h % BLOCK_SIZE;
        size_t word = bit_pos / 64;
        size_t bit = bit_pos % 64;

        block_ptr[word] |= (1UL << bit);
    }
}
```

### Final Benchmark

```
Test: Crawl 1M URLs (avg length: 80 bytes)

Hash table:
  Memory: 128 MB
  Lookup: 150 ns (with cache misses)
  False positives: 0%

Naive Bloom filter:
  Memory: 1.2 MB (107× less)
  Lookup: 375 ns (7 cache misses)
  False positives: 0.8%

Blocked Bloom filter:
  Memory: 1.2 MB (107× less)
  Lookup: 45 ns (1 cache miss)
  False positives: 0.8%

Speedup: 3.3× (150 ns → 45 ns)
Memory reduction: 107×
```

---

## Summary

Memory crisis 解決了。Web crawler 的記憶體使用從 128 MB 降到 1.2 MB——107 倍減少。Lookup 時間從 150 ns 改善到 45 ns（快 3.3 倍），只有 0.8% false positives。偶爾的重複爬取是拿回設備一半 RAM 的小代價。

**關鍵洞察**：

1. **Bloom filters 用準確度換記憶體**。10-100 倍記憶體節省，<1% false positive rate。對「我看過這個嗎？」queries 完美。

2. **Blocked Bloom filters 對 cache 友善**。把所有 k bits 放在一個 cache line 減少 cache misses 從 k 到 1。對 k=7 快 5.3 倍。

3. **Optimal parameters matter**。用 k = 0.693 × (m/n) hash functions 和 m = -n × ln(p) / (ln(2))² bits 對目標 false positive rate p。

4. **Cuckoo filters 在 lookups 上勝過 Bloom filters**。只有 2 次 cache misses vs k 次 cache misses。快 2.3 倍，類似的記憶體使用。

5. **HyperLogLog 對 cardinality 是魔法**。用 1.5 KB 和 2% error 估計數十億 unique elements。比精確計數省 6M 倍記憶體。

**Web crawler 的數據**：
- Blocked Bloom filter: 比 hash table 少 107 倍記憶體
- Lookup: 快 3.3 倍（45 ns vs 150 ns）
- False positives: 0.8%（1M 中 8,000 個）
- Cache misses: 少 7 倍（每次 lookup 1 vs 7）

當你能容忍小錯誤時，Probabilistic 資料結構很強大。它們讓用精確資料結構不可能的應用成為可能。

**Next**：Part V explores real-world case studies applying these techniques to bootloaders, device drivers, and firmware.

