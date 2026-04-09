# wasm3 源码导读与架构分析

> 本文基于 wasm3 v0.5.0 源码进行分析。wasm3 是目前最快的 WebAssembly 解释器之一，以极小的代码体积（~15K LOC）和出色的性能著称，特别适合嵌入式和资源受限环境。其核心创新在于 **M3（Meta-Machine-Method）** 架构——一种将 Wasm 字节码转译为 C 函数指针链的 threaded code 技术。

---

## 1. 项目整体架构

### 1.1 代码组织结构

wasm3 的源码组织非常精简，核心代码集中在 `source/` 目录下：

```
wasm3/source/
├── m3_api_defs.h          # API macro definitions
├── m3_api_libc.c/h        # libc bindings (optional)
├── m3_api_meta_wasi.c     # meta WASI implementation
├── m3_api_tracer.c/h      # execution tracer (debug)
├── m3_api_uvwasi.c/h      # uvwasi bindings
├── m3_api_wasi.c/h        # WASI implementation
├── m3_bind.c/h            # host function binding
├── m3_code.c/h            # M3 code page management
├── m3_compile.c/h         # ★ core: Wasm → M3 compilation
├── m3_core.c/h            # core utilities, LEB128, types
├── m3_env.c/h             # ★ core: environment/runtime/module management
├── m3_exec.c/h            # ★ core: M3 execution engine (operations)
├── m3_exec_defs.h         # execution macros and definitions
├── m3_function.c/h        # function metadata management
├── m3_info.c/h            # debug info and disassembly
├── m3_math_utils.h        # math helper macros
├── m3_parse.c/h           # ★ core: Wasm binary parser
├── wasm3.c/h              # public API
└── wasm3_defs.h           # global configuration defines
```

标注 ★ 的是最核心的四个模块，理解它们就掌握了 wasm3 的精髓。

### 1.2 三层架构模型

wasm3 采用清晰的三层架构：

```
┌─────────────────────────────────────────────────┐
│                  Public API Layer                │
│              (wasm3.c / wasm3.h)                 │
│  m3_NewEnvironment / m3_NewRuntime / m3_Call ... │
├─────────────────────────────────────────────────┤
│               Management Layer                   │
│             (m3_env.c / m3_parse.c)              │
│  Module parsing, Environment/Runtime lifecycle   │
├─────────────────────────────────────────────────┤
│            Compilation + Execution Layer          │
│          (m3_compile.c / m3_exec.c)              │
│  Wasm→M3 translation, Threaded code dispatch     │
└─────────────────────────────────────────────────┘
```

### 1.3 生命周期流程

一个 Wasm 模块从加载到执行的完整流程：

```
m3_ParseModule()          ← Parse .wasm binary into M3Module
       │
m3_LoadModule()           ← Attach module to runtime, resolve imports
       │
m3_FindFunction()         ← Locate exported function, trigger lazy compilation
       │                     (Wasm bytecode → M3 threaded code)
m3_Call() / m3_CallV()    ← Execute the M3 threaded code
```

关键设计决策：**延迟编译（lazy compilation）**。wasm3 不会在加载时编译所有函数，而是在首次调用（或查找）时才触发编译。这大幅减少了启动时间和内存占用。

---

## 2. 核心数据结构

### 2.1 M3Environment

