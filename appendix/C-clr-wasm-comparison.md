# 附录 C：CLR vs WebAssembly 全面对比

> **目标读者**：有 .NET CLR 虚拟机实现经验的开发者，希望通过系统性对比快速建立 Wasm 的认知映射。
> 本附录将从二进制格式、类型系统、指令集、验证机制、执行模型、内存模型、模块化机制七个维度进行深度对比。

---

## 1. 总体架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLR (.NET)                               │
│                                                                 │
│  .dll/.exe (PE Format)                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ PE Header│→ │ CLR Header│→ │ Metadata │→ │ IL Method Body │  │
│  └──────────┘  └──────────┘  │ Tables   │  │ (CIL Bytecode) │  │
│                               └──────────┘  └────────────────┘  │
│                                                                 │
│  Runtime: CLR VM (JIT/AOT + GC + Type System + Exception)       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     WebAssembly                                  │
│                                                                 │
│  .wasm (Custom Binary Format)                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  Header   │→ │  Type    │→ │ Function │→ │  Code Section  │  │
│  │ (magic+  │  │ Section  │  │ /Import/ │  │ (Wasm Bytecode)│  │
│  │  version) │  └──────────┘  │ Export   │  └────────────────┘  │
│  └──────────┘                 └──────────┘                      │
│                                                                 │
│  Runtime: Wasm VM (Interp/JIT + Linear Memory, NO GC)           │
└─────────────────────────────────────────────────────────────────┘
```

**核心哲学差异**：

| 维度 | CLR | WebAssembly |
|------|-----|-------------|
| 设计目标 | 通用应用平台（桌面/服务器/移动） | 安全沙箱化的可移植计算 |
| 复杂度 | 极高（GC、异常、泛型、反射、安全模型） | 极简（最小化规范，宿主提供能力） |
| 类型丰富度 | 丰富（类、接口、泛型、值类型、委托…） | 极简（4 种数值 + 2 种引用） |
| 内存管理 | 托管堆 + GC | 线性内存（手动管理） |
| 安全模型 | CAS + Type Safety + Verification | 沙箱隔离 + 类型验证 |
| 互操作 | P/Invoke, COM Interop | Import/Export + Host Functions |

---

## 2. 二进制格式对比

### 2.1 文件结构

| 特性 | CLR (PE/COFF) | Wasm (.wasm) |
|------|---------------|-------------|
| 文件格式 | PE/COFF（Windows 可执行格式扩展） | 自定义二进制格式 |
| Magic Number | `MZ` (DOS) + `PE\0\0` | `\0asm` |
| 版本标识 | PE Optional Header 中的多个版本字段 | 单个 uint32: `1` |
| 整体结构 | 多层嵌套（DOS → PE → Section → CLR → Metadata） | 扁平的 Section 序列 |
| 元数据组织 | 行列式表结构（#~ stream），支持随机访问 | 顺序流式编码，必须线性扫描 |
| 整数编码 | 固定宽度 + Compressed Integer（1/2/4 字节） | LEB128 变长编码 |
| 字符串编码 | #Strings heap（null-terminated UTF-8） | 长度前缀 UTF-8 |
| 典型文件大小 | 数 KB ~ 数 MB（含大量元数据） | 数百字节 ~ 数 MB（紧凑） |

### 2.2 元数据 vs Section

**CLR 元数据表**（约 45 种表，通过 table + row index 随机访问）：
```
TypeDef Table:    [Flags, Name, Namespace, Extends, FieldList, MethodList]
MethodDef Table:  [RVA, ImplFlags, Flags, Name, Signature, ParamList]
MemberRef Table:  [Class, Name, Signature]
StandAloneSig:    [Signature]
...
```

**Wasm Section**（13 种，顺序编码）：
```
Type Section:     [func_type, func_type, ...]
Function Section: [type_idx, type_idx, ...]
Code Section:     [func_body, func_body, ...]
Import Section:   [import_entry, import_entry, ...]
Export Section:   [export_entry, export_entry, ...]
...
```

**关键差异**：

1. **随机访问 vs 顺序访问**：CLR 的元数据表设计允许通过 token（table << 24 | row）直接定位任何元素；Wasm 必须从头解析，但解析后可建立索引数组
2. **信息密度**：CLR 的元数据包含大量运行时不需要的信息（属性、安全声明、调试信息等）；Wasm 只包含执行必需的信息
3. **代码与元数据的关系**：CLR 的 MethodDef 通过 RVA 指向 IL body（可能在文件任意位置）；Wasm 的 Function Section 和 Code Section 通过索引一一对应

### 2.3 解析复杂度对比

```
CLR PE 解析路径（约 6 层间接）：
  DOS Header → PE Header → Optional Header → Data Directory[14]
  → CLR Header → Metadata Root → #~ Stream → Tables

