# Data Structures in Practice

**A Hardware-Aware Approach for System Software Engineers**

**Author**: Danny Jiang  
**Version**: Draft v0p4  
**License**: CC BY 4.0 International  
**Language**: English | [繁體中文](README-zh-TW.md)

---

## 📖 About This Book

*Data Structures in Practice* is a comprehensive guide to data structures from a hardware-aware perspective, designed for system software engineers who need to understand not just *what* data structures do, but *how* they perform on real hardware.

### What Makes This Book Different

- **Hardware-First Approach**: Every data structure is analyzed through the lens of cache behavior, memory hierarchy, and CPU microarchitecture
- **Real-World Performance**: Actual benchmark data from modern processors (RISC-V, ARM, x86)
- **System Software Focus**: Examples from bootloaders, device drivers, firmware, and embedded systems
- **Bilingual**: Complete English and Traditional Chinese versions

### What You'll Learn

- How cache misses affect linked list performance
- Why array-of-structs vs struct-of-arrays matters for SIMD
- When to use B-trees vs hash tables in embedded systems
- How to benchmark and profile data structures correctly
- Lock-free data structures for concurrent systems
- Memory allocator design and fragmentation analysis

---

## 📚 Book Structure

**Part I: Foundations** (Chapters 1-3)
- Chapter 1: The Performance Gap
- Chapter 2: Memory Hierarchy
- Chapter 3: Benchmarking and Profiling

**Part II: Basic Data Structures** (Chapters 4-8)
- Chapter 4: Arrays and Cache Behavior
- Chapter 5: Linked Lists
- Chapter 6: Stacks and Queues
- Chapter 7: Hash Tables
- Chapter 8: Dynamic Arrays

**Part III: Trees and Hierarchies** (Chapters 9-12)
- Chapter 9: Binary Trees
- Chapter 10: B-Trees
- Chapter 11: Tries
- Chapter 12: Heaps

**Part IV: Advanced Topics** (Chapters 13-16)
- Chapter 13: Lock-Free Data Structures
- Chapter 14: String Processing
- Chapter 15: Graphs
- Chapter 16: Probabilistic Data Structures

**Part V: Case Studies** (Chapters 17-20)
- Chapter 17: Bootloader Case Study
- Chapter 18: Device Driver Case Study
- Chapter 19: Firmware Case Study
- Chapter 20: Benchmark Case Studies

**Appendices** (A-F)
- Appendix A: Benchmark Framework Reference
- Appendix B: Hardware Reference
- Appendix C: Tool Reference (QEMU RISC-V Guide)
- Appendix D: Further Reading
- Appendix E: Exercises (40 exercises across all chapters)
- Appendix F: Exercise Solutions

**Total**: 20 chapters + 6 appendices, ~105,000 words (~420 pages)

---

## 📥 Source Files

All Markdown source files are available in this repository:

- **English Version**: `manuscript/`
- **Traditional Chinese Version**: `manuscript-zh-TW/`

**Current Version**: Draft v0p4 - December 2025

---

## 🎯 Target Audience

This book is for:

- **System Software Engineers**: Working on bootloaders, firmware, device drivers
- **Embedded Systems Developers**: Need to optimize for constrained resources
- **Performance Engineers**: Want to understand hardware-level performance
- **Computer Science Students**: Learning data structures with real-world context
- **RISC-V Developers**: Examples include RISC-V assembly and architecture

**Prerequisites**:
- Basic C programming
- Understanding of pointers and memory
- Familiarity with computer architecture (helpful but not required)

---

## 📄 License

**Copyright © 2025 Danny Jiang**

This work is licensed under the **Creative Commons Attribution 4.0 International License (CC BY 4.0)**.

**You are free to:**

- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material for any purpose, even commercially

**Under the following terms:**

- **Attribution** — You must give appropriate credit, provide a link to the license, and indicate if changes were made.

**License**: https://creativecommons.org/licenses/by/4.0/

---

## 🔧 How to Use This Book

### Reading Online

Browse the Markdown files directly on GitHub:
- Start with `manuscript/front_matter/02_preface.md`
- Then read chapters in order: `manuscript/chapters/chapter01.md`, etc.

### Reading Offline

Clone this repository:
```bash
git clone https://github.com/djiangtw/data-structures-in-practice-public.git
cd data-structures-in-practice-public
```

Use any Markdown reader or text editor to read the files.

### Building PDF/EPUB (Advanced)

PDF/EPUB build scripts are not included in this release. You can use tools like Pandoc to convert Markdown to other formats:

```bash
# Example: Convert to PDF (requires pandoc and xelatex)
pandoc manuscript/chapters/*.md -o book.pdf --pdf-engine=xelatex
```

---

## 🤝 Contributing

This is a read-only public repository. The book is developed in a private repository.

**Feedback Welcome**:
- Open an issue for typos, errors, or suggestions
- Discussions and questions are encouraged

**Note**: Pull requests cannot be accepted as this is a one-way sync from the private development repository.

---

## 📧 Contact

**Author**: Danny Jiang

For questions or feedback, please open an issue in this repository.

---

## 🙏 Acknowledgments

Inspired by classic computer science texts and modern system programming practices. Special thanks to the RISC-V community and open-source contributors.

---

## 📅 Version History

- **v0p4** (December 2025): Complete book release - All 20 chapters + 6 appendices
- **v0p3** (December 2025): First public release - Part I & Part II (Chapters 1-8)

---

**Happy Reading!** 📖

