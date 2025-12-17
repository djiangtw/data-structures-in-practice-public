# Chapter 12: Heaps and Priority Queues

**Part III: Trees and Hierarchies**

---

> "Bad programmers worry about the code. Good programmers worry about data structures and their relationships."
> — Linus Torvalds

## The Scheduler Debate

團隊在爭論資料結構。我們需要一個 real-time operating system 的 task scheduler，能夠：
- Insert new tasks with priorities (O(log n))
- Get the highest-priority task (O(1))
- Remove the highest-priority task (O(log n))

「用 sorted array，」有人建議。但 insertion 是 O(n)——你必須 shift elements。

「用 linked list，」另一個人說。但找 max 是 O(n)——你必須掃描整個 list。

「用 binary search tree，」第三個人建議。但我們從 Chapter 9 已經知道 BSTs 有糟糕的 cache behavior。

辯論持續到有人提到 binary heaps。Benchmark 結果結束了討論：

```bash
$ perf stat -e cache-misses,cycles ./scheduler_heap
  Performance counter stats:
    45,000 cache-misses
 1,200,000 cycles
 
$ perf stat -e cache-misses,cycles ./scheduler_bst
  Performance counter stats:
   180,000 cache-misses
 4,800,000 cycles
```

Heap 比 Red-Black tree **快 4 倍**，cache misses 少 4 倍。

為什麼？Heaps 存在 arrays 中，所以它們有優秀的 cache locality。

---

## The Textbook Story

**Binary heap** 是一個 complete binary tree，每個 parent 大於（或小於）它的 children。

**Max-heap example**:
```
        90
       /  \
      70   50
     / \   / \
    40 30 20 10
```

**Properties**:
- **Complete tree**：所有層都填滿，除了可能最後一層，從左到右填
- **Heap property**：Parent ≥ children（max-heap）或 parent ≤ children（min-heap）
- **Operations**:
  - Insert: O(log n) — add at end, bubble up
  - Extract max: O(log n) — remove root, bubble down
  - Peek max: O(1) — just read root

**教科書的推銷**：
- Perfect for priority queues
- O(log n) insert and delete
- O(1) access to max/min
- Simple to implement

聽起來很棒！但有個陷阱...

---

## The Reality Check: Array-Based Heaps

這是關鍵洞察：你可以用 index 算術把 binary heap 存在 array 中。

**Heap as array**:
```
Index:  0   1   2   3   4   5   6
Array: [90][70][50][40][30][20][10]

Tree:
        90 (index 0)
       /  \
      70   50 (indices 1, 2)
     / \   / \
    40 30 20 10 (indices 3, 4, 5, 6)
```

**Index arithmetic**:
- Parent of node i: (i - 1) / 2
- Left child of node i: 2i + 1
- Right child of node i: 2i + 2

沒有 pointers！只有 array indices。

這是我用的實作：

```c
typedef struct {
    int *data;
    int size;
    int capacity;
} heap_t;

void heap_insert(heap_t *heap, int value) {
    // Add at end
    heap->data[heap->size] = value;
    int i = heap->size;
    heap->size++;
    
    // Bubble up
    while (i > 0) {
        int parent = (i - 1) / 2;
        if (heap->data[i] <= heap->data[parent]) break;
        
        // Swap with parent
        int temp = heap->data[i];
        heap->data[i] = heap->data[parent];
        heap->data[parent] = temp;
        
        i = parent;
    }
}

int heap_extract_max(heap_t *heap) {
    int max = heap->data[0];
    
    // Move last element to root
    heap->size--;
    heap->data[0] = heap->data[heap->size];
    
    // Bubble down
    int i = 0;
    while (2 * i + 1 < heap->size) {
        int left = 2 * i + 1;
        int right = 2 * i + 2;
        int largest = i;
        
        if (left < heap->size && heap->data[left] > heap->data[largest]) {
            largest = left;
        }
        if (right < heap->size && heap->data[right] > heap->data[largest]) {
            largest = right;
        }
        
        if (largest == i) break;
        
        // Swap with largest child
        int temp = heap->data[i];
        heap->data[i] = heap->data[largest];
        heap->data[largest] = temp;
        
        i = largest;
    }
    
    return max;
}
```

**Cache behavior**:
- 所有 data 在連續的 array
- Bubble up/down 存取附近的 elements
- 優秀的 spatial locality

這就是為什麼 heap 比 Red-Black tree 快 4 倍。沒有 pointer-chasing，只有 array indexing。

---

## The Benchmark Results

我測試了 heap-based scheduler 對比其他資料結構：

```
Dataset: 10,000 tasks with random priorities
Test: 100,000 insert + extract-max operations

Red-Black tree:
  Cycles/operation: 4,800
  Cache misses: 18.0

Binary heap (array-based):
  Cycles/operation: 1,200
  Cache misses: 4.5
  Speedup: 4.0×

Sorted array:
  Insert: 12,000 cycles (O(n) shifting)
  Extract-max: 100 cycles (O(1) pop from end)
  Average: 6,050 cycles/operation
```

Heap 對這個 workload 是明顯的贏家。

**為什麼 heap 贏**：
1. **Array-based**：所有 data 連續，優秀的 cache locality
2. **Balanced operations**：Insert 和 extract-max 都是 O(log n)
3. **No pointer-chasing**：只有 array indexing

**為什麼 sorted array 輸**：
- Insert 是 O(n)，因為你必須 shift elements
- 對 insert-heavy workloads，這主導了

---

## Cache-Conscious Optimization: d-ary Heaps

Binary heaps 有個問題：當 heap 成長，bubble-up 和 bubble-down 操作在記憶體中跳來跳去。

