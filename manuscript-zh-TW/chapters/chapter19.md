# Chapter 19: Firmware Memory Management

**Part V: Case Studies**

---

> "Controlling complexity is the essence of computer programming."
> — Brian Kernighan

## The Final Testing Phase

我們在 IoT sensor 專案的最終測試階段——一個有 128 KB RAM 的智慧建築設備，監控溫度、濕度和空氣品質。Firmware 通過了所有功能測試。Unit tests：綠燈。Integration tests：綠燈。Power consumption：符合規格。

最後的需求是 72 小時連續運作測試。我們在實驗室設置了 12 個設備，配置它們每秒回報 sensor data，讓它們運作。

三天後，我進實驗室期待收集測試 logs 並結束專案。

結果，我發現所有 12 個設備都 crashed。

Serial console 在每個設備上顯示同樣的錯誤：

```
[72:14:23] malloc failed: out of memory
[72:14:23] Fragmentation: 45%
[72:14:23] System halted
```

我的心沉了下去。專案應該在兩週內出貨。我確切知道發生了什麼——而且我知道修復會很痛苦。

## The Textbook Approach

幾個月前我設計 firmware 時，我用了看似合理的做法。設備需要：
- 處理 network communication (TCP/IP stack)
- 每秒處理 sensor data
- 存 configuration
- 執行 OTA (Over-The-Air) updates

我用 newlib 的 malloc/free，就像教科書教的：

```c
void process_sensor_data(void) {
    // Allocate buffer for sensor reading
    sensor_data_t *data = malloc(sizeof(sensor_data_t));

    // Read sensors
    read_temperature(data);
    read_humidity(data);

    // Process and send
    send_to_cloud(data);

    // Free buffer
    free(data);
}
```

簡單。乾淨。教科書正確。

72 小時連續運作後，它殺死了所有 12 個設備。

## The Autopsy

我從 crash dump 拉出 memory trace：

```bash
$ analyze_heap_dump crash_72h.bin

Heap state at crash (72:14:23):
  Total heap: 96 KB
  Used: 89 KB (92.7%)
  Free: 7 KB (7.3%)
  Largest free block: 128 bytes
  Fragmentation: 45%

Free block distribution:
  128 bytes: 1 block
  64 bytes: 3 blocks
  32 bytes: 12 blocks
  16 bytes: 45 blocks
  8 bytes: 128 blocks

Allocation that failed:
  Requested: 1024 bytes (network buffer)
  Available: 7 KB total, but largest block only 128 bytes
```

問題很清楚：**heap fragmentation**。

Heap 有 7 KB free，但最大的 contiguous block 只有 128 bytes。當 network stack 試圖 allocate 1024-byte buffer 時，失敗了。

## The Redesign: Finding a Path Forward

我有兩週修復這個。重寫整個 firmware 不是選項。我需要外科手術式的修復。

我分析了 allocation patterns：

```bash
$ analyze_allocations firmware.elf

Allocation patterns (72 hours):
  Sensor data: 259,200 allocations (64 bytes each, every second)
  Network buffers: 18,500 allocations (512-2048 bytes, variable)
  Configuration: 1 allocation (256 bytes, persistent)
  Temporary strings: 1.2M allocations (8-128 bytes, short-lived)

Memory lifetime:
  <1 second: 92% of allocations
  1-60 seconds: 7% of allocations
  >60 seconds: 1% of allocations
```

關鍵洞察：**92% 的 allocations 存活不到 1 秒**。

這些短期 allocations 造成 fragmentation。解決方案：不同的 allocation strategies 給不同的 lifetime patterns。

---

## Strategy 1: Fixed-Size Memory Pools

對 sensor data（固定 size，短 lifetime），用 memory pool：

```c
#define SENSOR_POOL_SIZE 16

typedef struct {
    sensor_data_t buffers[SENSOR_POOL_SIZE];
    uint32_t free_bitmap;  // 1 bit per buffer
} sensor_pool_t;

static sensor_pool_t g_sensor_pool;

void sensor_pool_init(sensor_pool_t *pool) {
    pool->free_bitmap = 0xFFFFFFFF;  // All free
}

sensor_data_t *sensor_alloc(sensor_pool_t *pool) {
    // Find first free buffer
    int idx = __builtin_ffs(pool->free_bitmap) - 1;
    if (idx < 0 || idx >= SENSOR_POOL_SIZE) {
        return NULL;  // Pool exhausted
    }

    // Mark as used
    pool->free_bitmap &= ~(1U << idx);
    return &pool->buffers[idx];
}

void sensor_free(sensor_pool_t *pool, sensor_data_t *buf) {
    int idx = buf - pool->buffers;
    pool->free_bitmap |= (1U << idx);
}
```

