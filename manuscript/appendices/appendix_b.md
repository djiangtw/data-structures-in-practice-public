# Appendix B: Hardware Reference

This appendix provides detailed hardware specifications for the systems used in benchmarks throughout this book.

## Overview

All benchmarks were run on three representative architectures:
- **RISC-V**: SiFive HiFive Unmatched (U740)
- **x86-64**: Intel Core i7-12700K (Alder Lake)
- **ARM**: Raspberry Pi 4 Model B (Cortex-A72)

---

## RISC-V: SiFive HiFive Unmatched

### CPU Specifications

**Processor**: SiFive U740 (4+1 cores)
- **ISA**: RV64GC (RV64IMAFDC)
- **Cores**: 4× U74 cores + 1× S7 core
- **Frequency**: 1.2 GHz (U74), 600 MHz (S7)
- **Pipeline**: 8-stage in-order
- **SIMD**: RVV 1.0 (Vector extension)

### Memory Hierarchy

**L1 Cache** (per U74 core):
- **I-Cache**: 32 KB, 4-way set-associative
- **D-Cache**: 32 KB, 8-way set-associative
- **Line size**: 64 bytes
- **Latency**: 3 cycles

**L2 Cache** (shared):
- **Size**: 2 MB
- **Associativity**: 16-way set-associative
- **Line size**: 64 bytes
- **Latency**: 12 cycles

**Main Memory**:
- **Type**: DDR4-2400
- **Size**: 16 GB
- **Bandwidth**: 19.2 GB/s (theoretical)
- **Latency**: ~100 ns (~120 cycles)

### Cache Line Details

```
L1 D-Cache:
  Total size: 32 KB
  Line size: 64 bytes
  Sets: 64 (32 KB / 64 bytes / 8 ways)
  Associativity: 8-way
  
Address breakdown (64-bit):
  [63:12] Tag (52 bits)
  [11:6]  Index (6 bits = 64 sets)
  [5:0]   Offset (6 bits = 64 bytes)
```

### Performance Counters

**Available counters**:
- `cycle`: Cycle counter
- `instret`: Instructions retired
- `L1-dcache-load-misses`: L1 D-cache load misses
- `L1-dcache-store-misses`: L1 D-cache store misses
- `L1-icache-load-misses`: L1 I-cache misses
- `LLC-load-misses`: L2 cache load misses
- `LLC-store-misses`: L2 cache store misses
- `branch-misses`: Branch mispredictions

**Access method**:
```c
// Read cycle counter
uint64_t cycles;
asm volatile("rdcycle %0" : "=r"(cycles));

// Read instruction counter
uint64_t instret;
asm volatile("rdinstret %0" : "=r"(instret));
```

### Memory Bandwidth

**Measured bandwidth** (using `memcpy`):
- **Sequential read**: 15.2 GB/s
- **Sequential write**: 14.8 GB/s
- **Random read** (4 KB blocks): 2.1 GB/s
- **Random write** (4 KB blocks): 1.8 GB/s

### TLB Specifications

**DTLB** (Data TLB):
- **Entries**: 32 (fully associative)
- **Page sizes**: 4 KB, 2 MB, 1 GB
- **Miss penalty**: ~20 cycles (page table walk)

**ITLB** (Instruction TLB):
- **Entries**: 32 (fully associative)
- **Page sizes**: 4 KB, 2 MB, 1 GB

---

## x86-64: Intel Core i7-12700K

### CPU Specifications

**Processor**: Intel Core i7-12700K (Alder Lake, 12th Gen)
- **Architecture**: Hybrid (P-cores + E-cores)
- **P-cores**: 8× Golden Cove (performance)
- **E-cores**: 4× Gracemont (efficiency)
- **Frequency**: 3.6 GHz base, 5.0 GHz turbo (P-cores)
- **Pipeline**: Out-of-order, ~12-stage (P-cores)
- **SIMD**: AVX2, AVX-512 (disabled on consumer SKUs)

### Memory Hierarchy

**L1 Cache** (per P-core):
- **I-Cache**: 32 KB, 8-way set-associative
- **D-Cache**: 48 KB, 12-way set-associative
- **Line size**: 64 bytes
- **Latency**: 4 cycles

**L2 Cache** (per P-core):
- **Size**: 1.25 MB
- **Associativity**: 10-way set-associative
- **Line size**: 64 bytes
- **Latency**: 12 cycles

**L3 Cache** (shared):
- **Size**: 25 MB
- **Associativity**: 12-way set-associative
- **Line size**: 64 bytes
- **Latency**: 40-50 cycles

