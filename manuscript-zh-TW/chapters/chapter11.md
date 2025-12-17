# Chapter 11: Tries and Radix Trees

**Part III: Trees and Hierarchies**

---

> "The cheapest, fastest, and most reliable components are those that aren't there."
> — Gordon Bell

## The Autocomplete Disaster

Trie 比 hash table 慢了 8 倍。而且它消耗了 128 MB 記憶體，hash table 只用 24 MB。

這不應該發生。Tries 是教科書上 autocomplete 的解決方案——O(k) lookup，k 是字串長度，跟資料集大小無關。對 prefix matching 很完美。Autocomplete、spell checkers 和 IP routing tables 的標準選擇。

「用 trie，」我隊友為我們 command-line tool 的 autocomplete 功能建議。我們有 50,000 個 commands 和 options 要搜尋。教科書同意這個選擇。

所以我們實作了 trie。Benchmark 結果很慘烈：

```bash
$ perf stat -e cache-misses,cycles ./autocomplete_trie "git com"
  Performance counter stats:
    125,000 cache-misses
  4,800,000 cycles

$ perf stat -e cache-misses,cycles ./autocomplete_hash "git com"
  Performance counter stats:
     18,000 cache-misses
    600,000 cycles
```

Trie 比簡單的 hash table **慢了 8 倍**。而且它用了 **128 MB 記憶體**，hash table 只用 24 MB。

哪裡出錯了？

## The Textbook Story

Trie（發音 "try"）是一個 tree，每個 edge 代表一個字元。這是 "cat"、"car" 和 "dog" 的 trie：

```
       root
      /    \
     c      d
     |      |
     a      o
    / \     |
   t   r    g
```

要查找 "car"，你跟著 edges 走：root → 'c' → 'a' → 'r'。

**教科書的推銷**：
- **Prefix sharing**："cat" 和 "car" 共享 "ca" prefix
- **O(k) lookup**：只取決於字串長度，不是資料集大小
- **No string comparisons**：只要跟著 pointers 走
- **Perfect for autocomplete**：找所有 prefix "ca" 的字，traverse subtree

聽起來很完美，對吧？

## The Reality Check

這是我實作的 trie node 結構：

```c
typedef struct trie_node {
    struct trie_node *children[256];  // 2,048 bytes (256 × 8-byte pointers)
    void *value;                      // 8 bytes
    bool is_end;                      // 1 byte
    // Padding: 7 bytes
    // Total: 2,064 bytes per node
} trie_node_t;
```

**每個 node 2,064 bytes！** 那是 32 個 cache lines（每個 64 bytes）。

對我們 50,000 個 commands，平均長度 8 個字元：
- 需要的 nodes：~400,000（每個字元一個，有 sharing）
- 記憶體：400,000 × 2,064 = **825 MB**
- Hash table：50,000 × 24 = **1.2 MB**

Trie 用了比 hash table **多 687 倍的記憶體**。

### The Cache Problem

讓我們追蹤 "hello" 的 lookup：

```
Step 1: root → children['h']     (cache miss - load root node)
Step 2: node → children['e']     (cache miss - load 'h' node)
Step 3: node → children['l']     (cache miss - load 'e' node)
Step 4: node → children['l']     (cache miss - load first 'l' node)
Step 5: node → children['o']     (cache miss - load second 'l' node)

Total: 5 cache misses for a 5-character word
```

每個 node 是 2 KB，所以它們不能全部放進 cache。每次字元 lookup 都是 cache miss。

比較 hash table：hash 字串（便宜），一次 cache miss 抓取 bucket，完成。總計：1-2 次 cache misses。

---

## Solution 1: Radix Trees (Patricia Tries)

第一個優化是 **壓縮只有單一 child 的 node chains**。

在標準 trie 中，"cat" 和 "car" 是：

```
    root
     |
     c
     |
     a
    / \
   t   r
```

'c' 和 'a' 的 nodes 都只有一個 child。我們可以把它們壓縮成一個 node，prefix "ca"：

```
    root
     |
    "ca"
    / \
  "t" "r"
```

這叫做 **radix tree** 或 **Patricia trie**。

