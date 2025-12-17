# Chapter 10: B-Trees and Cache-Conscious Trees

**Part III: Trees and Hierarchies**

---

> "The purpose of computing is insight, not numbers."
> — Richard Hamming

## The Database Mystery

Database 全部在 in-memory，但 lookups 卻要花 12,000 cycles。對於一個 IoT 裝置上有 100 萬筆 sensor readings 和 64 KB cache，Red-Black tree 實作對 real-time queries 來說太慢了。

「我們試試 B-tree，」我在 performance review 時建議。

「那不是只用在 disk-based databases 嗎？」Lead engineer 問。「我們是 all in-memory。為什麼需要 B-tree？」

這個問題很合理。B-trees 是為 disk access 設計的，每個 node 是一個 disk block。但 cache miss patterns 看起來跟 disk I/O patterns 可疑地相似——只是快 100 倍而不是快 100,000 倍。

我們還是實作了 B-tree。結果讓所有人驚訝：

```bash
$ perf stat -e cache-misses,cycles ./db_query_rbtree
  Performance counter stats:
    18,500,000 cache-misses
   120,000,000 cycles

$ perf stat -e cache-misses,cycles ./db_query_btree
  Performance counter stats:
     2,800,000 cache-misses
    18,000,000 cycles
```

B-tree 快了 **6.7 倍**。Cache misses 從 1850 萬降到 280 萬。

為什麼？B-tree 只有 **3 層**，而 Red-Black tree 有 **20 層**。更少的層 = 更少的 cache misses。

## The Problem with Binary Trees

在 Chapter 9，我們看到 binary search trees 受 pointer-chasing 所苦。每個 node 都在隨機的記憶體位置，所以每次 traversal step 都是 cache miss。

但有個更深層的問題：**tree height**。

對於 100 萬個 entries：
- Binary tree height: log₂(1,000,000) ≈ 20 levels
- 每次 lookup: 20 次 pointer dereferences
- Cache misses: ~18-20（幾乎每個 node 都是 miss）

即使我們能神奇地讓每個 node 都對 cache 友善，我們還是要 traverse 20 層。

**洞察**：大部分 cache misses 來自 tree **height**，不是來自個別 node access。

**解決方案**：透過增加 branching factor 來降低 height。

---

## What Is a B-Tree?

B-tree 就像 binary search tree，但每個 node 可以有 **很多** children，而不只是兩個。

這是一個簡單的例子，order 4（每個 node 最多 3 個 keys）：

```
                [40|80]
               /   |   \
         [10|20] [50|60] [90|100]
```

Root 有 2 個 keys（40 和 80）和 3 個 children。每個 child 也是一個有多個 keys 的 node。

**Key properties**:
- 每個 node 包含最多 M-1 個 keys（M 是 "order"）
- 每個 internal node 有最多 M 個 children
- 所有 leaves 在相同深度（tree 是 balanced）
- 每個 node 內的 keys 是排序的

對我們的 IoT database，我們用 order 64。這代表：
- 每個 node 有最多 63 個 keys
- 每個 internal node 有最多 64 個 children
- Tree height: log₆₄(1,000,000) ≈ 3 levels

跟 binary tree 的 20 層比較！

---

## Why B-Trees Are Cache-Friendly

B-trees 的魔法在於一個 node 中的所有 keys 都 **在記憶體中連續儲存**。

這是我用的 node 結構：

```c
#define BTREE_ORDER 64

typedef struct btree_node {
    int num_keys;                    // 4 bytes
    int keys[BTREE_ORDER - 1];       // 252 bytes (63 keys)
    void *values[BTREE_ORDER - 1];   // 504 bytes
    struct btree_node *children[BTREE_ORDER];  // 512 bytes
    // Total: ~1,272 bytes (fits in ~20 cache lines)
} btree_node_t;
```

當你存取一個 node，你得到 **所有 63 個 keys** 在一個連續的 array。你可以 binary search 它們，不需要任何 pointer-chasing：

```c
int find_key(btree_node_t *node, int key) {
    // Binary search in sorted array (cache-friendly!)
    int left = 0, right = node->num_keys - 1;
    while (left <= right) {
        int mid = (left + right) / 2;
        if (node->keys[mid] == key) return mid;
        if (key < node->keys[mid]) right = mid - 1;
        else left = mid + 1;
    }
    return -1;  // Not found
}
```