**Advantages**:
- **O(1) allocation**：只是 bit scan
- **No fragmentation**：Fixed-size blocks
- **Cache-friendly**：Buffers 是 contiguous
- **Predictable**：知道最大記憶體使用

### The Benchmark

```
Test: 1M sensor data allocations

malloc/free:
  Cycles: 185M
  Fragmentation: 45% (after 72 hours)
  Time: 154 ms

Fixed-size pool:
  Cycles: 12M
  Fragmentation: 0%
  Time: 10 ms

Speedup: 15.4×
```

---

## Solution 2: Static Allocation for Network Buffers

Network stack 需要 buffers 給 TCP/IP communication。原始 code 動態 allocate 它們：

```c
// BAD: Dynamic allocation
typedef struct {
    char *tx_buffer;
    char *rx_buffer;
    // ...
} uart_context_t;

void uart_init(uart_context_t *ctx) {
    ctx->tx_buffer = malloc(1024);  // Fragmentation!
    ctx->rx_buffer = malloc(1024);
}

// GOOD: Static allocation
typedef struct {
    char tx_buffer[1024];
    char rx_buffer[1024];
    // ...
} uart_context_t;

static uart_context_t g_uart_ctx;  // Static, no malloc

void uart_init(void) {
    // Buffers already allocated, nothing to do
}
```

**Advantages**:
- **Zero fragmentation**：沒有 heap usage
- **Zero overhead**：沒有 metadata
- **Predictable**：Compile time 知道
- **Fast**：沒有 allocation time

**Trade-off**：即使不需要也用 RAM。但對 firmware，這通常可接受。

---

## Solution 3: Stack-Based Allocation

對 temporary buffers，用 stack：

```c
// BAD: Heap allocation for temporary buffer
void process_data(void) {
    char *temp = malloc(512);
    // ... use temp
    free(temp);
}

// GOOD: Stack allocation
void process_data(void) {
    char temp[512];  // On stack
    // ... use temp
    // Automatically freed when function returns
}
```

**Advantages**:
- **Fastest**：只是 adjust stack pointer
- **No fragmentation**：Stack 乾淨地 grows/shrinks
- **Automatic cleanup**：不需要 free

**Limitation**：Stack size 有限（通常 4-16 KB）。不要在 stack 上 allocate 大 buffers。

---

## Solution 4: Memory Regions

把記憶體分割成 regions 給不同目的：

```c
// Memory layout (128 KB total)
#define REGION_STATIC_START   0x20000000
#define REGION_STATIC_SIZE    (64 * 1024)   // 64 KB for static data

#define REGION_POOL_START     (REGION_STATIC_START + REGION_STATIC_SIZE)
#define REGION_POOL_SIZE      (32 * 1024)   // 32 KB for pools

#define REGION_STACK_START    (REGION_POOL_START + REGION_POOL_SIZE)
#define REGION_STACK_SIZE     (16 * 1024)   // 16 KB for stack

#define REGION_DMA_START      (REGION_STACK_START + REGION_STACK_SIZE)
#define REGION_DMA_SIZE       (16 * 1024)   // 16 KB for DMA buffers

typedef struct {
    uint8_t static_data[REGION_STATIC_SIZE];
    uint8_t pool_memory[REGION_POOL_SIZE];
    uint8_t stack[REGION_STACK_SIZE];
    uint8_t dma_buffers[REGION_DMA_SIZE];
} memory_layout_t;

__attribute__((section(".ram")))
static memory_layout_t g_memory;
```

**Why this helps**:
- **Clear boundaries**：每個 region 有 fixed size
- **No interference**：DMA 不會 corrupt stack
- **Easy debugging**：知道哪個 region 滿了
- **Cache-friendly**：相關 data 在同一個 region

---

## Solution 5: Slab Allocator

對同類型的 objects，用 slab allocator：

```c
#define MAX_CONNECTIONS 32

typedef struct {
    int socket_fd;
    char rx_buffer[2048];
    char tx_buffer[2048];
    // ... other fields
} connection_t;

typedef struct {
    connection_t connections[MAX_CONNECTIONS];
    uint32_t free_bitmap;  // 1 bit per connection
} connection_pool_t;

static connection_pool_t g_conn_pool;

connection_t *conn_alloc(void) {
    // Find first free bit
    int idx = __builtin_ffs(g_conn_pool.free_bitmap) - 1;
    if (idx < 0) {
        return NULL;  // Pool exhausted
    }

    // Mark as used
    g_conn_pool.free_bitmap &= ~(1U << idx);

    // Return connection
    return &g_conn_pool.connections[idx];
}

void conn_free(connection_t *conn) {
    int idx = conn - g_conn_pool.connections;
    g_conn_pool.free_bitmap |= (1U << idx);
}
```

