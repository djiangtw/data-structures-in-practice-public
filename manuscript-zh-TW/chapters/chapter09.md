# Chapter 9: Binary Search Trees

**Part III: Trees and Hierarchies**

---

> "Everyone knows that debugging is twice as hard as writing a program in the first place. So if you're as clever as you can be when you write it, how will you ever debug it?"
> — Brian Kernighan

## The Red-Black Tree Disaster

Compiler 花了 60% 的時間在查找 symbols。不是 parsing，不是 code generation——只是 symbol table lookups。

對於一個典型的嵌入式程式，有 10,000 個 symbols，這是無法接受的。Symbol table 儲存變數名稱、函數名稱和型別定義。實作使用了 Red-Black tree——一種 self-balancing binary search tree。

「這是 O(log n)，」我同事說。「教科書上說這是完美的 use case。」

Profiler 告訴我不同的故事：

```bash
$ perf stat -e cache-misses,instructions ./compiler test.c
  Performance counter stats:
    2,847,234 cache-misses
    8,500,000 instructions
```

280 萬次 cache misses，850 萬條 instructions？這是每 3 條 instructions 就有一次 cache miss！

我試了一個看起來很瘋狂的做法：我把 Red-Black tree 換成 sorted array 加上 binary search。Binary search 也是 O(log n)，理論上應該一樣快。

**結果：Compiler 現在快了 3 倍。**

兩個 O(log n) 的演算法，怎麼會有這麼大的性能差異？

## The Investigation

我用 perf 跑了兩個實作，看看發生了什麼事：

```bash
# Red-Black tree version
$ perf stat -e cache-references,cache-misses,cycles ./compiler_rbtree test.c
  Performance counter stats:
    3,247,832  cache-references
    2,847,234  cache-misses  (87.7% miss rate)
   24,000,000  cycles

# Sorted array version
$ perf stat -e cache-references,cache-misses,cycles ./compiler_array test.c
  Performance counter stats:
    1,123,456  cache-references
      234,567  cache-misses  (20.9% miss rate)
    8,000,000  cycles
```

就是這個：**87.7% cache miss rate** 對比 **20.9%**。

每次 cache miss 在這個 RISC-V 系統上要花大約 100 cycles。Red-Black tree 大部分時間都在等記憶體。

## The Textbook Story

每個資料結構課程都教 binary search trees。推銷詞很吸引人：

**Binary Search Tree (BST)**:
- Insert: O(log n)
- Search: O(log n)
- Delete: O(log n)
- In-order traversal 給你排序好的順序

**Balanced trees**（AVL、Red-Black）保證即使是 adversarial input 也是 O(log n) height。

教科書的結論：「對於有頻繁 insertions 和 lookups 的動態資料集，使用 balanced BSTs。」

聽起來很適合 symbol table，對吧？

## The Reality Check

教科書不會告訴你的是：**Binary search trees 是 pointer-chasing 的惡夢。**

每次 tree traversal 都跳到一個隨機的記憶體位置。每次跳躍都很可能是 cache miss。

---

## Why Binary Search Trees Are Slow

問題在於 **memory layout**。

### Sorted Array: Sequential Memory

當你配置一個 array，所有元素在記憶體中是連續的：

```
Memory: [10][20][30][40][50][60][70][80]
         ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑
      0x1000 ...sequential, cache-friendly...
```

當你存取 `array[4]`，CPU 抓取一個 64-byte cache line，包含 `array[4]`、`array[5]`、`array[6]` 等等。如果你接下來存取 `array[5]`，它已經在 cache 裡了。

### Binary Search Tree: Scattered Memory

當你插入 nodes 到 BST，每個 node 都是由 `malloc()` 分別配置的。它們最後散落在 heap 各處：

```
       40 (@ 0x5000)
      /  \
    20    60 (@ 0x2000, @ 0x8000)
   /  \   /  \
  10  30 50  70 (@ 0x1000, @ 0x3000, @ 0x6000, @ 0x9000)
```

每個 node 都在不同的記憶體位置。跟著 pointer 走就是跳到隨機的 address。

### Cache Behavior: A Concrete Example

讓我們在兩個結構中搜尋值 70。

