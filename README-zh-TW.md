# Data Structures in Practice (繁體中文版)

**A Hardware-Aware Approach for System Software Engineers**

**作者**: Danny Jiang  
**版本**: Draft v0p4  
**授權**: CC BY 4.0 International  
**語言**: [English](README.md) | 繁體中文

---

## 📖 關於本書

*Data Structures in Practice* 是一本從硬體角度深入探討資料結構的綜合指南，專為需要理解資料結構不僅是「做什麼」，更要知道「在真實硬體上如何表現」的系統軟體工程師設計。

### 本書的獨特之處

- **硬體優先的方法**：每個資料結構都從 cache 行為、memory hierarchy 和 CPU microarchitecture 的角度分析
- **真實世界的效能**：來自現代處理器（RISC-V、ARM、x86）的實際 benchmark 數據
- **系統軟體焦點**：來自 bootloader、device driver、firmware 和 embedded systems 的範例
- **雙語版本**：完整的英文和繁體中文版本

### 你將學到

- Cache miss 如何影響 linked list 效能
- 為什麼 array-of-structs vs struct-of-arrays 對 SIMD 很重要
- 在 embedded systems 中何時使用 B-tree vs hash table
- 如何正確地 benchmark 和 profile 資料結構
- Concurrent systems 的 lock-free 資料結構
- Memory allocator 設計和 fragmentation 分析

---

## 📚 書籍結構

**Part I: Foundations**（Chapters 1-3）
- Chapter 1: The Performance Gap
- Chapter 2: Memory Hierarchy
- Chapter 3: Benchmarking and Profiling

**Part II: Basic Data Structures**（Chapters 4-8）
- Chapter 4: Arrays and Cache Behavior
- Chapter 5: Linked Lists
- Chapter 6: Stacks and Queues
- Chapter 7: Hash Tables
- Chapter 8: Dynamic Arrays

**Part III: Trees and Hierarchies**（Chapters 9-12）
- Chapter 9: Binary Trees
- Chapter 10: B-Trees
- Chapter 11: Tries
- Chapter 12: Heaps

**Part IV: Advanced Topics**（Chapters 13-16）
- Chapter 13: Lock-Free Data Structures
- Chapter 14: String Processing
- Chapter 15: Graphs
- Chapter 16: Probabilistic Data Structures

**Part V: Case Studies**（Chapters 17-20）
- Chapter 17: Bootloader Case Study
- Chapter 18: Device Driver Case Study
- Chapter 19: Firmware Case Study
- Chapter 20: Benchmark Case Studies

**附錄**（A-F）
- Appendix A: Benchmark Framework Reference
- Appendix B: Hardware Reference
- Appendix C: Tool Reference（QEMU RISC-V Guide）
- Appendix D: Further Reading
- Appendix E: Exercises（40 個練習題涵蓋所有章節）
- Appendix F: Exercise Solutions

**總計**：20 章 + 6 個附錄，約 105,000 字（約 420 頁）

---

## 📥 原始檔案

所有 Markdown 原始檔案都在此 repository 中：

- **英文版**：`manuscript/`
- **繁體中文版**：`manuscript-zh-TW/`

**目前版本**：Draft v0p4 - 2025 年 12 月

---

## 🎯 目標讀者

本書適合：

- **系統軟體工程師**：從事 bootloader、firmware、device driver 開發
- **Embedded Systems 開發者**：需要在受限資源下優化效能
- **效能工程師**：想要理解硬體層級的效能
- **電腦科學學生**：在真實世界情境中學習資料結構
- **RISC-V 開發者**：範例包含 RISC-V assembly 和架構

**先備知識**：
- 基本 C 程式設計
- 理解 pointer 和 memory
- 熟悉 computer architecture（有幫助但非必需）

---

## 📄 授權

**版權所有 © 2025 Danny Jiang**

本著作採用 **Creative Commons Attribution 4.0 International License (CC BY 4.0)** 授權。

**您可以自由地：**

- **分享** — 以任何媒介或格式複製及散布本素材
- **修改** — 重混、轉換本素材，及依本素材建立新素材，且為任何目的，包含商業性質之使用

**惟需遵守下列條件：**

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。

**授權條款**：https://creativecommons.org/licenses/by/4.0/

---

## 🔧 如何使用本書

### 線上閱讀

直接在 GitHub 上瀏覽 Markdown 檔案：
- 從 `manuscript-zh-TW/front_matter/02_preface.md` 開始
- 然後依序閱讀章節：`manuscript-zh-TW/chapters/chapter01.md` 等

### 離線閱讀

Clone 此 repository：
```bash
git clone https://github.com/djiangtw/data-structures-in-practice-public.git
cd data-structures-in-practice-public
```

使用任何 Markdown 閱讀器或文字編輯器閱讀檔案。

### 建立 PDF/EPUB（進階）

本次發布不包含 PDF/EPUB 建立腳本。您可以使用 Pandoc 等工具將 Markdown 轉換為其他格式：

```bash
# 範例：轉換為 PDF（需要 pandoc 和 xelatex）
pandoc manuscript-zh-TW/chapters/*.md -o book.pdf --pdf-engine=xelatex
```

---

## 🤝 貢獻

這是一個唯讀的公開 repository。本書在私有 repository 中開發。

**歡迎回饋**：
- 針對錯字、錯誤或建議開 issue
- 鼓勵討論和提問

**注意**：無法接受 pull request，因為這是從私有開發 repository 單向同步的。

---

## 📧 聯絡方式

**作者**：Danny Jiang

如有問題或回饋，請在此 repository 開 issue。

---

## 🙏 致謝

靈感來自經典電腦科學教材和現代系統程式設計實踐。特別感謝 RISC-V 社群和開源貢獻者。

---

## 📅 版本歷史

- **v0p4**（2025 年 12 月）：完整書籍發布 - 全部 20 章 + 6 個附錄
- **v0p3**（2025 年 12 月）：首次公開發布 - Part I & Part II（Chapters 1-8）

---

**祝閱讀愉快！** 📖

