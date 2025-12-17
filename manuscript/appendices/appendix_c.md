# Appendix C: Tool Reference

This appendix provides a reference for the profiling and analysis tools used throughout this book.

## Overview

The following tools are essential for performance analysis:
- **perf**: Linux performance profiler
- **cachegrind**: Cache profiler (Valgrind)
- **gdb**: GNU debugger
- **objdump**: Object file disassembler
- **readelf**: ELF file analyzer
- **size**: Binary size analyzer

---

## perf: Linux Performance Profiler

### Installation

```bash
# Ubuntu/Debian
sudo apt-get install linux-tools-common linux-tools-generic

# Fedora/RHEL
sudo dnf install perf

# Arch Linux
sudo pacman -S perf
```

### Basic Usage

**Record performance data**:
```bash
perf record -e cycles,cache-misses ./program
```

**View report**:
```bash
perf report
```

**Real-time monitoring**:
```bash
perf top
```

---

### Common Events

**CPU events**:
```bash
perf stat -e cycles,instructions,branches,branch-misses ./program
```

**Cache events**:
```bash
perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses ./program
```

**Memory events**:
```bash
perf stat -e dTLB-loads,dTLB-load-misses,page-faults ./program
```

**All events**:
```bash
perf stat -d ./program  # Detailed statistics
perf stat -dd ./program # Very detailed
```

---

### Event List

**View available events**:
```bash
perf list
```

**Common events**:
- `cycles`: CPU cycles
- `instructions`: Instructions retired
- `cache-references`: Cache accesses
- `cache-misses`: Cache misses (all levels)
- `L1-dcache-loads`: L1 D-cache loads
- `L1-dcache-load-misses`: L1 D-cache load misses
- `LLC-loads`: Last-level cache loads
- `LLC-load-misses`: Last-level cache load misses
- `branches`: Branch instructions
- `branch-misses`: Branch mispredictions
- `dTLB-loads`: Data TLB loads
- `dTLB-load-misses`: Data TLB load misses
- `page-faults`: Page faults

---

### Advanced Usage

**Record with call graph**:
```bash
perf record -g ./program
perf report -g
```

**Record specific function**:
```bash
perf record -e cycles -a --call-graph dwarf -- ./program
```

**Annotate source code**:
```bash
perf record ./program
perf annotate
```

**Differential profiling**:
```bash
perf record -o perf.data.old ./program_old
perf record -o perf.data.new ./program_new
perf diff perf.data.old perf.data.new
```

---

### Example Output

```bash
$ perf stat -e cycles,instructions,cache-misses ./linked_list_test

 Performance counter stats for './linked_list_test':

     1,245,678,901      cycles
       850,234,567      instructions              #    0.68  insn per cycle
        18,456,789      cache-misses              #   14.82 % of all cache refs

       0.520384123 seconds time elapsed
```

**Interpretation**:
- **IPC** (instructions per cycle): 0.68 (low, indicates stalls)
- **Cache miss rate**: 14.82% (high, indicates poor locality)

---

## cachegrind: Cache Profiler

### Installation

```bash
# Ubuntu/Debian
sudo apt-get install valgrind

# Fedora/RHEL
sudo dnf install valgrind

# Arch Linux
sudo pacman -S valgrind
```

### Basic Usage

**Run cachegrind**:
```bash
valgrind --tool=cachegrind ./program
```

**View results**:
```bash
cg_annotate cachegrind.out.<pid>
```

---

### Configuration

**Specify cache sizes**:
```bash
valgrind --tool=cachegrind \
  --I1=32768,8,64 \    # L1 I-cache: 32 KB, 8-way, 64-byte lines
  --D1=32768,8,64 \    # L1 D-cache: 32 KB, 8-way, 64-byte lines
  --LL=2097152,16,64 \ # L2 cache: 2 MB, 16-way, 64-byte lines
  ./program
```

---

### Example Output

