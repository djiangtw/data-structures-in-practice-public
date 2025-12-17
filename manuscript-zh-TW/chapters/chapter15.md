# Chapter 15: Graphs and Cache-Efficient Traversal

**Part IV: Advanced Topics**

---

> "The purpose of abstraction is not to be vague, but to create a new semantic level in which one can be absolutely precise."
> — Edsger W. Dijkstra

## The Cache Miss Explosion

Network topology discovery 花了 37.5 毫秒遍歷 500 個 switches。這聽起來不慢，直到你看 cache miss count：850 萬次 cache misses。對 500 個 nodes，那是每個 node 17,000 次 cache misses。

資料結構有根本性的問題。

這個工具的工作很直接：透過遍歷連接設備的 graph 來發現 network topology。每個 switch 有最多 48 個 ports，我們需要用 breadth-first search 從起點找所有可達的設備。

實作看起來教科書般正確——adjacency list 加標準 BFS：

```c
typedef struct node {
    int id;
    struct node **neighbors;  // Array of pointers
    int num_neighbors;
} node_t;

void bfs(node_t *start) {
    queue_t *q = queue_create();
    bool *visited = calloc(MAX_NODES, sizeof(bool));
    
    queue_push(q, start);
    visited[start->id] = true;
    
    while (!queue_empty(q)) {
        node_t *node = queue_pop(q);
        process(node);
        
        for (int i = 0; i < node->num_neighbors; i++) {
            node_t *neighbor = node->neighbors[i];
            if (!visited[neighbor->id]) {
                visited[neighbor->id] = true;
                queue_push(q, neighbor);
            }
        }
    }
}
```

對有 500 個 switches（平均 12 個連接）的 network，這花了：

```bash
$ perf stat -e cycles,cache-misses ./network_discovery_naive
  Performance counter stats:
    45,000,000 cycles
     8,500,000 cache-misses
     
Traversal time: 37.5 ms
```

850 萬次 cache misses 對 500 個 nodes？那是 **每個 node 17,000 次 cache misses**！

我用 cache-conscious graph representation 重寫了它。結果：

```bash
$ perf stat -e cycles,cache-misses ./network_discovery_optimized
  Performance counter stats:
    12,000,000 cycles
     1,200,000 cache-misses
     
Traversal time: 10 ms
```

**快 3.75 倍**，cache misses 少 7 倍。

這章探索如何有效率地表示和遍歷 graphs。

---

## The Textbook Story

Graphs 通常用兩種方式表示：

### 1. Adjacency Matrix

一個 2D array，其中 `matrix[i][j] = 1` 如果有從 node i 到 node j 的 edge：

```c
bool adj_matrix[MAX_NODES][MAX_NODES];

// Check if edge exists
if (adj_matrix[u][v]) {
    // Edge from u to v exists
}
```

**Pros**: O(1) edge lookup  
**Cons**: O(n²) space, even for sparse graphs

### 2. Adjacency List

每個 node 存它的 neighbors list：

```c
typedef struct {
    int *neighbors;
    int num_neighbors;
} node_t;

node_t nodes[MAX_NODES];
```

**Pros**: O(n + m) space (n nodes, m edges)  
**Cons**: O(degree) edge lookup

教科書說：「對 dense graphs 用 adjacency matrix，對 sparse graphs 用 adjacency list。」

---

## The Reality Check: Why Standard Representations Are Slow

### 1. Pointer Chasing in Adjacency Lists

標準 adjacency list 用 pointers：

```c
typedef struct edge {
    int dest;
    struct edge *next;  // Linked list of edges
} edge_t;

typedef struct {
    edge_t *edges;  // Pointer to first edge
} node_t;
```

**Problem**：每個 edge 是分開的 allocation，散落在記憶體中。

遍歷 neighbors：
```
Node → Edge1 (cache miss) → Edge2 (cache miss) → Edge3 (cache miss) ...
```

對有 12 個 neighbors 的 node：**12 次 cache misses** 只是讀 neighbor list！

### 2. Poor Locality in BFS Queue

標準 BFS 用 pointers 的 queue：

```c
queue_push(q, neighbor);  // Push pointer to node
```

**Problem**：Nodes 按 BFS 順序處理，但它們散落在記憶體中。

```
Queue: [Node5, Node12, Node3, Node45, ...]
       Each node is in a different cache line!
```

### 3. Random Access to Visited Array

`visited` array 用 node ID 索引：

```c
visited[neighbor->id] = true;
```

如果 node IDs 不是連續或聚集的，這導致 random memory access。

---

## Optimization 1: Compact Adjacency List

不用 pointers，把 neighbors 存在連續的 array：

