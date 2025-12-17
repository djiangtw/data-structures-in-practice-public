# Appendix E: Exercises

這個附錄提供實作練習來強化本書涵蓋的概念。每個練習都設計來幫助你獲得 hardware-aware data structure implementation 和 performance analysis 的實際經驗。

---

## Chapter 5: Linked Lists - The Cache Killer

### Exercise 1: Benchmark Challenge

**Objective**：比較 array-based 和 linked list implementations 的 stack。

**Task**:
1. 實作 stack 的 array 和 linked list 版本
2. 測量 10,000 operations 的 push/pop performance
3. 使用 Chapter 3 的 benchmark framework
4. 測量 execution time 和 cache misses

**Questions**:
- 哪個 implementation 更快？快多少？
- 每個的 cache miss rate 是多少？
- Performance 如何隨不同 stack sizes 變化（100, 1000, 10000 elements）？

---

### Exercise 2: Memory Pool

**Objective**：理解 allocation overhead 對 linked list performance 的影響。

**Task**:
1. 為 linked list nodes 實作 memory pool allocator
2. 與 malloc-based allocation 比較 performance
3. 測量 allocation time 和 fragmentation

**Questions**:
- Memory pool 快多少？
- Pool 的 memory overhead 是多少？
- Pool size 如何影響 performance？

---

### Exercise 3: Unrolled List

**Objective**：探索 cache-friendly 的 linked lists 變體。

**Task**:
1. 實作每個 node 16 elements 的 unrolled linked list
2. Benchmark 對比 standard linked list 和 array
3. 測量 sequential traversal 的 cache behavior

**Questions**:
- Unrolled list 與 standard linked list 比較如何？
- 每個 node 的最佳 elements 數量是多少？
- 什麼時候你會選 unrolled list 而不是 array？

---

### Exercise 4: Cache Analysis

**Objective**：分析不同 data sizes 的 cache behavior。

**Task**:
1. 用 `perf` 測量 array vs linked list traversal 的 cache misses
2. 改變 data size 從 1 KB 到 1 MB
3. 畫出 cache miss rate vs data size

**Questions**:
- 在什麼 size linked list 變得完全 cache-hostile？
- Cache miss rate 如何隨 data 超過 L1, L2, L3 cache sizes 變化？
- 你能從 data 識別 cache size thresholds 嗎？

---

### Exercise 5: Real-Time Analysis

**Objective**：理解 real-time systems 的 predictability requirements。

**Task**:
1. 測量 linked list operations 的 worst-case execution time
2. 跑 10,000 iterations 並記錄 min, max, median, P99
3. 比較 array 和 linked list 之間的 variance

**Questions**:
- 你在 linked list operations 看到多少 variance？
- Variance 對 1 kHz control loop 可接受嗎？
- 什麼造成 worst-case execution times？

---

## Chapter 1: The Performance Gap

### Exercise 1: Hash Table vs Binary Search

**Objective**：重現 Chapter 1 比較 hash tables 和 binary search 的實驗。

**Task**:
1. 為 500 device configurations 實作有 1024 buckets 的 hash table
2. 為相同 data 實作 sorted array 上的 binary search
3. 測量 10,000 lookups 的 cache misses 和 execution time
4. 改變 entries 數量（100, 500, 1000, 5000）

**Questions**:
- 哪個 data structure 更快？
- Cache miss rate 如何隨 dataset size 變化？
- 有 crossover point 嗎（hash table 變得比 binary search 慢）？

---

### Exercise 2: Memory Hierarchy Exploration

**Objective**：探索 memory hierarchy 對 performance 的影響。

**Task**:
1. 實作 sequential 和 random access patterns
2. 改變 working set size 從 1 KB 到 100 MB
3. 測量每個 size 的 throughput (GB/s)
4. 用 `perf` 測量 L1, L2, L3, DRAM access counts

**Questions**:
- 你能從 throughput graph 識別 cache sizes 嗎？
- Sequential 和 random access 的 performance gap 是多少？
- DRAM access 比 L1 cache 慢多少？

---

### Exercise 3: Compiler Optimization Impact

**Objective**：理解 compiler optimizations 如何影響 performance。

**Task**:
1. 用 `-O0`, `-O1`, `-O2`, `-O3` compile 一個簡單的 linked list traversal
2. 測量每個 optimization level 的 performance
3. 用 `objdump -d` 檢查生成的 assembly

