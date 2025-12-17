# Chapter 20: Benchmark Case Studies

**Part V: Case Studies**

---

> "There are three kinds of lies: lies, damned lies, and benchmarks."
> — Adapted from Mark Twain

凌晨 2:00，processor startup 的 lead architect Sarah Chen 收到會改變公司軌跡的 email。競爭對手發表了詳細的技術分析，拆解他們旗艦產品的 performance claims。標題很殘酷：「Marketing Hype vs. Reality: How Vendor X Inflated Benchmark Scores by 300%」。

問題不是他們的 processor 慢——它其實相當好。問題是他們選來展示它的 benchmark：Dhrystone。他們的競爭對手逐行展示現代 compilers 如何能 optimize away Dhrystone 的大部分工作，讓分數無意義。更糟的是，他們證明在 *真實* workloads——客戶實際跑的那種——performance 優勢消失了。

Sarah 花了下一週做她幾個月前就該做的事：理解 benchmarks 實際測量什麼。這章是那個調查的結果，檢視兩個業界標準 benchmarks——Dhrystone 和 Coremark——理解不只如何跑它們，而是它們揭示 processor performance 的什麼，更重要的是，它們隱藏什麼。

---

## 20.1 Why Benchmarks Matter (and Why They Fail)

### The Purpose of Benchmarks

在理想世界，我們會跑每個客戶的實際 workload 來測量 processor performance。實際上，我們需要標準化測試：

1. **Represent real work**：反映實際 application behavior
2. **Are reproducible**：跨 runs 給一致結果
3. **Are portable**：在不同 architectures 上跑
4. **Are understandable**：清楚顯示測量什麼

挑戰是這些目標常衝突。讓 benchmark 太簡單，它不代表真實工作。讓它太複雜，它不 reproducible 或 understandable。

### How Benchmarks Fail

Benchmarks 以可預測的方式失敗：

**Compiler optimization**：Compiler 認出 benchmark pattern 並 optimize it away。你在測量 compiler 的聰明，不是 processor 的 performance。

**Narrow workload**：Benchmark 只測試 performance 的一個方面（例如 integer arithmetic），而真實 applications 用混合的 operations。

**Unrealistic data**：Benchmark 用小的、對 cache 友善的 datasets，而真實 applications 處理大的、分散的 data。

**Gaming the benchmark**：Vendors 專門為 benchmark 優化，不是為真實 workloads。

讓我們看這些失敗如何在實踐中顯現。

---

## 20.2 Dhrystone: A Historical Lesson

### Origins and Intent

Dhrystone 在 1984 年由 Reinhold Weicker 創建，作為測量 integer performance 的 synthetic benchmark。名字是對早期 floating-point benchmark「Whetstone」的雙關。

**Design goals**:
- 測量典型 integer operations
- 小到能 fit in cache
- 簡單到能 port
- 避免 floating-point（很多 embedded processors 缺 FPUs）

**Workload composition**（從原始 paper）：
- 53% assignments
- 32% control flow (if/else, loops)
- 15% procedure calls
- String operations
- Record (struct) copying

### What Dhrystone Actually Does

讓我們看 Dhrystone 的核心（簡化）：

```c
typedef struct record {
    struct record *ptr_comp;
    int discr;
    int enum_comp;
    int int_comp;
    char str_comp[31];
} Rec_Type, *Rec_Pointer;

void Proc_1(Rec_Pointer ptr_val_par) {
    Rec_Pointer next_record = ptr_val_par->ptr_comp;
    
    // Structure assignment
    *ptr_val_par->ptr_comp = *ptr_val_par;
    
    ptr_val_par->int_comp = 5;
    next_record->int_comp = ptr_val_par->int_comp;
    next_record->ptr_comp = ptr_val_par->ptr_comp;
    
    // Procedure call
    Proc_3(&next_record->ptr_comp);
    
    // Conditional
    if (next_record->discr == 0) {
        next_record->int_comp = 6;
        Proc_6(ptr_val_par->enum_comp, &next_record->enum_comp);
        next_record->ptr_comp = ptr_val_par->ptr_comp;
        Proc_7(next_record->int_comp, 10, &next_record->int_comp);
    } else {
        *ptr_val_par = *ptr_val_par->ptr_comp;
    }
}

void Proc_2(int *int_par_ref) {
    int int_loc;
    int enum_loc;
    
    int_loc = *int_par_ref + 10;
    
    do {
        if (Ch_1_Glob == 'A') {
            int_loc -= 1;
            *int_par_ref = int_loc - Int_Glob;
            enum_loc = 0;
        }
    } while (enum_loc != 1);
}

// Main loop
int main(void) {
    for (int i = 0; i < NUMBER_OF_RUNS; i++) {
        Proc_1(&Rec_1);
        Proc_2(&Int_1_Loc);
        Proc_3(&Ptr_Glob);
        Proc_4();
        Proc_5();
        // ... more procedures
    }
}
```

