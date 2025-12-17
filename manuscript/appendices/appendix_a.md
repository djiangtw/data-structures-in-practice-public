# Appendix A: Benchmark Framework Reference

This appendix provides a complete reference for the benchmark framework used throughout this book.

## Overview

The benchmark framework is designed for embedded systems and supports three architectures:
- **RISC-V** (RV32I, RV64I)
- **x86-64** (Intel, AMD)
- **ARM** (ARMv7, ARMv8/AArch64)

**Key features**:
- Cycle-accurate timing using hardware counters
- Cache performance measurement (L1, L2, L3 misses)
- Branch prediction statistics
- Memory bandwidth measurement
- Statistical analysis (mean, median, stddev, percentiles)

---

## Installation

### Prerequisites

```bash
# RISC-V toolchain
sudo apt-get install gcc-riscv64-unknown-elf

# x86-64 toolchain (usually pre-installed)
sudo apt-get install build-essential

# ARM toolchain
sudo apt-get install gcc-arm-none-eabi

# Performance tools
sudo apt-get install linux-tools-common linux-tools-generic
```

### Building the Framework

```bash
git clone https://github.com/djiangtw/data-structures-in-practice.git
cd data-structures-in-practice/code

# Build for RISC-V
make ARCH=riscv64

# Build for x86-64
make ARCH=x86_64

# Build for ARM
make ARCH=arm
```

---

## Basic Usage

### Simple Benchmark

```c
#include "benchmark.h"

void test_function(void) {
    // Code to benchmark
    for (int i = 0; i < 1000; i++) {
        // ...
    }
}

int main(void) {
    benchmark_config_t config = {
        .iterations = 100,
        .warmup_iterations = 10,
        .measure_cache = true,
    };
    
    benchmark_result_t result;
    benchmark_run("test_function", test_function, &config, &result);
    
    benchmark_print_result(&result);
    return 0;
}
```

**Output**:
```
Benchmark: test_function
Iterations: 100 (10 warmup)

Cycles:
  Mean:   125,430
  Median: 124,890
  Stddev: 2,340
  Min:    122,100
  Max:    131,200
  
Cache:
  L1 misses: 1,245 (0.8%)
  L2 misses: 89 (0.06%)
  L3 misses: 12 (0.008%)
```

---

## API Reference

### Core Functions

#### `benchmark_run()`

Run a benchmark with specified configuration.

```c
void benchmark_run(
    const char *name,
    void (*func)(void),
    const benchmark_config_t *config,
    benchmark_result_t *result
);
```

**Parameters**:
- `name`: Benchmark name (for reporting)
- `func`: Function to benchmark
- `config`: Configuration (iterations, warmup, etc.)
- `result`: Output results

**Example**:
```c
benchmark_config_t config = {
    .iterations = 1000,
    .warmup_iterations = 100,
    .measure_cache = true,
    .measure_branches = true,
};

benchmark_result_t result;
benchmark_run("my_test", my_function, &config, &result);
```

---

#### `benchmark_start()` / `benchmark_stop()`

Manual timing for inline benchmarking.

```c
void benchmark_start(benchmark_context_t *ctx);
void benchmark_stop(benchmark_context_t *ctx);
```

**Example**:
```c
benchmark_context_t ctx;

benchmark_start(&ctx);
// Code to measure
my_function();
benchmark_stop(&ctx);

printf("Cycles: %llu\n", ctx.cycles);
printf("L1 misses: %llu\n", ctx.l1_misses);
```

---

#### `benchmark_compare()`

Compare two implementations.

```c
void benchmark_compare(
    const char *name1, void (*func1)(void),
    const char *name2, void (*func2)(void),
    const benchmark_config_t *config
);
```

**Example**:
```c
benchmark_compare(
    "linked_list", test_linked_list,
    "array", test_array,
    &config
);
```