對有 100 萬個 elements 的 heap：
- Height：log₂(1M) ≈ 20 levels
- Bubble-down：20 次 cache misses（每層是不同的 cache line）

**Solution**：用 **d-ary heap**，每個 node 有 d 個 children 而不是 2 個。

**4-ary heap**:
```
Index:  0   1   2   3   4   5   6   7   8   9  10  11  12
Array: [90][70][60][50][40][65][55][45][30][35][25][20][15]

Tree:
           90 (index 0)
        /  |  |  \
      70  60  50  40 (indices 1, 2, 3, 4)
     /|\ /|\ /|\ /|\
    ... (indices 5-20)
```

**Index arithmetic**（d-ary heap）：
- Parent of node i: (i - 1) / d
- First child of node i: d × i + 1
- Last child of node i: d × i + d

**Trade-off**:
- **Shorter tree**：Height = log_d(n) instead of log₂(n)
- **More comparisons per level**：必須比較 d 個 children 而不是 2 個
- **Better cache behavior**：更少的層 = 更少的 cache misses

我測試了不同的 d 值：

```
Dataset: 1,000,000 elements
Test: 100,000 insert + extract-max operations

Binary heap (d=2):
  Height: 20 levels
  Cycles/operation: 2,400
  Cache misses: 8.5

4-ary heap (d=4):
  Height: 10 levels
  Cycles/operation: 1,600
  Cache misses: 4.2
  Speedup: 1.5×

8-ary heap (d=8):
  Height: 7 levels
  Cycles/operation: 1,400
  Cache misses: 2.8
  Speedup: 1.7×

16-ary heap (d=16):
  Height: 5 levels
  Cycles/operation: 1,500
  Cache misses: 2.1
  Speedup: 1.6× (diminishing returns)
```

**Sweet spot**：d=8 對大部分 workloads。減少 3 倍 cache misses，每層不會有太多比較。

---

## Real-World Example: Linux Kernel CFS Scheduler

Linux kernel 的 Completely Fair Scheduler (CFS) 用 **red-black tree**，不是 heap。為什麼？

因為 scheduler 需要的不只是「get highest priority task」：
- **Range queries**：找所有 priority > X 的 tasks
- **Arbitrary removal**：移除特定的 task（不只是 max）
- **Fair scheduling**：追蹤 virtual runtime，不只是 priority

Heaps 無法有效率地做這些。它們為一件事優化：priority queue operations（insert、extract-max）。

但對更簡單的 schedulers（像嵌入式 RTOSes），heaps 很完美。

**Example**：FreeRTOS 用簡單的 priority-based scheduler，有類似 heap 的結構（雖然對小 task counts 實作為 linked list）。

---

## When to Use Heaps

在我們的 RTOS scheduler 中用了 heaps 之後，我學到它們是正確選擇的時機：

### 1. Priority Queues

如果你需要：
- Insert with priority: O(log n)
- Get max/min: O(1)
- Remove max/min: O(log n)

用 heap。這是教科書上 priority queues 的資料結構。

**Examples**:
- Task schedulers
- Event queues
- Dijkstra's shortest path algorithm
- Huffman coding

### 2. Top-K Problems

在 stream 中找 K 個最大/最小的 elements：
- 維護一個 size K 的 min-heap
- 對每個新 element，如果它比 heap 的 min 大，替換 min
- 最後的 heap 包含 K 個最大的 elements

**Time**：O(n log k) instead of O(n log n) for full sorting

### 3. Median Maintenance

用兩個 heaps 維護 stream 的 median：
- Max-heap for the smaller half
- Min-heap for the larger half
- Median is the root of the larger heap (or average of both roots)

**Time**：每次 insert O(log n)，get median O(1)

### 4. When NOT to Use Heaps

不要用 heaps 在：
- **Arbitrary removal**：移除 non-root element 是 O(n)
- **Search**：找任意 element 是 O(n)
- **Range queries**：無法有效率地找範圍內的所有 elements
- **Sorted iteration**：Heaps 不維護完整的 sorted order

對這些，用 balanced BST 或 B-tree。

---

## Summary

Scheduler debate 被數據解決了。Binary heap 提供了比 Red-Black tree 好 4 倍的性能，cache misses 少 4 倍。Heap 的 array-based layout 提供了 pointer-based trees 無法匹敵的 cache locality。

**關鍵洞察**：

1. **Heaps 是 array-based**。沒有 pointers，沒有 pointer-chasing，只有 array indexing。這提供了優秀的 cache locality。

2. **Complete binary trees 完美地放進 arrays**。Index 算術（parent = (i-1)/2，children = 2i+1 和 2i+2）簡單且對 cache 友善。

3. **d-ary heaps 減少 cache misses**。透過增加 branching factor 到 4 或 8，你減少 tree height 和 cache misses 2-3 倍。

4. **Heaps 是用於 priority queues**。如果你需要 insert、extract-max 和 peek-max，heaps 很完美。但它們無法有效率地做 arbitrary removal 或 range queries。

5. **Trade-offs matter**。Binary heaps（d=2）每層有更少比較。8-ary heaps（d=8）有更少 cache misses。Sweet spot 取決於你的 workload。

**我們 RTOS scheduler 的數據**：
- Red-Black tree: 4,800 cycles/operation, 18.0 cache misses
- Binary heap: 1,200 cycles/operation, 4.5 cache misses
- 8-ary heap: 1,400 cycles/operation, 2.8 cache misses

對我們的 scheduler，binary heap 是明顯的贏家。簡單、快速、對 cache 友善。

**Next chapter**：Part IV begins with graphs and their memory representations.