**Sorted array (binary search)**:
```
Step 1: Check middle element [40] @ 0x1020
  → Cache MISS (100 cycles)
  → CPU fetches cache line containing [30][40][50][60]

Step 2: Check [60] @ 0x1030
  → Cache HIT (1 cycle) — already in the cache line!

Step 3: Check [70] @ 0x1038
  → Cache HIT (1 cycle) — still in cache

Total: ~102 cycles, 1 cache miss
```

**Binary search tree**:
```
Step 1: Check root [40] @ 0x5000
  → Cache MISS (100 cycles)
  → Fetches cache line at 0x5000

Step 2: Go right, check [60] @ 0x8000
  → Cache MISS (100 cycles) — different memory location!

Step 3: Go right, check [70] @ 0x9000
  → Cache MISS (100 cycles) — yet another location!

Total: ~300 cycles, 3 cache misses
```

兩個演算法做了相同次數的比較（3 次）。但 BST 慢了 **3 倍**，因為 cache misses。

這就是為什麼我的 compiler 的 symbol table 這麼慢。每次 symbol lookup 都在追著 pointers 穿過散落的記憶體。

---

## The Benchmark

讓我展示我測試的實際程式碼。這是一個簡單的 BST 實作：

```c
// 計算 binary search tree node
typedef struct bst_node {
    int key;
    void *value;
    struct bst_node *left;
    struct bst_node *right;
} bst_node_t;

void* bst_search(bst_node_t *root, int key) {
    while (root) {
        if (key == root->key) return root->value;
        root = (key < root->key) ? root->left : root->right;
    }
    return NULL;
}
```

這是 sorted array 版本：

```c
typedef struct {
    int key;
    void *value;
} array_entry_t;

void* array_search(array_entry_t *arr, int n, int key) {
    int left = 0, right = n - 1;
    while (left <= right) {
        int mid = (left + right) / 2;
        if (arr[mid].key == key) return arr[mid].value;
        if (key < arr[mid].key) right = mid - 1;
        else left = mid + 1;
    }
    return NULL;
}
```

我在不同大小的資料集上跑了 10,000 次隨機 lookups：

```
Dataset: 1,000 entries
  BST:           2,400 cycles/lookup
  Sorted array:    800 cycles/lookup
  Speedup: 3.0×

Dataset: 10,000 entries
  BST:           3,200 cycles/lookup
  Sorted array:  1,100 cycles/lookup
  Speedup: 2.9×

Cache misses (perf stat):
  BST:           8.5 misses/lookup
  Sorted array:  2.1 misses/lookup
```

Sorted array 一致地快了 3 倍，即使兩者都是 O(log n)。

**為什麼 sorted array 贏**：

1. **Sequential layout**: Binary search 存取的是附近的元素，很可能在同一個 cache line
2. **Cache line reuse**: 每次 cache miss 載入 8 個 entries（64-byte cache line ÷ 8-byte entry）
3. **Prefetcher helps**: 硬體 prefetcher 可以偵測 stride pattern 並提前抓取

BST 沒有這些優勢。每次 pointer dereference 都是賭博。

---

## Memory Overhead

**BST node**（64-bit 系統）：
```c
struct bst_node {
    int key;           // 4 bytes
    void *value;       // 8 bytes
    struct bst_node *left;   // 8 bytes
    struct bst_node *right;  // 8 bytes
    // Padding: 4 bytes
};
// Total: 32 bytes per entry
```

**Sorted array entry**:
```c
struct array_entry {
    int key;     // 4 bytes
    void *value; // 8 bytes
    // Padding: 4 bytes (for alignment)
};
// Total: 16 bytes per entry
```

**Memory usage**（1,000 entries）：
- BST: 32 KB (32 bytes × 1,000)
- Array: 16 KB (16 bytes × 1,000)

**BST 用了 2 倍的記憶體**，而那些 pointers 還傷害了 cache performance。

---

## But Wait—What About Balanced Trees?

你可能在想：「當然，基本的 BST 如果你插入排序好的資料會退化成 linked list。但 balanced trees 像 AVL 或 Red-Black trees 呢？那些保證 O(log n) height！」

這就是我同事在我建議把他的 Red-Black tree 換成 sorted array 時的論點。

他說得對，balanced trees 解決了 worst-case 問題。如果你把 keys 按排序順序插入基本的 BST，你會得到一個 O(n) height 的 linked list。Balanced trees 防止這個。