**Questions**:
- 每個 optimization level 的 speedup 是多少？
- Compiler 做了什麼 optimizations？
- 有任何 optimizations 改變 algorithm behavior 嗎？

---

## Chapter 2: Memory Hierarchy

### Exercise 1: Cache Line Size Detection

**Objective**：實驗性地確定你 CPU 的 cache line size。

**Task**:
1. 創建 array 並用不同 strides 存取 elements（1, 2, 4, 8, 16, 32, 64, 128 bytes）
2. 測量每個 stride 的 cache misses
3. 畫出 cache misses vs stride
4. 從 inflection point 識別 cache line size

**Questions**:
- 你 CPU 的 cache line size 是多少？
- 當 stride 等於 cache line size 時 performance 如何變化？
- 當 stride 大於 cache line size 時發生什麼？

---

### Exercise 2: False Sharing

**Objective**：展示 false sharing 的 performance impact。

**Task**:
1. 創建 multi-threaded program，每個 thread 更新獨立的 counter
2. Version 1：在 array 中緊密 pack counters
3. Version 2：Pad counters 到獨立的 cache lines
4. 測量兩個版本的 throughput

**Questions**:
- Packed version 慢多少？
- Packed version 發生多少 cache line bounces？
- 最佳 padding size 是多少？

---

## Chapter 3: Benchmarking and Profiling

### Exercise 1: Build a Microbenchmark Framework

**Objective**：創建可重用的 benchmarking framework。

**Task**:
1. 用 RDTSC 或 clock_gettime 實作 high-precision timing
2. 加入 statistical analysis（mean, median, stddev, percentiles）
3. 實作 warmup runs 和 outlier detection
4. 用 perf_event_open 加入 cache miss measurement

**Questions**:
- 需要多少 iterations 才能得到穩定結果？
- 你的 timing mechanism 的 overhead 是多少？
- 你如何偵測和處理 outliers？

---

### Exercise 2: Profiling with perf

**Objective**：掌握 perf profiling tool。

**Task**:
1. 寫一個有明顯 performance bottleneck 的 program
2. 用 `perf record` 和 `perf report` 找 hotspot
3. 用 `perf stat` 測量 cache misses, branch mispredictions
4. 用 `perf annotate` 看 assembly-level performance

**Questions**:
- 多少百分比的時間花在 hotspot？
- Hotspot 的 cache miss rate 是多少？
- 你能識別造成 cache misses 的確切 instruction 嗎？

---

## Chapter 4: Arrays and Cache Locality

### Exercise 1: Row-Major vs Column-Major

**Objective**：測量 access patterns 的 performance impact。

**Task**:
1. 創建 1000×1000 matrix
2. 用 row-major order sum 所有 elements
3. 用 column-major order sum 所有 elements
4. 測量 cache misses 和 execution time

**Questions**:
- Column-major access 慢多少？
- 每個的 cache miss rate 是多少？
- Matrix size 如何影響 performance gap？

---

### Exercise 2: Structure of Arrays vs Array of Structures

**Objective**：比較 SoA 和 AoS layouts 的 cache efficiency。

**Task**:
1. 用 AoS layout 實作 particle simulation
2. 用 SoA layout 實作相同 simulation
3. 測量只有 position updates 的 performance
4. 測量存取所有 fields 時的 performance

**Questions**:
- 哪個 layout 對 position-only updates 更快？
- 哪個 layout 對存取所有 fields 更快？
- Fields 數量如何影響 trade-off？

---

## Chapter 6: Stacks and Queues

### Exercise 1: Ring Buffer Implementation

**Objective**：實作 cache-friendly ring buffer queue。

**Task**:
1. 實作 power-of-2 size 的 ring buffer
2. 與 linked list queue 比較
3. 測量 enqueue/dequeue operations 的 performance
4. 用不同 buffer sizes 測試（64, 256, 1024, 4096）

**Questions**:
- Ring buffer 快多少？
- Buffer full 時發生什麼？
- Buffer size 如何影響 cache performance？

---

### Exercise 2: Lock-Free Queue

**Objective**：實作 lock-free queue 並理解 atomic operations。

**Task**:
1. 實作 lock-free ring buffer 用 atomic operations
2. 與 mutex-based queue 比較
3. 測量 multi-threaded performance（2, 4, 8 threads）
4. 用 `perf` 測量 cache coherence traffic