> **源码位置**：[m3_env.h#L139](wasm3/source/m3_env.h#L139) — 结构定义；[m3_env.c - m3_NewEnvironment()](wasm3/source/m3_env.c#L17) — 创建；[m3_env.c - Environment_AddFuncType()](wasm3/source/m3_env.c#L87) — 类型去重注册。

```c
typedef struct M3Environment {
    IM3FuncType             funcTypes;          // linked list of unique function types
    IM3FuncType             retFuncTypes;       // return-only function types cache
    u16                     numFuncTypes;
}
M3Environment;
```

**Environment** 是最顶层的容器，管理全局共享的函数类型池。多个 Runtime 可以共享同一个 Environment。

**类比 CLR**：类似于 CLR 的 `AppDomain`（在 .NET Core 中已弱化），提供类型的全局注册表。但 wasm3 的 Environment 更轻量，仅管理函数签名的去重。

### 2.2 M3Runtime

> **源码位置**：[m3_env.h#L159](wasm3/source/m3_env.h#L161) — 结构定义；[m3_env.c - m3_NewRuntime()](wasm3/source/m3_env.c#L173) — 创建；[m3_env.c - m3_LoadModule()](wasm3/source/m3_env.c#L596) — 加载模块。

```c
typedef struct M3Runtime {
    M3Compilation           compilation;        // reusable compilation state
    IM3Environment          environment;

    IM3Module               modules;            // linked list of loaded modules

    void *                  stack;              // unified value + call stack
    u32                     stackSize;
    u32                     numStackSlots;
    IM3Module               stackMemoryModule;  // module owning the stack memory

    M3Memory                memory;             // linear memory instance
    u32                     numActiveCodePages;
    IM3CodePage             pagesFull;          // full code pages
    IM3CodePage             pagesOpen;          // code pages with free space

    // ... error handling, user data, etc.
}
M3Runtime;
```

**Runtime** 是执行的核心容器，拥有：
- **统一栈（unified stack）**：值栈和调用栈共享同一块内存，从两端向中间增长
- **线性内存（linear memory）**：Wasm 的 linear memory 实例
- **代码页（code pages）**：存储编译后的 M3 threaded code

**类比 CLR**：类似于 CLR 的执行引擎上下文，但 wasm3 将值栈和调用栈统一管理，而 CLR 的 evaluation stack 和 call stack 是分离的。

### 2.3 M3Module

> **源码位置**：[m3_env.h#L82](wasm3/source/m3_env.h#L82) — 结构定义（包含函数数组、全局变量、导入导出等）。

```c
typedef struct M3Module {
    struct M3Runtime *      runtime;
    struct M3Environment *  environment;

    bytes_t                 wasmBytes;          // raw .wasm binary (retained)

    u32                     numFuncTypes;
    IM3FuncType *           funcTypes;          // type section entries

    u32                     numFuncImports;
    u32                     numFunctions;
    IM3Function *           functions;          // all functions (imports + defined)

    u32                     numImports;
    // ... tables, memories, globals, exports, etc.

    u32                     numGlobals;
    M3Global *              globals;

    u32                     numElementSegments;
    // ... element segments, data segments, start function, etc.

    struct M3Module *       next;               // linked list in runtime
}
M3Module;
```

**Module** 对应一个解析后的 .wasm 文件。注意 `functions` 数组包含了导入函数和模块定义函数，导入函数排在前面（索引 0 ~ numFuncImports-1），定义函数紧随其后。

**类比 CLR**：类似于 CLR 的 `Module` / `Assembly`，但 Wasm Module 更扁平——没有命名空间、类层次结构，所有函数都在同一个平面索引空间中。

### 2.4 M3Function

> **源码位置**：[m3_function.h#L44](wasm3/source/m3_function.h#L44) — 结构定义；[m3_function.h#L17](wasm3/source/m3_function.h#L17) — `M3FuncType` 类型签名结构。

```c
typedef struct M3Function {
    struct M3Module *       module;

    M3ImportInfo            import;             // import module/field names (if imported)

    bytes_t                 wasm;               // pointer into wasm binary (code body)
    bytes_t                 wasmEnd;

    IM3FuncType             funcType;           // function signature

    pc_t                    compiled;           // ★ pointer to compiled M3 code (NULL if not yet compiled)

    u16                     maxStackSlots;      // max stack depth needed
    u16                     numRetSlots;
    u16                     numRetAndArgSlots;

    u16                     numLocals;          // number of local variables
    u16                     numLocalBytes;

    bool                    ownsWasmCode;
    u16                     numNames;
    cstr_t                  names[1];           // debug names
}
M3Function;
```

最关键的字段是 `compiled`——指向编译后的 M3 threaded code 的入口。当 `compiled == NULL` 时表示尚未编译，首次调用时触发编译。

### 2.5 M3Compilation（编译状态）

> **源码位置**：[m3_compile.h#L78](wasm3/source/m3_compile.h#L54) — `M3Compilation` 结构定义；[m3_compile.h#L60](wasm3/source/m3_compile.h#L60) — `M3CompilationScope` 控制流作用域；[m3_compile.h#L126](wasm3/source/m3_compile.h#L126) — `M3OpInfo` 操作码信息结构。

```c
typedef struct M3Compilation {
    IM3Runtime          runtime;
    IM3Module           module;
    IM3Function         function;

    // Wasm bytecode reading position
    bytes_t             wasm;
    bytes_t             wasmEnd;

    // M3 code emission
    IM3CodePage         page;                   // current code page being written to

    // Operand stack simulation (compile-time) — uses parallel arrays, NOT a struct
    u16                 stackIndex;             // current stack depth
    u16                 slotFirstDynamicIndex;
    u16                 wasmStack  [d_m3MaxFunctionStackHeight]; // each stack position → slot number (real slot or register alias)
    u8                  typeStack  [d_m3MaxFunctionStackHeight]; // each stack position → value type (i32/i64/f32/f64)
    u8                  m3Slots    [d_m3MaxFunctionSlots];       // each slot → allocation refcount

    // Control flow
    u16                 blockDepth;
    M3CompilationScope  block;                  // current block scope

    // Register allocation
    u16                 regStackIndexPlusOne[2]; // [0]=i64 reg, [1]=f64 reg; 0=unallocated

    // ... various flags and state
}
M3Compilation;
```

这是编译过程中的临时状态，**在 Runtime 中被复用**（不是每次编译都分配新的）。它维护了编译时的类型栈模拟、寄存器分配状态、以及当前的代码发射位置。

---

## 3. Wasm 二进制解析（m3_parse.c）

### 3.1 解析入口

> **源码位置**：[m3_parse.c - m3_ParseModule()](wasm3/source/m3_parse.c#L612) — 解析入口；[m3_parse.c - ParseModuleSection()](wasm3/source/m3_parse.c#L570) — Section 分发表。

```c
M3Result m3_ParseModule(IM3Environment env, IM3Module* o_module,
                        cbytes_t wasmBytes, u32 numWasmBytes);
```

解析流程非常直接：

```
1. Verify magic number (0x00 0x61 0x73 0x6D) and version (0x01)
2. Loop through sections:
   - Read section id (1 byte)
   - Read section size (LEB128)
   - Dispatch to section-specific parser:
     Section 1  → ParseType_Section()      // function signatures
     Section 2  → ParseImport_Section()    // imports
     Section 3  → ParseFunction_Section()  // function-to-type mapping
     Section 4  → ParseTable_Section()     // tables
     Section 5  → ParseMemory_Section()    // memories
     Section 6  → ParseGlobal_Section()    // globals
     Section 7  → ParseExport_Section()    // exports
     Section 8  → ParseStart_Section()     // start function
     Section 9  → ParseElement_Section()   // element segments
     Section 10 → ParseCode_Section()      // function bodies
     Section 11 → ParseData_Section()      // data segments
     Section 12 → ParseDataCount_Section() // data count (bulk memory)
     Section 0  → (skip custom sections)
```

### 3.2 关键设计：零拷贝解析

> **源码位置**：[m3_parse.c#L338](wasm3/source/m3_parse.c#L338) — Code Section 解析时只记录 `func->wasm` 和 `func->wasmEnd` 指针，不复制字节码。

wasm3 的解析器采用**零拷贝**策略——它不会复制 Wasm 字节码，而是保留原始字节的指针。`M3Function.wasm` 和 `M3Function.wasmEnd` 直接指向原始 `.wasm` 二进制中该函数体的位置。这意味着：

1. 原始 `.wasm` 字节必须在 Module 的整个生命周期内保持有效
2. 编译时直接从原始字节中读取操作码和立即数
3. 大幅减少内存分配和拷贝开销

**类比 CLR**：CLR 的 PE 文件加载也采用类似的内存映射策略（memory-mapped file），元数据表直接引用映射后的内存地址。wasm3 的零拷贝解析与此思路一致。

### 3.3 Code Section 的特殊处理

Code Section 的解析（`ParseCode_Section`）只做最基本的工作：

```c
// Pseudocode for ParseCode_Section
for each function body:
    read body_size (LEB128)
    read num_local_groups (LEB128)
    for each local group:
        read count (LEB128)
        read type (1 byte)
        accumulate total locals
    // Record the start of the expression (bytecode)
    function->wasm = current_position
    function->wasmEnd = current_position + remaining_body_size
    // Skip over the body — DO NOT parse instructions yet
    skip(remaining_body_size)
```

注意：**指令不在此时解析**。函数体的字节码只是被记录了起止位置，真正的指令解析发生在编译阶段（`m3_compile.c`）。这是延迟编译策略的基础。

---

## 4. M3 编译流程（m3_compile.c）—— 核心创新

### 4.1 什么是 M3 Threaded Code

传统的字节码解释器使用 switch-dispatch 循环：

```c
// Traditional switch-dispatch interpreter
while (running) {
    uint8_t opcode = *pc++;
    switch (opcode) {
        case OP_I32_ADD: {
            int32_t b = pop();
            int32_t a = pop();
            push(a + b);
            break;
        }
        case OP_I32_CONST: {
            int32_t val = read_leb128(&pc);
            push(val);
            break;
        }
        // ... hundreds of cases
    }
}
```

这种方式的性能瓶颈在于：
1. 每条指令都要经过 switch 分发（branch prediction 不友好）
2. 操作数需要从字节流中动态解码（LEB128 等）
3. 栈操作（push/pop）引入额外开销

**wasm3 的 M3 方法**将 Wasm 字节码**预编译**为一个 C 函数指针数组（threaded code）：

```
Wasm bytecode:        [i32.const 42] [i32.const 10] [i32.add] [return]
                              ↓ M3 Compilation ↓
M3 threaded code:     [op_Const32][slot_off][42] [op_Const32][slot_off][10] [op_i32_Add_ss][off1][off2]
                       ↑ fn ptr    ↑ imm   ↑imm  ↑ fn ptr    ↑ imm   ↑imm  ↑ fn ptr       ↑imm  ↑imm
```

每个 M3 operation 是一个 C 函数，执行完自己的逻辑后，直接跳转到下一个 operation：

```c
// Actual M3 operation for i32.add (register + slot variant)
// Source: m3_exec.h — generated by macro (line 206):
//   d_m3CommutativeOpFunc_i (i32, Add, OP_ADD_32)   // ← search this to find it in source
// The macro expands to the following equivalent code:
d_m3Op(i32_Add_rs)
{
    i32 operand = slot (i32);            // right operand (from stack slot)
    OP_ADD_32((_r0), operand, ((i32) _r0)); // _r0 = (i32)((u32)(operand) + (u32)(_r0))
    nextOp ();
}
```

### 4.2 编译入口与流程

> **源码位置**：[m3_compile.c - CompileFunction()](wasm3/source/m3_compile.c#L2841) — 函数编译入口；[m3_compile.c - CompileBlock()](wasm3/source/m3_compile.c#L2656) — 编译一个代码块；[m3_compile.c - CompileBlockStatements()](wasm3/source/m3_compile.c#L2568) — 逐条编译指令。

编译的入口是 `Compile_Function`：

```c
M3Result  Compile_Function  (IM3Function io_function)
{
    IM3Runtime runtime = io_function->module->runtime;
    M3Compilation * c = & runtime->compilation;  // reuse compilation state

    // Initialize compilation context
    c->function = io_function;
    c->wasm = io_function->wasm;
    c->wasmEnd = io_function->wasmEnd;
    c->stackIndex = 0;
    c->blockDepth = 0;
    // ... more initialization

    // Set up function parameters as initial stack entries
    for each parameter:
        push type onto compile-time stack

    // Set up local variables
    for each local:
        push type onto compile-time stack (initialized to zero)

    // Compile the function body (an implicit block)
    result = CompileBlock(c, funcType, c_waOp_block);

    // Record the compiled entry point
    io_function->compiled = entryPC;

    return result;
}
```

### 4.3 逐条指令编译

`CompileBlock` 内部循环读取 Wasm 操作码并分发到对应的编译函数：

```c
M3Result CompileBlock(M3Compilation* c, IM3FuncType blockType, m3opcode_t blockOpcode)
{
    // Push a new control frame
    PushBlock(c, blockType, blockOpcode);

    while (c->wasm < c->wasmEnd)
    {
        m3opcode_t opcode = read_u8(c->wasm++);

        // Handle 0xFC prefix (saturating truncation, bulk memory, etc.)
        if (opcode == 0xFC) {
            opcode = read_leb128(&c->wasm);
            opcode |= 0xFC00;  // encode as extended opcode
        }

        // Look up the compile function for this opcode
        M3OpInfo info = GetOpInfo(opcode);

        // Call the compile function
        result = info.compiler(c, opcode);

        if (opcode == c_waOp_end || opcode == c_waOp_else)
            break;
    }

    // Pop control frame
    PopBlock(c);
    return result;
}
```

### 4.4 操作码到编译函数的映射

> **源码位置**：[m3_compile.c - c_operations](wasm3/source/m3_compile.c#L2260) — 完整的操作码到 `M3OpInfo` 的映射表（包含栈偏移、类型、operation 函数指针、编译器函数）。

wasm3 使用一个静态表 `c_operations[]` 将每个 Wasm 操作码映射到其编译函数：

```c
// Simplified mapping table
const M3OpInfo c_operations[] = {
    [0x00] = { "unreachable",   0, none,   { Compile_Unreachable }},
    [0x01] = { "nop",           0, none,   { Compile_Nop }},
    [0x02] = { "block",         0, none,   { Compile_LoopOrBlock }},
    [0x03] = { "loop",          0, none,   { Compile_LoopOrBlock }},
    [0x04] = { "if",            0, none,   { Compile_If }},
    // ...
    [0x20] = { "local.get",     0, none,   { Compile_GetLocal }},
    [0x21] = { "local.set",     0, none,   { Compile_SetLocal }},
    // ...
    [0x41] = { "i32.const",     0, none,   { Compile_Const_i32 }},
    [0x6A] = { "i32.add",       0, none,   { Compile_Operator_i }},
    // ... etc.
};
```

每个编译函数负责：
1. 读取该指令的立即数（如果有）
2. 更新编译时类型栈
3. 向代码页发射（emit）对应的 M3 operation 函数指针和立即数

### 4.5 代码发射（Code Emission）

> **源码位置**：[m3_compile.c - EmitOp()](wasm3/source/m3_compile.c#L55) — 发射一个 operation 函数指针；[m3_code.c - EmitWord_impl()](wasm3/source/m3_code.c#L95) — 向 code page 写入一个 word；[m3_code.c - GetPagePC()](wasm3/source/m3_code.c#L139) — 获取当前写入位置。

编译函数通过 `EmitOp` 系列函数向代码页写入 M3 operation：

```c
// m3_compile.c#L55 — EmitOp: emit an operation function pointer
static M3_NOINLINE
M3Result  EmitOp  (IM3Compilation o, IM3Operation i_operation)
{
    // ... ensure code page has enough space ...
    EmitWord (o->page, i_operation);    // write fn ptr to code page
}

// m3_compile.c#L84 — EmitConstant32: emit a 32-bit immediate
static M3_NOINLINE
void  EmitConstant32  (IM3Compilation o, const u32 i_immediate)
{
    if (o->page)
        EmitWord32 (o->page, i_immediate);
}

// m3_compile.c#L90 — EmitSlotOffset: emit a slot index as i32
static M3_NOINLINE
void  EmitSlotOffset  (IM3Compilation o, const i32 i_offset)
{
    if (o->page)
        EmitWord32 (o->page, i_offset);
}

// m3_code.c#L95 — EmitWord_impl: the low-level write to code page
void  EmitWord_impl  (IM3CodePage i_page, void * i_word)
{
    i_page->code [i_page->info.lineIndex++] = i_word;
}
```

编译后的代码页内存布局：

```
Code Page Memory:
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬───┐
│ op_func1 │ imm1_a   │ imm1_b   │ op_func2 │ imm2_a   │ op_func3 │...│
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┴───┘
  ↑ fn ptr   ↑ data     ↑ data     ↑ fn ptr   ↑ data     ↑ fn ptr
```

每个 "word" 的大小等于 `sizeof(void*)`（32 位平台 4 字节，64 位平台 8 字节）。

### 4.6 编译时栈模拟与寄存器分配

> **源码位置**：[m3_compile.c - Push()](wasm3/source/m3_compile.c#L542) — 编译时压栈；[m3_compile.c - Pop()](wasm3/source/m3_compile.c#L584) — 编译时出栈；[m3_compile.h - M3Compilation](wasm3/source/m3_compile.h#L54) — `wasmStack`/`typeStack`/`m3Slots`/`regStackIndexPlusOne` 等字段实现栈模拟和寄存器跟踪。

wasm3 在编译时维护一个**模拟栈**（compile-time stack），用**平行数组**跟踪每个栈位置的类型和 slot 位置（并非一个独立的结构体，而是 `M3Compilation` 中的多个数组字段）：

```c
// m3_compile.h — M3Compilation 结构中的栈模拟字段（非独立结构体）
u16  wasmStack  [d_m3MaxFunctionStackHeight];  // wasmStack[i] = slot number for stack position i
                                               //   real slot (< 60000): value in _sp[slot]
                                               //   pseudo slot (>= 60000): value in register (_r0 or _fp0)
u8   typeStack  [d_m3MaxFunctionStackHeight];  // typeStack[i] = value type (c_m3Type_i32/i64/f32/f64)
u8   m3Slots    [d_m3MaxFunctionSlots];        // m3Slots[slot] = allocation refcount for that slot
u16  regStackIndexPlusOne [2];                 // [0]=_r0 (int), [1]=_fp0 (float); 0=unallocated
```

**寄存器优化**是 wasm3 的重要性能技巧。wasm3 模拟了两个"虚拟寄存器"：
- **整数寄存器**（`_r0`）：用于 i32/i64 值
- **浮点寄存器**（`_fp0`）：用于 f32/f64 值

编译器尽量将栈顶值保留在寄存器中，避免不必要的栈读写：

```
Wasm:           i32.const 42    i32.const 10    i32.add
                    ↓               ↓               ↓
Without reg:    push(42)        push(10)        b=pop; a=pop; push(a+b)
                    ↓               ↓               ↓
With reg:       _r0 = 42        spill _r0→slot  _r0 = slot + _r0
                                _r0 = 10
```

当寄存器被占用时，编译器会发射一个 "spill" 操作将寄存器值写回栈槽，然后复用寄存器。这种优化减少了约 30-40% 的栈操作。

#### 4.6.1 栈槽（Slot）与寄存器的数据结构

> **源码位置**：[m3_compile.h#L78](wasm3/source/m3_compile.h#L78) — `M3Compilation` 结构中的 `wasmStack`/`typeStack`/`m3Slots`/`regStackIndexPlusOne`；[m3_core.h#L162](wasm3/source/m3_core.h#L162) — `d_m3Reg0SlotAlias` / `d_m3Fp0SlotAlias` 定义。

编译期虚拟栈的核心字段（[m3_compile.h#L95-L115](wasm3/source/m3_compile.h#L95)）：

```c
typedef struct M3Compilation {
    // ...
    u16  stackIndex;                                    // current stack top index

    u16  wasmStack  [d_m3MaxFunctionStackHeight];       // each stack position → slot number
    u8   typeStack  [d_m3MaxFunctionStackHeight];       // each stack position → value type

    u8   m3Slots    [d_m3MaxFunctionSlots];             // each slot → allocation refcount

    u16  regStackIndexPlusOne [2];                      // [0]=_r0 (int), [1]=_fp0 (float)
                                                        // stores (stackIndex+1) that occupies the register; 0 = unallocated
    // ...
} M3Compilation;
```

**`wasmStack[i]` 存的不是值本身，而是一个 slot 编号（u16）**，表示"第 i 个栈元素的值存放在哪里"。slot 编号有两种含义：

- **真实 slot**（< `d_m3MaxFunctionSlots`）：值在运行时栈 `_sp[slot]` 中
- **伪 slot**（>= 60000）：值在 CPU 寄存器中

伪 slot 的定义（[m3_core.h#L162-163](wasm3/source/m3_core.h#L162)）：

```c
#define d_m3Reg0SlotAlias    60000       // integer register _r0
#define d_m3Fp0SlotAlias     (60000 + 2) // float register _fp0
```

判断栈顶值是否在寄存器中（[m3_compile.c#L263](wasm3/source/m3_compile.c#L263)）：

```c
bool IsStackIndexInRegister(IM3Compilation o, i32 i_stackIndex)
{
    if (i_stackIndex >= 0 and i_stackIndex < o->stackIndex)
        return (o->wasmStack[i_stackIndex] >= d_m3Reg0SlotAlias);  // >= 60000 → in register
    else
        return false;
}
```

运行时栈 `_sp` 的布局：

```
_sp array layout:
┌──────────┬──────────┬──────────────┬──────────────────┐
│  args    │  locals  │  constants   │  dynamic slots   │
│ (参数)   │ (局部变量)│ (编译期常量) │ (临时值/中间结果) │
└──────────┴──────────┴──────────────┴──────────────────┘
     ↑           ↑            ↑              ↑
slotFirstLocalIndex  slotFirstConstIndex  slotFirstDynamicIndex
```

#### 4.6.2 PushRegister：标记栈顶值在寄存器中

> **源码位置**：[m3_compile.c#L573 - PushRegister()](wasm3/source/m3_compile.c#L573) — 将寄存器伪 slot 压入编译期栈；[m3_compile.c#L542 - Push()](wasm3/source/m3_compile.c#L542) — 底层压栈操作。

`PushRegister` 的实现非常简单——它调用 `Push` 并传入寄存器的伪 slot 编号：

```c
// m3_compile.c#L573
static inline
M3Result  PushRegister  (IM3Compilation o, u8 i_type)
{
    // d_m3Reg0SlotAlias (60000) is guaranteed > d_m3MaxFunctionSlots, so it's distinguishable
    u16 slot = IsFpType (i_type) ? d_m3Fp0SlotAlias : d_m3Reg0SlotAlias;
    return Push (o, i_type, slot);
}

// m3_compile.c#L542 — Push: the low-level stack push
static
M3Result  Push  (IM3Compilation o, u8 i_type, u16 i_slot)
{
    u16 stackIndex = o->stackIndex++;
    o->wasmStack [stackIndex] = i_slot;      // record slot number (real or pseudo)
    o->typeStack [stackIndex] = i_type;      // record type

    if (IsRegisterSlotAlias (i_slot))        // if pseudo slot (>= 60000)
    {
        u32 regSelect = IsFpRegisterSlotAlias (i_slot);  // 0=int, 1=fp
        AllocateRegister (o, regSelect, stackIndex);     // mark register as occupied
    }
    return m3Err_none;
}
```

与之对比，当值在栈槽中时，直接传入真实 slot 编号：

```c
// Compile_GetLocal (m3_compile.c#L1322) — value stays in its slot, no register involved
static M3Result  Compile_GetLocal  (IM3Compilation o, m3opcode_t i_opcode)
{
    u32 localIndex;
    ReadLEB_u32 (& localIndex, & o->wasm, o->wasmEnd);

    u8 type = GetStackTypeFromBottom (o, localIndex);
    u16 slot = GetSlotForStackIndex (o, localIndex);    // real slot number (e.g. 0, 1, 2...)

    Push (o, type, slot);                               // wasmStack[top] = real slot, NOT register
}
```

#### 4.6.3 PushRegister 的三种典型调用场景

**场景 1：算术运算结果留在寄存器**

> **源码位置**：[m3_compile.c#L2092 - Compile_Operator()](wasm3/source/m3_compile.c#L2092)

这是最常见的场景。几乎所有算术/比较运算（`i32.add`、`i32.mul`、`i32.lt_s` 等）的结果都通过 `PushRegister` 标记为在寄存器中，因为运行时 op 函数会把结果写入 `_r0`：

```c
// m3_compile.c#L2092 — Compile_Operator (simplified, showing key logic)
static M3Result  Compile_Operator  (IM3Compilation o, m3opcode_t i_opcode)
{
    IM3OpInfo opInfo = GetOpInfo (i_opcode);
    IM3Operation op;

    // Select operation variant based on operand locations:
    if (IsStackTopInRegister (o))
        op = opInfo->operations [0];    // _rs: top in register, second in slot
    else if (IsStackTopMinus1InRegister (o))
        op = opInfo->operations [1];    // _sr: top in slot, second in register
    else {
        PreserveRegisterIfOccupied (o, opInfo->type);  // spill register first
        op = opInfo->operations [2];    // _ss: both in slots
    }

    EmitOp (o, op);
    EmitSlotNumOfStackTopAndPop (o);        // emit & pop operand(s)
    if (opInfo->stackOffset < 0)
        EmitSlotNumOfStackTopAndPop (o);

    // ★ Result is always in register (_r0 or _fp0)
    if (opInfo->type != c_m3Type_none)
        PushRegister (o, opInfo->type);     // wasmStack[top] = 60000 (d_m3Reg0SlotAlias)
}
```

对应的运行时 op（以 `i32.add` 的 `_rs` 变体为例）确实把结果写入 `_r0`：

```c
d_m3Op(i32_Add_rs)
{
    i32 operand = slot (i32);
    OP_ADD_32((_r0), operand, ((i32) _r0));  // result → _r0
    nextOp ();
}
```

**场景 2：`memory.size` / `memory.grow` 结果进寄存器**

> **源码位置**：[m3_compile.c#L1738 - Compile_Memory_Size()](wasm3/source/m3_compile.c#L1738)；[m3_compile.c#L1753 - Compile_Memory_Grow()](wasm3/source/m3_compile.c#L1753)

这些指令的结果（页数）也通过 `PushRegister` 标记为在寄存器中：

```c
// m3_compile.c#L1738
static M3Result  Compile_Memory_Size  (IM3Compilation o, m3opcode_t i_opcode)
{
    // ★ Must spill register first if occupied, because op_MemSize will write to _r0
    PreserveRegisterIfOccupied (o, c_m3Type_i32);

    EmitOp (o, op_MemSize);

    // ★ Result is in _r0
    PushRegister (o, c_m3Type_i32);
}
```

**场景 3：`CopyStackTopToRegister` — 将栈槽值搬到寄存器**

> **源码位置**：[m3_compile.c#L904 - CopyStackTopToRegister()](wasm3/source/m3_compile.c#L904)

当编译器需要将一个在栈槽中的值搬到寄存器时（例如 `br_if` 需要条件值在寄存器中），会调用此函数：

```c
// m3_compile.c#L904
static M3Result  CopyStackTopToRegister  (IM3Compilation o, bool i_updateStack)
{
    if (IsStackTopInSlot (o))                           // only if NOT already in register
    {
        u8 type = GetStackTopType (o);

        PreserveRegisterIfOccupied (o, type);           // spill old register value if any

        IM3Operation op = c_setRegisterOps [type];      // → op_SetRegister_i32 / _i64 / _f32 / _f64
        EmitOp (o, op);
        EmitSlotOffset (o, GetStackTopSlotNumber (o));  // emit source slot offset

        if (i_updateStack)
        {
            PopType (o, type);
            PushRegister (o, type);                     // ★ now stack top is in register
        }
    }
}
```

#### 4.6.4 Spill（溢出）：PreserveRegisterIfOccupied

> **源码位置**：[m3_compile.c#L474 - PreserveRegisterIfOccupied()](wasm3/source/m3_compile.c#L474)

当编译器需要使用寄存器但寄存器已被占用时，必须先将旧值"溢出"到栈槽：

```c
// m3_compile.c#L474
static M3Result  PreserveRegisterIfOccupied  (IM3Compilation o, u8 i_registerType)
{
    u32 regSelect = IsFpType (i_registerType);

    if (IsRegisterAllocated (o, regSelect))             // register is occupied?
    {
        u16 stackIndex = GetRegisterStackIndex (o, regSelect);
        DeallocateRegister (o, regSelect);              // mark register as free

        u8 type = GetStackTypeFromBottom (o, stackIndex);

        u16 slot = c_slotUnused;
        AllocateSlots (o, & slot, type);                // allocate a real stack slot
        o->wasmStack [stackIndex] = slot;               // ★ update: was 60000 (register), now real slot

        EmitOp (o, c_setSetOps [type]);                 // emit op_SetSlot_i32 / _i64 / _f32 / _f64
        EmitSlotOffset (o, slot);                       // emit target slot offset
    }
}
```

**关键操作**：`o->wasmStack[stackIndex] = slot` — 将编译期栈中该位置的记录从伪 slot（60000）改为真实 slot 编号，表示值已从寄存器溢出到栈内存。

#### 4.6.5 完整流程示例

以 `i32.const 42  i32.const 10  i32.add` 为例，追踪编译期栈和寄存器的变化：

```
Step 1: i32.const 42
  → PushConst(o, 42, i32)
  → 42 written to constants table, Push(o, i32, constSlot3)
  → wasmStack: [..., slot3]     register: _r0=free
  → (value is in a constant slot, not in register)

Step 2: i32.const 10
  → PushConst(o, 10, i32)
  → 10 written to constants table, Push(o, i32, constSlot4)
  → wasmStack: [..., slot3, slot4]     register: _r0=free

Step 3: i32.add
  → Compile_Operator(o, 0x6A)
  → Both operands in slots → PreserveRegisterIfOccupied (no-op, _r0 is free)
  → Select op = op_i32_Add_ss (both in slots)
  → EmitOp(op_i32_Add_ss), EmitSlotOffset(slot4), EmitSlotOffset(slot3)
  → Pop both operands
  → PushRegister(o, i32)                    ★ result marked as in _r0
  → wasmStack: [..., 60000]     register: _r0=occupied (stackIndex=top)

If another i32.const follows:
  → PushConst(o, 99, i32)
  → 99 is a constant, Push(o, i32, constSlot5)
  → wasmStack: [..., 60000, slot5]     register: _r0=still occupied
  → (no spill needed — the new const goes to a slot, not to register)

If another i32.add follows:
  → Compile_Operator: top=slot5 (in slot), top-1=60000 (in register)
  → Select op = op_i32_Add_sr (slot + register)
  → EmitOp(op_i32_Add_sr), EmitSlotOffset(slot5)
  → Pop both, PushRegister(o, i32)          ★ result in _r0 again
```

#### 4.6.6 设计思路总结

栈槽与寄存器优化的核心设计思路：

1. **统一的 slot 编号空间**：用 `wasmStack[]` 数组统一追踪所有值的位置，真实 slot（< 60000）和寄存器伪 slot（>= 60000）共用同一套 Push/Pop 机制，代码简洁
2. **编译期决策，运行时零开销**：编译器在编译时就确定每个值在寄存器还是栈槽中，并选择对应的 op 变体（`_rs`/`_sr`/`_ss`），运行时无需任何判断
3. **惰性溢出（Lazy Spill）**：只有当寄存器确实需要被复用时才溢出（`PreserveRegisterIfOccupied`），而不是每次都写回栈槽
4. **每种 op 多变体**：同一个 Wasm 操作码根据操作数位置生成不同的 M3 op（如 `op_i32_Add_rs` vs `op_i32_Add_ss`），每个变体的代码路径最短，无运行时分支
5. **只有两个寄存器**：刻意限制为 `_r0`（整数）和 `_fp0`（浮点）各一个，因为它们作为函数参数传递，在大多数 ABI 中会被分配到真实 CPU 寄存器，同时保持分配逻辑极其简单

### 4.7 控制流编译

> **源码位置**：[m3_compile.c - Compile_LoopOrBlock()](wasm3/source/m3_compile.c#L1851) — block/loop 编译；[m3_compile.c - Compile_If()](wasm3/source/m3_compile.c#L1927) — if/else 编译；[m3_compile.c - Compile_Branch()](wasm3/source/m3_compile.c#L1424) — br/br_if 编译。

控制流是编译中最复杂的部分。以 `block` 为例：

```c
M3Result Compile_LoopOrBlock(M3Compilation* c, m3opcode_t opcode)
{
    // Read block type
    i64 blockType = ReadLebSigned(&c->wasm);

    // For 'block': the branch target is the END of the block
    // For 'loop':  the branch target is the START of the block (loop header)

    if (opcode == c_waOp_loop) {
        // Record current PC as the loop header (branch target)
        pc_t loopPC = GetPagePC(c->page);
        result = CompileBlock(c, type, opcode);
        // Loop's branch target was already set to loopPC
    } else {
        // Block: compile body, then patch forward branches
        result = CompileBlock(c, type, opcode);
        // Patch all forward branch targets to current PC (after END)
    }
    return result;
}
```

**前向分支修补（Forward Branch Patching）**：

对于 `block` 和 `if`，`br` 指令在编译时还不知道跳转目标地址（因为 END 还没编译到）。wasm3 使用经典的**回填（backpatching）**技术：

```
1. Compile 'br 0' → emit [op_branch] [placeholder]
2. Record the placeholder's address in the control frame
3. Continue compiling...
4. When reaching 'end' → patch all recorded placeholders with current PC
```

对于 `loop`，`br` 跳转到循环头部，此时地址已知，无需回填。

### 4.8 编译示例：完整函数

以一个简单的加法函数为例：

```wat
(func $add (param i32 i32) (result i32)
  local.get 0
  local.get 1
  i32.add
)
```

编译过程：

```
Step 1: Setup
  - Compile-time stack: [slot0:i32(param0), slot1:i32(param1)]
  - Register: empty

Step 2: local.get 0
  - Emit: [op_SetRegister_i32] [slot_offset_0]
  - Register: _r0 = slot0
  - Stack: [slot0, slot1] (slot0 now also in _r0)

Step 3: local.get 1
  - _r0 is occupied → spill? No, emit a "slot + register" variant
  - Emit: nothing yet (operand tracked as slot reference)

Step 4: i32.add
  - Left operand in _r0, right operand in slot1
  - Emit: [op_i32_Add_rs] [slot_offset_1]   // rs = register + slot
  - Result in _r0

Step 5: end (implicit return)
  - Return value is in _r0 (matches calling convention)
  - No explicit op needed; function epilogue handles return

Final M3 code: [op_SetRegister_i32][off0][op_i32_Add_rs][off1]
```

注意 wasm3 为同一个 Wasm 操作码生成**不同变体**的 M3 operation，取决于操作数的位置：
- `op_i32_Add_ss`：两个操作数都在栈槽（slot + slot）
- `op_i32_Add_rs`：一个在寄存器，一个在栈槽（register + slot）
- `op_i32_Add_sr`：一个在栈槽，一个在寄存器（slot + register）

这种变体化进一步减少了运行时的数据移动。

---

## 5. M3 执行引擎（m3_exec.c / m3_exec.h）

### 5.1 Operation 的定义

> **源码位置**：[m3_exec_defs.h#L19 - d_m3BaseOpSig](wasm3/source/m3_exec_defs.h#L19) — 基础签名宏（4 参数）；[m3_exec_defs.h#L33 - d_m3OpSig](wasm3/source/m3_exec_defs.h#L33) — 最终签名宏（条件编译，含浮点时 5 参数）；[m3_exec_defs.h#L55 - d_m3Op](wasm3/source/m3_exec_defs.h#L55) — 函数定义宏；[m3_exec.h](wasm3/source/m3_exec.h) — 所有 operation 实现。

每个 M3 operation 都是一个 C 函数，使用 `d_m3Op` 宏定义：

```c
// m3_exec_defs.h — macro chain:
# define d_m3BaseOpSig      pc_t _pc, m3stack_t _sp, M3MemoryHeader * _mem, m3reg_t _r0
# define d_m3ExpOpSig(...)   d_m3BaseOpSig, __VA_ARGS__

# if d_m3HasFloat
#   define d_m3OpSig         d_m3ExpOpSig(f64 _fp0)   // 5 params (with float register)
# else
#   define d_m3OpSig         d_m3BaseOpSig             // 4 params (no float)
# endif

#define d_m3RetSig           static inline m3ret_t vectorcall
#define d_m3Op(NAME)         M3_NO_UBSAN d_m3RetSig op_##NAME (d_m3OpSig)
```

以 `i32.add` 的 register+slot 变体为例，它由宏 `d_m3CommutativeOpFunc_i(i32, Add, OP_ADD_32)` 展开生成（[m3_exec.h#L207](wasm3/source/m3_exec.h#L207)）：

```c
// d_m3CommutativeOpMacro expands to (search "d_m3CommutativeOpFunc_i (i32, Add" in source):
d_m3Op(i32_Add_rs)                      // → static inline m3ret_t vectorcall op_i32_Add_rs(d_m3OpSig)
{
    i32 operand = slot (i32);           // read operand from stack slot
    OP_ADD_32((_r0), operand, ((i32) _r0));  // _r0 = (i32)((u32)(operand) + (u32)(_r0))
    nextOp ();
}
```

每个 operation 函数的签名展开后（`d_m3HasFloat = true` 时）为：

```c
// m3ret_t = const void*; pc_t = code_t const*; m3stack_t = m3slot_t*; m3reg_t = i64
static inline m3ret_t vectorcall op_##NAME (
    pc_t            _pc,    // program counter (points to next operation in code stream)
    m3stack_t       _sp,    // stack pointer
    M3MemoryHeader* _mem,   // linear memory header (data follows immediately after)
    m3reg_t         _r0,    // integer register (i64)
    f64             _fp0    // float register (only when d_m3HasFloat is true)
);
```

### 5.2 分发机制：nextOp()

> **源码位置**：[m3_exec_defs.h#L62](wasm3/source/m3_exec_defs.h#L62) — `nextOpDirect`/`nextOpImpl` 宏定义；[m3_exec_defs.h - RunCode()](wasm3/source/m3_exec_defs.h#L66) — trampoline 入口。

`nextOp()` 是 M3 执行引擎的核心——它从代码流中读取下一个函数指针并跳转过去：

```c
// m3_exec_defs.h — dispatch macros:
#define nextOpImpl()     ((IM3Operation)(* _pc))(_pc + 1, d_m3OpArgs)
//                                                       ↑ expands to: _sp, _mem, _r0 [, _fp0]
#define nextOpDirect()   M3_MUSTTAIL return nextOpImpl()

// m3_exec.h — the macro used in operations:
#define nextOp()         nextOpDirect()   // when profiling/tracing is off
```

**关键性能洞察**：

1. **无 switch 开销**：没有中央分发循环，每个 operation 直接跳转到下一个
2. **尾调用优化（Tail Call Optimization）**：如果编译器将 `nextOp()` 优化为尾调用，则不会消耗调用栈空间，整个执行链变成一个扁平的跳转序列
3. **参数通过寄存器传递**：`_r0` 和 `_fp0` 作为函数参数，在支持的 ABI 中会被分配到 CPU 寄存器，避免内存访问

```
Execution flow:

op_SetRegister_i32 → op_i32_Add_rs → op_SetSlot_i32 → op_Branch → op_Const32 → ...
        │                  │                │              │            │
        └── nextOp() ──────┘── nextOp() ────┘── nextOp() ──┘── nextOp() ┘
            (tail call)       (tail call)      (tail call)    (tail call)
```

### 5.3 立即数读取

M3 operation 通过 `immediate()` 宏从代码流中读取预编译的立即数：

```c
// m3_exec.h#L41
# define immediate(TYPE)    * ((TYPE *) _pc++)

// Usage: op_Const32 writes a constant into a stack slot (m3_exec.h#L1241)
d_m3Op  (Const32)
{
    u32 value = * (u32 *)_pc++;
    slot (u32) = value;
    nextOp ();
}
```

由于立即数在编译时已经从 LEB128 解码为原生整数，运行时无需任何解码开销。

### 5.4 栈操作

运行时栈是一个 `u64` 数组（每个槽 8 字节），通过偏移量访问：

```c
// m3_exec.h#L41-L45
# define immediate(TYPE)    * ((TYPE *) _pc++)
# define slot(TYPE)         * (TYPE *) (_sp + immediate (i32))
# define slot_ptr(TYPE)     (TYPE *) (_sp + immediate (i32))

// local.set is compiled to SetSlot or CopySlot ops (not a dedicated "SetLocal" op).
// SetSlot_i32 is generated by macro (m3_exec.h#L944):
//   #define d_m3SetRegisterSetSlot(TYPE, REG) ...
//   d_m3SetRegisterSetSlot (i32, _r0)          // ← line 968, search this to find it
// The macro expands to the following equivalent code:
d_m3Op (SetSlot_i32)
{
    slot (i32) = (i32) _r0;     // *(i32*)(_sp + *(i32*)_pc++) = (i32)_r0
    nextOp ();
}
// When the value is in another slot, the compiler emits CopySlot_32 (m3_exec.h#L975):
d_m3Op (CopySlot_32)
{
    u32 * dst = slot_ptr (u32);
    u32 * src = slot_ptr (u32);
    * dst = * src;
    nextOp ();
}
```

### 5.5 内存访问操作

> **源码位置**：[m3_exec.h#L1321 - d_m3Load](wasm3/source/m3_exec.h#L1321) — `d_m3Load` 宏（生成所有 load 变体的 `_r` 和 `_s` 版本）；[m3_exec.h#L1393](wasm3/source/m3_exec.h#L1393) — `d_m3Store` 宏（生成所有 store 变体）；[m3_exec.h#L696 - op_MemSize()](wasm3/source/m3_exec.h#L696) — `memory.size`；[m3_exec.h#L706 - op_MemGrow()](wasm3/source/m3_exec.h#L706) — `memory.grow`。

线性内存操作需要进行边界检查。Load 操作由 `d_m3Load` 宏统一生成，例如 `d_m3Load_i(i32, i32)` 展开后生成 `op_i32_Load_i32_r`（地址在寄存器）和 `op_i32_Load_i32_s`（地址在栈槽）两个变体。以 `_r` 变体为例（[m3_exec.h#L1321](wasm3/source/m3_exec.h#L1321)）：

```c
// d_m3Load macro (simplified, showing the _r variant):
d_m3Op(i32_Load_i32_r)
{
    u32 offset = immediate (u32);               // static offset from code stream
    u64 operand = (u32) _r0;                    // base address from register
    operand += offset;                          // effective address

    if (m3MemCheck(                             // bounds check (M3_LIKELY wrapper)
        operand + sizeof (i32) <= _mem->length
    )) {
        u8* src8 = m3MemData(_mem) + operand;   // m3MemData = (u8*)(header+1)
        i32 value;
        memcpy(&value, src8, sizeof(value));     // memcpy for unaligned access support
        _r0 = (i32)value;
        nextOp ();
    } else d_outOfBounds;                       // → newTrap(m3Err_trapOutOfBoundsMemoryAccess)
}
```

`m3MemCheck` 内部使用 `M3_LIKELY` 宏提示编译器边界检查通过是常见路径，优化分支预测。使用 `memcpy` 而非直接指针解引用是为了支持非对齐内存访问的平台。

### 5.6 函数调用

> **源码位置**：[m3_exec.h - op_Call()](wasm3/source/m3_exec.h#L542) — 直接调用；[m3_exec.h - op_CallIndirect()](wasm3/source/m3_exec.h#L568) — 间接调用；[m3_exec.h - op_CallRawFunction()](wasm3/source/m3_exec.h#L623) — 宿主函数调用；[m3_exec.h - op_Entry()](wasm3/source/m3_exec.h#L802) — 函数入口（分配局部变量、检查栈溢出）；[m3_exec.h - op_Return()](wasm3/source/m3_exec.h#L1154) — 函数返回。

函数调用是最复杂的 operation 之一：

```c
// m3_exec.h#L542 — actual source:
d_m3Op  (Call)
{
    pc_t callPC                 = immediate (pc_t);     // target function's compiled code
    i32 stackOffset             = immediate (i32);      // stack frame offset
    IM3Memory memory            = m3MemInfo (_mem);      // save memory info (for realloc detection)

    m3stack_t sp = _sp + stackOffset;                    // callee's stack pointer

    m3ret_t r = Call (callPC, sp, _mem, d_m3OpDefaultArgs);  // enter callee via Call trampoline

    _mem = memory->mallocated;                           // reload _mem (may have changed via memory.grow)

    if (M3_LIKELY(not r))
        nextOp ();                                       // callee returned normally
    else
    {
        pushBacktraceFrame ();
        forwardTrap (r);                                 // propagate trap
    }
}
```

注意 `Call` 是一个 trampoline 函数（[m3_exec.h#L113](wasm3/source/m3_exec.h#L113)），它调用 `m3_Yield()` 检查后直接 `nextOpDirect()` 进入被调用函数。调用返回后需要重新加载 `_mem`，因为被调用函数可能执行了 `memory.grow` 导致内存基址变化。

**间接调用（call_indirect）** 额外需要运行时类型检查（[m3_exec.h#L568](wasm3/source/m3_exec.h#L568)）：

```c
d_m3Op  (CallIndirect)
{
    u32 tableIndex              = slot (u32);            // index from stack slot (not register)
    IM3Module module            = immediate (IM3Module);
    IM3FuncType type            = immediate (IM3FuncType);
    i32 stackOffset             = immediate (i32);
    IM3Memory memory            = m3MemInfo (_mem);

    m3stack_t sp = _sp + stackOffset;
    m3ret_t r = m3Err_none;

    if (M3_LIKELY(tableIndex < module->table0Size))
    {
        IM3Function function = module->table0 [tableIndex];

        if (M3_LIKELY(function))
        {
            if (M3_LIKELY(type == function->funcType))   // type signature check
            {
                if (M3_UNLIKELY(not function->compiled)) // lazy compile if needed
                    r = CompileFunction (function);

                if (M3_LIKELY(not r))
                {
                    r = Call (function->compiled, sp, _mem, d_m3OpDefaultArgs);
                    _mem = memory->mallocated;

                    if (M3_LIKELY(not r))
                        nextOpDirect ();
                    else { pushBacktraceFrame (); forwardTrap (r); }
                }
            }
            else r = m3Err_trapIndirectCallTypeMismatch;
        }
        else r = m3Err_trapTableElementIsNull;
    }
    else r = m3Err_trapTableIndexOutOfRange;

    if (M3_UNLIKELY(r))
        newTrap (r);
    else forwardTrap (r);
}
```

### 5.7 Trap 处理

> **源码位置**：[wasm3.h#L107](wasm3/source/wasm3.h#L107) — 所有错误码/trap 字符串定义（`m3Err_*`）；[m3_exec.h - op_Compile()](wasm3/source/m3_exec.h#L778) — 懒编译入口（首次调用时触发编译）。

wasm3 使用 C 的返回值来传播 trap（而非 setjmp/longjmp）：

```c
// Normal execution: nextOp() — continues the chain (M3_MUSTTAIL return)
// Trap: newTrap(err) — returns error pointer, unwinds the chain

// i32.div_s is generated by: d_m3OpMacro_i(i32, Divide, OP_DIV_S, INT32_MIN)
// which expands d_m3OpMacro → generates _sr, _rs, _ss variants.
// The actual trap logic is in the OP_DIV_S macro (m3_math_utils.h#L167):
#define OP_DIV_S(RES, A, B, TYPE_MIN)                         \
    if (M3_UNLIKELY(B == 0)) newTrap (m3Err_trapDivisionByZero); \
    if (M3_UNLIKELY(B == -1 and A == TYPE_MIN)) {                \
        newTrap (m3Err_trapIntegerOverflow);                  \
    }                                                         \
    RES = A / B;

// So op_i32_Divide_sr (for example) expands to roughly:
d_m3Op(i32_Divide_sr)
{
    i32 operand = slot (i32);                    // divisor from stack slot
    OP_DIV_S((_r0), ((i32) _r0), operand, INT32_MIN);
    nextOp ();
}
```

当 trap 发生时，错误指针沿着调用链逐层返回，最终到达 `m3_Call()` 的调用者。这比 setjmp/longjmp 更可移植，且在正常路径上零开销。

---

## 6. 代码页管理（m3_code.c）

### 6.1 代码页结构

> **源码位置**：[m3_core.h - M3CodePageHeader](wasm3/source/m3_core.h#L142) — 页头结构定义；[m3_code.c - NewCodePage()](wasm3/source/m3_code.c#L15) — 创建新页；[m3_code.c - EmitWord_impl()](wasm3/source/m3_code.c#L95) — 写入一个 word；[m3_env.c - AcquireCodePageWithCapacity()](wasm3/source/m3_env.c#L1113) — 获取有剩余容量的页。

M3 编译后的 threaded code 存储在**代码页（Code Page）**中：

```c
typedef struct M3CodePage {
    M3CodePageHeader        info;
    m3word_t                code[c_m3CodePageFreeLinesCount]; // actual code storage
}
M3CodePage;

typedef struct M3CodePageHeader {
    struct M3CodePage *     next;           // linked list
    u32                     lineIndex;      // next free write position
    u32                     numLines;       // total capacity
    u32                     sequence;       // page sequence number
    u32                     usageCount;     // reference count
}
M3CodePageHeader;
```

### 6.2 页面管理策略

```
Runtime
  ├── pagesOpen  → [Page A (has free space)] → [Page B] → ...
  └── pagesFull  → [Page C (full)] → [Page D] → ...
```

- 编译时从 `pagesOpen` 链表获取有空闲空间的页面
- 页面写满后移到 `pagesFull` 链表
- 页面大小通常为 ~4KB（`c_m3CodePageFreeLinesCount` 个 word）
- 如果所有页面都满了，分配新页面

这种设计避免了频繁的小内存分配，同时保持了代码的局部性（cache-friendly）。

---

## 7. 宿主函数绑定（m3_bind.c）

### 7.1 绑定机制

> **源码位置**：[m3_bind.c - m3_LinkRawFunction()](wasm3/source/m3_bind.c#L167) — 绑定宿主函数的主要 API；[m3_bind.c - SignatureToFuncType()](wasm3/source/m3_bind.c#L27) — 解析签名字符串；[m3_bind.c - FindAndLinkFunction()](wasm3/source/m3_bind.c#L123) — 查找并链接导入函数。

wasm3 允许将 C 函数绑定为 Wasm 的导入函数：

```c
// Public API
M3Result m3_LinkRawFunction(IM3Module module,
                            const char* moduleName,
                            const char* functionName,
                            const char* signature,
                            M3RawCall function);

// Raw call signature
typedef const void* (* M3RawCall)(
    IM3Runtime runtime,
    IM3ImportContext ctx,
    uint64_t* _sp,      // stack pointer (args and return values)
    void* _mem           // linear memory base
);
```

宿主函数通过栈指针直接读写参数和返回值：

```c
// Example: binding a host function
m3ApiRawFunction(my_add) {
    m3ApiReturnType(int32_t);
    m3ApiGetArg(int32_t, a);
    m3ApiGetArg(int32_t, b);
    m3ApiReturn(a + b);
}

// m3ApiGetArg expands to (wasm3.h#L340):
// int32_t a = * ((int32_t *) (_sp++));
```

### 7.2 绑定的编译处理

当编译器遇到对导入函数的调用时，会发射一个特殊的 `op_CallRawFunction` operation：

```c
// m3_exec.h#L623 — actual source (simplified, omitting strace):
d_m3Op  (CallRawFunction)
{
    M3ImportContext ctx;

    M3RawCall call = (M3RawCall) (* _pc++);      // host function pointer
    ctx.function   = immediate (IM3Function);
    ctx.userdata   = immediate (void *);
    u64* const sp  = ((u64*)_sp);
    IM3Memory memory = m3MemInfo (_mem);
    IM3Runtime runtime = m3MemRuntime(_mem);

    // Reconfigure stack to enable re-entrant m3_Call from host function
    void* stack_backup = runtime->stack;
    runtime->stack = sp;
    m3ret_t possible_trap = call (runtime, &ctx, sp, m3MemData(_mem));
    runtime->stack = stack_backup;

    if (M3_UNLIKELY(possible_trap)) {
        _mem = memory->mallocated;
        pushBacktraceFrame ();
    }
    forwardTrap (possible_trap);
}
```

---

## 8. 内存管理分析

### 8.1 统一栈设计

> **源码位置**：[m3_env.h - M3Runtime](wasm3/source/m3_env.h#L161) — `stack` 和 `stackSize` 字段；[m3_env.c - m3_NewRuntime()](wasm3/source/m3_env.c#L173) — 栈分配逻辑。

wasm3 最独特的内存设计是**统一栈**——值栈和调用栈共享同一块内存：

```
Stack Memory Layout:
┌─────────────────────────────────────────────────────────┐
│ ← Value Stack (grows →)          Call Stack (← grows) → │
│ [slot0][slot1][slot2]...    ...[frame2][frame1][frame0] │
└─────────────────────────────────────────────────────────┘
  ↑ sp (stack pointer)              ↑ call stack pointer
```

值栈从低地址向高地址增长，调用栈从高地址向低地址增长。当两者相遇时，表示栈溢出。

优点：
- 单次分配，无需分别管理两个栈
- 栈大小可以灵活分配给值或调用帧
- 减少内存碎片

### 8.2 线性内存管理

> **源码位置**：[m3_env.h - M3Memory](wasm3/source/m3_env.h#L20) — 内存结构；[m3_core.h - M3MemoryHeader](wasm3/source/m3_core.h#L132) — 内存头部（包含长度、maxStack 等）；[m3_env.c - ResizeMemory()](wasm3/source/m3_env.c#L353) — 内存增长实现。

```c
typedef struct M3Memory {
    M3MemoryHeader *        mallocated;     // allocated memory block
    u32                     numPages;       // current page count (64KB per page)
    u32                     maxPages;       // maximum page count
}
M3Memory;

// m3_core.h#L132 — actual source:
typedef struct M3MemoryHeader
{
    IM3Runtime      runtime;        // back-pointer to owning runtime
    void *          maxStack;       // stack overflow detection sentinel
    size_t          length;         // current byte length
}
M3MemoryHeader;
// Actual memory data follows immediately after this header: m3MemData(mem) = (u8*)(header+1)
```

`memory.grow` 的实现：

```c
M3Result ResizeMemory(IM3Runtime runtime, u32 numPages)
{
    u32 newSize = numPages * c_m3MemPageSize;  // 65536 bytes per page

    // Reallocate the memory block
    M3MemoryHeader* newMem = m3_Realloc(
        "Wasm Linear Memory",
        runtime->memory.mallocated,
        sizeof(M3MemoryHeader) + newSize,
        sizeof(M3MemoryHeader) + oldSize
    );

    if (!newMem)
        return m3Err_mallocFailed;

    // Zero-initialize the new pages
    memset((u8*)newMem + sizeof(M3MemoryHeader) + oldSize, 0, newSize - oldSize);

    newMem->length = newSize;
    runtime->memory.mallocated = newMem;
    runtime->memory.numPages = numPages;

    return m3Err_none;
}
```

**重要注意**：`memory.grow` 可能导致内存基址变化（realloc），因此所有 M3 operation 都通过 `_mem` 参数访问线性内存，而不是缓存基址指针。每次函数调用时 `_mem` 都会被更新。

---

## 9. Import/Export 与模块间交互

> **源码位置**：[m3_parse.c - ParseSection_Import()](wasm3/source/m3_parse.c#L148) — 导入段解析；[m3_parse.c - ParseSection_Export()](wasm3/source/m3_parse.c#L230) — 导出段解析；[m3_bind.c - FindAndLinkFunction()](wasm3/source/m3_bind.c#L123) — 导入函数链接；[m3_env.c - m3_LoadModule()](wasm3/source/m3_env.c#L596) — 模块加载与初始化。

Wasm 规范定义了四种可以通过 import/export 在模块间共享的实体：**函数（function）**、**线性内存（memory）**、**表（table）**、**全局变量（global）**。wasm3 对这四种实体都有实现，但完善程度不同。

### 9.1 Export：导出段解析

> **源码位置**：[m3_parse.c#L230 - ParseSection_Export()](wasm3/source/m3_parse.c#L230)

Export Section 的解析将导出名称绑定到模块内部的实体索引上：

```c
// m3_parse.c#L230 — ParseSection_Export (simplified)
M3Result  ParseSection_Export  (IM3Module io_module, bytes_t i_bytes, cbytes_t i_end)
{
    u32 numExports;
    ReadLEB_u32 (& numExports, & i_bytes, i_end);

    for (u32 i = 0; i < numExports; ++i)
    {
        const char* utf8;
        u8 exportKind;
        u32 index;

        Read_utf8 (& utf8, & i_bytes, i_end);
        Read_u8 (& exportKind, & i_bytes, i_end);
        ReadLEB_u32 (& index, & i_bytes, i_end);

        if (exportKind == d_externalKind_function)
        {
            IM3Function func = &(io_module->functions [index]);
            func->export_name = utf8;               // ★ store export name on function
        }
        else if (exportKind == d_externalKind_global)
        {
            IM3Global global = &(io_module->globals [index]);
            global->name = utf8;                     // ★ store export name on global
        }
        else if (exportKind == d_externalKind_memory)
        {
            io_module->memoryExportName = utf8;      // ★ store on module
        }
        else if (exportKind == d_externalKind_table)
        {
            io_module->table0ExportName = utf8;      // ★ store on module
        }
    }
}
```

`M3Module` 中与 export 相关的字段：

```c
// m3_env.h — M3Module (export-related fields)
typedef struct M3Module {
    // ...
    M3Function *    functions;          // func->export_name stores the export name
    M3Global *      globals;            // global->name stores the export name
    const char*     memoryExportName;   // memory export name (only one memory in MVP)
    const char*     table0ExportName;   // table export name (only table 0 in MVP)
    // ...
} M3Module;
```

### 9.2 Import：导入段解析

> **源码位置**：[m3_parse.c#L148 - ParseSection_Import()](wasm3/source/m3_parse.c#L148)

Import Section 的解析根据 `importKind` 分别处理四种导入类型：

```c
// m3_parse.c#L148 — ParseSection_Import (simplified)
M3Result  ParseSection_Import  (IM3Module io_module, bytes_t i_bytes, cbytes_t i_end)
{
    M3ImportInfo import = { NULL, NULL };

    u32 numImports;
    ReadLEB_u32 (& numImports, & i_bytes, i_end);

    for (u32 i = 0; i < numImports; ++i)
    {
        u8 importKind;
        Read_utf8 (& import.moduleUtf8, & i_bytes, i_end);   // e.g. "env"
        Read_utf8 (& import.fieldUtf8, & i_bytes, i_end);    // e.g. "memory"
        Read_u8 (& importKind, & i_bytes, i_end);

        switch (importKind)
        {
            case d_externalKind_function:
            {
                u32 typeIndex;
                ReadLEB_u32 (& typeIndex, & i_bytes, i_end);
                Module_AddFunction (io_module, typeIndex, & import);  // ★ add as imported function
                io_module->numFuncImports++;
            }
            break;

            case d_externalKind_memory:
            {
                ParseType_Memory (& io_module->memoryInfo, & i_bytes, i_end);
                io_module->memoryImported = true;                     // ★ mark memory as imported
                io_module->memoryImport = import;                     // ★ store import info
            }
            break;

            case d_externalKind_global:
            {
                u8 type, isMutable;
                // ... read type and mutability ...
                Module_AddGlobal (io_module, & global, type, isMutable, true /* isImport */);
                global->import = import;                              // ★ store import info on global
            }
            break;

            case d_externalKind_table:
                // table import (parsed but limited support in wasm3)
                break;
        }
    }
}
```

**关键设计**：导入的函数占据函数索引空间的**前部**（`functions[0..numFuncImports-1]`），本模块定义的函数紧随其后。这样 `call` 指令的函数索引可以统一寻址，无需区分导入和本地函数。

### 9.3 函数导入的链接（Linking）

> **源码位置**：[m3_bind.c#L123 - FindAndLinkFunction()](wasm3/source/m3_bind.c#L123) — 查找并链接导入函数；[m3_compile.c#L2222 - CompileRawFunction()](wasm3/source/m3_compile.c#L2222) — 为宿主函数生成调用桩。

函数导入的链接分为两种方式：

#### 方式 1：绑定宿主（C）函数

通过 `m3_LinkRawFunction` API 将 C 函数绑定到 Wasm 模块的导入函数上：

```c
// m3_bind.c#L123 — FindAndLinkFunction (simplified)
M3Result  FindAndLinkFunction  (IM3Module io_module,
                                ccstr_t i_moduleName,
                                ccstr_t i_functionName,
                                ccstr_t i_signature,
                                voidptr_t i_function,
                                voidptr_t i_userdata)
{
    for (u32 i = 0; i < io_module->numFunctions; ++i)
    {
        IM3Function f = & io_module->functions [i];

        if (f->import.moduleUtf8 and f->import.fieldUtf8)
        {
            // ★ Match by module name + function name
            if (strcmp (f->import.fieldUtf8, i_functionName) == 0 and
               (wildcardModule or strcmp (f->import.moduleUtf8, i_moduleName) == 0))
            {
                ValidateSignature (f, i_signature);       // type check
                CompileRawFunction (io_module, f, i_function, i_userdata);  // generate call stub
            }
        }
    }
}
```

`CompileRawFunction` 为宿主函数生成一个极简的 M3 调用桩：

```c
// m3_compile.c#L2222 — CompileRawFunction
M3Result  CompileRawFunction  (IM3Module io_module, IM3Function io_function,
                               const void * i_function, const void * i_userdata)
{
    IM3CodePage page = AcquireCodePageWithCapacity (io_module->runtime, 4);

    io_function->compiled = GetPagePC (page);       // ★ set entry point
    io_function->module = io_module;

    EmitWord (page, op_CallRawFunction);            // op code
    EmitWord (page, i_function);                    // C function pointer
    EmitWord (page, io_function);                   // M3Function* (for context)
    EmitWord (page, i_userdata);                    // user data

    ReleaseCodePage (io_module->runtime, page);
}
```

运行时调用宿主函数时，`op_CallRawFunction` 会从 `_mem` 中恢复 runtime 指针，然后调用 C 函数：

```c
// m3_exec.h#L623 — op_CallRawFunction (simplified)
d_m3Op  (CallRawFunction)
{
    M3RawCall call = (M3RawCall) (* _pc++);
    ctx.function   = immediate (IM3Function);
    ctx.userdata   = immediate (void *);
    IM3Runtime runtime = m3MemRuntime(_mem);

    // ★ Call the host function, passing runtime, context, stack, and memory
    m3ret_t possible_trap = call (runtime, &ctx, sp, m3MemData(_mem));
}
```

#### 方式 2：Wasm 模块间的函数调用

wasm3 中同一个 `M3Runtime` 可以加载多个模块（通过多次调用 `m3_LoadModule`），模块以**链表**形式挂在 `runtime->modules` 上：

```c
// m3_env.c#L596 — m3_LoadModule (simplified)
M3Result  m3_LoadModule  (IM3Runtime io_runtime, IM3Module io_module)
{
    io_module->runtime = io_runtime;

    InitMemory (io_runtime, io_module);     // initialize or share memory
    InitGlobals (io_module);                // evaluate global init expressions
    InitDataSegments (& io_runtime->memory, io_module);
    InitElements (io_module);

    // ★ Insert module at head of linked list
    io_module->next = io_runtime->modules;
    io_runtime->modules = io_module;
}
```

当通过 `m3_FindFunction` 查找函数时，会遍历 runtime 中所有模块：

```c
// m3_env.c#L734 — m3_FindFunction
M3Result  m3_FindFunction  (IM3Function * o_function, IM3Runtime i_runtime,
                            const char * const i_functionName)
{
    // ★ ForEachModule traverses the module linked list
    function = ForEachModule (i_runtime, v_FindFunction, (void *) i_functionName);
    // ...
}

// m3_env.c#L702 — v_FindFunction: search within one module
void *  v_FindFunction  (IM3Module i_module, const char * const i_name)
{
    // 1. Prefer exported functions (by export_name)
    for (u32 i = 0; i < i_module->numFunctions; ++i) {
        IM3Function f = & i_module->functions [i];
        if (f->export_name and strcmp (f->export_name, i_name) == 0)
            return f;
    }

    // 2. Fallback: search internal functions (by debug names)
    for (u32 i = 0; i < i_module->numFunctions; ++i) {
        IM3Function f = & i_module->functions [i];
        if (f->import.moduleUtf8) continue;  // skip imports
        for (int j = 0; j < f->numNames; j++) {
            if (f->names[j] and strcmp (f->names[j], i_name) == 0)
                return f;
        }
    }
    return NULL;
}
```

**注意**：wasm3 的 `Compile_Call` 在编译 `call` 指令时，通过 `Module_GetFunction` 按索引获取函数。如果该索引指向一个导入函数，且该导入函数已经被链接（`function->compiled != NULL`），则直接发射 `op_Call` 跳转到其编译后的代码。这意味着**跨模块调用和本模块内调用在运行时没有任何额外开销**——都是同样的 `op_Call` + 函数指针。

### 9.4 线性内存共享

> **源码位置**：[m3_env.h#L183](wasm3/source/m3_env.h#L183) — `M3Runtime.memory`（唯一的内存实例）；[m3_env.c#L335 - InitMemory()](wasm3/source/m3_env.c#L335) — 内存初始化逻辑。

**wasm3 中所有模块天然共享同一块线性内存**，因为内存是 `M3Runtime` 的属性，而非 `M3Module` 的属性：

```c
// m3_env.h — M3Runtime
typedef struct M3Runtime {
    // ...
    IM3Module       modules;        // linked list of all loaded modules
    M3Memory        memory;         // ★ THE memory — shared by all modules in this runtime
    // ...
} M3Runtime;
```

内存初始化逻辑（`InitMemory`）只在第一个声明内存的模块加载时分配：

```c
// m3_env.c#L335 — InitMemory
M3Result  InitMemory  (IM3Runtime io_runtime, IM3Module i_module)
{
    if (not i_module->memoryImported)        // ★ only allocate if module defines its own memory
    {
        u32 maxPages = i_module->memoryInfo.maxPages;
        io_runtime->memory.maxPages = maxPages ? maxPages : 65536;
        result = ResizeMemory (io_runtime, i_module->memoryInfo.initPages);
    }
    // If memoryImported == true, skip allocation — use the runtime's existing memory
}
```

**共享机制**：

```
┌─────────────────────────────────────────────────────────────┐
│                        M3Runtime                            │
│                                                             │
│  memory: M3Memory ─────────────────────────────────┐        │
│    mallocated → [M3MemoryHeader | page0 | page1 | ...]     │
│                                                             │
│  modules: ──→ Module A ──→ Module B ──→ NULL                │
│               (defines      (imports                        │
│                memory)       memory)                         │
│                                                             │
│  All ops in both modules access the SAME _mem pointer       │
└─────────────────────────────────────────────────────────────┘
```

- **模块 A** 在 Import Section 中没有导入 memory，在 Memory Section 中声明了 `(memory 1 4)`。`InitMemory` 会分配 1 页初始内存。
- **模块 B** 在 Import Section 中声明 `(import "env" "memory" (memory 1))`，`memoryImported = true`。`InitMemory` 跳过分配，直接使用 runtime 已有的内存。
- 两个模块中的所有 `load`/`store` 指令在运行时都通过 `_mem` 参数访问同一块内存。

**这意味着**：在 wasm3 中，只要两个模块加载到同一个 `M3Runtime`，它们就自动共享线性内存。不需要任何额外的配置。

### 9.5 全局变量的导入/导出

> **源码位置**：[m3_env.h#L42 - M3Global](wasm3/source/m3_env.h#L42) — 全局变量结构；[m3_env.c#L414 - InitGlobals()](wasm3/source/m3_env.c#L414) — 全局变量初始化；[m3_env.c#L631 - m3_FindGlobal()](wasm3/source/m3_env.c#L631) — 按名称查找全局变量。

全局变量的 import/export 在 wasm3 中的实现：

```c
// m3_env.h — M3Global
typedef struct M3Global
{
    M3ImportInfo    import;         // import info (moduleUtf8 + fieldUtf8), empty if not imported
    union {
        i32 i32Value;
        i64 i64Value;
        f64 f64Value;
        f32 f32Value;
    };
    cstr_t          name;           // export name (NULL if not exported)
    bytes_t         initExpr;       // init expression bytecode
    u32             initExprSize;
    u8              type;           // value type
    bool            imported;       // is this global imported?
    bool            isMutable;      // is this global mutable?
} M3Global;
```

**导出全局变量**：解析 Export Section 时，将导出名称存入 `global->name`。宿主可以通过 `m3_FindGlobal` + `m3_GetGlobal`/`m3_SetGlobal` API 读写：

```c
// m3_env.c#L631 — m3_FindGlobal
IM3Global  m3_FindGlobal  (IM3Module io_module, const char * const i_globalName)
{
    // Search exports
    for (u32 i = 0; i < io_module->numGlobals; ++i) {
        IM3Global g = & io_module->globals [i];
        if (g->name and strcmp (g->name, i_globalName) == 0)
            return g;
    }
    // Search imports (by field name)
    for (u32 i = 0; i < io_module->numGlobals; ++i) {
        IM3Global g = & io_module->globals [i];
        if (g->import.fieldUtf8 and strcmp (g->import.fieldUtf8, i_globalName) == 0)
            return g;
    }
    return NULL;
}
```

**导入全局变量**：wasm3 在解析 Import Section 时会创建一个 `M3Global` 并标记 `imported = true`，但**不会自动从其他模块解析导入的全局变量值**。导入全局变量的初始值需要宿主通过 `m3_FindGlobal` + `m3_SetGlobal` 手动设置，或者通过 init expression 中的 `global.get` 引用同模块内的其他全局变量。

### 9.6 表（Table）的导入/导出

> **源码位置**：[m3_env.h#L113](wasm3/source/m3_env.h#L113) — `table0`/`table0Size`/`table0ExportName` 字段。

wasm3 对 table 的支持比较基础——只支持 `table 0`（MVP 规范只允许一个表）：

```c
// m3_env.h — M3Module (table-related fields)
typedef struct M3Module {
    // ...
    IM3Function *   table0;             // function pointer table (for call_indirect)
    u32             table0Size;         // table size
    const char*     table0ExportName;   // export name (if exported)
    // ...
} M3Module;
```

- **导出**：解析 Export Section 时存储 `table0ExportName`
- **导入**：解析 Import Section 时识别 `d_externalKind_table`，但 wasm3 当前对 table 导入的处理非常有限（基本只是跳过）
- table 主要用于 `call_indirect` 指令（间接函数调用），通过 Element Section 初始化

### 9.7 四种可共享实体总结

| 实体类型 | Export 支持 | Import 支持 | 跨模块共享机制 | wasm3 实现完善度 |
|---------|------------|------------|--------------|----------------|
| **函数** | ✅ `func->export_name` | ✅ `func->import` + `m3_LinkRawFunction` | 同 runtime 内通过函数指针直接调用 | ★★★★★ 完善 |
| **线性内存** | ✅ `module->memoryExportName` | ✅ `module->memoryImported` | 同 runtime 天然共享 `runtime->memory` | ★★★★☆ 完善（仅单内存） |
| **全局变量** | ✅ `global->name` + `m3_FindGlobal` | ✅ `global->import` + `imported` 标记 | 宿主手动桥接，或 init expr 引用 | ★★★☆☆ 基本可用 |
| **表** | ✅ `module->table0ExportName` | ⚠️ 解析但不完整 | 仅单表，主要用于 `call_indirect` | ★★☆☆☆ 有限 |

### 9.8 完整交互流程示例

以下是两个 Wasm 模块通过 import/export 共享内存和函数的完整流程：

```
WAT (Module A — exporter):
  (module
    (memory (export "mem") 1)                    ;; export memory
    (func (export "get_value") (result i32)      ;; export function
      i32.const 0
      i32.load))

WAT (Module B — importer):
  (module
    (import "A" "mem" (memory 1))                ;; import memory
    (import "A" "get_value" (func $get (result i32)))  ;; import function
    (func (export "main") (result i32)
      i32.const 0
      i32.const 42
      i32.store                                  ;; write 42 to shared memory
      call $get))                                ;; call imported function → reads 42
```

在 wasm3 中的加载和链接流程：

```c
// 1. Create runtime
IM3Runtime runtime;
m3_NewRuntime(env, &runtime, 64*1024);

// 2. Load Module A (defines memory)
IM3Module moduleA;
m3_ParseModule(env, &moduleA, wasmA, sizeA);
m3_LoadModule(runtime, moduleA);
// → InitMemory allocates 1 page (memoryImported == false)
// → runtime->memory is now initialized

// 3. Load Module B (imports memory)
IM3Module moduleB;
m3_ParseModule(env, &moduleB, wasmB, sizeB);
m3_LoadModule(runtime, moduleB);
// → InitMemory skips allocation (memoryImported == true)
// → Module B uses the same runtime->memory as Module A

// 4. Link Module B's function import to Module A's export
// (wasm3 doesn't auto-resolve Wasm-to-Wasm imports;
//  the host must bridge them, or use m3_LinkRawFunction)
IM3Function getValueFunc;
m3_FindFunction(&getValueFunc, runtime, "get_value");  // found in Module A

// 5. Call Module B's "main"
IM3Function mainFunc;
m3_FindFunction(&mainFunc, runtime, "main");
m3_CallV(mainFunc);
// → Module B writes 42 to shared memory
// → Module B calls get_value (Module A reads 42 from same memory)
// → Returns 42
```

```
Runtime Memory Layout:
┌──────────────────────────────────────────────────────┐
│                    M3Runtime                         │
│                                                      │
│  modules: → [Module B] → [Module A] → NULL           │
│                                                      │
│  memory:  [Header | 0x2A000000 ... ]                 │
│            ↑                                         │
│            Both modules' load/store ops              │
│            access this same memory via _mem          │
│                                                      │
│  stack:   [unified value + call stack]               │
└──────────────────────────────────────────────────────┘
```

---

## 10. 性能优化技巧总结

### 9.1 编译时优化

| 优化技巧 | 描述 | 性能影响 |
|----------|------|---------|
| **寄存器分配** | 将栈顶值保留在 `_r0`/`_fp0` 虚拟寄存器中 | 减少 ~35% 栈操作 |
| **操作变体** | 为同一操作码生成 ss/rs/sr 等变体 | 减少数据移动 |
| **常量折叠** | 编译时已知的常量直接嵌入代码流 | 消除运行时解码 |
| **LEB128 预解码** | 所有立即数在编译时解码为原生整数 | 消除运行时 LEB128 解码 |
| **延迟编译** | 仅在首次调用时编译函数 | 减少启动时间 |

### 9.2 运行时优化

| 优化技巧 | 描述 | 性能影响 |
|----------|------|---------|
| **Threaded code** | 函数指针链替代 switch-dispatch | 消除分发开销 |
| **尾调用分发** | nextOp() 利用编译器尾调用优化 | 零调用栈开销 |
| **参数传递寄存器** | _r0/_fp0 通过 ABI 寄存器传递 | 避免内存访问 |
| **UNLIKELY 提示** | 错误路径标记为不太可能 | 优化分支预测 |
| **统一栈** | 值栈和调用栈共享内存 | 减少分配和缓存压力 |

### 9.3 与 CLR JIT 的对比

| 方面 | wasm3 M3 | CLR JIT |
|------|----------|---------|
| **编译目标** | C 函数指针链（threaded code） | 原生机器码 |
| **编译速度** | 极快（单遍，无优化 pass） | 较慢（多遍，有优化） |
| **执行速度** | 比原生慢 ~5-15x | 接近原生 |
| **内存占用** | 极小（~15KB 代码 + 数据） | 较大（JIT 编译器本身很大） |
| **可移植性** | 纯 C，任何平台 | 需要每个 CPU 架构的后端 |
| **适用场景** | 嵌入式、IoT、资源受限 | 桌面、服务器、高性能 |

wasm3 的 M3 方法是一种精妙的折中：它比纯解释快得多（消除了 switch 开销和 LEB128 解码），又比 JIT 简单得多（不需要生成机器码），非常适合资源受限的环境。

---

## 11. 渐进式实现路线图

基于对 wasm3 的分析，建议按以下步骤实现自己的 Wasm 解释器：

### Step 1：二进制解析器（1-2 周）

**目标**：能正确解析 .wasm 文件的所有 Section

```
实现要点：
├── LEB128 decoder (unsigned + signed)
├── Module header verification (magic + version)
├── Section dispatcher (id → parser)
├── Type Section parser (function signatures)
├── Function Section parser (type indices)
├── Code Section parser (function bodies — just record positions)
├── Export Section parser
├── Import Section parser
└── Memory/Table/Global/Element/Data Section parsers
```

**测试**：用 `wat2wasm` 生成各种 .wasm 文件，验证解析结果。

### Step 2：简单 Switch-Dispatch 解释器（2-3 周）

**目标**：能执行简单的数值计算函数

```
实现要点：
├── Value stack (u64 array)
├── Call frame structure
├── Switch-dispatch loop
├── Numeric operations (i32 arithmetic, comparison)
├── Local variable access (local.get/set/tee)
├── Control flow (block/loop/if/br/br_if/return)
└── Simple function calls
```

**测试**：实现阶乘、斐波那契等经典算法的 Wasm 版本。

### Step 3：完善功能（2-3 周）

**目标**：支持线性内存、全局变量、表和间接调用

```
实现要点：
├── Linear memory (alloc, bounds check, grow)
├── Memory load/store operations (all variants)
├── Global variables
├── Table and call_indirect
├── Import/Export resolution
├── Host function binding
├── Data/Element segment initialization
└── Start function invocation
```

**测试**：运行 WebAssembly spec test suite 的核心测试。

### Step 4：Threaded Code 优化（2-4 周）

**目标**：将 switch-dispatch 升级为 threaded code 执行

```
实现要点：
├── Code page allocator
├── Compilation pass (Wasm → threaded code)
├── Operation functions (d_m3Op style)
├── nextOp() dispatch (tail call or computed goto)
├── Register allocation (_r0 / _fp0)
├── Operation variants (ss/rs/sr)
└── Lazy compilation
```

**测试**：性能基准测试，对比 switch-dispatch 和 threaded code 的速度差异。

### 每步的关键建议

1. **先正确，再优化**：Step 2 的 switch-dispatch 解释器是基础，确保它完全正确后再考虑 Step 4 的优化
2. **善用 spec test suite**：WebAssembly 官方测试套件是验证正确性的金标准
3. **参考 wasm3 但不要照搬**：理解其设计思想，但用自己的方式实现，这样学到的更多
4. **控制流是最难的部分**：结构化控制流的实现（特别是 br_table 和多层嵌套 br）需要仔细设计，建议在 Step 2 花足够时间

---

## 12. 参考资源

- **wasm3 源码**：https://github.com/nicedoc/wasm3
- **WebAssembly 规范**：https://webassembly.github.io/spec/core/
- **wasm3 技术博客**：https://github.com/nicedoc/wasm3/blob/main/docs/Interpreter.md
- **Threaded Code 论文**：Bell, J.R. "Threaded Code" (1973, Communications of the ACM)
- **WebAssembly spec test suite**：https://github.com/nicedoc/WebAssembly/tree/main/test/core
