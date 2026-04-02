# Wasm 指令集与类型系统

> **前置阅读**：[01-binary-format.md](./01-binary-format.md)、[appendix/B-leb128.md](./appendix/B-leb128.md)
>
> **目标**：完整理解 Wasm 的类型系统设计和全部指令集，为实现指令解码器和执行引擎打下基础。

---

## 1. 类型系统总览

> **wasm3 实现参考**：[wasm3.h - M3ValueType](wasm3/source/wasm3.h#L70) — 值类型枚举定义（`c_m3Type_i32`/`i64`/`f32`/`f64`）；[m3_core.c - IsIntType()](wasm3/source/m3_core.c#L223)、[Is64BitType()](wasm3/source/m3_core.c#L231) — 类型判断辅助函数；[m3_compile.c - ReadBlockType()](wasm3/source/m3_compile.c#L1797) — 块类型解析（区分 empty/valtype/typeidx 三种情况）。

Wasm 的类型系统是**静态的、可验证的**。所有类型信息在编译期（加载验证期）即可完全确定，运行时无需类型元数据。这与 CLR 的丰富类型系统形成鲜明对比——Wasm 的类型系统极度精简，只服务于安全验证和正确执行。

### 1.1 值类型（Value Types）

Wasm 的值类型是指令操作的基本数据单元：

| 编码字节 | 类型名 | 描述 | 位宽 |
|----------|--------|------|------|
| `0x7F` | `i32` | 32-bit 整数 | 32 |
| `0x7E` | `i64` | 64-bit 整数 | 64 |
| `0x7D` | `f32` | 32-bit IEEE 754 浮点数 | 32 |
| `0x7C` | `f64` | 64-bit IEEE 754 浮点数 | 64 |
| `0x7B` | `v128` | 128-bit SIMD 向量（提案） | 128 |
| `0x70` | `funcref` | 函数引用 | — |
| `0x6F` | `externref` | 外部引用（宿主对象） | — |

**关键设计决策**：

1. **无符号/有符号不区分**：`i32` 和 `i64` 不区分有符号和无符号。符号性由**指令**决定，而非类型。例如 `i32.div_s`（有符号除法）vs `i32.div_u`（无符号除法）操作的都是同一个 `i32` 类型。

2. **无布尔类型**：布尔值用 `i32` 表示（0 = false，非 0 = true）。这与 CIL 类似（CIL 也用 `int32` 表示布尔）。

3. **无指针类型**：Wasm 没有原生指针。内存访问通过 `i32`（或 memory64 提案中的 `i64`）作为地址偏移来实现。这是安全沙箱的核心保障。

4. **引用类型是不透明的**：`funcref` 和 `externref` 不能被解释为整数或进行算术运算，只能传递、存储和调用。

> **CLR 对比**：CLR 有丰富的值类型（`int8/16/32/64`、`float32/64`、`bool`、`char`、用户定义的 `struct`）和引用类型（`class`、`interface`、`delegate`、`string`、`object`）。Wasm 的类型系统相当于 CLR 类型系统的一个极小子集——只保留了机器原语。

### 1.2 函数类型（Function Types）

函数类型描述了函数的签名——参数列表和返回值列表：

```
functype ::= 0x60 vec(valtype) vec(valtype)
             ~~~~~~ ~~~~~~~~~~ ~~~~~~~~~~
             标记    参数类型    返回类型
```

- 前缀 `0x60` 标识这是一个函数类型
- 参数和返回值都是 `valtype` 的向量（vec = LEB128 长度 + 元素序列）
- **多返回值**：Wasm 2.0 支持多个返回值（MVP 只允许 0 或 1 个）

**示例**：

```
; 函数签名: (i32, i32) -> (i32)
; 二进制编码:
0x60                ; functype 标记
0x02                ; 参数数量 = 2
  0x7F              ;   param[0] = i32
  0x7F              ;   param[1] = i32
0x01                ; 返回值数量 = 1
  0x7F              ;   result[0] = i32
```

函数类型存储在 **Type Section** 中，通过 **typeidx**（类型索引）引用。多个函数可以共享同一个类型索引（如果签名相同）。

> **CLR 对比**：CLR 的方法签名（MethodDef/MethodRef）包含更多信息：调用约定（default/vararg/generic）、this 指针、泛型参数等。Wasm 的函数类型纯粹是 `[valtype*] → [valtype*]` 的映射，没有任何面向对象的概念。

### 1.3 块类型（Block Types）

块类型用于 `block`、`loop`、`if` 等结构化控制流指令，描述该块的输入/输出类型：

| 编码 | 含义 | 描述 |
|------|------|------|
| `0x40` | empty | 无参数、无返回值 |
| `0x7F`/`0x7E`/`0x7D`/`0x7C` | valtype | 无参数、返回一个值 |
| LEB128 有符号整数 (≥0) | typeidx | 引用 Type Section 中的函数类型（支持多参数多返回值） |

**编码歧义消解**：`0x40` 和 valtype 编码（`0x7F`~`0x7B`、`0x70`、`0x6F`）都是单字节负数（作为有符号 LEB128 解释），而 typeidx 是非负整数。解析器通过符号位来区分：

```c
// Pseudocode for block type decoding
int64_t raw = read_leb128_signed(stream);
if (raw == -0x40) {
    // empty block type: [] -> []
} else if (raw < 0) {
    // valtype: [] -> [decode_valtype(raw)]
} else {
    // typeidx: types[raw] gives full functype
}
```

### 1.4 结果类型（Result Types）

结果类型是值类型的向量，用于描述块、函数的输出：

```
resulttype ::= vec(valtype)
```

在 MVP 中，结果类型最多包含一个值。多返回值提案（已合入 2.0）解除了这个限制。

### 1.5 限制类型（Limits）

用于描述 Memory 和 Table 的大小约束：

```
limits ::= 0x00 n:u32          ; min = n, max = 无限制
         | 0x01 n:u32 m:u32    ; min = n, max = m
```

### 1.6 内存类型与表类型

```
memtype   ::= limits                    ; 以页（64KB）为单位
tabletype ::= reftype limits            ; reftype = funcref | externref
globaltype ::= valtype mut              ; mut: 0x00 = const, 0x01 = var
```

---

## 2. 指令集架构总览

> **wasm3 实现参考**：[m3_compile.c - c_operations](wasm3/source/m3_compile.c#L2260) — 完整的操作码到 M3OpInfo 映射表，包含每条指令的栈偏移、类型、operation 函数指针和编译器函数；[appendix/A-opcode-table.md](appendix/A-opcode-table.md) — 完整操作码速查表。

### 2.1 指令编码格式

每条 Wasm 指令由一个**操作码**（opcode）和零或多个**立即数**（immediates）组成：

```
instr ::= opcode immediate*
```

- **操作码**：大多数是单字节（`0x00`~`0xC4`），部分使用双字节前缀（`0xFC xx` 用于扩展数值指令，`0xFD xx` 用于 SIMD 指令）
- **立即数**：根据指令不同，可能是 LEB128 编码的索引、内存偏移、常量值等

### 2.2 指令分类

Wasm 指令集按功能分为以下几大类：

```
┌─────────────────────────────────────────────────┐
│              Wasm Instruction Set                │
├──────────────┬──────────────────────────────────┤
│ Control Flow │ block, loop, if, br, br_if,      │
│   (0x00-0x11)│ br_table, return, call,          │
│              │ call_indirect, unreachable, nop   │
├──────────────┼──────────────────────────────────┤
│ Parametric   │ drop, select                     │
│   (0x1A-0x1C)│                                  │
├──────────────┼──────────────────────────────────┤
│ Variable     │ local.get/set/tee,               │
│   (0x20-0x24)│ global.get/set                   │
├──────────────┼──────────────────────────────────┤
│ Memory       │ i32.load, i32.store, ...         │
│   (0x28-0x40)│ memory.size, memory.grow         │
├──────────────┼──────────────────────────────────┤
│ Numeric      │ i32.const, i32.add, i32.mul, ... │
│   (0x41-0xC4)│ f64.sqrt, i32.wrap_i64, ...     │
├──────────────┼──────────────────────────────────┤
│ Reference    │ ref.null, ref.is_null, ref.func  │
│   (0xD0-0xD2)│                                  │
├──────────────┼──────────────────────────────────┤
│ Table        │ table.get/set/size/grow/fill/    │
│   (0x25-0x26,│ copy/init, elem.drop             │
│    0xFC 0C+) │                                  │
├──────────────┼──────────────────────────────────┤
│ Extended     │ i32.trunc_sat_f32_s, memory.copy,│
│   (0xFC xx)  │ memory.fill, table.*, ...        │
├──────────────┼──────────────────────────────────┤
│ SIMD         │ v128.load, i8x16.add, ...        │
│   (0xFD xx)  │ (128-bit vector operations)      │
└──────────────┴──────────────────────────────────┘
```

### 2.3 栈机模型

Wasm 是一个**栈式虚拟机**（stack machine）。指令从操作数栈（operand stack）弹出输入，将结果压入栈：

```
; Example: (a + b) * c
local.get 0    ; stack: [a]
local.get 1    ; stack: [a, b]
i32.add        ; stack: [a+b]
local.get 2    ; stack: [a+b, c]
i32.mul        ; stack: [(a+b)*c]
```

每条指令都有明确的**栈签名**（stack signature）：`[input types] → [output types]`

例如：
- `i32.add`: `[i32, i32] → [i32]`
- `i32.const n`: `[] → [i32]`
- `drop`: `[t] → []`（多态，接受任意类型）

> **CLR 对比**：CIL 也是栈机模型，这一点完全相同。`i32.add` 对应 CIL 的 `add`，`local.get` 对应 `ldloc`，`local.set` 对应 `stloc`。如果你实现过 CLR 的栈式求值，Wasm 的数值运算部分会非常熟悉。

---

## 3. 控制流指令（核心难点）

> **wasm3 实现参考**：控制流编译见 [m3_compile.c - Compile_LoopOrBlock()](wasm3/source/m3_compile.c#L1851)、[Compile_If()](wasm3/source/m3_compile.c#L1869)、[Compile_Branch()](wasm3/source/m3_compile.c#L1533)；运行时执行见 [m3_exec.h - op_Loop()](wasm3/source/m3_exec.h#L864)、[op_Branch()](wasm3/source/m3_exec.h#L1240)、[op_BranchTable()](wasm3/source/m3_exec.h#L1275)。

控制流是 Wasm 指令集中**最独特、最重要**的部分，也是与 CIL 差异最大的地方。理解结构化控制流模型是实现 Wasm 解释器的关键。

### 3.1 结构化控制流 vs 非结构化跳转

**CIL 的方式（非结构化）**：
```
; CIL: if (x > 0) { y = 1; } else { y = 2; }
ldloc.0
ldc.i4.0
ble ELSE_LABEL      ; 跳转到任意偏移
ldc.i4.1
stloc.1
br END_LABEL         ; 跳转到任意偏移
ELSE_LABEL:
ldc.i4.2
stloc.1
END_LABEL:
```

CIL 使用**标签 + 偏移跳转**，可以跳到函数内的任意位置（包括向前和向后）。这类似于汇编语言的 `jmp`。

**Wasm 的方式（结构化）**：
```wasm
;; Wasm: if (x > 0) { y = 1; } else { y = 2; }
local.get $x
i32.const 0
i32.gt_s
if (result i32)      ;; 开始 if 块
  i32.const 1        ;;   true 分支
else                 ;;   分隔符
  i32.const 2        ;;   false 分支
end                  ;; 结束 if 块
local.set $y
```

Wasm 使用**嵌套的块结构**（block/loop/if...end），所有跳转只能跳到**包围自己的块的边界**。不存在任意偏移跳转。

**为什么这样设计？**

1. **安全性**：结构化控制流保证不会跳到指令中间或非法位置
2. **可验证性**：单遍线性扫描即可完成类型检查（不需要构建控制流图）
3. **可编译性**：结构化控制流可以直接映射到 CPU 的分支指令，无需复杂的控制流分析

### 3.2 block 指令

```
block blocktype instr* end
```

`block` 定义一个顺序执行的代码块。`br` 跳转到 block 时，跳到 block 的 **end 之后**（向前跳转）：

```
block $B (result i32)     ; 标签 $B 指向 end 之后
  i32.const 42
  br $B                   ; 跳到 end 之后，栈顶的 i32 作为块的结果
  i32.const 99            ; 永远不会执行（dead code）
end
; 此处栈顶 = 42
```

**二进制编码**：
```
0x02        ; block opcode
blocktype   ; 块类型（empty / valtype / typeidx）
instr*      ; 块体指令序列
0x0B        ; end opcode
```

### 3.3 loop 指令

```
loop blocktype instr* end
```

`loop` 定义一个循环块。`br` 跳转到 loop 时，跳到 loop 的 **开头**（向后跳转，形成循环）：

```
block $exit (result i32)
  loop $continue            ; 标签 $continue 指向 loop 开头
    ;; loop body
    local.get $i
    i32.const 10
    i32.lt_s
    if
      ;; i < 10: increment and continue
      local.get $i
      i32.const 1
      i32.add
      local.set $i
      br $continue          ; 跳回 loop 开头（继续循环）
    end
    ;; i >= 10: exit
    local.get $i
    br $exit                ; 跳出外层 block
  end
end
```

**关键区别**：
- `br` 到 `block` → 跳到 block 的 **end**（前向，退出）
- `br` 到 `loop` → 跳到 loop 的 **开头**（后向，继续）

这个区别对实现者至关重要！

> **实现提示**：在解释器中，你需要为每个控制结构记录两个位置：
> - **continuation**（继续点）：block 的 end 之后 / loop 的开头
> - 当 `br` 执行时，跳转到目标标签的 continuation

### 3.4 if/else 指令

```
if blocktype instr* end
if blocktype instr* else instr* end
```

`if` 从栈顶弹出一个 `i32`，非零执行 true 分支，零执行 false 分支（else）：

```
local.get $cond
if (result i32)       ; 弹出 $cond，检查是否非零
  i32.const 1         ; true 分支
else
  i32.const 2         ; false 分支
end
; 栈顶 = 1 或 2
```

**二进制编码**：
```
0x04        ; if opcode
blocktype   ; 块类型
instr*      ; true 分支
[0x05       ; else opcode（可选）
instr*]     ; false 分支
0x0B        ; end opcode
```

注意：`else` 不是独立指令，而是 `if` 块内部的分隔符。如果没有 else 分支且块类型非 empty，则验证会失败（true 分支必须产生结果，false 分支也必须产生结果）。

### 3.5 br / br_if / br_table —— 分支跳转

这三条指令是 Wasm 控制流的核心：

#### br（无条件分支）

```
br labelidx
```

- `labelidx` 是一个 LEB128 编码的无符号整数，表示**相对深度**（relative depth）
- `0` = 最内层的包围块，`1` = 次内层，以此类推
- 跳转到目标块的 continuation 点

```
block $L0                    ; depth 2 (from br's perspective)
  block $L1                  ; depth 1
    block $L2                ; depth 0
      br 0                   ; 跳到 $L2 的 end
      br 1                   ; 跳到 $L1 的 end
      br 2                   ; 跳到 $L0 的 end
    end ;; $L2
  end ;; $L1
end ;; $L0
```

**br 的栈行为**：
- 跳转时，从当前栈中弹出目标块的**结果类型**对应的值
- 丢弃栈上多余的值
- 将结果值传递给目标块

```
block (result i32)
  i32.const 100
  i32.const 42
  br 0              ; 弹出 i32(42) 作为 block 的结果，丢弃 100
end
; 栈顶 = 42
```

#### br_if（条件分支）

```
br_if labelidx
```

从栈顶弹出一个 `i32` 条件值：
- 非零：执行 `br labelidx`
- 零：不跳转，继续执行下一条指令

```
local.get $x
i32.const 0
i32.eq
br_if $exit         ; if x == 0, break out
```

#### br_table（跳转表）

```
br_table vec(labelidx) labelidx
```

这是 Wasm 的 switch/case 实现。从栈顶弹出一个 `i32` 索引值：
- 如果索引在范围内（0 ~ len-1），跳转到对应的 label
- 如果索引越界，跳转到最后的 default label

```
; switch (x) { case 0: ...; case 1: ...; case 2: ...; default: ...; }
local.get $x
br_table 0 1 2 3    ; labels: [case0=0, case1=1, case2=2], default=3
```

**二进制编码**：
```
0x0E                ; br_table opcode
vec_count (LEB128)  ; label 数量
labelidx*           ; label 索引数组（每个 LEB128）
labelidx            ; default label 索引（LEB128）
```

> **CLR 对比**：`br_table` 类似于 CIL 的 `switch` 指令，但 CIL 的 `switch` 使用绝对偏移数组，而 Wasm 使用相对深度标签索引。CIL 可以跳到函数内任意位置，Wasm 只能跳到包围块的边界。

### 3.6 return 指令

```
return
```

等价于 `br` 到最外层（函数级别）的标签。从栈中弹出函数返回类型对应的值，终止函数执行。

### 3.7 unreachable 指令

```
unreachable
```

立即触发 trap（运行时错误）。用于标记不可达代码路径。在验证时，`unreachable` 之后的代码具有**栈多态性**（可以假设栈上有任意类型的值），这在验证章节会详细讨论。

### 3.8 结构化控制流的标签模型（实现者视角）

理解标签模型对实现解释器至关重要。让我们用一个统一的模型来描述：

```
┌─────────────────────────────────────────────┐
│ Label Stack (maintained during execution)    │
│                                              │
│ 每个 block/loop/if 进入时压入一个 Label：     │
│                                              │
│ Label {                                      │
│   arity: u32,        // 结果值数量            │
│   continuation: PC,  // br 跳转目标地址       │
│   kind: block|loop,  // 决定 continuation     │
│   stack_height: u32, // 进入时的栈高度         │
│ }                                            │
│                                              │
│ block: continuation = end 之后的 PC           │
│ loop:  continuation = loop 开头的 PC          │
│ if:    continuation = end 之后的 PC（同 block）│
└─────────────────────────────────────────────┘
```

**br 的执行算法**：

```c
void exec_br(uint32_t label_depth) {
    Label* target = &label_stack[label_stack_top - label_depth];

    // 1. Pop arity values from operand stack
    Value results[target->arity];
    for (int i = target->arity - 1; i >= 0; i--)
        results[i] = pop_value();

    // 2. Unwind: discard everything above target's stack height
    while (value_stack_top > target->stack_height)
        pop_value();

    // 3. Push results back
    for (int i = 0; i < target->arity; i++)
        push_value(results[i]);

    // 4. If branching out of blocks (not loop), also pop labels
    if (target->kind != LOOP) {
        // Pop all labels up to and including target
        label_stack_top -= (label_depth + 1);
    }

    // 5. Jump to continuation
    pc = target->continuation;
}
```

> **CLR 对比**：CLR 没有标签栈的概念。CIL 的 `br` 直接修改 PC 到目标偏移，不涉及栈展开（栈的一致性由验证器在加载时保证）。Wasm 的 `br` 更复杂，因为它需要在运行时处理栈展开和值传递。

---

## 4. 变量操作指令

> **wasm3 实现参考**：编译见 [m3_compile.c - Compile_GetLocal()](wasm3/source/m3_compile.c#L1323)、[Compile_SetLocal()](wasm3/source/m3_compile.c#L1293)（同时处理 `local.set` 和 `local.tee`）、[Compile_GetSetGlobal()](wasm3/source/m3_compile.c#L1382)；运行时执行见 [m3_exec.h - op_GetGlobal_s32()](wasm3/source/m3_exec.h#L506)、[op_SetGlobal_i32()](wasm3/source/m3_exec.h#L526)、[op_SetGlobal_s32()](wasm3/source/m3_exec.h#L1275)。局部变量在 wasm3 中通过栈槽（slot）直接寻址，无需专门的 get/set 运行时操作。

### 4.1 局部变量（Local Variables）

```
local.get localidx    ; 0x20 + LEB128 index → push value
local.set localidx    ; 0x21 + LEB128 index → pop value, store
local.tee localidx    ; 0x22 + LEB128 index → peek value, store (不弹出)
```

- 局部变量索引空间：**参数在前，局部声明在后**
- 例如函数 `(param i32 i64) (local f32)`：index 0 = i32 参数，index 1 = i64 参数，index 2 = f32 局部变量
- 所有局部变量在函数入口时初始化为零值

```
; local.tee is equivalent to:
; local.set $x
; local.get $x
; but more efficient (one instruction instead of two)

local.get $a
local.get $b
i32.add
local.tee $result    ; store to $result AND keep on stack
```

> **CLR 对比**：完全对应 CIL 的 `ldloc`/`stloc`/`ldarg`/`starg`。但 CIL 区分参数（`ldarg`）和局部变量（`ldloc`），Wasm 统一编址。CIL 没有 `tee` 等价物（需要 `dup` + `stloc`）。

### 4.2 全局变量（Global Variables）

```
global.get globalidx  ; 0x23 + LEB128 index → push value
global.set globalidx  ; 0x24 + LEB128 index → pop value, store
```

- 全局变量可以是 `const`（不可变）或 `var`（可变）
- `global.set` 只能用于 `var` 全局变量，否则验证失败
- 全局变量可以是导入的或模块内定义的

---

## 5. 数值运算指令

> **wasm3 实现参考**：[m3_compile.c - c_operations[]](wasm3/source/m3_compile.c#L2260) — 完整操作码映射表（含所有数值指令的 M3OpInfo）；[c_operationsFC[]](wasm3/source/m3_compile.c#L2527) — 0xFC 前缀扩展指令映射表。

数值指令是 Wasm 指令集中数量最多的部分，但也是最直观的——如果你实现过 CLR 的算术指令，这部分会非常熟悉。

### 5.1 常量指令

```
i32.const value    ; 0x41 + LEB128 signed i32
i64.const value    ; 0x42 + LEB128 signed i64
f32.const value    ; 0x43 + 4 bytes (IEEE 754 little-endian)
f64.const value    ; 0x44 + 8 bytes (IEEE 754 little-endian)
```

> **wasm3 实现参考**：编译见 [m3_compile.c - Compile_Const_i32()](wasm3/source/m3_compile.c#L1153)、[Compile_Const_f32()](wasm3/source/m3_compile.c#L1177)；运行时执行见 [m3_exec.h - op_Const32()](wasm3/source/m3_exec.h#L1240)、[op_Const64()](wasm3/source/m3_exec.h#L1249)。

注意浮点常量是**原始字节**编码（不是 LEB128），直接 4/8 字节小端序。

### 5.2 整数算术运算

> **wasm3 实现参考**：[m3_exec.h#L196](wasm3/source/m3_exec.h#L196) — `OP_ADD/SUB/MUL` 宏 + `d_m3CommutativeOpFunc_i` 生成 add/mul 运行时操作；[m3_exec.h#L240](wasm3/source/m3_exec.h#L240) — `d_m3OpMacro_i` 生成 Divide/Remainder 操作（含除零和溢出 trap 检查）。

| 指令 | 栈签名 | 语义 | 备注 |
|------|--------|------|------|
| `i32.add` | `[i32, i32] → [i32]` | a + b | 溢出回绕（wrapping） |
| `i32.sub` | `[i32, i32] → [i32]` | a - b | 溢出回绕 |
| `i32.mul` | `[i32, i32] → [i32]` | a × b | 溢出回绕 |
| `i32.div_s` | `[i32, i32] → [i32]` | a ÷ b（有符号） | 除零或溢出 → trap |
| `i32.div_u` | `[i32, i32] → [i32]` | a ÷ b（无符号） | 除零 → trap |
| `i32.rem_s` | `[i32, i32] → [i32]` | a % b（有符号） | 除零 → trap |
| `i32.rem_u` | `[i32, i32] → [i32]` | a % b（无符号） | 除零 → trap |

`i64` 版本完全对应，只是操作 64-bit 值。

**Trap 条件**：
- 整数除零：`div_s`/`div_u`/`rem_s`/`rem_u` 当除数为 0
- 有符号溢出：`i32.div_s` 当 `INT32_MIN / -1`（结果无法表示）

> **实现提示**：C 语言中 `INT32_MIN / -1` 是未定义行为！你必须在执行除法前显式检查这个边界情况。

### 5.3 整数位运算

> **wasm3 实现参考**：[m3_exec.h#L219](wasm3/source/m3_exec.h#L219) — `d_m3CommutativeOp_i` 生成 And/Or/Xor 操作；[m3_exec.h#L216](wasm3/source/m3_exec.h#L216) — `d_m3OpFunc_i` 生成 ShiftLeft/ShiftRight 操作；[m3_exec.h#L234](wasm3/source/m3_exec.h#L234) — `d_m3OpFunc_i` 生成 Rotl/Rotr 操作。

| 指令 | 语义 |
|------|------|
| `i32.and` | 按位与 |
| `i32.or` | 按位或 |
| `i32.xor` | 按位异或 |
| `i32.shl` | 左移（移位量 mod 32） |
| `i32.shr_s` | 算术右移（有符号，移位量 mod 32） |
| `i32.shr_u` | 逻辑右移（无符号，移位量 mod 32） |
| `i32.rotl` | 循环左移 |
| `i32.rotr` | 循环右移 |

**注意**：移位量自动取模（i32 mod 32，i64 mod 64），不会 trap。

### 5.4 整数比较运算

> **wasm3 实现参考**：[m3_exec.h#L175](wasm3/source/m3_exec.h#L175) — `d_m3CommutativeOp_i`/`d_m3Op_i` 生成 Equal/NotEqual/LessThan/GreaterThan 等比较操作；[m3_exec.h#L289](wasm3/source/m3_exec.h#L289) — `d_m3UnaryOp_i(i32, EqualToZero, OP_EQZ)` 生成 eqz 操作。

所有比较运算返回 `i32`（0 或 1）：

| 指令 | 语义 |
|------|------|
| `i32.eqz` | `[i32] → [i32]`：a == 0 ? 1 : 0 |
| `i32.eq` | a == b |
| `i32.ne` | a ≠ b |
| `i32.lt_s` / `i32.lt_u` | a < b（有符号/无符号） |
| `i32.gt_s` / `i32.gt_u` | a > b |
| `i32.le_s` / `i32.le_u` | a ≤ b |
| `i32.ge_s` / `i32.ge_u` | a ≥ b |

### 5.5 整数一元运算

> **wasm3 实现参考**：[m3_exec.h#L306](wasm3/source/m3_exec.h#L306) — `OP_CLZ_32/64`、`OP_CTZ_32/64` 宏（含平台特定的 `__builtin_clz` 优化和零值边界处理）；[m3_exec.h#L316](wasm3/source/m3_exec.h#L316) — `d_m3UnaryOp_i` 生成 Clz/Ctz/Popcnt 操作。

| 指令 | 语义 |
|------|------|
| `i32.clz` | 前导零计数（count leading zeros） |
| `i32.ctz` | 尾随零计数（count trailing zeros） |
| `i32.popcnt` | 置位计数（population count） |

> **实现提示**：大多数 CPU 有对应的硬件指令（`__builtin_clz`、`_lzcnt_u32` 等）。如果你的目标平台没有，需要软件模拟。

### 5.6 浮点运算

> **wasm3 实现参考**：[m3_exec.h#L229](wasm3/source/m3_exec.h#L229) — `d_m3CommutativeOp_f`/`d_m3Op_f` 生成浮点 Add/Sub/Mul/Div 操作；[m3_exec.h#L250](wasm3/source/m3_exec.h#L250) — `d_m3OpFunc_f` 生成 Min/Max/CopySign 操作；[m3_exec.h#L281](wasm3/source/m3_exec.h#L281) — `d_m3UnaryOp_f` 生成 Abs/Ceil/Floor/Trunc/Sqrt/Nearest/Negate 操作。

浮点运算严格遵循 **IEEE 754-2019** 标准：

| 指令 | 语义 | 特殊行为 |
|------|------|----------|
| `f32.add/sub/mul/div` | 基本算术 | NaN 传播 |
| `f32.sqrt` | 平方根 | 负数 → NaN |
| `f32.min/max` | 最小/最大值 | NaN 参数 → NaN；±0 处理有规定 |
| `f32.ceil/floor/trunc/nearest` | 取整 | nearest = 银行家舍入 |
| `f32.abs/neg` | 绝对值/取反 | 仅翻转符号位 |
| `f32.copysign` | 复制符号位 | |

**NaN 处理的关键规则**：
- 如果任何输入是 NaN，结果是 NaN
- Wasm 规范要求 NaN 的 payload 是**非确定性的**（arithmetic NaN），但必须是某个合法的 NaN
- `f32.abs` 和 `f32.neg` 不是算术运算，它们只操作符号位，保留 NaN payload

> **实现提示**：如果你用 C 的 `float`/`double` 类型直接运算，大部分情况下 IEEE 754 语义是正确的。但要注意：
> 1. `f32.min(-0, +0)` 必须返回 `-0`（C 的 `fmin` 可能不保证）
> 2. NaN 的 canonical 化：Wasm spec test 会检查 NaN 的 quiet/signaling 位

### 5.7 类型转换指令

> **wasm3 实现参考**：[m3_exec.h#L353](wasm3/source/m3_exec.h#L353) — `d_m3TruncMacro` 生成所有 Trunc/TruncSat 操作（含溢出和 NaN 检查）；[m3_exec.h#L425](wasm3/source/m3_exec.h#L425) — `d_m3TypeConvertOp` 生成 Convert 操作；[m3_exec.h#L421](wasm3/source/m3_exec.h#L421) — `d_m3TypeModifyOp` 生成 Demote/Promote 操作；[m3_exec.h#L465](wasm3/source/m3_exec.h#L465) — `d_m3ReinterpretOp` 生成 Reinterpret 操作；编译见 [m3_compile.c - Compile_Convert()](wasm3/source/m3_compile.c#L2176)。

Wasm 提供了丰富的类型转换指令：

**整数截断（float → int）**：
```
i32.trunc_f32_s    ; f32 → i32（有符号截断，溢出/NaN → trap）
i32.trunc_f32_u    ; f32 → i32（无符号截断，溢出/NaN → trap）
i32.trunc_f64_s    ; f64 → i32
i32.trunc_f64_u    ; f64 → i32
i64.trunc_f32_s    ; f32 → i64
...
```

**饱和截断（不 trap 版本，0xFC 前缀）**：
```
i32.trunc_sat_f32_s  ; 0xFC 0x00 — 溢出时饱和到 INT32_MIN/MAX，NaN → 0
i32.trunc_sat_f32_u  ; 0xFC 0x01
...
```

**整数扩展**：
```
i64.extend_i32_s     ; i32 → i64（符号扩展）
i64.extend_i32_u     ; i32 → i64（零扩展）
i32.wrap_i64         ; i64 → i32（截断高 32 位）
```

**浮点转换**：
```
f32.convert_i32_s    ; i32（有符号）→ f32
f64.promote_f32      ; f32 → f64（无损）
f32.demote_f64       ; f64 → f32（可能精度损失）
```

**位重解释（reinterpret）**：
```
i32.reinterpret_f32  ; 不改变位模式，只改变类型解释
f32.reinterpret_i32
i64.reinterpret_f64
f64.reinterpret_i64
```

> **CLR 对比**：CIL 的 `conv.i4`/`conv.r8` 等转换指令与 Wasm 的转换指令功能类似，但 CIL 的溢出行为不同——CIL 的 `conv.ovf.i4` 在溢出时抛出 `OverflowException`，而 Wasm 的 `trunc` 在溢出时触发 trap（不可恢复）。Wasm 的 `trunc_sat` 系列则提供了饱和语义（不 trap）。

### 5.8 符号扩展指令

> **wasm3 实现参考**：[m3_exec.h#L340](wasm3/source/m3_exec.h#L340) — `OP_EXTEND8_S_I32` 等宏定义 + `d_m3UnaryOp_i` 生成所有符号扩展操作；[m3_exec.h#L325](wasm3/source/m3_exec.h#L325) — `OP_WRAP_I64` 宏 + `op_i32_Wrap_i64` 操作。

```
i32.extend8_s      ; 将 i32 的低 8 位符号扩展到 32 位
i32.extend16_s     ; 将 i32 的低 16 位符号扩展到 32 位
i64.extend8_s      ; 将 i64 的低 8 位符号扩展到 64 位
i64.extend16_s     ; 将 i64 的低 16 位符号扩展到 64 位
i64.extend32_s     ; 将 i64 的低 32 位符号扩展到 64 位
```

---

## 6. 内存操作指令

> **wasm3 实现参考**：[m3_exec.h#L1399](wasm3/source/m3_exec.h#L1399) — `d_m3Load` 宏（生成所有 load 变体）；[m3_exec.h#L1459](wasm3/source/m3_exec.h#L1459) — `d_m3Store` 宏（生成所有 store 变体）；[m3_exec.h - op_MemSize()](wasm3/source/m3_exec.h#L696)、[op_MemGrow()](wasm3/source/m3_exec.h#L725)。

### 6.1 线性内存模型

Wasm 的内存是一个**连续的字节数组**（线性内存），以 **64KB 页**为单位分配。所有内存访问通过 `i32` 偏移地址进行（最大 4GB 地址空间）。

### 6.2 Load 指令

```
i32.load     memarg    ; 从内存加载 32-bit 整数
i32.load8_s  memarg    ; 加载 8-bit，符号扩展到 i32
i32.load8_u  memarg    ; 加载 8-bit，零扩展到 i32
i32.load16_s memarg    ; 加载 16-bit，符号扩展到 i32
i32.load16_u memarg    ; 加载 16-bit，零扩展到 i32
i64.load     memarg    ; 加载 64-bit 整数
i64.load8_s  memarg    ; ... (同理)
i64.load8_u  memarg
i64.load16_s memarg
i64.load16_u memarg
i64.load32_s memarg
i64.load32_u memarg
f32.load     memarg    ; 加载 32-bit 浮点
f64.load     memarg    ; 加载 64-bit 浮点
```

### 6.3 Store 指令

```
i32.store    memarg    ; 存储 32-bit
i32.store8   memarg    ; 截断存储低 8-bit
i32.store16  memarg    ; 截断存储低 16-bit
i64.store    memarg
i64.store8   memarg
i64.store16  memarg
i64.store32  memarg
f32.store    memarg
f64.store    memarg
```

### 6.4 memarg（内存参数）

每个 load/store 指令都有一个 `memarg` 立即数：

```
memarg ::= align:u32 offset:u32
```

- **align**：对齐提示（2 的幂的指数，即 `align=2` 表示 4 字节对齐）。这只是**提示**，不是强制要求。非对齐访问也必须正确工作（只是可能更慢）。
- **offset**：静态偏移量。实际地址 = 栈顶弹出的 `i32` 基址 + offset

```
; effective_address = pop(i32) + offset
; if effective_address + size > memory.size: trap (out of bounds)
; value = memory[effective_address .. effective_address + size]
```

**二进制编码**：
```
0x28        ; i32.load opcode
0x02        ; align = 2 (即 4 字节对齐，2^2 = 4)
0x08        ; offset = 8
; 执行时：addr = pop_i32() + 8; push_i32(mem[addr..addr+4])
```

> **实现提示**：
> 1. 所有内存访问都是**小端序**（little-endian），即使宿主机是大端
> 2. 边界检查：`effective_address + access_size` 不能超过当前内存大小
> 3. 对齐提示可以忽略（但如果你做 JIT，对齐信息可以用于优化）

### 6.5 memory.size 和 memory.grow

```
memory.size    ; 0x3F 0x00 → push 当前内存页数 (i32)
memory.grow    ; 0x40 0x00 → pop 增长页数 (i32), push 旧页数或 -1（失败）
```

`memory.grow` 的返回值：
- 成功：返回增长前的页数
- 失败：返回 `-1`（作为 i32，即 `0xFFFFFFFF`）

> **CLR 对比**：CLR 没有线性内存的概念。CLR 的内存管理由 GC 托管，对象分配在堆上，通过引用访问。Wasm 的线性内存更像 C 的 `malloc` + 指针算术，但被限制在一个固定的沙箱区域内。

### 6.6 批量内存操作（0xFC 前缀）

```
memory.copy    ; 0xFC 0x0A — 内存区域复制（类似 memmove）
memory.fill    ; 0xFC 0x0B — 内存区域填充（类似 memset）
```

栈签名：
- `memory.copy`: `[dest:i32, src:i32, len:i32] → []`
- `memory.fill`: `[dest:i32, val:i32, len:i32] → []`

---

## 7. 参数指令（Parametric Instructions）

> **wasm3 实现参考**：编译见 [m3_compile.c - Compile_Drop()](wasm3/source/m3_compile.c#L2063)、[Compile_Select()](wasm3/source/m3_compile.c#L1983)；运行时执行见 [m3_exec.h - d_m3Select_i()](wasm3/source/m3_exec.h#L1055)、[d_m3Select_f()](wasm3/source/m3_exec.h#L1105)。`drop` 在 wasm3 中纯粹是编译期操作（弹出虚拟栈），不生成运行时指令。

### 7.1 drop

```
drop    ; 0x1A — [t] → []
```

丢弃栈顶值。类似 CIL 的 `pop`。

### 7.2 select

```
select           ; 0x1B — [t, t, i32] → [t]
select (t*)      ; 0x1C — [t, t, i32] → [t]（带类型注解版本）
```

三元选择：弹出条件值（i32）和两个操作数，条件非零返回第一个，零返回第二个：

```
local.get $a       ; stack: [a]
local.get $b       ; stack: [a, b]
local.get $cond    ; stack: [a, b, cond]
select             ; stack: [cond ? a : b]
```

`select`（0x1B）要求两个操作数是相同的数值类型（i32/i64/f32/f64）。
`select (t*)`（0x1C）是带类型注解的版本，支持引用类型。

---

## 8. 函数调用指令

> **wasm3 实现参考**：[m3_exec.h - op_Call()](wasm3/source/m3_exec.h#L542) — 直接调用；[m3_exec.h - op_CallIndirect()](wasm3/source/m3_exec.h#L568) — 间接调用（含运行时类型检查）；[m3_compile.c - Compile_Call()](wasm3/source/m3_compile.c#L1659) — call 指令编译。

### 8.1 call

```
call funcidx    ; 0x10 + LEB128 函数索引
```

直接调用。从栈弹出参数（按函数签名），执行目标函数，将返回值压栈。

```
; 调用函数 $add (param i32 i32) (result i32)
i32.const 3
i32.const 4
call $add        ; pop 3, 4; push 7
```

### 8.2 call_indirect

```
call_indirect typeidx tableidx    ; 0x11 + LEB128 类型索引 + LEB128 表索引
```

间接调用（通过函数表）。从栈弹出一个 `i32` 索引，在指定的 table 中查找函数引用，然后调用：

```
; table[idx] 必须是一个函数引用，且其签名必须与 typeidx 匹配
i32.const 3        ; 参数 1
i32.const 4        ; 参数 2
local.get $idx     ; 表索引
call_indirect (type $binop_type) ; 类型检查 + 调用
```

**运行时检查**：
1. 表索引越界 → trap
2. 表项为 `null` → trap
3. 函数签名不匹配 → trap

这是 Wasm 中**唯一的运行时类型检查**。其他所有类型检查都在验证期完成。

> **CLR 对比**：`call_indirect` 类似于 CIL 的 `calli`（通过函数指针调用）或 `callvirt`（虚方法调用需要运行时查表）。但 Wasm 的间接调用更受限——只能通过 table 索引，不能使用任意函数指针。

### 8.3 return_call 和 return_call_indirect（尾调用提案）

```
return_call funcidx              ; 尾调用优化版 call
return_call_indirect typeidx tableidx  ; 尾调用优化版 call_indirect
```

这些指令复用当前函数的栈帧，避免栈增长。对于递归密集的 Wasm 程序很重要。

---

## 9. 引用指令

> **wasm3 实现参考**：wasm3 对引用类型的支持有限。在 [m3_compile.c - c_operations[]](wasm3/source/m3_compile.c#L2260) 中，`ref.null`（0xD0）、`ref.is_null`（0xD1）、`ref.func`（0xD2）被标记为 `M3OP_RESERVED`，表示尚未实现。

### 9.1 ref.null

```
ref.null reftype    ; 0xD0 + reftype → push null reference
```

将一个 null 引用压栈。`reftype` 可以是 `funcref`（0x70）或 `externref`（0x6F）。

### 9.2 ref.is_null

```
ref.is_null    ; 0xD1 — [ref] → [i32]
```

检查栈顶引用是否为 null。是 → 1，否 → 0。

### 9.3 ref.func

```
ref.func funcidx    ; 0xD2 + LEB128 函数索引 → push funcref
```

将指定函数的引用压栈。该函数必须在模块的 element segment 或 export 中被声明过。

---

## 10. 表操作指令

> **wasm3 实现参考**：wasm3 对表操作的支持有限。在 [m3_compile.c - c_operations[]](wasm3/source/m3_compile.c#L2260) 中，`table.get`（0x25）和 `table.set`（0x26）被标记为 `M3OP_RESERVED`；在 [c_operationsFC[]](wasm3/source/m3_compile.c#L2527) 中，`table.init`/`elem.drop`/`table.copy` 等扩展表操作同样未实现。

表（Table）是 Wasm 中存储引用类型的数组结构，主要用于实现间接函数调用。

### 10.1 基本表操作

```
table.get tableidx    ; 0x25 — [i32] → [ref]
table.set tableidx    ; 0x26 — [i32, ref] → []
```

### 10.2 扩展表操作（0xFC 前缀）

```
table.size tableidx     ; 0xFC 0x10 — [] → [i32]
table.grow tableidx     ; 0xFC 0x0F — [ref, i32] → [i32]
table.fill tableidx     ; 0xFC 0x11 — [i32, ref, i32] → []
table.copy dst src      ; 0xFC 0x0E — [i32, i32, i32] → []
table.init tableidx elemidx  ; 0xFC 0x0C — [i32, i32, i32] → []
elem.drop elemidx       ; 0xFC 0x0D — [] → []
```

---

## 11. 与 CIL 指令集的关键差异总结

| 维度 | Wasm | CIL (.NET) |
|------|------|------------|
| **控制流模型** | 结构化（block/loop/if + br 到标签深度） | 非结构化（br/brfalse + 任意偏移） |
| **跳转目标** | 只能跳到包围块的边界 | 可以跳到函数内任意位置 |
| **类型区分** | 符号性由指令决定（`div_s` vs `div_u`） | 符号性由类型决定（`int` vs `uint`） |
| **内存访问** | 显式 load/store + 线性地址 | 隐式（对象字段访问、数组索引） |
| **函数调用** | `call`（直接）+ `call_indirect`（表索引） | `call`/`callvirt`/`calli`/`newobj` |
| **异常处理** | 无（trap 不可恢复）；Exception Handling 提案进行中 | 完整的 try/catch/finally/throw |
| **对象模型** | 无 | 完整的 OOP（类、接口、继承、泛型） |
| **验证时机** | 加载时单遍验证 | 加载时 + JIT 时验证 |
| **栈帧** | 每个 block/loop/if 有隐式栈帧 | 只有方法调用有栈帧 |

### 11.1 最关键的思维转换

如果你从 CLR 转向 Wasm，最需要适应的是：

1. **没有 goto**：你不能随意跳转。所有控制流必须通过嵌套的 block/loop/if 表达。这意味着你的解释器不需要处理"跳到指令流中间"的情况，大大简化了实现。

2. **br 的语义不是"跳到标签"而是"退出到第 N 层块"**：这是一个根本性的思维差异。在 CIL 中，`br target` 的 target 是一个地址。在 Wasm 中，`br N` 的 N 是一个深度计数器。

3. **block 和 loop 的 br 方向相反**：`br` 到 `block` 是向前跳（退出），`br` 到 `loop` 是向后跳（继续）。这两个行为用同一条指令实现，只是目标块的类型不同。

4. **没有异常**：Wasm 的错误处理是 trap（类似于 SIGILL/SIGSEGV），不可恢复。没有 try/catch。这简化了执行引擎——你不需要实现异常展开机制。

---

## 12. 指令编码快速参考

以下是各类指令的操作码范围速览（详细表格见 [appendix/A-opcode-table.md](./appendix/A-opcode-table.md)）：

| 范围 | 类别 | 示例 |
|------|------|------|
| `0x00`-`0x01` | 控制（unreachable, nop） | |
| `0x02`-`0x04` | 块结构（block, loop, if） | |
| `0x05` | else | |
| `0x0B` | end | |
| `0x0C`-`0x0E` | 分支（br, br_if, br_table） | |
| `0x0F` | return | |
| `0x10`-`0x11` | 调用（call, call_indirect） | |
| `0x12`-`0x13` | 尾调用（return_call, return_call_indirect） | |
| `0x1A`-`0x1C` | 参数（drop, select） | |
| `0x20`-`0x24` | 变量（local.*, global.*） | |
| `0x25`-`0x26` | 表（table.get, table.set） | |
| `0x28`-`0x3E` | 内存 load/store | |
| `0x3F`-`0x40` | 内存管理（memory.size, memory.grow） | |
| `0x41`-`0x44` | 常量（*.const） | |
| `0x45`-`0x69` | i32 运算 | |
| `0x6A`-`0x8E` | i64 运算 | |
| `0x8F`-`0xA6` | f32 运算 | |
| `0xA7`-`0xBF` | f64 运算 | |
| `0xC0`-`0xC4` | 符号扩展 | |
| `0xD0`-`0xD2` | 引用（ref.*） | |
| `0xFC` + u32 | 扩展指令（trunc_sat, memory.*, table.*） | |
| `0xFD` + u32 | SIMD 指令（v128.*） | |

---

## 13. 实现者检查清单

在实现指令解码器和执行引擎时，确保覆盖以下要点：

- [ ] **LEB128 解码**：所有立即数索引使用 LEB128 编码
- [ ] **块类型解析**：正确区分 empty / valtype / typeidx 三种块类型
- [ ] **控制流标签栈**：维护标签栈，正确处理 br 的相对深度
- [ ] **block vs loop 的 br 方向**：block 向前跳，loop 向后跳
- [ ] **br_table 的 default 标签**：最后一个标签是 default
- [ ] **整数除法 trap**：除零和有符号溢出（INT_MIN / -1）
- [ ] **浮点 NaN 处理**：NaN 传播、canonical NaN
- [ ] **浮点截断 trap**：`trunc` 系列在溢出/NaN 时 trap
- [ ] **内存边界检查**：每次 load/store 都要检查
- [ ] **小端字节序**：内存访问始终小端
- [ ] **call_indirect 类型检查**：运行时签名匹配
- [ ] **select 类型一致性**：两个操作数必须同类型
- [ ] **局部变量零初始化**：函数入口时所有局部变量为零值

---

> **下一步**：[03-validation.md](./03-validation.md) — 理解 Wasm 如何在加载时验证指令序列的类型安全性