```bash
$ valgrind --tool=cachegrind ./linked_list_test
==12345== Cachegrind, a cache and branch-prediction profiler
==12345== 
==12345== I   refs:      850,234,567
==12345== I1  misses:        125,678
==12345== LLi misses:         12,345
==12345== I1  miss rate:        0.01%
==12345== LLi miss rate:        0.00%
==12345== 
==12345== D   refs:      450,123,456  (350,000,000 rd + 100,123,456 wr)
==12345== D1  misses:     18,456,789  ( 15,000,000 rd +   3,456,789 wr)
==12345== LLd misses:      1,234,567  (  1,000,000 rd +     234,567 wr)
==12345== D1  miss rate:        4.1% (        4.3%   +         3.5%  )
==12345== LLd miss rate:        0.3% (        0.3%   +         0.2%  )
==12345== 
==12345== LL refs:        18,582,467  ( 15,125,678 rd +   3,456,789 wr)
==12345== LL misses:       1,246,912  (  1,012,345 rd +     234,567 wr)
==12345== LL miss rate:         0.1% (        0.1%   +         0.2%  )
```

**Interpretation**:
- **D1 miss rate**: 4.1% (data cache misses)
- **LL miss rate**: 0.3% (last-level cache misses)
- Most misses are serviced by L2 cache

---

### Annotated Output

```bash
$ cg_annotate cachegrind.out.12345

--------------------------------------------------------------------------------
Ir          I1mr ILmr Dr          D1mr   DLmr   Dw         D1mw   DLmw  file:function
--------------------------------------------------------------------------------
850,234,567  125  12   450,123,456 18.5M  1.2M   100,123,456 3.4M   234K  linked_list.c:traverse
  5,678,901    5   0     2,345,678   234    12       890,123   45      2  linked_list.c:insert
  ...
```

**Columns**:
- **Ir**: Instruction reads
- **I1mr**: L1 I-cache misses
- **Dr**: Data reads
- **D1mr**: L1 D-cache read misses
- **Dw**: Data writes
- **D1mw**: L1 D-cache write misses

---

## gdb: GNU Debugger

### Basic Usage

**Start debugging**:
```bash
gdb ./program
```

**Common commands**:
```gdb
(gdb) break main          # Set breakpoint at main
(gdb) run                 # Run program
(gdb) next                # Step over
(gdb) step                # Step into
(gdb) continue            # Continue execution
(gdb) print variable      # Print variable value
(gdb) backtrace           # Show call stack
(gdb) quit                # Exit gdb
```

---

### Performance Analysis

**Measure cycles**:
```gdb
(gdb) break function_start
(gdb) commands
> silent
> set $start = $pc
> continue
> end

(gdb) break function_end
(gdb) commands
> silent
> print $pc - $start
> continue
> end

(gdb) run
```

**Inspect memory**:
```gdb
(gdb) x/16xb 0x12345678   # Examine 16 bytes in hex
(gdb) x/4xw 0x12345678    # Examine 4 words in hex
(gdb) x/s 0x12345678      # Examine as string
```

---

## objdump: Object File Disassembler

### Basic Usage

**Disassemble binary**:
```bash
objdump -d ./program
```

**Disassemble specific function**:
```bash
objdump -d ./program | grep -A 20 '<function_name>:'
```

**Show source code**:
```bash
objdump -S ./program  # Requires debug symbols (-g)
```

---

### Example Output

```bash
$ objdump -d linked_list_test

0000000000001234 <traverse>:
    1234:   55                      push   %rbp
    1235:   48 89 e5                mov    %rsp,%rbp
    1238:   48 83 ec 10             sub    $0x10,%rsp
    123c:   48 89 7d f8             mov    %rdi,-0x8(%rbp)
    1240:   48 8b 45 f8             mov    -0x8(%rbp),%rax
    1244:   48 85 c0                test   %rax,%rax
    1247:   74 1a                   je     1263 <traverse+0x2f>
    1249:   48 8b 45 f8             mov    -0x8(%rbp),%rax
    124d:   8b 00                   mov    (%rax),%eax
    124f:   89 c7                   mov    %eax,%edi
    1251:   e8 00 00 00 00          callq  1256 <process>
    1256:   48 8b 45 f8             mov    -0x8(%rbp),%rax
    125a:   48 8b 40 08             mov    0x8(%rax),%rax
    125e:   48 89 45 f8             mov    %rax,-0x8(%rbp)
    1262:   eb dc                   jmp    1240 <traverse+0xc>
    1264:   c9                      leaveq
    1265:   c3                      retq
```

---

## readelf: ELF File Analyzer

### Basic Usage