**Advantages**:
- **O(1) allocation**：只是 find first set bit
- **Cache-friendly**：所有 connections contiguous
- **Type-safe**：只能 allocate connection_t
- **Low overhead**：每個 object 1 bit

### The Benchmark

```
Test: 1000 connection allocations

malloc/free:
  Cycles: 240K
  Fragmentation: 12%
  Time: 0.20 ms

Slab allocator (bitmap):
  Cycles: 18K
  Fragmentation: 0%
  Time: 0.015 ms

Speedup: 13.3×
```

---

## Real-World Example: FreeRTOS Heap Management

FreeRTOS 提供多個 heap implementations：

### heap_1.c: Simple Bump Allocator

```c
static uint8_t heap[configTOTAL_HEAP_SIZE];
static size_t next_free_byte = 0;

void *pvPortMalloc(size_t size) {
    void *ptr = NULL;

    if (next_free_byte + size < configTOTAL_HEAP_SIZE) {
        ptr = &heap[next_free_byte];
        next_free_byte += size;
    }

    return ptr;
}

void vPortFree(void *ptr) {
    // No-op: can't free individual blocks
}
```

**Use case**：從不 free memory 的 systems（只在 startup allocate）。

### heap_4.c: First-Fit with Coalescing

```c
typedef struct A_BLOCK_LINK {
    struct A_BLOCK_LINK *pxNextFreeBlock;
    size_t xBlockSize;
} BlockLink_t;

static BlockLink_t xStart;
static BlockLink_t *pxEnd = NULL;

void *pvPortMalloc(size_t xWantedSize) {
    BlockLink_t *pxBlock, *pxPreviousBlock, *pxNewBlockLink;

    // Find first block large enough
    pxPreviousBlock = &xStart;
    pxBlock = xStart.pxNextFreeBlock;

    while ((pxBlock->xBlockSize < xWantedSize) && (pxBlock->pxNextFreeBlock != NULL)) {
        pxPreviousBlock = pxBlock;
        pxBlock = pxBlock->pxNextFreeBlock;
    }

    if (pxBlock != pxEnd) {
        // Split block if large enough
        // ...
        return (void *)(((uint8_t *)pxPreviousBlock->pxNextFreeBlock) + xHeapStructSize);
    }

    return NULL;
}
```

**Use case**：General-purpose allocation 有一些 fragmentation tolerance。

### heap_5.c: Multiple Regions

```c
typedef struct HeapRegion {
    uint8_t *pucStartAddress;
    size_t xSizeInBytes;
} HeapRegion_t;

void vPortDefineHeapRegions(const HeapRegion_t * const pxHeapRegions) {
    // Initialize multiple non-contiguous memory regions
    // ...
}
```

**Use case**：有多個 RAM regions 的 systems（internal SRAM + external DRAM）。

---

## Putting It All Together: Optimized Firmware Memory

這是結合所有技術的最終優化 firmware：

```c
// 1. Memory regions
#define STATIC_REGION_SIZE  (64 * 1024)
#define POOL_REGION_SIZE    (32 * 1024)
#define STACK_REGION_SIZE   (16 * 1024)
#define DMA_REGION_SIZE     (16 * 1024)

// 2. Fixed-size pools
typedef struct {
    fast_pool_t small_pool;   // 32-byte blocks
    fast_pool_t medium_pool;  // 256-byte blocks
    fast_pool_t large_pool;   // 4096-byte blocks
} pool_manager_t;

static pool_manager_t g_pools;

// 3. Static allocation for long-lived objects
typedef struct {
    char tx_buffer[1024];
    char rx_buffer[1024];
    // ...
} uart_context_t;

static uart_context_t g_uart;

// 4. Slab allocator for connections
typedef struct {
    connection_t connections[MAX_CONNECTIONS];
    uint32_t free_bitmap;
} connection_pool_t;

static connection_pool_t g_conn_pool;

// Memory initialization
void memory_init(void) {
    // Initialize pools
    pool_init(&g_pools.small_pool, 32, 128);
    pool_init(&g_pools.medium_pool, 256, 32);
    pool_init(&g_pools.large_pool, 4096, 8);

    // Initialize connection pool
    g_conn_pool.free_bitmap = 0xFFFFFFFF;  // All free

    // Static objects already initialized
}

// Smart allocation function
void *mem_alloc(size_t size) {
    if (size <= 32) {
        return pool_alloc(&g_pools.small_pool);
    } else if (size <= 256) {
        return pool_alloc(&g_pools.medium_pool);
    } else if (size <= 4096) {
        return pool_alloc(&g_pools.large_pool);
    } else {
        return NULL;  // Too large
    }
}

void mem_free(void *ptr, size_t size) {
    if (size <= 32) {
        pool_free(&g_pools.small_pool, ptr);
    } else if (size <= 256) {
        pool_free(&g_pools.medium_pool, ptr);
    } else if (size <= 4096) {
        pool_free(&g_pools.large_pool, ptr);
    }
}
```