Wasm 解析路径（2 层）：
  Module Header → Section* (linear scan)
```

**对你的启示**：如果你实现过 CLR PE 解析器，Wasm 的解析器会简单得多。没有多层间接寻址，没有 RVA 到文件偏移的转换，没有 heap 索引解引用。

---

## 3. 类型系统对比

### 3.1 类型丰富度

| CLR 类型 | Wasm 对应 | 说明 |
|----------|----------|------|
| `System.Int32` | `i32` | 直接对应 |
| `System.Int64` | `i64` | 直接对应 |
| `System.Single` | `f32` | 直接对应 |
| `System.Double` | `f64` | 直接对应 |
| `System.Boolean` | `i32` | Wasm 无布尔类型，用 i32 表示 |
| `System.Byte`, `Int16` 等 | `i32` | Wasm 无小整数类型，通过 load/store 变体处理 |
| `System.String` | ❌ | Wasm 无字符串类型，需在线性内存中手动管理 |
| 类/结构体 | ❌ | Wasm 无复合类型，需在线性内存中手动布局 |
| 数组 | ❌ | 同上 |
| 委托/函数指针 | `funcref` | 通过 table + call_indirect 实现 |
| `System.Object` | `externref` | 不透明的宿主引用 |
| 泛型 `T` | ❌ | Wasm 无泛型，需编译时单态化 |
| 接口 | ❌ | 无，可通过 table 模拟虚方法表 |
| 枚举 | `i32` | 编译为整数 |
| 可空类型 `T?` | ❌ | 无，需手动处理 |

### 3.2 函数签名

**CLR MethodSig**（Blob heap 中的压缩编码）：
```
CallingConvention | ParamCount | RetType | Param1 | Param2 | ...

- CallingConvention: DEFAULT/VARARG/GENERIC/...
- 支持 byref, pinned, typedbyref, sentinel 等修饰符
- 支持泛型参数 (GENERICINST)
- 返回值只能有 0 或 1 个
```

**Wasm FuncType**：
```
0x60 | param_count | param_types... | result_count | result_types...

- 无调用约定变体（只有一种）
- 无修饰符
- 无泛型
- 支持多返回值（Wasm 2.0）
```

**关键差异**：
- CLR 签名极其复杂（需要处理泛型实例化、byref、pinned 等），Wasm 签名极其简单
- CLR 只支持单返回值（通过 out 参数模拟多返回），Wasm 原生支持多返回值
- CLR 有 `this` 指针的隐式参数，Wasm 所有参数显式

### 3.3 类型验证强度

| 验证特性 | CLR | Wasm |
|---------|-----|------|
| 操作数栈类型检查 | ✅ | ✅ |
| 内存访问类型安全 | ✅（托管引用保证） | ❌（线性内存无类型，只有边界检查） |
| 空引用检查 | ✅（NullReferenceException） | 部分（funcref null → trap） |
| 数组边界检查 | ✅（IndexOutOfRangeException） | ❌（无数组概念） |
| 类型转换安全 | ✅（InvalidCastException） | 部分（call_indirect 类型检查） |
| 控制流完整性 | 部分（允许任意跳转） | ✅（结构化控制流，不可能跳到非法位置） |

---

## 4. 指令集对比

### 4.1 整体设计哲学

| 特性 | CIL (CLR) | Wasm |
|------|-----------|------|
| 指令数量 | ~220+ | ~170+（MVP），~400+（含扩展） |
| 编码方式 | 1-2 字节操作码 + 固定宽度操作数 | 1-2 字节操作码 + LEB128 操作数 |
| 栈模型 | 纯栈机 | 纯栈机（可优化为寄存器式） |
| 控制流 | 非结构化（任意偏移跳转） | 结构化（block/loop/if + 标签索引） |
| 异常处理 | 内置（try/catch/finally/fault） | 无（提案阶段） |
| 面向对象 | 内置（newobj, callvirt, ldfld...） | 无 |
| 内存模型 | 托管堆 + 值类型栈 | 单一线性内存 |

### 4.2 控制流指令对比（最重要的差异）

**CIL 的非结构化控制流**：
```cil
// CIL: 可以跳转到任意偏移
IL_0000: ldarg.0
IL_0001: ldc.i4.0
IL_0002: ble.s IL_000A      // 条件跳转到任意标签
IL_0004: ldarg.0
IL_0005: ldarg.0
IL_0006: ldc.i4.1
IL_0007: sub
IL_0008: call int32 Fib(int32)
IL_000D: br.s IL_000B        // 无条件跳转到任意标签
IL_000A: ldc.i4.1
IL_000B: ret
```

**Wasm 的结构化控制流**：
```wasm
;; Wasm: 只能跳转到包围的 block/loop 的边界
(func $fib (param i32) (result i32)
  (if (result i32) (i32.le_s (local.get 0) (i32.const 1))
    (then (local.get 0))          ;; 只能在 if/else/end 结构内
    (else
      (i32.add
        (call $fib (i32.sub (local.get 0) (i32.const 1)))
        (call $fib (i32.sub (local.get 0) (i32.const 2)))))))