這是我用的實作：

```c
typedef struct radix_node {
    char *prefix;                // Variable-length prefix
    int prefix_len;
    struct radix_node *children[256];
    void *value;
} radix_node_t;
```

Lookup 現在先 match prefix，然後 descend：

```c
void* radix_search(radix_node_t *node, const char *key) {
    while (node) {
        // Match prefix
        int i = 0;
        while (i < node->prefix_len && key[i] == node->prefix[i]) {
            i++;
        }

        // Prefix mismatch?
        if (i < node->prefix_len) {
            return NULL;
        }

        // Exact match?
        if (key[i] == '\0') {
            return node->value;
        }

        // Descend to child
        node = node->children[(unsigned char)key[i]];
        key += i + 1;
    }
    return NULL;
}
```

對我們的 autocomplete tool，這減少了 60% 記憶體使用（從 825 MB 到 330 MB）。但還是太多了。

---

## Solution 2: Adaptive Radix Tree (ART)

Radix tree 有幫助，但我們還有個問題：每個 node 有 256-pointer array（2,048 bytes），即使它只有 2 個 children。

我看了資料。大部分 nodes 有少於 10 個 children。我們浪費了那些 arrays 98% 的空間。

解決方案：**adaptive node types**。根據你有多少 children 使用不同的 node 結構。

### Node Types

**Node4**（1-4 children）：
```c
typedef struct {
    uint8_t num_children;
    uint8_t keys[4];              // 4 bytes
    void *children[4];            // 32 bytes
    // Total: 40 bytes
} node4_t;
```

**Node16**（5-16 children）：
```c
typedef struct {
    uint8_t num_children;
    uint8_t keys[16];             // 16 bytes
    void *children[16];           // 128 bytes
    // Total: 152 bytes
} node16_t;
```

**Node48**（17-48 children）：
```c
typedef struct {
    uint8_t num_children;
    uint8_t index[256];           // Map char → child index
    void *children[48];           // 384 bytes
    // Total: 640 bytes
} node48_t;
```

**Node256**（49-256 children）：
```c
typedef struct {
    void *children[256];          // 2,048 bytes
} node256_t;
```

### Adaptive Growth

**Strategy**：從 Node4 開始，隨著 children 加入而成長。

```
Insert 1st child:  Node4
Insert 5th child:  Node4 → Node16
Insert 17th child: Node16 → Node48
Insert 49th child: Node48 → Node256
```

**Memory savings**:
- 平均 node：40-152 bytes（vs 2,048 bytes）
- 10-50 倍記憶體減少

---

## The Benchmark Results

我用 Adaptive Radix Tree 重新實作了我們的 autocomplete tool。這是比較：

```
Dataset: 50,000 commands (avg length 8 chars)
Test: 1,000,000 random lookups

Standard trie:
  Memory: 825 MB
  Cycles/lookup: 4,800
  Cache misses: 12.5

Radix tree:
  Memory: 330 MB
  Cycles/lookup: 2,400
  Cache misses: 6.8
  Speedup: 2.0×

Adaptive Radix Tree (ART):
  Memory: 18 MB
  Cycles/lookup: 1,200
  Cache misses: 3.2
  Speedup: 4.0×

Hash table (baseline):
  Memory: 1.2 MB
  Cycles/lookup: 600
  Cache misses: 1.8
```

ART 比標準 trie 快 4 倍，用了少 45 倍的記憶體。但 hash table 還是快 2 倍。

**為什麼 ART 比標準 tries 好**：
1. **更小的 nodes**：Node4/Node16 放進 1-2 個 cache lines，而不是 32 個
2. **更少 cache misses**：每次 lookup 3.2 vs 12.5
3. **更少記憶體**：18 MB vs 825 MB

**為什麼 hash tables 對 exact lookups 還是贏**：
- **Single cache miss**：直接 hash 到 bucket
- **No pointer chasing**：一次 lookup，完成

---

## When Tries Make Sense

經過這一切，你可能想：「我應該用 trie 嗎？」

是的——但只有當你需要 hash tables 無法提供的 **prefix operations**。

