# WebAssembly 解释器实现 —— 学习资料

> **目标读者**：具有 .NET CLR 虚拟机实现经验的资深底层程序员
> **最终目标**：独立实现一个类似 [wasm3](https://github.com/wasm3/wasm3) 的 WebAssembly 解释器

---

## 前置知识假设

本系列资料假设你已经具备以下背景：

- ✅ 了解 WebAssembly 是什么、用于什么场景（不再赘述基础概念）
- ✅ 有虚拟机 / 解释器实现经验（特别是 .NET CLR 级别的）
- ✅ 熟悉栈式虚拟机的基本原理（操作数栈、调用栈、指令分发）
- ✅ 熟悉二进制文件格式解析（PE/COFF、CLR Metadata 等）
- ✅ 熟悉 C/C++ 语言（wasm3 使用 C 实现，示例代码以 C 为主）

因此，文档将 **跳过通用 VM 理论**，聚焦于 **Wasm 特有的设计决策和实现技巧**，并在关键处提供 **CLR 对比** 帮助你快速建立认知映射。

---

## 推荐学习路线

```
                    ┌─────────────────┐
                    │   附录 B: LEB128  │  ← 先掌握基础编码
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  01: 二进制格式   │  ← 理解 .wasm 文件结构
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───────┐ ┌───▼────────┐ ┌───▼──────────┐
     │ 02: 指令集/类型 │ │附录A: 操作码│ │ 03: 验证机制  │
     └────────┬───────┘ └────────────┘ └───┬──────────┘
              │                             │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │ 04: 执行引擎设计  │  ← 核心：如何跑起来
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ 05: 实例化与链接  │  ← 模块加载的完整流程
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ 06: wasm3 源码分析│  ← 学习业界最佳实践
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ 07: 实践项目指南  │  ← 动手实现你自己的解释器
                    └─────────────────┘
```

---

## 文档索引

### 核心文档

| # | 文档 | 内容概要 |
|---|------|---------|
| 01 | [二进制格式深度解析](01-binary-format.md) | .wasm 文件结构、13 种 Section 的二进制布局、索引空间、与 CLR PE 格式对比 |
| 02 | [指令集与类型系统](02-instruction-set.md) | 值类型体系、全部指令分类详解、结构化控制流模型、与 CIL 指令集对比 |
| 03 | [验证机制](03-validation.md) | 三层验证流程、操作数栈类型检查算法、栈多态性、控制帧栈、与 CLR 验证对比 |
| 04 | [执行引擎设计与实现](04-execution-engine.md) | 三种执行策略对比、运行时数据结构、函数调用约定、内存操作、wasm3 M3 架构专题 |
| 05 | [模块实例化与链接](05-instantiation.md) | 实例化流程、导入/导出解析、宿主函数绑定、WASI 基础、与 CLR Assembly 加载对比 |
| 06 | [wasm3 源码导读](06-wasm3-analysis.md) | wasm3 架构概览、M3 编译流程、核心执行循环、C 语言技巧、渐进式实现路线图 |
| 07 | [实践项目指南](07-practice-guide.md) | 6 阶段实现计划、测试策略、常见难点解决方案、C 项目脚手架 |

### 附录

| # | 文档 | 内容概要 |
|---|------|---------|
| A | [操作码速查表](appendix/A-opcode-table.md) | 全部 Wasm 操作码的十六进制编码、助记符、立即数格式、栈签名速查 |
| B | [LEB128 编码详解](appendix/B-leb128.md) | LEB128 无符号/有符号变长整数编码原理与 C 实现 |
| C | [CLR vs Wasm 全面对比](appendix/C-clr-wasm-comparison.md) | 二进制格式、类型系统、指令集、验证、执行模型、内存模型的系统性对比 |

---

## 关键参考资源

### 规范文档
- [WebAssembly Core Specification 2.0](https://webassembly.github.io/spec/core/) — 官方规范，最权威的参考
- [WebAssembly Binary Format](https://webassembly.github.io/spec/core/binary/index.html) — 规范中的二进制格式章节
- [WebAssembly Validation](https://webassembly.github.io/spec/core/valid/index.html) — 规范中的验证章节
- [WebAssembly Execution](https://webassembly.github.io/spec/core/exec/index.html) — 规范中的执行语义章节

### wasm3 相关
- [wasm3 GitHub 仓库](https://github.com/wasm3/wasm3) — 源码（已归档，仍可阅读和 clone）
- [Mastering WebAssembly VMs](https://github.com/aspect-build/aspect-workflows/files/9517579/aspect-aspect-workflows-m3-slides.pdf) — wasm3 作者 Volodymyr Shymanskyy 的 M3 架构设计演示

### 工具链
- [wabt (WebAssembly Binary Toolkit)](https://github.com/WebAssembly/wabt) — wat2wasm / wasm2wat / wasm-objdump 等工具
- [Binaryen](https://github.com/WebAssembly/binaryen) — WebAssembly 编译器基础设施与优化器
- [WebAssembly Spec Test Suite](https://github.com/WebAssembly/spec/tree/main/test/core) — 官方合规测试套件

### 其他解释器参考实现
- [wasm-micro-runtime (WAMR)](https://github.com/bytecodealliance/wasm-micro-runtime) — 字节码联盟维护的轻量级 Wasm 运行时
- [wasmi](https://github.com/wasmi-labs/wasmi) — Rust 实现的 Wasm 解释器（Parity 出品）
- [wain](https://github.com/rhysd/wain) — Rust 实现的极简 Wasm 解释器（适合学习，代码量小）

---

## 如何使用这些资料

1. **按顺序阅读**：建议按照上面的学习路线图从上到下阅读，每篇文档都建立在前一篇的基础上
2. **对照规范**：阅读时建议同时打开 [WebAssembly 官方规范](https://webassembly.github.io/spec/core/)，文档中会标注对应的规范章节
3. **动手验证**：使用 `wat2wasm` 和 `wasm-objdump` 工具实际查看 .wasm 文件的二进制内容，加深理解
4. **边学边做**：到第 07 篇实践指南时，你应该已经具备了实现解释器的全部知识，可以开始动手编码
5. **CLR 对比**：每篇文档中的 "CLR 对比" 部分是专门为你设计的，利用已有经验加速理解

---

## 规范版本说明

- 核心内容基于 **WebAssembly Core Specification 2.0**
- 涵盖 MVP + 以下已合并提案：多返回值（multi-value）、引用类型（reference types）、批量内存操作（bulk memory）
- **暂不覆盖**：SIMD (v128)、Threads、Exception Handling、GC 等提案（可作为后续扩展）
- wasm3 源码分析基于其最新稳定版本（注：wasm3 已进入维护模式）
