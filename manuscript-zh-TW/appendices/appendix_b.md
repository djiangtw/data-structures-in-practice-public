# Appendix B: Hardware Reference

這個附錄提供本書 benchmarks 使用的系統的詳細硬體規格。

## Overview

所有 benchmarks 在三個代表性 architectures 上跑：
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

**Measured bandwidth** (用 `memcpy`):
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
  
Address breakdown (64-bit):
  [63:12] Tag (52 bits)
  [11:6]  Index (6 bits = 64 sets)
  [5:0]   Offset (6 bits = 64 bytes)
```

### Performance Counters

**Access method** (RDTSC):
```c
static inline uint64_t read_cycles(void) {
    uint32_t lo, hi;
    asm volatile("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}
```

**Available counters** (via `perf`):
- All standard events (cycles, instructions, cache-misses, etc.)
- Extensive PMU (Performance Monitoring Unit) support
- Hardware prefetcher events
- Memory controller events

### Memory Bandwidth

**Measured bandwidth**:
- **Sequential read**: 68.5 GB/s
- **Sequential write**: 65.2 GB/s
- **Random read** (4 KB blocks): 12.3 GB/s
- **Random write** (4 KB blocks): 10.8 GB/s

### TLB Specifications

**DTLB** (Data TLB):
- **L1 DTLB**: 64 entries (4 KB pages), 32 entries (2 MB/4 MB pages)
- **L2 DTLB**: 1536 entries (shared)
- **Page sizes**: 4 KB, 2 MB, 1 GB
- **Miss penalty**: ~100 cycles (page table walk)

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
- **Latency**: 12 cycles

**Main Memory**:
- **Type**: LPDDR4-3200
- **Size**: 8 GB
- **Bandwidth**: 12.8 GB/s (theoretical)
- **Latency**: ~100 ns (~150 cycles)

### Performance Counters

**Access method** (PMCCNTR):
```c
static inline uint64_t read_cycles(void) {
    uint64_t val;
    asm volatile("mrs %0, pmccntr_el0" : "=r"(val));
    return val;
}
```

**Available counters**:
- `cpu-cycles`: CPU cycles
- `instructions`: Instructions retired
- `cache-references`: Cache accesses
- `cache-misses`: Cache misses
- `branch-misses`: Branch mispredictions

### Memory Bandwidth

**Measured bandwidth**:
- **Sequential read**: 10.2 GB/s
- **Sequential write**: 9.8 GB/s
- **Random read** (4 KB blocks): 1.5 GB/s
- **Random write** (4 KB blocks): 1.2 GB/s

---

## Cache Latency Comparison

| System | L1 | L2 | L3 | RAM |
|--------|----|----|----|----|
| **RISC-V U740** | 3 cycles | 12 cycles | - | 120 cycles |
| **Intel i7-12700K** | 4 cycles | 12 cycles | 45 cycles | 288 cycles |
| **ARM Cortex-A72** | 3 cycles | 12 cycles | - | 150 cycles |

---

## Memory Bandwidth Comparison

| System | Sequential Read | Sequential Write | Random Read | Random Write |
|--------|----------------|------------------|-------------|--------------|
| **RISC-V U740** | 15.2 GB/s | 14.8 GB/s | 2.1 GB/s | 1.8 GB/s |
| **Intel i7-12700K** | 68.5 GB/s | 65.2 GB/s | 12.3 GB/s | 10.8 GB/s |
| **ARM Cortex-A72** | 10.2 GB/s | 9.8 GB/s | 1.5 GB/s | 1.2 GB/s |

---

## Cache Line Size

所有三個 systems 都用 **64-byte cache lines**。這是現代 processors 的標準。

**Implications**:
- Struct padding 應該 align 到 64 bytes 來避免 false sharing
- Sequential access 每 64 bytes 觸發一次 cache miss
- Prefetching 通常 fetch 多個 cache lines（128-256 bytes）

---

## Page Sizes

所有三個 systems 都支援多個 page sizes：

| System | Supported Page Sizes |
|--------|---------------------|
| **RISC-V** | 4 KB, 2 MB, 1 GB |
| **x86-64** | 4 KB, 2 MB, 1 GB |
| **ARM** | 4 KB, 2 MB, 1 GB |

**Default**: 4 KB pages（Linux）

**Large pages** (2 MB, 1 GB) 減少 TLB misses，但增加 memory overhead。

---

## Compiler Versions

所有 benchmarks 用以下 compiler versions compile：

| Architecture | Compiler | Version | Flags |
|--------------|----------|---------|-------|
| **RISC-V** | GCC | 11.4.0 | `-O3 -march=rv64gc` |
| **x86-64** | GCC | 11.4.0 | `-O3 -march=native` |
| **ARM** | GCC | 11.4.0 | `-O3 -mcpu=cortex-a72` |

---

## Operating System

所有 systems 跑 **Linux kernel 5.15.0** (Ubuntu 22.04 LTS)。

**Kernel configuration**:
- Transparent Huge Pages (THP): Disabled
- CPU frequency scaling: Performance governor
- ASLR (Address Space Layout Randomization): Disabled for benchmarks

---

## Measurement Methodology

### Cycle Counting

所有 cycle measurements 用 architecture-specific hardware counters：
- **RISC-V**: `rdcycle` instruction
- **x86-64**: `rdtsc` instruction
- **ARM**: `pmccntr_el0` register

### Cache Miss Counting

Cache misses 用 Linux `perf_event_open` syscall 測量。

### Statistical Analysis

每個 benchmark 跑至少 100 iterations，報告：
- Median (50th percentile)
- Mean (arithmetic average)
- Standard deviation
- 95th percentile (P95)
- 99th percentile (P99)

---

## Hardware Variability

**Note**: Performance numbers 可能因以下因素變化：
- CPU frequency scaling (turbo boost)
- Thermal throttling
- Background processes
- NUMA effects (multi-socket systems)
- Memory channel configuration

**Recommendation**: 總是在你自己的硬體上測量，用這些 numbers 作為參考。

---

## Further Information

- **RISC-V**: https://sifive.com/boards/hifive-unmatched
- **Intel**: https://ark.intel.com/
- **ARM**: https://www.raspberrypi.com/products/raspberry-pi-4-model-b/

---