但 balanced trees 沒有修正 cache 問題。它們還是在追著 pointers 穿過散落的記憶體。

### Red-Black Trees

我們 compiler 中的 Red-Black tree 維護這些 invariants：
- 每個 node 不是 red 就是 black
- Root 是 black
- Red nodes 有 black children
- 從 root 到 leaves 的所有路徑有相同數量的 black nodes

這些規則保證 tree height 最多是 2×log₂(n)。

當你 insert 或 delete，tree 執行 **rotations** 來維持平衡：

```
Right rotation:
    y              x
   / \            / \
  x   C    →     A   y
 / \                / \
A   B              B   C
```

Rotations 只是 pointer 更新——在 CPU 操作上很便宜。但它們沒有改變根本問題：**每個 node 還是在隨機的記憶體位置**。

### The Cache Problem Remains

這是我用 Red-Black tree 測量到的：

```bash
$ perf stat -e cache-misses,L1-dcache-load-misses ./compiler_rbtree test.c
  Performance counter stats:
    2,847,234 cache-misses
    2,654,123 L1-dcache-load-misses
```

幾乎每次 tree traversal 都是 cache miss。Tree 是平衡的，但還是很慢。

Balanced trees 解決了 **演算法** 的 worst case。它們沒有解決 **硬體** 的 worst case。

---

## So When Should You Use BSTs?

在我把 compiler 中的 Red-Black tree 換成 sorted array 之後，有人問我：「BSTs 有適合的使用時機嗎？」

有。但 use cases 比教科書說的更具體。

### 1. When You Have Frequent Insertions and Deletions

Sorted array 對我們 compiler 的 symbol table 很完美，因為 symbols 在編譯期間大多是 **read-only** 的。你在函數開頭定義變數，然後重複查找它們。

但如果你不斷地 inserting 和 deleting 呢？

用 sorted array：
- Insert: O(n) — 必須把所有元素往右移
- Delete: O(n) — 必須把所有元素往左移

用 BST：
- Insert: O(log n) — 只要更新幾個 pointers
- Delete: O(log n) — 只要更新幾個 pointers

我用 1,000 次隨機 insert/delete 操作的 workload 測試：

```
Sorted array:   12,000 cycles/operation (shifting overhead)
Red-Black tree:  3,500 cycles/operation
Speedup: 3.4× for BST
```

如果你的 workload 是 insert/delete-heavy，BSTs 贏，即使有 cache misses。

### 2. When You Need Range Queries

BSTs 有個好特性：in-order traversal 按排序順序訪問 keys。

```c
void inorder(bst_node_t *node, void (*visit)(int key)) {
    if (!node) return;
    inorder(node->left, visit);
    visit(node->key);
    inorder(node->right, visit);
}
```

這讓 range queries 很有效率。如果你要「所有 100 到 200 之間的 keys」，你可以跳過整個在範圍外的 subtrees。

用 sorted array，你會 binary search 找到 100，然後線性掃描到 200。如果範圍很大，這會比較慢。

### 3. When the Dataset Is Small

對於小資料集（< 100 entries），cache miss penalty 沒那麼嚴重：

```
Dataset size: 50 entries
  BST:          180 cycles/lookup
  Sorted array: 150 cycles/lookup
  Difference: Only 20% (not 3×)
```

只有 50 個 entries，很多 BST nodes 都能放進 cache。Pointer-chasing 問題沒那麼嚴重。

對於小資料集，用最簡單實作的就好。性能差異不會有影響。

---

## Optimization: Cache-Conscious BST

### Implicit Binary Tree (Array-Based)

**想法**：用 index 算術把 tree 存在 array 中（像 binary heap）。

**Layout**:
```
Index:  0   1   2   3   4   5   6
Array: [40][20][60][10][30][50][70]

Tree structure:
       40 (index 0)
      /  \
    20    60 (index 1, 2)
   /  \   /  \
  10  30 50  70 (index 3, 4, 5, 6)

Parent of i: (i-1)/2
Left child:  2*i + 1
Right child: 2*i + 2
```

**Advantages**:
- Sequential memory layout
- No pointers (saves 16 bytes per node)
- Cache-friendly

**Disadvantages**:
- Must be complete tree (wastes space if unbalanced)
- Insert/delete requires array shifting

**When to use**: Static datasets (build once, query many times).

