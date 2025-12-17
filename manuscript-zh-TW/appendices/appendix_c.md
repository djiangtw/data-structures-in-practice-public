# Appendix C: Tool Reference

這個附錄提供本書使用的 profiling 和 analysis tools 的參考。

## Overview

以下 tools 對 performance analysis 必要：
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
- **IPC** (instructions per cycle): 0.68（低，表示 stalls）
- **Cache miss rate**: 14.82%（高，表示 poor locality）

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

### Example Output

```
I   refs:      1,000,000,000
I1  misses:        1,234,567
LLi misses:          123,456
I1  miss rate:          0.12%
LLi miss rate:          0.01%

D   refs:        500,000,000  (400,000,000 rd + 100,000,000 wr)
D1  misses:       18,456,789  ( 15,234,567 rd +   3,222,222 wr)
LLd misses:        2,345,678  (  1,234,567 rd +   1,111,111 wr)
D1  miss rate:           3.7% (       3.8%     +         3.2%  )
LLd miss rate:           0.5% (       0.3%     +         1.1%  )

LL refs:          19,691,356  ( 16,469,134 rd +   3,222,222 wr)
LL misses:         2,469,134  (  1,358,023 rd +   1,111,111 wr)
LL miss rate:            0.2% (       0.1%     +         1.1%  )
```

**Interpretation**:
- **I1**: L1 instruction cache
- **D1**: L1 data cache
- **LL**: Last-level cache
- **D1 miss rate 3.7%**: 每 100 次 data accesses 有 3.7 次 L1 misses

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

---

## size: Binary Size Analyzer

### Basic Usage

```bash
size ./program
```

**Output**:
```
   text    data     bss     dec     hex filename
   9029    2192    4096   15317    3bd5 program
```

**Interpretation**:
- **text**: Code size (9029 bytes)
- **data**: Initialized data (2192 bytes)
- **bss**: Uninitialized data (4096 bytes)
- **dec**: Total size in decimal (15317 bytes)

---

## Compiler Optimization Flags

### GCC/Clang Optimization Levels

```bash
-O0   # No optimization (default)
-O1   # Basic optimization
-O2   # Moderate optimization (recommended)
-O3   # Aggressive optimization
-Os   # Optimize for size
-Ofast # -O3 + fast math (non-standard compliant)
```

### Architecture-Specific Flags

**RISC-V**:
```bash
-march=rv64gc        # RV64 with G (IMAFD) + C extensions
-march=rv64imafdc    # Explicit extensions
-mtune=sifive-7      # Tune for SiFive U7 cores
```

**x86-64**:
```bash
-march=native        # Use all available CPU features
-march=x86-64-v3     # x86-64-v3 microarchitecture level
-mavx2               # Enable AVX2 instructions
```

**ARM**:
```bash
-mcpu=cortex-a72     # Target Cortex-A72
-march=armv8-a       # ARMv8-A architecture
-mfpu=neon           # Enable NEON SIMD
```

---

## Useful Compiler Flags

### Debug Information

```bash
-g                   # Generate debug symbols
-g3                  # Maximum debug info (includes macros)
-ggdb                # GDB-specific debug info
```

### Warnings

```bash
-Wall                # Enable common warnings
-Wextra              # Extra warnings
-Werror              # Treat warnings as errors
-Wpedantic           # Strict ISO C compliance
```

### Performance Analysis

```bash
-fno-omit-frame-pointer  # Keep frame pointer (for profiling)
-pg                      # Generate gprof profiling info
-fprofile-generate       # Generate profile data
-fprofile-use            # Use profile data (PGO)
```

---

## Quick Reference Card

### perf

```bash
# Basic profiling
perf stat ./program

# Record and report
perf record ./program
perf report

# Cache analysis
perf stat -e cache-misses,L1-dcache-load-misses ./program

# Call graph
perf record -g ./program
perf report -g
```

### cachegrind

```bash
# Run cachegrind
valgrind --tool=cachegrind ./program

# View results
cg_annotate cachegrind.out.<pid>

# Specific cache configuration
valgrind --tool=cachegrind --I1=32768,8,64 --D1=32768,8,64 --LL=2097152,16,64 ./program
```

### objdump

```bash
# Disassemble
objdump -d ./program

# With source
objdump -S ./program

# Specific function
objdump -d ./program | grep -A 50 '<function>:'
```

### gdb

```bash
# Start
gdb ./program

# Common commands
break main
run
next / step
print var
backtrace
quit
```

---

## Troubleshooting

### perf: Permission Denied

```bash
# Temporary fix
sudo sysctl -w kernel.perf_event_paranoid=-1

# Permanent fix (add to /etc/sysctl.conf)
kernel.perf_event_paranoid = -1
```

### cachegrind: Slow Execution

Cachegrind 用 simulation，比 native execution 慢 20-50×。對大 programs，用 `--cache-sim=no` 來 disable cache simulation。

### objdump: No Symbols

Compile 時加 `-g` flag：
```bash
gcc -g -O2 -o program program.c
```

---

## Further Reading

- **perf**: https://perf.wiki.kernel.org/
- **Valgrind**: https://valgrind.org/docs/manual/
- **GDB**: https://sourceware.org/gdb/documentation/
- **GCC**: https://gcc.gnu.org/onlinedocs/

---