```

**对比分析**：

| 特性 | CIL | Wasm |
|------|-----|------|
| 跳转目标 | 任意字节偏移（绝对/相对） | 标签索引（相对深度，de Bruijn） |
| 前向跳转 | ✅ `br IL_xxxx` | ✅ `br` 到 `block` 的 `end` |
| 后向跳转 | ✅ `br IL_xxxx` | ✅ `br` 到 `loop` 的开头 |
| 任意跳转 | ✅ 可跳到函数内任意位置 | ❌ 只能跳到嵌套块的边界 |
| 不可规约控制流 | ✅ 可以构造 | ❌ 不可能（结构化保证） |
| 验证复杂度 | 需要数据流分析 | 单遍线性扫描即可 |

**为什么 Wasm 选择结构化控制流？**
1. **安全性**：不可能跳转到指令中间或非法位置
2. **验证效率**：单遍前向扫描即可完成类型检查
3. **编译友好**：结构化控制流可直接映射到 SSA/基本块，便于 JIT 优化
4. **代价**：某些控制流模式（如 Duff's device）需要额外的 block 嵌套来表达

### 4.3 常用指令一一对比

| 操作 | CIL | Wasm | 差异说明 |
|------|-----|------|---------|
| 加载常量 | `ldc.i4 N` | `i32.const N` | CIL 有短格式 `ldc.i4.0`~`ldc.i4.8` |
| 加法 | `add` | `i32.add` / `i64.add` | CIL 类型无关，Wasm 类型化 |
| 溢出检查加法 | `add.ovf` | ❌ | Wasm 无溢出检查变体 |
| 局部变量读取 | `ldloc.0` | `local.get 0` | 基本等价 |
| 参数读取 | `ldarg.0` | `local.get 0` | Wasm 中参数和局部变量统一编号 |
| 函数调用 | `call` / `callvirt` | `call` / `call_indirect` | CIL 区分虚调用，Wasm 用间接调用 |
| 内存加载 | `ldind.i4` / `ldfld` | `i32.load` | CIL 通过引用，Wasm 通过地址 |
| 内存存储 | `stind.i4` / `stfld` | `i32.store` | 同上 |
| 对象创建 | `newobj` | ❌ | Wasm 无对象概念 |
| 类型转换 | `conv.i4` / `castclass` | `i32.wrap_i64` 等 | Wasm 转换指令更细粒度 |
| 异常抛出 | `throw` | `unreachable` | Wasm 只有 trap，无异常 |
| 返回 | `ret` | `return` | 基本等价 |
| 空操作 | `nop` | `nop` | 完全等价 |
| 丢弃栈顶 | `pop` | `drop` | 完全等价 |

### 4.4 CIL 独有的指令类别

这些在 Wasm 中完全没有对应：

- **面向对象**：`newobj`, `callvirt`, `ldfld`, `stfld`, `ldsfld`, `stsfld`, `castclass`, `isinst`, `box`, `unbox`
- **异常处理**：`throw`, `rethrow`, `leave`, `endfinally`, `endfilter`
- **数组操作**：`newarr`, `ldelem`, `stelem`, `ldlen`
- **指针操作**：`ldind.*`, `stind.*`, `localloc`, `cpblk`, `initblk`
- **前缀指令**：`volatile.`, `unaligned.`, `tail.`, `constrained.`, `readonly.`

---

## 5. 验证机制对比

### 5.1 验证流程

**CLR IL 验证**：
```
1. 方法签名验证
2. 局部变量签名验证
3. 指令流验证：
   a. 对每个基本块进行数据流分析
   b. 在合并点（分支目标）检查栈状态一致性
   c. 需要处理任意跳转导致的复杂控制流图
   d. 异常处理区域的特殊验证规则