**Show headers**:
```bash
readelf -h ./program      # ELF header
readelf -l ./program      # Program headers
readelf -S ./program      # Section headers
```

**Show symbols**:
```bash
readelf -s ./program      # Symbol table
```

**Show relocations**:
```bash
readelf -r ./program      # Relocations
```

---

### Example: Section Sizes

```bash
$ readelf -S ./program

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 1] .text             PROGBITS         0000000000001000  00001000
       0000000000002345  0000000000000000  AX       0     0     16
  [ 2] .rodata           PROGBITS         0000000000003400  00003400
       0000000000000890  0000000000000000   A       0     0     8
  [ 3] .data             PROGBITS         0000000000004000  00004000
       0000000000000120  0000000000000000  WA       0     0     8
  [ 4] .bss              NOBITS           0000000000004120  00004120
       0000000000001000  0000000000000000  WA       0     0     8
```

**Interpretation**:
- `.text`: Code (9029 bytes)
- `.rodata`: Read-only data (2192 bytes)
- `.data`: Initialized data (288 bytes)
- `.bss`: Uninitialized data (4096 bytes)

---

## size: Binary Size Analyzer

### Basic Usage

```bash
size ./program
```

**Example output**:
```bash
$ size ./program
   text    data     bss     dec     hex filename
   9029    2480    4096   15605    3cf5 ./program
```

**Interpretation**:
- **text**: Code size (9029 bytes)
- **data**: Initialized data (2480 bytes)
- **bss**: Uninitialized data (4096 bytes)
- **dec**: Total size in decimal (15605 bytes)

---

## Additional Tools

### nm: Symbol Lister

**List symbols**:
```bash
nm ./program
```

**Example output**:
```bash
$ nm ./program
0000000000001234 T traverse
0000000000001567 T insert
0000000000004000 D global_list
0000000000004120 B buffer
```

**Symbol types**:
- **T**: Text (code)
- **D**: Initialized data
- **B**: Uninitialized data (BSS)
- **U**: Undefined (external)

---

### addr2line: Address to Source Line

**Convert address to source line**:
```bash
addr2line -e ./program 0x1234
```

**Example**:
```bash
$ addr2line -e ./program 0x1234
/home/user/project/linked_list.c:42
```

---

### strace: System Call Tracer

**Trace system calls**:
```bash
strace ./program
```

**Count system calls**:
```bash
strace -c ./program
```

**Example output**:
```bash
$ strace -c ./program
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.23    0.012345          12      1024           read
 32.18    0.008765           8      1024           write
 12.34    0.003456          34       100           mmap
  8.25    0.002234          22       100           munmap
  2.00    0.000543           5       100           brk
------ ----------- ----------- --------- --------- ----------------
100.00    0.027343                  2348           total
```

---

### ltrace: Library Call Tracer

**Trace library calls**:
```bash
ltrace ./program
```

**Example output**:
```bash
$ ltrace ./program
malloc(16)                                = 0x555555559260
malloc(16)                                = 0x555555559280
free(0x555555559260)                      = <void>
free(0x555555559280)                      = <void>
```

---

## Compiler Optimization Flags

### GCC/Clang Optimization Levels

**-O0**: No optimization (default)
```bash
gcc -O0 -o program program.c
```

**-O1**: Basic optimization
```bash
gcc -O1 -o program program.c
```

**-O2**: Recommended optimization
```bash
gcc -O2 -o program program.c
```

**-O3**: Aggressive optimization
```bash
gcc -O3 -o program program.c
```

**-Os**: Optimize for size
```bash
gcc -Os -o program program.c
```

**-Ofast**: Aggressive + non-standard optimizations
```bash
gcc -Ofast -o program program.c
```

---

### Useful Flags

**Enable debug symbols**:
```bash
gcc -g -o program program.c
```

**Generate assembly**:
```bash
gcc -S -o program.s program.c
```

**Show optimization report**:
```bash
gcc -O3 -fopt-info-vec -o program program.c
```

**Enable specific optimizations**:
```bash
gcc -O2 -funroll-loops -finline-functions -o program program.c
```

**Disable specific optimizations**:
```bash
gcc -O3 -fno-tree-vectorize -o program program.c
```

---

## Architecture-Specific Tools

### RISC-V

**Spike simulator**:
```bash
spike pk ./program
```