**Output**:
```
Comparison: linked_list vs array

linked_list:
  Cycles: 450,000
  L1 misses: 18,500
  
array:
  Cycles: 85,000
  L1 misses: 2,800
  
Speedup: 5.3× (array is faster)
Cache miss reduction: 6.6×
```

---

### Configuration

#### `benchmark_config_t`

```c
typedef struct {
    int iterations;           // Number of iterations
    int warmup_iterations;    // Warmup iterations (not measured)
    bool measure_cache;       // Measure cache misses
    bool measure_branches;    // Measure branch mispredictions
    bool measure_memory_bw;   // Measure memory bandwidth
    bool verbose;             // Print detailed output
} benchmark_config_t;
```

**Default values**:
```c
benchmark_config_t default_config = {
    .iterations = 100,
    .warmup_iterations = 10,
    .measure_cache = true,
    .measure_branches = false,
    .measure_memory_bw = false,
    .verbose = false,
};
```

---

#### `benchmark_result_t`

```c
typedef struct {
    // Timing
    uint64_t cycles_mean;
    uint64_t cycles_median;
    uint64_t cycles_stddev;
    uint64_t cycles_min;
    uint64_t cycles_max;
    
    // Cache
    uint64_t l1_misses;
    uint64_t l2_misses;
    uint64_t l3_misses;
    
    // Branches
    uint64_t branches;
    uint64_t branch_misses;
    
    // Memory
    uint64_t bytes_read;
    uint64_t bytes_written;
} benchmark_result_t;
```

---

## Architecture-Specific Details

### RISC-V

**Performance counters**:
```c
// Cycle counter
uint64_t read_cycles(void) {
    uint64_t cycles;
    asm volatile("rdcycle %0" : "=r"(cycles));
    return cycles;
}

// Instruction counter
uint64_t read_instret(void) {
    uint64_t instret;
    asm volatile("rdinstret %0" : "=r"(instret));
    return instret;
}
```

**Cache measurement**: Requires hardware performance counters (HPM) support.

---

### x86-64

**Performance counters**:
```c
// RDTSC (Time Stamp Counter)
static inline uint64_t read_tsc(void) {
    uint32_t lo, hi;
    asm volatile("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

// RDTSCP (serializing version)
static inline uint64_t read_tscp(void) {
    uint32_t lo, hi;
    asm volatile("rdtscp" : "=a"(lo), "=d"(hi) :: "rcx");
    return ((uint64_t)hi << 32) | lo;
}
```

**Cache measurement**: Uses `perf_event_open()` for hardware counters.

---

### ARM

**Performance counters**:
```c
// Cycle counter (PMCCNTR)
static inline uint64_t read_cycles(void) {
    uint64_t val;
    asm volatile("mrs %0, pmccntr_el0" : "=r"(val));
    return val;
}

// Enable cycle counter
static inline void enable_cycle_counter(void) {
    uint64_t val = 1;
    asm volatile("msr pmcr_el0, %0" :: "r"(val));
    asm volatile("msr pmcntenset_el0, %0" :: "r"(val));
}
```

---

## Advanced Features

### Statistical Analysis

The framework automatically computes statistics:

```c
typedef struct {
    double mean;
    double median;
    double stddev;
    double p50;   // 50th percentile
    double p95;   // 95th percentile
    double p99;   // 99th percentile
} statistics_t;

void compute_statistics(uint64_t *samples, int count, statistics_t *stats);
```

**Example**:
```c
uint64_t samples[100];
// ... collect samples

statistics_t stats;
compute_statistics(samples, 100, &stats);

printf("Mean: %.2f\n", stats.mean);
printf("P95: %.2f\n", stats.p95);
printf("P99: %.2f\n", stats.p99);
```

---

### Memory Bandwidth Measurement

Measure memory read/write bandwidth:

```c
void benchmark_memory_bandwidth(void) {
    benchmark_config_t config = {
        .iterations = 100,
        .measure_memory_bw = true,
    };

    benchmark_result_t result;
    benchmark_run("memory_copy", test_memcpy, &config, &result);

    double bandwidth_gb_s = (double)result.bytes_read / 1e9;
    printf("Bandwidth: %.2f GB/s\n", bandwidth_gb_s);
}
```

