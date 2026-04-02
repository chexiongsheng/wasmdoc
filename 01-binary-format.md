# Wasm 二进制格式深度解析

> **前置知识**：建议先阅读 [附录 B - LEB128 编码详解](./appendix/B-leb128.md)，本文大量涉及 LEB128 编码。

## 1. 总览：模块的整体结构

一个 `.wasm` 二进制文件就是一个 **Module**（模块）的序列化表示。其整体结构非常简洁：

```
Module ::= magic | version | section*
```

```
┌──────────────────────────────────────────────────┐
│  Magic Number (4 bytes)                          │  0x00 0x61 0x73 0x6D  ("\0asm")
├──────────────────────────────────────────────────┤
│  Version (4 bytes)                               │  0x01 0x00 0x00 0x00  (little-endian 1)
├──────────────────────────────────────────────────┤
│  Section 1                                       │
├──────────────────────────────────────────────────┤
│  Section 2                                       │
├──────────────────────────────────────────────────┤
│  ...                                             │
├──────────────────────────────────────────────────┤
│  Section N                                       │
└──────────────────────────────────────────────────┘
```

### 1.1 Magic Number

前 4 个字节固定为 `0x00 0x61 0x73 0x6D`，即 ASCII 字符串 `"\0asm"`。这是 Wasm 文件的魔数标识。

> **CLR 类比**：类似于 PE 文件的 `MZ` 签名（`0x4D 0x5A`）和 PE 签名（`PE\0\0`）。不同的是 CLR 需要先经过 DOS Header → PE Signature 两层跳转，而 Wasm 直接在文件头部放置魔数，结构更扁平。

### 1.2 Version

紧接着 4 个字节是版本号，以**小端序** `uint32` 编码。当前规范版本为 `1`，即 `0x01 0x00 0x00 0x00`。

### 1.3 解析器入口伪代码

