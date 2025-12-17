# Chapter 20: Benchmark Case Studies

**Part V: Case Studies**

---

> "There are three kinds of lies: lies, damned lies, and benchmarks."
> — Adapted from Mark Twain

It was 2:00 AM when Sarah Chen, lead architect at a processor startup, received the email that would change her company's trajectory. A competitor had published a detailed technical analysis dismantling their flagship product's performance claims. The headline was brutal: "Marketing Hype vs. Reality: How Vendor X Inflated Benchmark Scores by 300%."

The problem wasn't that their processor was slow—it was actually quite good. The problem was the benchmark they'd chosen to showcase it: Dhrystone. Their competitor had shown, line by line, how modern compilers could optimize away most of Dhrystone's work, making the scores meaningless. Worse, they demonstrated that on *real* workloads—the kind customers actually run—the performance advantage evaporated.

Sarah spent the next week doing what she should have done months earlier: understanding what benchmarks actually measure. This chapter is the result of that investigation, examining two industry-standard benchmarks—Dhrystone and Coremark—to understand not just how to run them, but what they reveal about processor performance and, more importantly, what they hide.

---

## 20.1 Why Benchmarks Matter (and Why They Fail)

### The Purpose of Benchmarks

In an ideal world, we'd measure processor performance by running every customer's actual workload. In reality, we need standardized tests that:

1. **Represent real work**: Reflect actual application behavior
2. **Are reproducible**: Give consistent results across runs
3. **Are portable**: Run on different architectures
4. **Are understandable**: Clearly show what's being measured

The challenge is that these goals often conflict. Make a benchmark too simple, and it doesn't represent real work. Make it too complex, and it's not reproducible or understandable.

### How Benchmarks Fail

Benchmarks fail in predictable ways:

**Compiler optimization**: The compiler recognizes the benchmark pattern and optimizes it away. You're measuring the compiler's cleverness, not the processor's performance.

**Narrow workload**: The benchmark tests only one aspect of performance (e.g., integer arithmetic) while real applications use a mix of operations.

**Unrealistic data**: The benchmark uses small, cache-friendly datasets while real applications work with large, scattered data.

**Gaming the benchmark**: Vendors optimize specifically for the benchmark, not for real workloads.

Let's see how these failures manifest in practice.

---

## 20.2 Dhrystone: A Historical Lesson

### Origins and Intent

Dhrystone was created in 1984 by Reinhold Weicker as a synthetic benchmark to measure integer performance. The name is a play on "Whetstone," an earlier floating-point benchmark.

**Design goals**:
- Measure typical integer operations
- Be small enough to fit in cache
- Be simple to port
- Avoid floating-point (many embedded processors lacked FPUs)

**Workload composition** (from the original paper):
- 53% assignments
- 32% control flow (if/else, loops)
- 15% procedure calls
- String operations
- Record (struct) copying

### What Dhrystone Actually Does

Let's look at the core of Dhrystone (simplified):

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
```

**String operations**:
```c
void Proc_2(int *int_par_ref) {
    int int_loc;
    char char_loc;
    
    int_loc = *int_par_ref + 10;
    
    do {
        if (Func_1('A', 'C') == 0) {
            char_loc = 'A';
            int_loc++;
        }
    } while (char_loc != 'A');
    
    *int_par_ref = int_loc;
}
```

### The Fatal Flaws

**Problem 1: Dead Code Elimination**

Modern compilers can prove that much of Dhrystone's work has no observable effect:

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

Dhrystone's data fits entirely in L1 cache (a few KB). Real applications have cache misses. Dhrystone measures best-case performance, not typical performance.

**Problem 4: No Pointer Chasing**

While Dhrystone uses pointers, the access patterns are predictable. Modern processors prefetch the data before it's needed.

### The Compiler Optimization Disaster

Here's what happens with `-O3` optimization:

```bash
$ gcc -O0 dhrystone.c -o dhry_O0
$ gcc -O3 dhrystone.c -o dhry_O3
$ ./dhry_O0
Dhrystones per second: 500,000

