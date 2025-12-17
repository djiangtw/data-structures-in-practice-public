# Chapter 19: Firmware Memory Management

**Part V: Case Studies**

---

> "Controlling complexity is the essence of computer programming."
> — Brian Kernighan

## The Final Testing Phase

We were in the final testing phase of an IoT sensor project—a smart building device with 128 KB of RAM that monitored temperature, humidity, and air quality. The firmware had passed all functional tests. Unit tests: green. Integration tests: green. Power consumption: within spec.

The last requirement was a 72-hour continuous operation test. We set up twelve devices in the lab, configured them to report sensor data every second, and let them run.

After three days, I came into the lab expecting to collect the test logs and close the project.

Instead, I found all twelve devices had crashed.

The serial console showed the same error on every device:

```
[72:14:23] malloc failed: out of memory
[72:14:23] Fragmentation: 45%
[72:14:23] System halted
```

My stomach sank. The project was supposed to ship in two weeks. I knew exactly what had happened—and I knew it was going to be painful to fix.

## The Textbook Approach

When I'd designed the firmware months earlier, I'd used what seemed like reasonable practices. The device needed to:
- Handle network communication (TCP/IP stack)
- Process sensor data every second
- Store configuration
- Perform OTA (Over-The-Air) updates

I used malloc/free from newlib, just like the textbooks teach:

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

Simple. Clean. Textbook-correct.

And after 72 hours of continuous operation, it killed all twelve devices.

## The Autopsy

I pulled the memory trace from the crash dump:

```
[72:14:23] malloc failed: out of memory
[72:14:23] Available: 8 KB
[72:14:23] Requested: 16 KB
[72:14:23] Fragmentation: 45%
[72:14:23] Total allocations: 259,200 (72 hours × 3600 seconds/hour)
[72:14:23] Average allocation size: 156 bytes
[72:14:23] System halted
```

**45% fragmentation.** Out of 128 KB of RAM, only 8 KB was available in contiguous blocks. The firmware needed 16 KB for a network buffer, but couldn't find it.

The problem wasn't a memory leak—we were freeing everything correctly. The problem was **fragmentation**.

After 259,200 allocations and frees over 72 hours, the heap looked like Swiss cheese:

```
Initial state (128 KB free):
[                                                    ]

After 72 hours:
[used][free][used][free][used][free][used][free]...
      4KB       2KB       8KB       1KB

Total free: 58 KB
Largest contiguous: 8 KB
Can't allocate 16 KB!
```

I had two weeks before the scheduled ship date. I needed a solution that wouldn't require rewriting the entire firmware.

---

## The Redesign: Finding a Path Forward

I couldn't rewrite the entire firmware in two weeks, but I could fix the memory management.

The key insight: **our allocations fell into predictable patterns**.

I analyzed the crash dump and found:
- **Sensor data**: 156 bytes, allocated every second
- **Network packets**: 1024 bytes, allocated every 5 seconds
- **Configuration**: 2048 bytes, allocated once at startup
- **Temporary buffers**: 256 bytes, allocated during processing

All predictable sizes. All predictable lifetimes.

I didn't need a general-purpose allocator. I needed **specialized allocators** for each use case.

## Strategy 1: Fixed-Size Memory Pools

For the sensor data (156 bytes, allocated every second), I created a fixed-size pool:

```c
#define SENSOR_POOL_SIZE 256  // Round up to power of 2
#define SENSOR_POOL_COUNT 10  // Max 10 concurrent readings

typedef struct free_block {
    struct free_block *next;
} free_block_t;

typedef struct {
    uint8_t memory[SENSOR_POOL_SIZE * SENSOR_POOL_COUNT];
    free_block_t *free_list;
} sensor_pool_t;

static sensor_pool_t g_sensor_pool;

void sensor_pool_init(void) {
    g_sensor_pool.free_list = NULL;

    // Link all blocks into free list
    for (int i = 0; i < SENSOR_POOL_COUNT; i++) {
        free_block_t *block = (free_block_t *)&g_sensor_pool.memory[i * SENSOR_POOL_SIZE];
        block->next = g_sensor_pool.free_list;
        g_sensor_pool.free_list = block;
    }
}

void *sensor_alloc(void) {
    if (!g_sensor_pool.free_list) {
        return NULL;  // Pool exhausted
    }

    void *ptr = g_sensor_pool.free_list;
    g_sensor_pool.free_list = g_sensor_pool.free_list->next;
    return ptr;
}

void sensor_free(void *ptr) {
    free_block_t *block = (free_block_t *)ptr;
    block->next = g_sensor_pool.free_list;
    g_sensor_pool.free_list = block;
}
```

