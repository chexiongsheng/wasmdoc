# 第七章：实践项目指南 —— 从零实现 Mini-Wasm 解释器

> **目标**：通过 6 个渐进阶段，从零开始用 C 语言实现一个可工作的 WebAssembly 解释器（Mini-Wasm）。
> 每个阶段都有明确的里程碑和测试标准，确保你始终在一个可运行的状态上迭代。

---

## 1. 项目总体架构

### 1.1 设计原则

在动手之前，确立几个核心设计原则：

1. **正确性优先**：先做对，再做快。初始版本使用最简单的 switch-dispatch 解释器
2. **模块化设计**：解析器、验证器、执行引擎严格分离，各自可独立测试
3. **渐进式实现**：每个阶段产出可运行、可测试的代码，避免"大爆炸"式集成
4. **最小依赖**：纯 C 实现，不依赖第三方库（仅依赖 C 标准库）

### 1.2 推荐目录结构

```
mini-wasm/
├── CMakeLists.txt              # CMake build configuration
├── README.md
├── include/
│   ├── wasm_types.h            # Core type definitions (value types, limits, etc.)
│   ├── wasm_module.h           # Module structure definitions
│   ├── wasm_decoder.h          # Binary decoder API
│   ├── wasm_validator.h        # Validator API
│   ├── wasm_runtime.h          # Runtime/execution engine API
│   ├── wasm_instance.h         # Module instantiation API
│   └── wasm_host.h             # Host function binding API
├── src/
│   ├── main.c                  # Entry point, CLI interface
│   ├── wasm_decoder.c          # Binary format parser
│   ├── wasm_validator.c        # Module/function validator
│   ├── wasm_interp.c           # Interpreter (switch-dispatch)
│   ├── wasm_instance.c         # Instantiation logic
│   ├── wasm_memory.c           # Linear memory management
│   ├── wasm_table.c            # Table management
│   ├── wasm_host.c             # Host function registry
│   └── wasm_leb128.c           # LEB128 encoding/decoding utilities
├── test/
│   ├── test_decoder.c          # Decoder unit tests
│   ├── test_validator.c        # Validator unit tests
│   ├── test_interp.c           # Interpreter unit tests
│   ├── test_memory.c           # Memory operation tests
│   └── wasm_files/             # Test .wasm binaries
│       ├── add.wasm
│       ├── factorial.wasm
│       ├── memory_test.wasm
│       └── ...
└── tools/
    └── wat_examples/           # WAT source files for generating test binaries
        ├── add.wat
        ├── factorial.wat
        └── ...
```

### 1.3 CMake 基础配置

```cmake
cmake_minimum_required(VERSION 3.10)
project(mini-wasm C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Strict warnings — treat warnings as errors during development
add_compile_options(-Wall -Wextra -Wpedantic -Werror)

# Core library
add_library(miniwasm_core STATIC
    src/wasm_leb128.c
    src/wasm_decoder.c
    src/wasm_validator.c
    src/wasm_interp.c
    src/wasm_instance.c
    src/wasm_memory.c
    src/wasm_table.c
    src/wasm_host.c
)
target_include_directories(miniwasm_core PUBLIC include)

# Main executable
add_executable(miniwasm src/main.c)
target_link_libraries(miniwasm miniwasm_core m)  # link math library for float ops

# Test executables
enable_testing()
file(GLOB TEST_SOURCES test/test_*.c)
foreach(test_src ${TEST_SOURCES})
    get_filename_component(test_name ${test_src} NAME_WE)
    add_executable(${test_name} ${test_src})
    target_link_libraries(${test_name} miniwasm_core m)
    add_test(NAME ${test_name} COMMAND ${test_name})
endforeach()
```

### 1.4 核心类型定义（起步骨架）

```c
// wasm_types.h — Start with this minimal set, expand as needed

#ifndef WASM_TYPES_H
#define WASM_TYPES_H

#include <stdint.h>
#include <stdbool.h>
#include <stddef.h>

// ============================================================
// Value Types
// ============================================================
typedef enum {
    WASM_TYPE_I32       = 0x7F,
    WASM_TYPE_I64       = 0x7E,
    WASM_TYPE_F32       = 0x7D,
    WASM_TYPE_F64       = 0x7C,
    WASM_TYPE_FUNCREF   = 0x70,
    WASM_TYPE_EXTERNREF = 0x6F,
} WasmValType;

// ============================================================
// Runtime Values (tagged union)
// ============================================================
typedef struct {
    WasmValType type;
    union {
        int32_t  i32;
        int64_t  i64;
        float    f32;
        double   f64;
        uint32_t ref;       // funcref or externref index
    } of;
} WasmValue;

// ============================================================
// Function Type Signature
// ============================================================
typedef struct {
    uint32_t     param_count;
    WasmValType *param_types;
    uint32_t     result_count;
    WasmValType *result_types;
} WasmFuncType;

// ============================================================
// Limits (for memory and table)
// ============================================================
typedef struct {
    uint32_t min;
    uint32_t max;       // UINT32_MAX means no maximum
    bool     has_max;
} WasmLimits;

// ============================================================
// Error Handling
// ============================================================
typedef enum {
    WASM_OK = 0,
    WASM_ERROR_INVALID_MAGIC,
    WASM_ERROR_INVALID_VERSION,
    WASM_ERROR_UNEXPECTED_END,
    WASM_ERROR_INVALID_SECTION,
    WASM_ERROR_INVALID_TYPE,
    WASM_ERROR_INVALID_OPCODE,
    WASM_ERROR_TYPE_MISMATCH,
    WASM_ERROR_STACK_UNDERFLOW,
    WASM_ERROR_STACK_OVERFLOW,
    WASM_ERROR_OUT_OF_BOUNDS,
    WASM_ERROR_UNDEFINED_ELEMENT,
    WASM_ERROR_UNINITIALIZED_ELEMENT,
    WASM_ERROR_INDIRECT_CALL_TYPE_MISMATCH,
    WASM_ERROR_UNREACHABLE,
    WASM_ERROR_MEMORY_OUT_OF_BOUNDS,
    WASM_ERROR_DIVISION_BY_ZERO,
    WASM_ERROR_INTEGER_OVERFLOW,
    WASM_ERROR_INVALID_CONVERSION,
    WASM_ERROR_IMPORT_NOT_FOUND,
    WASM_ERROR_IMPORT_TYPE_MISMATCH,
    WASM_ERROR_OUT_OF_MEMORY,
    // ... extend as needed
} WasmError;

// Human-readable error description
const char *wasm_error_string(WasmError err);

#endif // WASM_TYPES_H
```