$ ./dhry_O3
Dhrystones per second: 5,000,000
```

**10x speedup from compiler flags alone!** You're not measuring the processor—you're measuring the compiler's ability to recognize and eliminate Dhrystone's patterns.

Different compilers give wildly different results:
- GCC 10.2: 4.2 DMIPS/MHz
- Clang 12: 5.1 DMIPS/MHz  
- ICC 21: 5.8 DMIPS/MHz

Same processor, different scores. The benchmark is broken.

### What We Learn from Dhrystone

Dhrystone teaches us what *not* to do:

1. ❌ Don't use predictable, constant inputs
2. ❌ Don't allow dead code elimination
3. ❌ Don't use unrealistically small datasets
4. ❌ Don't focus on a single operation type

But it also teaches us what benchmarks *should* do—which brings us to Coremark.

---

## 20.3 Coremark: A Modern Approach

### Design Philosophy

Coremark was created in 2009 by EEMBC (Embedded Microprocessor Benchmark Consortium) specifically to address Dhrystone's flaws.

**Design goals**:
1. Resist compiler optimization
2. Represent diverse real-world operations
3. Be portable across architectures
4. Have clear, enforceable run rules

### The Four Workloads

Coremark consists of four distinct workloads, each testing different aspects of processor performance:

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

**What it tests**:
- Pointer chasing (cache misses)
- Unpredictable branches
- List traversal patterns (Chapter 5)

**Why it resists optimization**:
- List contents determined at runtime
- Search criteria varies
- Results are used (CRC'd at end)

#### Workload 2: Matrix Operations

```c
typedef int16_t mat_elem;
typedef mat_elem *matrix_row;

// Matrix multiply (simplified)
void core_bench_matrix(mat_params *A, int16_t seed) {
    uint32_t N = A->N;
    matrix_row *C = A->C;
    matrix_row *A_mat = A->A;
    matrix_row *B = A->B;

    // C = A * B
    for (uint32_t i = 0; i < N; i++) {
        for (uint32_t j = 0; j < N; j++) {
            mat_elem temp = 0;
            for (uint32_t k = 0; k < N; k++) {
                temp += A_mat[i][k] * B[k][j];
            }
            C[i][j] = temp;
        }
    }
}
```

**What it tests**:
- Arithmetic intensity
- Cache blocking opportunities
- Memory access patterns (Chapter 4)

**Why it resists optimization**:
- Matrix size determined at runtime
- Results verified with checksum
- Multiple operations prevent constant folding

#### Workload 3: State Machine

```c
enum CORE_STATE {
    CORE_START = 0,
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
enum CORE_STATE core_state_transition(uint8_t **instr, uint32_t *transition_count) {
    uint8_t *str = *instr;
    uint8_t ch;
    enum CORE_STATE state = CORE_START;

    for (; *str && state != CORE_INVALID; str++) {
        ch = *str;
        (*transition_count)++;

        switch (state) {
        case CORE_START:
            if (isdigit(ch)) {
                state = CORE_INT;
            } else if (ch == '+' || ch == '-') {
                state = CORE_S1;
            } else if (ch == '.') {
                state = CORE_FLOAT;
            } else {
                state = CORE_INVALID;
            }
            break;

        case CORE_S1:
            if (isdigit(ch)) {
                state = CORE_INT;
            } else if (ch == '.') {
                state = CORE_FLOAT;
            } else {
                state = CORE_INVALID;
            }
            break;

        case CORE_INT:
            if (ch == '.') {
                state = CORE_FLOAT;
            } else if (!isdigit(ch)) {
                state = CORE_INVALID;
            }
            break;

        // ... more states
        }
    }

