# Appendix D: Further Reading

This appendix provides curated resources for deeper exploration of hardware-aware programming, data structures, and performance optimization.

## Books

### Computer Architecture

**Computer Architecture: A Quantitative Approach** (6th Edition)  
*John L. Hennessy and David A. Patterson*  
Morgan Kaufmann, 2017

The definitive reference on computer architecture. Covers cache hierarchies, memory systems, pipelining, and performance analysis in depth.

**Relevant chapters**:
- Chapter 2: Memory Hierarchy Design
- Chapter 3: Instruction-Level Parallelism
- Appendix B: Review of Memory Hierarchy

---

**Modern Processor Design: Fundamentals of Superscalar Processors**  
*John Paul Shen and Mikko H. Lipasti*  
Waveland Press, 2013

Deep dive into modern processor microarchitecture, including out-of-order execution, branch prediction, and cache design.

**Relevant chapters**:
- Chapter 5: Memory Hierarchy
- Chapter 6: Cache Design
- Chapter 7: Virtual Memory

---

### Performance Optimization

**Systems Performance: Enterprise and the Cloud** (2nd Edition)  
*Brendan Gregg*  
Addison-Wesley, 2020

Comprehensive guide to performance analysis and optimization. Covers profiling tools, methodologies, and real-world case studies.

**Relevant chapters**:
- Chapter 6: CPUs
- Chapter 7: Memory
- Chapter 8: File Systems
- Chapter 9: Disks

---

**The Art of Writing Efficient Programs**  
*Fedor G. Pikus*  
Packt Publishing, 2021

Practical guide to writing high-performance C++ code. Covers cache optimization, branch prediction, and SIMD programming.

**Relevant chapters**:
- Chapter 2: Performance Measurements
- Chapter 3: CPU Architecture and Performance
- Chapter 4: Memory Architecture and Performance
- Chapter 5: Threads, Memory, and Concurrency

---

**Optimizing Software in C++**  
*Agner Fog*  
Free online resource, 2023  
https://www.agner.org/optimize/

Detailed manual on optimizing C++ code for x86/x64 processors. Covers instruction timing, cache optimization, and vectorization.

---

### Data Structures and Algorithms

**Introduction to Algorithms** (4th Edition)  
*Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest, and Clifford Stein*  
MIT Press, 2022

The classic algorithms textbook. Provides theoretical foundation for data structures and algorithms.

**Relevant chapters**:
- Chapter 10: Elementary Data Structures
- Chapter 11: Hash Tables
- Chapter 12: Binary Search Trees
- Chapter 13: Red-Black Trees
- Chapter 18: B-Trees

---

**The Art of Computer Programming, Volume 3: Sorting and Searching** (2nd Edition)  
*Donald E. Knuth*  
Addison-Wesley, 1998

Comprehensive treatment of sorting and searching algorithms. Mathematical and rigorous.

**Relevant sections**:
- Section 6.2: Searching by Comparison of Keys
- Section 6.3: Digital Searching
- Section 6.4: Hashing

---

**Cache-Oblivious Algorithms and Data Structures**
*Erik D. Demaine*
Lecture Notes in Advanced Data Structures (MIT 6.851), 2012

Theoretical foundation for algorithms that work well regardless of cache size. Covers cache-oblivious B-trees, matrix multiplication, and sorting.

**Relevant topics**:
- Van Emde Boas layout for trees
- Cache-oblivious B-trees
- Optimal I/O complexity

---

### Embedded Systems

**Embedded Systems Architecture** (2nd Edition)
*Tammy Noergaard*
Newnes, 2012

Practical guide to embedded systems design, including memory management and real-time constraints.

**Relevant chapters**:
- Chapter 4: Memory
- Chapter 5: I/O
- Chapter 7: Real-Time Operating Systems

---

**Programming Embedded Systems** (2nd Edition)  
*Michael Barr and Anthony Massa*  
O'Reilly Media, 2006

Hands-on guide to embedded programming in C. Covers bootloaders, device drivers, and memory management.