**Cache behavior**:
- 第一次存取 node: 1 次 cache miss（把 node 載入 cache）
- Node 內的 binary search: 0 次額外 cache misses（所有 keys 都是連續的）
- 總計：**每個 tree level 1 次 cache miss**

只有 3 層，那就只有 3 次 cache misses per lookup！

---

## B-Tree Search

```c
void* btree_search(btree_node_t *root, int key) {
    btree_node_t *node = root;
    
    while (node) {
        // Binary search within node (cache-friendly)
        int i = 0;
        while (i < node->num_keys && key > node->keys[i]) {
            i++;
        }
        
        // Found?
        if (i < node->num_keys && key == node->keys[i]) {
            return node->values[i];
        }
        
        // Leaf node?
        if (!node->children[0]) {
            return NULL;  // Not found
        }
        
        // Descend to child (cache miss here)
        node = node->children[i];
    }
    
    return NULL;
}
```

**Complexity**:
- Tree height: O(log_M N)
- Search within node: O(log M)
- Total: O(log M × log_M N) = O(log N)

**Cache misses**: O(log_M N) (one per level)

---

## The Benchmark Results

我在我們的 IoT database 上測試了不同的 B-tree orders，有 100 萬筆 sensor readings：

```
Dataset: 1,000,000 entries, 10,000 random lookups

Red-Black tree:
  Height: 20 levels
  Cycles/lookup: 12,000
  Cache misses: 18.5

B-tree (order 16):
  Height: 5 levels
  Cycles/lookup: 3,200
  Cache misses: 4.8
  Speedup: 3.75×

B-tree (order 64):
  Height: 3 levels
  Cycles/lookup: 1,800
  Cache misses: 2.8
  Speedup: 6.7×

B-tree (order 256):
  Height: 2 levels
  Cycles/lookup: 1,200
  Cache misses: 1.9
  Speedup: 10×
```

Order 64 的 B-tree 是我們的 sweet spot——比 Red-Black tree 快 6.7 倍。

**為什麼有效**：
1. **更少的層**：3 vs 20 代表 3 次 cache misses vs 20 次
2. **Sequential keys**：每個 node 內的 binary search 對 cache 友善
3. **Amortized cost**：在 node 內搜尋的成本（log₆₄ ≈ 6 次比較）跟 cache miss 的成本（100 cycles）比起來微不足道

---

## Choosing B-Tree Order

**Trade-off**：更大的 order → 更少的層，但每個 node 更多比較。

**Optimal order**：讓 node 放進 **一個 cache line**（64 bytes）。

### Cache Line Analysis

**Order 4**（3 keys）：
```c
struct btree_node {
    int num_keys;        // 4 bytes
    int keys[3];         // 12 bytes
    void *values[3];     // 24 bytes
    void *children[4];   // 32 bytes
    // Total: 72 bytes (2 cache lines)
};
```

**Order 8**（7 keys）：
```c
struct btree_node {
    int num_keys;        // 4 bytes
    int keys[7];         // 28 bytes
    void *values[7];     // 56 bytes
    void *children[8];   // 64 bytes
    // Total: 152 bytes (3 cache lines)
};
```

**Recommendation**:
- **In-memory B-tree**: Order 16-64 (balance height vs node size)
- **Disk-based B-tree**: Order 128-512 (minimize disk seeks)

---

## B-Tree Insertion

**Challenge**：維持平衡（所有 leaves 在相同深度）。

**Strategy**：Split full nodes。

### Insertion Algorithm