看起來合理：procedures、conditionals、struct operations。但有個問題...

### The Fatal Flaws

**Problem 1: Dead Code Elimination**

現代 compilers 能證明 Dhrystone 的大部分工作沒有可觀察的效果：

```c
// Compiler sees:
int x = 5;
x = x + 10;
x = x * 2;
// Result never used

// Compiler generates:
// (nothing - entire computation eliminated)
```

**Problem 2: Constant Propagation**

```c
// Source code:
if (Func_1('A', 'C') == 0) {
    // ...
}

// Compiler knows 'A' and 'C' are constants
// Evaluates Func_1 at compile time
// Replaces entire if statement with constant branch
```

**Problem 3: Unrealistic Data Access**

Dhrystone 的 data 完全 fit in L1 cache（幾 KB）。真實 applications 有 cache misses。Dhrystone 測量 best-case performance，不是 typical performance。

**Problem 4: No Pointer Chasing**

雖然 Dhrystone 用 pointers，access patterns 是可預測的。現代 processors 在需要前 prefetch data。

### The Compiler Optimization Disaster

這是用 `-O3` optimization 發生的事：

```bash
$ gcc -O0 dhrystone.c -o dhry_O0
$ gcc -O3 dhrystone.c -o dhry_O3
$ ./dhry_O0
Dhrystones per second: 500,000

$ ./dhry_O3
Dhrystones per second: 5,000,000
```

**只從 compiler flags 就 10 倍 speedup！** 你不是在測量 processor——你在測量 compiler 認出並消除 Dhrystone patterns 的能力。

不同 compilers 給非常不同的結果：
- GCC 10.2: 4.2 DMIPS/MHz
- Clang 12: 5.1 DMIPS/MHz
- ICC 21: 5.8 DMIPS/MHz

同一個 processor，不同分數。Benchmark 壞了。

### What We Learn from Dhrystone

Dhrystone 教我們什麼 *不該* 做：

1. ❌ 不要用可預測的、constant inputs
2. ❌ 不要允許 dead code elimination
3. ❌ 不要用不切實際的小 datasets
4. ❌ 不要專注於單一 operation type

但它也教我們 benchmarks *應該* 做什麼——這帶我們到 Coremark。

---

## 20.3 Coremark: A Modern Approach

### Design Philosophy

Coremark 在 2009 年由 EEMBC (Embedded Microprocessor Benchmark Consortium) 創建，專門解決 Dhrystone 的缺陷。

**Design goals**:
1. 抵抗 compiler optimization
2. 代表多樣的 real-world operations
3. 跨 architectures portable
4. 有清楚、可執行的 run rules

### The Four Workloads

Coremark 包含四個不同的 workloads，每個測試 processor performance 的不同方面：

#### Workload 1: Linked List Operations

```c
typedef struct list_data_s {
    int16_t data16;
    int16_t idx;
} list_data;

typedef struct list_head_s {
    struct list_head_s *next;
    struct list_data_s *info;
} list_head;

// Find element in list
list_head *core_list_find(list_head *list, list_data *info) {
    if (info->idx >= 0) {
        while (list && (list->info->idx != info->idx))
            list = list->next;
        return list;
    } else {
        while (list && ((list->info->data16 & 0xff) != info->data16))
            list = list->next;
        return list;
    }
}

// Reverse list
list_head *core_list_reverse(list_head *list) {
    list_head *next = NULL, *tmp;
    while (list) {
        tmp = list->next;
        list->next = next;
        next = list;
        list = tmp;
    }
    return next;
}
```

**Why this matters**：Linked lists 測試 pointer chasing 和 unpredictable memory access——真實 applications 的常見 patterns。

#### Workload 2: Matrix Operations

```c
typedef int16_t mat_elem;
typedef mat_elem *matrix_row;

// Matrix multiply
void core_bench_matrix(matrix_params *p) {
    uint32_t N = p->N;
    matrix_row *A = p->A;
    matrix_row *B = p->B;
    matrix_row *C = p->C;

    for (uint32_t i = 0; i < N; i++) {
        for (uint32_t j = 0; j < N; j++) {
            mat_elem sum = 0;
            for (uint32_t k = 0; k < N; k++) {
                sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
        }
    }
}
```