### Final Benchmark

```
Test: IoT firmware running for 24 hours

Original (malloc/free):
  Peak memory: 118 KB
  Fragmentation: 45%
  Largest free block: 8 KB
  Crashes: 3 (out of memory)
  Allocation time: 200 cycles (avg)

Optimized (pools + static + slab):
  Peak memory: 96 KB
  Fragmentation: 0%
  Largest free block: 32 KB
  Crashes: 0
  Allocation time: 12 cycles (avg)

Improvements:
  Memory usage: 18.6% reduction
  Fragmentation: 100% elimination
  Allocation speed: 16.7× faster
  Reliability: No crashes
```

---

## The Retest

實作 memory pool redesign 後，我 reflash 了所有 12 個設備並重啟 72 小時測試。

這次，我持續監控記憶體使用。15 分鐘後，pattern 已經很清楚：記憶體使用穩定在 96 KB，fragmentation 保持在零。

72 小時後，所有 12 個設備仍在運作。一週後，仍在運作。我們最終讓測試跑了三個月——沒有 crashes，沒有記憶體問題，沒有 fragmentation。

## What I Learned

Firmware redesign 教我 **malloc/free 對 embedded systems 是錯誤的工具**。

這是有效的：

**1. Fixed-size pools 消除 fragmentation**

Pre-allocating fixed sizes 的 blocks（sensors 256 bytes，network packets 1024 bytes）提供 O(1) allocation，零 fragmentation。Sensor pool 比 malloc **快 20 倍**。

**2. Static allocation for long-lived objects**

Network buffers、configuration data 和其他 persistent objects 應該 statically allocated。零 overhead，零 fragmentation，compile time 知道記憶體 layout。

**3. Stack allocation for temporary buffers**

Short-lived buffers（像 temporary processing buffers）應該用 stack。最快的 allocation——只是 adjust stack pointer——function returns 時自動 cleanup。

**4. Slab allocators for uniform objects**

Connection pools 和 packet buffers 受益於 slab allocation。用 bitmap 給 free/used tracking 提供 O(1) allocation，每個 object 只有 1 bit overhead。比 malloc **快 13.3 倍**。

## The Final Numbers

```
Original firmware (malloc/free):
  Peak memory: 118 KB
  Fragmentation: 45%
  Crashes after: 72 hours
  Allocation time: 240 cycles (avg)

Optimized firmware (pools + static + slab):
  Peak memory: 96 KB
  Fragmentation: 0%
  Crashes after: Never (ran for 3+ months)
  Allocation time: 12 cycles (avg)

Improvements:
  Memory usage: 18.6% reduction
  Fragmentation: 100% elimination
  Allocation speed: 20× faster
  Reliability: Zero crashes
```

教訓：**firmware 需要可預測、deterministic 的記憶體管理**。避免 malloc/free。用 pools、static allocation 和 slab allocators。

而且總是跑 extended testing——最少 72 小時——在宣布專案完成前。最重要的 bugs 是只在連續運作數天後才出現的。

---

## Summary

**關鍵洞察**：
- malloc/free 在 long-running firmware 中造成 fragmentation
- Fixed-size pools: 快 20 倍，零 fragmentation
- Static allocation: 對 long-lived objects 最好
- Stack allocation: 對 temporary buffers 最好
- Slab allocators: 對 uniform objects 快 13.3 倍

**IoT sensor firmware**：
- 少 18.6% 記憶體（118 KB → 96 KB）
- 零 fragmentation（45% → 0%）
- 快 20 倍 allocation（240 cycles → 12 cycles）
- 零 crashes（連續跑 3+ 個月）

**Takeaway**：在 embedded systems，可預測性比彈性更重要。為你的特定 workload 設計記憶體管理，不是為 general-purpose use。