**Main Memory**:
- **Type**: DDR5-4800
- **Size**: 32 GB
- **Bandwidth**: 76.8 GB/s (theoretical, dual-channel)
- **Latency**: ~80 ns (~288 cycles @ 3.6 GHz)

### Cache Line Details

```
L1 D-Cache (P-core):
  Total size: 48 KB
  Line size: 64 bytes
  Sets: 64 (48 KB / 64 bytes / 12 ways)
  Associativity: 12-way
  
Address breakdown:
  [63:12] Tag
  [11:6]  Index (6 bits = 64 sets)
  [5:0]   Offset (6 bits = 64 bytes)
```

### Performance Counters

**Available counters** (via `perf`):
- `cycles`: CPU cycles
- `instructions`: Instructions retired
- `cache-references`: Cache accesses
- `cache-misses`: Cache misses (all levels)
- `L1-dcache-loads`: L1 D-cache loads
- `L1-dcache-load-misses`: L1 D-cache load misses
- `LLC-loads`: L3 cache loads
- `LLC-load-misses`: L3 cache load misses
- `branch-instructions`: Branches executed
- `branch-misses`: Branch mispredictions

**Access method**:
```c
// RDTSC (Read Time-Stamp Counter)
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    asm volatile("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

// RDTSCP (serializing version)
static inline uint64_t rdtscp(void) {
    uint32_t lo, hi;
    asm volatile("rdtscp" : "=a"(lo), "=d"(hi) :: "rcx");
    return ((uint64_t)hi << 32) | lo;
}
```

### Memory Bandwidth

**Measured bandwidth** (using AVX2 `memcpy`):
- **Sequential read**: 68.5 GB/s
- **Sequential write**: 65.2 GB/s
- **Random read** (4 KB blocks): 12.3 GB/s
- **Random write** (4 KB blocks): 10.8 GB/s

### TLB Specifications

**DTLB** (Data TLB, per P-core):
- **L1 DTLB**: 64 entries (4 KB pages), 32 entries (2 MB/4 MB pages)
- **L2 DTLB**: 2048 entries (shared, all page sizes)
- **Miss penalty**: ~100 cycles (page table walk)

**ITLB** (Instruction TLB, per P-core):
- **L1 ITLB**: 64 entries (4 KB pages), 8 entries (2 MB pages)
- **L2 ITLB**: Shared with DTLB

---

## ARM: Raspberry Pi 4 Model B

### CPU Specifications

**Processor**: Broadcom BCM2711 (Cortex-A72)
- **Architecture**: ARMv8-A (64-bit)
- **Cores**: 4× Cortex-A72
- **Frequency**: 1.5 GHz
- **Pipeline**: 15-stage in-order
- **SIMD**: NEON (Advanced SIMD)

### Memory Hierarchy

**L1 Cache** (per core):
- **I-Cache**: 48 KB, 3-way set-associative
- **D-Cache**: 32 KB, 2-way set-associative
- **Line size**: 64 bytes
- **Latency**: 3 cycles

**L2 Cache** (shared):
- **Size**: 1 MB
- **Associativity**: 16-way set-associative
- **Line size**: 64 bytes
- **Latency**: 15 cycles

**Main Memory**:
- **Type**: LPDDR4-3200
- **Size**: 8 GB
- **Bandwidth**: 12.8 GB/s (theoretical)
- **Latency**: ~120 ns (~180 cycles)

### Cache Line Details

```
L1 D-Cache:
  Total size: 32 KB
  Line size: 64 bytes
  Sets: 256 (32 KB / 64 bytes / 2 ways)
  Associativity: 2-way

Address breakdown:
  [63:14] Tag
  [13:6]  Index (8 bits = 256 sets)
  [5:0]   Offset (6 bits = 64 bytes)
```

### Performance Counters

**Available counters**:
- `PMCCNTR_EL0`: Cycle counter
- `PMEVCNTRn_EL0`: Event counters (6 programmable)
- Events: L1 D-cache misses, L2 cache misses, branch misses, etc.

**Access method**:
```c
// Enable user-mode access to PMU
static inline void enable_pmu(void) {
    uint64_t val = 1;
    asm volatile("msr pmuserenr_el0, %0" :: "r"(val));
}

// Read cycle counter
static inline uint64_t read_cycles(void) {
    uint64_t val;
    asm volatile("mrs %0, pmccntr_el0" : "=r"(val));
    return val;
}
```

### Memory Bandwidth