**QEMU emulator**:
```bash
qemu-riscv64 ./program
```

**Disassemble RISC-V binary**:
```bash
riscv64-unknown-elf-objdump -d ./program
```

---

### x86-64

**Intel VTune Profiler**:
```bash
vtune -collect hotspots ./program
vtune -report hotspots
```

**AMD uProf**:
```bash
AMDuProfCLI collect --config tbp ./program
AMDuProfCLI report -i ./program.prd
```

---

### ARM

**ARM Streamline**:
```bash
streamline-cli capture -o capture.apc ./program
```

**perf on ARM**:
```bash
perf stat -e armv8_pmuv3/l1d_cache_refill/ ./program
```

---

## Quick Reference

### Performance Analysis Workflow

1. **Profile with perf**:
```bash
perf record -g ./program
perf report
```

2. **Identify hotspots**:
```bash
perf annotate
```

3. **Analyze cache behavior**:
```bash
perf stat -e cache-misses,L1-dcache-load-misses ./program
```

4. **Detailed cache analysis**:
```bash
valgrind --tool=cachegrind ./program
cg_annotate cachegrind.out.<pid>
```

5. **Optimize code**:
```bash
# Recompile with optimizations
gcc -O3 -march=native -o program program.c
```

6. **Verify improvement**:
```bash
perf stat ./program_old
perf stat ./program_new
```

---

### Common Performance Metrics

**CPU metrics**:
```bash
perf stat -e cycles,instructions,branches,branch-misses ./program
```

**Cache metrics**:
```bash
perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses ./program
```

**Memory metrics**:
```bash
perf stat -e dTLB-loads,dTLB-load-misses,page-faults ./program
```

**All metrics**:
```bash
perf stat -d ./program
```

---

### Interpreting Results

**Good performance indicators**:
- **IPC** (instructions per cycle): > 1.0 (out-of-order CPUs), > 0.8 (in-order CPUs)
- **Cache miss rate**: < 5% (L1), < 1% (L2/L3)
- **Branch miss rate**: < 5%
- **TLB miss rate**: < 1%

**Bad performance indicators**:
- **IPC**: < 0.5 (indicates stalls)
- **Cache miss rate**: > 10% (poor locality)
- **Branch miss rate**: > 10% (unpredictable branches)
- **TLB miss rate**: > 5% (working set too large)

---

### Optimization Checklist

1. **Profile first**: Don't optimize without data
2. **Focus on hotspots**: 80/20 rule applies
3. **Measure cache behavior**: Cache misses are expensive
4. **Check compiler output**: Use `-S` to see assembly
5. **Enable optimizations**: Use `-O2` or `-O3`
6. **Use SIMD**: Vectorize when possible
7. **Reduce branches**: Branchless code is faster
8. **Improve locality**: Keep related data together
9. **Align data**: Align to cache line boundaries
10. **Verify improvement**: Always measure before/after

---

## Summary

This appendix covers essential tools for performance analysis:

**Profiling tools**:
- **perf**: CPU profiling, cache analysis, event counting
- **cachegrind**: Detailed cache simulation
- **gdb**: Debugging and inspection

**Analysis tools**:
- **objdump**: Disassembly and code inspection
- **readelf**: ELF file analysis
- **size**: Binary size analysis
- **nm**: Symbol listing
- **addr2line**: Address to source mapping

**Tracing tools**:
- **strace**: System call tracing
- **ltrace**: Library call tracing

**Compiler flags**:
- **-O0** to **-O3**: Optimization levels
- **-g**: Debug symbols
- **-S**: Generate assembly
- **-fopt-info**: Optimization reports

**Best practices**:
- Profile before optimizing
- Focus on hotspots
- Measure cache behavior
- Verify improvements

---

## QEMU: RISC-V Emulator

### Overview

QEMU is an open-source machine emulator that supports RISC-V architecture. It's essential for:
- Testing RISC-V code without hardware
- Debugging with cycle-accurate simulation
- Running benchmarks in a controlled environment
- Learning RISC-V assembly and system programming

**Supported RISC-V variants**:
- RV32I, RV64I (base integer ISA)
- RV32G, RV64G (general-purpose: IMAFD extensions)
- RV32GC, RV64GC (compressed instructions)
- Vector extension (RVV)

---

### Installation