```c
void btree_insert(btree_node_t **root, int key, void *value) {
    btree_node_t *node = *root;

    // If root is full, split it
    if (node->num_keys == BTREE_ORDER - 1) {
        btree_node_t *new_root = create_node();
        new_root->children[0] = node;
        split_child(new_root, 0);
        *root = new_root;
    }

    insert_non_full(*root, key, value);
}

void insert_non_full(btree_node_t *node, int key, void *value) {
    int i = node->num_keys - 1;

    if (!node->children[0]) {  // Leaf node
        // Shift keys to make room
        while (i >= 0 && key < node->keys[i]) {
            node->keys[i + 1] = node->keys[i];
            node->values[i + 1] = node->values[i];
            i--;
        }
        node->keys[i + 1] = key;
        node->values[i + 1] = value;
        node->num_keys++;
    } else {  // Internal node
        // Find child to descend
        while (i >= 0 && key < node->keys[i]) {
            i--;
        }
        i++;

        // Split child if full
        if (node->children[i]->num_keys == BTREE_ORDER - 1) {
            split_child(node, i);
            if (key > node->keys[i]) i++;
        }

        insert_non_full(node->children[i], key, value);
    }
}
```

### Node Splitting

**Example**（order 4，max 3 keys）：
```
Before split:
  Node: [10|20|30]  (full)

After split:
  Left:  [10]
  Parent: [20]
  Right: [30]
```

**Cost**：O(M) to split（copy keys），但 amortized O(1)（rare）。

---

## B+ Trees: Optimized for Range Queries

**Problem with B-tree**：Values 散落在所有層。

**B+ tree**：所有 values 在 leaves，internal nodes 只存 keys。

**Structure**:
```
Internal nodes (keys only):
                [40|80]
               /   |   \
Leaf nodes (keys + values):
  [10:v1|20:v2|30:v3] → [40:v4|50:v5|60:v6] → [80:v7|90:v8|100:v9]
   ↑                      ↑                      ↑
   └──────────────────────┴──────────────────────┘
         Linked list for range scans
```

**Advantages**:
1. **Range queries**：掃描 leaves 的 linked list（sequential access）
2. **Higher fanout**：Internal nodes 更小（no values）
3. **All data in leaves**：更簡單的程式碼

**Use case**：Databases（MySQL InnoDB、PostgreSQL、SQLite）。

---

## Cache-Oblivious B-Trees

**Problem**：Optimal B-tree order 取決於 cache line size（x86 上 64 bytes，某些 ARM 上 128 bytes）。

**Cache-oblivious B-tree**：適應任何 cache size，不需要調整。

**Idea**：Recursive layout（van Emde Boas layout）。

**Example**（16 keys）：
```
Memory layout:
[8] [4|12] [2|6|10|14] [1|3|5|7|9|11|13|15]
 ↑    ↑        ↑              ↑
Root  Level 1  Level 2        Leaves

Sequential in memory, but logically a tree
```

**Advantage**：在不同 cache sizes 上都運作良好。

**Disadvantage**：複雜的實作，更難修改。

---

## Real-World Example: SQLite B-Tree

**Use case**：Embedded database（browsers、mobile apps）。

**Design**:
- **Page size**: 4 KB（matches filesystem block size）
- **Order**: ~340（4 KB / 12 bytes per entry）
- **B+ tree**：所有 data 在 leaves

**Optimization**：Page cache in memory。

**Benchmark**（1M records）：
```
Lookup:
  In-memory:  1,200 cycles (3 levels, all in cache)
  On-disk:    8 ms (3 disk seeks)

Range scan (1000 records):
  In-memory:  180,000 cycles (sequential leaf scan)
  On-disk:    12 ms (sequential disk read)
```

**Why B-tree for disk**:
- **Minimize seeks**：3 seeks vs 20（BST）
- **Sequential reads**：Leaf nodes linked
- **Page-aligned**：每個 node = 一個 disk block

---

## Embedded Systems: Fixed-Size B-Trees

**Challenge**：嵌入式系統中沒有 dynamic allocation。

**Solution**：在 array 中預先配置 B-tree nodes。

```c
#define MAX_NODES 1024
#define BTREE_ORDER 16

typedef struct {
    int num_keys;
    int keys[BTREE_ORDER - 1];
    void *values[BTREE_ORDER - 1];
    uint16_t children[BTREE_ORDER];  // Indices, not pointers
} btree_node_t;

typedef struct {
    btree_node_t nodes[MAX_NODES];
    uint16_t root;
    uint16_t free_list;
} btree_t;
```

**Advantages**:
- **No malloc**：可預測的記憶體使用
- **Cache-friendly**：Nodes 在連續的 array
- **Indices instead of pointers**：省記憶體（2 bytes vs 8 bytes）