**Why this matters**：Matrix operations 測試 arithmetic performance 和 cache behavior。

#### Workload 3: State Machine

```c
enum CORE_STATE {
    CORE_START,
    CORE_INVALID,
    CORE_S1,
    CORE_S2,
    CORE_INT,
    CORE_FLOAT,
    CORE_EXPONENT,
    CORE_SCIENTIFIC,
    NUM_CORE_STATES
};

// State machine for parsing numbers
enum CORE_STATE core_state_transition(char c, enum CORE_STATE state) {
    switch (state) {
    case CORE_START:
        if (isdigit(c)) return CORE_INT;
        if (c == '+' || c == '-') return CORE_S1;
        return CORE_INVALID;

    case CORE_INT:
        if (isdigit(c)) return CORE_INT;
        if (c == '.') return CORE_FLOAT;
        if (c == 'e' || c == 'E') return CORE_EXPONENT;
        return CORE_INVALID;

    // ... more states
    }
}
```

**Why this matters**：State machines 測試 control flow 和 branch prediction。

#### Workload 4: CRC Calculation

```c
uint16_t crcu16(uint16_t newval, uint16_t crc) {
    uint8_t i;

    for (i = 0; i < 16; i++) {
        if ((crc & 0x8000) != 0) {
            crc = (crc << 1) ^ 0x1021;
        } else {
            crc = crc << 1;
        }

        if ((newval & 0x8000) != 0) {
            crc ^= 0x1021;
        }

        newval = newval << 1;
    }

    return crc;
}

// CRC all results
uint16_t core_bench_crc(void *memblock, uint32_t size) {
    uint16_t crc = 0;
    uint8_t *data = (uint8_t *)memblock;

    for (uint32_t i = 0; i < size; i++) {
        crc = crcu16(data[i], crc);
    }

    return crc;
}
```

**Why this matters**：CRC 測試 bit manipulation 和 data-dependent branches。更重要的是，它驗證所有其他 workloads 的結果——防止 compiler 消除它們。

### How Coremark Resists Optimization

Coremark 的關鍵創新是 **result verification**：

```c
int main(void) {
    uint16_t crc = 0;

    // Run all workloads
    crc = core_bench_list(crc);
    crc = core_bench_matrix(crc);
    crc = core_bench_state(crc);

    // Verify results
    if (crc != EXPECTED_CRC) {
        printf("ERROR: Invalid results!\n");
        return -1;
    }

    printf("Coremark score: %d\n", iterations / time);
    return 0;
}
```

Compiler 不能消除任何計算，因為：
1. 所有結果都 CRC'd
2. CRC 在最後檢查
3. 錯誤的 CRC 導致 failure

這強制 compiler 實際執行工作。

### Coremark Run Rules

EEMBC 強制執行嚴格的 run rules：

1. **Minimum runtime**: 10 秒（確保測量穩定）
2. **Iterations**: 至少 10 次（平均多次 runs）
3. **Validation**: CRC 必須匹配已知值
4. **Compiler flags**: 必須報告（防止不公平比較）
5. **Source modifications**: 不允許（除了 porting）

這些 rules 防止 gaming the benchmark。

### Coremark Performance Analysis

讓我們分析 Coremark 在 RISC-V processor 上的 performance：

```bash
$ ./coremark
CoreMark 1.0 : 12500 / GCC 10.2.0 -O3 / Heap
Iterations: 125000
Total time: 10.0 seconds
Iterations/sec: 12500
CoreMark/MHz: 10.42

Performance breakdown:
  List operations: 35% of time
  Matrix multiply: 28% of time
  State machine: 22% of time
  CRC: 15% of time

Cache behavior:
  L1 I-cache misses: 0.2%
  L1 D-cache misses: 3.8%
  L2 cache misses: 0.5%

Branch prediction:
  Branches: 2.8M
  Mispredictions: 185K (6.6%)
```

這告訴我們：
- List operations 主導（pointer chasing）
- Cache behavior 合理（不是全在 cache）
- Branch mispredictions 現實（不是完美可預測）

---

## 20.4 Performance Analysis

### Comparing Dhrystone vs Coremark

在同一個 RISC-V processor (1.2 GHz)：