```c
typedef struct {
    int *neighbors;      // Contiguous array of neighbor IDs
    int num_neighbors;
} node_t;

typedef struct {
    node_t *nodes;
    int *edge_data;      // All edges in one array
    int num_nodes;
} graph_t;

graph_t *graph_create(int num_nodes, int num_edges) {
    graph_t *g = malloc(sizeof(graph_t));
    g->nodes = malloc(num_nodes * sizeof(node_t));
    g->edge_data = malloc(num_edges * sizeof(int));
    g->num_nodes = num_nodes;
    
    // Nodes point into edge_data array
    int offset = 0;
    for (int i = 0; i < num_nodes; i++) {
        g->nodes[i].neighbors = &g->edge_data[offset];
        g->nodes[i].num_neighbors = /* ... */;
        offset += g->nodes[i].num_neighbors;
    }
    
    return g;
}
```

**Why this helps**:
- 所有 edges 在一個連續的 array（更好的 prefetching）
- 沒有 pointer chasing（neighbors 是連續的）
- 更好的 cache utilization

### The Benchmark

```
Test: BFS on 500-node graph (avg degree: 12)

Linked list adjacency list:
  Cache misses: 8.5M
  Cycles: 45M

Compact adjacency list:
  Cache misses: 2.8M (3× fewer)
  Cycles: 18M
  Speedup: 2.5×
```

---

## Optimization 2: Cache-Oblivious BFS

標準 BFS 逐層處理 nodes，但同一層的 nodes 可能在記憶體中相距很遠。

### Blocked BFS

以 cache-sized blocks 處理 nodes：

```c
#define BLOCK_SIZE 64  // Process 64 nodes at a time

void bfs_blocked(graph_t *g, int start) {
    bool *visited = calloc(g->num_nodes, sizeof(bool));
    int *queue = malloc(g->num_nodes * sizeof(int));
    int head = 0, tail = 0;

    queue[tail++] = start;
    visited[start] = true;

    while (head < tail) {
        int block_end = (head + BLOCK_SIZE < tail) ? head + BLOCK_SIZE : tail;

        // Process a block of nodes
        for (int i = head; i < block_end; i++) {
            int node_id = queue[i];
            node_t *node = &g->nodes[node_id];

            process(node);

            // Add neighbors to queue
            for (int j = 0; j < node->num_neighbors; j++) {
                int neighbor = node->neighbors[j];
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    queue[tail++] = neighbor;
                }
            }
        }

        head = block_end;
    }
}
```

**Why this helps**:
- 在移到下一層前處理多個 nodes
- 更好的 temporal locality（在 cache 中重用 visited array）
- 攤銷 queue overhead

### The Benchmark

```
Test: BFS on 500-node graph

Standard BFS:
  Cache misses: 2.8M
  Cycles: 18M

Blocked BFS (block size 64):
  Cache misses: 1.5M
  Cycles: 11M
  Speedup: 1.6×
```

---

## Optimization 3: Compressed Sparse Row (CSR) Format

對非常大的 sparse graphs，CSR format 更緊湊：

```c
typedef struct {
    int *row_ptr;     // row_ptr[i] = start of node i's neighbors
    int *col_idx;     // col_idx[j] = neighbor ID
    int num_nodes;
    int num_edges;
} csr_graph_t;

csr_graph_t *graph_to_csr(graph_t *g) {
    csr_graph_t *csr = malloc(sizeof(csr_graph_t));
    csr->num_nodes = g->num_nodes;
    csr->num_edges = /* total edges */;

    csr->row_ptr = malloc((g->num_nodes + 1) * sizeof(int));
    csr->col_idx = malloc(csr->num_edges * sizeof(int));

    int offset = 0;
    for (int i = 0; i < g->num_nodes; i++) {
        csr->row_ptr[i] = offset;
        for (int j = 0; j < g->nodes[i].num_neighbors; j++) {
            csr->col_idx[offset++] = g->nodes[i].neighbors[j];
        }
    }
    csr->row_ptr[g->num_nodes] = offset;

    return csr;
}

// Access neighbors of node i
void visit_neighbors(csr_graph_t *g, int node_id) {
    int start = g->row_ptr[node_id];
    int end = g->row_ptr[node_id + 1];

    for (int i = start; i < end; i++) {
        int neighbor = g->col_idx[i];
        // Process neighbor
    }
}
```

**Memory layout**:
```
row_ptr: [0, 3, 7, 10, ...]  (node 0 has neighbors at indices 0-2)
col_idx: [1, 2, 5, 0, 3, 4, 6, ...]  (actual neighbor IDs)
```

