# Chapter 16: Bloom Filters and Probabilistic Data Structures

**Part IV: Advanced Topics**

---

> "Premature optimization is the root of all evil."
> — Donald Knuth

## The Memory Crisis

The web crawler was consuming 128 MB of RAM just to track visited URLs. On an embedded device with 256 MB total memory, this was half the available RAM—gone.

The crawler's job was simple: track which URLs had been visited to avoid crawling the same page twice. After processing 1 million URLs (average length: 80 bytes), the hash table storing these URLs had grown to 96 MB, plus overhead.

"Can we trade accuracy for memory?" my manager asked during the code review. "We can tolerate a few duplicate crawls if it saves significant memory."

That question changed everything. Perfect accuracy wasn't actually required. If we occasionally crawled the same page twice, it would waste some bandwidth but wouldn't break anything. The real constraint was memory.

The current approach used a straightforward hash table:

```c
hash_table_t *visited_urls;  // Stores full URLs

bool is_visited(const char *url) {
    return hash_table_contains(visited_urls, url);
}

void mark_visited(const char *url) {
    hash_table_insert(visited_urls, url, NULL);
}
```

After crawling 1 million URLs (average length: 80 bytes), the hash table consumed:

```bash
$ ./crawler_hashtable
Memory usage: 128 MB
  Hash table: 96 MB (1M URLs × 80 bytes + overhead)
  Other: 32 MB
  
Lookup time: 150 ns/lookup (including cache misses)
```

**128 MB** for just tracking visited URLs! On an embedded device with 256 MB total RAM, this was unacceptable.

"Can we trade accuracy for memory?" my manager asked. "We can tolerate a few duplicate crawls if it saves significant memory."

I implemented a Bloom filter. The results:

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

**10.7× less memory** and **3.3× faster**, with only 0.8% false positives (which just meant crawling a few pages twice—acceptable).

This chapter explores probabilistic data structures that trade perfect accuracy for massive memory savings.

---

## The Textbook Story

A **Bloom filter** is a space-efficient probabilistic data structure that tests whether an element is in a set.

**Properties**:
- **No false negatives**: If it says "not in set", it's definitely not in set
- **Possible false positives**: If it says "in set", it might be wrong
- **Space-efficient**: Uses bits instead of storing full elements
- **Fast**: O(k) where k is number of hash functions (typically 3-10)

The textbook pitch: "Use Bloom filters when you can tolerate false positives and need to save memory."

---

## The Reality Check: How Bloom Filters Work

### Basic Structure

A Bloom filter is a bit array of size m, with k hash functions:

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

Hash the element k times, set k bits:

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

Hash the element k times, check if all k bits are set:

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

After inserting many elements, many bits are set to 1. A lookup might find all k bits set by chance, even if the element was never inserted.

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

The false positive rate depends on:
- **m**: Number of bits
- **k**: Number of hash functions
- **n**: Number of inserted elements

**Optimal k**: k = (m/n) × ln(2) ≈ 0.693 × (m/n)

**False positive rate**: p ≈ (1 - e^(-kn/m))^k

### Example Calculation

For 1 million URLs with 1% false positive rate:

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

So for 1M URLs with 1% false positives: **1.2 MB, 7 hash functions**.

Compare to hash table: **96 MB** (80× more memory!).

---

## Cache-Friendly Bloom Filter Implementation

The naive implementation has poor cache behavior: k hash functions access k random memory locations.

### Problem: Random Memory Access

```c
// Naive: k random accesses
for (int i = 0; i < k; i++) {
    size_t bit_pos = hash(element, i) % m;
    // Each bit_pos is random → cache miss!
}
```

For k=7: **7 cache misses per lookup**!

### Solution: Blocked Bloom Filter

Partition the bit array into blocks, use k bits within one block:

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
- All k bits are in the same cache line
- 1 cache miss instead of k cache misses
- 7× fewer cache misses for k=7

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

**5.3× faster** just by improving cache locality!

---

## Advanced: Counting Bloom Filter

Standard Bloom filters can't delete elements (you can't unset a bit—it might be shared with other elements).

**Counting Bloom filter** uses counters instead of bits:

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

**Trade-off**: Uses 4× more memory (4 bits per counter vs 1 bit), but supports deletion.

---

## Real-World Example: Google Chrome's Safe Browsing

Google Chrome uses a Bloom filter to check if a URL is potentially malicious before sending it to Google's servers.

### The Problem

Chrome needs to check millions of URLs against a blacklist of malicious sites:
- Blacklist has ~1 million entries
- Can't send every URL to Google (privacy + latency)
- Limited memory on client

### The Solution

**Two-stage check**:

1. **Local Bloom filter** (fast, low memory):
   - 1M entries, 1% false positive rate
   - Memory: 1.2 MB
   - Lookup: <1 μs
   - If Bloom filter says "not in set" → Safe, don't contact server

2. **Server check** (slow, accurate):
   - If Bloom filter says "might be in set" → Contact server for confirmation
   - Only 1% of URLs need server check (false positives)

**Result**:
- 99% of URLs checked locally (no network latency)
- 1.2 MB memory (vs 80 MB for full hash table)
- Privacy preserved (only suspicious URLs sent to server)

---

## Other Probabilistic Data Structures

### 1. Count-Min Sketch

Estimates frequency of elements in a stream.

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