> **wasm3 实现参考**：[m3_parse.c - m3_ParseModule()](wasm3/source/m3_parse.c#L612) — 模块解析入口，验证魔数和版本号，按顺序解析各 Section。

```c
typedef struct {
    uint32_t magic;
    uint32_t version;
    // ... sections parsed dynamically
} WasmModule;

WasmModule* parse_module(const uint8_t* bytes, size_t len) {
    const uint8_t* ptr = bytes;
    
    // 1. Validate magic number
    uint32_t magic = read_u32_le(&ptr);  // 0x0061736D
    assert(magic == 0x6D736100);         // Note: little-endian read
    
    // 2. Validate version
    uint32_t version = read_u32_le(&ptr);
    assert(version == 1);
    
    // 3. Parse sections in order
    while (ptr < bytes + len) {
        uint8_t section_id = *ptr++;
        uint32_t section_size = read_leb128_u32(&ptr);
        const uint8_t* section_end = ptr + section_size;
        
        parse_section(module, section_id, ptr, section_size);
        ptr = section_end;
    }
    return module;
}
```

---

## 2. Section 通用结构

每个 Section 都遵循统一的编码格式：

```
Section ::= id:byte | size:u32 | content:byte[size]
```

```
┌─────────┬──────────────────┬─────────────────────────┐
│ id (1B) │ size (LEB128 u32)│ content (size bytes)    │
└─────────┴──────────────────┴─────────────────────────┘
```

- **id**：1 个字节，标识 Section 类型（0-12，以及 0 表示 Custom Section）
- **size**：LEB128 编码的 `u32`，表示 `content` 的字节长度（不包括 id 和 size 本身）
- **content**：Section 的实际内容，长度恰好为 `size` 字节

### 2.1 Section ID 一览

| ID | 名称 | 描述 |
|----|------|------|
| 0  | Custom | 自定义段（名称段、调试信息等） |
| 1  | Type | 函数类型签名声明 |
| 2  | Import | 导入声明 |
| 3  | Function | 函数声明（类型索引） |
| 4  | Table | 表声明 |
| 5  | Memory | 线性内存声明 |
| 6  | Global | 全局变量声明 |
| 7  | Export | 导出声明 |
| 8  | Start | 启动函数索引 |
| 9  | Element | 表初始化数据 |
| 10 | Code | 函数体（字节码） |
| 11 | Data | 内存初始化数据 |
| 12 | DataCount | Data 段计数（用于单遍验证） |

### 2.2 Section 排序规则

**关键规则**：除 Custom Section（id=0）外，所有已知 Section 必须按 ID **严格递增**顺序出现，且每种最多出现一次。Custom Section 可以出现在任意位置、任意次数。

> **wasm3 实现参考**：[m3_parse.c - ParseModuleSection()](wasm3/source/m3_parse.c#L570) — Section 分发表，根据 section id 调用对应的解析函数；[m3_parse.c#L636](wasm3/source/m3_parse.c#L636) — `sectionsOrder` 数组实现 Section 排序检查。

```
[Custom]* [Type]? [Custom]* [Import]? [Custom]* [Function]? [Custom]* ...
```

> **CLR 类比**：CLR 的 PE 文件中，各 Section（.text/.rsrc/.reloc）的顺序也有约定，但 CLR 元数据表（TypeDef/MethodDef/Field 等）的排列更像是一个关系数据库的多张表，通过 token 索引互相引用。Wasm 的 Section 设计更像是一个线性流，解析器可以单遍顺序读取。

### 2.3 通用编码约定

在深入各 Section 之前，先明确几个贯穿全文的编码约定：

| 符号 | 含义 | 编码方式 |
|------|------|----------|
| `byte` | 单个字节 | 原始字节 |
| `u32` | 无符号 32 位整数 | LEB128 |
| `s32` | 有符号 32 位整数 | LEB128 (signed) |
| `s64` | 有符号 64 位整数 | LEB128 (signed) |
| `f32` | 32 位浮点数 | IEEE 754, 4 bytes, little-endian |
| `f64` | 64 位浮点数 | IEEE 754, 8 bytes, little-endian |
| `name` | UTF-8 字符串 | `len:u32` + `bytes:byte[len]` |
| `vec(T)` | T 类型的向量 | `count:u32` + `items:T[count]` |

**`vec(T)` 是 Wasm 二进制格式中最核心的复合编码**——几乎所有 Section 的内容都是某种 `vec(T)` 结构。

---

## 3. 类型编码基础

在解析各 Section 之前，需要了解 Wasm 中类型的二进制编码：

### 3.1 值类型（Value Type）

| 字节值 | 类型 | 说明 |
|--------|------|------|
| `0x7F` | i32 | 32 位整数 |
| `0x7E` | i64 | 64 位整数 |
| `0x7D` | f32 | 32 位浮点 |
| `0x7C` | f64 | 64 位浮点 |
| `0x7B` | v128 | 128 位 SIMD（提案） |
| `0x70` | funcref | 函数引用 |
| `0x6F` | externref | 外部引用 |

> 注意这些值是 **负数的 LEB128 有符号编码**（`0x7F` = -1, `0x7E` = -2, ...），这是一个巧妙的设计——正数空间留给了类型索引（用于块类型的 `typeidx`）。

### 3.2 函数类型（Function Type）

```
functype ::= 0x60 | params:vec(valtype) | results:vec(valtype)
```

以 `0x60` 开头，后跟参数类型向量和返回值类型向量。

**示例**：函数签名 `(i32, i32) -> (i32)` 的编码：

```
0x60                    // functype tag
0x02                    // params count = 2
  0x7F                  //   param[0] = i32
  0x7F                  //   param[1] = i32
0x01                    // results count = 1
  0x7F                  //   result[0] = i32
```

### 3.3 Limits 类型

用于描述 Memory 和 Table 的大小限制：

```
limits ::= 0x00 min:u32           // only minimum
         | 0x01 min:u32 max:u32   // minimum and maximum
```

> **单位说明**：`min` 和 `max` 的单位取决于使用场景——用于 **Memory** 时单位是**页（page）**，每页 = 64 KiB（65536 字节）；用于 **Table** 时单位是**元素个数（槽位数）**。

**示例 1**：仅指定最小值（min=1，无上限）：

```
0x00                    // flags = 0 (no max)
0x01                    // min = 1
```

用于 Memory 时表示初始 1 页（64 KiB），可无限增长（实际受引擎限制）。

**示例 2**：同时指定最小值和最大值（min=2, max=10）：

```
0x01                    // flags = 1 (has max)
0x02                    // min = 2
0x0A                    // max = 10
```

用于 Memory 时表示初始 2 页（128 KiB），最大可增长到 10 页（640 KiB）。用于 Table 时表示初始 2 个槽位，最大 10 个槽位。

> **注意**：第一个字节是 flags 而非 min。`0x00` 表示只有 min，`0x01` 表示同时有 min 和 max。max 必须 ≥ min，否则模块非法。

### 3.4 块类型（Block Type）

控制流指令（block/loop/if）使用块类型描述其签名：

```
blocktype ::= 0x40                // empty: [] -> []
            | valtype             // shorthand: [] -> [valtype]
            | s33                 // type index (positive s33 as LEB128)
```

---

## 4. 各 Section 详解

### 4.1 Type Section (id = 1)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Type()](wasm3/source/m3_parse.c#L46) — 解析类型段，创建 `M3FuncType` 并通过 [m3_env.c - Environment_AddFuncType()](wasm3/source/m3_env.c#L87) 去重存入环境。类型结构定义见 [m3_function.h - M3FuncType](wasm3/source/m3_function.h#L17)。

**作用**：声明模块中所有用到的函数类型签名。其他 Section 通过**类型索引**（typeidx）引用这里的签名。

```
typesec ::= vec(functype)
```

**二进制布局**：

```
┌─────────┬──────────┬───────────────────────────────────────┐
│ 0x01    │ size     │ count | functype[0] | functype[1] ... │
└─────────┴──────────┴───────────────────────────────────────┘
```

**解析伪代码**：

```c
void parse_type_section(const uint8_t** ptr, WasmModule* mod) {
    uint32_t count = read_leb128_u32(ptr);
    mod->type_count = count;
    mod->types = malloc(count * sizeof(FuncType));
    
    for (uint32_t i = 0; i < count; i++) {
        uint8_t tag = read_byte(ptr);
        assert(tag == 0x60);  // functype marker
        
        // Parse parameter types
        mod->types[i].param_count = read_leb128_u32(ptr);
        mod->types[i].params = read_bytes(ptr, mod->types[i].param_count);
        
        // Parse result types
        mod->types[i].result_count = read_leb128_u32(ptr);
        mod->types[i].results = read_bytes(ptr, mod->types[i].result_count);
    }
}
```

> **CLR 类比**：Type Section 类似于 CLR 元数据中的 **StandAloneSig** 表 + **MethodDef** 表中的签名 blob。但 CLR 的签名编码更复杂（包含调用约定、泛型参数等），而 Wasm 的函数类型非常纯粹——只有参数和返回值类型列表。

---

### 4.2 Import Section (id = 2)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Import()](wasm3/source/m3_parse.c#L150) — 解析导入段，按 `importKind` 分别处理函数/表/内存/全局变量导入。

**作用**：声明模块需要从外部导入的实体（函数、表、内存、全局变量）。

```
importsec  ::= vec(import)
import     ::= module:name | name:name | desc:importdesc
importdesc ::= 0x00 typeidx:u32       // function import
             | 0x01 tabletype         // table import
             | 0x02 memtype           // memory import
             | 0x03 globaltype        // global import
```

**二进制布局示例**（导入函数 `env.print_i32`，类型索引为 0）：

```
0x02                    // Import Section id
0x11                    // section size = 17 bytes
0x01                    // import count = 1
  0x03                  //   module name length = 3
  0x65 0x6E 0x76        //   module name = "env"
  0x09                  //   field name length = 9
  0x70 0x72 0x69 0x6E   //   field name = "print_i32"
  0x74 0x5F 0x69 0x33
  0x32
  0x00                  //   import kind = function
  0x00                  //   type index = 0
```

**关键点**：
- 每个导入由**两级名称**标识：模块名（module）+ 字段名（name）
- 导入的函数会占据函数索引空间的**前部**（在本模块定义的函数之前）

**各种导入描述符的编码**：

| 类型 | Tag | 后续编码 |
|------|-----|----------|
| 函数 | `0x00` | `typeidx:u32` |
| 表   | `0x01` | `elemtype:byte` + `limits` |
| 内存 | `0x02` | `limits` |
| 全局 | `0x03` | `valtype:byte` + `mut:byte`（0=const, 1=var） |

> **CLR 类比**：类似于 CLR 元数据中的 **AssemblyRef** + **MemberRef** 表。CLR 通过 `[DllImport]` 或程序集引用来导入外部符号，Wasm 的导入机制更简单直接——只有两级命名空间（module.name），没有版本号、公钥等复杂的程序集标识。

---

### 4.3 Function Section (id = 3)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Function()](wasm3/source/m3_parse.c#L127) — 解析函数声明段，通过 `Module_AddFunction` 将类型索引关联到函数。

**作用**：声明本模块定义的函数的**类型索引**。注意，这里只有类型索引，函数体在 Code Section 中。

```
funcsec ::= vec(typeidx:u32)
```

**这是最简单的 Section 之一**——就是一个 `u32` 数组，每个元素是指向 Type Section 的索引。

```
0x03                    // Function Section id
0x03                    // section size = 3 bytes
0x02                    // function count = 2
  0x00                  //   func[0] type index = 0
  0x01                  //   func[1] type index = 1
```

**设计哲学**：为什么把函数声明和函数体分开？

1. **支持单遍验证**：解析器可以先知道所有函数的签名，再去验证函数体中的 `call` 指令是否类型正确
2. **支持流式编译**：编译器可以在收到 Function Section 后就开始准备编译框架，不必等到所有 Code Section 到达
3. **减少前向引用**：函数 A 可以调用函数 B，即使 B 的代码在 A 之后

> **CLR 类比**：CLR 的 **MethodDef** 表同时包含方法签名和 RVA（指向方法体的偏移），而 Wasm 将这两者拆分到不同的 Section。这种拆分设计在流式场景下更有优势。

---

### 4.4 Table Section (id = 4)

**作用**：声明模块的表（Table），表是一个存放引用类型值的数组，主要用于实现间接函数调用（`call_indirect`）。

```
tablesec  ::= vec(tabletype)
tabletype ::= elemtype:reftype | limits
reftype   ::= 0x70 (funcref) | 0x6F (externref)
```

**示例**（声明一个最小 10、最大 100 的 funcref 表）：

```
0x04                    // Table Section id
0x05                    // section size
0x01                    // table count = 1
  0x70                  //   element type = funcref
  0x01                  //   limits: has max
  0x0A                  //   min = 10
  0x64                  //   max = 100
```

> MVP 规范中，一个模块最多只能有**一个表**（后续提案放宽了此限制）。

---

### 4.5 Memory Section (id = 5)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Memory()](wasm3/source/m3_parse.c#L451) — 解析内存段；[m3_parse.c - ParseType_Memory()](wasm3/source/m3_parse.c#L22) — 解析 Limits 类型。内存数据结构见 [m3_env.h - M3Memory](wasm3/source/m3_env.h#L20)。

**作用**：声明模块的线性内存（Linear Memory）。

```
memsec  ::= vec(memtype)
memtype ::= limits
```

Limits 中的数值单位是 **页（page）**，每页 = 64 KiB（65536 字节）。

**示例**（声明一个初始 1 页、最大 256 页 = 16 MiB 的内存）：

```
0x05                    // Memory Section id
0x04                    // section size
0x01                    // memory count = 1
  0x01                  //   limits: has max
  0x01                  //   min = 1 page (64 KiB)
  0x80 0x02             //   max = 256 pages (16 MiB), LEB128 encoding
```

> **CLR 类比**：Wasm 的线性内存是一块连续的、可增长的字节数组，类似于 C 的 `malloc` 堆，但更受控。CLR 没有直接对应物——CLR 的内存模型是基于 GC 堆 + 值类型栈的，不暴露原始线性地址空间。这是 Wasm 和 CLR 在内存模型上的**根本差异**。

---

### 4.6 Global Section (id = 6)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Global()](wasm3/source/m3_parse.c#L468) — 解析全局变量段，包括初始化表达式。全局变量结构见 [m3_env.h - M3Global](wasm3/source/m3_env.h#L57)。

**作用**：声明模块的全局变量，每个全局变量有类型、可变性和初始值。

```
globalsec  ::= vec(global)
global     ::= globaltype | init:expr
globaltype ::= valtype:byte | mut:byte    // mut: 0x00=const, 0x01=var
expr       ::= instr* 0x0B               // expression terminated by 'end' opcode
```

**初始化表达式（`expr`）** 是一个受限的指令序列，只允许以下指令：
- `*.const`（常量指令：`i32.const`, `i64.const`, `f32.const`, `f64.const`）
- `global.get`（引用已导入的全局变量）
- `ref.null`, `ref.func`
- 以 `0x0B`（`end`）结尾

**示例**（声明一个可变的 i32 全局变量，初始值为 42）：

```
0x06                    // Global Section id
0x06                    // section size
0x01                    // global count = 1
  0x7F                  //   type = i32
  0x01                  //   mutable = yes
  0x41 0x2A             //   init expr: i32.const 42
  0x0B                  //   end of init expr
```

---

### 4.7 Export Section (id = 7)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Export()](wasm3/source/m3_parse.c#L230) — 解析导出段，将导出名关联到函数/全局变量/内存/表。

**作用**：声明模块对外导出的实体。

```
exportsec  ::= vec(export)
export     ::= name:name | desc:exportdesc
exportdesc ::= 0x00 funcidx:u32      // function export
             | 0x01 tableidx:u32     // table export
             | 0x02 memidx:u32       // memory export
             | 0x03 globalidx:u32    // global export
```

**示例**（导出名为 "add" 的函数，函数索引为 1）：

```
0x07                    // Export Section id
0x07                    // section size
0x01                    // export count = 1
  0x03                  //   name length = 3
  0x61 0x64 0x64        //   name = "add"
  0x00                  //   kind = function
  0x01                  //   function index = 1
```

**关键点**：
- 导出名在模块内必须**唯一**
- 导出的函数索引是**全局函数索引**（包含导入函数的偏移）

> **CLR 类比**：类似于 CLR 中将类型/方法标记为 `public` 的可见性控制。但 CLR 的导出是基于类型系统的（public class/method），而 Wasm 的导出是基于**扁平命名空间**的——只有一个名字字符串，没有类/命名空间层级。

---

### 4.8 Start Section (id = 8)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Start()](wasm3/source/m3_parse.c#L290) — 解析启动函数索引；[m3_env.c - m3_RunStart()](wasm3/source/m3_env.c#L549) — 执行启动函数。

**作用**：指定模块实例化后自动调用的启动函数。

```
startsec ::= funcidx:u32
```

这是**最简单的 Section**——只有一个函数索引，没有 `vec` 包装。

```
0x08                    // Start Section id
0x01                    // section size = 1
0x02                    // start function index = 2
```

**约束**：启动函数的类型签名必须是 `[] -> []`（无参数、无返回值）。

> **CLR 类比**：类似于 CLR 的 **Module Initializer**（`.cctor`）或程序集入口点（`EntryPointToken`）。

---

### 4.9 Element Section (id = 9)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Element()](wasm3/source/m3_parse.c#L328) — 解析元素段（仅记录位置）；[m3_env.c - InitElements()](wasm3/source/m3_env.c#L485) — 实例化时将函数引用填充到 `table0`。

**作用**：用于初始化表（Table）的内容，将函数引用填充到表的指定位置。这是实现 `call_indirect` 的关键——间接调用通过表索引查找目标函数。

Element Section 的编码在 Wasm 2.0 中变得相当复杂，有多种变体。通过一个 **flags 字段**（u32）来区分：

```
elem ::= flags:u32 ...  // flags determines the variant
```

**主要变体**：

| flags | 含义 | 编码 |
|-------|------|------|
| 0 | Active, table 0, funcidx vector | `expr` + `vec(funcidx)` |
| 1 | Passive, elemkind, funcidx vector | `elemkind` + `vec(funcidx)` |
| 2 | Active, explicit table, elemkind, funcidx vector | `tableidx` + `expr` + `elemkind` + `vec(funcidx)` |
| 3 | Declarative, elemkind, funcidx vector | `elemkind` + `vec(funcidx)` |
| 4 | Active, table 0, expr vector | `expr` + `vec(expr)` |
| 5 | Passive, reftype, expr vector | `reftype` + `vec(expr)` |
| 6 | Active, explicit table, reftype, expr vector | `tableidx` + `expr` + `reftype` + `vec(expr)` |
| 7 | Declarative, reftype, expr vector | `reftype` + `vec(expr)` |

**最常见的变体（flags=0）示例**：将函数 0、1、2 填充到表 0 的偏移 0 处：

```
0x09                    // Element Section id
0x09                    // section size
0x01                    // element segment count = 1
  0x00                  //   flags = 0 (active, table 0, funcidx)
  0x41 0x00             //   offset expr: i32.const 0
  0x0B                  //   end of expr
  0x03                  //   funcidx count = 3
  0x00                  //     funcidx[0] = 0
  0x01                  //     funcidx[1] = 1
  0x02                  //     funcidx[2] = 2
```

**Active vs Passive vs Declarative**：
- **Active**：在实例化时自动将元素复制到表中（需要偏移表达式）
- **Passive**：不自动初始化，通过 `table.init` 指令在运行时按需复制
- **Declarative**：仅声明函数引用的存在（用于 `ref.func` 验证），不实际存储

---

### 4.10 Code Section (id = 10)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Code()](wasm3/source/m3_parse.c#L345) — 解析代码段，为每个函数记录 `wasm`/`wasmEnd` 指针（零拷贝）。函数结构见 [m3_function.h - M3Function](wasm3/source/m3_function.h#L41)。

**作用**：包含所有本模块定义的函数的**函数体**。这是通常最大的 Section。

```
codesec ::= vec(code)
code    ::= size:u32 | func
func    ::= locals:vec(locals_entry) | body:expr
locals_entry ::= count:u32 | type:valtype
```

**关键设计**：每个 `code` 条目有自己的 `size` 前缀，这允许解析器**跳过**不需要的函数体，或者并行解析多个函数体。

**局部变量的压缩编码**：局部变量不是逐个声明的，而是按类型分组压缩：

```
// 3 个 i32 + 2 个 f64 的局部变量
// 不是编码为 [i32, i32, i32, f64, f64]
// 而是编码为 [(3, i32), (2, f64)]
locals_entry[0] = { count: 3, type: i32 }
locals_entry[1] = { count: 2, type: f64 }
```

**完整示例**（一个将两个 i32 参数相加的函数体）：

```
0x0A                    // Code Section id
0x09                    // section size
0x01                    // function body count = 1
  0x07                  //   body size = 7 bytes
  0x00                  //   local declaration count = 0 (no locals)
  0x20 0x00             //   local.get 0 (first param)
  0x20 0x01             //   local.get 1 (second param)
  0x6A                  //   i32.add
  0x0B                  //   end
```

**函数体结构详解**：

```
┌──────────────────────────────────────────────────────────┐
│ code entry                                               │
├──────────┬───────────────────────────────────────────────┤
│ size(u32)│ func                                          │
│          ├──────────────────┬────────────────────────────┤
│          │ locals: vec(     │ body: expr                 │
│          │  (count, type))  │ (instructions... + 0x0B)   │
│          │                  │                            │
└──────────┴──────────────────┴────────────────────────────┘
```

> **CLR 类比**：Code Section 类似于 CLR 中 MethodDef 表的 RVA 指向的**方法体**（Method Body）。CLR 方法体也有类似结构：头部（flags + max stack + code size + local var sig token）+ IL 字节码 + 异常处理表。Wasm 的函数体更简洁——没有异常处理表（Wasm 没有异常机制，除了 trap），也没有 max stack 字段（通过验证时静态计算）。

**注意**：Code Section 中函数体的顺序必须与 Function Section 中类型索引的顺序**一一对应**。即 `code[i]` 是 `function[i]` 的函数体。

---

### 4.11 Data Section (id = 11)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Data()](wasm3/source/m3_parse.c#L412) — 解析数据段；[m3_env.c - InitDataSegments()](wasm3/source/m3_env.c#L456) — 实例化时将数据复制到线性内存。

**作用**：用于初始化线性内存的内容，类似于可执行文件中的 `.data` 段。

```
datasec ::= vec(data)
data    ::= 0x00 offset:expr init:vec(byte)     // active, memory 0
          | 0x01 init:vec(byte)                  // passive
          | 0x02 memidx:u32 offset:expr init:vec(byte)  // active, explicit memory
```

**示例**（将字符串 "Hello" 写入内存偏移 0 处）：

```
0x0B                    // Data Section id
0x0B                    // section size
0x01                    // data segment count = 1
  0x00                  //   flags = 0 (active, memory 0)
  0x41 0x00             //   offset expr: i32.const 0
  0x0B                  //   end of expr
  0x05                  //   data length = 5
  0x48 0x65 0x6C 0x6C   //   "Hell"
  0x6F                  //   "o"
```

**Active vs Passive**：
- **Active**：实例化时自动将数据复制到内存的指定偏移处
- **Passive**：不自动初始化，通过 `memory.init` 指令在运行时按需复制

---

### 4.12 DataCount Section (id = 12)

**作用**：声明 Data Section 中数据段的数量。这个 Section 是后来添加的，目的是支持**单遍验证**。

```
datacountsec ::= count:u32
```

**为什么需要它？** 在验证 Code Section 中的 `memory.init` 和 `data.drop` 指令时，验证器需要知道数据段的数量来检查索引是否越界。但 Data Section 在 Code Section **之后**。DataCount Section（id=12）放在 Code Section（id=10）**之前**，解决了这个前向引用问题。

```
0x0C                    // DataCount Section id
0x01                    // section size = 1
0x03                    // data segment count = 3
```

---

### 4.13 Custom Section (id = 0)

> **wasm3 实现参考**：[m3_parse.c - ParseSection_Custom()](wasm3/source/m3_parse.c#L551) — 解析自定义段，识别 `"name"` 段并调用 [ParseSection_Name()](wasm3/source/m3_parse.c#L459)；支持通过 [m3_env.c - m3_SetCustomSectionHandler()](wasm3/source/m3_env.c#L80) 注册自定义处理器。

**作用**：存放自定义数据，解析器可以忽略不认识的 Custom Section。

```
customsec ::= name:name | bytes:byte*
```

Custom Section 的内容以一个 **name** 开头（标识其用途），后面是任意字节。

**常见的 Custom Section**：

| 名称 | 用途 |
|------|------|
| `"name"` | 调试信息：函数名、局部变量名等 |
| `"producers"` | 生成工具信息 |
| `"target_features"` | 目标特性声明 |
| `"linking"` | 链接信息（用于目标文件） |
| `.debug_*` | DWARF 调试信息 |

**Name Section 详解**（最常用的 Custom Section）：

Name Section 包含多个子段（subsection），每个子段有自己的 id：

| 子段 ID | 名称 | 内容 |
|---------|------|------|
| 0 | Module name | 模块名称 |
| 1 | Function names | 函数索引 → 名称映射 |
| 2 | Local names | 函数索引 → (局部变量索引 → 名称) 映射 |

```
// Name Section example
0x00                    // Custom Section id
0x12                    // section size
0x04                    // name length = 4
0x6E 0x61 0x6D 0x65    // name = "name"
// subsections follow...
0x01                    //   subsection id = 1 (function names)
0x09                    //   subsection size
0x02                    //   name count = 2
  0x00                  //     func index = 0
  0x03 0x61 0x64 0x64   //     name = "add"
  0x01                  //     func index = 1
  0x03 0x73 0x75 0x62   //     name = "sub" (partial, for illustration)
```

---

## 5. 索引空间（Index Spaces）

Wasm 使用多个独立的**索引空间**来引用不同类型的实体。理解索引空间是正确解析和验证的关键。

### 5.1 索引空间一览

| 索引空间 | 引用目标 | 来源 |
|----------|----------|------|
| `typeidx` | Type Section 中的函数类型 | Type Section 顺序 |
| `funcidx` | 函数（导入 + 本地） | Import(func) ++ Function Section |
| `tableidx` | 表（导入 + 本地） | Import(table) ++ Table Section |
| `memidx` | 内存（导入 + 本地） | Import(memory) ++ Memory Section |
| `globalidx` | 全局变量（导入 + 本地） | Import(global) ++ Global Section |
| `elemidx` | Element 段 | Element Section 顺序 |
| `dataidx` | Data 段 | Data Section 顺序 |
| `localidx` | 局部变量（含参数） | 参数 ++ 局部变量声明 |
| `labelidx` | 控制流标签 | 运行时栈（相对深度） |

### 5.2 函数索引空间的合并规则

> **wasm3 实现参考**：[m3_env.h - M3Module](wasm3/source/m3_env.h#L82) — `numFuncImports` 和 `numFunctions` 字段体现了导入/本地函数的合并；[m3_env.h#L96](wasm3/source/m3_env.h#L96) — `functions` 数组包含所有函数（导入在前）。

**这是最容易出错的地方**：函数索引空间由导入函数和本地函数**合并**而成。

```
funcidx:  0        1        2        3        4
          ├────────┼────────┼────────┼────────┤
          │ import │ import │ local  │ local  │ local
          │ func 0 │ func 1 │ func 0 │ func 1 │ func 2
          └────────┴────────┴────────┴────────┘
```

- 导入函数占据索引 `0` 到 `import_func_count - 1`
- 本地函数从索引 `import_func_count` 开始
- Function Section 中的第 `i` 个条目对应 `funcidx = import_func_count + i`
- Code Section 中的第 `i` 个函数体对应 Function Section 的第 `i` 个条目

**同样的合并规则适用于 table、memory、global 索引空间。**

### 5.3 局部变量索引空间

在函数体内部，局部变量索引空间由**参数**和**局部变量声明**合并：

```
localidx: 0      1      2      3      4
          ├──────┼──────┼──────┼──────┤
          │param │param │local │local │local
          │  0   │  1   │  0   │  1   │  2
          └──────┴──────┴──────┴──────┘
```

> **CLR 类比**：CLR 的元数据表也使用索引（token）来互相引用，但 CLR 使用的是**编码 token**（高 8 位是表 ID，低 24 位是行号），而 Wasm 使用的是**纯序号索引**。CLR 的 token 系统更灵活（可以跨表引用），但 Wasm 的纯索引更简单高效。

---

## 6. 完整解析流程

将所有 Section 的解析串联起来，一个完整的模块解析器的工作流程如下：

```
┌─────────────────────────────────────────────────────────────┐
│                    Module Parsing Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Read & validate magic (0x0061736D)                      │
│  2. Read & validate version (1)                             │
│  3. Loop: read section_id + section_size                    │
│     │                                                       │
│     ├─ id=0  → parse_custom_section (or skip)               │
│     ├─ id=1  → parse_type_section    → types[]              │
│     ├─ id=2  → parse_import_section  → imports[]            │
│     ├─ id=3  → parse_function_section → func_type_indices[] │
│     ├─ id=4  → parse_table_section   → tables[]             │
│     ├─ id=5  → parse_memory_section  → memories[]           │
│     ├─ id=6  → parse_global_section  → globals[]            │
│     ├─ id=7  → parse_export_section  → exports[]            │
│     ├─ id=8  → parse_start_section   → start_funcidx       │
│     ├─ id=9  → parse_element_section → elements[]           │
│     ├─ id=10 → parse_code_section    → codes[]              │
│     ├─ id=11 → parse_data_section    → datas[]              │
│     └─ id=12 → parse_datacount_section → data_count         │
│                                                             │
│  4. Post-parse validation:                                  │
│     - func_type_indices.len == codes.len                    │
│     - All type indices in range                             │
│     - All function indices in range                         │
│     - Section ordering is correct                           │
│     - DataCount matches actual data segment count           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.1 解析器核心数据结构（C 语言）

```c
// Value types
typedef enum {
    WASM_TYPE_I32     = 0x7F,
    WASM_TYPE_I64     = 0x7E,
    WASM_TYPE_F32     = 0x7D,
    WASM_TYPE_F64     = 0x7C,
    WASM_TYPE_V128    = 0x7B,
    WASM_TYPE_FUNCREF = 0x70,
    WASM_TYPE_EXTERNREF = 0x6F,
} WasmValType;

// Function type signature
typedef struct {
    uint32_t    param_count;
    WasmValType *params;
    uint32_t    result_count;
    WasmValType *results;
} WasmFuncType;

// Limits (for memory and table)
typedef struct {
    uint32_t min;
    uint32_t max;       // UINT32_MAX if no max
    bool     has_max;
} WasmLimits;

// Import entry
typedef struct {
    char        *module_name;
    char        *field_name;
    uint8_t     kind;       // 0=func, 1=table, 2=memory, 3=global
    union {
        uint32_t    type_index;     // for func
        struct { uint8_t elem_type; WasmLimits limits; } table;
        WasmLimits  memory;
        struct { WasmValType type; bool mutable_; } global;
    } desc;
} WasmImport;

// Export entry
typedef struct {
    char        *name;
    uint8_t     kind;
    uint32_t    index;
} WasmExport;

// Code body
typedef struct {
    uint32_t    local_count;    // total locals (expanded from compressed form)
    WasmValType *local_types;
    uint32_t    body_size;
    const uint8_t *body;        // raw bytecode (pointer into original buffer)
} WasmCode;

// Complete module
typedef struct {
    // Type Section
    uint32_t      type_count;
    WasmFuncType  *types;
    
    // Import Section
    uint32_t      import_count;
    WasmImport    *imports;
    uint32_t      import_func_count;   // derived: count of function imports
    uint32_t      import_table_count;
    uint32_t      import_memory_count;
    uint32_t      import_global_count;
    
    // Function Section
    uint32_t      func_count;
    uint32_t      *func_type_indices;
    
    // Table Section
    uint32_t      table_count;
    // ... table types
    
    // Memory Section
    uint32_t      memory_count;
    WasmLimits    *memories;
    
    // Global Section
    uint32_t      global_count;
    // ... global types + init exprs
    
    // Export Section
    uint32_t      export_count;
    WasmExport    *exports;
    
    // Start Section
    bool          has_start;
    uint32_t      start_funcidx;
    
    // Element Section
    uint32_t      element_count;
    // ... element segments
    
    // Code Section
    uint32_t      code_count;
    WasmCode      *codes;
    
    // Data Section
    uint32_t      data_count;
    // ... data segments
    
    // DataCount Section
    bool          has_data_count;
    uint32_t      data_count_value;
} WasmModule;
```

---

## 7. 实战：手工解析一个最小 Wasm 模块

让我们手工解析一个完整的 `.wasm` 文件。这个模块导出一个 `add` 函数，接受两个 `i32` 参数，返回它们的和。

对应的 WAT（WebAssembly Text Format）：

```wasm
(module
  (func $add (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add)
  (export "add" (func $add)))
```

**完整的二进制编码**（逐字节注释）：

```
; === Module Header ===
00 61 73 6D          ; magic: "\0asm"
01 00 00 00          ; version: 1

; === Type Section (id=1) ===
01                   ; section id = 1 (Type)
07                   ; section size = 7 bytes
01                   ; type count = 1
  60                 ;   functype tag
  02                 ;   param count = 2
    7F               ;     param[0] = i32
    7F               ;     param[1] = i32
  01                 ;   result count = 1
    7F               ;     result[0] = i32

; === Function Section (id=3) ===
03                   ; section id = 3 (Function)
02                   ; section size = 2 bytes
01                   ; function count = 1
  00                 ;   func[0] type index = 0

; === Export Section (id=7) ===
07                   ; section id = 7 (Export)
07                   ; section size = 7 bytes
01                   ; export count = 1
  03                 ;   name length = 3
  61 64 64           ;   name = "add"
  00                 ;   kind = function
  00                 ;   function index = 0

; === Code Section (id=10) ===
0A                   ; section id = 10 (Code)
09                   ; section size = 9 bytes
01                   ; function body count = 1
  07                 ;   body size = 7 bytes
  00                 ;   local declaration count = 0
  20 00              ;   local.get 0
  20 01              ;   local.get 1
  6A                 ;   i32.add
  0B                 ;   end
```

**总计 41 字节**。你可以用以下命令验证：

```bash
# Write the hex bytes to a file
echo -n -e '\x00\x61\x73\x6d\x01\x00\x00\x00\x01\x07\x01\x60\x02\x7f\x7f\x01\x7f\x03\x02\x01\x00\x07\x07\x01\x03\x61\x64\x64\x00\x00\x0a\x09\x01\x07\x00\x20\x00\x20\x01\x6a\x0b' > add.wasm

# Verify with wasm tools
wasm-objdump -d add.wasm
wasm-validate add.wasm
```

---

## 8. 与 CLR PE 格式的结构对比

| 维度 | Wasm Binary | CLR PE |
|------|-------------|--------|
| **文件标识** | 4 字节魔数 `\0asm` | DOS Header (`MZ`) → PE Signature (`PE\0\0`) |
| **版本** | 4 字节 little-endian u32 | PE Optional Header 中的多个版本字段 |
| **元数据组织** | 线性 Section 序列（id 递增） | 多层嵌套（PE Section → CLR Header → Metadata Root → Stream → Tables） |
| **类型信息** | Type Section（函数签名列表） | TypeDef/TypeRef/TypeSpec 等多张元数据表 |
| **方法声明** | Function Section（类型索引数组） | MethodDef 表（含 RVA、flags、签名） |
| **方法体** | Code Section（独立的函数体数组） | .text section 中的 IL 方法体（通过 RVA 定位） |
| **导入** | Import Section（module.name 二级命名） | AssemblyRef + MemberRef + ImplMap 表 |
| **导出** | Export Section（扁平命名空间） | 类型/方法的 public 可见性 |
| **初始化数据** | Data Section → 线性内存 | .data PE section / FieldRVA 表 |
| **字符串编码** | UTF-8, length-prefixed | #Strings 堆（null-terminated）/ #US 堆（length-prefixed UTF-16） |
| **整数编码** | LEB128 | Compressed unsigned int（1/2/4 字节，类似但不同于 LEB128） |
| **交叉引用** | 纯序号索引（u32） | 编码 token（table_id << 24 | row_index） |
| **可扩展性** | Custom Section（可忽略） | Custom Attribute 元数据 |
| **设计目标** | 流式解析、单遍验证、紧凑 | 随机访问、丰富的类型系统、向后兼容 |

**核心差异总结**：

1. **扁平 vs 层级**：Wasm 的二进制格式是扁平的线性结构，适合流式处理；CLR 的 PE 格式是多层嵌套的，适合随机访问
2. **简单 vs 丰富**：Wasm 只有函数类型签名；CLR 有完整的面向对象类型系统（类、接口、泛型、继承等）
3. **声明/定义分离**：Wasm 将函数声明（Function Section）和定义（Code Section）分开；CLR 在 MethodDef 表中同时包含声明和定义的 RVA
4. **编码效率**：Wasm 大量使用 LEB128 变长编码，对小数值非常紧凑；CLR 使用固定大小的表行（2 或 4 字节列宽），查找效率更高但空间利用率较低

---

## 9. 解析器实现要点与常见陷阱

### 9.1 错误处理策略

解析器应该区分两类错误：
- **格式错误（malformed）**：二进制编码不合法（如 LEB128 超长、Section 大小不匹配）
- **非法模块（invalid）**：格式正确但语义不合法（如类型索引越界、函数数量不匹配）

建议在解析阶段只处理格式错误，语义验证留给专门的验证器。

### 9.2 常见陷阱

1. **LEB128 最大字节数**：Wasm 规范要求 u32 的 LEB128 编码最多 5 字节，u64 最多 10 字节。超过此限制的编码是非法的。

2. **Section 大小必须精确消耗**：解析完一个 Section 的内容后，当前位置必须恰好等于 `section_start + section_size`。多了或少了都是错误。

3. **局部变量数量溢出**：Code Section 中的局部变量使用压缩编码 `(count, type)`，展开后的总数可能溢出 u32。需要在累加时检查。

4. **UTF-8 验证**：所有 `name` 字段必须是合法的 UTF-8 编码。

5. **函数体的 `end` 指令**：每个函数体必须以 `0x0B`（`end`）结尾，这个 `end` 是函数体隐含的最外层 block 的结束标记。

6. **DataCount 与 Data 的一致性**：如果存在 DataCount Section，其值必须等于 Data Section 中的实际段数。

### 9.3 性能考虑

> **wasm3 实现参考**：wasm3 的解析器正是采用了下述优化策略。参见 [m3_parse.c#L338](wasm3/source/m3_parse.c#L338) — Code Section 解析时只记录 `func->wasm` 和 `func->wasmEnd` 指针，不复制字节码。

- **零拷贝解析**：对于 Code Section 中的函数体字节码，可以只记录指针和长度，不复制数据。实际的指令解码推迟到验证或执行阶段。
- **惰性解析**：Custom Section 通常可以跳过不解析（只记录位置），除非调试器需要。
- **内存预分配**：每个 `vec(T)` 都有前置的 count，可以一次性分配数组，避免动态扩容。

---

## 10. 下一步

掌握了二进制格式后，你已经可以编写一个完整的 `.wasm` 文件解析器了。下一步建议：

1. **动手实现**：用 C 语言实现一个模块解析器，能够解析并打印出所有 Section 的内容
2. **测试验证**：用 `wat2wasm` 生成各种测试用例，用你的解析器解析并与 `wasm-objdump` 的输出对比
3. **继续学习**：阅读 [02-instruction-set.md](./02-instruction-set.md) 深入了解指令集，这是实现执行引擎的基础