---

### Cache Line Analysis

Analyze cache line utilization:

```c
typedef struct {
    int cache_line_size;      // 64 bytes typical
    int l1_cache_size;        // 32 KB typical
    int l2_cache_size;        // 256 KB typical
    int l3_cache_size;        // 8 MB typical
} cache_info_t;

void get_cache_info(cache_info_t *info);

void analyze_cache_usage(void *data, size_t size, cache_info_t *cache) {
    int cache_lines = (size + cache->cache_line_size - 1) / cache->cache_line_size;
    printf("Data size: %zu bytes\n", size);
    printf("Cache lines: %d\n", cache_lines);
    printf("L1 coverage: %.1f%%\n",
           100.0 * size / cache->l1_cache_size);
}
```

---

## Example Benchmarks

### Array vs Linked List

```c
#include "benchmark.h"

#define SIZE 10000

// Array implementation
int array[SIZE];

void test_array_sequential(void) {
    int sum = 0;
    for (int i = 0; i < SIZE; i++) {
        sum += array[i];
    }
}

// Linked list implementation
typedef struct node {
    int value;
    struct node *next;
} node_t;

node_t *list_head;

void test_list_sequential(void) {
    int sum = 0;
    for (node_t *n = list_head; n; n = n->next) {
        sum += n->value;
    }
}

int main(void) {
    // Initialize data structures
    for (int i = 0; i < SIZE; i++) {
        array[i] = i;
    }

    list_head = NULL;
    for (int i = SIZE - 1; i >= 0; i--) {
        node_t *n = malloc(sizeof(node_t));
        n->value = i;
        n->next = list_head;
        list_head = n;
    }

    // Benchmark
    benchmark_config_t config = {
        .iterations = 1000,
        .warmup_iterations = 100,
        .measure_cache = true,
    };

    benchmark_compare(
        "array", test_array_sequential,
        "linked_list", test_list_sequential,
        &config
    );

    return 0;
}
```

**Expected output**:
```
Comparison: array vs linked_list

array:
  Cycles: 12,500
  L1 misses: 156 (1.2%)
  L2 misses: 8 (0.06%)

linked_list:
  Cycles: 185,000
  L1 misses: 9,850 (98.5%)
  L2 misses: 1,240 (12.4%)

Speedup: 14.8× (array is faster)
Cache miss increase: 63.1×
```

---

### Hash Table Benchmark

```c
#include "benchmark.h"

#define TABLE_SIZE 1024
#define NUM_KEYS 10000

typedef struct entry {
    int key;
    int value;
    struct entry *next;
} entry_t;

entry_t *hash_table[TABLE_SIZE];

int hash(int key) {
    return key % TABLE_SIZE;
}

void test_hash_insert(void) {
    for (int i = 0; i < NUM_KEYS; i++) {
        int h = hash(i);
        entry_t *e = malloc(sizeof(entry_t));
        e->key = i;
        e->value = i * 2;
        e->next = hash_table[h];
        hash_table[h] = e;
    }
}

void test_hash_lookup(void) {
    for (int i = 0; i < NUM_KEYS; i++) {
        int h = hash(i);
        for (entry_t *e = hash_table[h]; e; e = e->next) {
            if (e->key == i) {
                break;
            }
        }
    }
}

int main(void) {
    benchmark_config_t config = {
        .iterations = 100,
        .measure_cache = true,
    };

    benchmark_result_t result;

    benchmark_run("hash_insert", test_hash_insert, &config, &result);
    printf("Insert: %llu cycles, %llu L1 misses\n",
           result.cycles_mean, result.l1_misses);

    benchmark_run("hash_lookup", test_hash_lookup, &config, &result);
    printf("Lookup: %llu cycles, %llu L1 misses\n",
           result.cycles_mean, result.l1_misses);

    return 0;
}
```

---