**Questions**:
- Lock-free version 快多少？
- Contention 如何隨 thread count 變化？
- Atomic operations 造成多少 cache coherence traffic？

---

### Exercise 2: Stack Overflow Detection

**Objective**：理解 stack memory layout 和 overflow detection。

**Task**:
1. 寫一個 overflow stack 的 recursive function
2. 加入 canary values 來偵測 overflow
3. 測量你系統上的 stack size
4. 實作有 overflow protection 的 custom stack

**Questions**:
- 你系統上的 default stack size 是多少？
- 你如何在 crash 前偵測 stack overflow？
- Canary checks 的 performance overhead 是多少？

---

## Chapter 7: Hash Tables and Cache Conflicts

### Exercise 1: Hash Function Quality

**Objective**：比較不同 hash functions 的 cache behavior。

**Task**:
1. 實作三個 hash functions：simple sum, FNV-1a, MurmurHash
2. 測量 distribution quality（bucket occupancy variance）
3. 測量 lookups 的 cache miss rate
4. 用 real-world string data 測試（例如 dictionary words）

**Questions**:
- 哪個 hash function 有最好的 distribution？
- 哪個有最好的 cache behavior？
- Distribution 和 cache performance 之間有 trade-off 嗎？

---

### Exercise 2: Open Addressing vs Chaining

**Objective**：比較 collision resolution strategies。

**Task**:
1. 實作 chaining 的 hash table
2. 實作 linear probing 的 hash table
3. 測量不同 load factors 的 performance（0.5, 0.7, 0.9）
4. 測量兩個 implementations 的 cache misses

**Questions**:
- 哪個在 low load factors 更快？
- 哪個在 high load factors 更快？
- 每個的 cache miss rate 是多少？

---

## Chapter 8: Dynamic Arrays and Memory Management

### Exercise 1: Growth Factor Comparison

**Objective**：比較 dynamic arrays 的不同 growth strategies。

**Task**:
1. 實作 1.5× growth 的 dynamic array
2. 實作 2× growth 的 dynamic array
3. 實作 φ (1.618) growth 的 dynamic array
4. 測量 growing 到 1M elements 的 total reallocations 和 memory waste

**Questions**:
- 哪個 growth factor 最小化 reallocations？
- 哪個最小化 memory waste？
- 哪個有最好的 overall performance？

---

### Exercise 2: Custom Allocator

**Objective**：實作簡單的 memory allocator。

**Task**:
1. 實作 bump allocator (arena)
2. 實作 free list allocator
3. 與 malloc 比較 small allocations
4. 測量隨時間的 fragmentation

**Questions**:
- Bump allocator 快多少？
- Free list allocator 何時 fragment？
- 每個 allocator 的 memory overhead 是多少？

---

## Chapter 9: Binary Search Trees

### Exercise 1: BST vs Sorted Array

**Objective**：比較 tree-based 和 array-based search structures。

**Task**:
1. 實作 Red-Black tree
2. 實作 sorted array 用 binary search
3. 測量 10,000 elements 的 lookup performance
4. 測量兩者的 cache misses

**Questions**:
- 哪個對 lookups 更快？
- 哪個對 insertions 更快？
- 在什麼 size tree 變得更慢？

---

### Exercise 2: Tree Layout Optimization

**Objective**：探索 cache-friendly tree layouts。

**Task**:
1. 實作 standard pointer-based BST
2. 實作 array-based BST（implicit pointers）
3. 實作 van Emde Boas layout
4. 測量 tree traversal 的 cache misses

**Questions**:
- 哪個 layout 有最少 cache misses？
- Tree depth 如何影響 performance gap？
- 每個 layout 的 memory overhead 是多少？

---

## Chapter 10: B-Trees and Cache-Conscious Trees

### Exercise 1: Optimal Node Size

**Objective**：找你硬體的最佳 B-tree node size。

**Task**:
1. 實作可配置 node size 的 B-tree
2. 測試 node sizes：16, 32, 64, 128, 256 bytes
3. 測量 100,000 elements 的 lookup performance
4. 測量每個 node size 的 cache misses

**Questions**:
- 最佳 node size 是多少？
- 它如何與你的 cache line size 相關？
- 當 node size 超過 cache line size 時發生什麼？

---

### Exercise 2: B-Tree vs Hash Table

**Objective**：比較 B-trees 和 hash tables 用於 in-memory databases。

