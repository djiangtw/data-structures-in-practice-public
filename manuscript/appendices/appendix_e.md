# Appendix E: Exercises

This appendix provides hands-on exercises to reinforce the concepts covered throughout the book. Each exercise is designed to help you gain practical experience with hardware-aware data structure implementation and performance analysis.

---

## Chapter 5: Linked Lists - The Cache Killer

### Exercise 1: Benchmark Challenge

**Objective**: Compare array-based and linked list implementations of a stack.

**Task**:
1. Implement both array and linked list versions of a stack
2. Measure push/pop performance for 10,000 operations
3. Use the benchmark framework from Chapter 3
4. Measure both execution time and cache misses

**Questions**:
- Which implementation is faster? By how much?
- What is the cache miss rate for each?
- How does performance change with different stack sizes (100, 1000, 10000 elements)?

---

### Exercise 2: Memory Pool

**Objective**: Understand the impact of allocation overhead on linked list performance.

**Task**:
1. Implement a memory pool allocator for linked list nodes
2. Compare performance with malloc-based allocation
3. Measure allocation time and fragmentation

**Questions**:
- How much faster is the memory pool?
- What is the memory overhead of the pool?
- How does pool size affect performance?

---

### Exercise 3: Unrolled List

**Objective**: Explore cache-friendly variations of linked lists.

**Task**:
1. Implement an unrolled linked list with 16 elements per node
2. Benchmark against standard linked list and array
3. Measure cache behavior for sequential traversal

**Questions**:
- How does the unrolled list compare to standard linked list?
- What is the optimal number of elements per node?
- When would you choose an unrolled list over an array?

---

### Exercise 4: Cache Analysis

**Objective**: Analyze cache behavior at different data sizes.

**Task**:
1. Use `perf` to measure cache misses for array vs linked list traversal
2. Vary the data size from 1 KB to 1 MB
3. Plot cache miss rate vs data size

**Questions**:
- At what size does the linked list become completely cache-hostile?
- How does the cache miss rate change as data exceeds L1, L2, L3 cache sizes?
- Can you identify the cache size thresholds from the data?

---

### Exercise 5: Real-Time Analysis

**Objective**: Understand predictability requirements for real-time systems.

**Task**:
1. Measure worst-case execution time for linked list operations
2. Run 10,000 iterations and record min, max, median, P99
3. Compare variance between array and linked list

**Questions**:
- How much variance do you see in linked list operations?
- Is the variance acceptable for a 1 kHz control loop?
- What causes the worst-case execution times?

---

## Chapter 1: The Performance Gap

### Exercise 1: Hash Table vs Binary Search

**Objective**: Reproduce the Chapter 1 experiment comparing hash tables and binary search.

**Task**:
1. Implement a hash table with 1024 buckets for 500 device configurations
2. Implement binary search on a sorted array for the same data
3. Measure cache misses and execution time for 10,000 lookups
4. Vary the number of entries (100, 500, 1000, 5000)

**Questions**:
- At what size does the hash table become slower than binary search?
- What is the cache miss rate for each implementation?
- How does the hash table size (number of buckets) affect performance?

---

### Exercise 2: Cache Miss Analysis

**Objective**: Understand the relationship between cache misses and execution time.

**Task**:
1. Write a simple array traversal program
2. Use `perf` to measure cache misses and cycles
3. Calculate the cost per cache miss
4. Compare with the theoretical 100-cycle penalty

**Questions**:
- What is the actual cache miss penalty on your hardware?
- How does it vary with L1, L2, and L3 cache misses?
- Can you identify the cache sizes from the performance data?

---

## Chapter 2: Memory Hierarchy

### Exercise 1: Cache Line Size Detection

**Objective**: Experimentally determine your CPU's cache line size.

**Task**:
1. Create an array and access elements with varying strides (1, 2, 4, 8, 16, 32, 64, 128 bytes)
2. Measure cache misses for each stride
3. Plot cache misses vs stride
4. Identify the cache line size from the inflection point

**Questions**:
- What is your CPU's cache line size?
- How does performance change when stride equals cache line size?
- What happens when stride is larger than cache line size?

---

### Exercise 2: False Sharing

**Objective**: Demonstrate the performance impact of false sharing.

**Task**:
1. Create a multi-threaded program where each thread updates a separate counter
2. Version 1: Pack counters tightly in an array
3. Version 2: Pad counters to separate cache lines
4. Measure throughput for both versions

**Questions**:
- How much slower is the packed version?
- How many cache line bounces occur in the packed version?
- What is the optimal padding size?