## Troubleshooting

### Permission Denied for Performance Counters

**Problem**: `perf_event_open()` fails with `EACCES`.

**Solution**:
```bash
# Temporarily allow access (until reboot)
sudo sysctl -w kernel.perf_event_paranoid=-1

# Permanently allow access
echo "kernel.perf_event_paranoid = -1" | sudo tee -a /etc/sysctl.conf
```

---

### Inconsistent Results

**Problem**: Benchmark results vary widely between runs.

**Solutions**:

1. **Increase warmup iterations**:
```c
config.warmup_iterations = 100;  // More warmup
```

2. **Disable CPU frequency scaling**:
```bash
sudo cpupower frequency-set --governor performance
```

3. **Pin to specific CPU**:
```c
#include <sched.h>

cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(0, &set);  // Pin to CPU 0
sched_setaffinity(0, sizeof(set), &set);
```

4. **Disable interrupts** (embedded systems only):
```c
// RISC-V
asm volatile("csrci mstatus, 0x8");  // Disable interrupts

benchmark_run(...);

asm volatile("csrsi mstatus, 0x8");  // Re-enable interrupts
```

---

### Cache Measurement Not Working

**Problem**: Cache miss counters always return 0.

**Solutions**:

1. **Check hardware support**:
```bash
# x86-64
cat /proc/cpuinfo | grep -i pmu

# ARM
cat /proc/cpuinfo | grep -i pmu
```

2. **Enable performance counters** (ARM):
```c
// Enable user-mode access to PMU
asm volatile("msr pmuserenr_el0, %0" :: "r"(1));
```

3. **Use perf instead**:
```bash
perf stat -e cache-misses,cache-references ./benchmark
```

---

## Best Practices

### 1. Always Use Warmup Iterations

```c
// BAD: No warmup
config.warmup_iterations = 0;

// GOOD: Warmup to stabilize caches
config.warmup_iterations = 100;
```

**Why**: First iterations include cold cache effects, instruction cache misses, branch predictor training.

---

### 2. Run Multiple Iterations

```c
// BAD: Single iteration
config.iterations = 1;

// GOOD: Multiple iterations for statistics
config.iterations = 1000;
```

**Why**: Single measurements are noisy. Statistics (mean, median, stddev) require multiple samples.

---

### 3. Measure What Matters

```c
// BAD: Measure everything
config.measure_cache = true;
config.measure_branches = true;
config.measure_memory_bw = true;

// GOOD: Measure only what you need
config.measure_cache = true;  // Focus on cache behavior
```

**Why**: Measuring too many counters can interfere with each other (multiplexing overhead).

---

### 4. Compare Apples to Apples

```c
// BAD: Different data sizes
test_array_1000();
test_list_10000();

// GOOD: Same data size
test_array_10000();
test_list_10000();
```

**Why**: Fair comparison requires identical workloads.

---

### 5. Report Context

Always report:
- CPU model and frequency
- Cache sizes (L1, L2, L3)
- Compiler and optimization flags
- Data size

**Example**:
```
Benchmark: array vs linked list
CPU: RISC-V RV64GC @ 1.2 GHz
L1: 32 KB, L2: 256 KB, L3: 8 MB
Compiler: GCC 12.2.0 -O2
Data size: 10,000 elements
```

---

## Summary

The benchmark framework provides:
- **Cycle-accurate timing** using hardware counters
- **Cache performance measurement** (L1, L2, L3 misses)
- **Statistical analysis** (mean, median, percentiles)
- **Cross-architecture support** (RISC-V, x86-64, ARM)
- **Easy comparison** of implementations

**Key functions**:
- `benchmark_run()`: Run a benchmark
- `benchmark_compare()`: Compare two implementations
- `benchmark_start()` / `benchmark_stop()`: Manual timing

**Best practices**:
- Use warmup iterations
- Run multiple iterations
- Measure what matters
- Compare fairly
- Report context

For more examples, see the `code/benchmarks/` directory in the repository.