```
Dhrystone:
  Score: 5.1 DMIPS/MHz
  Compiler: GCC 10.2 -O3
  Cache misses: 0.1% (everything in L1)
  Branch mispredictions: 1.2%

Coremark:
  Score: 10.4 CoreMark/MHz
  Compiler: GCC 10.2 -O3
  Cache misses: 3.8% (realistic)
  Branch mispredictions: 6.6%

Real application (web server):
  Throughput: 8500 req/sec
  Cache misses: 12.5%
  Branch mispredictions: 8.2%
```

**Observation**：Coremark 的 cache 和 branch behavior 更接近真實 application。

### Compiler Sensitivity

測試不同 optimization levels：

```
Dhrystone:
  -O0: 0.8 DMIPS/MHz
  -O1: 2.1 DMIPS/MHz (2.6× speedup)
  -O2: 4.2 DMIPS/MHz (5.3× speedup)
  -O3: 5.1 DMIPS/MHz (6.4× speedup)

Coremark:
  -O0: 3.2 CoreMark/MHz
  -O1: 6.8 CoreMark/MHz (2.1× speedup)
  -O2: 9.5 CoreMark/MHz (3.0× speedup)
  -O3: 10.4 CoreMark/MHz (3.3× speedup)
```

**Observation**：Dhrystone 對 compiler optimization 更敏感（6.4× vs 3.3×）。

### Architecture Comparison

比較不同 processors：

```
Processor A (RISC-V, 1.2 GHz, in-order):
  Dhrystone: 5.1 DMIPS/MHz
  Coremark: 10.4 CoreMark/MHz
  Real app: 8500 req/sec

Processor B (ARM Cortex-A53, 1.2 GHz, in-order):
  Dhrystone: 4.8 DMIPS/MHz
  Coremark: 9.8 CoreMark/MHz
  Real app: 8200 req/sec

Processor C (ARM Cortex-A72, 1.8 GHz, out-of-order):
  Dhrystone: 6.2 DMIPS/MHz
  Coremark: 15.3 CoreMark/MHz
  Real app: 18500 req/sec
```

**Observation**：Coremark 更好地預測真實 application performance。

---

## 20.5 Benchmark Design Principles

### Lessons from History

比較 Dhrystone 和 Coremark 教我們如何設計好的 benchmarks：

| Principle | Dhrystone | Coremark |
|-----------|-----------|----------|
| **Diverse workloads** | ❌ 大部分 assignments | ✅ 4 個不同 workloads |
| **Resist optimization** | ❌ 容易 optimized | ✅ 多種技術 |
| **Runtime inputs** | ❌ Compile-time constants | ✅ Seed-based generation |
| **Result verification** | ❌ 弱 | ✅ CRC validation |
| **Run rules** | ❌ Informal | ✅ 嚴格、可執行 |
| **Portability** | ✅ 好 | ✅ 優秀 |
| **Understandability** | ✅ 簡單 | ⚠️ 更複雜 |

### Designing Your Own Benchmark

當你需要為特定 use case 創建 benchmark：

**1. Identify the workload**:
```c
// Don't benchmark generic "performance"
// Benchmark specific operations:

// ❌ Too generic
void benchmark_processor(void);

// ✅ Specific workload
void benchmark_packet_processing(void);
void benchmark_image_filtering(void);
void benchmark_crypto_operations(void);
```

**2. Use realistic data**:
```c
// ❌ Unrealistic
int data[100] = {1, 2, 3, 4, ...};  // Fits in cache

// ✅ Realistic
#define DATA_SIZE (1024 * 1024)  // 1 MB
int *data = malloc(DATA_SIZE * sizeof(int));
init_random_data(data, DATA_SIZE, seed);
```

**3. Prevent optimization**:
```c
// ❌ Compiler can eliminate
int sum = 0;
for (int i = 0; i < n; i++) {
    sum += data[i];
}
// sum never used

// ✅ Force computation
volatile int result;
int sum = 0;
for (int i = 0; i < n; i++) {
    sum += data[i];
}
result = sum;  // Volatile prevents elimination
```

**4. Validate results**:
```c
// ✅ Checksum validation
uint32_t expected_crc = compute_expected_crc(seed);
uint32_t actual_crc = run_benchmark(data, size);

if (actual_crc != expected_crc) {
    fprintf(stderr, "ERROR: Benchmark validation failed!\n");
    fprintf(stderr, "Expected: 0x%08x, Got: 0x%08x\n",
            expected_crc, actual_crc);
    return -1;
}
```