---

## Chapter 3: Benchmarking and Profiling

### Exercise 1: Build a Microbenchmark Framework

**Objective**: Create a reusable benchmarking framework.

**Task**:
1. Implement high-precision timing using RDTSC or clock_gettime
2. Add statistical analysis (mean, median, stddev, percentiles)
3. Implement warmup runs and outlier detection
4. Add cache miss measurement using perf_event_open

**Questions**:
- How many iterations are needed for stable results?
- What is the overhead of your timing mechanism?
- How do you detect and handle outliers?

---

### Exercise 2: Profiling with perf

**Objective**: Master the perf profiling tool.

**Task**:
1. Write a program with an obvious performance bottleneck
2. Use `perf record` and `perf report` to find the hotspot
3. Use `perf stat` to measure cache misses, branch mispredictions
4. Use `perf annotate` to see assembly-level performance

**Questions**:
- What percentage of time is spent in the hotspot?
- What is the cache miss rate in the hotspot?
- Can you identify the exact instruction causing cache misses?

---

## Chapter 4: Arrays and Cache Locality

### Exercise 1: Row-Major vs Column-Major

**Objective**: Measure the performance impact of access patterns.

**Task**:
1. Create a 1000×1000 matrix
2. Sum all elements using row-major order
3. Sum all elements using column-major order
4. Measure cache misses and execution time

**Questions**:
- How much slower is column-major access?
- What is the cache miss rate for each?
- How does matrix size affect the performance gap?

---

### Exercise 2: Structure of Arrays vs Array of Structures

**Objective**: Compare SoA and AoS layouts for cache efficiency.

**Task**:
1. Implement particle simulation with AoS layout
2. Implement the same simulation with SoA layout
3. Measure performance for position updates only
4. Measure performance when accessing all fields

**Questions**:
- Which layout is faster for position-only updates?
- Which layout is faster when accessing all fields?
- How does the number of fields affect the trade-off?

---

## Chapter 6: Stacks and Queues

### Exercise 1: Ring Buffer Implementation

**Objective**: Implement a cache-friendly ring buffer queue.

**Task**:
1. Implement a ring buffer with power-of-2 size
2. Compare with a linked list queue
3. Measure performance for enqueue/dequeue operations
4. Test with different buffer sizes (64, 256, 1024, 4096)

**Questions**:
- How much faster is the ring buffer?
- What happens when the buffer is full?
- How does buffer size affect cache performance?

---

### Exercise 2: Stack Overflow Detection

**Objective**: Understand stack memory layout and overflow detection.

**Task**:
1. Write a recursive function that overflows the stack
2. Add canary values to detect overflow
3. Measure the stack size on your system
4. Implement a custom stack with overflow protection

**Questions**:
- What is the default stack size on your system?
- How can you detect stack overflow before it crashes?
- What is the performance overhead of canary checks?

---

## Chapter 7: Hash Tables and Cache Conflicts

### Exercise 1: Hash Function Quality

**Objective**: Compare different hash functions for cache behavior.

**Task**:
1. Implement three hash functions: simple sum, FNV-1a, MurmurHash
2. Measure distribution quality (bucket occupancy variance)
3. Measure cache miss rate for lookups
4. Test with real-world string data (e.g., dictionary words)

**Questions**:
- Which hash function has the best distribution?
- Which has the best cache behavior?
- Is there a trade-off between distribution and cache performance?

---

### Exercise 2: Open Addressing vs Chaining

**Objective**: Compare collision resolution strategies.

**Task**:
1. Implement hash table with chaining
2. Implement hash table with linear probing
3. Measure performance at different load factors (0.5, 0.7, 0.9)
4. Measure cache misses for both implementations

**Questions**:
- Which is faster at low load factors?
- Which is faster at high load factors?
- What is the cache miss rate for each?

---

## Chapter 8: Dynamic Arrays and Memory Management

### Exercise 1: Growth Factor Comparison

**Objective**: Compare different growth strategies for dynamic arrays.

**Task**:
1. Implement dynamic array with 1.5× growth
2. Implement dynamic array with 2× growth
3. Implement dynamic array with φ (1.618) growth
4. Measure total reallocations and memory waste for growing to 1M elements

**Questions**:
- Which growth factor minimizes reallocations?
- Which minimizes memory waste?
- Which has the best overall performance?

---

### Exercise 2: Custom Allocator

**Objective**: Implement a simple memory allocator.

