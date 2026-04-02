# 第四章：Wasm 执行引擎设计与实现

> **前置知识**：已阅读第一至三章（二进制格式、指令集与类型系统、验证机制）
>
> **本章目标**：理解 Wasm 解释器执行引擎的核心设计方案，能够选择并实现一个高效的执行引擎

---

## 目录

1. [执行引擎概述](#1-执行引擎概述)
2. [三种执行策略对比](#2-三种执行策略对比)
3. [运行时数据结构设计](#3-运行时数据结构设计)
4. [函数调用机制](#4-函数调用机制)
5. [内存操作实现](#5-内存操作实现)
6. [表（Table）与间接调用](#6-表table与间接调用)
7. [Trap 与错误处理](#7-trap-与错误处理)
8. [wasm3 M3 架构专题](#8-wasm3-m3-架构专题)
9. [与 CLR 执行模型对比](#9-与-clr-执行模型对比)
10. [性能优化技巧](#10-性能优化技巧)

---

## 1. 执行引擎概述

Wasm 执行引擎的职责是：给定一个已验证的 Wasm 模块实例，执行其中的函数。从宏观上看，执行引擎需要：

1. **管理运行时状态**：操作数栈、调用栈、线性内存、表、全局变量
2. **解码并执行指令**：逐条（或批量）处理 Wasm 指令
3. **处理控制流**：分支、循环、函数调用与返回
4. **处理异常情况**：越界访问、类型不匹配、栈溢出等 trap

### 1.1 执行引擎在整体架构中的位置

```
┌─────────────────────────────────────────────────┐
│                  Host Application                │
│  (C/C++/Rust program that embeds the runtime)    │
├─────────────────────────────────────────────────┤
│              Wasm Runtime API Layer              │
│  (module loading, instantiation, function call)  │
├──────────┬──────────┬───────────────────────────┤
│  Decoder │Validator │    Execution Engine ◄──────│── THIS CHAPTER
│ (Ch.1)   │ (Ch.3)   │                           │
├──────────┴──────────┴───────────────────────────┤
│              Runtime Data Structures             │
│  (value stack, call stack, linear memory,        │
│   tables, globals, module instances)             │
└─────────────────────────────────────────────────┘
```

### 1.2 设计决策的核心权衡

| 维度 | 权衡 |
|------|------|
| **性能 vs 复杂度** | 更快的执行需要更复杂的编译/转译阶段 |
| **启动时间 vs 运行时间** | 预编译提升运行速度但增加启动延迟 |
| **内存占用 vs 速度** | 缓存友好的数据布局可能需要更多内存 |
| **可移植性 vs 性能** | 平台特定优化（如 computed goto）牺牲可移植性 |
| **安全性 vs 性能** | 每次内存访问的边界检查有运行时开销 |

---

## 2. 三种执行策略对比

### 2.1 策略一：AST 遍历解释（Tree-Walking Interpreter）

最简单的方式——将 Wasm 字节码解析为 AST，然后递归遍历执行。

```c
// Conceptual AST node
typedef struct AstNode {
    uint8_t opcode;
    union {
        int32_t i32_val;       // i32.const immediate
        uint32_t local_idx;    // local.get/set index
        struct {               // block/loop/if
            uint8_t block_type;
            struct AstNode *body;
            struct AstNode *else_body; // for if
            int body_count;
            int else_count;
        } block;
        struct {               // br/br_if
            uint32_t label_depth;
        } branch;
    } data;
} AstNode;

// Recursive evaluation
WasmValue eval(AstNode *node, ExecContext *ctx) {
    switch (node->opcode) {
        case OP_I32_CONST:
            return (WasmValue){.i32 = node->data.i32_val};

        case OP_I32_ADD: {
            WasmValue b = pop(ctx);
            WasmValue a = pop(ctx);
            return (WasmValue){.i32 = a.i32 + b.i32};
        }

        case OP_LOCAL_GET:
            return ctx->locals[node->data.local_idx];

        case OP_BLOCK:
            for (int i = 0; i < node->data.block.body_count; i++) {
                eval(&node->data.block.body[i], ctx);
                if (ctx->branch_target >= 0) {
                    // Handle branch
                    break;
                }
            }
            return WASM_VOID;

        // ... hundreds more cases
    }
}
```

**优点**：
- 实现最简单，容易理解和调试
- 代码结构清晰，与规范一一对应

**缺点**：
- **极慢**：每条指令都需要递归调用、switch 分发、指针追踪
- **内存开销大**：AST 节点需要大量堆分配
- **缓存不友好**：AST 节点散布在堆上，cache miss 严重
- 不适合生产使用，仅适合原型验证

### 2.2 策略二：字节码 Switch-Dispatch 解释器

直接在 Wasm 字节码流上执行，用一个大 switch 语句分发指令。这是最经典的解释器模式。

```c
typedef struct {
    // Program counter - points into wasm bytecode
    const uint8_t *pc;

    // Operand stack
    WasmValue *stack;
    WasmValue *sp;  // stack pointer (points to top)

    // Call frame info
    WasmValue *locals;       // base of local variables
    WasmValue *stack_base;   // base of operand stack for current frame

    // Control flow
    ControlFrame *ctrl_stack;
    int ctrl_depth;
} InterpState;

// The main dispatch loop
WasmResult interpret(InterpState *state) {
    const uint8_t *pc = state->pc;
    WasmValue *sp = state->sp;

    for (;;) {
        uint8_t opcode = *pc++;

        switch (opcode) {
            case 0x00: // unreachable
                return TRAP_UNREACHABLE;

            case 0x01: // nop
                break;

            case 0x02: { // block
                int8_t block_type = (int8_t)*pc++;
                push_control_frame(state, FRAME_BLOCK, block_type, pc);
                break;
            }

            case 0x03: { // loop
                int8_t block_type = (int8_t)*pc++;
                // loop's continuation is the START of the loop body
                push_control_frame(state, FRAME_LOOP, block_type, pc);
                break;
            }

            case 0x0B: { // end
                ControlFrame *frame = pop_control_frame(state);
                if (frame == NULL) {
                    // End of function
                    state->sp = sp;
                    return RESULT_OK;
                }
                break;
            }

            case 0x0C: { // br
                uint32_t depth = read_leb128_u32(&pc);
                branch_to(state, depth, &pc, &sp);
                break;
            }

            case 0x20: { // local.get
                uint32_t idx = read_leb128_u32(&pc);
                *sp++ = state->locals[idx];
                break;
            }

            case 0x41: { // i32.const
                int32_t val = read_leb128_i32(&pc);
                (sp++)->i32 = val;
                break;
            }

            case 0x6A: { // i32.add
                sp--;
                (sp - 1)->i32 += sp->i32;
                break;
            }

            case 0x6B: { // i32.sub
                sp--;
                (sp - 1)->i32 -= sp->i32;
                break;
            }

            // ... all other opcodes

            default:
                return TRAP_INVALID_OPCODE;
        }
    }
}
```

**优点**：
- 实现相对简单，代码紧凑
- 直接操作字节码流，无需构建 AST，启动快
- 内存占用小

**缺点**：
- **switch 分发开销**：每条指令都经过一次 switch 跳转，现代 CPU 的分支预测器难以预测（因为跳转目标取决于运行时的 opcode 值）
- 在 x86 上，一个 switch 大约有 10-20 个时钟周期的分发开销
- 可通过 computed goto 优化（见 2.2.1）

#### 2.2.1 Computed Goto 优化（GCC/Clang 扩展）

```c
// Label address table - one entry per opcode
static const void *dispatch_table[256] = {
    [0x00] = &&op_unreachable,
    [0x01] = &&op_nop,
    [0x02] = &&op_block,
    // ...
    [0x41] = &&op_i32_const,
    [0x6A] = &&op_i32_add,
    // ...
};

#define DISPATCH() goto *dispatch_table[*pc++]

// Start execution
DISPATCH();

op_nop:
    DISPATCH();

op_i32_const: {
    int32_t val = read_leb128_i32(&pc);
    (sp++)->i32 = val;
    DISPATCH();
}

op_i32_add: {
    sp--;
    (sp - 1)->i32 += sp->i32;
    DISPATCH();
}

// ... etc
```

**Computed goto 的优势**：
- 每条指令末尾直接跳转到下一条指令的处理代码，消除了 switch 的集中跳转瓶颈
- CPU 分支预测器可以为每个跳转点维护独立的预测历史（因为每个 `goto *` 在不同的代码位置）
- 实测可提升 30%-100% 的解释速度

**局限**：
- 这是 GCC/Clang 扩展，MSVC 不支持
- 代码可读性下降

### 2.3 策略三：Threaded Code（wasm3 的 M3 方式）

> **wasm3 实现参考**：[m3_exec_defs.h - d_m3OpSig](wasm3/source/m3_exec_defs.h#L17) — M3 operation 的统一函数签名；[m3_exec_defs.h - nextOpDirect](wasm3/source/m3_exec_defs.h#L62) — 尾调用分发宏；[m3_exec.h](wasm3/source/m3_exec.h) — 所有 M3 operation 的实现。

wasm3 采用了一种更激进的方式：在加载时将 Wasm 字节码**编译**为一个 C 函数指针数组（threaded code），运行时通过函数指针链式调用执行。

```
Wasm bytecode:        M3 threaded code:
                      ┌──────────────────────┐
i32.const 42    ──►   │ op_i32_const_ptr ─────│──► C function: op_Const32
                      │ immediate: 42         │
i32.const 10    ──►   │ op_i32_const_ptr ─────│──► C function: op_Const32
                      │ immediate: 10         │
i32.add         ──►   │ op_i32_add_ptr   ─────│──► C function: op_Add_i32
                      │                       │
end             ──►   │ op_end_ptr       ─────│──► C function: op_End
                      └──────────────────────┘
```

每个 M3 operation 是一个 C 函数，执行完自己的逻辑后，通过尾调用（tail call）跳转到下一个 operation：

```c
// M3 operation signature (simplified from wasm3)
typedef const void * (*M3Operation)(
    pc_t pc,           // program counter (points to next operation)
    u64 *sp,           // stack pointer
    M3MemoryHeader *mem, // linear memory
    d_m3OpSig          // additional register args
);

// Example: i32.const
d_m3Op(Const32) {
    u32 value = *(u32 *)pc;     // read immediate from pc stream
    pc += sizeof(u32);
    push_i32(sp, value);
    return nextOp();             // tail-call to next operation
}

// Example: i32.add
d_m3Op(Add_i32) {
    i32 b = pop_i32(sp);
    i32 a = pop_i32(sp);
    push_i32(sp, a + b);
    return nextOp();
}

// nextOp macro - the core of threaded dispatch
#define nextOp() ((M3Operation)(*pc))(pc + 1, sp, mem, d_m3OpArgs)
```

**优点**：
- **极高的分发效率**：每条指令通过直接函数指针调用跳转，无 switch 开销
- **立即数预解码**：LEB128 等编码在编译阶段已解码为原生整数，运行时零开销
- **编译器友好**：每个 operation 是独立的小函数，编译器可以充分优化
- **适合资源受限环境**：不需要 JIT 编译器，纯 C 实现，可移植性好

**缺点**：
- **实现复杂度高**：需要一个完整的编译阶段（Wasm → M3 operations）
- **启动时间增加**：编译阶段需要时间
- **调试困难**：threaded code 的执行流不直观
- **依赖编译器优化**：需要编译器支持尾调用优化（TCO），否则会栈溢出

### 2.4 三种策略总结对比

| 特性 | AST 遍历 | Switch-Dispatch | Threaded Code (M3) |
|------|----------|-----------------|---------------------|
| **实现复杂度** | ★☆☆ 低 | ★★☆ 中 | ★★★ 高 |
| **启动时间** | 中（需构建AST） | 快（直接执行） | 中（需编译阶段） |
| **执行速度** | 慢 (1x) | 中 (3-5x) | 快 (8-15x) |
| **内存占用** | 高（AST节点） | 低（原始字节码） | 中（operation数组） |
| **可移植性** | 高 | 高（computed goto需GCC） | 高（纯C，需TCO） |
| **调试友好** | 高 | 中 | 低 |
| **适用场景** | 原型/教学 | 通用解释器 | 嵌入式/高性能解释器 |

> **建议**：初学者从 switch-dispatch 开始实现，这是理解执行引擎的最佳起点。在此基础上再尝试 threaded code 优化。

---

## 3. 运行时数据结构设计

### 3.1 操作数栈（Value Stack）

> **wasm3 实现参考**：[m3_env.h - M3Runtime](wasm3/source/m3_env.h#L159) — `stack` 字段为运行时操作数栈；[m3_core.h#L36](wasm3/source/m3_core.h#L36) — `m3slot_t` 定义为 `uint64_t`，即 8 字节无类型标签栈槽。

Wasm 是栈式虚拟机，操作数栈是最核心的数据结构。

#### 3.1.1 栈槽（Stack Slot）设计

关键决策：每个栈槽应该多大？

```c
// Option A: Tagged union (type-safe but larger)
typedef struct {
    enum { VAL_I32, VAL_I64, VAL_F32, VAL_F64, VAL_REF } type;
    union {
        int32_t  i32;
        int64_t  i64;
        float    f32;
        double   f64;
        void    *ref;
    } value;
} WasmValue_Tagged;  // 16 bytes per slot

// Option B: Untagged 64-bit slot (fast but relies on validation)
typedef union {
    int32_t  i32;
    int64_t  i64;
    float    f32;
    double   f64;
    uint64_t raw;    // for bitwise operations
    void    *ref;
} WasmValue;  // 8 bytes per slot

// Option C: Raw 64-bit (wasm3 style, maximum performance)
typedef uint64_t m3slot_t;  // 8 bytes, type info discarded
```

**推荐方案**：使用 **Option B（Untagged 64-bit slot）**。

理由：
- Wasm 已经过验证，运行时不需要类型标签
- 8 字节对齐，cache 友好
- i32 值存入 64-bit slot 时，高 32 位清零（规范要求）
- 与 wasm3 的做法一致

#### 3.1.2 栈的内存布局

```c
typedef struct {
    WasmValue *data;       // heap-allocated array
    WasmValue *sp;         // stack pointer (points to next free slot)
    WasmValue *base;       // bottom of stack
    WasmValue *limit;      // top of allocated region (for overflow check)
    size_t capacity;       // total slots allocated
} ValueStack;

// Stack operations
static inline void stack_push_i32(ValueStack *s, int32_t val) {
    assert(s->sp < s->limit);  // overflow check
    (s->sp++)->i32 = val;
}

static inline int32_t stack_pop_i32(ValueStack *s) {
    assert(s->sp > s->base);   // underflow check
    return (--s->sp)->i32;
}

// Peek without popping (useful for local.tee)
static inline int32_t stack_peek_i32(ValueStack *s) {
    return (s->sp - 1)->i32;
}
```

> **实现提示**：在 debug 模式下保留 assert 检查，release 模式下移除（因为验证阶段已保证栈安全）。

### 3.2 调用栈与帧（Call Stack & Frames）

每次函数调用创建一个新的调用帧（call frame），记录返回信息和局部变量。

```c
typedef struct CallFrame {
    // Return information
    const uint8_t *return_pc;    // where to resume after return
    WasmValue *return_sp;        // stack pointer to restore on return
    uint32_t func_idx;           // for debugging/stack traces

    // Local variables
    WasmValue *locals;           // pointer to locals array
    uint32_t num_locals;         // total locals (params + declared locals)

    // Control flow
    ControlFrame *ctrl_stack;    // control frame stack for this function
    int ctrl_depth;

    // Module context
    ModuleInstance *module;       // for accessing globals, memory, tables
} CallFrame;

typedef struct {
    CallFrame *frames;           // array of call frames
    int depth;                   // current call depth
    int max_depth;               // limit (e.g., 1000) to prevent stack overflow
} CallStack;
```

#### 3.2.1 局部变量的存储策略

有两种常见方案：

**方案 A：局部变量在操作数栈上（wasm3 风格）**

```
Value Stack:
┌─────────────────────────────────────────────────┐
│ ... caller's operands ... │ param0 │ param1 │ local0 │ local1 │ ... callee's operands ... │
└─────────────────────────────────────────────────┘
                             ▲                     ▲
                             │                     │
                        frame.locals          frame.stack_base
```

优点：局部变量和操作数在同一块连续内存中，cache 友好。
缺点：函数调用时需要在栈上预留局部变量空间。

**方案 B：局部变量独立分配**

```
Value Stack:                    Locals Array (per frame):
┌──────────────────────┐       ┌──────────────────────┐
│ ... operands ...     │       │ param0 │ param1 │     │
└──────────────────────┘       │ local0 │ local1 │     │
                               └──────────────────────┘
```

优点：操作数栈更简洁，局部变量管理独立。
缺点：额外的内存分配，可能的 cache miss。

> **推荐**：方案 A（栈上局部变量），这是 wasm3 和大多数高性能解释器的选择。

### 3.3 控制帧栈（Control Frame Stack）

用于跟踪 block/loop/if 的嵌套结构，支持 br 指令的跳转。

```c
typedef enum {
    CTRL_BLOCK,
    CTRL_LOOP,
    CTRL_IF,
    CTRL_ELSE
} ControlFrameKind;

typedef struct ControlFrame {
    ControlFrameKind kind;

    // Type information
    uint32_t param_count;     // number of block parameters
    uint32_t result_count;    // number of block results

    // Stack state
    WasmValue *stack_base;    // operand stack height at block entry

    // Branch targets
    const uint8_t *branch_target;  // where to jump on br
    // For block/if: branch_target = end of block (after 'end')
    // For loop:     branch_target = start of loop body (after 'loop')

    // For if/else tracking
    const uint8_t *else_target;    // location of 'else' (for if blocks)
    bool has_else;
} ControlFrame;
```

**关键语义**：`br` 跳转到的位置取决于目标控制帧的类型：
- **block/if**：跳转到 block 的 **末尾**（end 之后），相当于 break
- **loop**：跳转到 loop 的 **开头**（loop 指令之后），相当于 continue

```
block:                          loop:
  ┌─────────┐                    ┌─────────┐
  │ block   │                    │ loop    │ ◄── br jumps HERE
  │  ...    │                    │  ...    │
  │  br 0 ──│──┐                 │  br 0 ──│──┘
  │  ...    │  │                 │  ...    │
  │ end     │  │                 │ end     │
  └─────────┘  │                 └─────────┘
               ▼ (continues here)
```

### 3.4 线性内存（Linear Memory）

> **wasm3 实现参考**：[m3_env.h - M3Memory](wasm3/source/m3_env.h#L28) — 内存结构定义；[m3_env.c - ResizeMemory()](wasm3/source/m3_env.c#L262) — 内存增长实现；[m3_core.h - M3MemoryHeader](wasm3/source/m3_core.h#L123) — 内存头部结构。

```c
typedef struct {
    uint8_t *data;          // the actual memory buffer
    uint32_t size;          // current size in bytes (always multiple of 64KB)
    uint32_t max_size;      // maximum size in bytes (from memory type limits)
    uint32_t page_count;    // current size in pages (1 page = 65536 bytes)
    uint32_t max_pages;     // maximum pages (0xFFFF if no max specified)
} LinearMemory;

#define WASM_PAGE_SIZE 65536  // 64 KiB

// Allocate initial memory
LinearMemory *memory_create(uint32_t initial_pages, uint32_t max_pages) {
    LinearMemory *mem = calloc(1, sizeof(LinearMemory));
    mem->page_count = initial_pages;
    mem->max_pages = max_pages;
    mem->size = initial_pages * WASM_PAGE_SIZE;
    mem->max_size = max_pages * WASM_PAGE_SIZE;

    // Use mmap on Unix for efficient grow, or malloc
    mem->data = calloc(1, mem->size);  // zero-initialized per spec
    return mem;
}
```

### 3.5 表（Table）

```c
typedef struct {
    WasmValue *elements;    // array of references (funcref or externref)
    uint32_t size;          // current number of elements
    uint32_t max_size;      // maximum elements
    uint8_t elem_type;      // 0x70 = funcref, 0x6F = externref
} WasmTable;
```

### 3.6 全局变量（Globals）

```c
typedef struct {
    WasmValue value;
    uint8_t type;           // value type (i32/i64/f32/f64/ref)
    bool mutable;           // const or var
} WasmGlobal;
```

### 3.7 模块实例（Module Instance）—— 汇总

> **wasm3 实现参考**：[m3_env.h - M3Module](wasm3/source/m3_env.h#L82) — 模块结构（包含函数/全局变量/导入导出等）；[m3_env.h - M3Runtime](wasm3/source/m3_env.h#L159) — 运行时结构（包含栈/内存/模块链表）；[m3_env.h - M3Environment](wasm3/source/m3_env.h#L139) — 环境结构（全局类型去重）。

```c
typedef struct ModuleInstance {
    // Decoded module (shared, read-only)
    WasmModule *module;

    // Runtime instances (per-instantiation)
    LinearMemory **memories;     // array of memory instances
    uint32_t memory_count;

    WasmTable **tables;          // array of table instances
    uint32_t table_count;

    WasmGlobal *globals;         // array of global instances
    uint32_t global_count;

    // Function instances (includes imported + defined)
    FuncInstance *functions;
    uint32_t func_count;

    // Exports lookup
    ExportEntry *exports;
    uint32_t export_count;
} ModuleInstance;

typedef struct FuncInstance {
    bool is_import;
    union {
        struct {
            WasmModule *module;
            uint32_t code_offset;   // offset into code section
            uint32_t type_idx;
        } wasm_func;
        struct {
            HostFuncPtr callback;
            void *user_data;
            uint32_t type_idx;
        } host_func;
    };
} FuncInstance;
```

---

## 4. 函数调用机制

### 4.1 直接调用（call）

> **wasm3 实现参考**：[m3_compile.c - Compile_Call()](wasm3/source/m3_compile.c#L1705) — 编译阶段处理 `call` 指令；[m3_exec.h - op_Call()](wasm3/source/m3_exec.h#L554) — 运行时直接调用实现；[m3_exec.h - op_Entry()](wasm3/source/m3_exec.h#L1168) — 函数入口（分配局部变量、检查栈溢出）。

`call` 指令的立即数是函数索引。执行流程：

```
call funcidx
```

```c
case OP_CALL: {
    uint32_t func_idx = read_leb128_u32(&pc);
    FuncInstance *callee = &module->functions[func_idx];

    if (callee->is_import) {
        // Host function call
        call_host_function(callee, state);
    } else {
        // Wasm function call
        FuncType *type = get_func_type(module, callee);

        // 1. Save current frame
        push_call_frame(state, pc, sp);

        // 2. Set up new frame
        //    Arguments are already on the stack (pushed by caller)
        WasmValue *args_base = sp - type->param_count;

        // 3. Allocate space for declared locals (zero-initialized)
        uint32_t num_declared_locals = get_local_count(callee);
        state->locals = args_base;
        sp = args_base + type->param_count + num_declared_locals;
        memset(args_base + type->param_count, 0,
               num_declared_locals * sizeof(WasmValue));

        // 4. Set new PC to function body
        pc = get_func_body(module, callee);

        // 5. Reset control frame stack for new function
        state->ctrl_depth = 0;
        push_control_frame(state, CTRL_BLOCK, type->result_count, NULL);
    }
    break;
}
```

### 4.2 间接调用（call_indirect）

> **wasm3 实现参考**：[m3_exec.h - op_CallIndirect()](wasm3/source/m3_exec.h#L580) — 运行时间接调用实现（包含表越界检查、类型匹配检查）。

`call_indirect` 通过表（table）进行间接函数调用，类似 C 的函数指针调用。

```
call_indirect typeidx tableidx
```

```c
case OP_CALL_INDIRECT: {
    uint32_t type_idx = read_leb128_u32(&pc);
    uint32_t table_idx = read_leb128_u32(&pc);  // usually 0 in MVP

    // 1. Pop the index from the stack
    int32_t elem_idx = stack_pop_i32(sp);

    // 2. Bounds check on table
    WasmTable *table = module->tables[table_idx];
    if (elem_idx < 0 || (uint32_t)elem_idx >= table->size) {
        return TRAP_UNDEFINED_ELEMENT;
    }

    // 3. Get function reference from table
    FuncInstance *callee = table->elements[elem_idx].ref;
    if (callee == NULL) {
        return TRAP_UNINITIALIZED_ELEMENT;
    }

    // 4. Runtime type check! (this is the key safety mechanism)
    FuncType *expected = &module->types[type_idx];
    FuncType *actual = get_func_type(callee->module, callee);
    if (!func_types_equal(expected, actual)) {
        return TRAP_INDIRECT_CALL_TYPE_MISMATCH;
    }

    // 5. Proceed with call (same as direct call)
    // ...
    break;
}
```

> **重要**：`call_indirect` 的运行时类型检查是 Wasm 安全模型的关键部分。即使模块已通过验证，间接调用的目标函数类型只能在运行时确定，因此必须检查。

### 4.3 函数返回

```c
// When 'end' is reached at function level (ctrl_depth == 0)
// or 'return' instruction is executed:

case OP_RETURN: {
    // 1. Get current function's return type
    FuncType *type = get_current_func_type(state);

    // 2. Collect return values from stack top
    //    (they are the top N values on the stack)
    WasmValue results[MAX_RESULTS];
    for (int i = type->result_count - 1; i >= 0; i--) {
        results[i] = *(--sp);
    }

    // 3. Restore caller's frame
    CallFrame *caller = pop_call_frame(state);
    pc = caller->return_pc;
    sp = caller->return_sp;
    state->locals = caller->locals;

    // 4. Push return values onto caller's stack
    for (uint32_t i = 0; i < type->result_count; i++) {
        *sp++ = results[i];
    }
    break;
}
```

### 4.4 多返回值（Multi-Value）

Wasm 2.0 支持函数和 block 返回多个值。这对栈布局有影响：

```wasm
;; Function returning two values
(func $divmod (param i32 i32) (result i32 i32)
  local.get 0
  local.get 1
  i32.div_u      ;; quotient
  local.get 0
  local.get 1
  i32.rem_u      ;; remainder
)
;; After call: stack has [quotient, remainder] (quotient pushed first)
```

实现要点：
- 返回值按顺序留在栈顶
- `br` 跳出 block 时，需要将 block 的结果值从栈顶"搬运"到 block 入口的栈位置

```c
// Branch with multi-value: need to move results over intermediate values
void branch_to(InterpState *state, uint32_t depth, ...) {
    ControlFrame *target = &state->ctrl_stack[state->ctrl_depth - 1 - depth];
    uint32_t result_count = target->result_count;

    // Save result values
    WasmValue results[MAX_RESULTS];
    for (int i = result_count - 1; i >= 0; i--) {
        results[i] = *(--sp);
    }

    // Reset stack to block entry point
    sp = target->stack_base;

    // For loop: also need to push back the loop parameters
    if (target->kind == CTRL_LOOP) {
        // Loop branch: params are the "results" for the branch
        // (loop's continuation expects its parameters)
    }

    // Push results back
    for (uint32_t i = 0; i < result_count; i++) {
        *sp++ = results[i];
    }

    // Set PC to branch target
    pc = target->branch_target;
}
```

### 4.5 宿主函数调用（Host Function Call）

> **wasm3 实现参考**：[m3_exec.h - op_CallRawFunction()](wasm3/source/m3_exec.h#L629) — 调用原始宿主函数；[wasm3.h - M3RawCall](wasm3/source/wasm3.h#L233) — 宿主函数回调类型定义；[wasm3.h - m3ApiRawFunction](wasm3/source/wasm3.h#L340) — 宿主函数声明宏。

宿主函数是由嵌入环境（C/C++ 程序）提供的函数，通过 import 机制绑定。

```c
// Host function signature
typedef WasmResult (*HostFuncPtr)(
    void *user_data,           // arbitrary context
    const WasmValue *args,     // input arguments
    WasmValue *results         // output results
);

// Calling a host function
WasmResult call_host_function(FuncInstance *func, InterpState *state) {
    FuncType *type = get_func_type(func);

    // 1. Pop arguments from stack (in reverse order)
    WasmValue args[MAX_PARAMS];
    for (int i = type->param_count - 1; i >= 0; i--) {
        args[i] = *(--state->sp);
    }

    // 2. Call the host function
    WasmValue results[MAX_RESULTS];
    WasmResult result = func->host_func.callback(
        func->host_func.user_data, args, results
    );

    if (result != RESULT_OK) {
        return result;  // propagate trap
    }

    // 3. Push results onto stack
    for (uint32_t i = 0; i < type->result_count; i++) {
        *state->sp++ = results[i];
    }

    return RESULT_OK;
}
```

---

## 5. 内存操作实现

### 5.1 字节序

Wasm 规范要求线性内存使用**小端字节序（little-endian）**。在小端平台（x86、ARM 默认模式）上可以直接 memcpy；在大端平台上需要字节交换。

```c
// Load i32 from linear memory
static inline int32_t memory_load_i32(LinearMemory *mem, uint32_t addr) {
    // Bounds check
    if (addr + 4 > mem->size) {
        trap(TRAP_MEMORY_OUT_OF_BOUNDS);
    }

    int32_t value;
    memcpy(&value, mem->data + addr, 4);  // works on little-endian

    // On big-endian platforms, would need:
    // value = __builtin_bswap32(value);

    return value;
}
```

### 5.2 对齐（Alignment）

Wasm 的 load/store 指令有一个 `align` 立即数（以 2 的幂次表示），但这只是一个**提示（hint）**，不是强制要求：

```
i32.load align=2 offset=0
         ^^^^^ log2(alignment) = 2, meaning 4-byte aligned
```

```c
case OP_I32_LOAD: {
    uint32_t align = read_leb128_u32(&pc);  // alignment hint (log2)
    uint32_t offset = read_leb128_u32(&pc); // static offset

    int32_t base_addr = stack_pop_i32(sp);
    uint64_t effective_addr = (uint64_t)base_addr + offset;

    // Bounds check (use 64-bit to detect overflow)
    if (effective_addr + 4 > mem->size) {
        return TRAP_MEMORY_OUT_OF_BOUNDS;
    }

    // Note: alignment is just a hint, we must handle unaligned access
    // Using memcpy is safe for both aligned and unaligned access
    int32_t value;
    memcpy(&value, mem->data + effective_addr, 4);

    stack_push_i32(sp, value);
    break;
}
```

> **关键点**：即使 `align` 提示为 4 字节对齐，实际地址可能不对齐。实现必须正确处理非对齐访问。使用 `memcpy` 是最安全的方式（编译器会优化为适当的指令）。

### 5.3 部分加载与符号扩展

Wasm 提供了多种宽度的 load/store 指令：

```c
// i32.load8_s  - load 1 byte, sign-extend to i32
// i32.load8_u  - load 1 byte, zero-extend to i32
// i32.load16_s - load 2 bytes, sign-extend to i32
// i32.load16_u - load 2 bytes, zero-extend to i32
// i32.load     - load 4 bytes

case OP_I32_LOAD8_S: {
    uint32_t align = read_leb128_u32(&pc);
    uint32_t offset = read_leb128_u32(&pc);
    uint32_t addr = stack_pop_i32(sp) + offset;

    if (addr + 1 > mem->size) return TRAP_MEMORY_OUT_OF_BOUNDS;

    int8_t byte_val = (int8_t)mem->data[addr];  // sign-extend via cast
    stack_push_i32(sp, (int32_t)byte_val);
    break;
}

case OP_I32_LOAD8_U: {
    uint32_t align = read_leb128_u32(&pc);
    uint32_t offset = read_leb128_u32(&pc);
    uint32_t addr = stack_pop_i32(sp) + offset;

    if (addr + 1 > mem->size) return TRAP_MEMORY_OUT_OF_BOUNDS;

    uint8_t byte_val = mem->data[addr];  // zero-extend via cast
    stack_push_i32(sp, (int32_t)(uint32_t)byte_val);
    break;
}
```

### 5.4 memory.grow 实现

> **wasm3 实现参考**：[m3_exec.h - op_MemGrow()](wasm3/source/m3_exec.h#L725) — 运行时 `memory.grow` 指令实现；[m3_env.c - ResizeMemory()](wasm3/source/m3_env.c#L262) — 实际的内存重分配逻辑。

```c
case OP_MEMORY_GROW: {
    // Operand: number of pages to grow
    int32_t delta_pages = stack_pop_i32(sp);
    int32_t old_page_count = (int32_t)mem->page_count;

    if (delta_pages < 0) {
        stack_push_i32(sp, -1);  // failure
        break;
    }

    uint64_t new_pages = (uint64_t)mem->page_count + delta_pages;

    // Check against maximum
    if (new_pages > mem->max_pages || new_pages > 65536) {
        stack_push_i32(sp, -1);  // failure: return -1
        break;
    }

    uint64_t new_size = new_pages * WASM_PAGE_SIZE;

    // Reallocate
    uint8_t *new_data = realloc(mem->data, new_size);
    if (new_data == NULL) {
        stack_push_i32(sp, -1);  // failure
        break;
    }

    // Zero-initialize new pages (required by spec)
    memset(new_data + mem->size, 0, new_size - mem->size);

    mem->data = new_data;
    mem->size = (uint32_t)new_size;
    mem->page_count = (uint32_t)new_pages;

    stack_push_i32(sp, old_page_count);  // success: return previous size
    break;
}
```

### 5.5 边界检查策略

有两种主流的边界检查方式：

#### 显式检查（Explicit Bounds Check）

```c
// Every memory access checks bounds
if (addr + size > mem->size) trap();
```

- 简单可靠
- 每次访问多一次比较和分支
- 适合解释器

#### Guard Page（信号处理）

```c
// Allocate extra guard pages after the memory region
// Set them as PROT_NONE (no access)
// On access violation, signal handler catches SIGSEGV/SIGBUS
mmap(mem->data + mem->max_size, GUARD_SIZE, PROT_NONE, ...);
```

- 零运行时开销（正常路径无额外指令）
- 需要操作系统支持（mmap + signal handler）
- 实现复杂，需要区分 Wasm trap 和真正的 bug
- 主要用于 JIT 编译器（如 V8、Wasmtime）

> **推荐**：解释器使用显式检查即可。Guard page 技术主要用于 JIT 编译器。

---

## 6. 表（Table）与间接调用

### 6.1 表的作用

表是 Wasm 中实现间接函数调用的机制。它本质上是一个引用数组（通常是 funcref）。

```
Table (funcref):
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ func[3] │ func[7] │  NULL   │ func[1] │ func[5] │
└─────────┴─────────┴─────────┴─────────┴─────────┘
  idx 0     idx 1     idx 2     idx 3     idx 4
```

### 6.2 表的初始化

表通过 Element Segment 初始化（在实例化阶段）：

```c
void init_table_from_elem_segment(ModuleInstance *inst, ElemSegment *seg) {
    // Active segment: has a table index and offset expression
    if (seg->mode == ELEM_ACTIVE) {
        uint32_t offset = eval_init_expr(seg->offset_expr, inst);
        WasmTable *table = inst->tables[seg->table_idx];

        // Bounds check
        if (offset + seg->count > table->size) {
            trap(TRAP_OUT_OF_BOUNDS_TABLE_ACCESS);
        }

        // Copy function references into table
        for (uint32_t i = 0; i < seg->count; i++) {
            uint32_t func_idx = seg->func_indices[i];
            table->elements[offset + i].ref = &inst->functions[func_idx];
        }
    }
}
```

### 6.3 call_indirect 的完整流程

```
call_indirect $type_idx $table_idx

1. Pop i32 index from stack
2. table[table_idx].elements[index] → get funcref
3. Check: funcref != NULL (else trap: uninitialized element)
4. Check: funcref.type == types[type_idx] (else trap: type mismatch)
5. Pop arguments from stack (based on type signature)
6. Call the function
7. Push results onto stack
```

这个运行时类型检查是 Wasm 安全性的关键——它防止了通过被篡改的函数指针调用错误类型的函数。

---

## 7. Trap 与错误处理

### 7.1 Trap 的种类

Wasm 规范定义了以下 trap 条件：

| Trap | 触发条件 |
|------|---------|
| `unreachable` | 执行 `unreachable` 指令 |
| `out of bounds memory access` | 内存访问越界 |
| `out of bounds table access` | 表访问越界 |
| `uninitialized element` | `call_indirect` 访问 NULL 表项 |
| `indirect call type mismatch` | `call_indirect` 类型不匹配 |
| `integer divide by zero` | 整数除以零 |
| `integer overflow` | `i32.trunc_f64_s` 等转换溢出 |
| `undefined element` | 表索引超出范围 |
| `call stack exhausted` | 调用栈溢出 |

### 7.2 Trap 的实现方式

**方式 A：返回值传播（推荐用于解释器）**

```c
typedef enum {
    RESULT_OK = 0,
    TRAP_UNREACHABLE,
    TRAP_MEMORY_OUT_OF_BOUNDS,
    TRAP_DIV_BY_ZERO,
    TRAP_INT_OVERFLOW,
    TRAP_INDIRECT_CALL_TYPE_MISMATCH,
    TRAP_UNDEFINED_ELEMENT,
    TRAP_UNINITIALIZED_ELEMENT,
    TRAP_CALL_STACK_EXHAUSTED,
    // ...
} WasmResult;

// Every function returns WasmResult, caller checks
WasmResult interpret(InterpState *state) {
    // ...
    case OP_I32_DIV_S: {
        int32_t b = stack_pop_i32(sp);
        int32_t a = stack_pop_i32(sp);
        if (b == 0) return TRAP_DIV_BY_ZERO;
        if (a == INT32_MIN && b == -1) return TRAP_INT_OVERFLOW;
        stack_push_i32(sp, a / b);
        break;
    }
}
```

**方式 B：setjmp/longjmp（wasm3 风格）**

```c
#include <setjmp.h>

typedef struct {
    jmp_buf trap_handler;
    WasmResult trap_reason;
} TrapContext;

// Set up trap handler
WasmResult execute_function(ModuleInstance *inst, uint32_t func_idx, ...) {
    TrapContext ctx;
    WasmResult result = setjmp(ctx.trap_handler);
    if (result != 0) {
        // Trap occurred, clean up and return
        return result;
    }
    // Normal execution
    interpret(inst, func_idx, &ctx);
    return RESULT_OK;
}

// Trigger trap from anywhere in the call chain
void trap(TrapContext *ctx, WasmResult reason) {
    ctx->trap_reason = reason;
    longjmp(ctx->trap_handler, reason);
}
```

setjmp/longjmp 的优势是不需要在每条指令后检查返回值，正常执行路径零开销。缺点是需要小心资源清理。

---

## 8. wasm3 M3 架构专题

> **wasm3 核心源码导航**：
> - [m3_compile.c](wasm3/source/m3_compile.c) — Wasm → M3 编译器（最大的源文件）
> - [m3_exec.h](wasm3/source/m3_exec.h) — 所有 M3 operation 的实现
> - [m3_exec_defs.h](wasm3/source/m3_exec_defs.h) — 执行引擎核心宏定义
> - [m3_code.c](wasm3/source/m3_code.c) — Code Page 内存管理
> - [m3_compile.h](wasm3/source/m3_compile.h) — 编译器数据结构

### 8.1 M3 的核心思想

wasm3 的 M3 架构可以理解为一个**元解释器（meta-interpreter）**：

1. **编译阶段**：将 Wasm 字节码转译为 M3 operation 序列
2. **执行阶段**：M3 operations 通过函数指针链式调用执行

这不是 JIT 编译（不生成机器码），而是一种**高级解释**——将 Wasm 的紧凑编码转换为更适合解释执行的中间表示。

### 8.2 M3 Operation 的结构

```
M3 Code Page (array of slots):
┌──────────────┬──────────┬──────────────┬──────────┬──────────────┬─────┐
│ op_Const32   │ 42       │ op_Const32   │ 10       │ op_Add_i32   │ ... │
│ (func ptr)   │ (imm)    │ (func ptr)   │ (imm)    │ (func ptr)   │     │
└──────────────┴──────────┴──────────────┴──────────┴──────────────┴─────┘
  ▲ pc points here
```

每个 slot 是一个 `void *` 大小的值，可以是：
- **函数指针**：指向一个 M3 operation 的 C 函数
- **立即数**：操作的参数（已从 LEB128 解码为原生值）

### 8.3 M3 Operation 的执行模型

> **wasm3 实现参考**：[m3_exec_defs.h - d_m3OpSig](wasm3/source/m3_exec_defs.h#L17) — operation 函数签名宏；[m3_exec_defs.h - nextOpDirect](wasm3/source/m3_exec_defs.h#L62) — 尾调用分发宏；[m3_exec_defs.h - RunCode](wasm3/source/m3_exec_defs.h#L66) — trampoline 入口。

```c
// Simplified M3 operation signature
// pc: program counter into the M3 code page
// sp: stack pointer
// mem: linear memory base
typedef const void * (*IM3Operation)(pc_t pc, u64 *sp, u8 *mem, ...);

// The NEXT_OP macro: tail-call to the next operation
// This is the heart of M3's threaded dispatch
#define nextOp()  (*(IM3Operation)(*pc))(pc + 1, sp, mem, ...)

// Example operations:

// Push a 32-bit constant
d_m3Op(Const32) {
    // Read immediate value from the code stream
    i32 value = *(i32 *)(pc);
    pc += sizeof(i32) / sizeof(void*);  // advance past immediate

    // Push onto stack
    *(i32 *)(sp) = value;
    sp++;

    return nextOp();  // tail-call to next operation
}

// i32 addition
d_m3Op(Add_i32) {
    sp--;
    *(i32 *)(sp - 1) += *(i32 *)(sp);
    return nextOp();
}

// local.get (with slot index as immediate)
d_m3Op(GetLocal_i32) {
    u32 slot = *(u32 *)(pc);
    pc += sizeof(u32) / sizeof(void*);

    // locals are at negative offsets from frame pointer
    *(i32 *)(sp) = *(i32 *)(sp - slot);
    sp++;

    return nextOp();
}
```

### 8.4 编译阶段：Wasm → M3

> **wasm3 实现参考**：[m3_compile.c - CompileFunction()](wasm3/source/m3_compile.c#L2832) — 函数编译入口；[m3_compile.c - CompileBlockStatements()](wasm3/source/m3_compile.c#L2649) — 逐条编译指令；[m3_compile.c - EmitOp()](wasm3/source/m3_compile.c#L55) — 发射 M3 operation 到 code page；[m3_compile.c - c_operations](wasm3/source/m3_compile.c#L2047) — 操作码到 M3OpInfo 的映射表。

wasm3 的编译器（`m3_compile.c`）逐条读取 Wasm 操作码，为每条指令生成对应的 M3 operation：

```c
// Simplified compilation flow
M3Result compile_function(M3Compilation *comp, M3Function *func) {
    const u8 *wasm = func->wasm_code;
    const u8 *end = wasm + func->wasm_code_size;

    while (wasm < end) {
        u8 opcode = *wasm++;

        switch (opcode) {
            case 0x41: { // i32.const
                i32 value = read_leb128_i32(&wasm);
                // Emit M3 operation: function pointer + immediate
                emit_op(comp, op_Const32);
                emit_i32(comp, value);
                break;
            }

            case 0x6A: { // i32.add
                emit_op(comp, op_Add_i32);
                break;
            }

            case 0x20: { // local.get
                u32 idx = read_leb128_u32(&wasm);
                // Convert local index to stack slot offset
                u32 slot = compute_slot_offset(comp, idx);
                emit_op(comp, op_GetLocal_i32);
                emit_u32(comp, slot);
                break;
            }

            // ... all other opcodes
        }
    }
}
```

### 8.5 M3 的寄存器优化

wasm3 的一个关键优化是**寄存器槽（register slot）**：将栈顶的值保留在 C 函数的参数/局部变量中（即 CPU 寄存器），而不是每次都写入内存中的栈。

```c
// Without register optimization:
d_m3Op(Add_i32) {
    sp--;
    *(i32*)(sp-1) += *(i32*)(sp);  // two memory accesses
    return nextOp();
}

// With register optimization (wasm3 actual approach):
// The top-of-stack value is kept in a register variable (_r0)
d_m3Op(Add_i32_rs) {
    // _r0 holds the top of stack (in a CPU register)
    // sp points to the second value
    _r0 = *(i32*)(sp - 1) + (i32)_r0;
    sp--;
    return nextOp();
}
```

这减少了内存访问次数，因为 `_r0` 通常会被编译器分配到 CPU 寄存器中。

### 8.6 M3 的控制流处理

> **wasm3 实现参考**：[m3_exec.h - op_Branch()](wasm3/source/m3_exec.h#L1240) — 无条件跳转；[m3_exec.h - op_BranchTable()](wasm3/source/m3_exec.h#L1275) — 分支表；[m3_exec.h - op_Loop()](wasm3/source/m3_exec.h#L1218) — 循环入口；[m3_compile.c - Compile_If()](wasm3/source/m3_compile.c#L1869) — if/else 编译；[m3_compile.c - Compile_LoopOrBlock()](wasm3/source/m3_compile.c#L1812) — block/loop 编译。

M3 中的控制流通过在 code page 中嵌入跳转目标来实现：

```c
// Branch: the target address is embedded as an immediate
d_m3Op(Branch) {
    pc_t target = *(pc_t *)(pc);  // read branch target
    pc = target;                   // jump
    return nextOp();
}

// Conditional branch
d_m3Op(BranchIf_i32) {
    pc_t target = *(pc_t *)(pc);
    pc += sizeof(pc_t) / sizeof(void*);

    i32 condition = (i32)_r0;
    sp--;  // pop condition

    if (condition) {
        pc = target;
    }
    // else: fall through to next op

    return nextOp();
}
```

### 8.7 M3 的尾调用要求

> **wasm3 实现参考**：[m3_exec_defs.h#L62](wasm3/source/m3_exec_defs.h#L62) — `nextOpDirect`/`nextOpImpl` 宏定义，这是尾调用的核心；[m3_exec_defs.h - RunCode()](wasm3/source/m3_exec_defs.h#L66) — trampoline 后备方案。

M3 的正确性**依赖于编译器的尾调用优化（TCO）**。每个 operation 通过 `return nextOp()` 调用下一个 operation——如果编译器不将其优化为尾调用（即跳转而非调用），C 调用栈会迅速溢出。

```c
// This MUST be compiled as a tail call:
return nextOp();
// Should generate: jmp *next_op  (not: call *next_op)
```

wasm3 通过以下方式确保 TCO：
- 所有 operation 函数具有相同的签名（参数类型和返回类型一致）
- 使用 `__attribute__((optimize("O3")))` 等编译器提示
- 在不支持 TCO 的平台上，使用 trampoline 模式作为后备

```c
// Trampoline fallback (when TCO is not available)
void trampoline_loop(pc_t pc, u64 *sp, u8 *mem) {
    while (pc) {
        IM3Operation op = (IM3Operation)(*pc);
        pc = op(pc + 1, sp, mem);
        // Each op returns the next pc instead of tail-calling
    }
}
```

---

## 9. 与 CLR 执行模型对比

### 9.1 栈机模型对比

| 特性 | Wasm | CLR (CIL) |
|------|------|-----------|
| **栈槽大小** | 固定 64-bit（i32 零扩展到 64-bit） | 可变（int32 = 4B, int64 = 8B, 引用 = 指针大小） |
| **类型系统** | 4 种数值 + 2 种引用 | 丰富的值类型 + 引用类型 + 泛型 |
| **栈上类型信息** | 运行时无类型标签（依赖验证） | 运行时可通过 GC 获取类型信息 |
| **操作数栈深度** | 编译时确定（验证保证） | 编译时确定（验证保证） |

### 9.2 内存模型对比

```
Wasm Memory Model:                    CLR Memory Model:
┌─────────────────────┐              ┌─────────────────────┐
│   Linear Memory     │              │    Managed Heap     │
│   (flat byte array) │              │  ┌───────────────┐  │
│   ┌───────────────┐ │              │  │ Object Header │  │
│   │ 0x00000000    │ │              │  │ Type Handle   │  │
│   │ ...           │ │              │  │ Fields...     │  │
│   │ raw bytes     │ │              │  └───────────────┘  │
│   │ ...           │ │              │  ┌───────────────┐  │
│   │ 0x0000FFFF    │ │              │  │ Another Obj   │  │
│   └───────────────┘ │              │  └───────────────┘  │
│   No GC, no headers │              │  GC managed         │
│   Manual layout      │              │  Auto layout        │
└─────────────────────┘              ├─────────────────────┤
                                     │   Value Type Stack  │
                                     │  (unboxed, no GC)   │
                                     └─────────────────────┘
```

| 特性 | Wasm | CLR |
|------|------|-----|
| **内存模型** | 单一线性内存（flat byte array） | 托管堆 + 值类型栈 + 非托管内存 |
| **内存安全** | 边界检查（每次访问） | 类型安全 + GC（无悬挂指针） |
| **内存布局** | 程序完全控制（手动 malloc） | 运行时控制（字段对齐、GC 压缩） |
| **垃圾回收** | 无（手动管理或 WASM GC 提案） | 分代 GC |
| **内存增长** | `memory.grow`（页粒度，64KB） | 堆自动扩展 |
| **多内存** | MVP 仅 1 个，多内存提案支持多个 | 多个堆（LOH、SOH、POH） |

### 9.3 函数调用对比

| 特性 | Wasm | CLR |
|------|------|-----|
| **直接调用** | `call funcidx` | `call method_token` |
| **间接调用** | `call_indirect` (通过 table) | `callvirt` (通过 vtable) / `calli` (函数指针) |
| **类型检查** | `call_indirect` 运行时全签名检查 | `callvirt` 通过继承层次保证 |
| **多返回值** | 原生支持（Wasm 2.0） | 不支持（用 out 参数或 tuple） |
| **异常处理** | trap（不可恢复） + 异常处理提案 | try/catch/finally（结构化异常） |
| **尾调用** | 尾调用提案（`return_call`） | `.tail` 前缀（JIT 可能忽略） |

### 9.4 执行策略对比

| 特性 | Wasm (wasm3) | CLR |
|------|-------------|-----|
| **主要策略** | Threaded code 解释 | JIT 编译（RyuJIT） |
| **备选策略** | Switch-dispatch | 解释器（.NET 8 引入） / AOT（NativeAOT） |
| **启动时间** | 快（编译为 threaded code 很快） | 较慢（JIT 编译开销） |
| **峰值性能** | 中等（解释器上限） | 高（原生机器码） |
| **代码缓存** | M3 operation 序列 | JIT 编译后的机器码 |
| **内存占用** | 低（适合嵌入式） | 高（JIT 编译器 + 元数据） |

### 9.5 对你实现的启示

如果你已经实现过 CLR 执行引擎，以下是关键的思维转换：

1. **没有 GC**：Wasm 的线性内存就是一个 `byte[]`，你不需要实现垃圾回收器。内存管理完全由 Wasm 程序自己负责（通常通过编译器生成的 malloc/free）。

2. **没有对象头**：线性内存中没有类型信息、没有同步块、没有 GC 标记位。所有数据都是原始字节。

3. **更简单的类型系统**：只有 4 种数值类型 + 2 种引用类型，不需要处理继承、接口、泛型。

4. **结构化控制流**：这是最大的区别。CLR 的 `br` 可以跳转到任意偏移，而 Wasm 的 `br` 只能跳转到包围它的 block/loop 的边界。这使得验证更简单，但实现控制流时需要维护一个控制帧栈。

5. **确定性**：Wasm 的执行是完全确定性的（除了 NaN 位模式和资源限制）。不需要处理线程同步、内存模型等复杂问题（除非实现 Threads 提案）。

---

## 10. 性能优化技巧

### 10.1 解释器级优化

#### 超级指令（Superinstructions）

将常见的指令序列合并为单个"超级指令"：

```c
// Common pattern: local.get + i32.add
// Instead of two separate dispatches, combine into one:
d_m3Op(GetLocal_Add_i32) {
    u32 slot = *(u32 *)(pc);
    pc += sizeof(u32) / sizeof(void*);

    *(i32 *)(sp - 1) += *(i32 *)(sp - 1 - slot);
    return nextOp();
}
```

#### 常量折叠

在编译阶段（Wasm → M3）识别常量表达式并预计算：

```c
// Wasm: i32.const 3, i32.const 4, i32.add
// Can be compiled to: i32.const 7
```

### 10.2 内存访问优化

#### 快速路径边界检查

```c
// Instead of checking every access:
if (addr + size > mem->size) trap();

// Pre-compute the check boundary:
uint8_t *mem_end = mem->data + mem->size;

// Fast path (common case: within bounds)
if (__builtin_expect(mem->data + addr + size <= mem_end, 1)) {
    // fast path: direct access
    value = *(int32_t *)(mem->data + addr);
} else {
    // slow path: trap
    return TRAP_MEMORY_OUT_OF_BOUNDS;
}
```

#### 内存基址缓存

将内存基址指针保存在局部变量中（CPU 寄存器），避免每次通过结构体指针间接访问：

```c
// At the start of the interpret loop:
uint8_t *mem_base = inst->memories[0]->data;
uint32_t mem_size = inst->memories[0]->size;

// Use mem_base directly in load/store operations
// Note: must update after memory.grow!
```

### 10.3 栈操作优化

#### 栈顶缓存（Top-of-Stack Caching）

将栈顶的一个或两个值保留在 C 局部变量中：

```c
// Without TOS caching:
case OP_I32_ADD:
    sp--;
    (sp-1)->i32 += sp->i32;  // 2 memory reads + 1 write
    break;

// With TOS caching (tos holds top of stack):
case OP_I32_ADD:
    tos.i32 = (--sp)->i32 + tos.i32;  // 1 memory read
    break;
```

### 10.4 分支预测友好

```c
// Use likely/unlikely hints for common paths
#define likely(x)   __builtin_expect(!!(x), 1)
#define unlikely(x) __builtin_expect(!!(x), 0)

// Memory bounds check (usually passes)
if (unlikely(addr + 4 > mem->size)) {
    return TRAP_MEMORY_OUT_OF_BOUNDS;
}

// Branch instruction (depends on workload)
if (likely(condition)) {
    pc = branch_target;
}
```

### 10.5 编译阶段优化清单

| 优化 | 描述 | 复杂度 |
|------|------|--------|
| LEB128 预解码 | 编译时解码所有 LEB128 立即数 | 低 |
| 分支目标预计算 | 编译时计算所有 br 的跳转目标地址 | 低 |
| 局部变量槽映射 | 将局部变量索引转换为栈偏移 | 低 |
| 常量折叠 | 合并连续的常量运算 | 中 |
| 超级指令 | 合并常见指令序列 | 中 |
| 寄存器分配 | 将栈操作转换为寄存器操作 | 高 |
| 死代码消除 | 移除 unreachable 后的代码 | 中 |

---

## 小结

本章覆盖了 Wasm 执行引擎的核心设计：

1. **三种执行策略**：从简单的 AST 遍历到高效的 threaded code，理解了各自的权衡
2. **运行时数据结构**：操作数栈、调用帧、线性内存、表、全局变量的设计方案
3. **函数调用机制**：直接调用、间接调用、宿主函数调用、多返回值处理
4. **内存操作**：字节序、对齐、边界检查、memory.grow
5. **wasm3 M3 架构**：threaded code 的编译与执行、寄存器优化、尾调用要求
6. **CLR 对比**：帮助你从已有经验快速迁移到 Wasm 执行模型

> **下一步**：阅读 [第五章：模块实例化与链接](05-instantiation.md)，了解如何将解析后的模块变为可执行的实例。
