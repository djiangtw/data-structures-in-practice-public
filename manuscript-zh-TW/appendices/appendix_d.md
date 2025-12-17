# Appendix D: Further Reading

這個附錄提供 hardware-aware programming、data structures 和 performance optimization 的精選資源，供深入探索。

## Books

### Computer Architecture

**Computer Architecture: A Quantitative Approach** (6th Edition)  
*John L. Hennessy and David A. Patterson*  
Morgan Kaufmann, 2017

Computer architecture 的權威參考。深入涵蓋 cache hierarchies、memory systems、pipelining 和 performance analysis。

**Relevant chapters**:
- Chapter 2: Memory Hierarchy Design
- Chapter 3: Instruction-Level Parallelism
- Appendix B: Review of Memory Hierarchy

---

**Modern Processor Design: Fundamentals of Superscalar Processors**  
*John Paul Shen and Mikko H. Lipasti*  
Waveland Press, 2013

深入探討現代 processor microarchitecture，包括 out-of-order execution、branch prediction 和 cache design。

**Relevant chapters**:
- Chapter 5: Memory Hierarchy
- Chapter 6: Cache Design
- Chapter 7: Virtual Memory

---

### Performance Optimization

**Systems Performance: Enterprise and the Cloud** (2nd Edition)  
*Brendan Gregg*  
Addison-Wesley, 2020

Performance analysis 和 optimization 的全面指南。涵蓋 profiling tools、methodologies 和真實世界 case studies。

**Relevant chapters**:
- Chapter 6: CPUs
- Chapter 7: Memory
- Chapter 8: File Systems
- Chapter 9: Disks

---

**The Art of Writing Efficient Programs**  
*Fedor G. Pikus*  
Packt Publishing, 2021

撰寫高效能 C++ code 的實用指南。涵蓋 cache optimization、branch prediction 和 SIMD programming。

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

優化 x86/x64 processors 的 C++ code 的詳細手冊。涵蓋 instruction timing、cache optimization 和 vectorization。

---

### Data Structures and Algorithms

**Introduction to Algorithms** (4th Edition)  
*Thomas H. Cormen, Charles E. Leiserson, Ronald L. Rivest, and Clifford Stein*  
MIT Press, 2022

經典 algorithms 教科書。提供 data structures 和 algorithms 的理論基礎。

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

Sorting 和 searching algorithms 的全面處理。數學且嚴謹。

**Relevant sections**:
- Section 6.2: Searching by Comparison of Keys
- Section 6.3: Digital Searching
- Section 6.4: Hashing

---

**Cache-Oblivious Algorithms and Data Structures**
*Erik D. Demaine*
Lecture Notes in Advanced Data Structures (MIT 6.851), 2012

無論 cache size 如何都能良好運作的 algorithms 的理論基礎。涵蓋 cache-oblivious B-trees、matrix multiplication 和 sorting。

**Relevant topics**:
- Van Emde Boas layout for trees
- Cache-oblivious B-trees
- Optimal I/O complexity

---

### Embedded Systems

**Embedded Systems Architecture** (2nd Edition)
*Tammy Noergaard*
Newnes, 2012

Embedded systems design 的實用指南，包括 memory management 和 real-time constraints。

**Relevant chapters**:
- Chapter 4: Memory
- Chapter 5: I/O
- Chapter 7: Real-Time Operating Systems

---

**Programming Embedded Systems** (2nd Edition)  
*Michael Barr and Anthony Massa*  
O'Reilly Media, 2006

C 語言 embedded programming 的實作指南。涵蓋 bootloaders、device drivers 和 memory management。

**Relevant chapters**:
- Chapter 5: Memory
- Chapter 6: Peripherals
- Chapter 8: Putting It All Together

---

## Papers

### Cache-Conscious Data Structures

**Cache-Conscious Data Structures**  
*Rao and Ross*  
ACM SIGMOD, 1999

開創性論文，展示 cache-conscious B-trees 如何大幅改善 performance。