**Task**:
1. Implement a bump allocator (arena)
2. Implement a free list allocator
3. Compare with malloc for small allocations
4. Measure fragmentation over time

**Questions**:
- How much faster is the bump allocator?
- When does the free list allocator fragment?
- What is the memory overhead of each allocator?

---

## Chapter 9: Binary Search Trees

### Exercise 1: BST vs Sorted Array

**Objective**: Compare tree-based and array-based search structures.

**Task**:
1. Implement Red-Black tree
2. Implement sorted array with binary search
3. Measure lookup performance for 10,000 elements
4. Measure cache misses for both

**Questions**:
- Which is faster for lookups?
- Which is faster for insertions?
- At what size does the tree become slower?

---

### Exercise 2: Tree Layout Optimization

**Objective**: Explore cache-friendly tree layouts.

**Task**:
1. Implement standard pointer-based BST
2. Implement array-based BST (implicit pointers)
3. Implement van Emde Boas layout
4. Measure cache misses for tree traversal

**Questions**:
- Which layout has the fewest cache misses?
- How does tree depth affect the performance gap?
- What is the memory overhead of each layout?

---

## Chapter 10: B-Trees and Cache-Conscious Trees

### Exercise 1: Optimal Node Size

**Objective**: Find the optimal B-tree node size for your hardware.

**Task**:
1. Implement B-tree with configurable node size
2. Test node sizes: 16, 32, 64, 128, 256 bytes
3. Measure lookup performance for 100,000 elements
4. Measure cache misses for each node size

**Questions**:
- What is the optimal node size?
- How does it relate to your cache line size?
- What happens when node size exceeds cache line size?

---

### Exercise 2: B-Tree vs Hash Table

**Objective**: Compare B-trees and hash tables for in-memory databases.

**Task**:
1. Implement B-tree with optimal node size
2. Implement cache-friendly hash table
3. Measure performance for random lookups
4. Measure performance for range queries

**Questions**:
- Which is faster for point queries?
- Which is faster for range queries?
- How does dataset size affect the trade-off?

---

## Chapter 11: Tries and Radix Trees

### Exercise 1: Trie Memory Optimization

**Objective**: Reduce trie memory consumption.

**Task**:
1. Implement standard trie (26 pointers per node)
2. Implement compressed trie (radix tree)
3. Implement array-mapped trie (bitmap + compact array)
4. Measure memory usage and lookup performance

**Questions**:
- How much memory does each implementation use?
- Which has the best lookup performance?
- What is the trade-off between memory and speed?

---

### Exercise 2: Autocomplete Performance

**Objective**: Compare data structures for autocomplete.

**Task**:
1. Implement autocomplete with trie
2. Implement autocomplete with sorted array + binary search
3. Implement autocomplete with hash table
4. Test with 50,000 words from a dictionary

**Questions**:
- Which is fastest for prefix search?
- Which uses the least memory?
- How does prefix length affect performance?

---

## Chapter 12: Heaps and Priority Queues

### Exercise 1: Heap Implementations

**Objective**: Compare different heap implementations.

**Task**:
1. Implement binary heap (array-based)
2. Implement d-ary heap (d=4, d=8)
3. Implement Fibonacci heap
4. Measure insert and extract-min performance

**Questions**:
- Which heap has the best cache behavior?
- What is the optimal d for d-ary heap?
- When is Fibonacci heap worth the complexity?

---

### Exercise 2: Priority Queue for Task Scheduling

**Objective**: Build a real-time task scheduler.

**Task**:
1. Implement priority queue with binary heap
2. Add tasks with different priorities
3. Measure worst-case extract-min time
4. Ensure deterministic timing for real-time use

**Questions**:
- What is the worst-case execution time?
- Is it acceptable for a 1 kHz control loop?
- How can you reduce worst-case time?

---

## Chapter 13: Lock-Free Data Structures

### Exercise 1: Lock-Free Queue

**Objective**: Implement a lock-free queue using CAS.

**Task**:
1. Implement Michael-Scott lock-free queue
2. Implement mutex-based queue for comparison
3. Measure throughput with 1, 2, 4, 8 threads
4. Measure contention using perf

**Questions**:
- At what thread count does lock-free win?
- What is the overhead of CAS operations?
- How do you handle the ABA problem?

---

### Exercise 2: Lock-Free Stack

**Objective**: Build a simpler lock-free data structure.

**Task**:
1. Implement lock-free stack using CAS
2. Test with multiple producer/consumer threads
3. Measure performance vs mutex-based stack
4. Identify and fix ABA problem