**Ubuntu/Debian**:
```bash
sudo apt-get update
sudo apt-get install qemu-system-misc
```

**Fedora/RHEL**:
```bash
sudo dnf install qemu-system-riscv
```

**macOS** (via Homebrew):
```bash
brew install qemu
```

**Build from source** (for latest features):
```bash
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
./configure --target-list=riscv32-softmmu,riscv64-softmmu
make -j$(nproc)
sudo make install
```

**Verify installation**:
```bash
qemu-system-riscv64 --version
# Expected: QEMU emulator version 7.0.0 (or later)
```

---

### RISC-V Toolchain

Before using QEMU, install the RISC-V cross-compiler:

**Ubuntu/Debian**:
```bash
sudo apt-get install gcc-riscv64-unknown-elf
```

**Or build from source**:
```bash
git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain
./configure --prefix=/opt/riscv --with-arch=rv64gc --with-abi=lp64d
make -j$(nproc)
export PATH=/opt/riscv/bin:$PATH
```

**Verify**:
```bash
riscv64-unknown-elf-gcc --version
```

---

### Running Bare-Metal Programs

#### Simple Example

**hello.c**:
```c
#include <stdio.h>

int main(void) {
    printf("Hello from RISC-V!\n");
    return 0;
}
```

**Compile**:
```bash
riscv64-unknown-elf-gcc -o hello.elf hello.c
```

**Run on QEMU**:
```bash
qemu-system-riscv64 -machine virt -bios none -kernel hello.elf -nographic
```

**Explanation**:
- `-machine virt`: Use generic RISC-V virtual machine
- `-bios none`: No BIOS/bootloader
- `-kernel hello.elf`: Load ELF directly
- `-nographic`: Console output (no GUI)

**Exit QEMU**: Press `Ctrl-A` then `X`

---

#### QEMU Machines

QEMU provides several RISC-V machine types:

| Machine | Description | Use Case |
|---------|-------------|----------|
| `virt` | Generic virtual machine | General testing, Linux |
| `sifive_e` | SiFive E-series (RV32) | Embedded, bare-metal |
| `sifive_u` | SiFive U-series (RV64) | Application processors |
| `spike` | Spike ISA simulator | ISA testing |

**Example - SiFive E machine**:
```bash
qemu-system-riscv32 -machine sifive_e -nographic -kernel app.elf
```

---

### Running with GDB

QEMU supports remote debugging with GDB:

**Terminal 1 - Start QEMU with GDB server**:
```bash
qemu-system-riscv64 \
  -machine virt \
  -kernel hello.elf \
  -nographic \
  -s \
  -S
```

**Flags**:
- `-s`: Start GDB server on port 1234
- `-S`: Halt CPU at startup (wait for GDB)

**Terminal 2 - Connect GDB**:
```bash
riscv64-unknown-elf-gdb hello.elf

# In GDB:
(gdb) target remote localhost:1234
(gdb) break main
(gdb) continue
(gdb) step
(gdb) info registers
(gdb) x/10i $pc
```

**Common GDB commands**:
```gdb
# Breakpoints
break main
break *0x80000000

# Execution
continue
step
next
finish

# Inspection
info registers
print $pc
print $sp
x/10x $sp
disassemble

# RISC-V specific
info all-registers
print $mstatus
print $mepc
```

---

### Performance Measurement

#### Cycle Counting

QEMU can provide instruction and cycle counts:

**Enable instruction counting**:
```bash
qemu-system-riscv64 \
  -machine virt \
  -kernel benchmark.elf \
  -nographic \
  -icount shift=0
```

**In your code, use RISC-V cycle counter**:
```c
#include <stdint.h>

static inline uint64_t rdcycle(void) {
    uint64_t cycles;
    asm volatile ("rdcycle %0" : "=r" (cycles));
    return cycles;
}

int main(void) {
    uint64_t start = rdcycle();

    // Code to benchmark
    for (int i = 0; i < 1000; i++) {
        // ...
    }

    uint64_t end = rdcycle();
    printf("Cycles: %lu\n", end - start);
    return 0;
}
```

---

#### Instruction Trace

**Generate instruction trace**:
```bash
qemu-system-riscv64 \
  -machine virt \
  -kernel app.elf \
  -nographic \
  -d in_asm,cpu \
  -D trace.log
```