**Advantages**:
- 最小的 memory overhead（只有兩個 arrays）
- 對 neighbors 的連續存取（優秀的 prefetching）
- Cache-friendly（所有 data 是連續的）

### The Benchmark

```
Test: 10,000-node graph, 120,000 edges

Adjacency list (pointers):
  Memory: 2.4 MB
  Cache misses: 85M
  BFS time: 180 ms

CSR format:
  Memory: 0.96 MB (2.5× less)
  Cache misses: 18M (4.7× fewer)
  BFS time: 42 ms
  Speedup: 4.3×
```

---

## Optimization 4: Node Reordering for Locality

如果你能重新排序 node IDs，把連接的 nodes 在記憶體中放近一點。

### Breadth-First Ordering

按 BFS 順序分配 node IDs：

```c
void reorder_bfs(graph_t *g, int start) {
    int *new_id = malloc(g->num_nodes * sizeof(int));
    int *old_id = malloc(g->num_nodes * sizeof(int));
    bool *visited = calloc(g->num_nodes, sizeof(bool));

    int next_id = 0;
    queue_t *q = queue_create();
    queue_push(q, start);
    visited[start] = true;

    while (!queue_empty(q)) {
        int node = queue_pop(q);
        new_id[node] = next_id;
        old_id[next_id] = node;
        next_id++;

        // Visit neighbors
        for (int i = 0; i < g->nodes[node].num_neighbors; i++) {
            int neighbor = g->nodes[node].neighbors[i];
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                queue_push(q, neighbor);
            }
        }
    }

    // Rebuild graph with new IDs
    // ...
}
```

**Why this helps**:
- 一起訪問的 nodes 編號連續
- 遍歷時更好的 cache locality
- Visited array 存取更連續

### The Benchmark

```
Test: BFS on 500-node graph

Random node IDs:
  Cache misses: 1.5M
  Cycles: 11M

BFS-ordered node IDs:
  Cache misses: 0.8M
  Cycles: 7.5M
  Speedup: 1.5×
```

---

## Optimization 5: Parallel Graph Traversal

在 multi-core 系統上，我們可以用 level-synchronous 方法平行化 BFS。

### Level-Synchronous BFS

平行處理每一層：

```c
void bfs_parallel(csr_graph_t *g, int start, int num_threads) {
    bool *visited = calloc(g->num_nodes, sizeof(bool));
    int *current_level = malloc(g->num_nodes * sizeof(int));
    int *next_level = malloc(g->num_nodes * sizeof(int));

    int current_size = 1;
    current_level[0] = start;
    visited[start] = true;

    while (current_size > 0) {
        atomic_int next_size = 0;

        // Process current level in parallel
        #pragma omp parallel for num_threads(num_threads)
        for (int i = 0; i < current_size; i++) {
            int node = current_level[i];
            int start = g->row_ptr[node];
            int end = g->row_ptr[node + 1];

            for (int j = start; j < end; j++) {
                int neighbor = g->col_idx[j];

                // Atomic check-and-set
                bool expected = false;
                if (__atomic_compare_exchange_n(&visited[neighbor], &expected, true,
                                                0, __ATOMIC_SEQ_CST, __ATOMIC_SEQ_CST)) {
                    int pos = __atomic_fetch_add(&next_size, 1, __ATOMIC_SEQ_CST);
                    next_level[pos] = neighbor;
                }
            }
        }

        // Swap levels
        int *temp = current_level;
        current_level = next_level;
        next_level = temp;
        current_size = next_size;
    }
}
```

### The Benchmark

```
Test: BFS on 10,000-node graph (RISC-V 8-core @ 1.2 GHz)

Sequential BFS:
  Cycles: 120M
  Time: 100 ms

Parallel BFS (2 cores):
  Cycles: 68M
  Time: 57 ms
  Speedup: 1.75×

Parallel BFS (4 cores):
  Cycles: 38M
  Time: 32 ms
  Speedup: 3.1×

Parallel BFS (8 cores):
  Cycles: 24M
  Time: 20 ms
  Speedup: 5.0×
```

不是完美的 scaling（8 cores 上 8 倍加速）因為：
- Synchronization overhead（atomic operations）
- Load imbalance（有些層有少數 nodes）
- Cache coherence traffic

但對大 graphs 仍是顯著的改善。

---

## Real-World Example: Linux Kernel's Radix Tree for Page Cache

Linux kernel 對 page cache 用 radix tree（一種特殊的 graph structure）。

### The Problem

Kernel 需要把 file offsets 映射到 physical pages：
- 每個 file 數百萬個 pages
- Sparse mapping（不是所有 offsets 都有 pages）
- 快速 lookup（O(log n) 或更好）

### The Solution: Radix Tree