**Relevant chapters**:
- Chapter 5: Memory
- Chapter 6: Peripherals
- Chapter 8: Putting It All Together

---

## Papers

### Cache-Conscious Data Structures

**Cache-Conscious Data Structures**  
*Rao and Ross*  
SIGMOD 1999

Introduces cache-conscious B-trees and analyzes cache behavior of tree structures.

**Key insights**:
- B-tree node size should match cache line size
- Prefetching improves sequential access
- Cache-conscious layouts provide 2-5× speedup

---

**Cache-Oblivious Data Structures**  
*Frigo, Leiserson, Prokop, and Ramachandran*  
FOCS 1999

Introduces cache-oblivious algorithms that work well across all cache levels without tuning.

**Key insights**:
- Recursive divide-and-conquer naturally adapts to cache hierarchy
- Van Emde Boas layout for trees
- Optimal I/O complexity without knowing cache parameters

---

### Lock-Free Data Structures

**Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms**  
*Michael and Scott*  
PODC 1996

Classic paper on lock-free queues using compare-and-swap.

**Key insights**:
- Lock-free queues avoid contention
- ABA problem and solutions
- Memory ordering requirements

---

**The Art of Multiprocessor Programming** (2nd Edition)  
*Maurice Herlihy and Nir Shavit*  
Morgan Kaufmann, 2020

Comprehensive textbook on concurrent programming and lock-free data structures.

**Relevant chapters**:
- Chapter 7: Spin Locks and Contention
- Chapter 10: Concurrent Queues and the ABA Problem
- Chapter 11: Concurrent Stacks and Elimination

---

### Memory Allocation

**The Memory Fragmentation Problem: Solved?**  
*Wilson, Johnstone, Neely, and Boles*  
ISMM 1995

Survey of memory allocation algorithms and fragmentation analysis.

**Key insights**:
- Fragmentation is inevitable with general-purpose allocators
- Fixed-size pools eliminate fragmentation
- Segregated free lists reduce fragmentation

---

**Hoard: A Scalable Memory Allocator for Multithreaded Applications**  
*Berger, McKinley, Blumofe, and Wilson*  
ASPLOS 2000

Introduces Hoard, a scalable memory allocator that avoids false sharing.

**Key insights**:
- Per-thread heaps reduce contention
- Superblock-based allocation improves locality
- Provable bounds on fragmentation

---

## Online Resources

### Documentation

**Intel 64 and IA-32 Architectures Optimization Reference Manual**  
https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html

Official Intel optimization guide. Covers cache optimization, branch prediction, and SIMD programming.

---

**ARM Cortex-A Series Programmer's Guide**  
https://developer.arm.com/documentation/

ARM's official documentation for Cortex-A processors. Covers NEON, cache management, and performance optimization.

---

**RISC-V Specifications**  
https://riscv.org/technical/specifications/

Official RISC-V ISA specifications, including vector extension (RVV) and memory model (RVWMO).

---

### Blogs and Articles

**"What Every Programmer Should Know About Memory"**
*Ulrich Drepper*
https://people.freebsd.org/~lstewart/articles/cpumemory.pdf

Comprehensive article on memory hierarchy and cache behavior. Essential reading for understanding hardware-aware programming.

**Topics covered**:
- Memory hierarchy architecture
- Cache organization and behavior
- NUMA systems
- Memory performance optimization

---

**Brendan Gregg's Blog**
https://www.brendangregg.com/

Performance analysis expert. Covers profiling tools, flame graphs, and system performance.

**Recommended posts**:
- "CPU Flame Graphs"
- "Off-CPU Analysis"
- "perf Examples"

---

**Agner Fog's Optimization Resources**  
https://www.agner.org/optimize/

Comprehensive resources on x86/x64 optimization, including instruction tables and microarchitecture guides.

---

**Mechanical Sympathy**  
https://mechanical-sympathy.blogspot.com/

Martin Thompson's blog on hardware-aware programming. Covers cache coherence, false sharing, and lock-free programming.