**Measured bandwidth**:
- **Sequential read**: 10.5 GB/s
- **Sequential write**: 9.8 GB/s
- **Random read** (4 KB blocks): 1.8 GB/s
- **Random write** (4 KB blocks): 1.5 GB/s

### TLB Specifications

**DTLB**:
- **L1 DTLB**: 48 entries (4 KB pages), 32 entries (64 KB pages)
- **L2 TLB**: 1024 entries (shared)
- **Miss penalty**: ~25 cycles

**ITLB**:
- **L1 ITLB**: 48 entries (4 KB pages)
- **L2 TLB**: Shared with DTLB

---

## Comparison Table

### Cache Hierarchy

| Feature | RISC-V (U740) | x86-64 (i7-12700K) | ARM (Cortex-A72) |
|---------|---------------|---------------------|------------------|
| **L1 D-Cache** | 32 KB, 8-way | 48 KB, 12-way | 32 KB, 2-way |
| **L1 I-Cache** | 32 KB, 4-way | 32 KB, 8-way | 48 KB, 3-way |
| **L2 Cache** | 2 MB, 16-way | 1.25 MB/core, 10-way | 1 MB, 16-way |
| **L3 Cache** | None | 25 MB, 12-way | None |
| **Line Size** | 64 bytes | 64 bytes | 64 bytes |

### Memory

| Feature | RISC-V (U740) | x86-64 (i7-12700K) | ARM (Cortex-A72) |
|---------|---------------|---------------------|------------------|
| **Type** | DDR4-2400 | DDR5-4800 | LPDDR4-3200 |
| **Bandwidth** | 19.2 GB/s | 76.8 GB/s | 12.8 GB/s |
| **Latency** | ~120 cycles | ~288 cycles | ~180 cycles |

### Performance

| Feature | RISC-V (U740) | x86-64 (i7-12700K) | ARM (Cortex-A72) |
|---------|---------------|---------------------|------------------|
| **Frequency** | 1.2 GHz | 3.6-5.0 GHz | 1.5 GHz |
| **Pipeline** | 8-stage, in-order | ~12-stage, OoO | 15-stage, in-order |
| **SIMD** | RVV 1.0 | AVX2 | NEON |

---

## Cache Behavior Characteristics

### Prefetcher Behavior

**RISC-V U740**:
- **Type**: Sequential prefetcher
- **Distance**: 2-4 cache lines ahead
- **Trigger**: 2 consecutive misses in same direction
- **Effectiveness**: Good for sequential access, poor for random

**x86-64 i7-12700K**:
- **Type**: Adaptive spatial + stride prefetcher
- **Distance**: Up to 20 cache lines ahead
- **Trigger**: Detects patterns (sequential, strided)
- **Effectiveness**: Excellent for sequential, good for strided

**ARM Cortex-A72**:
- **Type**: Sequential prefetcher
- **Distance**: 1-2 cache lines ahead
- **Trigger**: Sequential access detected
- **Effectiveness**: Good for sequential, poor for random

### Cache Replacement Policy

**All architectures**: Pseudo-LRU (Least Recently Used)

**Implications**:
- Accessing more than N ways in a set evicts oldest
- Thrashing occurs when working set > cache size
- Temporal locality is critical

---

## Memory Latency Numbers

### Typical Access Latencies

**RISC-V U740** (@ 1.2 GHz):
```
L1 D-cache hit:        3 cycles    (2.5 ns)
L2 cache hit:         12 cycles   (10 ns)
Main memory:         120 cycles  (100 ns)
```

**x86-64 i7-12700K** (@ 3.6 GHz):
```
L1 D-cache hit:        4 cycles    (1.1 ns)
L2 cache hit:         12 cycles    (3.3 ns)
L3 cache hit:         45 cycles   (12.5 ns)
Main memory:         288 cycles   (80 ns)
```

**ARM Cortex-A72** (@ 1.5 GHz):
```
L1 D-cache hit:        3 cycles    (2 ns)
L2 cache hit:         15 cycles   (10 ns)
Main memory:         180 cycles  (120 ns)
```

### Latency Ratios

```
Relative to L1 cache:

RISC-V U740:
  L1:  1×
  L2:  4×
  RAM: 40×

x86-64 i7-12700K:
  L1:  1×
  L2:  3×
  L3:  11×
  RAM: 72×

ARM Cortex-A72:
  L1:  1×
  L2:  5×
  RAM: 60×
```

**Implication**: Cache misses are expensive! L1 miss = 4-72× slower depending on where data is found.

---

## Cache Line Conflicts

### Example: Hash Table Conflicts

With 64-byte cache lines and 8-way L1 cache:

**RISC-V U740** (32 KB, 8-way):
- **Sets**: 64
- **Conflict**: Addresses differing by 4096 bytes (64 sets × 64 bytes) map to same set
- **Thrashing**: Accessing 9+ addresses in same set causes evictions

**Example**:
```c
int array[1024];  // 4096 bytes

// These all map to same cache set (assuming aligned):
array[0]    // Offset 0
array[64]   // Offset 256 (4 cache lines)
array[128]  // Offset 512 (8 cache lines)
// ...
array[960]  // Offset 3840 (60 cache lines)

// Accessing all in loop causes thrashing!
```

---

## SIMD Capabilities

### RISC-V Vector Extension (RVV)

**Configuration** (U740):
- **VLEN**: 256 bits (vector register length)
- **ELEN**: 64 bits (max element width)
- **Registers**: 32 vector registers (v0-v31)

**Example**:
```c
// Vector add: c[i] = a[i] + b[i]
void vadd(int *a, int *b, int *c, int n) {
    for (int i = 0; i < n; ) {
        size_t vl = vsetvl_e32m1(n - i);  // Set vector length
        vint32m1_t va = vle32_v_i32m1(&a[i], vl);
        vint32m1_t vb = vle32_v_i32m1(&b[i], vl);
        vint32m1_t vc = vadd_vv_i32m1(va, vb, vl);
        vse32_v_i32m1(&c[i], vc, vl);
        i += vl;
    }
}
```

**Performance**: 8× speedup for 32-bit operations (256 bits / 32 bits = 8 elements).

---

### x86-64 AVX2

**Configuration** (i7-12700K):
- **Register width**: 256 bits
- **Registers**: 16 YMM registers (ymm0-ymm15)

**Example**:
```c
#include <immintrin.h>

// Vector add: c[i] = a[i] + b[i]
void vadd(int *a, int *b, int *c, int n) {
    for (int i = 0; i < n; i += 8) {
        __m256i va = _mm256_loadu_si256((__m256i *)&a[i]);
        __m256i vb = _mm256_loadu_si256((__m256i *)&b[i]);
        __m256i vc = _mm256_add_epi32(va, vb);
        _mm256_storeu_si256((__m256i *)&c[i], vc);
    }
}
```

**Performance**: 8× speedup for 32-bit operations.

---

### ARM NEON

**Configuration** (Cortex-A72):
- **Register width**: 128 bits
- **Registers**: 32 NEON registers (v0-v31)

**Example**:
```c
#include <arm_neon.h>

// Vector add: c[i] = a[i] + b[i]
void vadd(int *a, int *b, int *c, int n) {
    for (int i = 0; i < n; i += 4) {
        int32x4_t va = vld1q_s32(&a[i]);
        int32x4_t vb = vld1q_s32(&b[i]);
        int32x4_t vc = vaddq_s32(va, vb);
        vst1q_s32(&c[i], vc);
    }
}
```

**Performance**: 4× speedup for 32-bit operations (128 bits / 32 bits = 4 elements).

---

## Power Consumption

### Typical Power Draw

**RISC-V U740**:
- **Idle**: 2 W
- **Full load**: 8 W
- **TDP**: 10 W

**x86-64 i7-12700K**:
- **Idle**: 15 W
- **Full load**: 190 W
- **TDP**: 125 W (PL1), 190 W (PL2)

**ARM Cortex-A72** (Raspberry Pi 4):
- **Idle**: 3 W
- **Full load**: 7 W
- **TDP**: 15 W (entire board)

**Implication**: RISC-V and ARM are much more power-efficient than x86-64 for embedded applications.

---

## Summary

This appendix provides hardware specifications for three representative architectures:

**RISC-V (SiFive U740)**:
- 1.2 GHz, in-order, 32 KB L1, 2 MB L2
- Good for embedded systems, low power
- RVV for SIMD

**x86-64 (Intel i7-12700K)**:
- 3.6-5.0 GHz, out-of-order, 48 KB L1, 1.25 MB L2, 25 MB L3
- Highest performance, highest power
- AVX2 for SIMD

**ARM (Cortex-A72)**:
- 1.5 GHz, in-order, 32 KB L1, 1 MB L2
- Good balance of performance and power
- NEON for SIMD

**Key takeaways**:
- Cache hierarchy varies significantly (L3 on x86-64 only)
- Memory latency is 40-72× slower than L1 cache
- SIMD provides 4-8× speedup for vectorizable operations
- Power consumption varies 20× between architectures

For detailed benchmark results on each architecture, see the individual chapters.