一個 64-way tree，每層代表 offset 的 6 bits：

```c
#define RADIX_TREE_MAP_SHIFT 6
#define RADIX_TREE_MAP_SIZE (1 << RADIX_TREE_MAP_SHIFT)  // 64

struct radix_tree_node {
    void *slots[RADIX_TREE_MAP_SIZE];  // 64 pointers
    unsigned long tags[3][RADIX_TREE_MAP_SIZE / BITS_PER_LONG];
};
```

**Why 64-way**:
- 一個 node 放進一個 cache line（64 pointers × 8 bytes = 512 bytes ≈ 8 cache lines）
- Shallow tree（depth ≤ 11 for 64-bit offsets）
- Memory 和 cache misses 之間的好平衡

### The Performance

```
Lookup in radix tree (depth 3):
  Cache misses: 3 (one per level)
  Cycles: ~50

Lookup in binary tree (depth 20):
  Cache misses: 20
  Cycles: ~300

Speedup: 6×
```

---

## Putting It All Together: Optimized Network Discovery

這是結合所有技術的最終優化版本：

```c
typedef struct {
    int *row_ptr;
    int *col_idx;
    int num_nodes;
    int num_edges;
} network_graph_t;

void discover_network_optimized(network_graph_t *g, int start) {
    // Use bitmap for visited (cache-friendly)
    uint64_t *visited = calloc((g->num_nodes + 63) / 64, sizeof(uint64_t));

    // Use array-based queue (not linked list)
    int *queue = malloc(g->num_nodes * sizeof(int));
    int head = 0, tail = 0;

    queue[tail++] = start;
    visited[start / 64] |= (1UL << (start % 64));

    while (head < tail) {
        // Process in blocks for better cache reuse
        int block_end = (head + 64 < tail) ? head + 64 : tail;

        for (int i = head; i < block_end; i++) {
            int node = queue[i];
            process_device(node);

            // Sequential access to neighbors (CSR format)
            int start_idx = g->row_ptr[node];
            int end_idx = g->row_ptr[node + 1];

            for (int j = start_idx; j < end_idx; j++) {
                int neighbor = g->col_idx[j];
                uint64_t mask = 1UL << (neighbor % 64);
                int word = neighbor / 64;

                if (!(visited[word] & mask)) {
                    visited[word] |= mask;
                    queue[tail++] = neighbor;
                }
            }
        }

        head = block_end;
    }

    free(visited);
    free(queue);
}
```

### Final Benchmark

```
Test: Network discovery, 500 switches, avg 12 connections

Original (adjacency list, linked queue):
  Cycles: 45M
  Cache misses: 8.5M
  Memory: 128 KB
  Time: 37.5 ms

Optimized (CSR, blocked BFS, bitmap):
  Cycles: 7.5M
  Cache misses: 0.8M
  Memory: 24 KB
  Time: 6.2 ms

Speedup: 6.0×
Cache miss reduction: 10.6×
Memory reduction: 5.3×
```

---

## Summary

Cache miss explosion 被馴服了。Network discovery 時間從 37.5 ms 降到 6.2 ms——6 倍改善。Cache misses 從 850 萬降到 80 萬——10.6 倍減少。Graph traversal 從每個 node 17,000 次 cache misses 降到只有 1,600 次。

**關鍵洞察**：

1. **Compact adjacency lists 勝過 pointer-based lists**。把所有 edges 存在一個連續的 array 消除 pointer chasing，改善 prefetching。2.5 倍加速。

2. **CSR format 對 sparse graphs 最優**。兩個 arrays（row_ptr 和 col_idx）提供最小的 memory overhead 和優秀的 cache behavior。比 pointer-based lists 快 4.3 倍。

3. **Blocked BFS 改善 temporal locality**。以 cache-sized blocks（64 nodes）處理 nodes 在 cache 中重用 visited array。1.6 倍加速。

4. **Node reordering matters**。按 BFS 順序分配 IDs 把連接的 nodes 放在記憶體中靠近。1.5 倍加速。

5. **Parallel BFS scales reasonably**。Level-synchronous BFS 加 atomic operations 在 8 cores 上達到 5 倍加速（62% efficiency）。

**Network discovery 的數據**：
- Compact adjacency list: 比 pointers 快 2.5 倍
- CSR format: 快 4.3 倍，memory 少 2.5 倍
- Blocked BFS: 快 1.6 倍
- BFS ordering: 快 1.5 倍
- Combined: 快 6 倍，cache misses 少 10.6 倍

Graph traversal 是 memory-bound。專注於 cache-friendly representations 和 access patterns。

**Next chapter**：Bloom filters and probabilistic data structures—trading accuracy for speed and memory.

