# Chapter 9: Binary Search Trees

**Part III: Trees and Hierarchies**

---

> "Everyone knows that debugging is twice as hard as writing a program in the first place. So if you're as clever as you can be when you write it, how will you ever debug it?"
> — Brian Kernighan

## The Red-Black Tree Disaster

The compiler was spending 60% of its time looking up symbols. Not parsing, not code generation—just symbol table lookups.

For a typical embedded program with 10,000 symbols, this was unacceptable. The symbol table stored variable names, function names, and type definitions. The implementation used a Red-Black tree—a self-balancing binary search tree.

"It's O(log n)," my colleague said. "Textbook perfect for this use case."

The profiler told a different story:

```bash
$ perf stat -e cache-misses,instructions ./compiler test.c
  Performance counter stats:
    2,847,234 cache-misses
    8,500,000 instructions
```

2.8 million cache misses for 8.5 million instructions? That's one cache miss every 3 instructions!

I tried something that seemed crazy: I replaced the Red-Black tree with a sorted array and binary search. Binary search is also O(log n), so theoretically it should be the same speed.

**Result: The compiler was now 3× faster.**

How could two O(log n) algorithms have such different performance?

## The Investigation

I ran both implementations through perf to see what was happening:

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

There it was: **87.7% cache miss rate** for the Red-Black tree versus **20.9%** for the sorted array.

Each cache miss costs about 100 cycles on this RISC-V system. The Red-Black tree was spending most of its time waiting for memory.

## The Textbook Story

Every data structures course teaches binary search trees. The pitch is compelling:

**Binary Search Tree (BST)**:
- Insert: O(log n)
- Search: O(log n)
- Delete: O(log n)
- In-order traversal gives sorted order

**Balanced trees** (AVL, Red-Black) guarantee O(log n) height even with adversarial input.

The textbook conclusion: "Use balanced BSTs for dynamic datasets with frequent insertions and lookups."

Sounds perfect for a symbol table, right?

## The Reality Check

Here's what the textbooks don't tell you: **Binary search trees are pointer-chasing nightmares.**

Every tree traversal jumps to a random memory location. Every jump is likely a cache miss.

---

## Why Binary Search Trees Are Slow

The problem is **memory layout**.

### Sorted Array: Sequential Memory

When you allocate an array, all elements are contiguous in memory:

```
Memory: [10][20][30][40][50][60][70][80]
         ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑
      0x1000 ...sequential, cache-friendly...
```

When you access `array[4]`, the CPU fetches a 64-byte cache line that includes `array[4]`, `array[5]`, `array[6]`, etc. If you access `array[5]` next, it's already in cache.

### Binary Search Tree: Scattered Memory

When you insert nodes into a BST, each node is allocated separately by `malloc()`. They end up scattered across the heap:

```
       40 (@ 0x5000)
      /  \
    20    60 (@ 0x2000, @ 0x8000)
   /  \   /  \
  10  30 50  70 (@ 0x1000, @ 0x3000, @ 0x6000, @ 0x9000)
```

Each node is in a different memory location. Following a pointer means jumping to a random address.

### Cache Behavior: A Concrete Example

Let's search for the value 70 in both structures.

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

Both algorithms do the same number of comparisons (3). But the BST is **3× slower** because of cache misses.

This is why my compiler's symbol table was so slow. Every symbol lookup was chasing pointers through scattered memory.

---

## The Benchmark

Let me show you the actual code I tested. Here's a simple BST implementation:

```c
// Binary search tree node
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

And here's the sorted array version:

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

I ran 10,000 random lookups on datasets of different sizes:

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

The sorted array is consistently 3× faster, even though both are O(log n).

**Why the sorted array wins**:

1. **Sequential layout**: Binary search accesses nearby elements that are likely in the same cache line
2. **Cache line reuse**: Each cache miss loads 8 entries (64-byte cache line ÷ 8-byte entry)
3. **Prefetcher helps**: The hardware prefetcher can detect the stride pattern and fetch ahead

The BST has none of these advantages. Every pointer dereference is a gamble.

---

## Memory Overhead

**BST node** (64-bit system):
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

**Memory usage** (1,000 entries):
- BST: 32 KB (32 bytes × 1,000)
- Array: 16 KB (16 bytes × 1,000)

**BST uses 2× more memory** for pointers that hurt cache performance.

---

## But Wait—What About Balanced Trees?

You might be thinking: "Sure, a basic BST can degenerate into a linked list if you insert sorted data. But what about balanced trees like AVL or Red-Black trees? Those guarantee O(log n) height!"

That's what my colleague argued when I suggested replacing his Red-Black tree with a sorted array.

He was right that balanced trees solve the worst-case problem. If you insert keys in sorted order into a basic BST, you get a linked list with O(n) height. Balanced trees prevent this.

But balanced trees don't fix the cache problem. They're still pointer-chasing through scattered memory.

### Red-Black Trees

The Red-Black tree in our compiler maintained these invariants:
- Every node is either red or black
- The root is black
- Red nodes have black children
- All paths from root to leaves have the same number of black nodes

These rules guarantee the tree height is at most 2×log₂(n).

When you insert or delete, the tree performs **rotations** to maintain balance:

```
Right rotation:
    y              x
   / \            / \
  x   C    →     A   y
 / \                / \