Simple. **O(1) allocation and free.** Zero fragmentation.

I benchmarked it:

```
Test: 10,000 sensor readings (256-byte blocks)

malloc/free:
  Cycles: 2.4M (240 cycles per operation)
  Fragmentation: 18%
  Time: 2.0 ms

Fixed-size pool (free list):
  Cycles: 120K (12 cycles per operation)
  Fragmentation: 0%
  Time: 0.10 ms

Speedup: 20×
```

**20× faster** and zero fragmentation. That solved the sensor data problem.

---

## Strategy 2: Static Allocation for Network Buffers

The network stack needed buffers for TCP/IP communication. The original code allocated them dynamically:

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
- **Zero fragmentation**: No heap usage
- **Zero overhead**: No metadata
- **Predictable**: Known at compile time
- **Fast**: No allocation time

**Trade-off**: Uses RAM even when not needed. But for firmware, this is usually acceptable.

---

## Solution 3: Stack-Based Allocation

For temporary buffers, use the stack:

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
- **Fastest**: Just adjust stack pointer
- **No fragmentation**: Stack grows/shrinks cleanly
- **Automatic cleanup**: No need to free

**Limitation**: Stack size is limited (typically 4-16 KB). Don't allocate large buffers on stack.

---

## Solution 4: Memory Regions

Partition memory into regions for different purposes:

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
- **Clear boundaries**: Each region has fixed size
- **No interference**: DMA doesn't corrupt stack
- **Easy debugging**: Know which region is full
- **Cache-friendly**: Related data in same region

---

## Solution 5: Slab Allocator

For objects of the same type, use a slab allocator:

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
- **O(1) allocation**: Just find first set bit
- **Cache-friendly**: All connections contiguous
- **Type-safe**: Can only allocate connection_t
- **Low overhead**: 1 bit per object

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

FreeRTOS offers multiple heap implementations:

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

**Use case**: Systems that never free memory (allocate at startup only).

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

**Use case**: General-purpose allocation with some fragmentation tolerance.

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

**Use case**: Systems with multiple RAM regions (internal SRAM + external DRAM).

---

## Putting It All Together: Optimized Firmware Memory

Here's the final optimized firmware combining all techniques:

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

After implementing the memory pool redesign, I reflashed all twelve devices and restarted the 72-hour test.

This time, I monitored the memory usage continuously. After 15 minutes, the pattern was already clear: memory usage had stabilized at 96 KB, and fragmentation remained at zero.

After 72 hours, all twelve devices were still running. After a week, still running. We eventually let the test run for three months—no crashes, no memory issues, no fragmentation.

## What I Learned

The firmware redesign taught me that **malloc/free is the wrong tool for embedded systems**.

Here's what worked:

**1. Fixed-size pools eliminate fragmentation**

Pre-allocating blocks of fixed sizes (256 bytes for sensors, 1024 bytes for network packets) provides O(1) allocation with zero fragmentation. The sensor pool was **20× faster** than malloc.

**2. Static allocation for long-lived objects**

Network buffers, configuration data, and other persistent objects should be statically allocated. Zero overhead, zero fragmentation, and the memory layout is known at compile time.

**3. Stack allocation for temporary buffers**

Short-lived buffers (like temporary processing buffers) should use the stack. Fastest allocation—just adjust the stack pointer—and automatic cleanup when the function returns.

**4. Slab allocators for uniform objects**

Connection pools and packet buffers benefit from slab allocation. Using a bitmap for free/used tracking provides O(1) allocation with just 1 bit of overhead per object. **13.3× faster** than malloc.

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

The lesson: **firmware needs predictable, deterministic memory management**. Avoid malloc/free. Use pools, static allocation, and slab allocators.

And always run extended testing—72 hours minimum—before declaring a project complete. The bugs that matter most are the ones that only appear after days of continuous operation.

---

## Summary

**Key insights**:
- malloc/free causes fragmentation in long-running firmware
- Fixed-size pools: 20× faster, zero fragmentation
- Static allocation: Best for long-lived objects
- Stack allocation: Best for temporary buffers
- Slab allocators: 13.3× faster for uniform objects

**The IoT sensor firmware**:
- 18.6% less memory (118 KB → 96 KB)
- Zero fragmentation (45% → 0%)
- 20× faster allocation (240 cycles → 12 cycles)
- Zero crashes (ran for 3+ months continuously)

**Takeaway**: In embedded systems, predictability matters more than flexibility. Design your memory management for your specific workload, not for general-purpose use.

