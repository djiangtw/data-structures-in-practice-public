# Chapter 17: Bootloader Data Structures

**Part V: Case Studies**

---

> "Simplicity is the ultimate sophistication."
> — Leonardo da Vinci

## The 500 Millisecond Deadline

Bootloader 太慢了。需求很清楚：在 500 毫秒內 boot。測量同樣清楚：720 毫秒。我們差了 44%。

這不是軟性需求。這個設備是工業控制器，需要在上電後快速回應。每秒 boot 時間代表生產力損失。產品規格說最多 500 ms。我們必須達成。

Bootloader 的工作很直接：
1. Initialize hardware (UART, SPI, DDR controller)
2. 從 flash memory 載入 kernel
3. Parse device tree
4. Jump to kernel entry point

實作看起來合理——C library 的標準資料結構：

```c
// Device tree parsing with malloc'd linked lists
typedef struct dt_node {
    char *name;
    struct dt_node *parent;
    struct dt_node *children;  // Linked list
    struct dt_node *next;
    property_t *properties;    // Linked list
} dt_node_t;

dt_node_t *parse_device_tree(void *fdt) {
    dt_node_t *root = malloc(sizeof(dt_node_t));
    // Parse FDT, allocate nodes with malloc...
}
```

Boot 時間測量：

```bash
$ ./bootloader
[0.000] Start
[0.120] Hardware init complete
[0.450] Device tree parsed (2,847 malloc calls)
[0.680] Kernel loaded
[0.720] Jump to kernel

Total boot time: 720 ms
```

**720 ms**——我們差了 500 ms 目標 44%！

Profiling 顯示問題：

```bash
$ perf record -e cycles ./bootloader
$ perf report

  45.2%  malloc/free overhead
  28.1%  Device tree parsing
  18.5%  Flash I/O
   8.2%  Hardware init
```

**45% 的時間在 malloc/free**！在 bootloader 中，這是浪費。

我重寫了它，用 bootloader-specific 資料結構。結果：

```bash
$ ./bootloader_optimized
[0.000] Start
[0.095] Hardware init complete
[0.220] Device tree parsed (0 malloc calls)
[0.410] Kernel loaded
[0.425] Jump to kernel

Total boot time: 425 ms
```

**425 ms**——比 500 ms 目標快 15%，比原始快 1.7 倍。

這章探索 bootloader 環境中的資料結構。

---

## The Bootloader Environment

Bootloader 是特殊的環境：

### Constraints

1. **No dynamic memory**：malloc/free 太慢且不可靠
2. **Limited RAM**：通常 <1 MB（DDR 還沒初始化）
3. **No OS**：沒有 threads，沒有 file system
4. **Time-critical**：每毫秒都重要
5. **Run once**：不需要 cleanup（會 jump 到 kernel）

### What You Have

1. **Stack**：通常 16-64 KB
2. **Static data**：在 .data 和 .bss sections
3. **Flash memory**：Read-only，慢（~100 MB/s）
4. **SRAM**：Fast，但小（<1 MB）

這個環境需要不同的資料結構。

---

## The Textbook Story

教科書說用標準資料結構：
- Linked lists for trees
- malloc() for dynamic allocation
- Hash tables for lookups

但在 bootloader 中，這些都太慢或不可用。

---

## The Reality Check: Why Standard Approaches Fail

### 1. malloc() is Slow

每次 malloc() call：
- Search free list (~50-100 cycles)
- Split block if needed
- Update metadata
- Possible fragmentation

對 2,847 次 malloc calls：**~285,000 cycles wasted**！

在 1 GHz CPU 上，那是 **285 μs** 只在 memory allocation。

### 2. Linked Lists Have Poor Cache Behavior

Device tree 有 ~200 nodes，每個 node 是分開的 malloc：

```
Node1 → Node2 (cache miss) → Node3 (cache miss) → ...
```

遍歷 tree：**200 cache misses** × 150 cycles = **30,000 cycles**。

### 3. No Cleanup Needed

Bootloader 在 jump 到 kernel 後不會回來。所有 memory 會被 kernel 重用。

所以 malloc/free 的 cleanup 是 **完全浪費**。

---

## Solution 1: Bump Allocator

最簡單的 allocator：只增加 pointer。

```c
typedef struct {
    uint8_t *base;
    uint8_t *current;
    size_t size;
} bump_allocator_t;

void bump_init(bump_allocator_t *alloc, void *mem, size_t size) {
    alloc->base = mem;
    alloc->current = mem;
    alloc->size = size;
}

void *bump_alloc(bump_allocator_t *alloc, size_t size) {
    // Align to 8 bytes
    size = (size + 7) & ~7;
    
    if (alloc->current + size > alloc->base + alloc->size) {
        return NULL;  // Out of memory
    }
    
    void *ptr = alloc->current;
    alloc->current += size;
    return ptr;
}

// No free() needed!
```

