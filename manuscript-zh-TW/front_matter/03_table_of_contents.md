# 目錄

## Front Matter

- 封面
- 版權與授權
- 序言
- 目錄

## Part I: Foundations

### Chapter 1: The Performance Gap
1.1 The 2:00 AM Mystery  
1.2 Big-O vs Reality  
1.3 Memory Hierarchy Basics  
1.4 Cache Behavior  
1.5 First Benchmark  
1.6 Summary

### Chapter 2: Memory Hierarchy
2.1 The 100-Cycle Problem  
2.2 Cache Organization  
2.3 Spatial and Temporal Locality  
2.4 Cache Lines and Prefetching  
2.5 Set-Associative Caches  
2.6 MESI Protocol and False Sharing  
2.7 Memory Bandwidth  
2.8 RISC-V Memory Model  
2.9 Summary

### Chapter 3: Benchmarking and Profiling
3.1 The Measurement Problem  
3.2 High-Precision Timing  
3.3 Statistical Analysis  
3.4 Hardware Performance Counters  
3.5 The perf Tool  
3.6 Common Pitfalls  
3.7 Embedded Considerations  
3.8 Summary

## Part II: Basic Data Structures

### Chapter 4: Arrays and Cache Locality
4.1 The Simplest Data Structure  
4.2 Sequential vs Random Access  
4.3 Stride Patterns  
4.4 Matrix Traversal  
4.5 Structure of Arrays vs Array of Structures  
4.6 Alignment and Padding  
4.7 Hot/Cold Data Separation  
4.8 Packet Buffer Optimization  
4.9 Guidelines  
4.10 Summary

### Chapter 5: Linked Lists - The Cache Killer
5.1 The Textbook Story  
5.2 Reality Check: Benchmarks  
5.3 Why Linked Lists Are Slow  
5.4 Memory Overhead  
5.5 When to Use Linked Lists  
5.6 Optimization Strategies  
5.7 Summary

### Chapter 6: Stacks and Queues
6.1 The Invisible Data Structure  
6.2 Array-Based Stack  
6.3 Ring Buffer Queue  
6.4 Lock-Free Queues  
6.5 Priority Queues  
6.6 ISR-Safe Design  
6.7 Task Scheduler Case Study  
6.8 Summary

### Chapter 7: Hash Tables and Cache Conflicts
7.1 The O(1) Myth  
7.2 Chaining vs Open Addressing  
7.3 Hash Function Quality  
7.4 Cache-Friendly Design  
7.5 Robin Hood Hashing  
7.6 Perfect Hashing  
7.7 Symbol Table Optimization  
7.8 Load Factor Considerations  
7.9 Guidelines  
7.10 Summary

### Chapter 8: Dynamic Arrays and Memory Management
8.1 The Reallocation Problem  
8.2 Exponential Growth Strategy  
8.3 Reserve and Capacity  
8.4 Small Vector Optimization  
8.5 Memory Allocator Considerations  
8.6 Gap Buffer for Text Editing  
8.7 Fixed-Capacity Vectors  
8.8 Log Buffer Case Study  
8.9 Summary

## Part III: Trees and Hierarchies

### Chapter 9: Binary Search Trees
9.1 Red-Black Tree Disaster  
9.2 BST vs Sorted Array  
9.3 Cache Miss Analysis  
9.4 Tree Layout Optimization  
9.5 Array-Based Trees  
9.6 van Emde Boas Layout  
9.7 When to Use Trees  
9.8 Guidelines  
9.9 Summary

### Chapter 10: B-Trees and Cache-Conscious Trees
10.1 Database Mystery  
10.2 B-Tree Fundamentals  
10.3 Optimal Node Size  
10.4 In-Memory B-Trees  
10.5 Cache-Oblivious Algorithms  
10.6 B-Tree vs Hash Table  
10.7 Implementation Considerations  
10.8 Guidelines  
10.9 Summary

### Chapter 11: Tries and Radix Trees
11.1 Autocomplete Disaster  
11.2 Trie Fundamentals  
11.3 Memory Consumption  
11.4 Radix Trees (Compressed Tries)  
11.5 Array-Mapped Tries  
11.6 Adaptive Radix Trees  
11.7 Use Cases  
11.8 Guidelines  
11.9 Summary

### Chapter 12: Heaps and Priority Queues
12.1 Scheduler Debate  
12.2 Binary Heap Fundamentals  
12.3 d-ary Heaps  
12.4 Cache Behavior  
12.5 Worst-Case Timing  
12.6 Real-Time Considerations  
12.7 Fibonacci Heaps  
12.8 Guidelines  
12.9 Summary

## Part IV: Advanced Topics

