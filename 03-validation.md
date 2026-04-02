
# Wasm 验证（Validation）机制

> **前置阅读**：[01-binary-format.md](./01-binary-format.md)、[02-instruction-set.md](./02-instruction-set.md)

WebAssembly 的一个核心设计原则是 **"验证先于执行"**。任何 Wasm 模块在被实例化和执行之前，必须通过完整的静态验证。验证确保模块的结构完整性和类型安全性，使得执行引擎可以省略大量运行时检查（如操作数类型检查），从而提升性能。

对于有 CLR 经验的你来说，这类似于 CLR 在 JIT 编译前对 IL 代码进行的验证（verification），但 Wasm 的验证算法更简洁——它被设计为 **单遍前向扫描（single-pass forward scan）**，时间复杂度为 O(n)。

---

## 1. 验证的整体架构

> **wasm3 实现参考**：wasm3 将验证和编译合并在同一个过程中——编译器在生成 M3 operation 的同时进行类型检查。核心实现见 [m3_compile.c - CompileBlockStatements()](wasm3/source/m3_compile.c#L2568) — 逐条指令编译+验证；[m3_compile.c - Push()](wasm3/source/m3_compile.c#L542) / [Pop()](wasm3/source/m3_compile.c#L584) — 编译时栈模拟（等价于类型栈操作）；[m3_compile.h - M3CompilationScope](wasm3/source/m3_compile.h#L54) — 控制帧栈结构。

Wasm 验证分为三个层次，由外到内依次进行：

```
┌─────────────────────────────────────────┐
│         Module-level Validation         │  ← Layer 1: structure & index bounds
│  ┌───────────────────────────────────┐  │
│  │    Function-level Validation      │  │  ← Layer 2: locals & signature
│  │  ┌─────────────────────────────┐  │  │
│  │  │ Instruction-level Type Check│  │  │  ← Layer 3: operand stack typing
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 1.1 Layer 1：模块级验证（Module-level Validation）

模块级验证在解析完二进制格式后立即进行，检查模块的整体结构合法性：

**结构完整性检查：**
- Magic number 必须是 `\0asm`（`0x00 0x61 0x73 0x6D`）
- Version 必须是 `1`（`0x01 0x00 0x00 0x00`）
- Section 必须按规定顺序出现（Custom Section 除外，可出现在任意位置）
- 每个 Section 的声明大小必须与实际内容大小一致
- Function Section 中的函数数量必须与 Code Section 中的函数体数量一致

**索引范围检查：**
- 所有 type index 必须在 Type Section 的范围内
- 所有 function index 必须在函数索引空间内（imports + local functions）
- 所有 table index、memory index、global index 同理
- Import Section 中引用的类型索引必须有效
- Export Section 中引用的各类索引必须有效
- Start Section 引用的函数索引必须有效，且该函数签名必须是 `[] → []`（无参数无返回值）
- Element Section 中的表索引和函数索引必须有效
- Data Section 中的内存索引必须有效

**限制检查：**
- 当前规范中，一个模块最多只能有 1 个 Memory 和 1 个 Table（MVP 限制）
- Memory 的 limits：min ≤ max（如果有 max），且 max ≤ 65536（页数上限，即 4GB）
- Table 的 limits：min ≤ max（如果有 max）

**与 CLR 的类比：** 这一层类似于 CLR 加载 PE 文件时对 Metadata 表的结构验证——检查 token 引用是否越界、表之间的交叉引用是否一致等。

### 1.2 Layer 2：函数级验证（Function-level Validation）

对每个函数体进行独立验证：

- 函数的类型索引必须指向一个有效的函数类型签名
- 局部变量声明（locals）的类型必须是合法的值类型
- 局部变量总数（参数 + 声明的局部变量）不能超过实现限制
- 函数体的表达式（expression）必须以 `end`（0x0B）结尾
- 函数体的指令序列必须通过指令级类型检查（Layer 3）

### 1.3 Layer 3：指令级类型检查（Instruction-level Type Checking）

这是验证的核心，也是最复杂的部分。它通过模拟操作数栈的类型变化，逐条检查每条指令的类型安全性。详见下文第 2-4 节。

---

## 2. 类型检查算法核心

> **wasm3 实现参考**：wasm3 的编译时栈模拟实现了等价的类型检查。参见 [m3_compile.h - M3Compilation](wasm3/source/m3_compile.h#L54) — `wasmStack`/`typeStack` 字段对应类型栈；[m3_compile.c - Push()](wasm3/source/m3_compile.c#L542) — 压栈时记录类型；[m3_compile.c - Pop()](wasm3/source/m3_compile.c#L584) — 出栈时检查类型。

### 2.1 算法概述

Wasm 的指令级类型检查是一个 **抽象解释（abstract interpretation）** 过程：

- 维护一个 **类型栈（type stack）**，记录操作数栈上每个槽位的类型
- 维护一个 **控制帧栈（control frame stack）**，跟踪嵌套的控制结构
- 逐条扫描指令，根据每条指令的类型规则更新类型栈
- 如果任何指令违反类型规则，验证失败

关键特性：**单遍前向扫描**。不需要回溯，不需要迭代到不动点。这是因为 Wasm 的结构化控制流保证了每个控制结构的入口类型和出口类型在进入之前就已知。

### 2.2 核心数据结构

```c
// Type stack: tracks the type of each operand on the stack
typedef struct {
    ValType* types;     // dynamic array of value types
    int      top;       // stack pointer
    int      capacity;
} TypeStack;

// Control frame: tracks a nested control structure
typedef struct {
    uint8_t  opcode;        // block / loop / if / function body
    FuncType block_type;    // the type signature of this block [params] -> [results]
    int      height;        // type stack height at entry (before params)
    bool     unreachable;   // whether we've seen unreachable/br/return in this frame
} CtrlFrame;

// Control frame stack
typedef struct {
    CtrlFrame* frames;
    int        top;
    int        capacity;
} CtrlStack;

// Validation context
typedef struct {
    TypeStack   operands;    // the operand type stack
    CtrlStack   controls;    // the control frame stack
    ValType*    locals;      // types of all locals (params + declared locals)
    int         num_locals;
    FuncType*   types;       // module's type section (all function signatures)
    int         num_types;
    // ... other module-level info (functions, tables, memories, globals)
} ValidationContext;
```

### 2.3 类型栈操作

类型栈的基本操作：

```c
// Push a type onto the operand stack
void push_operand(ValidationContext* ctx, ValType type) {
    // simply push the type
    ctx->operands.types[ctx->operands.top++] = type;
}

// Pop a type from the operand stack
// 'expect' is the expected type; VAL_UNKNOWN means "accept any type"
ValType pop_operand(ValidationContext* ctx, ValType expect) {
    CtrlFrame* frame = &ctx->controls.frames[ctx->controls.top - 1];

    // If stack is at the control frame's entry height
    if (ctx->operands.top == frame->height) {
        if (frame->unreachable) {
            // In unreachable code, we can pop any type (stack polymorphism)
            return (expect != VAL_UNKNOWN) ? expect : VAL_UNKNOWN;
        }
        // Error: stack underflow
        validation_error("operand stack underflow");
    }

    ValType actual = ctx->operands.types[--ctx->operands.top];

    if (expect != VAL_UNKNOWN && actual != VAL_UNKNOWN && actual != expect) {
        // Type mismatch
        validation_error("type mismatch: expected %s, got %s", 
                         type_name(expect), type_name(actual));
    }

    return (actual != VAL_UNKNOWN) ? actual : expect;
}
```

**关键点：** `pop_operand` 中对 `frame->unreachable` 的处理是栈多态性的核心（详见第 4 节）。

### 2.4 指令类型规则示例

每条指令都有明确的类型规则，描述它从栈上弹出什么类型、压入什么类型：

```
Instruction         Pop             Push            Rule
─────────────────────────────────────────────────────────────
i32.const n         -               i32             push i32
i64.const n         -               i64             push i64
i32.add             i32, i32        i32             pop 2 i32, push i32
i32.eqz             i32             i32             pop i32, push i32 (boolean)
i64.extend_i32_s    i32             i64             pop i32, push i64
i32.load            i32             i32             pop i32 (addr), push i32
i32.store           i32, i32        -               pop i32 (value), pop i32 (addr)
local.get x         -               locals[x]       push type of local x
local.set x         locals[x]       -               pop type matching local x
drop                any             -               pop any single value
select              any, any, i32   any             pop i32, pop 2 same type, push that type
```

验证器中的实现模式：

```c
void validate_instruction(ValidationContext* ctx, uint8_t opcode, ...) {
    switch (opcode) {
        case OP_I32_CONST:
            push_operand(ctx, VAL_I32);
            break;

        case OP_I32_ADD:
            pop_operand(ctx, VAL_I32);
            pop_operand(ctx, VAL_I32);
            push_operand(ctx, VAL_I32);
            break;

        case OP_I32_LOAD: {
            // check memory exists (memidx 0)
            // check alignment <= natural alignment (4 for i32)
            pop_operand(ctx, VAL_I32);   // address
            push_operand(ctx, VAL_I32);  // loaded value
            break;
        }

        case OP_LOCAL_GET: {
            uint32_t idx = read_local_index();
            if (idx >= ctx->num_locals) validation_error("invalid local index");
            push_operand(ctx, ctx->locals[idx]);
            break;
        }

        case OP_LOCAL_SET: {
            uint32_t idx = read_local_index();
            if (idx >= ctx->num_locals) validation_error("invalid local index");
            pop_operand(ctx, ctx->locals[idx]);
            break;
        }

        case OP_DROP:
            pop_operand(ctx, VAL_UNKNOWN);  // accept any type
            break;

        case OP_SELECT: {
            pop_operand(ctx, VAL_I32);              // condition
            ValType t1 = pop_operand(ctx, VAL_UNKNOWN);
            ValType t2 = pop_operand(ctx, t1);      // must match t1
            push_operand(ctx, (t2 != VAL_UNKNOWN) ? t2 : t1);
            break;
        }
        // ... more instructions
    }
}
```

---

## 3. 控制帧栈与控制流验证

控制流验证是 Wasm 验证中最精妙的部分。每个控制结构（block/loop/if/function body）都会在控制帧栈上压入一个帧，用于跟踪类型约束。

### 3.1 控制帧的生命周期

```
push_ctrl(opcode, block_type)    ← entering a block/loop/if
    ... validate instructions inside ...
pop_ctrl()                       ← reaching the 'end' of the structure
```

```c
// Enter a new control structure
void push_ctrl(ValidationContext* ctx, uint8_t opcode, FuncType block_type) {
    CtrlFrame frame = {
        .opcode      = opcode,
        .block_type  = block_type,
        .height      = ctx->operands.top,  // record stack height BEFORE params
        .unreachable = false
    };

    // Push the block's parameter types onto the operand stack
    // (block params are consumed from the outer stack and become the block's inputs)
    for (int i = 0; i < block_type.num_params; i++) {
        // Note: params were already on the stack from outer context
        // We just record the height before them
    }

    // Actually, the correct logic:
    // 1. Pop the block's parameter types from the operand stack (validate they match)
    // 2. Record the stack height (this is the "label height")
    // 3. Push the parameter types back (they become the block's initial stack)

    // Pop params (in reverse order)
    for (int i = block_type.num_params - 1; i >= 0; i--) {
        pop_operand(ctx, block_type.params[i]);
    }

    frame.height = ctx->operands.top;  // height AFTER popping params

    ctx->controls.frames[ctx->controls.top++] = frame;

    // Push params back as the block's initial operand types
    for (int i = 0; i < block_type.num_params; i++) {
        push_operand(ctx, block_type.params[i]);
    }
}

// Leave a control structure (at 'end')
FuncType pop_ctrl(ValidationContext* ctx) {
    if (ctx->controls.top == 0) validation_error("control stack underflow");

    CtrlFrame* frame = &ctx->controls.frames[ctx->controls.top - 1];

    // Pop the block's result types from the operand stack
    for (int i = frame->block_type.num_results - 1; i >= 0; i--) {
        pop_operand(ctx, frame->block_type.results[i]);
    }

    // Stack height must return to the entry height
    if (ctx->operands.top != frame->height) {
        validation_error("stack height mismatch at end of block");
    }

    ctx->controls.top--;

    // Push the block's result types onto the outer operand stack
    FuncType bt = frame->block_type;
    for (int i = 0; i < bt.num_results; i++) {
        push_operand(ctx, bt.results[i]);
    }

    return bt;
}
```

### 3.2 block / loop / if 的验证

#### block

```wasm
block [t1*] -> [t2*]     ;; block_type = [t1*] -> [t2*]
  instr*
end
```

验证流程：
1. 从操作数栈弹出参数类型 `[t1*]`
2. `push_ctrl(BLOCK, [t1*] -> [t2*])`，将参数类型压回栈
3. 验证内部指令序列
4. 在 `end` 处：`pop_ctrl()`，检查栈上有结果类型 `[t2*]`

**br 跳转到 block 的标签时，目标类型是 block 的 result types `[t2*]`**（跳到 block 的末尾）。

#### loop

```wasm
loop [t1*] -> [t2*]
  instr*
end
```

验证流程与 block 相同，但关键区别在于：

**br 跳转到 loop 的标签时，目标类型是 loop 的 param types `[t1*]`**（跳回 loop 的开头）。

这是 block 和 loop 在验证中的核心区别：

```c
// Get the types expected by a branch to label at depth 'depth'
FuncType label_types(ValidationContext* ctx, uint32_t depth) {
    CtrlFrame* frame = &ctx->controls.frames[ctx->controls.top - 1 - depth];
    if (frame->opcode == OP_LOOP) {
        // branch to loop = jump to beginning, need param types
        return (FuncType){ .params = frame->block_type.params,
                           .num_params = frame->block_type.num_params,
                           // For label_types, we return these as "results" 
                           // because br pops them like results
                           .results = frame->block_type.params,
                           .num_results = frame->block_type.num_params };
    } else {
        // branch to block/if = jump to end, need result types
        return (FuncType){ .results = frame->block_type.results,
                           .num_results = frame->block_type.num_results };
    }
}
```

#### if / else

```wasm
if [t1*] -> [t2*]
  instr*        ;; "then" branch
else
  instr*        ;; "else" branch
end
```

验证流程：
1. 弹出 `i32`（条件值）
2. 弹出参数类型 `[t1*]`
3. `push_ctrl(IF, [t1*] -> [t2*])`，压回参数
4. 验证 "then" 分支的指令
5. 在 `else` 处：
   - 检查栈上有结果类型 `[t2*]`（then 分支的输出）
   - 重置操作数栈到帧的 entry height
   - 压入参数类型 `[t1*]`（else 分支的输入）
   - 标记帧为 `ELSE`
6. 在 `end` 处：`pop_ctrl()`
7. **特殊规则**：如果没有 `else` 分支，则 block_type 必须是 `[t*] -> [t*]`（参数和结果相同），因为 "不执行 if" 等价于 "什么都不做"

### 3.3 br / br_if / br_table 的验证

#### br（无条件跳转）

```c
case OP_BR: {
    uint32_t depth = read_u32_leb128();
    if (depth >= ctx->controls.top) validation_error("invalid branch depth");

    // Get the label types for the target
    ValType* label_tys;
    int num_label_tys;
    get_label_types(ctx, depth, &label_tys, &num_label_tys);

    // Pop the label types from the operand stack
    for (int i = num_label_tys - 1; i >= 0; i--) {
        pop_operand(ctx, label_tys[i]);
    }

    // Mark current frame as unreachable (code after br is dead)
    mark_unreachable(ctx);
    break;
}
```

#### br_if（条件跳转）

```c
case OP_BR_IF: {
    uint32_t depth = read_u32_leb128();
    if (depth >= ctx->controls.top) validation_error("invalid branch depth");

    pop_operand(ctx, VAL_I32);  // condition

    ValType* label_tys;
    int num_label_tys;
    get_label_types(ctx, depth, &label_tys, &num_label_tys);

    // Pop and re-push label types (branch may or may not be taken)
    for (int i = num_label_tys - 1; i >= 0; i--) {
        pop_operand(ctx, label_tys[i]);
    }
    for (int i = 0; i < num_label_tys; i++) {
        push_operand(ctx, label_tys[i]);
    }
    // NOT unreachable — execution may continue to next instruction
    break;
}
```

**关键区别**：`br` 之后标记为 unreachable，`br_if` 之后不标记（因为条件可能为 false，继续执行）。

#### br_table（跳转表）

```c
case OP_BR_TABLE: {
    uint32_t num_targets = read_u32_leb128();
    uint32_t* targets = read_n_u32(num_targets);
    uint32_t default_target = read_u32_leb128();

    pop_operand(ctx, VAL_I32);  // index

    // All targets (including default) must have the same label arity
    ValType* default_tys;
    int num_default_tys;
    get_label_types(ctx, default_target, &default_tys, &num_default_tys);

    for (uint32_t i = 0; i < num_targets; i++) {
        ValType* target_tys;
        int num_target_tys;
        get_label_types(ctx, targets[i], &target_tys, &num_target_tys);

        // Arity must match default
        if (num_target_tys != num_default_tys) {
            validation_error("br_table target arity mismatch");
        }
        // Types must be compatible
        for (int j = 0; j < num_target_tys; j++) {
            if (target_tys[j] != default_tys[j]) {
                validation_error("br_table target type mismatch");
            }
        }
    }

    // Pop the label types
    for (int i = num_default_tys - 1; i >= 0; i--) {
        pop_operand(ctx, default_tys[i]);
    }

    mark_unreachable(ctx);  // always branches
    break;
}
```

### 3.4 return 的验证

`return` 等价于 `br` 到最外层的函数帧：

```c
case OP_RETURN: {
    // The function frame is at the bottom of the control stack (index 0)
    // depth = controls.top - 1 (branch to the outermost frame)
    uint32_t depth = ctx->controls.top - 1;

    CtrlFrame* func_frame = &ctx->controls.frames[0];
    // Pop the function's result types
    for (int i = func_frame->block_type.num_results - 1; i >= 0; i--) {
        pop_operand(ctx, func_frame->block_type.results[i]);
    }

    mark_unreachable(ctx);
    break;
}
```

### 3.5 call / call_indirect 的验证

```c
case OP_CALL: {
    uint32_t func_idx = read_u32_leb128();
    FuncType* ft = get_function_type(ctx, func_idx);

    // Pop parameters (in reverse order)
    for (int i = ft->num_params - 1; i >= 0; i--) {
        pop_operand(ctx, ft->params[i]);
    }
    // Push results
    for (int i = 0; i < ft->num_results; i++) {
        push_operand(ctx, ft->results[i]);
    }
    break;
}

case OP_CALL_INDIRECT: {
    uint32_t type_idx = read_u32_leb128();
    uint32_t table_idx = read_u32_leb128();  // must be 0 in MVP

    if (type_idx >= ctx->num_types) validation_error("invalid type index");
    // table must exist and be of funcref type
    FuncType* ft = &ctx->types[type_idx];

    pop_operand(ctx, VAL_I32);  // table index operand

    // Pop parameters, push results (same as call)
    for (int i = ft->num_params - 1; i >= 0; i--) {
        pop_operand(ctx, ft->params[i]);
    }
    for (int i = 0; i < ft->num_results; i++) {
        push_operand(ctx, ft->results[i]);
    }
    break;
}
```

---

## 4. 栈多态性（Stack Polymorphism）

栈多态性是 Wasm 验证中最微妙也最容易出错的部分。理解它对于正确实现验证器至关重要。

### 4.1 什么是栈多态性？

当执行流到达一个 **必然不可达** 的点时（例如 `br`、`return`、`unreachable` 指令之后），后续的指令在运行时永远不会被执行。但这些指令仍然存在于字节码中，验证器仍然需要处理它们。

在不可达代码中，操作数栈的行为变得 **多态**：
- 可以弹出任意类型（即使栈为空）
- 弹出的类型自动匹配期望的类型

### 4.2 为什么需要栈多态性？

考虑以下合法的 Wasm 代码：

```wasm
(func (result i32)
  block (result i32)
    i32.const 1
    br 0            ;; unconditional branch to end of block, carrying i32
    i32.const 2     ;; dead code — but still valid!
    i32.add         ;; dead code — needs two i32 on stack, but only one exists
                    ;; without polymorphism, this would fail validation
  end
)
```

在 `br 0` 之后，代码不可达。`i32.add` 需要两个 `i32`，但栈上只有一个 `i32.const 2` 压入的值。如果没有栈多态性，验证会失败。但这段代码是完全合法的——不可达代码不需要严格的类型检查。

### 4.3 unreachable 标记的实现

```c
void mark_unreachable(ValidationContext* ctx) {
    CtrlFrame* frame = &ctx->controls.frames[ctx->controls.top - 1];

    // Reset operand stack to the frame's entry height
    // This effectively "forgets" everything pushed in unreachable code
    ctx->operands.top = frame->height;

    // Mark the frame as unreachable
    frame->unreachable = true;
}
```

当 `unreachable` 为 true 时，`pop_operand` 的行为改变（回顾 2.3 节的代码）：
- 如果栈已经到达帧的 entry height（即"空"了），不报错，而是返回期望的类型
- 这使得不可达代码中的任何指令序列都能通过类型检查

### 4.4 unreachable 状态的重置

`unreachable` 状态在以下情况下被重置：

1. **遇到 `else`**：then 分支可能不可达，但 else 分支是新的开始
2. **遇到 `end`**：控制结构结束，外层代码恢复可达性（除非外层也是不可达的）

```c
case OP_ELSE: {
    CtrlFrame* frame = &ctx->controls.frames[ctx->controls.top - 1];
    if (frame->opcode != OP_IF) validation_error("else without matching if");

    // Check that 'then' branch produced the right result types
    for (int i = frame->block_type.num_results - 1; i >= 0; i--) {
        pop_operand(ctx, frame->block_type.results[i]);
    }

    // Reset for 'else' branch
    ctx->operands.top = frame->height;
    frame->unreachable = false;  // ← reset!
    frame->opcode = OP_ELSE;     // mark that we've seen else

    // Push params for else branch
    for (int i = 0; i < frame->block_type.num_params; i++) {
        push_operand(ctx, frame->block_type.params[i]);
    }
    break;
}
```

### 4.5 栈多态性的边界情况

**情况 1：unreachable 指令本身**

```wasm
unreachable     ;; marks everything after as unreachable
i32.add         ;; valid! polymorphic stack provides two i32
```

**情况 2：嵌套的不可达代码**

```wasm
block (result i32)
  unreachable
  block (result f64)
    ;; still unreachable
    f64.const 1.0   ;; valid
  end               ;; this end pops f64, pushes f64 result
                    ;; but we're still in outer unreachable context
  ;; stack is polymorphic, so the f64 from inner block is fine
  ;; and we can still produce an i32 for the outer block
end
```

**情况 3：br_if 不设置 unreachable**

```wasm
block (result i32)
  i32.const 1
  br_if 0           ;; conditional — might NOT branch
  ;; NOT unreachable! must still have valid types
  i32.const 2       ;; this must be here to provide the i32 result
end
```

### 4.6 与 CLR 的对比

| 方面 | Wasm | CLR IL |
|------|------|--------|
| 不可达代码处理 | 栈多态性：不可达代码中栈可以弹出任意类型 | 类似：不可达代码不参与验证（但 CLR 的处理更复杂，因为有非结构化跳转） |
| 检测方式 | 通过 `unreachable` 标志在控制帧上传播 | 通过控制流图的可达性分析 |
| 复杂度 | 简单——结构化控制流保证单遍即可确定 | 更复杂——需要构建 CFG 并进行可达性分析 |

---

## 5. 完整验证流程伪代码

将以上所有部分组合，一个函数体的完整验证流程如下：

```c
Result validate_function(Module* module, uint32_t func_idx) {
    FuncType* func_type = get_func_type(module, func_idx);
    CodeEntry* code = get_func_code(module, func_idx);

    // Initialize validation context
    ValidationContext ctx;
    init_validation_context(&ctx, module);

    // Set up locals: params + declared locals
    setup_locals(&ctx, func_type, code);

    // Push the function body as the outermost control frame
    // The function body is treated as a block with the function's signature
    push_ctrl(&ctx, OP_FUNCTION, *func_type);

    // Iterate through all instructions
    ByteReader reader;
    init_reader(&reader, code->body, code->body_size);

    while (reader.pos < reader.end) {
        uint8_t opcode = read_byte(&reader);

        switch (opcode) {
            // === Control Flow ===
            case OP_UNREACHABLE:
                mark_unreachable(&ctx);
                break;

            case OP_NOP:
                break;

            case OP_BLOCK: {
                FuncType bt = read_block_type(&reader, module);
                // Pop params from outer stack, validate types
                for (int i = bt.num_params - 1; i >= 0; i--)
                    pop_operand(&ctx, bt.params[i]);
                push_ctrl(&ctx, OP_BLOCK, bt);
                // Push params as block's initial stack
                for (int i = 0; i < bt.num_params; i++)
                    push_operand(&ctx, bt.params[i]);
                break;
            }

            case OP_LOOP: {
                FuncType bt = read_block_type(&reader, module);
                for (int i = bt.num_params - 1; i >= 0; i--)
                    pop_operand(&ctx, bt.params[i]);
                push_ctrl(&ctx, OP_LOOP, bt);
                for (int i = 0; i < bt.num_params; i++)
                    push_operand(&ctx, bt.params[i]);
                break;
            }

            case OP_IF: {
                FuncType bt = read_block_type(&reader, module);
                pop_operand(&ctx, VAL_I32);  // condition
                for (int i = bt.num_params - 1; i >= 0; i--)
                    pop_operand(&ctx, bt.params[i]);
                push_ctrl(&ctx, OP_IF, bt);
                for (int i = 0; i < bt.num_params; i++)
                    push_operand(&ctx, bt.params[i]);
                break;
            }

            case OP_ELSE:
                validate_else(&ctx);
                break;

            case OP_END: {
                FuncType bt = pop_ctrl(&ctx);
                // Results are already pushed by pop_ctrl
                if (ctx.controls.top == 0) {
                    // We've closed the function body frame
                    // Validation complete for this function
                    goto done;
                }
                break;
            }

            case OP_BR:
                validate_br(&ctx, &reader);
                break;

            case OP_BR_IF:
                validate_br_if(&ctx, &reader);
                break;

            case OP_BR_TABLE:
                validate_br_table(&ctx, &reader);
                break;

            case OP_RETURN:
                validate_return(&ctx);
                break;

            case OP_CALL:
                validate_call(&ctx, &reader);
                break;

            case OP_CALL_INDIRECT:
                validate_call_indirect(&ctx, &reader);
                break;

            // === Numeric Instructions ===
            case OP_I32_CONST:
                read_i32_leb128(&reader);  // consume immediate
                push_operand(&ctx, VAL_I32);
                break;

            case OP_I64_CONST:
                read_i64_leb128(&reader);
                push_operand(&ctx, VAL_I64);
                break;

            case OP_F32_CONST:
                read_f32(&reader);
                push_operand(&ctx, VAL_F32);
                break;

            case OP_F64_CONST:
                read_f64(&reader);
                push_operand(&ctx, VAL_F64);
                break;

            // Binary i32 operations: [i32 i32] -> [i32]
            case OP_I32_ADD: case OP_I32_SUB: case OP_I32_MUL:
            case OP_I32_DIV_S: case OP_I32_DIV_U:
            case OP_I32_REM_S: case OP_I32_REM_U:
            case OP_I32_AND: case OP_I32_OR: case OP_I32_XOR:
            case OP_I32_SHL: case OP_I32_SHR_S: case OP_I32_SHR_U:
            case OP_I32_ROTL: case OP_I32_ROTR:
                pop_operand(&ctx, VAL_I32);
                pop_operand(&ctx, VAL_I32);
                push_operand(&ctx, VAL_I32);
                break;

            // ... (similar patterns for i64, f32, f64 operations)

            // === Variable Instructions ===
            case OP_LOCAL_GET: {
                uint32_t idx = read_u32_leb128(&reader);
                validate_local_index(&ctx, idx);
                push_operand(&ctx, ctx.locals[idx]);
                break;
            }

            case OP_LOCAL_SET: {
                uint32_t idx = read_u32_leb128(&reader);
                validate_local_index(&ctx, idx);
                pop_operand(&ctx, ctx.locals[idx]);
                break;
            }

            case OP_LOCAL_TEE: {
                uint32_t idx = read_u32_leb128(&reader);
                validate_local_index(&ctx, idx);
                pop_operand(&ctx, ctx.locals[idx]);
                push_operand(&ctx, ctx.locals[idx]);
                break;
            }

            // === Memory Instructions ===
            case OP_I32_LOAD: {
                MemArg memarg = read_memarg(&reader);
                validate_memarg(memarg, 2);  // natural alignment = 4 bytes = 2^2
                validate_memory_exists(&ctx);
                pop_operand(&ctx, VAL_I32);   // address
                push_operand(&ctx, VAL_I32);  // result
                break;
            }

            // ... more instructions

            default:
                validation_error("unknown opcode: 0x%02X", opcode);
        }
    }

done:
    // Final check: control stack should be empty
    if (ctx.controls.top != 0) {
        validation_error("unclosed control structures");
    }

    cleanup_validation_context(&ctx);
    return RESULT_OK;
}
```

---

## 6. 常量表达式验证（Constant Expressions）

某些上下文中只允许 **常量表达式（constant expressions）**，它们在编译时求值：

- Global 变量的初始化值
- Element Segment 的偏移量
- Data Segment 的偏移量

常量表达式只允许以下指令：

| 指令 | 说明 |
|------|------|
| `*.const` | 数值常量（i32.const, i64.const, f32.const, f64.const） |
| `global.get` | 仅限引用 **imported** 且 **immutable** 的全局变量 |
| `ref.null` | 空引用 |
| `ref.func` | 函数引用 |
| `end` | 表达式结束 |

验证规则：
1. 只能包含上述指令
2. 表达式必须恰好产生一个值（与期望的类型匹配）
3. `global.get` 只能引用在当前全局变量之前定义的 imported immutable global

```c
Result validate_const_expr(Module* module, Expression* expr, ValType expected_type) {
    TypeStack stack;
    init_type_stack(&stack);

    for (int i = 0; i < expr->num_instrs; i++) {
        Instr* instr = &expr->instrs[i];
        switch (instr->opcode) {
            case OP_I32_CONST:
                push_type(&stack, VAL_I32);
                break;
            case OP_I64_CONST:
                push_type(&stack, VAL_I64);
                break;
            case OP_F32_CONST:
                push_type(&stack, VAL_F32);
                break;
            case OP_F64_CONST:
                push_type(&stack, VAL_F64);
                break;
            case OP_GLOBAL_GET: {
                uint32_t idx = instr->immediate.u32;
                // Must be an imported, immutable global
                if (idx >= module->num_imported_globals)
                    return ERROR("const expr: global.get must reference imported global");
                Global* g = &module->globals[idx];
                if (g->mutability != MUT_CONST)
                    return ERROR("const expr: global.get must reference immutable global");
                push_type(&stack, g->type);
                break;
            }
            case OP_REF_NULL:
                push_type(&stack, instr->immediate.reftype);
                break;
            case OP_REF_FUNC:
                push_type(&stack, VAL_FUNCREF);
                break;
            case OP_END:
                break;
            default:
                return ERROR("illegal instruction in constant expression: 0x%02X",
                             instr->opcode);
        }
    }

    if (stack.top != 1)
        return ERROR("const expr must produce exactly one value");
    if (stack.types[0] != expected_type)
        return ERROR("const expr type mismatch: expected %s, got %s",
                     type_name(expected_type), type_name(stack.types[0]));

    return RESULT_OK;
}
```

---

## 7. 验证中的常见陷阱与实现建议

### 7.1 容易出错的地方

**1. 弹出顺序**

栈是 LIFO 的，但函数签名是从左到右写的。弹出参数时要 **从右到左**：

```c
// Function type: [i32 i64] -> [f32]
// Stack (top on right): ... i32 i64
//                              ↑   ↑
//                           first  second (top)

// Pop in reverse order:
pop_operand(ctx, VAL_I64);  // pop second (top)
pop_operand(ctx, VAL_I32);  // pop first
push_operand(ctx, VAL_F32); // push result
```

**2. block 的参数处理（多返回值提案）**

在 MVP 中，block 只有 result（0 或 1 个值），没有参数。但多返回值提案允许 block 有参数。如果你要支持这个特性，需要在 `push_ctrl` 时正确处理参数的弹出和重新压入。

**3. if 没有 else 的情况**

```wasm
;; This is VALID: block type is [] -> [] (no results)
if
  nop
end

;; This is INVALID: block type is [] -> [i32] but no else branch
if (result i32)
  i32.const 1
end
;; Error! If condition is false, where does the i32 come from?
```

规则：如果 `if` 没有 `else`，则 block_type 的 params 和 results 必须相同。

**4. select 指令的类型检查**

```c
case OP_SELECT: {
    pop_operand(&ctx, VAL_I32);                // condition
    ValType t1 = pop_operand(&ctx, VAL_UNKNOWN); // second value
    ValType t2 = pop_operand(&ctx, t1);          // first value (must match)

    // Both must be numeric types (not reference types) for untyped select
    if (is_ref_type(t1) || is_ref_type(t2)) {
        validation_error("select requires numeric types (use select t for ref types)");
    }

    push_operand(&ctx, (t1 != VAL_UNKNOWN) ? t1 : t2);
    break;
}
```

### 7.2 实现建议

**1. 使用动态数组**

类型栈和控制帧栈都需要动态增长。预分配一个合理的初始大小（如 256），然后按需扩展。

**2. 错误信息要详细**

验证失败时，提供尽可能详细的错误信息：指令偏移、期望类型 vs 实际类型、控制栈深度等。这对调试非常有帮助。

**3. 先实现不带多返回值的版本**

MVP 版本中 block 没有参数，只有 0 或 1 个结果。先实现这个简化版本，验证通过后再添加多返回值支持。

**4. 使用 spec test suite 验证**

WebAssembly 官方测试套件中有大量的验证测试用例（以 `assert_invalid` 和 `assert_malformed` 标记），是验证你的验证器实现是否正确的最佳工具。

---

## 8. 与 CLR IL 验证的系统性对比

| 维度 | Wasm 验证 | CLR IL 验证 |
|------|-----------|-------------|
| **扫描方式** | 单遍前向扫描，O(n) | 需要多遍或迭代到不动点（因为有后向跳转到任意位置） |
| **控制流** | 结构化——block/loop/if 嵌套，无 goto | 非结构化——br/brfalse 可跳转到任意偏移 |
| **合并点类型** | 由 block_type 显式声明，无需推断 | 需要在 join point 处合并类型（可能需要迭代） |
| **栈深度** | 在每个控制结构的边界处确定性已知 | 需要在所有路径上验证栈深度一致 |
| **不可达代码** | 栈多态性，简单标记即可 | 需要 CFG 可达性分析 |
| **类型系统复杂度** | 简单——4 种数值类型 + 2 种引用类型 | 复杂——值类型、引用类型、泛型、继承层次 |
| **验证时机** | 加载时一次性完成 | JIT 编译时按方法验证（可延迟） |
| **子类型** | 无子类型关系（类型必须精确匹配） | 有子类型——赋值兼容性、接口实现等 |
| **局部变量初始化** | 所有局部变量自动初始化为零值 | 需要 definite assignment 分析（C# 层面）；IL 层面 `.locals init` 标志 |

**核心差异总结：** Wasm 的结构化控制流是其验证简洁性的根本原因。CLR 允许任意跳转，导致需要构建控制流图、计算支配关系、在合并点迭代类型推断。Wasm 通过限制控制流的表达能力，换取了验证的简单性和确定性——这是一个深思熟虑的设计权衡。

---

## 9. 小结

Wasm 验证的核心要点：

1. **三层验证**：模块结构 → 函数签名 → 指令类型检查
2. **类型栈 + 控制帧栈**：两个栈协同工作，完成单遍前向类型检查
3. **block vs loop 的标签类型差异**：block 的标签类型是 results，loop 的标签类型是 params
4. **栈多态性**：不可达代码中栈可以弹出任意类型，这是正确处理死代码的关键
5. **常量表达式**：全局变量初始化和 segment 偏移量只允许受限的指令集

掌握了验证机制，你就理解了 Wasm 类型系统的全部约束。这些约束直接影响执行引擎的设计——因为验证保证了类型安全，执行时可以省略类型检查，只需关注值的操作。

> **下一步**：[04-execution-engine.md](./04-execution-engine.md) —— 了解如何设计执行引擎来高效运行已验证的 Wasm 代码。