**Recommended posts**:
- "Memory Barriers/Fences"
- "CPU Cache Flushing Fallacy"
- "False Sharing"

---

**Easyperf Blog**  
https://easyperf.net/

Performance analysis tutorials and case studies. Covers perf, cache optimization, and compiler optimizations.

**Recommended posts**:
- "Top-Down Microarchitecture Analysis"
- "Data-Driven Optimizations"
- "Cache-Friendly Code"

---

### Video Courses

**Performance Ninja Class**  
https://github.com/dendibakh/perf-ninja

Hands-on course on performance optimization. Includes exercises and solutions.

**Topics**:
- Cache optimization
- Branch prediction
- SIMD programming
- Profiling with perf

---

**CppCon Talks**  
https://www.youtube.com/user/CppCon

Annual C++ conference with many talks on performance optimization.

**Recommended talks**:
- "Efficiency with Algorithms, Performance with Data Structures" (Chandler Carruth)
- "There Are No Zero-Cost Abstractions" (Chandler Carruth)
- "The CPU Cache: Instruction Re-Ordering Made Obvious" (Andreas Fertig)

---

## Tools and Libraries

### Profiling Tools

**perf**  
https://perf.wiki.kernel.org/

Linux performance profiler. Essential tool for performance analysis.

---

**Valgrind**  
https://valgrind.org/

Suite of tools including cachegrind (cache profiler) and callgrind (call graph profiler).

---

**Intel VTune Profiler**  
https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html

Advanced profiler for Intel CPUs. Provides microarchitecture-level analysis.

---

**AMD uProf**  
https://developer.amd.com/amd-uprof/

Profiler for AMD CPUs. Similar to VTune for AMD processors.

---

### Benchmarking Libraries

**Google Benchmark**  
https://github.com/google/benchmark

C++ microbenchmarking library. Provides statistical analysis and comparison.

---

**Criterion**  
https://github.com/Snaipe/Criterion

C/C++ benchmarking library with statistical analysis.

---

### Data Structure Libraries

**Abseil**
https://abseil.io/

Google's C++ library with optimized data structures (flat_hash_map, etc.).

---

**Folly**
https://github.com/facebook/folly

Facebook's C++ library with high-performance data structures.

---

### Source Code Examples

**Linux Kernel List Implementation**
https://github.com/torvalds/linux/blob/master/include/linux/list.h

Intrusive doubly-linked lists used throughout the Linux kernel. Study how the kernel uses embedded list nodes for cache efficiency.

**Key files**:
- `include/linux/list.h` - List macros and inline functions
- `lib/list_sort.c` - List sorting implementation
- `kernel/sched/core.c` - Scheduler using lists

---

**FreeRTOS Source Code**
https://github.com/FreeRTOS/FreeRTOS-Kernel

Real-time operating system source code. See how RTOS uses linked lists for task scheduling and queue management.

**Key files**:
- `tasks.c` - Task scheduler implementation
- `queue.c` - Queue implementation
- `list.c` - List implementation

---

**jemalloc**  
https://github.com/jemalloc/jemalloc

Scalable memory allocator used by Firefox and FreeBSD.

---

**mimalloc**  
https://github.com/microsoft/mimalloc

Microsoft's high-performance allocator with excellent cache locality.

---

## Chapter-Specific Resources

This section provides curated resources for each chapter, organized by topic.

### Chapter 1: The Performance Gap

**Essential Reading**:
- "What Every Programmer Should Know About Memory" (Ulrich Drepper) - Sections 2-3 on cache hierarchy
- "Computer Architecture: A Quantitative Approach" (Hennessy & Patterson) - Chapter 2: Memory Hierarchy Design

**Papers**:
- "Hitting the Memory Wall: Implications of the Obvious" (Wulf & McKee, 1995)
- "Memory Performance and Scalability of Intel's and AMD's Dual-Core Processors" (Molka et al., 2009)

**Online Resources**:
- Gallery of Processor Cache Effects: https://igoro.com/archive/gallery-of-processor-cache-effects/
- Intel Optimization Manual: Section 2.1 on cache architecture