### Chapter 13: Lock-Free Data Structures
13.1 The 60% Problem  
13.2 Lock Contention  
13.3 Compare-And-Swap (CAS)  
13.4 ABA Problem  
13.5 Memory Ordering  
13.6 Lock-Free Queue  
13.7 Lock-Free Stack  
13.8 Hazard Pointers  
13.9 Performance Considerations  
13.10 Guidelines  
13.11 Summary

### Chapter 14: String Processing and Cache Efficiency
14.1 Throughput Gap  
14.2 String Search Algorithms  
14.3 Cache-Friendly Parsing  
14.4 SIMD Optimization  
14.5 Boyer-Moore Algorithm  
14.6 Log Parser Case Study  
14.7 Guidelines  
14.8 Summary

### Chapter 15: Graphs and Cache-Efficient Traversal
15.1 Cache Miss Explosion  
15.2 Graph Representations  
15.3 Adjacency List vs Array  
15.4 CSR Format  
15.5 BFS and DFS  
15.6 Cache-Oblivious Traversal  
15.7 Prefetching  
15.8 Guidelines  
15.9 Summary

### Chapter 16: Bloom Filters and Probabilistic Data Structures
16.1 Memory Crisis  
16.2 Bloom Filter Fundamentals  
16.3 False Positive Rate  
16.4 Hash Function Selection  
16.5 Cache-Friendly Implementation  
16.6 Counting Bloom Filters  
16.7 HyperLogLog  
16.8 Use Cases  
16.9 Summary

## Part V: Case Studies

### Chapter 17: Bootloader Data Structures
17.1 The 500ms Deadline  
17.2 Bootloader Constraints  
17.3 Fixed-Size Structures  
17.4 Device Tree Parsing  
17.5 Symbol Table  
17.6 Memory-Constrained Design  
17.7 Optimization Results  
17.8 Summary

### Chapter 18: Device Driver Queues
18.1 Packet Loss Mystery  
18.2 DMA Ring Buffers  
18.3 Interrupt Handler Design  
18.4 Lock-Free Techniques  
18.5 Cache Alignment  
18.6 Performance Tuning  
18.7 Debugging  
18.8 Guidelines  
18.9 Summary

### Chapter 19: Firmware Memory Management
19.1 The 72-Hour Test Failure  
19.2 Memory Fragmentation  
19.3 Fixed-Size Pools  
19.4 Slab Allocators  
19.5 Memory Leak Detection  
19.6 Long-Term Stability  
19.7 Best Practices  
19.8 Guidelines  
19.9 Summary

### Chapter 20: Benchmark Case Studies
20.1 The Dhrystone Trap  
20.2 Why Dhrystone is Obsolete  
20.3 CoreMark: A Better Benchmark  
20.4 Designing Meaningful Benchmarks  
20.5 Compiler Optimization Resistance  
20.6 Result Validation  
20.7 Case Study: Custom Benchmark  
20.8 Guidelines  
20.9 Summary

## 附錄

### Appendix A: Benchmark Framework Reference
A.1 High-Precision Timing  
A.2 Statistical Analysis  
A.3 perf Integration  
A.4 Benchmark Design Patterns  
A.5 Common Pitfalls  
A.6 Example Benchmarks  

### Appendix B: Hardware Reference
B.1 Cache Hierarchy  
B.2 Memory Latency Numbers  
B.3 RISC-V Architecture  
B.4 x86 Architecture  
B.5 ARM Architecture  
B.6 Atomic Operations  

### Appendix C: Tool Reference
C.1 perf  
C.2 Valgrind  
C.3 Intel VTune  
C.4 gprof  
C.5 Custom Tools  
C.6 Visualization  

### Appendix D: Further Reading
D.1 Chapter-Specific Resources (Chapters 1-20)  
D.2 Books  
D.3 Papers  
D.4 Online Resources  
D.5 Open Source Projects  

### Appendix E: Exercises
E.1 Chapter 1 Exercises  
E.2 Chapter 2 Exercises  
E.3 Chapter 3 Exercises  
E.4 Chapter 4 Exercises  
E.5 Chapter 5 Exercises  
E.6 Chapter 6 Exercises  
E.7 Chapter 7 Exercises  
E.8 Chapter 8 Exercises  
E.9 Chapter 9 Exercises  
E.10 Chapter 10 Exercises  
E.11 Chapter 11 Exercises  
E.12 Chapter 12 Exercises  
E.13 Chapter 13 Exercises  
E.14 Chapter 14 Exercises  
E.15 Chapter 15 Exercises  
E.16 Chapter 16 Exercises  
E.17 Chapter 17 Exercises  
E.18 Chapter 18 Exercises  
E.19 Chapter 19 Exercises  
E.20 Chapter 20 Exercises  
E.21 Submission Guidelines  

### Appendix F: Exercise Solutions
F.1 Selected Solutions  
F.2 Implementation Details  
F.3 Performance Analysis  

## Back Matter

- 關於作者
- 參考文獻