### 1. Autocomplete

我們的 autocomplete tool 需要找所有以 "git co" 開頭的 commands。Hash table 無法有效率地做這個——你必須掃描所有 50,000 個 entries。

用 ART，你 traverse 到 "git co" prefix，然後 enumerate 所有 children。這是 O(k + m)，k 是 prefix 長度，m 是 matches 數量。

我們最後用 ART 做 autocomplete，即使比 hash tables 慢 2 倍，因為我們需要 prefix matching。

### 2. IP Routing Tables

IP routers 需要 **longest prefix matching**。對 IP address 192.168.1.100，找最長的 matching route：
- 192.168.0.0/16 → Gateway A
- 192.168.1.0/24 → Gateway B（longer match，用這個）

Tries 對這個很完美。IP address 的每個 bit 是 tree 中的一個 branch。

### 3. Spell Checkers

找跟拼錯的字 edit distance 1-2 的字需要探索相似的 prefixes。Tries 讓這個很有效率。

### 4. When NOT to Use Tries

不要用 tries 在：
- **Exact lookups only**：用 hash table（2 倍快，10 倍少記憶體）
- **Small datasets**（< 1,000 entries）：Hash table overhead 可以忽略
- **Random strings**：如果沒有 prefix sharing，tries 浪費記憶體

---

## Real-World Example: Linux Kernel Radix Trees

Linux kernel 用 radix trees 在：
- **Page cache**：Mapping file offsets to memory pages
- **IDR (ID allocator)**：Allocating unique IDs
- **XArray**：Generic indexed storage

這是 kernel 的 radix tree node（從 `lib/radix-tree.c`）：

```c
struct radix_tree_node {
    unsigned char shift;      // Height in tree
    unsigned char offset;     // Slot offset in parent
    unsigned int count;       // Number of children
    struct radix_tree_node *parent;
    void *slots[RADIX_TREE_MAP_SIZE];  // 64 slots
};
```

Kernel 用固定的 branching factor 64（每層 6 bits）。對 32-bit index：
- Height：32 ÷ 6 ≈ 6 levels
- Cache misses：每次 lookup ~6 次

這比 binary tree 的 32 層好多了。

**為什麼 kernel 用 radix trees**：
1. **Sparse arrays**：File offsets 是 sparse 的（不是每個 page 都 cached）
2. **Range operations**：Iterate over pages in a file range
3. **Predictable performance**：O(log₆₄ n) worst case

---

## Summary

Autocomplete disaster 被挽救了。把標準 trie 換成 Adaptive Radix Tree，記憶體使用從 825 MB 降到 18 MB，lookups 快了 4 倍。ART 提供了我們需要的 prefix matching，雖然 hash tables 對 exact lookups 還是快 2 倍。

**關鍵洞察**：

1. **標準 tries 是記憶體怪獸**。每個 node 有 256-pointer arrays，它們用了比 hash tables 多 50-100 倍的記憶體。

2. **Radix trees 壓縮 chains**。透過合併 single-child nodes，你可以減少 60-70% 記憶體。

3. **Adaptive node types 很關鍵**。大部分 nodes 有少數 children。用 Node4/Node16 而不是 256-pointer arrays 再減少 10 倍記憶體。

4. **Tries 是用於 prefix operations**。如果你只需要 exact lookups，用 hash table。Tries 在你需要 autocomplete、longest prefix matching 或 edit distance queries 時發光。

5. **Cache misses 主導**。即使用 ART，你還是在 traverse k 層，字串長度 k。每層都是潛在的 cache miss。Hash tables 用 1-2 次 cache misses 總計就贏了。

**我們 autocomplete tool 的數據**：
- Standard trie: 4,800 cycles/lookup, 825 MB memory
- Adaptive Radix Tree: 1,200 cycles/lookup, 18 MB memory
- Hash table: 600 cycles/lookup, 1.2 MB memory

我們選 ART 因為我們需要 prefix matching，但如果我們只需要 exact lookups，hash tables 會是明顯的贏家。

**Next chapter**：Heaps and priority queues—how to maintain sorted order with O(log n) operations.