**Questions**:
- Is the lock-free stack faster than mutex-based?
- How many CAS retries occur under contention?
- What is the memory ordering requirement?

---

## Chapter 14: String Processing and Cache Efficiency

### Exercise 1: String Search Optimization

**Objective**: Optimize string search for cache efficiency.

**Task**:
1. Implement naive string search
2. Implement Boyer-Moore algorithm
3. Implement SIMD-based search (if available)
4. Measure cache misses for each

**Questions**:
- Which algorithm has the fewest cache misses?
- How does string length affect performance?
- When is SIMD worth the complexity?

---

### Exercise 2: Log Parser Optimization

**Objective**: Build a high-performance log parser.

**Task**:
1. Parse log lines using strchr/strncpy
2. Optimize using manual parsing (avoid string functions)
3. Add SIMD optimization for timestamp parsing
4. Measure throughput (lines per second)

**Questions**:
- How much faster is manual parsing?
- What is the cache miss rate for each approach?
- Can you achieve 3M lines/second?

---

## Chapter 15: Graphs and Cache-Efficient Traversal

### Exercise 1: Graph Representations

**Objective**: Compare graph representations for cache efficiency.

**Task**:
1. Implement adjacency list (array of pointers)
2. Implement adjacency array (CSR format)
3. Implement adjacency matrix
4. Measure BFS performance for each

**Questions**:
- Which has the fewest cache misses?
- Which is fastest for sparse graphs?
- Which is fastest for dense graphs?

---

### Exercise 2: Graph Traversal Optimization

**Objective**: Optimize BFS for cache efficiency.

**Task**:
1. Implement standard BFS with adjacency list
2. Optimize using CSR format
3. Add prefetching hints
4. Measure cache misses and execution time

**Questions**:
- How much does CSR format improve performance?
- Does prefetching help?
- What is the optimal prefetch distance?

---

## Chapter 16: Bloom Filters and Probabilistic Data Structures

### Exercise 1: Bloom Filter Implementation

**Objective**: Build and tune a Bloom filter.

**Task**:
1. Implement Bloom filter with configurable size and hash count
2. Test false positive rate with different parameters
3. Compare memory usage with hash table
4. Measure lookup performance

**Questions**:
- What is the optimal number of hash functions?
- How does filter size affect false positive rate?
- What is the memory savings vs hash table?

---

### Exercise 2: Counting Bloom Filter

**Objective**: Extend Bloom filter to support deletions.

**Task**:
1. Implement counting Bloom filter
2. Test with insertions and deletions
3. Measure memory overhead vs standard Bloom filter
4. Measure false positive rate

**Questions**:
- How much more memory does counting require?
- Does deletion increase false positive rate?
- When is counting Bloom filter worth it?

---

## Chapter 17: Bootloader Data Structures

### Exercise 1: Bootloader Optimization

**Objective**: Minimize bootloader execution time.

**Task**:
1. Implement device tree parser with linked lists
2. Optimize using fixed-size arrays
3. Measure boot time for both implementations
4. Profile to find remaining bottlenecks

**Questions**:
- How much faster is the array-based version?
- What is the largest bottleneck in boot time?
- Can you boot in under 500ms?

---

### Exercise 2: Memory-Constrained Data Structures

**Objective**: Design data structures for bootloader constraints.

**Task**:
1. Implement symbol table with minimal memory
2. Avoid dynamic allocation entirely
3. Measure memory usage and lookup performance
4. Compare with standard implementations

**Questions**:
- How much memory can you save?
- What is the performance trade-off?
- Is the complexity worth it?

---

## Chapter 18: Device Driver Queues

### Exercise 1: DMA Ring Buffer

**Objective**: Implement a high-performance DMA ring buffer.

**Task**:
1. Implement ring buffer for packet reception
2. Add overflow detection and handling
3. Measure packet loss rate at line rate
4. Optimize for cache efficiency

**Questions**:
- What buffer size minimizes packet loss?
- How do you handle buffer overflow?
- What is the cache miss rate?

---

### Exercise 2: Interrupt Handler Optimization

**Objective**: Minimize interrupt handler execution time.

**Task**:
1. Implement interrupt handler with linked list queue
2. Optimize using lock-free ring buffer
3. Measure interrupt latency
4. Measure worst-case execution time

**Questions**:
- How much faster is the ring buffer?
- What is the worst-case interrupt latency?
- Is it acceptable for real-time requirements?

---

## Chapter 19: Firmware Memory Management

### Exercise 1: Memory Pool Allocator