**5. Report methodology**:
```c
printf("=== Benchmark Results ===\n");
printf("Workload: Packet processing\n");
printf("Data size: %d packets\n", num_packets);
printf("Iterations: %d\n", iterations);
printf("Compiler: %s %s\n", COMPILER_NAME, COMPILER_VERSION);
printf("Flags: %s\n", COMPILER_FLAGS);
printf("Time: %.2f ms\n", elapsed_ms);
printf("Throughput: %.2f Mpps\n", packets_per_sec / 1e6);
```

### Common Pitfalls

**Pitfall 1: Measuring the wrong thing**:
```c
// ❌ Measures malloc, not computation
start_timer();
int *data = malloc(size);
compute(data, size);
free(data);
stop_timer();

// ✅ Measures only computation
int *data = malloc(size);
start_timer();
compute(data, size);
stop_timer();
free(data);
```

**Pitfall 2: Insufficient warm-up**:
```c
// ❌ First run includes cold cache
for (int i = 0; i < 100; i++) {
    start_timer();
    benchmark();
    stop_timer();
}

// ✅ Warm up first
for (int i = 0; i < 10; i++) {
    benchmark();  // Warm-up, don't measure
}
for (int i = 0; i < 100; i++) {
    start_timer();
    benchmark();
    stop_timer();
}
```

**Pitfall 3: Ignoring variance**:
```c
// ❌ Single measurement
double time = measure_once();
printf("Time: %.2f ms\n", time);

// ✅ Statistical analysis
double times[100];
for (int i = 0; i < 100; i++) {
    times[i] = measure_once();
}
printf("Mean: %.2f ms\n", mean(times, 100));
printf("Median: %.2f ms\n", median(times, 100));
printf("Std dev: %.2f ms\n", stddev(times, 100));
printf("Min: %.2f ms\n", min(times, 100));
printf("Max: %.2f ms\n", max(times, 100));
```

---

## 20.6 Summary

### Key Takeaways

**Dhrystone 已過時**：
- 現代 compilers optimize away 大部分工作
- 分數在 compilers 之間差異很大
- 不代表真實 workloads
- 只用於歷史比較

**Coremark 更好，但不完美**：
- 透過多種技術抵抗 compiler optimization
- 代表多樣的 integer workloads
- 有嚴格、可執行的 run rules
- 但：小 dataset，沒有 FP/SIMD，沒有 OS overhead

**Benchmark design principles**：
1. 用多樣、現實的 workloads
2. 防止 dead code elimination
3. 用 runtime-determined inputs
4. Validate results
5. Report full methodology
6. 理解 limitations

**Benchmarks 是工具，不是目標**：
- 高 Coremark 分數不保證在 *你的* workload 上好 performance
- 理解 benchmark 測量什麼
- 用 application-specific benchmarks 補充
- Profile 真實 applications

### The Bigger Picture

這章詳細檢視了兩個 benchmarks，但教訓廣泛適用：

**From Chapter 3** (Benchmarking)：Statistical rigor matters。跑多次 iterations，report variance，控制 confounding factors。

**From Chapter 2** (Memory Hierarchy)：Cache behavior 主導 performance。有不切實際 data access patterns 的 benchmarks（像 Dhrystone）錯過這個。

**From Chapters 5, 11, 13, 14**：真實 applications 用多樣的 data structures。好的 benchmarks（像 Coremark）測試多種 patterns。

**Looking forward**：當你設計 systems，記住 optimization targets matter。為 benchmark 優化很容易。為真實 workloads 優化——有它們混亂、不可預測的 access patterns 和多樣 operations——是真正的挑戰。

### Practical Advice

**When evaluating processors**：
1. 看超越 headline number
2. 問：「什麼 benchmark？什麼 compiler？什麼 flags？」
3. 如果可能跑你自己的 workload
4. 理解 benchmark 的 limitations

**When designing benchmarks**：
1. 從真實 application traces 開始
2. 識別 critical operations
3. 創建 minimal reproducible test
4. 對照真實 application validate
5. Document everything

**When reporting results**：
1. Full disclosure: hardware, compiler, flags
2. Statistical analysis: mean, median, variance
3. Methodology: warm-up, iterations, validation
4. Limitations: benchmark 不測量什麼

---

Sarah Chen 的公司艱難地學到這些教訓。公開尷尬後，他們切換到 Coremark，更重要的是，基於實際客戶 workloads 開發 application-specific benchmarks。他們的下一個產品發布不專注於 benchmark 分數，而是真實世界的 performance 改善——客戶注意到了。

最好的 benchmark 是匹配你 workload 的那個。其他一切只是 proxy。