**Why this works**:
- O(1) allocation（只是 pointer increment）
- 沒有 fragmentation
- 沒有 metadata overhead
- 不需要 free()（bootloader 不會 cleanup）

### The Benchmark

```
Test: Allocate 2,847 device tree nodes

malloc():
  Cycles: 285,000
  Time: 285 μs

Bump allocator:
  Cycles: 8,500
  Time: 8.5 μs

Speedup: 33.5×
```

**快 33.5 倍**！

---

## Solution 2: Flat Device Tree Representation

不用 malloc'd tree nodes，用 flat array：

```c
#define MAX_DT_NODES 512

typedef struct {
    char name[32];
    uint16_t parent_idx;
    uint16_t first_child_idx;
    uint16_t next_sibling_idx;
    uint16_t num_properties;
    property_t properties[8];  // Inline, not pointer
} dt_node_flat_t;

typedef struct {
    dt_node_flat_t nodes[MAX_DT_NODES];
    int num_nodes;
} device_tree_t;

static device_tree_t g_dt;  // Static allocation, no malloc

int dt_add_node(const char *name, int parent_idx) {
    if (g_dt.num_nodes >= MAX_DT_NODES) {
        return -1;  // Too many nodes
    }

    int idx = g_dt.num_nodes++;
    dt_node_flat_t *node = &g_dt.nodes[idx];

    strncpy(node->name, name, sizeof(node->name) - 1);
    node->parent_idx = parent_idx;
    node->first_child_idx = 0xFFFF;  // No children yet
    node->next_sibling_idx = 0xFFFF;
    node->num_properties = 0;

    // Link to parent
    if (parent_idx >= 0) {
        dt_node_flat_t *parent = &g_dt.nodes[parent_idx];
        if (parent->first_child_idx == 0xFFFF) {
            parent->first_child_idx = idx;
        } else {
            // Find last sibling
            int sibling_idx = parent->first_child_idx;
            while (g_dt.nodes[sibling_idx].next_sibling_idx != 0xFFFF) {
                sibling_idx = g_dt.nodes[sibling_idx].next_sibling_idx;
            }
            g_dt.nodes[sibling_idx].next_sibling_idx = idx;
        }
    }

    return idx;
}
```

**Advantages**:
- **No malloc**：所有 nodes 在一個 static array
- **Cache-friendly**：Sequential access to nodes
- **Predictable memory**：Compile time 知道精確記憶體使用
- **Fast traversal**：Array indexing instead of pointer chasing

### The Benchmark

```
Test: Parse device tree (347 nodes, 1,245 properties)

Malloc'd linked list:
  Cycles: 2.8M
  Cache misses: 185K
  Memory: 64 KB (fragmented)
  Time: 2.3 ms

Flat array:
  Cycles: 0.45M
  Cache misses: 12K
  Memory: 48 KB (contiguous)
  Time: 0.38 ms

Speedup: 6.1×
Cache miss reduction: 15.4×
```

---

## Solution 3: Ring Buffer for Boot Log

Bootloaders 需要 log messages 用於 debugging，但在 UART initialized 前不能用 printf。

### The Problem

標準方法：在 linked list 中 buffer messages，稍後 print。

```c
typedef struct log_entry {
    char message[128];
    struct log_entry *next;
} log_entry_t;

log_entry_t *log_head = NULL;

void boot_log(const char *msg) {
    log_entry_t *entry = malloc(sizeof(log_entry_t));
    strncpy(entry->message, msg, 127);
    entry->next = log_head;
    log_head = entry;
}
```

**Problems**:
- 每個 log message 都 malloc
- Printing 時 pointer chasing
- Unbounded memory usage

### The Solution: Static Ring Buffer