**Trace flags**:
- `in_asm`: Disassemble executed instructions
- `cpu`: CPU state (registers)
- `int`: Interrupts
- `exec`: Execution trace
- `mmu`: Memory management

**Example trace output**:
```
0x80000000:  00000297          auipc   t0,0x0
0x80000004:  02028593          addi    a1,t0,32
0x80000008:  f1402573          csrr    a0,mhartid
```

---

### Memory Configuration

**Specify RAM size**:
```bash
qemu-system-riscv64 -machine virt -m 2G -kernel app.elf -nographic
```

**Memory map for `virt` machine**:
```
0x00001000 - 0x00001FFF  Boot ROM
0x02000000 - 0x0200FFFF  CLINT (timer, IPI)
0x0C000000 - 0x0FFFFFFF  PLIC (interrupts)
0x10000000 - 0x100000FF  UART
0x80000000 - ...         RAM (default: 128 MB)
```

---

### Common Use Cases

#### Running Benchmarks

```bash
# Compile benchmark
riscv64-unknown-elf-gcc -O3 -march=rv64gc -o coremark.elf coremark.c

# Run on QEMU
qemu-system-riscv64 -machine virt -m 1G -kernel coremark.elf -nographic
```

#### Testing Different ISA Extensions

```bash
# RV64GC (with compressed instructions)
riscv64-unknown-elf-gcc -march=rv64gc -o app.elf app.c
qemu-system-riscv64 -cpu rv64,c=true -machine virt -kernel app.elf -nographic

# RV64G (without compressed)
riscv64-unknown-elf-gcc -march=rv64g -o app.elf app.c
qemu-system-riscv64 -cpu rv64,c=false -machine virt -kernel app.elf -nographic
```

#### Semihosting (for printf)

If your program uses `printf` but has no UART driver:

```bash
qemu-system-riscv64 \
  -machine virt \
  -kernel app.elf \
  -nographic \
  -semihosting
```

---

### Troubleshooting

#### Program doesn't output anything

**Problem**: No UART driver or wrong memory map

**Solution 1**: Use semihosting
```bash
qemu-system-riscv64 -machine virt -kernel app.elf -nographic -semihosting
```

**Solution 2**: Use QEMU's built-in UART (0x10000000)
```c
#define UART_BASE 0x10000000

void uart_putc(char c) {
    *(volatile char *)UART_BASE = c;
}

void uart_puts(const char *s) {
    while (*s) uart_putc(*s++);
}
```

---

#### QEMU hangs or crashes

**Problem**: Infinite loop or illegal instruction

**Solution**: Use GDB to debug
```bash
# Terminal 1
qemu-system-riscv64 -machine virt -kernel app.elf -nographic -s -S

# Terminal 2
riscv64-unknown-elf-gdb app.elf
(gdb) target remote :1234
(gdb) break main
(gdb) continue
```

---

#### Wrong architecture

**Problem**: Compiled for RV32 but running on RV64 QEMU

**Solution**: Match architecture
```bash
# For RV32
riscv32-unknown-elf-gcc -o app.elf app.c
qemu-system-riscv32 -machine virt -kernel app.elf -nographic

# For RV64
riscv64-unknown-elf-gcc -o app.elf app.c
qemu-system-riscv64 -machine virt -kernel app.elf -nographic
```

---

### QEMU vs Real Hardware

**QEMU advantages**:
- ✅ No hardware needed
- ✅ Deterministic execution
- ✅ Easy debugging with GDB
- ✅ Fast iteration

**QEMU limitations**:
- ❌ Not cycle-accurate (timing differs from real hardware)
- ❌ Simplified cache model
- ❌ No real I/O devices
- ❌ Different performance characteristics

**Best practice**: Use QEMU for functional testing and debugging, verify on real hardware for performance.

---

### Quick Reference

**Basic run**:
```bash
qemu-system-riscv64 -machine virt -kernel app.elf -nographic
```

**With GDB**:
```bash
qemu-system-riscv64 -machine virt -kernel app.elf -nographic -s -S
```

**With trace**:
```bash
qemu-system-riscv64 -machine virt -kernel app.elf -nographic -d in_asm -D trace.log
```

**Exit QEMU**: `Ctrl-A` then `X`

---

For detailed examples of using these tools, see the individual chapters.