---

## 2. 阶段 1：二进制解析器（Module Decoder）

> **wasm3 参考实现**：[m3_parse.c](wasm3/source/m3_parse.c) — 完整的二进制解析器实现（~600 行），入口为 [m3_ParseModule()](wasm3/source/m3_parse.c#L598)。

### 2.1 目标

实现一个完整的 `.wasm` 二进制文件解析器，能够将二进制字节流解码为内存中的模块结构体。

**里程碑**：给定任意合法的 `.wasm` 文件，能正确解析出所有 Section 的内容，并以结构化方式打印。

### 2.2 关键数据结构

```c
// wasm_module.h

typedef struct {
    // Decoded sections
    uint32_t       type_count;
    WasmFuncType  *types;           // Type Section

    uint32_t       import_count;
    WasmImport    *imports;         // Import Section

    uint32_t       func_count;      // Function Section (type indices)
    uint32_t      *func_type_indices;

    uint32_t       table_count;
    WasmTable     *tables;          // Table Section

    uint32_t       memory_count;
    WasmMemory    *memories;        // Memory Section

    uint32_t       global_count;
    WasmGlobal    *globals;         // Global Section

    uint32_t       export_count;
    WasmExport    *exports;         // Export Section

    uint32_t       start_func;      // Start Section (-1 if absent)
    bool           has_start;

    uint32_t       elem_count;
    WasmElemSeg   *elements;        // Element Section

    uint32_t       code_count;
    WasmCode      *codes;           // Code Section

    uint32_t       data_count;
    WasmDataSeg   *data_segments;   // Data Section

    // Raw bytes for custom sections (optional)
    uint32_t       custom_count;
    WasmCustomSec *customs;
} WasmModule;
```

### 2.3 实现步骤

1. **实现 LEB128 读取器**（参考附录 B）
   ```c
   // Returns bytes consumed, or 0 on error
   uint32_t read_u32_leb128(const uint8_t *bytes, size_t len, uint32_t *result);
   uint32_t read_i32_leb128(const uint8_t *bytes, size_t len, int32_t *result);
   uint32_t read_i64_leb128(const uint8_t *bytes, size_t len, int64_t *result);
   ```

2. **实现字节流读取器**（封装游标位置）
   ```c
   typedef struct {
       const uint8_t *data;
       size_t         size;
       size_t         pos;
   } WasmReader;

   WasmError reader_read_byte(WasmReader *r, uint8_t *out);
   WasmError reader_read_u32(WasmReader *r, uint32_t *out);
   WasmError reader_read_bytes(WasmReader *r, size_t n, const uint8_t **out);
   WasmError reader_read_name(WasmReader *r, char **out, uint32_t *len);
   ```

3. **实现模块头解析**：验证 magic number `\0asm` 和 version `1`

4. **实现 Section 分发循环**：
   ```c
   while (reader_has_more(&reader)) {
       uint8_t section_id;
       uint32_t section_size;
       reader_read_byte(&reader, &section_id);
       reader_read_u32(&reader, &section_size);

       WasmReader section_reader = reader_slice(&reader, section_size);
       switch (section_id) {
           case 1:  err = decode_type_section(&section_reader, module); break;
           case 2:  err = decode_import_section(&section_reader, module); break;
           case 3:  err = decode_function_section(&section_reader, module); break;
           // ... all 13 sections
           case 0:  err = decode_custom_section(&section_reader, module); break;
           default: err = WASM_ERROR_INVALID_SECTION; break;
       }
       if (err != WASM_OK) return err;
   }
   ```

5. **逐个实现各 Section 的解码函数**（建议顺序）：
   - Type Section（最简单，先练手）
   - Function Section（仅索引数组）
   - Export Section
   - Import Section
   - Memory Section / Table Section
   - Global Section
   - Code Section（最复杂：locals 压缩编码 + 指令序列）
   - Element Section / Data Section
   - Start Section / DataCount Section
   - Custom Section（可选，先跳过 name section 解析）

### 2.4 测试策略

**工具链准备**：
```bash
# Install WABT (WebAssembly Binary Toolkit)
# Provides: wat2wasm, wasm2wat, wasm-objdump, wasm-validate

# On Ubuntu/Debian:
sudo apt install wabt

# On macOS:
brew install wabt

# On Windows:
# Download from https://github.com/WebAssembly/wabt/releases
```

**测试用例 1 —— 最简模块**：
```wasm
;; empty.wat — The simplest valid module
(module)
```
```bash
wat2wasm empty.wat -o empty.wasm
xxd empty.wasm
# Expected: 00 61 73 6d 01 00 00 00 (8 bytes, header only)
```

**测试用例 2 —— 加法函数**：
```wasm
;; add.wat
(module
  (func (export "add") (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add))
```

**测试用例 3 —— 使用 wasm-objdump 对照验证**：
```bash
# Dump all sections for comparison
wasm-objdump -x add.wasm

# Your decoder output should match wasm-objdump's output
```

**测试用例 4 —— 错误处理**：
- 截断的文件（只有 magic number 的前 2 字节）
- 错误的 magic number
- Section size 超出文件范围
- 非法的 value type 编码

### 2.5 CLR 经验迁移提示

如果你实现过 CLR PE 解析器，这个阶段会非常熟悉：

| CLR 概念 | Wasm 对应 |
|----------|----------|
| PE DOS Header + PE Signature | Magic Number `\0asm` + Version |
| Section Table (`.text`, `.rsrc`) | Section ID + Size 的流式结构 |
| Metadata Tables (#~) | Type/Function/Import/Export Sections |
| MethodDef Table → RVA → IL Body | Function Section (type idx) + Code Section (body) |
| StandAloneSig | Type Section (function signatures) |

**关键差异**：CLR 的元数据表是基于行号的随机访问结构（table + row index），而 Wasm 的 Section 是顺序流式编码，必须从头到尾线性解析。

---

## 3. 阶段 2：验证器（Validator）

### 3.1 目标

实现模块级和函数级验证，确保解析出的模块在语义上合法。

**里程碑**：能正确拒绝 WebAssembly spec test suite 中的所有 `assert_invalid` 和 `assert_malformed` 测试用例。

### 3.2 实现步骤

1. **模块级验证**（较简单）：
   ```c
   WasmError validate_module(const WasmModule *module) {
       // 1. Check: function count == code count
       // 2. Check: all type indices in bounds
       // 3. Check: all function indices in bounds (imports + local funcs)
       // 4. Check: at most 1 memory, 1 table (MVP restriction)
       // 5. Check: start function index valid and signature is [] -> []
       // 6. Check: export names are unique
       // 7. Check: memory limits valid (min <= max, max <= 65536)
       // ...
   }
   ```

2. **函数体验证**（核心难点）：
   ```c
   // Validation state for a single function
   typedef struct {
       // Operand type stack
       WasmValType *type_stack;
       uint32_t     type_stack_size;
       uint32_t     type_stack_capacity;

       // Control frame stack
       ControlFrame *ctrl_stack;
       uint32_t      ctrl_stack_size;
       uint32_t      ctrl_stack_capacity;
   } ValidationContext;

   typedef struct {
       uint8_t     opcode;         // block, loop, if
       WasmFuncType block_type;    // type of the block
       uint32_t    height;         // type stack height at entry
       bool        unreachable;    // is this frame in unreachable state?
   } ControlFrame;
   ```

3. **实现类型检查核心算法**：
   ```c
   // Push a type onto the operand stack
   void push_type(ValidationContext *ctx, WasmValType type);

   // Pop a type, checking it matches expected (WASM_TYPE_ANY for polymorphic)
   WasmError pop_type(ValidationContext *ctx, WasmValType expected, WasmValType *actual);

   // Push a new control frame
   void push_ctrl(ValidationContext *ctx, uint8_t opcode, WasmFuncType type);

   // Pop a control frame, validating result types
   WasmError pop_ctrl(ValidationContext *ctx, ControlFrame *frame);

   // Mark current frame as unreachable (after br, return, unreachable)
   void set_unreachable(ValidationContext *ctx);
   ```

4. **逐条指令实现类型规则**（按优先级）：
   - 先实现：`local.get/set/tee`, `i32.const`, `i32.add/sub/mul`, `drop`, `return`
   - 然后：`block`, `loop`, `if/else/end`, `br`, `br_if`
   - 然后：所有数值运算指令（批量实现，类型规则相似）
   - 然后：内存指令、全局变量指令
   - 最后：`call`, `call_indirect`, `br_table`

### 3.3 栈多态性处理要点

这是验证器中最微妙的部分。当控制帧被标记为 `unreachable` 后：

```c
WasmError pop_type(ValidationContext *ctx, WasmValType expected, WasmValType *actual) {
    ControlFrame *frame = current_frame(ctx);

    // If stack is at the frame's entry height...
    if (ctx->type_stack_size == frame->height) {
        if (frame->unreachable) {
            // Polymorphic: pretend we popped the expected type
            *actual = expected;
            return WASM_OK;
        }
        return WASM_ERROR_STACK_UNDERFLOW;
    }

    *actual = ctx->type_stack[--ctx->type_stack_size];
    if (expected != WASM_TYPE_ANY && *actual != expected) {
        return WASM_ERROR_TYPE_MISMATCH;
    }
    return WASM_OK;
}
```

### 3.4 测试策略

```bash
# Use the official spec test suite
git clone https://github.com/WebAssembly/spec.git
cd spec/test/core

# Each .wast file contains assert_invalid / assert_malformed cases
# Example: block.wast, br.wast, call.wast, etc.

# You'll need a simple .wast parser or manually extract test cases
# Alternatively, use wast2json to convert to a JSON test format:
wast2json block.wast -o block.json
```

关键测试文件（按优先级）：
- `block.wast` — 块结构验证
- `br.wast`, `br_if.wast`, `br_table.wast` — 分支验证
- `call.wast`, `call_indirect.wast` — 调用验证
- `type.wast` — 类型系统验证
- `unreachable.wast` — 栈多态性验证（最重要！）

---

## 4. 阶段 3：简单栈式解释器（数值运算 + 控制流）

### 4.1 目标

实现一个基于 switch-dispatch 的栈式解释器，支持数值运算和控制流指令。

**里程碑**：能正确执行 `factorial`、`fibonacci` 等递归函数。

### 4.2 关键数据结构

```c
// Runtime execution state
typedef struct {
    // Value stack (unified 64-bit slots)
    uint64_t    *stack;
    uint32_t     stack_top;
    uint32_t     stack_capacity;

    // Call frame stack
    CallFrame   *frames;
    uint32_t     frame_count;
    uint32_t     frame_capacity;
} WasmInterp;

typedef struct {
    const WasmModule   *module;
    const WasmCode     *code;       // current function's code
    uint32_t            pc;         // program counter (offset in code bytes)
    uint32_t            sp_base;    // stack base for this frame
    uint32_t            local_count;
    uint64_t           *locals;     // pointer into stack for locals
} CallFrame;
```

### 4.3 解释器主循环

```c
WasmError wasm_interp_execute(WasmInterp *interp) {
    CallFrame *frame = &interp->frames[interp->frame_count - 1];
    const uint8_t *code = frame->code->body;

    while (true) {
        uint8_t opcode = code[frame->pc++];
        switch (opcode) {
            // ---- Constants ----
            case 0x41: { // i32.const
                int32_t val;
                frame->pc += read_i32_leb128(code + frame->pc, ..., &val);
                push_i32(interp, val);
                break;
            }

            // ---- Arithmetic ----
            case 0x6A: { // i32.add
                int32_t b = pop_i32(interp);
                int32_t a = pop_i32(interp);
                push_i32(interp, a + b);
                break;
            }

            // ---- Control Flow ----
            case 0x02: { // block
                // Read block type, push label
                ...
                break;
            }
            case 0x0C: { // br
                uint32_t depth;
                frame->pc += read_u32_leb128(code + frame->pc, ..., &depth);
                branch_to(interp, depth);
                break;
            }

            case 0x0B: { // end
                // Pop label or return from function
                ...
                break;
            }

            case 0x0F: { // return
                return return_from_function(interp);
            }

            case 0x00: { // unreachable
                return WASM_ERROR_UNREACHABLE;
            }

            default:
                return WASM_ERROR_INVALID_OPCODE;
        }
    }
}
```

### 4.4 控制流实现 —— 标签栈方案

这是解释器实现中最关键的设计决策之一。

**方案 A：运行时标签栈（推荐初始方案）**

```c
typedef struct {
    uint8_t  opcode;        // block / loop / if
    uint32_t pc_end;        // PC of matching 'end' (for block/if: branch target)
    uint32_t pc_start;      // PC of block start (for loop: branch target)
    uint32_t stack_height;  // stack height at block entry
    uint32_t arity;         // number of result values
} Label;

// Branch implementation:
void branch_to(WasmInterp *interp, uint32_t depth) {
    Label *target = &interp->labels[interp->label_count - 1 - depth];

    // Preserve top N values (arity) on stack
    uint32_t arity = target->arity;
    // Copy top 'arity' values
    // Reset stack to target->stack_height
    // Push back the arity values

    if (target->opcode == 0x03) {
        // loop: branch to start
        frame->pc = target->pc_start;
    } else {
        // block/if: branch to end
        frame->pc = target->pc_end;
        // Pop labels up to and including target
    }
}
```

**方案 B：预计算跳转目标（优化方案）**

在解析/编译阶段预计算每个 `block`/`loop`/`if` 的 `end` 位置和每个 `br` 的跳转目标，存储在辅助数组中。运行时直接查表跳转，无需维护标签栈。

```c
// Pre-computed during a compilation pass
typedef struct {
    uint32_t *block_ends;    // block_ends[pc_of_block] = pc_of_end
    uint32_t *br_targets;    // br_targets[pc_of_br] = target_pc
} JumpTable;
```

> **建议**：先用方案 A 实现正确性，阶段 6 再优化为方案 B。

### 4.5 测试用例

**阶乘函数**：
```wasm
;; factorial.wat
(module
  (func (export "factorial") (param i32) (result i32)
    (local i32)
    (local.set 1 (i32.const 1))
    (block $break
      (loop $continue
        (br_if $break (i32.eqz (local.get 0)))
        (local.set 1 (i32.mul (local.get 1) (local.get 0)))
        (local.set 0 (i32.sub (local.get 0) (i32.const 1)))
        (br $continue)))
    (local.get 1)))
```

**斐波那契（递归版）**：
```wasm
;; fibonacci.wat
(module
  (func (export "fib") (param i32) (result i32)
    (if (result i32) (i32.le_s (local.get 0) (i32.const 1))
      (then (local.get 0))
      (else
        (i32.add
          (call 0 (i32.sub (local.get 0) (i32.const 1)))
          (call 0 (i32.sub (local.get 0) (i32.const 2))))))))
```

预期结果验证：
```
factorial(0)  = 1
factorial(5)  = 120
factorial(10) = 3628800
fib(0)  = 0
fib(1)  = 1
fib(10) = 55
fib(20) = 6765
```

---

## 5. 阶段 4：内存操作与函数调用

### 5.1 目标

添加线性内存支持和完整的函数调用机制（包括 `call_indirect`）。

**里程碑**：能执行涉及内存读写和间接调用的程序，如简单的字符串处理。

### 5.2 线性内存实现

```c
typedef struct {
    uint8_t  *data;          // raw memory buffer
    uint32_t  size;          // current size in bytes
    uint32_t  max_size;      // maximum size in bytes (from limits)
    uint32_t  page_count;    // current size in pages (64KB each)
    uint32_t  max_pages;     // maximum pages
} WasmLinearMemory;

WasmError memory_init(WasmLinearMemory *mem, uint32_t min_pages, uint32_t max_pages) {
    mem->page_count = min_pages;
    mem->max_pages = max_pages;
    mem->size = min_pages * 65536;
    mem->max_size = max_pages * 65536;
    mem->data = calloc(1, mem->size);  // zero-initialized per spec
    return mem->data ? WASM_OK : WASM_ERROR_OUT_OF_MEMORY;
}

int32_t memory_grow(WasmLinearMemory *mem, uint32_t delta_pages) {
    uint32_t old_pages = mem->page_count;
    uint32_t new_pages = old_pages + delta_pages;

    if (new_pages > mem->max_pages) return -1;  // failure
    if (new_pages > 65536) return -1;            // absolute max: 4GB

    uint32_t new_size = new_pages * 65536;
    uint8_t *new_data = realloc(mem->data, new_size);
    if (!new_data) return -1;

    // Zero-initialize new pages
    memset(new_data + mem->size, 0, new_size - mem->size);

    mem->data = new_data;
    mem->size = new_size;
    mem->page_count = new_pages;
    return (int32_t)old_pages;  // return previous size on success
}
```

### 5.3 内存访问与边界检查

```c
// Generic load with bounds check
// All Wasm memory accesses are LITTLE-ENDIAN regardless of host
static inline WasmError mem_load_i32(WasmLinearMemory *mem,
                                      uint32_t addr, uint32_t offset,
                                      int32_t *result) {
    uint64_t effective_addr = (uint64_t)addr + offset;
    if (effective_addr + 4 > mem->size) {
        return WASM_ERROR_MEMORY_OUT_OF_BOUNDS;
    }
    // Little-endian load (portable)
    uint8_t *p = mem->data + effective_addr;
    *result = (int32_t)(p[0] | (p[1] << 8) | (p[2] << 16) | (p[3] << 24));
    return WASM_OK;
}

// For hosts that are already little-endian (x86/ARM), you can optimize:
// memcpy(result, mem->data + effective_addr, 4);
```

### 5.4 函数调用实现

```c
WasmError call_function(WasmInterp *interp, uint32_t func_idx) {
    // 1. Resolve function (imported or local)
    // 2. Get function type signature
    WasmFuncType *type = resolve_func_type(interp, func_idx);

    // 3. Pop arguments from caller's stack
    // 4. Push new call frame
    CallFrame *new_frame = push_frame(interp);
    new_frame->module = ...;
    new_frame->code = ...;
    new_frame->pc = 0;
    new_frame->sp_base = interp->stack_top - type->param_count;

    // 5. Initialize locals (params are already on stack, add zero-init locals)
    for (uint32_t i = 0; i < code->local_count; i++) {
        push_u64(interp, 0);  // zero-initialize
    }

    // 6. Execute (recursive call to interpreter or iterative with frame check)
    return WASM_OK;
}

WasmError call_indirect(WasmInterp *interp, uint32_t type_idx, uint32_t table_idx) {
    // 1. Pop function index from stack
    uint32_t elem_idx = pop_i32(interp);

    // 2. Bounds check on table
    WasmTable *table = &interp->instance->tables[table_idx];
    if (elem_idx >= table->size) return WASM_ERROR_UNDEFINED_ELEMENT;

    // 3. Get function reference from table
    uint32_t func_idx = table->elements[elem_idx];
    if (func_idx == WASM_REF_NULL) return WASM_ERROR_UNINITIALIZED_ELEMENT;

    // 4. Type check: actual function type must match expected type
    WasmFuncType *expected = &interp->module->types[type_idx];
    WasmFuncType *actual = resolve_func_type(interp, func_idx);
    if (!func_type_equals(expected, actual)) {
        return WASM_ERROR_INDIRECT_CALL_TYPE_MISMATCH;
    }

    // 5. Call the function
    return call_function(interp, func_idx);
}
```

### 5.5 测试用例

**内存读写测试**：
```wasm
(module
  (memory (export "mem") 1)
  (func (export "store_and_load") (param i32 i32) (result i32)
    (i32.store (local.get 0) (local.get 1))
    (i32.load (local.get 0))))
```

**间接调用测试**：
```wasm
(module
  (type $binop (func (param i32 i32) (result i32)))
  (table 2 funcref)
  (elem (i32.const 0) $add $sub)
  (func $add (param i32 i32) (result i32) (i32.add (local.get 0) (local.get 1)))
  (func $sub (param i32 i32) (result i32) (i32.sub (local.get 0) (local.get 1)))
  (func (export "call_op") (param i32 i32 i32) (result i32)
    (call_indirect (type $binop) (local.get 0) (local.get 1) (local.get 2))))
```

---

## 6. 阶段 5：导入/导出与宿主函数绑定

> **wasm3 参考实现**：[m3_bind.c](wasm3/source/m3_bind.c) — 宿主函数绑定实现；[m3_env.c - m3_LoadModule()](wasm3/source/m3_env.c#L596) — 模块加载与导入解析；[m3_api_wasi.c - m3_LinkWASI()](wasm3/source/m3_api_wasi.c#L784) — WASI 宿主函数注册示例。

### 6.1 目标

实现模块实例化的完整流程，支持导入解析和宿主函数绑定。

**里程碑**：能绑定一个简单的 `print` 宿主函数，运行 Hello World 程序。

### 6.2 宿主函数接口设计

```c
// Host function callback signature
typedef WasmError (*WasmHostFunc)(
    void        *user_data,     // arbitrary context
    const WasmValue *args,      // input arguments
    uint32_t     arg_count,
    WasmValue   *results,       // output results
    uint32_t     result_count
);

// Host function registration
typedef struct {
    const char   *module_name;
    const char   *func_name;
    WasmFuncType  type;
    WasmHostFunc  callback;
    void         *user_data;
} WasmHostBinding;

// Example: binding a print function
WasmError host_print_i32(void *ud, const WasmValue *args, uint32_t argc,
                          WasmValue *results, uint32_t resc) {
    (void)ud; (void)results; (void)resc;
    printf("%d\n", args[0].of.i32);
    return WASM_OK;
}
```

### 6.3 实例化流程

```c
WasmError wasm_instantiate(WasmModule *module,
                            const WasmHostBinding *bindings,
                            uint32_t binding_count,
                            WasmInstance **out_instance) {
    WasmInstance *inst = calloc(1, sizeof(WasmInstance));

    // Step 1: Resolve imports
    for (uint32_t i = 0; i < module->import_count; i++) {
        WasmImport *imp = &module->imports[i];
        bool found = false;
        for (uint32_t j = 0; j < binding_count; j++) {
            if (strcmp(imp->module, bindings[j].module_name) == 0 &&
                strcmp(imp->name, bindings[j].func_name) == 0) {
                // Type check
                if (!func_type_equals(&module->types[imp->type_idx],
                                       &bindings[j].type)) {
                    return WASM_ERROR_IMPORT_TYPE_MISMATCH;
                }
                inst->functions[i] = make_host_func(&bindings[j]);
                found = true;
                break;
            }
        }
        if (!found) return WASM_ERROR_IMPORT_NOT_FOUND;
    }

    // Step 2: Allocate memories
    for (uint32_t i = 0; i < module->memory_count; i++) {
        memory_init(&inst->memories[i],
                    module->memories[i].limits.min,
                    module->memories[i].limits.has_max ?
                        module->memories[i].limits.max : 65536);
    }

    // Step 3: Allocate tables
    // Step 4: Initialize globals (evaluate init expressions)
    // Step 5: Apply element segments (fill tables)
    // Step 6: Apply data segments (fill memory)
    // Step 7: Call start function (if present)

    *out_instance = inst;
    return WASM_OK;
}
```

### 6.4 测试用例

**Hello World（使用自定义 print 宿主函数）**：
```wasm
(module
  (import "env" "print_i32" (func $print (param i32)))
  (func (export "main")
    (call $print (i32.const 42))
    (call $print (i32.const 100))))
```

**简易 WASI fd_write 模拟**：
```c
// Minimal fd_write implementation for stdout
WasmError host_fd_write(void *ud, const WasmValue *args, uint32_t argc,
                         WasmValue *results, uint32_t resc) {
    WasmInstance *inst = (WasmInstance *)ud;
    int32_t fd = args[0].of.i32;
    int32_t iovs_ptr = args[1].of.i32;
    int32_t iovs_len = args[2].of.i32;
    // int32_t nwritten_ptr = args[3].of.i32;

    if (fd != 1) { /* only stdout */ }

    uint32_t total = 0;
    for (int32_t i = 0; i < iovs_len; i++) {
        int32_t buf_ptr, buf_len;
        mem_load_i32(&inst->memories[0], iovs_ptr + i * 8, 0, &buf_ptr);
        mem_load_i32(&inst->memories[0], iovs_ptr + i * 8 + 4, 0, &buf_len);
        fwrite(inst->memories[0].data + buf_ptr, 1, buf_len, stdout);
        total += buf_len;
    }

    results[0].type = WASM_TYPE_I32;
    results[0].of.i32 = 0;  // success
    return WASM_OK;
}
```

---

## 7. 阶段 6：优化 —— Threaded Code 执行引擎

> **wasm3 参考实现**：这正是 wasm3 的核心创新。编译器见 [m3_compile.c](wasm3/source/m3_compile.c)；执行引擎见 [m3_exec.h](wasm3/source/m3_exec.h)；核心宏定义见 [m3_exec_defs.h](wasm3/source/m3_exec_defs.h)；代码页管理见 [m3_code.c](wasm3/source/m3_code.c)。

### 7.1 目标

将 switch-dispatch 解释器升级为 threaded code 风格，显著提升执行性能。

**里程碑**：在 fibonacci(35) 等计算密集型测试中，性能提升 2-5 倍。

### 7.2 从 Switch-Dispatch 到 Threaded Code

**Switch-Dispatch 的性能瓶颈**：
```
每条指令的执行路径：
  fetch opcode → switch jump → execute → loop back → fetch next opcode
                  ↑
          Branch prediction failure!
          (indirect jump through switch table, hard to predict)
```

**Direct Threaded Code 的改进**：
```
每条指令的执行路径：
  execute → jump to next handler (via function pointer or computed goto)
            ↑
    No central dispatch! Each handler directly jumps to the next.
    Better branch prediction (each jump site has its own prediction entry).
```

### 7.3 实现方案 A：Computed Goto（GCC/Clang 扩展）

```c
// Compilation phase: translate wasm opcodes to label addresses
typedef void* OpHandler;

static void compile_to_threaded(WasmCode *code, OpHandler **out_ops) {
    // Label address table
    static void* dispatch_table[] = {
        [0x00] = &&op_unreachable,
        [0x01] = &&op_nop,
        [0x02] = &&op_block,
        // ...
        [0x41] = &&op_i32_const,
        [0x6A] = &&op_i32_add,
        // ...
    };

    // First pass: convert opcode stream to handler pointer stream
    // Each entry is: handler_address [immediate_data...]
    // ...
}

// Execution with computed goto
WasmError execute_threaded(OpHandler *ops, WasmInterp *interp) {
    OpHandler *ip = ops;  // instruction pointer

    #define DISPATCH() goto *(*ip++)
    #define NEXT()     DISPATCH()

    DISPATCH();

    op_i32_const: {
        int32_t val = *(int32_t *)ip; ip = (OpHandler*)((int32_t*)ip + 1);
        push_i32(interp, val);
        NEXT();
    }

    op_i32_add: {
        int32_t b = pop_i32(interp);
        int32_t a = pop_i32(interp);
        push_i32(interp, a + b);
        NEXT();
    }

    // ... more handlers

    op_unreachable: {
        return WASM_ERROR_UNREACHABLE;
    }
}
```

### 7.4 实现方案 B：Tail-Call 函数指针链（wasm3 风格，更可移植）

```c
// Each operation is a C function that returns the next operation to execute
typedef const void* (*M3Operation)(
    uint64_t *sp,           // stack pointer
    uint8_t  *mem,          // linear memory base
    const void **ip,        // instruction pointer (array of op pointers + immediates)
    uint64_t *regs          // register slots (optimization)
);

// Example operation
const void* op_i32_add(uint64_t *sp, uint8_t *mem, const void **ip, uint64_t *regs) {
    int32_t b = (int32_t)*(sp);
    int32_t a = (int32_t)*(sp - 1);
    *(--sp) = (uint64_t)(int32_t)(a + b);

    // Tail call to next operation
    return ((M3Operation)(*ip++))(sp, mem, ip, regs);
}

// Trampoline loop (avoids actual deep recursion via compiler tail-call optimization)
WasmError execute_m3_style(const void **ops, WasmInterp *interp) {
    uint64_t *sp = interp->stack + interp->stack_top;
    uint8_t *mem = interp->instance->memories[0].data;
    const void **ip = ops;
    uint64_t regs[2] = {0};  // register slots

    M3Operation op = (M3Operation)(*ip++);
    while (op) {
        op = (M3Operation)op(sp, mem, ip, regs);
    }
    return WASM_OK;
}
```

### 7.5 编译阶段：Wasm → Threaded Code

```c
// Compilation context
typedef struct {
    OpHandler  *ops;         // output operation stream
    uint32_t    ops_count;
    uint32_t    ops_capacity;

    // Patch list for forward branches (block ends not yet known)
    PatchEntry *patches;
    uint32_t    patch_count;
} CompileContext;

void compile_function(const WasmCode *code, CompileContext *ctx) {
    WasmReader reader = { code->body, code->body_size, 0 };

    while (reader.pos < reader.body_size) {
        uint8_t opcode = read_byte(&reader);
        switch (opcode) {
            case 0x41: { // i32.const
                int32_t val = read_i32_leb128(&reader);
                emit_op(ctx, op_i32_const);
                emit_immediate_i32(ctx, val);
                break;
            }
            case 0x6A: { // i32.add
                emit_op(ctx, op_i32_add);
                break;
            }
            case 0x02: { // block
                // Emit block_begin, record patch point for end address
                ...
                break;
            }
            // ...
        }
    }

    // Resolve all forward branch patches
    resolve_patches(ctx);
}
```

### 7.6 性能对比基准

用以下测试衡量优化效果：

```wasm
;; fib_bench.wat — Recursive fibonacci for benchmarking
(module
  (func $fib (export "fib") (param i32) (result i32)
    (if (result i32) (i32.le_s (local.get 0) (i32.const 1))
      (then (local.get 0))
      (else (i32.add
        (call $fib (i32.sub (local.get 0) (i32.const 1)))
        (call $fib (i32.sub (local.get 0) (i32.const 2)))))))
)
```

| 实现方式 | fib(35) 预期耗时 | 相对性能 |
|---------|-----------------|---------|
| Switch-dispatch | ~500ms | 1x (baseline) |
| Computed goto | ~150ms | ~3x |
| Tail-call chain (M3 style) | ~200ms | ~2.5x |
| wasm3 (参考) | ~120ms | ~4x |

> 注：实际性能取决于编译器优化、CPU 分支预测等因素。

---

## 8. 常见难点与解决方案

### 8.1 结构化控制流的标签栈实现

**难点**：Wasm 的 `br` 指令使用相对深度索引（de Bruijn index），而不是绝对地址。`block` 和 `loop` 的分支目标不同（block 跳到 end，loop 跳到开头）。

**解决方案**：
```c
// During execution, maintain a label stack
// br N means: branch to the Nth label from the top

// Key insight: pre-scan each function to find matching end positions
// This avoids runtime scanning for 'end' instructions

typedef struct {
    uint32_t target_pc;     // where to jump
    uint32_t stack_height;  // stack height to restore
    uint32_t arity;         // number of values to keep
    bool     is_loop;       // loop branches to start, block to end
} RuntimeLabel;
```

### 8.2 br_table 的高效实现

**难点**：`br_table` 有一个标签索引数组 + 默认标签，需要高效分发。

**解决方案**：
```c
case 0x0E: { // br_table
    uint32_t count = read_u32_leb128(&reader);
    uint32_t *targets = alloca((count + 1) * sizeof(uint32_t));
    for (uint32_t i = 0; i <= count; i++) {
        targets[i] = read_u32_leb128(&reader);
    }

    int32_t index = pop_i32(interp);
    uint32_t depth;
    if (index >= 0 && (uint32_t)index < count) {
        depth = targets[index];
    } else {
        depth = targets[count];  // default
    }
    branch_to(interp, depth);
    break;
}
```

**优化**：在编译阶段将 `br_table` 的所有目标预解析为绝对 PC 地址，运行时直接查表跳转。

### 8.3 IEEE 754 浮点语义

**难点**：Wasm 规范对浮点运算有严格要求，包括 NaN 传播规则和特定的舍入行为。

**关键规则**：
1. **NaN Canonicalization**：Wasm 规范要求算术 NaN（canonical NaN）为 `0x7FC00000`（f32）或 `0x7FF8000000000000`（f64）
2. **NaN Propagation**：如果任一操作数是 NaN，结果应为 canonical NaN（某些操作保留 NaN payload）
3. **Division**：`0.0 / 0.0 = NaN`，`x / 0.0 = ±Infinity`
4. **Conversions**：`i32.trunc_f32_s` 等截断转换在溢出或 NaN 时必须 trap

```c
// NaN canonicalization helper
static inline float canonicalize_nan_f32(float val) {
    if (isnan(val)) {
        uint32_t canonical = 0x7FC00000;
        float result;
        memcpy(&result, &canonical, 4);
        return result;
    }
    return val;
}

// Safe truncation with trap
static inline WasmError trunc_f32_to_i32(float val, int32_t *result) {
    if (isnan(val)) return WASM_ERROR_INVALID_CONVERSION;
    if (val >= 2147483648.0f || val < -2147483648.0f) {
        return WASM_ERROR_INTEGER_OVERFLOW;
    }
    *result = (int32_t)val;
    return WASM_OK;
}
```

### 8.4 多返回值的栈布局

**难点**：Wasm 2.0 支持多返回值，`block`/`if`/`loop` 也可以有多个输入和输出。

**解决方案**：
```
Stack layout for multi-value block:

Before block entry:     [..., input_0, input_1]
                                              ^ block_stack_height
During block:           [..., input_0, input_1, local_vals...]
At block end/br:        [..., input_0, input_1, local_vals..., result_0, result_1]

After end (cleanup):    [..., result_0, result_1]
                        (inputs consumed, locals discarded, results kept)
```

```c
void end_block(WasmInterp *interp, RuntimeLabel *label) {
    uint32_t arity = label->arity;

    // Save top 'arity' values
    uint64_t results[MAX_RESULTS];
    for (uint32_t i = 0; i < arity; i++) {
        results[arity - 1 - i] = pop_u64(interp);
    }

    // Reset stack to block entry height
    interp->stack_top = label->stack_height;

    // Push results back
    for (uint32_t i = 0; i < arity; i++) {
        push_u64(interp, results[i]);
    }
}
```

### 8.5 调用栈深度限制

**难点**：恶意或错误的 Wasm 模块可能导致无限递归，耗尽宿主栈。

**解决方案**：
```c
#define MAX_CALL_DEPTH 1000

WasmError call_function(WasmInterp *interp, uint32_t func_idx) {
    if (interp->frame_count >= MAX_CALL_DEPTH) {
        return WASM_ERROR_STACK_OVERFLOW;
    }
    // ... proceed with call
}
```

---

## 9. 测试策略总览

### 9.1 测试金字塔

```
                    ┌─────────────┐
                    │  Spec Tests │  ← WebAssembly official test suite
                    │  (wast)     │
                   ┌┴─────────────┴┐
                   │ Integration    │  ← Full module load + execute
                   │ Tests          │
                  ┌┴────────────────┴┐
                  │  Unit Tests       │  ← Individual component tests
                  │  (decoder,        │
                  │   validator,      │
                  │   interpreter)    │
                 ┌┴───────────────────┴┐
                 │  Utility Tests       │  ← LEB128, memory helpers, etc.
                 └─────────────────────┘
```

### 9.2 使用官方 Spec Test Suite

```bash
# Clone the spec repo
git clone https://github.com/WebAssembly/spec.git

# Convert .wast to JSON format for easier parsing
cd spec/test/core
for f in *.wast; do
    wast2json "$f" -o "json/$(basename $f .wast).json"
done
```

JSON 格式示例：
```json
{
  "source_filename": "i32.wast",
  "commands": [
    {"type": "module", "filename": "i32.0.wasm"},
    {"type": "assert_return", "action": {"type": "invoke", "field": "add", "args": [{"type": "i32", "value": "1"}, {"type": "i32", "value": "1"}]}, "expected": [{"type": "i32", "value": "2"}]},
    {"type": "assert_trap", "action": {"type": "invoke", "field": "div_s", "args": [{"type": "i32", "value": "1"}, {"type": "i32", "value": "0"}]}, "text": "integer divide by zero"}
  ]
}
```

### 9.3 推荐的测试优先级

| 优先级 | 测试文件 | 覆盖内容 |
|--------|---------|---------|
| P0 | `i32.wast`, `i64.wast` | 整数运算 |
| P0 | `block.wast`, `loop.wast`, `if.wast` | 控制流 |
| P0 | `br.wast`, `br_if.wast` | 分支 |
| P0 | `call.wast`, `return.wast` | 函数调用 |
| P1 | `local_get.wast`, `local_set.wast`, `local_tee.wast` | 局部变量 |
| P1 | `memory.wast`, `load.wast`, `store.wast` | 内存操作 |
| P1 | `call_indirect.wast` | 间接调用 |
| P1 | `f32.wast`, `f64.wast` | 浮点运算 |
| P2 | `br_table.wast` | 跳转表 |
| P2 | `global.wast` | 全局变量 |
| P2 | `elem.wast`, `data.wast` | 段初始化 |
| P2 | `imports.wast`, `exports.wast` | 导入导出 |
| P3 | `unreachable.wast` | 栈多态性 |
| P3 | `conversions.wast` | 类型转换 |
| P3 | `traps.wast` | 运行时错误 |

### 9.4 自制测试用例模板

```c
// test_interp.c — Simple test framework
#include <stdio.h>
#include <assert.h>
#include "wasm_decoder.h"
#include "wasm_runtime.h"

#define TEST(name) static void test_##name(void)
#define RUN(name) do { printf("  %-40s", #name); test_##name(); printf("PASS\n"); } while(0)

TEST(i32_add) {
    // Load test module
    WasmModule module;
    assert(wasm_decode_file("test/wasm_files/add.wasm", &module) == WASM_OK);

    // Instantiate
    WasmInstance *inst;
    assert(wasm_instantiate(&module, NULL, 0, &inst) == WASM_OK);

    // Call exported function
    WasmValue args[] = {{.type = WASM_TYPE_I32, .of.i32 = 3},
                         {.type = WASM_TYPE_I32, .of.i32 = 4}};
    WasmValue result;
    assert(wasm_call(inst, "add", args, 2, &result, 1) == WASM_OK);
    assert(result.of.i32 == 7);

    wasm_instance_free(inst);
    wasm_module_free(&module);
}

TEST(factorial) {
    // ... similar pattern
}

int main(void) {
    printf("Interpreter Tests:\n");
    RUN(i32_add);
    RUN(factorial);
    printf("All tests passed!\n");
    return 0;
}
```

---

## 10. 工具链参考

### 10.1 必备工具

| 工具 | 用途 | 安装 |
|------|------|------|
| **wat2wasm** | WAT 文本格式 → .wasm 二进制 | WABT 套件 |
| **wasm2wat** | .wasm 二进制 → WAT 文本格式（反汇编） | WABT 套件 |
| **wasm-objdump** | 查看 .wasm 文件的 Section 结构 | WABT 套件 |
| **wasm-validate** | 验证 .wasm 文件合法性 | WABT 套件 |
| **wast2json** | .wast 测试文件 → JSON 格式 | WABT 套件 |
| **xxd** / **hexdump** | 查看原始十六进制字节 | 系统自带 |

### 10.2 调试技巧

1. **对照反汇编**：实现每条指令时，用 `wasm2wat` 反汇编确认你的理解
2. **逐字节对照**：用 `xxd` 查看二进制，手动对照你的解析结果
3. **参考实现对比**：用 wasm3 或 wasmer 运行同一模块，对比执行结果
4. **渐进式调试**：每实现一条新指令，立即编写对应的测试用例

### 10.3 有用的在线资源

- [WebAssembly Spec (官方规范)](https://webassembly.github.io/spec/core/)
- [wasm3 源码](https://github.com/nicedoc/wasm3)
- [WABT 工具套件](https://github.com/WebAssembly/wabt)
- [WebAssembly Reference Manual (非官方，更易读)](https://github.com/nicedoc/reference-manual)
- [WAT 在线编辑器 (wat2wasm demo)](https://webassembly.github.io/wabt/demo/wat2wasm/)
- [WebAssembly Spec Test Suite](https://github.com/nicedoc/spec/tree/main/test/core)

---

## 11. 里程碑检查清单

### 阶段 1 ✓ 检查项
- [ ] 能解析空模块 `(module)`
- [ ] 能解析包含单个函数的模块
- [ ] 能解析包含 memory、table、global 的模块
- [ ] 能解析包含 import/export 的模块
- [ ] 能正确处理截断/损坏的文件（返回错误而非崩溃）
- [ ] 输出与 `wasm-objdump -x` 一致

### 阶段 2 ✓ 检查项
- [ ] 能通过 `block.wast` 的所有 `assert_invalid` 用例
- [ ] 能通过 `unreachable.wast` 的验证用例（栈多态性）
- [ ] 能正确拒绝类型不匹配的函数体
- [ ] 能正确拒绝索引越界的引用

### 阶段 3 ✓ 检查项
- [ ] `factorial(10)` = 3628800
- [ ] `fibonacci(20)` = 6765
- [ ] 所有 i32/i64 算术运算正确
- [ ] `block`/`loop`/`if`/`br`/`br_if` 控制流正确
- [ ] `br_table` 正确分发

### 阶段 4 ✓ 检查项
- [ ] 内存 load/store 正确（包括非对齐访问）
- [ ] `memory.grow` 正确工作
- [ ] 内存越界访问正确 trap
- [ ] `call_indirect` 正确工作
- [ ] 间接调用类型不匹配正确 trap

### 阶段 5 ✓ 检查项
- [ ] 能绑定宿主函数并正确调用
- [ ] 导入类型不匹配时正确报错
- [ ] Element/Data segment 正确初始化
- [ ] Start function 正确执行
- [ ] 能运行简单的 WASI 程序（Hello World）

### 阶段 6 ✓ 检查项
- [ ] Threaded code 编译正确
- [ ] 所有之前的测试仍然通过
- [ ] fib(35) 性能提升 ≥ 2x
- [ ] 无内存泄漏（valgrind clean）

---

> **最后的建议**：不要试图一次性实现所有东西。每个阶段都应该是一个可工作的、可测试的状态。
> 当你卡住时，回去看 wasm3 的源码（参考第六章的导读），理解它是如何解决同样问题的。
> 记住：你已经实现过 CLR 虚拟机，Wasm 解释器在很多方面更简单 —— 没有 GC、没有复杂的类型系统、没有异常处理机制。你完全有能力做到。