**Task**:
1. 實作 B-tree 和 hash table 用相同 API
2. 測量 point queries（exact match lookups）
3. 測量 range queries（scan 100 consecutive keys）
4. 測量 memory usage

**Questions**:
- 哪個對 point queries 更快？
- 哪個對 range queries 更快？
- Memory overhead 的 trade-off 是什麼？

---

## Chapter 11: Tries and Radix Trees

### Exercise 1: Trie Memory Optimization

**Objective**：減少 trie memory consumption。

**Task**:
1. 實作 standard trie（每個 node 26 pointers）
2. 實作 compressed trie (radix tree)
3. 實作 array-mapped trie（bitmap + compact array）
4. 測量 memory usage 和 lookup performance

**Questions**:
- 每個 implementation 用多少 memory？
- 哪個有最好的 lookup performance？
- Memory 和 speed 之間的 trade-off 是什麼？

---

### Exercise 2: Autocomplete Performance

**Objective**：比較 autocomplete 的 data structures。

**Task**:
1. 用 trie 實作 autocomplete
2. 用 sorted array + binary search 實作 autocomplete
3. 用 hash table 實作 autocomplete
4. 用 dictionary 的 50,000 words 測試

**Questions**:
- 哪個對 prefix search 最快？
- 哪個用最少 memory？
- Prefix length 如何影響 performance？

---

## Chapter 12: Heaps and Priority Queues

### Exercise 1: Heap Implementations

**Objective**：比較不同 heap implementations。

**Task**:
1. 實作 binary heap（array-based）
2. 實作 d-ary heap（d=4, d=8）
3. 實作 Fibonacci heap
4. 測量 insert 和 extract-min performance

**Questions**:
- 哪個 heap 有最好的 cache behavior？
- D-ary heap 的最佳 d 是多少？
- Fibonacci heap 何時值得複雜性？

---

### Exercise 2: Priority Queue for Task Scheduling

**Objective**：建立 real-time task scheduler。

**Task**:
1. 用 binary heap 實作 priority queue
2. 加入不同 priorities 的 tasks
3. 測量 worst-case extract-min time
4. 確保 real-time use 的 deterministic timing

**Questions**:
- Worst-case execution time 是多少？
- 對 1 kHz control loop 可接受嗎？
- 你如何減少 worst-case time？

---

## Chapter 13: Lock-Free Data Structures

### Exercise 1: Lock-Free Queue

**Objective**：用 CAS 實作 lock-free queue。

**Task**:
1. 實作 Michael-Scott lock-free queue
2. 實作 mutex-based queue 來比較
3. 用 1, 2, 4, 8 threads 測量 throughput
4. 用 perf 測量 contention

**Questions**:
- 在什麼 thread count lock-free 贏？
- CAS operations 的 overhead 是多少？
- 你如何處理 ABA problem？

---

### Exercise 2: Lock-Free Stack

**Objective**：建立更簡單的 lock-free data structure。

**Task**:
1. 用 CAS 實作 lock-free stack
2. 用多個 producer/consumer threads 測試
3. 測量 performance vs mutex-based stack
4. 識別並修復 ABA problem

**Questions**:
- Lock-free stack 比 mutex-based 快嗎？
- Contention 下發生多少 CAS retries？
- Memory ordering requirement 是什麼？

---

## Chapter 14: String Processing and Cache Efficiency

### Exercise 1: String Search Optimization

**Objective**：為 cache efficiency 優化 string search。

**Task**:
1. 實作 naive string search
2. 實作 Boyer-Moore algorithm
3. 實作 SIMD-based search（如果可用）
4. 測量每個的 cache misses

**Questions**:
- 哪個 algorithm 有最少 cache misses？
- String length 如何影響 performance？
- SIMD 何時值得複雜性？

---

### Exercise 2: Log Parser Optimization

**Objective**：建立 high-performance log parser。

**Task**:
1. 用 strchr/strncpy parse log lines
2. 用 manual parsing 優化（避免 string functions）
3. 為 timestamp parsing 加入 SIMD optimization
4. 測量 throughput（lines per second）

**Questions**:
- Manual parsing 快多少？
- SIMD 提供多少 speedup？
- Cache miss rate 是多少？

---

## Chapter 15: Graphs and Cache-Efficient Traversal

### Exercise 1: Graph Representations

**Objective**：比較 graph representations 的 cache efficiency。