---

### Chapter 2: Memory Hierarchy

**Essential Reading**:
- "Computer Architecture: A Quantitative Approach" - Appendix B: Review of Memory Hierarchy
- "Modern Processor Design" (Shen & Lipasti) - Chapter 5: Memory Hierarchy

**Papers**:
- "The Memory Hierarchy is Dead: Long Live the Memory Hierarchy" (Burger et al., 2004)
- "Understanding the Backward Compatibility of Intel Processors" (Intel, 2019)

**Online Resources**:
- CPU Cache visualization: https://www.7-cpu.com/
- ARM Cortex-A Series Programmer's Guide - Chapter 8: Caches

---

### Chapter 3: Benchmarking and Profiling

**Essential Reading**:
- "Systems Performance" (Brendan Gregg) - Chapter 6: CPUs, Chapter 7: Memory
- "The Art of Writing Efficient Programs" (Fedor Pikus) - Chapter 2: Performance Measurements

**Papers**:
- "Statistically Rigorous Java Performance Evaluation" (Georges et al., 2007)
- "Producing Wrong Data Without Doing Anything Obviously Wrong!" (Mytkowicz et al., 2009)

**Online Resources**:
- perf Examples: https://www.brendangregg.com/perf.html
- Easyperf Blog: https://easyperf.net/blog/
- Performance Ninja Class: https://github.com/dendibakh/perf-ninja

---

### Chapter 4: Arrays and Cache Locality

**Essential Reading**:
- "Data-Oriented Design" (Richard Fabian) - Chapter 2: Hardware
- "The Art of Writing Efficient Programs" - Chapter 4: Memory Architecture and Performance

**Papers**:
- "Cache-Conscious Data Structures" (Rao & Ross, 1999)
- "Data Alignment: Straighten Up and Fly Right" (IBM developerWorks, 2004)

**Online Resources**:
- Mechanical Sympathy Blog: "CPU Cache Flushing Fallacy"
- Intel Optimization Manual: Section 3.6 on data alignment

---

### Chapter 5: Linked Lists - The Cache Killer

**Essential Reading**:
- "What Every Programmer Should Know About Memory" - Section 3.3 on pointer chasing
- Linux Kernel Documentation: Intrusive linked lists

**Papers**:
- "Cache Performance of Traversals and Random Accesses" (Chilimbi et al., 1999)
- "Memory Allocator Designs" (Wilson et al., 1995)

**Online Resources**:
- Linux Kernel list.h implementation
- FreeRTOS list.c source code
- "Why You Should Avoid Linked Lists" (Bjarne Stroustrup, Going Native 2012)

---

### Chapter 6: Stacks and Queues

**Essential Reading**:
- "Introduction to Algorithms" (CLRS) - Chapter 10: Elementary Data Structures
- "Embedded Systems Architecture" (Noergaard) - Chapter 4: Memory

**Papers**:
- "Implementing Lock-Free Queues" (Michael & Scott, 1996)
- "Ring Buffers and Queues" (Embedded Systems Programming, 2008)

**Online Resources**:
- Linux Kernel kfifo implementation
- Boost.Lockfree documentation

---

### Chapter 7: Hash Tables and Cache Conflicts

**Essential Reading**:
- "The Art of Computer Programming, Vol 3" (Knuth) - Section 6.4: Hashing
- "Introduction to Algorithms" (CLRS) - Chapter 11: Hash Tables

**Papers**:
- "Cache-Conscious Collision Resolution in String Hash Tables" (Askitis & Zobel, 2005)
- "Cuckoo Hashing" (Pagh & Rodler, 2004)

**Online Resources**:
- Google's Swiss Tables (Abseil): https://abseil.io/about/design/swisstables
- Facebook's F14 Hash Table: https://engineering.fb.com/2019/04/25/developer-tools/f14/

---

### Chapter 8: Dynamic Arrays and Memory Management

**Essential Reading**:
- "The C++ Programming Language" (Stroustrup) - Chapter 31: STL Containers
- "Effective STL" (Scott Meyers) - Item 14: Use reserve to avoid unnecessary reallocations