4. 类型安全验证（可验证代码 vs 不可验证代码）
```

**Wasm 验证**：
```
1. 模块级验证（索引范围、Section 一致性）
2. 函数体验证：
   a. 单遍前向扫描
   b. 维护操作数类型栈 + 控制帧栈
   c. 每条指令检查输入类型、推导输出类型
   d. 在 block/loop/if 边界检查栈高度和类型
3. 无"不可验证"代码的概念 — 所有代码必须通过验证
```

### 5.2 关键差异

| 特性 | CLR 验证 | Wasm 验证 |
|------|---------|----------|
| 算法复杂度 | O(n²) 最坏情况（数据流迭代） | O(n) 严格单遍 |
| 控制流图 | 需要构建完整 CFG | 隐式（结构化控制流） |
| 合并点处理 | 需要栈状态合并（可能需要多次迭代） | 块边界自然保证栈状态一致 |
| 不可验证代码 | 允许（unsafe 代码，跳过验证） | 不允许（所有代码必须验证通过） |
| 异常处理验证 | 复杂（try/catch/finally 区域嵌套规则） | 无（Wasm 无异常） |
| 栈多态性 | 有限（仅在 `throw`/`rethrow` 后） | 完整（`unreachable`/`br`/`return` 后） |

### 5.3 实现复杂度对比

如果你实现过 CLR 验证器，Wasm 验证器会简单很多：

```
CLR 验证器需要处理的复杂情况：
├── 任意跳转目标的栈状态合并
├── 异常处理区域的嵌套验证
├── protected block 的进入/退出规则
├── 泛型类型的实例化验证
├── byref 的生命周期跟踪
├── delegate 创建的类型安全
└── 不可验证代码的标记

Wasm 验证器需要处理的复杂情况：
├── 栈多态性（unreachable 后的类型推断）
├── 多返回值的块类型检查
└── 控制帧栈的正确维护
```

---

## 6. 执行模型对比

### 6.1 运行时架构

**CLR 运行时**：
```
┌─────────────────────────────────────────────┐
│                CLR Runtime                   │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ JIT/AOT  │  │    GC    │  │ Type      │  │
│  │ Compiler │  │ (Gen0/1/2│  │ Loader    │  │
│  │          │  │  + LOH)  │  │           │  │
│  └──────────┘  └──────────┘  └───────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ Thread   │  │ Exception│  │ Security  │  │
│  │ Pool     │  │ Handler  │  │ Manager   │  │
│  └──────────┘  └──────────┘  └───────────┘  │
│  ┌──────────────────────────────────────┐    │
│  │     Managed Heap + Value Type Stack  │    │
│  └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