**Task**:
1. 實作 adjacency list（array of pointers）
2. 實作 adjacency array（CSR format）
3. 實作 adjacency matrix
4. 測量每個的 BFS performance

**Questions**:
- 哪個有最少 cache misses？
- 哪個對 sparse graphs 最快？
- 哪個對 dense graphs 最快？

---

### Exercise 2: Graph Traversal Optimization

**Objective**：為 cache efficiency 優化 BFS。

**Task**:
1. 用 adjacency list 實作 standard BFS
2. 用 CSR format 優化
3. 加入 prefetching hints
4. 測量 cache misses 和 execution time

**Questions**:
- CSR format 改善多少 performance？
- Prefetching 有幫助嗎？
- 最佳 prefetch distance 是多少？

---

## Chapter 16: Bloom Filters and Probabilistic Data Structures

### Exercise 1: Bloom Filter Implementation

**Objective**：建立並調整 Bloom filter。

**Task**:
1. 實作可配置 size 和 hash count 的 Bloom filter
2. 用不同 parameters 測試 false positive rate
3. 與 hash table 比較 memory usage
4. 測量 lookup performance

**Questions**:
- 最佳 hash functions 數量是多少？
- Filter size 如何影響 false positive rate？
- 與 hash table 相比的 memory savings 是多少？

---

### Exercise 2: Counting Bloom Filter

**Objective**：擴展 Bloom filter 來支援 deletions。

**Task**:
1. 實作 counting Bloom filter
2. 用 insertions 和 deletions 測試
3. 測量 memory overhead vs standard Bloom filter
4. 測量 false positive rate

**Questions**:
- Counting 需要多少額外 memory？
- Deletion 增加 false positive rate 嗎？
- Counting Bloom filter 何時值得？

---

## Chapter 17: Bootloader Data Structures

### Exercise 1: Bootloader Optimization

**Objective**：最小化 bootloader execution time。

**Task**:
1. 用 linked lists 實作 device tree parser
2. 用 fixed-size arrays 優化
3. 測量兩個 implementations 的 boot time
4. Profile 找剩餘的 bottlenecks

**Questions**:
- Array-based version 快多少？
- Boot time 的最大 bottleneck 是什麼？
- 你能在 500ms 內 boot 嗎？

---

### Exercise 2: Memory-Constrained Data Structures

**Objective**：為 bootloader constraints 設計 data structures。

**Task**:
1. 用最小 memory 實作 symbol table
2. 完全避免 dynamic allocation
3. 測量 memory usage 和 lookup performance
4. 與 standard implementations 比較

**Questions**:
- 你能節省多少 memory？
- Performance trade-off 是什麼？
- 複雜性值得嗎？

---

## Chapter 18: Device Driver Queues

### Exercise 1: DMA Ring Buffer

**Objective**：實作 high-performance DMA ring buffer。

**Task**:
1. 為 packet reception 實作 ring buffer
2. 加入 overflow detection 和 handling
3. 測量 line rate 的 packet loss rate
4. 為 cache efficiency 優化

**Questions**:
- 什麼 buffer size 最小化 packet loss？
- 你如何處理 buffer overflow？
- Cache miss rate 是多少？

---

### Exercise 2: Interrupt Handler Optimization

**Objective**：最小化 interrupt handler execution time。

**Task**:
1. 用 linked list queue 實作 interrupt handler
2. 用 lock-free ring buffer 優化
3. 測量 interrupt latency
4. 測量 worst-case execution time

**Questions**:
- Ring buffer 快多少？
- Worst-case interrupt latency 是多少？
- 對 real-time requirements 可接受嗎？

---

## Chapter 19: Firmware Memory Management

### Exercise 1: Memory Pool Allocator

**Objective**：消除 firmware 中的 fragmentation。

**Task**:
1. 實作 fixed-size memory pools
2. 為多個 sizes 實作 slab allocator
3. 測量 72 小時的 fragmentation
4. 與 malloc 比較

**Questions**:
- Memory pools 發生 fragmentation 嗎？
- Memory overhead 是多少？
- 你需要多少 pool sizes？

---

### Exercise 2: Long-Running Firmware Test

**Objective**：確保 firmware 隨時間的穩定性。

**Task**:
1. 用你的 memory allocator 實作 firmware
2. 跑 72 小時的 continuous operation test
3. 監控 memory usage 和 fragmentation
4. 識別並修復任何 memory leaks