**Papers**:
- "The Memory Fragmentation Problem: Solved?" (Wilson et al., 1995)
- "Hoard: A Scalable Memory Allocator" (Berger et al., 2000)

**Online Resources**:
- jemalloc documentation: http://jemalloc.net/
- mimalloc paper: https://www.microsoft.com/en-us/research/publication/mimalloc-free-list-sharding-in-action/

---

### Chapter 9: Binary Search Trees

**Essential Reading**:
- "Introduction to Algorithms" (CLRS) - Chapter 12: Binary Search Trees, Chapter 13: Red-Black Trees
- "The Art of Computer Programming, Vol 3" (Knuth) - Section 6.2.3: Trees

**Papers**:
- "Cache-Oblivious Search Trees via Binary Trees of Small Height" (Bender et al., 2000)
- "Fast Set Operations Using Treaps" (Blelloch & Reid-Miller, 1998)

**Online Resources**:
- Red-Black Tree visualization: https://www.cs.usfca.edu/~galles/visualization/RedBlack.html
- Linux Kernel rbtree implementation

---

### Chapter 10: B-Trees and Cache-Conscious Trees

**Essential Reading**:
- "Introduction to Algorithms" (CLRS) - Chapter 18: B-Trees
- "Database System Concepts" (Silberschatz et al.) - Chapter 11: Indexing and Hashing

**Papers**:
- "Cache-Conscious Data Structures" (Rao & Ross, 1999) - Original B-tree cache analysis
- "The Adaptive Radix Tree" (Leis et al., 2013)
- "Cache-Oblivious B-Trees" (Bender et al., 2000)

**Online Resources**:
- SQLite B-tree implementation: https://www.sqlite.org/btreemodule.html
- BW-Tree (Microsoft): https://www.microsoft.com/en-us/research/publication/the-bw-tree-a-b-tree-for-new-hardware/

---

### Chapter 11: Tries and Radix Trees

**Essential Reading**:
- "Introduction to Algorithms" (CLRS) - Section 12.3: Radix Trees
- "The Art of Computer Programming, Vol 3" (Knuth) - Section 6.3: Digital Searching

**Papers**:
- "The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases" (Leis et al., 2013)
- "HAT-trie: A Cache-conscious Trie-based Data Structure" (Askitis & Sinha, 2007)
- "Judy Arrays" (Baskins, 2004)

**Online Resources**:
- Linux Kernel radix tree implementation
- Redis Rax (radix tree): https://github.com/antirez/rax

---

### Chapter 12: Heaps and Priority Queues

**Essential Reading**:
- "Introduction to Algorithms" (CLRS) - Chapter 6: Heapsort
- "The Art of Computer Programming, Vol 3" (Knuth) - Section 5.2.3: Sorting by Selection

**Papers**:
- "A Back-to-Basics Empirical Study of Priority Queues" (Larkin et al., 2014)
- "Cache-Oblivious Priority Queue and Graph Algorithm Applications" (Arge et al., 2005)
- "Fibonacci Heaps and Their Uses" (Fredman & Tarjan, 1987)

**Online Resources**:
- Linux Kernel heap implementation (lib/prio_heap.c)
- C++ std::priority_queue implementation notes

---

### Chapter 13: Lock-Free Data Structures

**Essential Reading**:
- "The Art of Multiprocessor Programming" (Herlihy & Shavit) - Chapters 7, 10, 11
- "C++ Concurrency in Action" (Anthony Williams) - Chapter 7: Lock-Free Data Structures

**Papers**:
- "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms" (Michael & Scott, 1996)
- "Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects" (Michael, 2004)
- "Epoch-Based Reclamation" (Fraser, 2004)

**Online Resources**:
- Boost.Lockfree documentation: https://www.boost.org/doc/libs/release/doc/html/lockfree.html
- Folly's lock-free structures: https://github.com/facebook/folly/tree/main/folly/concurrency
- 1024cores.net: http://www.1024cores.net/home/lock-free-algorithms