**Key contributions**:
- CSR (Cache-Sensitive Radix) trees
- Cache-conscious B+ trees
- Experimental validation on real hardware

---

**Making Data Structures Persistent**  
*James R. Driscoll, Neil Sarnak, Daniel D. Sleator, Robert E. Tarjan*  
Journal of Computer and System Sciences, 1989

Persistent data structures 的理論基礎。

---

### Lock-Free Data Structures

**Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms**  
*Maged M. Michael and Michael L. Scott*  
PODC, 1996

Lock-free queue 的經典論文。

**Key contributions**:
- CAS-based lock-free queue
- ABA problem solution
- Performance comparison with locks

---

**Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects**  
*Maged M. Michael*  
IEEE TPDS, 2004

Lock-free data structures 的 safe memory reclamation。

---

### Benchmarking

**SPEC CPU Benchmarks**  
*Standard Performance Evaluation Corporation*  
https://www.spec.org/cpu2017/

Industry-standard CPU benchmarks。

---

**CoreMark: A Simple Benchmark for Embedded Systems**  
*Shay Gal-On*  
EEMBC, 2009

Embedded systems 的現代 benchmark。

---

## Online Resources

### Tutorials and Guides

**What Every Programmer Should Know About Memory**  
*Ulrich Drepper*  
https://people.freebsd.org/~lstewart/articles/cpumemory.pdf

Memory hierarchies 和 cache optimization 的全面指南。

---

**Gallery of Processor Cache Effects**  
*Igor Ostrovsky*  
http://igoro.com/archive/gallery-of-processor-cache-effects/

Interactive demonstrations of cache effects。

---

### Documentation

**Intel 64 and IA-32 Architectures Optimization Reference Manual**  
https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html

Intel processors 的官方 optimization guide。

---

**ARM Cortex-A Series Programmer's Guide**  
https://developer.arm.com/documentation/

ARM processors 的官方文檔。

---

**RISC-V Specifications**  
https://riscv.org/specifications/

RISC-V ISA 和 extensions 的官方規格。

---

## Courses

**MIT 6.172: Performance Engineering of Software Systems**  
https://ocw.mit.edu/courses/6-172-performance-engineering-of-software-systems-fall-2018/

MIT 的 performance engineering 課程。涵蓋 cache optimization、parallelism 和 profiling。

---

**Stanford CS107: Computer Organization and Systems**  
https://web.stanford.edu/class/cs107/

Stanford 的 systems programming 課程。

---

## Tools and Libraries

**Google Benchmark**  
https://github.com/google/benchmark

C++ microbenchmarking library。

---

**Folly**  
https://github.com/facebook/folly

Facebook 的 C++ library，包含 cache-efficient data structures。

---

**jemalloc**  
https://github.com/jemalloc/jemalloc

High-performance memory allocator。

---

## Blogs and Articles

**Brendan Gregg's Blog**  
https://www.brendangregg.com/

Performance analysis 和 profiling 的深入文章。

---

**Mechanical Sympathy**  
https://mechanical-sympathy.blogspot.com/

Hardware-aware programming 的討論。

---

**Agner Fog's Optimization Resources**  
https://www.agner.org/optimize/

x86/x64 optimization 的全面資源。

---

## Summary

這些資源涵蓋：

**Theory**: Computer architecture, algorithms, complexity analysis  
**Practice**: Profiling tools, optimization techniques, real-world case studies  
**Hardware**: x86-64, ARM, RISC-V architectures  
**Software**: Data structures, memory management, concurrency

**建議學習路徑**：
1. 從 Hennessy & Patterson 開始理解 architecture fundamentals
2. 用 Brendan Gregg 的書學習 profiling 和 analysis
3. 讀 Rao & Ross 的論文理解 cache-conscious design
4. 用 MIT 6.172 課程實作 optimization techniques
5. 探索 Agner Fog 的資源深入了解特定 architectures

---