**Questions**:
- Firmware 能跑 72 小時不 crash 嗎？
- Memory usage 隨時間的趨勢是什麼？
- 有任何 memory leaks 嗎？

---

## Chapter 20: Benchmark Case Studies

### Exercise 1: Dhrystone Analysis

**Objective**：理解 compiler optimization 如何影響 Dhrystone scores。

**Task**:
1. 下載 Dhrystone 2.1 source code
2. 用不同 optimization levels compile：`-O0`, `-O1`, `-O2`, `-O3`
3. 用不同 compilers compile：GCC, Clang（如果可用）
4. 測量每個 configuration 的 DMIPS/MHz
5. 用 `objdump -d` 檢查生成的 assembly code

**Questions**:
- Scores 在 optimization levels 之間差異多少？
- Scores 在 compilers 之間差異多少？
- 你能識別 inflate score 的特定 optimizations 嗎？
- 看 assembly：compiler 在消除 dead code 嗎？
- 為什麼 Dhrystone 被認為過時？

**Expected Results**:
- `-O0` 到 `-O3` 有 5-10× speedup
- Compilers 之間 20-50% variance
- Constant propagation 和 dead code elimination 的證據

---

### Exercise 2: Coremark Implementation and Analysis

**Objective**：跑 Coremark 並理解它測量什麼。

**Task**:
1. 從 GitHub clone Coremark：https://github.com/eembc/coremark
2. 為你的 platform compile（native x86/ARM 或 RISC-V QEMU）
3. 跑至少 10 秒的 iterations
4. 分析四個 workloads：
   - Linked list operations (core_list_join.c)
   - Matrix operations (core_matrix.c)
   - State machine (core_state.c)
   - CRC calculation (core_util.c)
5. 用 `perf` 測量每個 workload 的 cache misses
6. 用不同 flags compile 並比較 scores

**Questions**:
- 你的 CoreMark/MHz score 是多少？
- 哪個 workload 有最高 cache miss rate？
- 哪個 workload 花最多時間？
- Compiler flags 如何影響 score？
- 為什麼 compiler 不能像 Dhrystone 那樣 optimize away Coremark？

**Expected Results**:
- CoreMark/MHz 在 2.5-5.5 之間（取決於 processor）
- Linked list workload 有最高 cache miss rate
- Matrix workload 花最多時間
- `-O3` 比 `-O2` 改善 10-30%

**Advanced**:
- 修改 Coremark 用不同 list sizes
- 測量 cache size 如何影響 performance
- 比較不同 architectures 的 performance（x86 vs ARM vs RISC-V）

---

### Exercise 3: Design Your Own Benchmark (Optional Challenge)

**Objective**：應用 benchmark design principles 創建 domain-specific benchmark。

**Task**:
1. 選擇特定 workload（例如 packet processing, image filtering, crypto）
2. 識別那個 workload 的 key operations
3. 設計一個 benchmark：
   - 用 runtime-determined inputs
   - 抵抗 compiler optimization
   - Validates results
   - 代表 realistic data sizes
4. 實作 benchmark
5. 用不同 compilers 和 optimization levels 測試
6. Document 你的 methodology

**Questions**:
- 你的 benchmark 測量什麼 operations？
- 你如何防止 dead code elimination？
- 你如何 validate correctness？
- 你的 benchmark 的 limitations 是什麼？
- 它與現有 benchmarks 比較如何？

**Example Workloads**:
- **Packet processing**：Parse headers, checksum, routing table lookup
- **Image filtering**：Convolution, color space conversion
- **Crypto**：AES encryption, SHA hashing
- **JSON parsing**：Tokenization, validation, tree building

**Deliverables**:
- 有清楚 documentation 的 source code
- Run rules（iterations, validation, reporting）
- 至少一個 platform 的 benchmark results
- 分析 benchmark 測量和不測量什麼

---

## Submission Guidelines

對想要 feedback 的讀者：

1. **Code**：在 GitHub 分享你的 implementation
2. **Benchmarks**：包含 benchmark results 和 hardware specifications
3. **Analysis**：寫簡短的 findings 分析
4. **Discussion**：加入書的 discussion forum（URL TBD）

---

## Resources

- Benchmark framework：見 Appendix A
- Hardware specifications：見 Appendix B
- Profiling tools：見 Appendix C
- Further reading：見 Appendix D