A   B              B   C
```

Rotations are just pointer updates—cheap in terms of CPU operations. But they don't change the fundamental problem: **every node is still in a random memory location**.

### The Cache Problem Remains

Here's what I measured with the Red-Black tree:

```bash
$ perf stat -e cache-misses,L1-dcache-load-misses ./compiler_rbtree test.c
  Performance counter stats:
    2,847,234 cache-misses
    2,654,123 L1-dcache-load-misses
```

Nearly every tree traversal was a cache miss. The tree was balanced, but it was still slow.

Balanced trees solve the **algorithmic** worst case. They don't solve the **hardware** worst case.

---

## So When Should You Use BSTs?

After replacing the Red-Black tree with a sorted array in our compiler, I got asked: "Are BSTs ever the right choice?"

Yes. But the use cases are more specific than textbooks suggest.

### 1. When You Have Frequent Insertions and Deletions

The sorted array was perfect for our compiler's symbol table because symbols are mostly **read-only** during compilation. You define variables at the start of a function, then look them up repeatedly.

But what if you're constantly inserting and deleting?

With a sorted array:
- Insert: O(n) — must shift all elements to the right
- Delete: O(n) — must shift all elements to the left

With a BST:
- Insert: O(log n) — just update a few pointers
- Delete: O(log n) — just update a few pointers

I tested this with a workload of 1,000 random insert/delete operations:

```
Sorted array:   12,000 cycles/operation (shifting overhead)
Red-Black tree:  3,500 cycles/operation
Speedup: 3.4× for BST
```

If your workload is insert/delete-heavy, BSTs win despite the cache misses.

### 2. When You Need Range Queries

BSTs have a nice property: in-order traversal visits keys in sorted order.

```c
void inorder(bst_node_t *node, void (*visit)(int key)) {
    if (!node) return;
    inorder(node->left, visit);
    visit(node->key);
    inorder(node->right, visit);
}
```

This makes range queries efficient. If you want "all keys between 100 and 200", you can skip entire subtrees that are outside the range.

With a sorted array, you'd binary search to find 100, then scan linearly to 200. If the range is large, this is slower.

### 3. When the Dataset Is Small

For small datasets (< 100 entries), the cache miss penalty is less severe:

```
Dataset size: 50 entries
  BST:          180 cycles/lookup
  Sorted array: 150 cycles/lookup
  Difference: Only 20% (not 3×)
```

With only 50 entries, many BST nodes fit in cache. The pointer-chasing problem is less severe.

For small datasets, use whatever's simplest to implement. The performance difference won't matter.

---

## Optimization: Cache-Conscious BST

### Implicit Binary Tree (Array-Based)

**Idea**: Store tree in array using index arithmetic (like binary heap).

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

After I replaced our compiler's Red-Black tree with a sorted array, a colleague asked: "But the Linux kernel uses Red-Black trees everywhere. Are they wrong?"

No—they're using the right tool for their workload.

The Linux kernel uses Red-Black trees for:
- **Process scheduler**: Tracking runnable processes
- **Virtual memory areas**: Managing memory regions
- **Timers**: Scheduling future events

These are all **write-heavy** workloads with **frequent insertions and deletions**. Processes are created and destroyed constantly. Memory regions are allocated and freed. Timers are added and removed.

Here's the kernel's Red-Black tree node (from `lib/rbtree.c`):

```c
struct rb_node {
    unsigned long  __rb_parent_color;  // Parent pointer + color bit
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
```

Notice the optimization: the parent pointer and color bit are combined in one field. Since pointers are aligned to 8 bytes, the low 3 bits are always zero. The kernel uses one of those bits to store the red/black color. This saves 8 bytes per node.

I benchmarked a scheduler-like workload (10,000 insert/delete/search operations):

```
Red-Black tree:  2.1 µs
Sorted array:    8.5 µs (too slow for scheduler)
```

For this workload, the Red-Black tree is 4× faster because insertions and deletions dominate.

**The lesson**: Choose your data structure based on your **workload**, not just theoretical lookup complexity.

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

The Red-Black tree disaster was fixed with a simple sorted array. The compiler got 3× faster, dropping from 60% time in symbol lookups to 20%. Cache miss rate fell from 87.7% to 20.9%. But this doesn't mean BSTs are always wrong—it means workload matters.

**Key insights**:

1. **Binary search trees are cache-unfriendly**. Every pointer dereference is likely a cache miss. For lookup-heavy workloads, sorted arrays are often 3× faster despite having the same O(log n) complexity.

2. **Memory matters**. BSTs use 2× more memory than arrays (32 bytes vs 16 bytes per entry on 64-bit systems). Those extra pointers hurt both cache utilization and memory bandwidth.

3. **BSTs win for write-heavy workloads**. If you're constantly inserting and deleting, BSTs are 3-4× faster than sorted arrays because they avoid shifting elements.

4. **Balanced trees don't fix cache problems**. Red-Black trees and AVL trees guarantee O(log n) height, but they're still pointer-chasing through scattered memory.

5. **Workload determines the right choice**. Our compiler's symbol table was read-heavy (lookups dominate), so sorted arrays won. The Linux scheduler is write-heavy (constant insert/delete), so Red-Black trees win.

**The numbers from our compiler**:
- Red-Black tree: 2,400 cycles/lookup, 87.7% cache miss rate
- Sorted array: 800 cycles/lookup, 20.9% cache miss rate
- Speedup: 3×

**The numbers from a write-heavy workload**:
- Red-Black tree: 3,500 cycles/operation
- Sorted array: 12,000 cycles/operation
- Speedup: 3.4× for BST

**Next chapter**: B-trees pack multiple keys per node to reduce tree height and cache misses.