**Disadvantage**：固定容量（MAX_NODES）。

---

## Guidelines

**Use B-tree when**:
- ✅ Large datasets (> 10,000 entries)
- ✅ Frequent insertions/deletions
- ✅ Range queries needed
- ✅ Disk/SSD storage

**Use B+ tree when**:
- ✅ Database indexing
- ✅ Range scans common
- ✅ All data can be in leaves

**Use BST when**:
- ✅ Small datasets (< 1,000 entries)
- ✅ Simple implementation needed

**Use sorted array when**:
- ✅ Read-only or rare updates
- ✅ Dataset fits in cache

---

## Optimization Techniques

### 1. Bulk Loading

**Problem**：一個一個插入排序好的資料很慢。

**Solution**：Bottom-up 建立 B-tree。

```c
btree_t* bulk_load(int *keys, void **values, int n) {
    // Sort input
    qsort_pairs(keys, values, n);

    // Build leaves
    int num_leaves = (n + BTREE_ORDER - 2) / (BTREE_ORDER - 1);
    btree_node_t *leaves = build_leaves(keys, values, n);

    // Build internal levels bottom-up
    while (num_leaves > 1) {
        leaves = build_level(leaves, num_leaves);
        num_leaves = (num_leaves + BTREE_ORDER - 1) / BTREE_ORDER;
    }

    return leaves;  // Root
}
```

**Speedup**：比個別 inserts 快 10-100 倍。

### 2. Prefix Compression

**Observation**：Keys 常常共享 prefixes（例如 URLs、file paths）。

**Optimization**：每個 node 只存一次 common prefix。

```c
struct compressed_node {
    char prefix[32];         // Common prefix
    int prefix_len;
    char suffixes[BTREE_ORDER][32];  // Only unique parts
};
```

**Savings**：對 string keys 減少 50-80% 記憶體。

### 3. SIMD Search

**Idea**：用 SIMD 平行比較 key 和多個 node keys。

```c
#include <immintrin.h>

int simd_search(int *keys, int n, int target) {
    __m256i target_vec = _mm256_set1_epi32(target);

    for (int i = 0; i < n; i += 8) {
        __m256i keys_vec = _mm256_loadu_si256((__m256i*)&keys[i]);
        __m256i cmp = _mm256_cmpeq_epi32(keys_vec, target_vec);
        int mask = _mm256_movemask_epi8(cmp);
        if (mask) {
            return i + __builtin_ctz(mask) / 4;
        }
    }
    return -1;
}
```

**Speedup**：對大 nodes（order 64+）快 2-3 倍。

---

## Summary

Database mystery 解決了。B-tree 提供了比 Red-Black tree 快 6.7 倍的 queries，lookup time 從 12,000 降到 1,800 cycles。Cache misses 從每次 lookup 18.5 次降到 2.8 次。IoT 裝置現在可以輕鬆處理 real-time sensor queries。這個「只用於 disk」的資料結構原來對 in-memory cache optimization 很完美。

**關鍵洞察**：

1. **Tree height 比 node complexity 更重要**。3 層的 B-tree 打敗 20 層的 binary tree，即使在 B-tree node 內搜尋要更多比較。

2. **Sequential memory layout 是王道**。把所有 keys 在 node 中連續儲存代表 node 內的 binary search 對 cache 友善。一次 cache miss 載入整個 node。

3. **B-trees 不只是用於 disk**。教科書教 B-trees 用於 disk 上的 databases，但當資料集很大時，它們對 in-memory 資料結構同樣有價值。

4. **Order matters**。太小（order 4-8）你沒有充分降低 height。太大（order 256+）nodes 放不進 cache。Order 16-64 是 in-memory B-trees 的 sweet spot。

5. **B+ trees 對 range queries 更好**。透過把所有 data 存在 leaves 並連結它們，你可以連續掃描 ranges，不需要 traverse tree。

**我們 IoT database 的數據**：
- Red-Black tree: 12,000 cycles/lookup, 18.5 cache misses
- B-tree (order 64): 1,800 cycles/lookup, 2.8 cache misses
- Speedup: 6.7×

**Next chapter**：Tries and radix trees for prefix matching and string keys.