### B-Tree (Preview)

**Better solution**: Multi-way trees (Chapter 10)
- Store multiple keys per node
- Each node fits in cache line
- Reduces tree height and cache misses

---

## Real-World Example: Linux Kernel Red-Black Trees

在我把 compiler 的 Red-Black tree 換成 sorted array 之後，同事問：「但 Linux kernel 到處都用 Red-Black trees。他們錯了嗎？」

不——他們用對了工具來處理他們的 workload。

Linux kernel 用 Red-Black trees 來處理：
- **Process scheduler**: 追蹤 runnable processes
- **Virtual memory areas**: 管理 memory regions
- **Timers**: 排程未來的 events

這些都是 **write-heavy** workloads，有 **頻繁的 insertions 和 deletions**。Processes 不斷被建立和銷毀。Memory regions 被配置和釋放。Timers 被加入和移除。

這是 kernel 的 Red-Black tree node（從 `lib/rbtree.c`）：

```c
struct rb_node {
    unsigned long  __rb_parent_color;  // Parent pointer + color bit
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```

注意這個優化：parent pointer 和 color bit 合併在一個 field。因為 pointers 對齊到 8 bytes，低 3 bits 總是 zero。Kernel 用其中一個 bit 來存 red/black color。這省了每個 node 8 bytes。

我 benchmark 了一個類似 scheduler 的 workload（10,000 次 insert/delete/search 操作）：

```
Red-Black tree:  2.1 µs
Sorted array:    8.5 µs (too slow for scheduler)
```

對這個 workload，Red-Black tree 快了 4 倍，因為 insertions 和 deletions 佔主導。

**教訓**：根據你的 **workload** 選擇資料結構，不只是理論上的 lookup complexity。

---

## Guidelines

**Use sorted array when**:
- ✅ Mostly lookups (read-heavy)
- ✅ Dataset fits in cache (< 10,000 entries)
- ✅ Infrequent updates

**Use BST (Red-Black/AVL) when**:
- ✅ Frequent insertions/deletions
- ✅ Range queries needed
- ✅ Dataset too large for array shifting

**Use B-tree when**:
- ✅ Large datasets (> 10,000 entries)
- ✅ Cache efficiency critical
- ✅ Disk/SSD storage (Chapter 10)

**Avoid BST when**:
- ❌ Pure lookup workload
- ❌ Small dataset (< 100 entries) → use linear search
- ❌ Need predictable performance → use hash table

---

## Summary

Red-Black tree disaster 用一個簡單的 sorted array 就修好了。Compiler 快了 3 倍，從 60% 時間在 symbol lookups 降到 20%。Cache miss rate 從 87.7% 降到 20.9%。但這不代表 BSTs 總是錯的——這代表 workload 很重要。

**關鍵洞察**：

1. **Binary search trees 對 cache 不友善**。每次 pointer dereference 都很可能是 cache miss。對於 lookup-heavy workloads，sorted arrays 通常快 3 倍，即使有相同的 O(log n) complexity。

2. **Memory matters**。BSTs 用了 2 倍的記憶體（64-bit 系統上每個 entry 32 bytes vs 16 bytes）。那些額外的 pointers 傷害了 cache utilization 和 memory bandwidth。

3. **BSTs 在 write-heavy workloads 贏**。如果你不斷 inserting 和 deleting，BSTs 快 3-4 倍，因為它們避免了 shifting elements。

4. **Balanced trees 沒有修正 cache 問題**。Red-Black trees 和 AVL trees 保證 O(log n) height，但它們還是在追著 pointers 穿過散落的記憶體。

5. **Workload 決定正確的選擇**。我們 compiler 的 symbol table 是 read-heavy（lookups 佔主導），所以 sorted arrays 贏。Linux scheduler 是 write-heavy（不斷 insert/delete），所以 Red-Black trees 贏。

**我們 compiler 的數據**：
- Red-Black tree: 2,400 cycles/lookup, 87.7% cache miss rate
- Sorted array: 800 cycles/lookup, 20.9% cache miss rate
- Speedup: 3×

**Write-heavy workload 的數據**：
- Red-Black tree: 3,500 cycles/operation
- Sorted array: 12,000 cycles/operation
- Speedup: 3.4× for BST

**Next chapter**: B-trees pack multiple keys per node to reduce tree height and cache misses.