---

### Chapter 14: String Processing and Cache Efficiency

**Essential Reading**:
- "Flexible and Efficient Regular Expression Matching" (Russ Cox)
- "The Art of Computer Programming, Vol 3" (Knuth) - Section 6.3: Digital Searching

**Papers**:
- "Fast String Searching" (Boyer & Moore, 1977)
- "SIMD-friendly Algorithms for Substring Searching" (Kocsis et al., 2013)
- "Hyperscan: A Fast Multi-pattern Regex Matcher" (Wang et al., 2019)

**Online Resources**:
- Intel Hyperscan: https://www.hyperscan.io/
- SIMD string search examples: https://github.com/WojciechMula/sse4-strstr
- Cloudflare's string matching blog: https://blog.cloudflare.com/

---

### Chapter 15: Graphs and Cache-Efficient Traversal

**Essential Reading**:
- "Introduction to Algorithms" (CLRS) - Chapter 22: Elementary Graph Algorithms
- "Algorithm Design" (Kleinberg & Tardos) - Chapter 3: Graphs

**Papers**:
- "Cache-Oblivious Algorithms" (Frigo et al., 1999)
- "Graph Traversal in Compressed Space" (Asano et al., 2000)
- "Ligra: A Lightweight Graph Processing Framework" (Shun & Blelloch, 2013)

**Online Resources**:
- Boost Graph Library: https://www.boost.org/doc/libs/release/libs/graph/
- Graph500 benchmark: https://graph500.org/
- WebGraph framework: http://webgraph.di.unimi.it/

---

### Chapter 16: Bloom Filters and Probabilistic Data Structures

**Essential Reading**:
- "Probabilistic Data Structures and Algorithms" (Andrii Gakhov)
- "Randomized Algorithms" (Motwani & Raghavan) - Chapter 5

**Papers**:
- "Space/Time Trade-offs in Hash Coding with Allowable Errors" (Bloom, 1970) - Original paper
- "Network Applications of Bloom Filters: A Survey" (Broder & Mitzenmacher, 2004)
- "Cuckoo Filter: Practically Better Than Bloom" (Fan et al., 2014)
- "HyperLogLog: The Analysis of a Near-Optimal Cardinality Estimation Algorithm" (Flajolet et al., 2007)

**Online Resources**:
- Redis Bloom filter module: https://redis.io/docs/stack/bloom/
- Guava's Bloom filter: https://github.com/google/guava/wiki/HashingExplained

---

### Chapter 17: Bootloader Data Structures

**Essential Reading**:
- "Embedded Systems Architecture" (Noergaard) - Chapter 3: Boot Process
- "Programming Embedded Systems" (Barr & Massa) - Chapter 3: Bootloaders

**Papers**:
- "U-Boot: A Boot Loader for Embedded Systems" (Denx Software Engineering)
- "Device Tree Usage" (Linux Kernel Documentation)

**Online Resources**:
- U-Boot source code: https://github.com/u-boot/u-boot
- Device Tree Specification: https://www.devicetree.org/
- RISC-V SBI Specification: https://github.com/riscv-non-isa/riscv-sbi-doc

---

### Chapter 18: Device Driver Queues

**Essential Reading**:
- "Linux Device Drivers" (Corbet et al.) - Chapter 10: Interrupt Handling
- "Embedded Systems Architecture" (Noergaard) - Chapter 5: I/O

**Papers**:
- "The Linux Kernel: Networking" (Benvenuti, 2005)
- "NAPI: New API for Network Drivers" (Salim & Olsson, 2001)

**Online Resources**:
- Linux Kernel networking documentation
- DPDK (Data Plane Development Kit): https://www.dpdk.org/
- Intel IXGBE driver source code

---

### Chapter 19: Firmware Memory Management

**Essential Reading**:
- "Embedded Systems Architecture" (Noergaard) - Chapter 4: Memory
- "Programming Embedded Systems" (Barr & Massa) - Chapter 5: Memory