**Objective**: Eliminate fragmentation in firmware.

**Task**:
1. Implement fixed-size memory pools
2. Implement slab allocator for multiple sizes
3. Measure fragmentation over 72 hours
4. Compare with malloc

**Questions**:
- Does fragmentation occur with memory pools?
- What is the memory overhead?
- How many pool sizes do you need?

---

### Exercise 2: Long-Running Firmware Test

**Objective**: Ensure firmware stability over time.

**Task**:
1. Implement firmware with your memory allocator
2. Run continuous operation test for 72 hours
3. Monitor memory usage and fragmentation
4. Identify and fix any memory leaks

**Questions**:
- Does the firmware run for 72 hours without crashing?
- What is the memory usage trend over time?
- Are there any memory leaks?

---

## Chapter 20: Benchmark Case Studies

### Exercise 1: Dhrystone Analysis

**Objective**: Understand how compiler optimization affects Dhrystone scores.

**Task**:
1. Download Dhrystone 2.1 source code
2. Compile with different optimization levels: `-O0`, `-O1`, `-O2`, `-O3`
3. Compile with different compilers: GCC, Clang (if available)
4. Measure DMIPS/MHz for each configuration
5. Use `objdump -d` to examine the generated assembly code

**Questions**:
- How much do scores vary between optimization levels?
- How much do scores vary between compilers?
- Can you identify specific optimizations that inflate the score?
- Look at the assembly: is the compiler eliminating dead code?
- Why is Dhrystone considered obsolete?

**Expected Results**:
- 5-10× speedup from `-O0` to `-O3`
- 20-50% variance between compilers
- Evidence of constant propagation and dead code elimination

---

### Exercise 2: Coremark Implementation and Analysis

**Objective**: Run Coremark and understand what it measures.

**Task**:
1. Clone Coremark from GitHub: https://github.com/eembc/coremark
2. Compile for your platform (native x86/ARM or RISC-V QEMU)
3. Run with at least 10 seconds of iterations
4. Analyze the four workloads:
   - Linked list operations (core_list_join.c)
   - Matrix operations (core_matrix.c)
   - State machine (core_state.c)
   - CRC calculation (core_util.c)
5. Use `perf` to measure cache misses for each workload
6. Compile with different flags and compare scores

**Questions**:
- What is your CoreMark/MHz score?
- Which workload has the highest cache miss rate?
- Which workload takes the most time?
- How do compiler flags affect the score?
- Why can't the compiler optimize away Coremark like it does Dhrystone?

**Expected Results**:
- CoreMark/MHz between 2.5-5.5 (depending on processor)
- Linked list workload has highest cache miss rate
- Matrix workload takes most time
- `-O3` gives 10-30% improvement over `-O2`

**Advanced**:
- Modify Coremark to use different list sizes
- Measure how cache size affects performance
- Compare performance on different architectures (x86 vs ARM vs RISC-V)

---

### Exercise 3: Design Your Own Benchmark (Optional Challenge)

**Objective**: Apply benchmark design principles to create a domain-specific benchmark.

**Task**:
1. Choose a specific workload (e.g., packet processing, image filtering, crypto)
2. Identify the key operations in that workload
3. Design a benchmark that:
   - Uses runtime-determined inputs
   - Resists compiler optimization
   - Validates results
   - Represents realistic data sizes
4. Implement the benchmark
5. Test with different compilers and optimization levels
6. Document your methodology

**Questions**:
- What operations does your benchmark measure?
- How do you prevent dead code elimination?
- How do you validate correctness?
- What are the limitations of your benchmark?
- How does it compare to existing benchmarks?

**Example Workloads**:
- **Packet processing**: Parse headers, checksum, routing table lookup
- **Image filtering**: Convolution, color space conversion
- **Crypto**: AES encryption, SHA hashing
- **JSON parsing**: Tokenization, validation, tree building

**Deliverables**:
- Source code with clear documentation
- Run rules (iterations, validation, reporting)
- Benchmark results on at least one platform
- Analysis of what the benchmark measures and doesn't measure

---

## Submission Guidelines

For readers who want feedback on their implementations:

1. **Code**: Share your implementation on GitHub
2. **Benchmarks**: Include benchmark results with hardware specifications
3. **Analysis**: Write a brief analysis of your findings
4. **Discussion**: Join the book's discussion forum (URL TBD)

---

## Resources

- Benchmark framework: See Appendix A
- Hardware specifications: See Appendix B
- Profiling tools: See Appendix C
- Further reading: See Appendix D