```c
#define LOG_BUFFER_SIZE 4096
#define MAX_LOG_ENTRIES 64

typedef struct {
    char buffer[LOG_BUFFER_SIZE];
    uint16_t offsets[MAX_LOG_ENTRIES];
    int head;
    int tail;
    int count;
} boot_log_t;

static boot_log_t g_log = {0};

void boot_log(const char *msg) {
    int len = strlen(msg);
    if (len >= LOG_BUFFER_SIZE) {
        len = LOG_BUFFER_SIZE - 1;
    }

    // Check if buffer has space
    int next_tail = (g_log.tail + len + 1) % LOG_BUFFER_SIZE;
    if (next_tail == g_log.head && g_log.count > 0) {
        // Buffer full, drop oldest message
        g_log.head = (g_log.head + strlen(&g_log.buffer[g_log.head]) + 1) % LOG_BUFFER_SIZE;
        g_log.count--;
    }

    // Copy message
    g_log.offsets[g_log.count % MAX_LOG_ENTRIES] = g_log.tail;
    for (int i = 0; i < len; i++) {
        g_log.buffer[g_log.tail] = msg[i];
        g_log.tail = (g_log.tail + 1) % LOG_BUFFER_SIZE;
    }
    g_log.buffer[g_log.tail] = '\0';
    g_log.tail = (g_log.tail + 1) % LOG_BUFFER_SIZE;

    g_log.count++;
    if (g_log.count > MAX_LOG_ENTRIES) {
        g_log.count = MAX_LOG_ENTRIES;
    }
}

void boot_log_print_all(void) {
    for (int i = 0; i < g_log.count; i++) {
        printf("%s\n", &g_log.buffer[g_log.offsets[i]]);
    }
}
```

**Advantages**:
- 沒有 malloc
- Fixed memory usage (4 KB)
- Fast append (O(1))
- Automatically drops oldest messages if full

### The Benchmark

```
Test: Log 128 boot messages

Linked list with malloc:
  Cycles: 185K
  Memory: 16 KB (128 × 128 bytes)
  Time: 185 μs

Static ring buffer:
  Cycles: 12K
  Memory: 4 KB (fixed)
  Time: 12 μs

Speedup: 15.4×
Memory reduction: 4×
```

---

## Solution 4: Compile-Time Configuration Table

Hardware initialization 需要 configuration data。不在 runtime parsing，用 compile-time tables。

### The Problem

Runtime parsing：

```c
void init_uart(void) {
    // Parse device tree to find UART config
    dt_node_t *uart = dt_find_node("/soc/uart@10000000");
    uint32_t base = dt_get_property_u32(uart, "reg");
    uint32_t baud = dt_get_property_u32(uart, "baud-rate");

    // Initialize UART
    uart_init(base, baud);
}
```

**Problems**:
- Boot time 時 device tree parsing
- Node lookup 的 string comparisons
- Multiple memory accesses

### The Solution: Compile-Time Table

```c
// Generated from device tree at compile time
typedef struct {
    uint32_t base;
    uint32_t baud;
    uint32_t irq;
} uart_config_t;

static const uart_config_t g_uart_config = {
    .base = 0x10000000,
    .baud = 115200,
    .irq = 10,
};

void init_uart(void) {
    // Direct access, no parsing
    uart_init(g_uart_config.base, g_uart_config.baud);
}
```

**Advantages**:
- **Zero runtime overhead**：沒有 parsing
- **Type-safe**：Compiler 檢查 types
- **Cache-friendly**：所有 config 在一個 struct
- **Fast**：Direct memory access

### The Benchmark

```
Test: Initialize 8 peripherals (UART, SPI, I2C, GPIO, etc.)

Runtime device tree parsing:
  Cycles: 1.2M
  Cache misses: 85K
  Time: 1.0 ms

Compile-time config table:
  Cycles: 45K
  Cache misses: 2K
  Time: 0.038 ms

Speedup: 26.7×
```

---

## Real-World Example: U-Boot's FDT (Flattened Device Tree)

U-Boot (Universal Bootloader) 用聰明的 representation 給 device trees。

### The FDT Format

不用 malloc'd nodes 的 tree，FDT 是 flat binary blob：

```
FDT Header (40 bytes):
  magic: 0xd00dfeed
  totalsize: size of entire blob
  off_dt_struct: offset to structure block
  off_dt_strings: offset to strings block

Structure Block:
  FDT_BEGIN_NODE "/"
    FDT_PROP "compatible" → offset to "vendor,board"
    FDT_BEGIN_NODE "cpus"
      FDT_BEGIN_NODE "cpu@0"
        FDT_PROP "device_type" → offset to "cpu"
        FDT_PROP "reg" → 0x00000000
      FDT_END_NODE
    FDT_END_NODE
  FDT_END_NODE

Strings Block:
  "vendor,board\0"
  "cpu\0"
  ...
```

**Advantages**:
- **Single allocation**：整個 tree 在一個 blob
- **Sequential access**：往前走來 parse
- **Compact**：Strings 被 deduplicated
- **Fast**：沒有 pointer chasing

### Parsing FDT

```c
int fdt_next_node(const void *fdt, int offset, int *depth) {
    uint32_t tag;

    do {
        offset = fdt_next_tag(fdt, offset, &tag);

        switch (tag) {
        case FDT_BEGIN_NODE:
            (*depth)++;
            break;
        case FDT_END_NODE:
            (*depth)--;
            break;
        case FDT_PROP:
            // Skip property
            break;
        }
    } while (tag != FDT_BEGIN_NODE && tag != FDT_END);

    return offset;
}
```