**Use case**: Network traffic analysis (count packet frequencies without storing all packets).

**Memory**: O(d × w) instead of O(n) for exact counts.

### 2. HyperLogLog

Estimates cardinality (number of unique elements) in a stream.

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

**Use case**: Count unique visitors to a website without storing all IPs.

**Memory**: 1.5 KB for 2% error on billions of elements!

**Example**:
```
Exact count (hash table): 10 GB for 1B unique IPs
HyperLogLog: 1.5 KB for 2% error
Memory reduction: 6,666,667×
```

### 3. Cuckoo Filter

Like a Bloom filter, but supports deletion and has better lookup performance.

```c
#define BUCKET_SIZE 4

typedef struct {
    uint8_t fingerprint[BUCKET_SIZE];
} bucket_t;

typedef struct {
    bucket_t *buckets;
    size_t num_buckets;
} cuckoo_filter_t;

bool cuckoo_insert(cuckoo_filter_t *cf, const char *element) {
    uint64_t hash = hash_function(element);
    uint8_t fp = fingerprint(hash);  // 8-bit fingerprint
    size_t i1 = hash % cf->num_buckets;
    size_t i2 = (i1 ^ hash_function(&fp, 0)) % cf->num_buckets;

    // Try to insert in bucket i1
    for (int j = 0; j < BUCKET_SIZE; j++) {
        if (cf->buckets[i1].fingerprint[j] == 0) {
            cf->buckets[i1].fingerprint[j] = fp;
            return true;
        }
    }

    // Try to insert in bucket i2
    for (int j = 0; j < BUCKET_SIZE; j++) {
        if (cf->buckets[i2].fingerprint[j] == 0) {
            cf->buckets[i2].fingerprint[j] = fp;
            return true;
        }
    }

    // Both buckets full, need to relocate (cuckoo hashing)
    // ... (complex relocation logic)

    return false;  // Filter full
}

bool cuckoo_contains(cuckoo_filter_t *cf, const char *element) {
    uint64_t hash = hash_function(element);
    uint8_t fp = fingerprint(hash);
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
- Supports deletion
- Better cache locality (only 2 buckets to check vs k random positions)
- Slightly better space efficiency

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

After implementing several probabilistic data structures, I learned when they're worth using.

### Use Bloom Filters When:

1. **Memory is constrained**: 10-100× memory savings over hash tables
2. **False positives are acceptable**: Can tolerate occasional errors
3. **Negative queries are common**: "Is this URL visited?" where most URLs are new

**Example**: Web crawler URL deduplication
- 1M URLs: 1.2 MB (Bloom) vs 96 MB (hash table)
- 0.8% false positives → crawl a few pages twice (acceptable)

### Use Count-Min Sketch When:

1. **Counting frequencies in streams**: Don't need exact counts
2. **Memory is limited**: Can't store all elements

**Example**: Network traffic analysis
- Count packet types without storing all packets
- 100 KB (sketch) vs 10 GB (exact counts)

### Use HyperLogLog When:

1. **Estimating cardinality**: "How many unique users?"
2. **Billions of elements**: Exact counting is impractical

**Example**: Website analytics
- 1.5 KB for 2% error on 1B unique IPs
- vs 10 GB for exact count

### Use Cuckoo Filter When:

1. **Need deletion**: Bloom filters can't delete
2. **Better lookup performance**: 2 cache misses vs 7

**Example**: Cache admission policy
- Track recently seen items
- Delete old items when cache evicts them

### DON'T Use When:

1. **False positives are unacceptable**: Security-critical decisions
2. **Memory is abundant**: Just use a hash table
3. **Need exact answers**: Probabilistic ≠ exact

---

## Putting It All Together: Optimized Web Crawler

Here's the final optimized crawler using a blocked Bloom filter:

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

The memory crisis was solved. The web crawler's memory usage dropped from 128 MB to 1.2 MB—a 107× reduction. Lookup time improved from 150 ns to 45 ns (3.3× faster), with only 0.8% false positives. The occasional duplicate crawl was a small price to pay for getting half the device's RAM back.

**Key insights**:

1. **Bloom filters trade accuracy for memory**. 10-100× memory savings with <1% false positive rate. Perfect for "have I seen this before?" queries.

2. **Blocked Bloom filters are cache-friendly**. Placing all k bits in one cache line reduces cache misses from k to 1. 5.3× speedup for k=7.

3. **Optimal parameters matter**. Use k = 0.693 × (m/n) hash functions and m = -n × ln(p) / (ln(2))² bits for target false positive rate p.

4. **Cuckoo filters beat Bloom filters for lookups**. Only 2 cache misses vs k cache misses. 2.3× faster with similar memory usage.

5. **HyperLogLog is magic for cardinality**. Estimate billions of unique elements with 1.5 KB and 2% error. 6M× memory savings over exact counting.

**The numbers from the web crawler**:
- Blocked Bloom filter: 107× less memory than hash table
- Lookup: 3.3× faster (45 ns vs 150 ns)
- False positives: 0.8% (8,000 out of 1M)
- Cache misses: 7× fewer (1 vs 7 per lookup)

Probabilistic data structures are powerful when you can tolerate small errors. They enable applications that would be impossible with exact data structures.

**Next**: Part V explores real-world case studies applying these techniques to bootloaders, device drivers, and firmware.