    *instr = str;
    return state;
}
```

**What it tests**:
- Branch prediction
- Switch statement performance
- String processing (Chapter 14)

**Why it resists optimization**:
- Input strings vary
- State transitions unpredictable
- Transition count prevents elimination

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

**What it tests**:
- Bit manipulation
- Loop optimization
- Data-dependent operations (Chapter 13)

**Why it resists optimization**:
- CRC depends on all previous data
- Cannot be parallelized easily
- Result must match known value

### Preventing Compiler Optimization

Coremark uses several techniques to prevent dead code elimination:

**1. Runtime-determined inputs**:
```c
// Not this (compiler can optimize):
int data[100] = {1, 2, 3, ...};

// But this (runtime-determined):
void init_data(int *data, int seed) {
    for (int i = 0; i < 100; i++) {
        data[i] = (seed * i) & 0xFF;
        seed = (seed * 1103515245 + 12345) & 0x7FFFFFFF;
    }
}
```

**2. Result verification**:
```c
// All results are CRC'd
uint16_t final_crc = 0;
final_crc = crcu16(list_result, final_crc);
final_crc = crcu16(matrix_result, final_crc);
final_crc = crcu16(state_result, final_crc);

// Must match known value
if (final_crc != EXPECTED_CRC) {
    printf("ERROR: Invalid results!\n");
    return -1;
}
```

**3. Volatile results**:
```c
// Prevent optimization of result storage
volatile uint16_t results[4];
results[0] = list_crc;
results[1] = matrix_crc;
results[2] = state_crc;
results[3] = crc_crc;
```

### Run Rules

Coremark has strict run rules to ensure fair comparison:

1. **Minimum iterations**: Must run for at least 10 seconds
2. **No source modifications**: Core algorithms cannot be changed
3. **Validation**: Results must match known CRC values
4. **Reporting**: Must report iterations/second and iterations/MHz
5. **Compiler flags**: Must be disclosed

**Example valid run**:
```
CoreMark 1.0 : 12500.00 / GCC 10.2.0 -O3 -march=rv64gc / Heap
CoreMark/MHz: 5.00
```

---

## 20.4 Performance Analysis

### Understanding the Scores

**Dhrystone** reports DMIPS (Dhrystone MIPS):
- DMIPS = (Dhrystones/sec) / 1757
- 1757 is the score of a VAX 11/780 (the reference)
- DMIPS/MHz normalizes for clock frequency

**Coremark** reports iterations/second:
- Higher is better
- CoreMark/MHz normalizes for clock frequency
- Typical range: 2.5-5.5 CoreMark/MHz

### What Affects Coremark Scores?

**1. Compiler optimization**:
```bash
# -O0 (no optimization)
CoreMark/MHz: 1.2

# -O2 (standard optimization)
CoreMark/MHz: 4.5

# -O3 (aggressive optimization)
CoreMark/MHz: 5.0

# -O3 -funroll-loops
CoreMark/MHz: 5.2
```

**2. ISA extensions**:
```bash
# RV64GC (base)
CoreMark/MHz: 4.8

# RV64GC + B extension (bit manipulation)
CoreMark/MHz: 5.1

# RV64GC + V extension (vector) - scalar mode
CoreMark/MHz: 5.0
```

**3. Cache configuration**:
```
16 KB I$ + 16 KB D$: 4.2 CoreMark/MHz
32 KB I$ + 32 KB D$: 4.8 CoreMark/MHz
64 KB I$ + 64 KB D$: 5.0 CoreMark/MHz
```

**4. Memory latency**:
```
SRAM (1 cycle):  5.2 CoreMark/MHz
DRAM (100 cycles): 3.8 CoreMark/MHz
```

### Typical Scores (Public Data)

Based on EEMBC's published results and academic papers:

**Embedded processors** (RV32):
- Simple in-order: 2.5-3.0 CoreMark/MHz
- With caches: 3.0-3.5 CoreMark/MHz

**Application processors** (RV64):
- In-order, single-issue: 3.5-4.0 CoreMark/MHz
- In-order, dual-issue: 4.0-4.5 CoreMark/MHz
- Out-of-order: 4.5-5.5 CoreMark/MHz

**For comparison** (x86/ARM):
- ARM Cortex-A53: 3.5 CoreMark/MHz
- ARM Cortex-A72: 4.5 CoreMark/MHz
- Intel Atom: 4.0 CoreMark/MHz
- Intel Core i7: 5.0+ CoreMark/MHz

### What Coremark Doesn't Measure

Coremark is better than Dhrystone, but it's not perfect:

**Missing workloads**:
- ❌ Floating-point operations
- ❌ Vector/SIMD operations
- ❌ System calls
- ❌ I/O operations
- ❌ Multi-threading

**Unrealistic aspects**:
- Small dataset (fits in cache)
- No OS overhead
- No interrupts
- Deterministic execution

**What it measures well**:
- ✅ Integer arithmetic
- ✅ Pointer chasing
- ✅ Branch prediction
- ✅ Compiler effectiveness
- ✅ Cache performance (for small datasets)

---

## 20.5 Benchmark Design Principles

### Lessons from History

Comparing Dhrystone and Coremark teaches us how to design good benchmarks:

| Principle | Dhrystone | Coremark |
|-----------|-----------|----------|
| **Diverse workloads** | ❌ Mostly assignments | ✅ 4 distinct workloads |
| **Resist optimization** | ❌ Easily optimized | ✅ Multiple techniques |
| **Runtime inputs** | ❌ Compile-time constants | ✅ Seed-based generation |
| **Result verification** | ❌ Weak | ✅ CRC validation |
| **Run rules** | ❌ Informal | ✅ Strict, enforceable |
| **Portability** | ✅ Good | ✅ Excellent |
| **Understandability** | ✅ Simple | ⚠️ More complex |

### Designing Your Own Benchmark

When you need to create a benchmark for your specific use case:

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

**Dhrystone is obsolete**:
- Modern compilers optimize away most of the work
- Scores vary wildly between compilers
- Doesn't represent real workloads
- Use only for historical comparison

**Coremark is better, but not perfect**:
- Resists compiler optimization through multiple techniques
- Represents diverse integer workloads
- Has strict, enforceable run rules
- But: small dataset, no FP/SIMD, no OS overhead

**Benchmark design principles**:
1. Use diverse, realistic workloads
2. Prevent dead code elimination
3. Use runtime-determined inputs
4. Validate results
5. Report full methodology
6. Understand limitations

**Benchmarks are tools, not goals**:
- A high Coremark score doesn't guarantee good performance on *your* workload
- Understand what the benchmark measures
- Supplement with application-specific benchmarks
- Profile real applications

### The Bigger Picture

This chapter examined two benchmarks in detail, but the lessons apply broadly:

**From Chapter 3** (Benchmarking): Statistical rigor matters. Run multiple iterations, report variance, control for confounding factors.

**From Chapter 2** (Memory Hierarchy): Cache behavior dominates performance. Benchmarks with unrealistic data access patterns (like Dhrystone) miss this.

**From Chapters 5, 11, 13, 14**: Real applications use diverse data structures. Good benchmarks (like Coremark) test multiple patterns.

**Looking forward**: As you design systems, remember that optimization targets matter. Optimizing for a benchmark is easy. Optimizing for real workloads—with their messy, unpredictable access patterns and diverse operations—is the real challenge.

### Practical Advice

**When evaluating processors**:
1. Look beyond the headline number
2. Ask: "What benchmark? What compiler? What flags?"
3. Run your own workload if possible
4. Understand the benchmark's limitations

**When designing benchmarks**:
1. Start with real application traces
2. Identify the critical operations
3. Create a minimal reproducible test
4. Validate against the real application
5. Document everything

**When reporting results**:
1. Full disclosure: hardware, compiler, flags
2. Statistical analysis: mean, median, variance
3. Methodology: warm-up, iterations, validation
4. Limitations: what the benchmark doesn't measure

---

Sarah Chen's company learned these lessons the hard way. After the public embarrassment, they switched to Coremark and, more importantly, developed application-specific benchmarks based on actual customer workloads. Their next product launch focused not on benchmark scores, but on real-world performance improvements—and customers noticed.

The best benchmark is the one that matches your workload. Everything else is just a proxy.


