# Appendix A: Benchmark Framework Reference

這個附錄提供本書使用的 benchmark framework 的完整參考。

## Overview

Benchmark framework 為 embedded systems 設計，支援三個 architectures：
- **RISC-V** (RV32I, RV64I)
- **x86-64** (Intel, AMD)
- **ARM** (ARMv7, ARMv8/AArch64)

**Key features**:
- 用 hardware counters 的 cycle-accurate timing
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

用指定 configuration 跑 benchmark。

```c
void benchmark_run(
    const char *name,
    void (*func)(void),
    const benchmark_config_t *config,
    benchmark_result_t *result
);
```

**Parameters**:
- `name`: Benchmark name（用於 reporting）
- `func`: 要 benchmark 的 function
- `config`: Configuration（iterations, warmup, etc.）
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

Manual timing 用於 inline benchmarking。

```c
void benchmark_start(benchmark_context_t *ctx);
void benchmark_stop(benchmark_context_t *ctx);
uint64_t benchmark_get_cycles(benchmark_context_t *ctx);
```

**Example**:
```c
benchmark_context_t ctx;

benchmark_start(&ctx);
// Code to measure
my_critical_section();
benchmark_stop(&ctx);

uint64_t cycles = benchmark_get_cycles(&ctx);
printf("Cycles: %lu\n", cycles);
```

---

## Configuration Options

### `benchmark_config_t`

```c
typedef struct {
    size_t iterations;          // Number of test iterations
    size_t warmup_iterations;   // Warmup runs (not measured)
    bool measure_cache;         // Enable cache miss measurement
    bool measure_branches;      // Enable branch prediction stats
    bool measure_bandwidth;     // Enable memory bandwidth
    uint32_t seed;              // Random seed for reproducibility
} benchmark_config_t;
```

**Default values**:
```c
benchmark_config_t default_config = {
    .iterations = 100,
    .warmup_iterations = 10,
    .measure_cache = false,
    .measure_branches = false,
    .measure_bandwidth = false,
    .seed = 12345,
};
```

---

## Results Structure

### `benchmark_result_t`

```c
typedef struct {
    // Timing
    uint64_t cycles_mean;
    uint64_t cycles_median;
    uint64_t cycles_stddev;
    uint64_t cycles_min;
    uint64_t cycles_max;
    uint64_t cycles_p95;
    uint64_t cycles_p99;
    
    // Cache (if enabled)
    uint64_t l1_misses;
    uint64_t l2_misses;
    uint64_t l3_misses;
    
    // Branches (if enabled)
    uint64_t branches;
    uint64_t branch_misses;
    
    // Bandwidth (if enabled)
    double bandwidth_gbps;
} benchmark_result_t;
```

---

## Architecture-Specific Details

### RISC-V

**Cycle counter**:
```c
static inline uint64_t read_cycles(void) {
    uint64_t cycles;
    asm volatile ("rdcycle %0" : "=r" (cycles));
    return cycles;
}
```

**Instruction counter**:
```c
static inline uint64_t read_instret(void) {
    uint64_t instret;
    asm volatile ("rdinstret %0" : "=r" (instret));
    return instret;
}
```

### x86-64

**RDTSC**:
```c
static inline uint64_t read_cycles(void) {
    uint32_t lo, hi;
    asm volatile ("rdtsc" : "=a" (lo), "=d" (hi));
    return ((uint64_t)hi << 32) | lo;
}
```

### ARM

**PMCCNTR**:
```c
static inline uint64_t read_cycles(void) {
    uint64_t val;
    asm volatile ("mrs %0, pmccntr_el0" : "=r" (val));
    return val;
}
```

---

## Statistical Functions

### `calculate_statistics()`

計算 benchmark results 的 statistics。

```c
void calculate_statistics(
    const uint64_t *samples,
    size_t count,
    benchmark_result_t *result
);
```

**Metrics calculated**:
- Mean (arithmetic average)
- Median (50th percentile)
- Standard deviation
- Min/Max
- 95th percentile (P95)
- 99th percentile (P99)

---

## Cache Measurement

### Linux `perf_event_open`

```c
#include <linux/perf_event.h>
#include <sys/syscall.h>

int setup_cache_counters(void) {
    struct perf_event_attr pe = {
        .type = PERF_TYPE_HW_CACHE,
        .size = sizeof(struct perf_event_attr),
        .config = PERF_COUNT_HW_CACHE_L1D |
                  (PERF_COUNT_HW_CACHE_OP_READ << 8) |
                  (PERF_COUNT_HW_CACHE_RESULT_MISS << 16),
        .disabled = 1,
        .exclude_kernel = 1,
        .exclude_hv = 1,
    };
    
    return syscall(__NR_perf_event_open, &pe, 0, -1, -1, 0);
}
```

---

## Best Practices

### 1. Warmup Iterations

總是用 warmup iterations 來 populate caches：

```c
config.warmup_iterations = config.iterations / 10;  // 10% warmup
```

### 2. Sufficient Iterations

用足夠的 iterations 來獲得穩定結果：

```c
// For fast operations (< 1000 cycles)
config.iterations = 10000;

// For slow operations (> 100,000 cycles)
config.iterations = 100;
```

### 3. Statistical Significance

檢查 standard deviation：

```c
double cv = (double)result.cycles_stddev / result.cycles_mean;
if (cv > 0.05) {
    printf("Warning: High variance (CV = %.2f%%)\n", cv * 100);
}
```

### 4. Outlier Detection

用 percentiles 識別 outliers：

```c
if (result.cycles_p99 > result.cycles_median * 1.5) {
    printf("Warning: Outliers detected\n");
}
```

---

## Example: Complete Benchmark

```c
#include "benchmark.h"

void array_sum(void) {
    int arr[1000];
    int sum = 0;
    for (int i = 0; i < 1000; i++) {
        sum += arr[i];
    }
    // Prevent optimization
    asm volatile ("" : : "r" (sum));
}

int main(void) {
    benchmark_config_t config = {
        .iterations = 1000,
        .warmup_iterations = 100,
        .measure_cache = true,
        .measure_branches = false,
        .seed = 12345,
    };
    
    benchmark_result_t result;
    benchmark_run("array_sum", array_sum, &config, &result);
    
    printf("=== Benchmark Results ===\n");
    printf("Cycles (median): %lu\n", result.cycles_median);
    printf("L1 misses: %lu (%.2f%%)\n",
           result.l1_misses,
           100.0 * result.l1_misses / 1000);
    
    return 0;
}
```

---

## Troubleshooting

### Permission Denied for `perf_event_open`

```bash
# Temporarily allow perf for all users
sudo sysctl -w kernel.perf_event_paranoid=-1

# Or run with sudo
sudo ./benchmark
```

### High Variance in Results

- 增加 iterations
- 關閉 background processes
- Pin process 到特定 CPU core
- Disable frequency scaling

### Cache Counters Not Available

某些 embedded systems 不支援 hardware cache counters。用 simulation 或 estimation：

```c
// Estimate cache misses from timing
uint64_t estimated_misses = (cycles - baseline_cycles) / CACHE_MISS_PENALTY;
```

---

## Further Reading

- Linux perf documentation: https://perf.wiki.kernel.org/
- RISC-V privileged spec: https://riscv.org/specifications/
- Intel optimization manual: https://software.intel.com/
- ARM performance monitoring: https://developer.arm.com/

---

完整 source code 和 examples 在 `code/benchmark/` 目錄。