**Papers**:
- "The Memory Fragmentation Problem: Solved?" (Wilson et al., 1995)
- "TLSF: A New Dynamic Memory Allocator for Real-Time Systems" (Masmano et al., 2004)
- "A Memory Allocator for Embedded Systems" (Lea, 1996)

**Online Resources**:
- FreeRTOS heap implementations: https://www.freertos.org/a00111.html
- TLSF allocator: http://www.gii.upv.es/tlsf/
- Embedded Artistry's memory management: https://embeddedartistry.com/

---

### Chapter 20: Benchmark Case Studies

**Essential Reading**:
- "Dhrystone: A Synthetic Systems Programming Benchmark" (Weicker, 1984) - Original Dhrystone paper
- "CoreMark: A Simple Benchmark for Embedded Processors" (EEMBC, 2009) - Official Coremark documentation

**Papers**:
- "Benchmarking Embedded Processors: Myths and Realities" (Gal-On & Levy, 2003)
- "The Computer Benchmarking Handbook" (Weicker, 1990)
- "Performance Evaluation and Benchmarking" (Huppler, 2009)

**Online Resources**:
- EEMBC CoreMark: https://www.eembc.org/coremark/
- CoreMark GitHub: https://github.com/eembc/coremark
- SPEC Benchmarks: https://www.spec.org/
- Dhrystone source code and analysis: https://fossies.org/linux/privat/old/dhrystone-2.1.tar.gz/

**Benchmark Design**:
- "How to Lie with Benchmarks" (Fleming & Wallace, 1986)
- "Benchmarking: An Overview" (Lilja, 2000)
- "The Art of Computer Systems Performance Analysis" (Jain, 1991)

**RISC-V Specific**:
- RISC-V Benchmarks: https://github.com/riscv-boom/riscv-benchmarks
- Embench: https://www.embench.org/ - Modern embedded benchmark suite
- RISC-V Performance Analysis: https://riscv.org/technical/specifications/

**Compiler Optimization**:
- "Optimizing Compilers for Modern Architectures" (Allen & Kennedy, 2001)
- GCC Optimization Options: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html
- LLVM Optimization Guide: https://llvm.org/docs/Passes.html

**Key Insights**:
- Dhrystone is obsolete due to compiler optimization vulnerabilities
- Coremark represents diverse workloads: lists, matrices, state machines, CRC
- Good benchmarks resist dead code elimination and use runtime-determined inputs
- Benchmark scores are tools for analysis, not goals for optimization
- Always disclose full methodology: hardware, compiler, flags, run rules

**Practical Resources**:
- How to run Coremark on RISC-V: https://github.com/eembc/coremark/blob/main/barebones_porting.md
- Benchmark validation and result submission: https://www.eembc.org/coremark/submit.php
- Statistical analysis of benchmark results (Chapter 3 techniques apply)

---

## Summary

This appendix provides resources for further exploration:

**Books**:
- Computer architecture: Hennessy & Patterson
- Performance optimization: Brendan Gregg, Fedor Pikus
- Data structures: CLRS, Knuth
- Embedded systems: Barr & Massa

**Papers**:
- Cache-conscious data structures (Rao & Ross)
- Lock-free algorithms (Michael & Scott)
- Memory allocation (Wilson et al., Berger et al.)

**Online resources**:
- Intel/ARM/RISC-V documentation
- Blogs: Brendan Gregg, Agner Fog, Mechanical Sympathy
- Video courses: Performance Ninja, CppCon

**Tools**:
- Profiling: perf, Valgrind, VTune
- Benchmarking: Google Benchmark, Criterion
- Libraries: Abseil, Folly, jemalloc

**Chapter-Specific Resources**:
- Each chapter now has curated papers, books, and online resources
- Focus on both theoretical foundations and practical implementations
- Mix of classic papers and modern research

**Next steps**:
1. Read Hennessy & Patterson for architecture fundamentals
2. Study Brendan Gregg's blog for profiling techniques
3. Practice with Performance Ninja exercises
4. Experiment with the benchmark framework from Appendix A

Happy optimizing!