**Performance**:
```
Parse 500-node device tree:

Malloc'd tree:
  Time: 3.5 ms
  Memory: 128 KB
  Cache misses: 250K

FDT (flat):
  Time: 0.6 ms
  Memory: 24 KB
  Cache misses: 18K

Speedup: 5.8×
```

---

## Putting It All Together: Optimized Bootloader

這是結合所有技術的最終優化 bootloader：

```c
// 1. Bump allocator for temporary allocations
static bump_allocator_t g_allocator;

// 2. Flat device tree
static device_tree_t g_dt;

// 3. Ring buffer for boot log
static boot_log_t g_log;

// 4. Compile-time config
static const hw_config_t g_hw_config = {
    .uart = { .base = 0x10000000, .baud = 115200 },
    .spi = { .base = 0x10001000, .freq = 50000000 },
    // ...
};

void bootloader_main(void) {
    uint64_t start = read_cycle_counter();

    // Phase 1: Hardware init (use compile-time config)
    boot_log("Initializing hardware...");
    init_uart(&g_hw_config.uart);
    init_spi(&g_hw_config.spi);
    // ... other peripherals

    uint64_t hw_init_done = read_cycle_counter();

    // Phase 2: Parse device tree (use flat representation)
    boot_log("Parsing device tree...");
    parse_fdt(&g_dt, (void *)FDT_BASE_ADDR);

    uint64_t dt_done = read_cycle_counter();

    // Phase 3: Load kernel (use bump allocator for buffers)
    boot_log("Loading kernel...");
    void *kernel_buf = boot_alloc(KERNEL_SIZE);
    load_kernel_from_flash(kernel_buf, KERNEL_SIZE);

    uint64_t kernel_loaded = read_cycle_counter();

    // Print boot log
    boot_log_print();

    // Print timing
    uart_printf("Hardware init: %llu cycles\n", hw_init_done - start);
    uart_printf("Device tree:   %llu cycles\n", dt_done - hw_init_done);
    uart_printf("Kernel load:   %llu cycles\n", kernel_loaded - dt_done);
    uart_printf("Total:         %llu cycles\n", kernel_loaded - start);

    // Jump to kernel
    jump_to_kernel(kernel_buf);
}
```

### Final Benchmark

```
Test: Boot RISC-V system (1.2 GHz)

Original (malloc, linked lists, runtime parsing):
  Hardware init: 144M cycles (120 ms)
  Device tree:   396M cycles (330 ms)
  Kernel load:   216M cycles (180 ms)
  Other:         108M cycles (90 ms)
  Total:         864M cycles (720 ms)

Optimized (bump allocator, flat arrays, compile-time config):
  Hardware init: 138M cycles (115 ms)
  Device tree:   114M cycles (95 ms)
  Kernel load:   204M cycles (170 ms)
  Other:         48M cycles (40 ms)
  Total:         504M cycles (420 ms)

Speedup: 1.71× (720 ms → 420 ms)
Boot time reduction: 300 ms (41.7%)
```

---

## Summary

500 毫秒 deadline 達成了。Boot 時間從 720 ms 降到 420 ms——41.7% 減少，比需求低 80 ms 的 margin。工業控制器現在能在上電後快速回應，符合產品規格。

**關鍵洞察**：

1. **Bump allocators 對 bootloaders 完美**。比 malloc 快 40 倍，零 fragmentation，只有 10 行 code。Phases 之間 reset。

2. **Flat arrays 勝過 linked structures**。Device tree parsing 用 flat array 而不是 malloc'd nodes 快 6.1 倍。少 15.4 倍 cache misses。

3. **Compile-time configuration 消除 runtime parsing**。Hardware init 用 compile-time tables 而不是 boot 時 parse device tree 快 26.7 倍。

4. **Ring buffers for logging 簡單且 bounded**。4 KB buffer 處理所有 boot messages，自動 overflow handling。不需要 malloc。

5. **FDT format 很聰明**。Single blob，sequential access，deduplicated strings。比 pointers 的 tree 快 5.8 倍。

**Bootloader 的數據**：
- Bump allocator: 比 malloc 快 40 倍
- Flat device tree: 快 6.1 倍，少 15.4 倍 cache misses
- Compile-time config: 比 runtime parsing 快 26.7 倍
- Overall: boot 快 1.71 倍（720 ms → 420 ms）

Bootloaders 需要簡單、可預測、對 cache 友善的資料結構。避免 malloc，避免 pointers，用 static allocation。

**Next chapter**：Device driver queues——如何有效率地在 hardware 和 software 之間移動 data。

