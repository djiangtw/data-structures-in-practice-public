# Chapter 11: Tries and Radix Trees

**Part III: Trees and Hierarchies**

---

> "The cheapest, fastest, and most reliable components are those that aren't there."
> — Gordon Bell

## The Autocomplete Disaster

The trie was 8× slower than a hash table. And it consumed 128 MB of memory versus the hash table's 24 MB.

This wasn't supposed to happen. Tries are the textbook solution for autocomplete—O(k) lookup where k is the string length, independent of dataset size. Perfect for prefix matching. The standard choice for autocomplete, spell checkers, and IP routing tables.

"Use a trie," my teammate had suggested for our command-line tool's autocomplete feature. We had 50,000 commands and options to search through. The textbook agreed with the choice.

So we implemented a trie. The benchmark results were devastating:

```bash
$ perf stat -e cache-misses,cycles ./autocomplete_trie "git com"
  Performance counter stats:
    125,000 cache-misses
  4,800,000 cycles

$ perf stat -e cache-misses,cycles ./autocomplete_hash "git com"
  Performance counter stats:
     18,000 cache-misses
    600,000 cycles
```

The trie was **8× slower** than a simple hash table. And it used **128 MB of memory** versus the hash table's 24 MB.

What went wrong?

## The Textbook Story

A trie (pronounced "try") is a tree where each edge represents a character. Here's a trie for the words "cat", "car", and "dog":

```
       root
      /    \
     c      d
     |      |
     a      o
    / \     |
   t   r    g
```

To look up "car", you follow edges: root → 'c' → 'a' → 'r'.

**The textbook pitch**:
- **Prefix sharing**: "cat" and "car" share the "ca" prefix
- **O(k) lookup**: Only depends on string length, not dataset size
- **No string comparisons**: Just follow pointers
- **Perfect for autocomplete**: Find all words with prefix "ca" by traversing the subtree

Sounds perfect, right?

## The Reality Check

Here's the trie node structure I implemented:

```c
typedef struct trie_node {
    struct trie_node *children[256];  // 2,048 bytes (256 × 8-byte pointers)
    void *value;                      // 8 bytes
    bool is_end;                      // 1 byte
    // Padding: 7 bytes
    // Total: 2,064 bytes per node
} trie_node_t;
```

**2,064 bytes per node!** That's 32 cache lines (64 bytes each).

For our 50,000 commands with an average length of 8 characters:
- Nodes needed: ~400,000 (one per character, with sharing)
- Memory: 400,000 × 2,064 = **825 MB**
- Hash table: 50,000 × 24 = **1.2 MB**

The trie used **687× more memory** than a hash table.

### The Cache Problem

Let's trace a lookup for "hello":

```
Step 1: root → children['h']     (cache miss - load root node)
Step 2: node → children['e']     (cache miss - load 'h' node)
Step 3: node → children['l']     (cache miss - load 'e' node)
Step 4: node → children['l']     (cache miss - load first 'l' node)
Step 5: node → children['o']     (cache miss - load second 'l' node)

Total: 5 cache misses for a 5-character word
```

Each node is 2 KB, so they can't all fit in cache. Every character lookup is a cache miss.

Compare this to a hash table: hash the string (cheap), one cache miss to fetch the bucket, done. Total: 1-2 cache misses.

---

## Solution 1: Radix Trees (Patricia Tries)

The first optimization is to **compress chains of single-child nodes**.

In a standard trie for "cat" and "car", you have:

```
    root
     |
     c
     |
     a
    / \
   t   r
```

The nodes for 'c' and 'a' each have only one child. We can compress them into a single node with the prefix "ca":

```
    root
     |
    "ca"
    / \
  "t" "r"
```

This is called a **radix tree** or **Patricia trie**.

Here's the implementation I used:

```c
typedef struct radix_node {
    char *prefix;                // Variable-length prefix
    int prefix_len;
    struct radix_node *children[256];
    void *value;
} radix_node_t;
```

Lookup now matches the prefix first, then descends:

```c
void* radix_search(radix_node_t *node, const char *key) {
    while (node) {
        // Match prefix
        int i = 0;
        while (i < node->prefix_len && key[i] == node->prefix[i]) {
            i++;
        }

        // Prefix mismatch?
        if (i < node->prefix_len) {
            return NULL;
        }

        // Exact match?
        if (key[i] == '\0') {
            return node->value;
        }

        // Descend to child
        node = node->children[(unsigned char)key[i]];
        key += i + 1;
    }
    return NULL;
}
```

For our autocomplete tool, this reduced memory usage by 60% (from 825 MB to 330 MB). But it was still way too much.

---

## Solution 2: Adaptive Radix Tree (ART)

The radix tree helped, but we still had a problem: each node had a 256-pointer array (2,048 bytes), even if it only had 2 children.

I looked at the data. Most nodes had fewer than 10 children. We were wasting 98% of the space in those arrays.

The solution: **adaptive node types**. Use different node structures depending on how many children you have.

### Node Types

**Node4** (1-4 children):
```c
typedef struct {
    uint8_t num_children;
    uint8_t keys[4];              // 4 bytes
    void *children[4];            // 32 bytes
    // Total: 40 bytes
} node4_t;
```

**Node16** (5-16 children):
```c
typedef struct {
    uint8_t num_children;
    uint8_t keys[16];             // 16 bytes
    void *children[16];           // 128 bytes
    // Total: 152 bytes
} node16_t;
```

**Node48** (17-48 children):
```c
typedef struct {
    uint8_t num_children;
    uint8_t index[256];           // Map char → child index
    void *children[48];           // 384 bytes
    // Total: 640 bytes
} node48_t;
```

