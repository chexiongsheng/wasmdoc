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

    // Operand stack simulation (compile-time)
    u16                 stackIndex;             // current stack depth
    u16                 stackFirstDynamicIndex;
    M3CompileStack      typeStack[c_m3MaxFunctionStackHeight]; // compile-time type stack

    // Control flow
    u16                 blockDepth;
    M3CompileBlock      block;                  // current block info

    // Register allocation
    u16                 regStackIndexPlusOne[2]; // [0]=i64 reg, [1]=f64 reg

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
M3 threaded code:     [op_i32Const] [42] [op_i32Const] [10] [op_i32Add] [op_return]
                       ↑ fn ptr      ↑ imm  ↑ fn ptr    ↑imm  ↑ fn ptr    ↑ fn ptr
```

每个 M3 operation 是一个 C 函数，执行完自己的逻辑后，直接跳转到下一个 operation：

```c
// Simplified M3 operation for i32.add
d_m3Op(i32_Add) {
    i32_t lhs = _r0;                    // left operand (in register)
    i32_t rhs = slot(i32_t);            // right operand (from stack slot)
    _r0 = lhs + rhs;                    // result goes to register
    nextOp();                            // jump to next operation
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
// Emit an M3 operation (function pointer)
static inline void EmitOp(M3Compilation* c, IM3Operation op)
{
    // Get current write position in code page
    pc_t pc = GetPagePC(c->page);
    // Write the function pointer
    *pc = (m3word_t)op;
    c->page->pc++;
}

// Emit an immediate value (constant, slot index, etc.)
static inline void EmitConstant(M3Compilation* c, u64 value)
{
    pc_t pc = GetPagePC(c->page);
    *pc = (m3word_t)value;
    c->page->pc++;
}

// Emit a slot index (stack position)
static inline void EmitSlotOffset(M3Compilation* c, u16 slotIndex)
{
    // Slot offset is relative to the stack pointer
    EmitConstant(c, slotIndex * sizeof(m3slot_t));
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

wasm3 在编译时维护一个**模拟栈**（compile-time stack），跟踪每个栈槽的类型和位置：

```c
typedef struct M3CompileStack {
    u8      type;           // value type (i32/i64/f32/f64)
    u16     slotIndex;      // position in runtime stack
    u8      flags;          // constant? in register? etc.
} M3CompileStack;
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
  - Emit: [op_GetLocal_r] [slot_offset_0]
  - Register: _r0 = slot0
  - Stack: [slot0, slot1] (slot0 now also in _r0)

Step 3: local.get 1
  - _r0 is occupied → spill? No, emit a "slot + register" variant
  - Emit: nothing yet (operand tracked as slot reference)

Step 4: i32.add
  - Left operand in _r0, right operand in slot1
  - Emit: [op_i32Add_rs] [slot_offset_1]    // rs = register + slot
  - Result in _r0

Step 5: end (implicit return)
  - Return value is in _r0 (matches calling convention)
  - Emit: [op_return]

Final M3 code: [op_GetLocal_r][off0][op_i32Add_rs][off1][op_return]
```

注意 wasm3 为同一个 Wasm 操作码生成**不同变体**的 M3 operation，取决于操作数的位置：
- `op_i32Add_ss`：两个操作数都在栈槽（slot + slot）
- `op_i32Add_rs`：一个在寄存器，一个在栈槽（register + slot）
- `op_i32Add_sr`：一个在栈槽，一个在寄存器（slot + register）

这种变体化进一步减少了运行时的数据移动。

---

## 5. M3 执行引擎（m3_exec.c / m3_exec.h）

### 5.1 Operation 的定义

> **源码位置**：[m3_exec_defs.h - d_m3OpSig](wasm3/source/m3_exec_defs.h#L33) — 统一函数签名宏；[m3_exec.h](wasm3/source/m3_exec.h) — 所有 operation 实现。

每个 M3 operation 都是一个 C 函数，使用 `d_m3Op` 宏定义：

```c
// m3_exec_defs.h
#define d_m3Op(NAME)                                    \
    d_m3RetSig  op_##NAME  (d_m3OpSig)

// Expanded (simplified):
// m3word_t* op_i32Add(m3word_t* pc, u64* sp, M3MemoryHeader* mem, m3reg_t _r0, f64 _fp0)

d_m3Op(i32_Add_rs)          // register + slot variant
{
    i32_t * stack = (i32_t*)(sp + immediate(u32));  // read slot offset from code stream
    _r0 = (i32_t)_r0 + *stack;                      // perform addition
    return nextOp();                                  // dispatch to next operation
}
```

每个 operation 函数的签名统一为：

```c
m3word_t* operation(
    m3word_t*       _pc,    // program counter (points to next operation in code stream)
    u64*            _sp,    // stack pointer
    M3MemoryHeader* _mem,   // linear memory base
    m3reg_t         _r0,    // integer register
    f64             _fp0    // float register
);
```

### 5.2 分发机制：nextOp()

> **源码位置**：[m3_exec_defs.h#L62](wasm3/source/m3_exec_defs.h#L62) — `nextOpDirect`/`nextOpImpl` 宏定义；[m3_exec_defs.h - RunCode()](wasm3/source/m3_exec_defs.h#L66) — trampoline 入口。

`nextOp()` 是 M3 执行引擎的核心——它从代码流中读取下一个函数指针并跳转过去：

```c
// Option 1: Tail call (most compilers optimize this)
#define nextOp()                        \
    ((IM3Operation)(* _pc))(_pc + 1, _sp, _mem, _r0, _fp0)

// Option 2: Computed goto (GCC/Clang extension, even faster)
// Used when d_m3EnableOpTracing or specific platforms
```

**关键性能洞察**：

1. **无 switch 开销**：没有中央分发循环，每个 operation 直接跳转到下一个
2. **尾调用优化（Tail Call Optimization）**：如果编译器将 `nextOp()` 优化为尾调用，则不会消耗调用栈空间，整个执行链变成一个扁平的跳转序列
3. **参数通过寄存器传递**：`_r0` 和 `_fp0` 作为函数参数，在支持的 ABI 中会被分配到 CPU 寄存器，避免内存访问

```
Execution flow:

op_GetLocal_r → op_i32Add_rs → op_SetLocal → op_Branch → op_i32Const → ...
     │                │              │             │            │
     └── nextOp() ────┘── nextOp() ──┘── nextOp() ┘── nextOp() ┘
         (tail call)     (tail call)    (tail call)   (tail call)
```

### 5.3 立即数读取

M3 operation 通过 `immediate()` 宏从代码流中读取预编译的立即数：

```c
#define immediate(TYPE)  (*(TYPE*)(_pc++))

// Usage in an operation:
d_m3Op(i32_Const_r)
{
    _r0 = immediate(i32_t);     // read 32-bit constant from code stream
    return nextOp();
}
```

由于立即数在编译时已经从 LEB128 解码为原生整数，运行时无需任何解码开销。

### 5.4 栈操作

运行时栈是一个 `u64` 数组（每个槽 8 字节），通过偏移量访问：

```c
// Read from stack slot
#define slot(TYPE)       (*(TYPE*)(sp + immediate(u32)))

// Write to stack slot
#define slot_ptr(TYPE)   ((TYPE*)(sp + immediate(u32)))

// Example: local.set
d_m3Op(SetLocal_i)
{
    *(i32_t*)(sp + immediate(u32)) = (i32_t)_r0;
    return nextOp();
}
```

### 5.5 内存访问操作

> **源码位置**：[m3_exec.h#L1399](wasm3/source/m3_exec.h#L1399) — `d_m3Load` 宏（生成所有 load 变体）；[m3_exec.h#L1459](wasm3/source/m3_exec.h#L1459) — `d_m3Store` 宏（生成所有 store 变体）；[m3_exec.h - op_MemSize()](wasm3/source/m3_exec.h#L696) — `memory.size`；[m3_exec.h - op_MemGrow()](wasm3/source/m3_exec.h#L706) — `memory.grow`。

线性内存操作需要进行边界检查：

```c
d_m3Op(i32_Load)
{
    u32 offset = immediate(u32);                    // static offset from code stream
    u32 addr = (u32)_r0 + offset;                   // effective address
    u32 end = addr + sizeof(i32_t);

    if (UNLIKELY(end > _mem->length)) {             // bounds check
        return m3Err_trapOutOfBoundsMemoryAccess;
    }

    u8* memBase = m3MemData(_mem);
    _r0 = *(i32_t*)(memBase + addr);                // load value (little-endian)
    return nextOp();
}
```

`UNLIKELY` 宏提示编译器边界检查失败是罕见路径，优化分支预测。

### 5.6 函数调用

> **源码位置**：[m3_exec.h - op_Call()](wasm3/source/m3_exec.h#L542) — 直接调用；[m3_exec.h - op_CallIndirect()](wasm3/source/m3_exec.h#L568) — 间接调用；[m3_exec.h - op_CallRawFunction()](wasm3/source/m3_exec.h#L623) — 宿主函数调用；[m3_exec.h - op_Entry()](wasm3/source/m3_exec.h#L802) — 函数入口（分配局部变量、检查栈溢出）；[m3_exec.h - op_Return()](wasm3/source/m3_exec.h#L1154) — 函数返回。

函数调用是最复杂的 operation 之一：

```c
d_m3Op(Call)
{
    pc_t callPC = immediate(pc_t);      // target function's compiled code
    i32 stackOffset = immediate(i32);   // stack frame offset

    // Save current state on the call stack (grows from the other end)
    m3stack_t callStack = (m3stack_t)(_sp + stackOffset);
    *(void**)(callStack + 0) = _pc;     // save return address
    *(void**)(callStack + 1) = _sp;     // save stack pointer
    *(void**)(callStack + 2) = _mem;    // save memory base

    // Set up new frame
    _sp = _sp + stackOffset;            // advance stack pointer
    _pc = callPC;                       // jump to callee

    return nextOp();                    // enter callee's first operation
}
```

**间接调用（call_indirect）** 额外需要运行时类型检查：

```c
d_m3Op(CallIndirect)
{
    u32 tableIndex = (u32)_r0;          // index from stack top
    IM3FuncType expectedType = immediate(IM3FuncType);

    // Bounds check on table
    if (tableIndex >= table->size)
        return m3Err_trapTableIndexOutOfRange;

    IM3Function target = table->functions[tableIndex];

    // Type signature check
    if (target->funcType != expectedType)
        return m3Err_trapIndirectCallTypeMismatch;

    // Lazy compile if needed
    if (!target->compiled)
        Compile_Function(target);

    // Proceed with call (similar to direct call)
    // ...
}
```

### 5.7 Trap 处理

> **源码位置**：[wasm3.h#L107](wasm3/source/wasm3.h#L107) — 所有错误码/trap 字符串定义（`m3Err_*`）；[m3_exec.h - op_Compile()](wasm3/source/m3_exec.h#L778) — 懒编译入口（首次调用时触发编译）。

wasm3 使用 C 的返回值来传播 trap（而非 setjmp/longjmp）：

```c
// Normal execution: return nextOp() — continues the chain
// Trap: return error pointer — unwinds the chain

d_m3Op(i32_DivS)
{
    i32_t divisor = slot(i32_t);
    if (UNLIKELY(divisor == 0))
        return m3Err_trapDivisionByZero;
    if (UNLIKELY(divisor == -1 && (i32_t)_r0 == INT32_MIN))
        return m3Err_trapIntegerOverflow;
    _r0 = (i32_t)_r0 / divisor;
    return nextOp();
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

// m3ApiGetArg expands to:
// int32_t a = *(int32_t*)(_sp++);
```

### 7.2 绑定的编译处理

当编译器遇到对导入函数的调用时，会发射一个特殊的 `op_CallRawFunction` operation：

```c
d_m3Op(CallRawFunction)
{
    M3RawCall rawCall = immediate(M3RawCall);   // host function pointer
    IM3Function function = immediate(IM3Function);

    // Set up stack for the host function
    // Arguments are already on the stack in the correct layout

    // Call the host function
    m3ret_t result = rawCall(runtime, &ctx, _sp, _mem);

    if (result)
        return result;  // propagate error

    return nextOp();
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

typedef struct M3MemoryHeader {
    u32                     length;         // current byte length
    u32                     maxLength;      // maximum byte length
    // Actual memory data follows immediately after this header
}
M3MemoryHeader;
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

## 9. 性能优化技巧总结

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

## 10. 渐进式实现路线图

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

## 11. 参考资源

- **wasm3 源码**：https://github.com/nicedoc/wasm3
- **WebAssembly 规范**：https://webassembly.github.io/spec/core/
- **wasm3 技术博客**：https://github.com/nicedoc/wasm3/blob/main/docs/Interpreter.md
- **Threaded Code 论文**：Bell, J.R. "Threaded Code" (1973, Communications of the ACM)
- **WebAssembly spec test suite**：https://github.com/nicedoc/WebAssembly/tree/main/test/core