**Wasm 运行时**：
```
┌─────────────────────────────────────────────┐
│              Wasm Runtime                    │
│  ┌──────────┐  ┌──────────────────────────┐  │
│  │Interpreter│  │    Linear Memory         │  │
│  │ or JIT   │  │    (flat byte array)      │  │
│  └──────────┘  └──────────────────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ Value    │  │  Call    │  │  Table    │  │
│  │ Stack    │  │  Stack   │  │ (funcref) │  │
│  └──────────┘  └──────────┘  └───────────┘  │
│  ┌──────────────────────────────────────┐    │
│  │     Global Variables                  │    │
│  └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

### 6.2 栈帧对比

**CLR 栈帧**：
```
┌─────────────────────────┐ ← High Address
│ Arguments (pushed by     │
│ caller, including 'this')│
├─────────────────────────┤
│ Return Address           │
├─────────────────────────┤
│ Saved Registers          │
├─────────────────────────┤
│ Local Variables          │
│ (typed, may contain      │
│  managed references)     │
├─────────────────────────┤
│ Evaluation Stack         │
│ (typed, tracked by JIT)  │
├─────────────────────────┤
│ Security Object (opt)    │
├─────────────────────────┤
│ GC Info (for stack walk) │
└─────────────────────────┘ ← Low Address
```

**Wasm 栈帧**：
```
┌─────────────────────────┐ ← Stack Top
│ Evaluation Stack         │
│ (untyped 64-bit slots)   │
├─────────────────────────┤
│ Local Variables          │
│ (params + locals,        │
│  untyped 64-bit slots)   │
├─────────────────────────┤ ← Frame Base (sp_base)
│ [Previous Frame's Stack] │
└─────────────────────────┘
```

**关键差异**：
- CLR 栈帧包含 GC 信息（用于栈遍历和根标记），Wasm 不需要
- CLR 局部变量有精确类型（GC 需要区分引用和值），Wasm 局部变量是无类型的 64-bit 槽
- CLR 有安全对象和异常处理信息，Wasm 没有

### 6.3 执行策略对比

| 策略 | CLR | Wasm |
|------|-----|------|
| 解释执行 | 早期 CLR 有解释器（现在主要用于 Mono） | wasm3, wasm-micro-runtime |
| JIT 编译 | 主流方式（RyuJIT） | V8 (Liftoff + TurboFan), wasmtime (Cranelift) |
| AOT 编译 | .NET Native, NativeAOT | wasmer (Cranelift/LLVM), wasmtime |
| Tiered 编译 | ✅ (.NET 6+ 默认) | ✅ (V8: Liftoff → TurboFan) |
| Threaded Code | ❌（不常见） | ✅ (wasm3 的核心策略) |

### 6.4 函数调用对比

| 特性 | CLR | Wasm |
|------|-----|------|
| 调用方式 | `call` (静态) / `callvirt` (虚) / `calli` (间接) | `call` (直接) / `call_indirect` (间接) |
| 虚方法分派 | vtable 查找 | 无（通过 table + call_indirect 模拟） |
| 参数传递 | 栈传递（JIT 可能优化为寄存器） | 栈传递 |
| 返回值 | 单返回值 | 多返回值 |
| this 指针 | 隐式第一个参数 | 无（需显式传递） |
| 尾调用 | `tail.` 前缀（可选优化） | `return_call` (提案) |
| 调用深度限制 | 由 OS 栈大小决定（默认 1MB） | 由实现定义（通常 1000-10000） |

---

## 7. 内存模型对比

### 7.1 整体架构

**CLR 内存模型**：
```
┌─────────────────────────────────────────────┐
│                 Process Memory               │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │         Managed Heap (GC)            │    │
│  │  ┌────────┐ ┌────────┐ ┌─────────┐  │    │
│  │  │  Gen 0 │ │  Gen 1 │ │  Gen 2  │  │    │
│  │  └────────┘ └────────┘ └─────────┘  │    │
│  │  ┌──────────────────────────────┐    │    │
│  │  │    Large Object Heap (LOH)   │    │    │
│  │  └──────────────────────────────┘    │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │         Thread Stacks                │    │
│  │  (value types, managed pointers,     │    │
│  │   evaluation stack)                  │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │    Unmanaged Memory (P/Invoke)       │    │
│  └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

**Wasm 内存模型**：
```
┌─────────────────────────────────────────────┐
│              Wasm Instance Memory            │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │       Linear Memory                  │    │
│  │  (single contiguous byte array,      │    │
│  │   max 4GB, page-granularity growth)  │    │
│  │                                      │    │
│  │  ┌──────────────────────────────┐    │    │
│  │  │ 0x0000: [data segment init] │    │    │
│  │  │ 0x....: [heap area]         │    │    │
│  │  │ 0x....: [stack area]        │    │    │
│  │  │ (all managed by the Wasm    │    │    │
│  │  │  program itself, not VM)    │    │    │
│  │  └──────────────────────────────┘    │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌──────────────┐  ┌──────────────────┐      │
│  │ Value Stack  │  │ Global Variables │      │
│  │ (VM managed) │  │ (VM managed)     │      │
│  └──────────────┘  └──────────────────┘      │
└─────────────────────────────────────────────┘
```

### 7.2 详细对比

| 特性 | CLR | Wasm |
|------|-----|------|
| 内存分配 | `new` → GC 管理 | 线性内存中手动管理（malloc/free 由编译器提供） |
| 垃圾回收 | 分代 GC（Gen0/1/2 + LOH） | 无 GC（宿主可通过 externref 提供） |
| 内存安全 | 类型系统保证（无野指针） | 边界检查保证（无越界访问，但可能有逻辑错误） |
| 内存增长 | 自动（GC 管理堆大小） | 显式 `memory.grow`（页粒度，64KB/页） |
| 最大内存 | 受进程地址空间限制 | 4GB（32-bit 地址空间），Memory64 提案扩展到 64-bit |
| 内存隔离 | AppDomain（已废弃）/ 进程隔离 | 每个实例独立的线性内存 |
| 字节序 | 平台相关（通常小端） | 始终小端 |
| 对齐要求 | 平台相关 | 无硬性要求（非对齐访问合法但可能慢） |

### 7.3 对你的启示

实现 Wasm 线性内存比实现 CLR GC 堆简单几个数量级：

```c
// Wasm linear memory: essentially just a byte array with bounds checking
typedef struct {
    uint8_t *data;
    uint32_t size;
    uint32_t max_size;
} LinearMemory;

// Compare with CLR GC heap:
// - Object headers (sync block index + method table pointer)
// - Generation tracking (card tables, write barriers)
// - Pinning support
// - Finalization queue
// - Weak references
// - Large object heap special handling
// - Concurrent/background collection
// ... hundreds of thousands of lines of code
```

---

## 8. 模块化机制对比

### 8.1 模块 vs Assembly

| 特性 | CLR Assembly | Wasm Module |
|------|-------------|-------------|
| 基本单元 | Assembly（.dll/.exe） | Module（.wasm） |
| 版本管理 | 强名称 + 版本号 + Culture | 无（由宿主管理） |
| 命名空间 | 多层命名空间（System.Collections.Generic） | 二级命名（module.name） |
| 可见性 | public/internal/private 等 | 只有 export（公开）和非 export（私有） |
| 依赖声明 | AssemblyRef 表 | Import Section |
| 依赖解析 | GAC / probing path / binding redirect | 宿主负责提供 |
| 延迟加载 | ✅（Assembly.Load） | 取决于宿主实现 |
| 反射 | ✅（完整的运行时类型信息） | ❌（无运行时类型信息） |

### 8.2 实例化对比

**CLR Assembly 加载**：
```
1. Locate assembly (GAC, probing paths, binding redirects)
2. Load PE file into memory
3. Verify strong name (if applicable)
4. Parse metadata
5. JIT compile methods on first call (lazy)
6. Run module initializer (.cctor)
7. Type loading is lazy (on first use)
```

**Wasm Module 实例化**：
```
1. Decode binary format
2. Validate module
3. Resolve imports (from host or other instances)
4. Allocate: memories, tables, globals
5. Initialize: element segments → tables, data segments → memory
6. Call start function (if present)
7. All functions available immediately (no lazy loading)
```

**关键差异**：
- CLR 是懒加载（类型和方法按需加载/编译），Wasm 是急加载（实例化时一次性完成）
- CLR 有复杂的程序集解析策略（GAC、版本重定向等），Wasm 完全依赖宿主
- CLR 支持反射和动态代码生成，Wasm 不支持

### 8.3 导入/导出对比

**CLR 的跨 Assembly 引用**：
```
// MemberRef: [TypeRef(AssemblyRef + TypeName), MethodName, Signature]
// 通过 token 解析链：
//   MemberRef → TypeRef → AssemblyRef → 加载目标 Assembly → 查找类型 → 查找方法
```

**Wasm 的导入/导出**：
```
// Import: (module_name, field_name, descriptor)
// Export: (name, descriptor)
// 直接的二级名称查找，无复杂的解析链
```

| 导入类型 | CLR | Wasm |
|---------|-----|------|
| 函数 | MethodRef / MemberRef | ✅ (func import) |
| 类型 | TypeRef | ❌ |
| 字段 | FieldRef / MemberRef | ✅ (global import) |
| 内存 | ❌ | ✅ (memory import) |
| 表 | ❌ | ✅ (table import) |
| 事件/属性 | ✅ | ❌ |

---

## 9. 安全模型对比

| 特性 | CLR | Wasm |
|------|-----|------|
| 安全边界 | AppDomain（已废弃）→ 进程 | 实例（每个实例完全隔离） |
| 代码访问安全 | CAS（已废弃） | 能力模型（只能访问导入的能力） |
| 内存安全 | 类型系统 + GC | 边界检查 + 沙箱 |
| 控制流安全 | 验证器（可绕过：unsafe） | 结构化控制流（不可绕过） |
| 系统调用 | P/Invoke（不受限） | 必须通过导入（宿主控制） |
| 文件系统访问 | 默认允许 | 默认禁止（需 WASI 导入） |
| 网络访问 | 默认允许 | 默认禁止（需宿主提供） |

**Wasm 的安全优势**：
- **默认拒绝**：Wasm 模块默认不能做任何事（无系统调用、无文件访问、无网络），所有能力必须由宿主显式授予
- **不可逃逸**：结构化控制流 + 线性内存边界检查，理论上不可能逃出沙箱
- **无 unsafe 后门**：不像 CLR 有 `unsafe` 关键字可以绕过类型安全

---

## 10. 实现复杂度对比总结

如果你实现过一个 CLR 虚拟机，以下是实现 Wasm 解释器时可以"省掉"的工作：

| 组件 | CLR 实现复杂度 | Wasm 实现复杂度 | 说明 |
|------|--------------|----------------|------|
| 二进制解析 | ★★★★☆ | ★★☆☆☆ | Wasm 格式远比 PE+Metadata 简单 |
| 类型系统 | ★★★★★ | ★☆☆☆☆ | Wasm 只有 6 种类型 |
| 验证器 | ★★★★☆ | ★★★☆☆ | 结构化控制流大幅简化验证 |
| 执行引擎 | ★★★★☆ | ★★★☆☆ | 指令更少，无 OOP 指令 |
| 内存管理 | ★★★★★ | ★☆☆☆☆ | 线性内存 vs GC 堆 |
| 异常处理 | ★★★★☆ | ☆☆☆☆☆ | Wasm 无异常（只有 trap） |
| 模块加载 | ★★★★☆ | ★★☆☆☆ | 无版本管理、无 GAC |
| 线程支持 | ★★★★☆ | ★☆☆☆☆ | Wasm threads 提案很简单 |
| 泛型支持 | ★★★★★ | ☆☆☆☆☆ | Wasm 无泛型 |
| 反射 | ★★★★☆ | ☆☆☆☆☆ | Wasm 无反射 |
| **总计** | **~42/50** | **~14/50** | **Wasm 约为 CLR 的 1/3 复杂度** |

---

## 11. 概念映射速查表

当你在实现 Wasm 解释器时遇到某个概念，可以在这里找到 CLR 中的对应物：

| Wasm 概念 | CLR 对应物 | 快速理解 |
|-----------|-----------|---------|
| Module | Assembly | 部署和加载的基本单元 |
| Type Section | StandAloneSig 表 | 函数签名定义 |
| Function Section | MethodDef 表的 Signature 列 | 函数到签名的映射 |
| Code Section | MethodDef 表的 RVA → IL Body | 函数体字节码 |
| Import Section | AssemblyRef + MemberRef 表 | 外部依赖声明 |
| Export Section | 无直接对应（public 可见性） | 对外暴露的接口 |
| Linear Memory | 非托管内存（Marshal.AllocHGlobal） | 原始字节数组 |
| Table | 委托数组 / 虚方法表 | 函数引用的间接表 |
| Global | 静态字段 | 模块级变量 |
| Element Segment | 无直接对应 | 表的初始化数据 |
| Data Segment | .data PE Section | 内存的初始化数据 |
| Start Function | Module Initializer (.cctor) | 模块加载时自动执行 |
| `block`/`loop`/`if` | 无直接对应（CIL 用 br 跳转） | 结构化控制流块 |
| `br` (label depth) | `br` (byte offset) | 分支跳转 |
| `call_indirect` | `calli` | 间接函数调用 |
| `memory.grow` | Marshal.ReAllocHGlobal | 扩展内存 |
| trap | 无直接对应（类似 AccessViolation） | 不可恢复的运行时错误 |
| Host Function | P/Invoke 目标函数 | 宿主提供的原生函数 |
| WASI | .NET BCL (System.IO 等) | 系统接口抽象层 |
| Instance | AppDomain（概念上） | 模块的运行时实例 |
| Store | CLR Runtime 本身 | 所有实例的全局容器 |

---

> **总结**：如果你已经成功实现了一个 CLR 虚拟机，那么实现一个 Wasm 解释器对你来说应该是一个"降维"的体验。
> Wasm 的设计哲学是"极简"——用最少的概念实现安全、高效、可移植的计算。
> 你在 CLR 中积累的关于栈机执行、类型验证、模块加载的经验可以直接迁移，
> 而 CLR 中最复杂的部分（GC、泛型、异常处理、反射）在 Wasm 中完全不存在。
> 享受这种简洁带来的愉悦吧！