**Node256** (49-256 children):
```c
typedef struct {
    void *children[256];          // 2,048 bytes
} node256_t;
```

### Adaptive Growth

**Strategy**: Start with Node4, grow as children added.

```
Insert 1st child:  Node4
Insert 5th child:  Node4 → Node16
Insert 17th child: Node16 → Node48
Insert 49th child: Node48 → Node256
```

**Memory savings**:
- Average node: 40-152 bytes (vs 2,048 bytes)
- 10-50× memory reduction

---

## The Benchmark Results

I reimplemented our autocomplete tool with an Adaptive Radix Tree. Here's how it compared:

```
Dataset: 50,000 commands (avg length 8 chars)
Test: 1,000,000 random lookups

Standard trie:
  Memory: 825 MB
  Cycles/lookup: 4,800
  Cache misses: 12.5

Radix tree:
  Memory: 330 MB
  Cycles/lookup: 2,400
  Cache misses: 6.8
  Speedup: 2.0×

Adaptive Radix Tree (ART):
  Memory: 18 MB
  Cycles/lookup: 1,200
  Cache misses: 3.2
  Speedup: 4.0×

Hash table (baseline):
  Memory: 1.2 MB
  Cycles/lookup: 600
  Cache misses: 1.8
```

The ART was 4× faster than the standard trie and used 45× less memory. But the hash table was still 2× faster.

**Why ART is better than standard tries**:
1. **Smaller nodes**: Node4/Node16 fit in 1-2 cache lines instead of 32
2. **Fewer cache misses**: 3.2 vs 12.5 per lookup
3. **Less memory**: 18 MB vs 825 MB

**Why hash tables still win for exact lookups**:
- **Single cache miss**: Hash directly to the bucket
- **No pointer chasing**: One lookup, done

---

## When Tries Make Sense

After all this, you might wonder: "Should I ever use a trie?"

Yes—but only when you need **prefix operations** that hash tables can't provide.

### 1. Autocomplete

Our autocomplete tool needed to find all commands starting with "git co". A hash table can't do this efficiently—you'd have to scan all 50,000 entries.

With an ART, you traverse to the "git co" prefix, then enumerate all children. This is O(k + m) where k is the prefix length and m is the number of matches.

We ended up using an ART for autocomplete despite the 2× slowdown compared to hash tables, because we needed prefix matching.

### 2. IP Routing Tables

IP routers need **longest prefix matching**. For IP address 192.168.1.100, find the longest matching route:
- 192.168.0.0/16 → Gateway A
- 192.168.1.0/24 → Gateway B (longer match, use this)

Tries are perfect for this. Each bit of the IP address is a branch in the tree.

### 3. Spell Checkers

Finding words within edit distance 1-2 of a misspelled word requires exploring similar prefixes. Tries make this efficient.

### 4. When NOT to Use Tries

Don't use tries for:
- **Exact lookups only**: Use a hash table (2× faster, 10× less memory)
- **Small datasets** (< 1,000 entries): Hash table overhead is negligible
- **Random strings**: If there's no prefix sharing, tries waste memory

---

## Real-World Example: Linux Kernel Radix Trees

The Linux kernel uses radix trees for:
- **Page cache**: Mapping file offsets to memory pages
- **IDR (ID allocator)**: Allocating unique IDs
- **XArray**: Generic indexed storage

Here's the kernel's radix tree node (from `lib/radix-tree.c`):

```c
struct radix_tree_node {
    unsigned char shift;      // Height in tree
    unsigned char offset;     // Slot offset in parent
    unsigned int count;       // Number of children
    struct radix_tree_node *parent;
    void *slots[RADIX_TREE_MAP_SIZE];  // 64 slots
};
```

The kernel uses a fixed branching factor of 64 (6 bits per level). For a 32-bit index:
- Height: 32 ÷ 6 ≈ 6 levels
- Cache misses: ~6 per lookup

This is much better than a binary tree's 32 levels.

**Why the kernel uses radix trees**:
1. **Sparse arrays**: File offsets are sparse (not every page is cached)
2. **Range operations**: Iterate over pages in a file range
3. **Predictable performance**: O(log₆₄ n) worst case

---

## Summary

The autocomplete disaster was salvaged. Replacing the standard trie with an Adaptive Radix Tree dropped memory usage from 825 MB to 18 MB, and made lookups 4× faster. The ART provided the prefix matching we needed, though hash tables remained 2× faster for exact lookups.

**Key insights**:

1. **Standard tries are memory hogs**. With 256-pointer arrays per node, they use 50-100× more memory than hash tables.

2. **Radix trees compress chains**. By merging single-child nodes, you can reduce memory by 60-70%.

3. **Adaptive node types are crucial**. Most nodes have few children. Using Node4/Node16 instead of 256-pointer arrays reduces memory by another 10×.

4. **Tries are for prefix operations**. If you only need exact lookups, use a hash table. Tries shine when you need autocomplete, longest prefix matching, or edit distance queries.

5. **Cache misses dominate**. Even with ART, you're traversing k levels for a string of length k. Each level is a potential cache miss. Hash tables win with 1-2 cache misses total.

**The numbers from our autocomplete tool**:
- Standard trie: 4,800 cycles/lookup, 825 MB memory
- Adaptive Radix Tree: 1,200 cycles/lookup, 18 MB memory
- Hash table: 600 cycles/lookup, 1.2 MB memory

We chose ART because we needed prefix matching, but if we only needed exact lookups, hash tables would be the clear winner.

**Next chapter**: Heaps and priority queues—how to maintain sorted order with O(log n) operations.


