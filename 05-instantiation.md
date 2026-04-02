# 第五章：Wasm 模块实例化与链接

> **前置阅读**：[第一章：二进制格式](01-binary-format.md)、[第三章：验证机制](03-validation.md)、[第四章：执行引擎](04-execution-engine.md)

## 目录

1. [概述：从字节到运行](#1-概述从字节到运行)
2. [实例化的完整生命周期](#2-实例化的完整生命周期)
3. [Store —— 全局状态容器](#3-store--全局状态容器)
4. [导入解析（Import Resolution）](#4-导入解析import-resolution)
5. [实例分配（Allocation）](#5-实例分配allocation)
6. [初始化：Element 与 Data Segments](#6-初始化element-与-data-segments)
7. [导出机制（Exports）](#7-导出机制exports)
8. [宿主函数绑定（Host Functions）](#8-宿主函数绑定host-functions)
9. [WASI —— 系统接口标准](#9-wasi--系统接口标准)
10. [多模块链接](#10-多模块链接)
11. [CLR 对比：Assembly 加载与 AppDomain](#11-clr-对比assembly-加载与-appdomain)
12. [实现要点与常见陷阱](#12-实现要点与常见陷阱)

---

## 1. 概述：从字节到运行

> **wasm3 实现参考**：完整的加载流程参见 [wasm3.h](wasm3/source/wasm3.h) 中的公共 API：[m3_NewEnvironment()](wasm3/source/m3_env.c#L18) → [m3_NewRuntime()](wasm3/source/m3_env.c#L153) → [m3_ParseModule()](wasm3/source/m3_parse.c#L598) → [m3_LoadModule()](wasm3/source/m3_env.c#L430) → [m3_FindFunction()](wasm3/source/m3_env.c#L505) → [m3_Call()](wasm3/source/m3_env.c#L536)。

一个 `.wasm` 文件从加载到执行，需要经历以下阶段：

```
┌──────────┐    ┌──────────┐    ┌─────────────┐    ┌──────────┐
│  Decode   │───▶│ Validate  │───▶│ Instantiate  │───▶│  Execute  │
│ (解析)    │    │ (验证)    │    │ (实例化)     │    │ (执行)    │
└──────────┘    └──────────┘    └─────────────┘    └──────────┘
     │                │                │                  │
  二进制字节流     Module 结构体    ModuleInstance      调用导出函数
  → Module        类型检查通过     所有资源已分配       开始运行
```

**实例化（Instantiation）** 是中间的关键步骤。它将一个已验证的抽象 Module 转化为一个可执行的 **Module Instance**，过程中需要：

1. 解析所有导入，将外部提供的函数/内存/表/全局变量绑定到模块
2. 为模块声明的内存、表、全局变量分配实际资源
3. 用 Element Segments 初始化表（填充函数引用）
4. 用 Data Segments 初始化线性内存（填充初始数据）
5. 如果模块声明了 Start Function，则自动调用它

> **与 CLR 的直觉对比**：这类似于 CLR 中 `Assembly.Load()` 之后、在 `AppDomain` 中解析程序集引用、分配静态字段、执行 `.cctor`（类型初始化器）的过程。但 Wasm 更加显式和可控。

---

## 2. 实例化的完整生命周期

### 2.1 规范定义的实例化步骤

根据 WebAssembly 规范，`instantiate(store, module, externvals)` 的执行步骤如下：

```
instantiate(store, module, externvals):

  1. 验证 externvals 与 module.imports 的数量和类型匹配
     ├── 函数：签名必须完全匹配
     ├── 表：element type 匹配 + limits 兼容
     ├── 内存：limits 兼容
     └── 全局变量：值类型 + mutability 匹配

  2. 分配模块实例 (allocate)
     ├── 为每个本地函数创建 function instance
     ├── 为每个本地表创建 table instance
     ├── 为每个本地内存创建 memory instance
     ├── 为每个本地全局变量创建 global instance
     └── 为每个导出创建 export instance

  3. 初始化全局变量
     └── 对每个全局变量，求值其 init_expr 得到初始值

  4. 初始化 Element Segments
     ├── 对 active segment：求值 offset_expr，将函数引用写入表
     ├── 对 passive segment：暂存，等待 table.init 指令
     └── 对 declarative segment：仅声明，不实际初始化

  5. 初始化 Data Segments
     ├── 对 active segment：求值 offset_expr，将数据写入内存
     └── 对 passive segment：暂存，等待 memory.init 指令

  6. 如果存在 Start Function
     └── 调用 start function（无参数、无返回值）

  7. 返回 module instance
```

### 2.2 错误处理：实例化可能失败

实例化过程中的任何错误都应导致整个实例化失败，已分配的资源应被回收：

| 错误类型 | 触发条件 | 示例 |
|---------|---------|------|
| **Link Error** | 导入不匹配 | 期望导入函数签名 `(i32) → i32`，实际提供 `(i32, i32) → i32` |
| **Link Error** | 导入缺失 | 模块声明了 `import "env" "memory"`，但宿主未提供 |
| **Trap** | Element Segment 越界 | active element segment 的 offset + 元素数量超过表大小 |
| **Trap** | Data Segment 越界 | active data segment 的 offset + 数据长度超过内存大小 |
| **Trap** | Start Function 陷入 | start function 执行中触发了 unreachable 或其他 trap |

> **关键实现细节**：规范要求 element/data segment 的越界检查在**任何初始化写入之前**完成。也就是说，如果第 3 个 data segment 越界，那么第 1、2 个 data segment 也不应该被写入。这是一个原子性要求。

### 2.3 用 C 伪代码表示

```c
typedef struct {
    Module*         module;          // source module (shared, immutable)
    FuncInstance*   funcs;           // function instances (imported + local)
    TableInstance*  tables;          // table instances (imported + local)
    MemInstance*    mems;            // memory instances (imported + local)
    GlobalInstance* globals;         // global instances (imported + local)
    ExportInstance* exports;         // export instances
    uint32_t        num_funcs;
    uint32_t        num_tables;
    uint32_t        num_mems;
    uint32_t        num_globals;
    uint32_t        num_exports;
} ModuleInstance;

WasmError instantiate(Store* store, Module* module,
                       ExternVal* imports, uint32_t num_imports,
                       ModuleInstance** out_instance)
{
    // Step 1: Validate imports
    WasmError err = validate_imports(module, imports, num_imports);
    if (err) return err;

    // Step 2: Allocate module instance
    ModuleInstance* inst = allocate_module_instance(store, module, imports);
    if (!inst) return WASM_ERROR_OOM;

    // Step 3: Initialize globals (evaluate init exprs)
    err = init_globals(inst);
    if (err) goto cleanup;

    // Step 4 & 5: Bounds-check ALL segments first (atomicity)
    err = check_all_segments_bounds(inst);
    if (err) goto cleanup;

    // Step 4: Initialize element segments
    err = init_element_segments(inst);
    if (err) goto cleanup;

    // Step 5: Initialize data segments
    err = init_data_segments(inst);
    if (err) goto cleanup;

    // Step 6: Call start function if present
    if (module->has_start) {
        err = call_function(inst, module->start_func_idx, NULL, 0, NULL, 0);
        if (err) goto cleanup;
    }

    *out_instance = inst;
    return WASM_OK;

cleanup:
    free_module_instance(inst);
    return err;
}
```

---

## 3. Store —— 全局状态容器

### 3.1 概念

**Store** 是 Wasm 规范中定义的全局状态容器，持有所有运行时实例的集合：

```
Store = {
    funcs:    FuncInstance[]      // all function instances
    tables:   TableInstance[]     // all table instances
    mems:     MemInstance[]       // all memory instances
    globals:  GlobalInstance[]    // all global instances
    elems:    ElemInstance[]      // all element instances (for passive segments)
    datas:    DataInstance[]      // all data instances (for passive segments)
}
```

每种实例通过 **地址（address）** 引用，地址本质上就是 Store 中对应数组的索引：

```
funcaddr   = index into store.funcs
tableaddr  = index into store.tables
memaddr    = index into store.mems
globaladdr = index into store.globals
```

### 3.2 Module Instance 与 Store 的关系

```
┌─────────────────────────────────────────────────────┐
│                      Store                           │
│                                                      │
│  funcs:   [F0] [F1] [F2] [F3] [F4] [F5] [F6] ...  │
│  tables:  [T0] [T1] [T2] ...                        │
│  mems:    [M0] [M1] ...                              │
│  globals: [G0] [G1] [G2] [G3] ...                   │
│                                                      │
└─────────────────────────────────────────────────────┘
       ▲           ▲          ▲          ▲
       │           │          │          │
  ┌────┴───────────┴──────────┴──────────┴────┐
  │           ModuleInstance A                 │
  │  funcaddrs:   [0, 1, 2]    (F0,F1,F2)    │
  │  tableaddrs:  [0]          (T0)           │
  │  memaddrs:    [0]          (M0)           │
  │  globaladdrs: [0, 1]       (G0,G1)       │
  └───────────────────────────────────────────┘

  ┌───────────────────────────────────────────┐
  │           ModuleInstance B                 │
  │  funcaddrs:   [3, 4, 5, 6] (F3-F6)       │
  │  tableaddrs:  [1, 2]       (T1,T2)       │
  │  memaddrs:    [0]          (M0) ← shared! │
  │  globaladdrs: [2, 3]       (G2,G3)       │
  └───────────────────────────────────────────┘
```

注意：模块 B 的 `memaddrs[0]` 指向 `M0`，与模块 A 共享同一块内存。这就是 Wasm 模块间共享内存的机制。

### 3.3 实现建议

在实际实现中，Store 不一定需要作为一个显式的全局对象存在。常见的简化方案：

```c
// Approach 1: Explicit Store (closest to spec)
typedef struct {
    Vec_FuncInst   funcs;
    Vec_TableInst  tables;
    Vec_MemInst    mems;
    Vec_GlobalInst globals;
} Store;

// Approach 2: Instances owned by ModuleInstance (simpler, wasm3-style)
// Each ModuleInstance directly owns its instances.
// Shared resources (imported memory/table) use pointers.
typedef struct ModuleInstance {
    FuncInstance**   funcs;      // array of pointers (some point to imported)
    TableInstance**  tables;
    MemInstance**    mems;
    GlobalInstance** globals;
    // ...
} ModuleInstance;
```

wasm3 采用的是方案 2 的变体，更加实用。

---

## 4. 导入解析（Import Resolution）

> **wasm3 实现参考**：[m3_env.c - m3_LoadModule()](wasm3/source/m3_env.c#L430) — 加载模块时解析导入；[m3_bind.c - FindAndLinkFunction()](wasm3/source/m3_bind.c#L117) — 查找并链接导入函数；[m3_bind.c - m3_LinkRawFunction()](wasm3/source/m3_bind.c#L168) — 绑定宿主函数的公共 API。

### 4.1 导入的结构

每个导入由三部分组成：

```
Import = {
    module: string,    // module name (e.g., "env", "wasi_snapshot_preview1")
    name:   string,    // field name (e.g., "memory", "fd_write")
    desc:   ImportDesc  // type descriptor
}
```

`ImportDesc` 是以下四种之一：

| 导入类型 | 描述符内容 | 示例 WAT |
|---------|-----------|---------|
| **func** | `typeidx` (函数签名索引) | `(import "env" "log" (func $log (type 0)))` |
| **table** | `tabletype` (元素类型 + limits) | `(import "env" "tbl" (table 10 funcref))` |
| **memory** | `memtype` (limits) | `(import "env" "mem" (memory 1 4))` |
| **global** | `globaltype` (值类型 + mut) | `(import "env" "g" (global (mut i32)))` |

### 4.2 类型匹配规则

导入解析的核心是**类型匹配**。规范定义了精确的匹配规则：

#### 函数导入匹配

```
match_func(expected_type, actual_type):
    // Exact match required
    expected.params == actual.params  AND
    expected.results == actual.results
```

函数签名必须**完全一致**，没有子类型关系（与 CLR 的委托协变/逆变不同）。

#### 表导入匹配

```
match_table(expected, actual):
    expected.elem_type == actual.elem_type  AND
    match_limits(expected.limits, actual.limits)
```

#### 内存导入匹配

```
match_memory(expected, actual):
    match_limits(expected.limits, actual.limits)
```

#### Limits 匹配（表和内存共用）

```
match_limits(expected, actual):
    actual.min >= expected.min  AND
    (expected.max == none  OR
     (actual.max != none  AND  actual.max <= expected.max))
```

直觉：实际提供的资源必须**至少和期望的一样大**（min），但**不超过期望的上限**（max）。

```
Expected: min=1, max=4    Actual: min=2, max=3   → ✓ Match
Expected: min=1, max=4    Actual: min=0, max=4   → ✗ (actual.min < expected.min)
Expected: min=1, max=4    Actual: min=2, max=5   → ✗ (actual.max > expected.max)
Expected: min=1, max=none Actual: min=2, max=10  → ✓ (no max constraint)
Expected: min=1, max=4    Actual: min=2, max=none→ ✗ (actual has no max, but expected requires one)
```

#### 全局变量导入匹配

```
match_global(expected, actual):
    expected.valtype == actual.valtype  AND
    expected.mutability == actual.mutability
```

全局变量的可变性（mut/const）也必须完全匹配。

### 4.3 实现：导入解析器

```c
typedef struct {
    const char* module_name;
    const char* field_name;
    ExternType  type;       // FUNC / TABLE / MEMORY / GLOBAL
    union {
        FuncType*   func_type;
        TableType   table_type;
        MemType     mem_type;
        GlobalType  global_type;
    };
    void* value;  // pointer to the actual instance
} ExternVal;

WasmError resolve_imports(Module* module, ExternVal* provided, uint32_t num_provided)
{
    for (uint32_t i = 0; i < module->num_imports; i++) {
        Import* imp = &module->imports[i];

        // Find matching extern val
        ExternVal* ext = find_extern(provided, num_provided,
                                      imp->module_name, imp->field_name);
        if (!ext) {
            return error("import not found: %s.%s", imp->module_name, imp->field_name);
        }

        // Type check
        if (ext->type != imp->desc.type) {
            return error("import type mismatch: %s.%s", imp->module_name, imp->field_name);
        }

        switch (imp->desc.type) {
        case EXTERN_FUNC:
            if (!func_types_equal(module->types[imp->desc.func.typeidx], ext->func_type))
                return error("func signature mismatch");
            break;
        case EXTERN_TABLE:
            if (!match_table_type(&imp->desc.table, &ext->table_type))
                return error("table type mismatch");
            break;
        case EXTERN_MEMORY:
            if (!match_limits(&imp->desc.memory.limits, &ext->mem_type.limits))
                return error("memory limits mismatch");
            break;
        case EXTERN_GLOBAL:
            if (!match_global_type(&imp->desc.global, &ext->global_type))
                return error("global type mismatch");
            break;
        }
    }
    return WASM_OK;
}
```

---

## 5. 实例分配（Allocation）

> **wasm3 实现参考**：[m3_env.c - m3_LoadModule()](wasm3/source/m3_env.c#L430) — 将解析后的模块加载到运行时，分配内存/全局变量等资源；[m3_env.c - ResizeMemory()](wasm3/source/m3_env.c#L262) — 内存实例分配与增长。

### 5.1 函数实例

```c
typedef struct {
    FuncType*        type;       // function signature
    ModuleInstance*  module;     // owning module instance (for closures)
    union {
        // Wasm function
        struct {
            uint8_t* code;       // pointer to bytecode
            uint32_t code_len;
            LocalDecl* locals;   // local variable declarations
            uint32_t num_locals;
        } wasm;
        // Host function
        struct {
            HostFuncCallback callback;
            void* userdata;
        } host;
    };
    bool is_host;
} FuncInstance;
```

每个 Wasm 函数实例持有对其所属 `ModuleInstance` 的引用。这是因为函数体中的指令可能引用模块的全局变量、内存、表等，需要通过 `ModuleInstance` 来定位这些资源。

> **CLR 类比**：这类似于 CLR 中 `MethodInfo` 持有对 `Type` 和 `Assembly` 的引用，方法执行时需要通过这些引用来解析 metadata token。

### 5.2 表实例

```c
typedef struct {
    RefType     elem_type;   // funcref or externref
    Ref*        elems;       // array of references
    uint32_t    size;        // current size
    uint32_t    max;         // maximum size (UINT32_MAX if no limit)
} TableInstance;

// Allocate a new table
TableInstance* alloc_table(TableType* type) {
    TableInstance* t = calloc(1, sizeof(TableInstance));
    t->elem_type = type->elem_type;
    t->size = type->limits.min;
    t->max = type->limits.has_max ? type->limits.max : UINT32_MAX;
    t->elems = calloc(t->size, sizeof(Ref));
    // All elements initialized to ref.null
    for (uint32_t i = 0; i < t->size; i++) {
        t->elems[i] = REF_NULL;
    }
    return t;
}
```

表的元素初始值为 `ref.null`。调用 `call_indirect` 时如果目标元素是 `ref.null`，会触发 trap。

### 5.3 内存实例

```c
typedef struct {
    uint8_t*    data;        // linear memory buffer
    uint32_t    size;        // current size in pages (1 page = 64KB)
    uint32_t    max;         // maximum pages (UINT32_MAX if no limit)
} MemInstance;

#define WASM_PAGE_SIZE 65536  // 64 KiB

MemInstance* alloc_memory(MemType* type) {
    MemInstance* m = calloc(1, sizeof(MemInstance));
    m->size = type->limits.min;
    m->max = type->limits.has_max ? type->limits.max : 65536; // spec max: 4GB
    m->data = calloc(m->size, WASM_PAGE_SIZE);  // zero-initialized
    return m;
}

// memory.grow implementation
int32_t memory_grow(MemInstance* mem, uint32_t delta) {
    uint32_t old_size = mem->size;
    uint64_t new_size = (uint64_t)old_size + delta;

    if (new_size > mem->max || new_size > 65536) {
        return -1;  // failure
    }

    uint8_t* new_data = realloc(mem->data, (uint32_t)new_size * WASM_PAGE_SIZE);
    if (!new_data) return -1;

    // Zero-initialize new pages
    memset(new_data + old_size * WASM_PAGE_SIZE, 0, delta * WASM_PAGE_SIZE);

    mem->data = new_data;
    mem->size = (uint32_t)new_size;
    return (int32_t)old_size;  // return previous size on success
}
```

### 5.4 全局变量实例

```c
typedef struct {
    ValType     type;        // i32 / i64 / f32 / f64 / funcref / externref
    bool        mutable_;    // is mutable?
    Value       value;       // current value
} GlobalInstance;
```

全局变量的初始值由 **初始化表达式（init expression / constant expression）** 决定。

### 5.5 初始化表达式求值

> **wasm3 实现参考**：[m3_env.c - EvaluateExpression()](wasm3/source/m3_env.c#L210) — 求值初始化表达式（支持 `*.const`、`global.get`、`ref.func` 等）。

初始化表达式是一个受限的指令序列，只允许以下指令：

| 指令 | 用途 |
|------|------|
| `i32.const`, `i64.const`, `f32.const`, `f64.const` | 常量值 |
| `global.get` | 引用已导入的不可变全局变量 |
| `ref.null` | 空引用 |
| `ref.func` | 函数引用 |
| `end` | 表达式结束标记 |

```c
Value eval_init_expr(ModuleInstance* inst, uint8_t* expr, uint32_t len) {
    Value result;
    uint8_t* pc = expr;

    switch (*pc) {
    case 0x41: // i32.const
        result.type = VAL_I32;
        result.i32 = read_leb128_i32(&pc);
        break;
    case 0x42: // i64.const
        result.type = VAL_I64;
        result.i64 = read_leb128_i64(&pc);
        break;
    case 0x43: // f32.const
        result.type = VAL_F32;
        memcpy(&result.f32, pc + 1, 4);
        break;
    case 0x44: // f64.const
        result.type = VAL_F64;
        memcpy(&result.f64, pc + 1, 8);
        break;
    case 0x23: // global.get
    {
        uint32_t idx = read_leb128_u32(&pc);
        result = inst->globals[idx]->value;
        break;
    }
    case 0xD0: // ref.null
        result.type = VAL_REF;
        result.ref = REF_NULL;
        break;
    case 0xD2: // ref.func
    {
        uint32_t idx = read_leb128_u32(&pc);
        result.type = VAL_REF;
        result.ref = make_func_ref(inst, idx);
        break;
    }
    }
    // Next byte should be 0x0B (end)
    return result;
}
```

> **注意**：`global.get` 在初始化表达式中只能引用**已导入的不可变全局变量**。这确保了初始化顺序不会产生循环依赖。

---

## 6. 初始化：Element 与 Data Segments

> **wasm3 实现参考**：[m3_env.c - InitElements()](wasm3/source/m3_env.c#L340) — 初始化元素段（将函数引用填充到 table0）；[m3_env.c - InitDataSegments()](wasm3/source/m3_env.c#L316) — 初始化数据段（将数据复制到线性内存）。

### 6.1 Element Segments

Element Segments 用于初始化表（Table）的内容，主要是填充函数引用。

Wasm 2.0 定义了三种 Element Segment 模式：

| 模式 | 行为 | 用途 |
|------|------|------|
| **Active** | 实例化时自动将元素复制到指定表的指定偏移处 | 静态初始化函数表 |
| **Passive** | 不自动初始化，等待 `table.init` 指令 | 延迟初始化、按需加载 |
| **Declarative** | 仅声明函数引用的存在，不实际初始化任何表 | 为 `ref.func` 提供前向声明 |

#### Active Element Segment 初始化

```c
WasmError init_active_elem_segment(ModuleInstance* inst, ElemSegment* seg) {
    // Evaluate offset expression
    Value offset_val = eval_init_expr(inst, seg->offset_expr, seg->offset_expr_len);
    uint32_t offset = offset_val.i32;

    // Get target table
    TableInstance* table = inst->tables[seg->table_idx];

    // Bounds check
    if ((uint64_t)offset + seg->num_elems > table->size) {
        return WASM_TRAP_TABLE_OOB;
    }

    // Copy elements
    for (uint32_t i = 0; i < seg->num_elems; i++) {
        Ref ref = eval_elem_expr(inst, &seg->elems[i]);
        table->elems[offset + i] = ref;
    }

    // After initialization, active segment is "dropped"
    seg->dropped = true;

    return WASM_OK;
}
```

#### Passive Element Segment 与 table.init / elem.drop

```wasm
;; Passive element segment (not auto-initialized)
(elem $seg funcref (ref.func $f1) (ref.func $f2) (ref.func $f3))

;; Later, in code: copy 3 elements from segment $seg offset 0 to table 0 offset 5
(table.init $seg
  (i32.const 5)    ;; destination offset in table
  (i32.const 0)    ;; source offset in segment
  (i32.const 3))   ;; number of elements

;; After use, drop the segment to free memory
(elem.drop $seg)
```

### 6.2 Data Segments

Data Segments 用于初始化线性内存，与 Element Segments 类似，也有 Active 和 Passive 两种模式。

#### Active Data Segment 初始化

```c
WasmError init_active_data_segment(ModuleInstance* inst, DataSegment* seg) {
    // Evaluate offset expression
    Value offset_val = eval_init_expr(inst, seg->offset_expr, seg->offset_expr_len);
    uint32_t offset = offset_val.i32;

    // Get target memory (currently always memory 0)
    MemInstance* mem = inst->mems[seg->mem_idx];

    // Bounds check
    uint64_t end = (uint64_t)offset + seg->data_len;
    if (end > (uint64_t)mem->size * WASM_PAGE_SIZE) {
        return WASM_TRAP_MEMORY_OOB;
    }

    // Copy data
    memcpy(mem->data + offset, seg->data, seg->data_len);

    // After initialization, active segment is "dropped"
    seg->dropped = true;

    return WASM_OK;
}
```

### 6.3 原子性要求

**关键实现细节**：规范要求所有 active segment 的边界检查必须在任何写入之前完成：

```c
WasmError init_all_segments(ModuleInstance* inst) {
    Module* mod = inst->module;

    // Phase 1: Bounds-check ALL active segments first
    for (uint32_t i = 0; i < mod->num_elem_segments; i++) {
        if (mod->elem_segments[i].mode == ELEM_ACTIVE) {
            WasmError err = check_elem_bounds(inst, &mod->elem_segments[i]);
            if (err) return err;  // Fail before any writes
        }
    }
    for (uint32_t i = 0; i < mod->num_data_segments; i++) {
        if (mod->data_segments[i].mode == DATA_ACTIVE) {
            WasmError err = check_data_bounds(inst, &mod->data_segments[i]);
            if (err) return err;  // Fail before any writes
        }
    }

    // Phase 2: Now perform all writes (guaranteed to succeed)
    for (uint32_t i = 0; i < mod->num_elem_segments; i++) {
        if (mod->elem_segments[i].mode == ELEM_ACTIVE) {
            init_active_elem_segment(inst, &mod->elem_segments[i]);
        }
    }
    for (uint32_t i = 0; i < mod->num_data_segments; i++) {
        if (mod->data_segments[i].mode == DATA_ACTIVE) {
            init_active_data_segment(inst, &mod->data_segments[i]);
        }
    }

    return WASM_OK;
}
```

---

## 7. 导出机制（Exports）

> **wasm3 实现参考**：[m3_env.c - m3_FindFunction()](wasm3/source/m3_env.c#L505) — 通过名称查找导出函数；[m3_env.c - m3_GetFunctionName()](wasm3/source/m3_env.c#L487) — 获取函数名称。

### 7.1 导出的结构

导出是模块向外部暴露的接口：

```
Export = {
    name: string,       // export name
    desc: ExportDesc    // what is being exported
}

ExportDesc = func <funcidx>
           | table <tableidx>
           | memory <memidx>
           | global <globalidx>
```

### 7.2 导出查找

宿主通过名称查找导出：

```c
ExternVal* find_export(ModuleInstance* inst, const char* name) {
    for (uint32_t i = 0; i < inst->num_exports; i++) {
        if (strcmp(inst->exports[i].name, name) == 0) {
            return &inst->exports[i].value;
        }
    }
    return NULL;  // not found
}

// Typical usage: find and call an exported function
ExternVal* main_fn = find_export(instance, "main");
if (main_fn && main_fn->type == EXTERN_FUNC) {
    Value args[] = { {.i32 = argc}, {.i32 = argv_ptr} };
    Value results[1];
    call_function(instance, main_fn->func_addr, args, 2, results, 1);
    int exit_code = results[0].i32;
}
```

### 7.3 导出的重要规则

1. **导出名称必须唯一**：同一模块不能有两个同名导出（验证阶段检查）
2. **导出可以是导入的再导出**：模块可以导入一个函数然后再导出它
3. **导出的内存/表是共享的**：如果模块 A 导出内存，模块 B 导入它，两者操作的是同一块内存

```wasm
;; Module A: exports its memory
(module
  (memory (export "shared_mem") 1 10)
  (func (export "write") (param i32 i32)
    (i32.store (local.get 0) (local.get 1))))

;; Module B: imports A's memory, can read what A wrote
(module
  (import "A" "shared_mem" (memory 1))
  (func (export "read") (param i32) (result i32)
    (i32.load (local.get 0))))
```

---

## 8. 宿主函数绑定（Host Functions）

> **wasm3 实现参考**：[m3_bind.c - m3_LinkRawFunction()](wasm3/source/m3_bind.c#L168) — 绑定宿主函数的主要 API；[m3_bind.c - SignatureToFuncType()](wasm3/source/m3_bind.c#L29) — 将签名字符串（如 `"i(ii)"`）解析为 `M3FuncType`；[wasm3.h - m3_LinkRawFunctionEx()](wasm3/source/wasm3.h#L260) — 带 userdata 的绑定 API。

### 8.1 设计模式

宿主函数是由嵌入环境（C/C++/Rust 等）实现的函数，通过导入机制提供给 Wasm 模块调用。设计宿主函数绑定接口是实现 Wasm 运行时的关键决策之一。

#### 方案 A：统一回调签名（简单但有类型转换开销）

```c
// All host functions have the same C signature
typedef WasmError (*HostFunc)(
    void*    userdata,       // user-provided context
    Value*   args,           // input arguments
    uint32_t num_args,
    Value*   results,        // output results
    uint32_t num_results
);

// Registration
void register_host_func(Runtime* rt,
                         const char* module, const char* name,
                         FuncType* type, HostFunc func, void* userdata);

// Example: implementing "env.print_i32"
WasmError host_print_i32(void* ud, Value* args, uint32_t nargs,
                          Value* results, uint32_t nresults) {
    printf("%d\n", args[0].i32);
    return WASM_OK;
}

register_host_func(rt, "env", "print_i32",
                    make_func_type(1, (ValType[]){VAL_I32}, 0, NULL),
                    host_print_i32, NULL);
```

#### 方案 B：类型化绑定（wasm3 风格，更高效）

```c
// wasm3-style: host function directly receives typed arguments
// The runtime generates a "trampoline" that unpacks stack values

// User writes a normal C function:
int32_t host_add(int32_t a, int32_t b) {
    return a + b;
}

// Bind with type information:
m3_LinkRawFunction(module, "env", "add", "i(ii)", &host_add);
//                                        ^^^^
//                                        signature string:
//                                        return i32, params (i32, i32)
```

wasm3 使用签名字符串来描述函数类型，并在内部生成适配代码将栈上的值转换为 C 函数参数。

#### 方案 C：带内存访问的宿主函数

很多宿主函数需要访问 Wasm 线性内存（例如读写字符串）：

```c
// Host function that reads a string from Wasm memory
WasmError host_log_string(void* ud, Value* args, uint32_t nargs,
                           Value* results, uint32_t nresults) {
    Runtime* rt = (Runtime*)ud;
    MemInstance* mem = get_memory(rt, 0);

    uint32_t ptr = args[0].i32;  // pointer into Wasm memory
    uint32_t len = args[1].i32;  // string length

    // Bounds check!
    if ((uint64_t)ptr + len > mem->size * WASM_PAGE_SIZE) {
        return WASM_TRAP_MEMORY_OOB;
    }

    // Safe to access
    printf("%.*s\n", len, (char*)(mem->data + ptr));
    return WASM_OK;
}
```

### 8.2 Trap 传播

宿主函数可以触发 trap，trap 需要正确传播回 Wasm 调用栈：

```c
WasmError host_div(void* ud, Value* args, uint32_t nargs,
                    Value* results, uint32_t nresults) {
    if (args[1].i32 == 0) {
        return WASM_TRAP_DIV_BY_ZERO;  // propagates as trap
    }
    results[0].i32 = args[0].i32 / args[1].i32;
    return WASM_OK;
}
```

在执行引擎中，调用宿主函数后需要检查返回值：

```c
case OP_CALL:
{
    FuncInstance* callee = resolve_func(inst, func_idx);
    if (callee->is_host) {
        WasmError err = callee->host.callback(
            callee->host.userdata, args, nargs, results, nresults);
        if (err) return err;  // trap propagation
    } else {
        // enter Wasm function...
    }
    break;
}
```

### 8.3 设计考量

| 考量 | 说明 |
|------|------|
| **安全性** | 宿主函数必须验证所有来自 Wasm 的指针/索引，Wasm 代码不可信 |
| **重入** | 宿主函数可能回调 Wasm 函数（如 callback 模式），需要支持重入 |
| **错误处理** | 宿主函数的错误应转化为 Wasm trap，而非 C 层面的 crash |
| **生命周期** | 宿主函数的 userdata 生命周期必须覆盖整个 Runtime 生命周期 |
| **线程安全** | 如果支持多线程，宿主函数需要考虑并发访问 |

---

## 9. WASI —— 系统接口标准

> **wasm3 实现参考**：[m3_api_wasi.c - m3_LinkWASI()](wasm3/source/m3_api_wasi.c#L772) — 注册所有 WASI 函数的入口；[m3_api_wasi.c - m3_wasi_generic_fd_write()](wasm3/source/m3_api_wasi.c#L455) — `fd_write` 实现；[m3_api_wasi.c - m3_wasi_generic_fd_read()](wasm3/source/m3_api_wasi.c#L413) — `fd_read` 实现。

### 9.1 什么是 WASI

**WASI（WebAssembly System Interface）** 是一套标准化的系统调用接口，让 Wasm 模块能够以可移植的方式访问操作系统功能（文件 I/O、环境变量、时钟等）。

WASI 本质上就是一组预定义的宿主函数，模块通过标准的导入名称来使用：

```wasm
(import "wasi_snapshot_preview1" "fd_write"
  (func $fd_write (param i32 i32 i32 i32) (result i32)))
(import "wasi_snapshot_preview1" "fd_read"
  (func $fd_read (param i32 i32 i32 i32) (result i32)))
(import "wasi_snapshot_preview1" "proc_exit"
  (func $proc_exit (param i32)))
```

### 9.2 核心 WASI 函数

以下是实现一个最小可用 WASI 运行时需要的核心函数：

| 函数 | 签名 | 用途 |
|------|------|------|
| `fd_write` | `(fd, iovs, iovs_len, nwritten) → errno` | 写入文件描述符（stdout/stderr） |
| `fd_read` | `(fd, iovs, iovs_len, nread) → errno` | 从文件描述符读取 |
| `fd_close` | `(fd) → errno` | 关闭文件描述符 |
| `fd_seek` | `(fd, offset, whence, newoffset) → errno` | 文件定位 |
| `fd_prestat_get` | `(fd, buf) → errno` | 获取预打开目录信息 |
| `fd_prestat_dir_name` | `(fd, path, path_len) → errno` | 获取预打开目录名 |
| `path_open` | `(fd, dirflags, path, ...) → errno` | 打开文件 |
| `args_get` | `(argv, argv_buf) → errno` | 获取命令行参数 |
| `args_sizes_get` | `(argc, argv_buf_size) → errno` | 获取参数大小 |
| `environ_get` | `(environ, environ_buf) → errno` | 获取环境变量 |
| `environ_sizes_get` | `(count, buf_size) → errno` | 获取环境变量大小 |
| `clock_time_get` | `(id, precision, time) → errno` | 获取时钟时间 |
| `proc_exit` | `(code)` | 退出进程 |

### 9.3 实现 fd_write 示例

`fd_write` 是最常用的 WASI 函数（`printf` 最终会调用它）：

```c
// WASI iovec structure (in Wasm linear memory)
// struct iovec { uint32_t buf; uint32_t buf_len; }  // 8 bytes each

WasmError wasi_fd_write(void* ctx, Value* args, uint32_t nargs,
                         Value* results, uint32_t nresults) {
    WasiCtx* wasi = (WasiCtx*)ctx;
    MemInstance* mem = wasi->memory;

    uint32_t fd          = args[0].i32;
    uint32_t iovs_ptr    = args[1].i32;  // pointer to iovec array in Wasm memory
    uint32_t iovs_len    = args[2].i32;  // number of iovecs
    uint32_t nwritten_ptr = args[3].i32; // where to write bytes-written count

    // Validate fd (only stdout=1 and stderr=2 for minimal impl)
    int host_fd;
    switch (fd) {
        case 1: host_fd = STDOUT_FILENO; break;
        case 2: host_fd = STDERR_FILENO; break;
        default:
            results[0].i32 = WASI_ERRNO_BADF;
            return WASM_OK;
    }

    uint32_t total_written = 0;

    for (uint32_t i = 0; i < iovs_len; i++) {
        uint32_t iov_addr = iovs_ptr + i * 8;

        // Bounds check for iovec struct
        if (iov_addr + 8 > mem->size * WASM_PAGE_SIZE)
            return WASM_TRAP_MEMORY_OOB;

        // Read iovec fields (little-endian)
        uint32_t buf_ptr = read_u32_le(mem->data + iov_addr);
        uint32_t buf_len = read_u32_le(mem->data + iov_addr + 4);

        // Bounds check for buffer
        if ((uint64_t)buf_ptr + buf_len > (uint64_t)mem->size * WASM_PAGE_SIZE)
            return WASM_TRAP_MEMORY_OOB;

        // Write to host fd
        ssize_t written = write(host_fd, mem->data + buf_ptr, buf_len);
        if (written < 0) {
            results[0].i32 = errno_to_wasi(errno);
            return WASM_OK;
        }
        total_written += (uint32_t)written;
    }

    // Write number of bytes written
    if (nwritten_ptr + 4 > mem->size * WASM_PAGE_SIZE)
        return WASM_TRAP_MEMORY_OOB;
    write_u32_le(mem->data + nwritten_ptr, total_written);

    results[0].i32 = WASI_ERRNO_SUCCESS;
    return WASM_OK;
}
```

### 9.4 WASI 的能力模型（Capability-based Security）

WASI 采用**基于能力的安全模型**：

- Wasm 模块不能直接访问文件系统路径
- 宿主通过 **预打开目录（preopened directories）** 授予有限的文件系统访问权限
- 每个文件描述符都有关联的 **权限集（rights）**，限制可执行的操作

```c
typedef struct {
    int         host_fd;        // actual OS file descriptor
    uint64_t    rights_base;    // allowed operations
    uint64_t    rights_inherit; // rights for newly opened files
    char*       preopen_path;   // preopened directory path (if applicable)
} WasiFd;

typedef struct {
    MemInstance*  memory;       // reference to Wasm linear memory
    WasiFd*       fd_table;    // file descriptor table
    uint32_t      fd_count;
    char**        args;         // command line arguments
    uint32_t      argc;
    char**        env;          // environment variables
    uint32_t      envc;
} WasiCtx;
```

---

## 10. 多模块链接

### 10.1 模块间链接

多个 Wasm 模块可以通过导入/导出机制链接在一起：

```
┌─────────────┐     export "add"      ┌─────────────┐
│  Module A    │─────────────────────▶│  Module B    │
│              │     import "A"."add"  │              │
│  (math lib)  │                      │  (app logic) │
│              │◀─────────────────────│              │
│              │     export "memory"   │              │
│              │     import "A"."mem"  │              │
└─────────────┘                       └─────────────┘
```

### 10.2 链接顺序

模块必须按依赖顺序实例化：

```c
// Step 1: Instantiate module A (no imports, or only host imports)
ModuleInstance* inst_a;
instantiate(store, module_a, host_imports, num_host_imports, &inst_a);

// Step 2: Collect A's exports as B's imports
ExternVal b_imports[2];

ExternVal* add_export = find_export(inst_a, "add");
b_imports[0] = (ExternVal){
    .module_name = "A", .field_name = "add",
    .type = EXTERN_FUNC, .value = add_export->value
};

ExternVal* mem_export = find_export(inst_a, "memory");
b_imports[1] = (ExternVal){
    .module_name = "A", .field_name = "memory",
    .type = EXTERN_MEMORY, .value = mem_export->value
};

// Step 3: Instantiate module B with A's exports
ModuleInstance* inst_b;
instantiate(store, module_b, b_imports, 2, &inst_b);
```

### 10.3 循环依赖

Wasm 规范**不直接支持**模块间的循环依赖。如果模块 A 导入 B 的函数，同时 B 也导入 A 的函数，则无法按上述顺序实例化。

解决方案：
1. **重构消除循环**：将共享接口提取到第三个模块
2. **两阶段实例化**：先创建占位实例，再回填（非标准，需运行时支持）
3. **使用表间接调用**：通过共享表和 `call_indirect` 实现延迟绑定

---

## 11. CLR 对比：Assembly 加载与 AppDomain

### 11.1 生命周期对比

```
┌──────────────────────────────────────────────────────────────────┐
│                        Wasm                                      │
│                                                                  │
│  .wasm bytes → decode → Module → validate → instantiate          │
│                                     │            │               │
│                                     │     ModuleInstance         │
│                                     │     (owns resources)       │
│                                     │            │               │
│                                     │     call exported func     │
│                                     │            │               │
│                                     │     destroy instance       │
│                                     │     (free all resources)   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                        CLR                                       │
│                                                                  │
│  .dll/.exe → PE parse → Assembly → load into AppDomain           │
│                                        │                         │
│                                  resolve references              │
│                                  (other assemblies)              │
│                                        │                         │
│                                  JIT compile on first call       │
│                                        │                         │
│                                  execute                         │
│                                        │                         │
│                                  AppDomain.Unload()              │
│                                  (unload all assemblies)         │
└──────────────────────────────────────────────────────────────────┘
```

### 11.2 关键差异

| 维度 | Wasm | CLR |
|------|------|-----|
| **加载单元** | Module（单文件） | Assembly（可含多个 Module） |
| **导入解析** | 显式，由嵌入者提供 externvals | 隐式，CLR 自动探测和加载依赖程序集 |
| **类型匹配** | 结构化类型匹配（签名相同即匹配） | 名义类型匹配（必须是同一类型定义） |
| **内存模型** | 显式线性内存，手动管理 | 托管堆 + GC，自动管理 |
| **实例化** | 同一 Module 可创建多个独立 Instance | Assembly 在 AppDomain 中只加载一次 |
| **隔离** | Instance 级别隔离（各自独立的内存/表/全局变量） | AppDomain 级别隔离（.NET Core 已移除） |
| **卸载** | 销毁 Instance 即可释放所有资源 | 需要卸载整个 AppDomain（或 AssemblyLoadContext） |
| **静态状态** | 全局变量属于 Instance，多实例互不影响 | 静态字段属于 Type，同 AppDomain 内共享 |
| **初始化** | Start Function（可选，最多一个） | 类型初始化器 `.cctor`（每个类型一个，惰性执行） |

### 11.3 多实例化 —— Wasm 的独特优势

Wasm 允许同一个 Module 创建多个完全独立的 Instance：

```c
// Same module, two independent instances with separate memory
ModuleInstance* inst1;
ModuleInstance* inst2;

instantiate(store, module, imports_1, n, &inst1);  // inst1 has its own memory
instantiate(store, module, imports_2, n, &inst2);  // inst2 has its own memory

// Writing to inst1's memory does NOT affect inst2
```

这在 CLR 中没有直接对应物。CLR 的静态字段在同一 AppDomain 内是共享的，无法为同一类型创建多个独立的静态状态副本。

这个特性使 Wasm 非常适合：
- **沙箱化**：每个租户一个实例，完全隔离
- **快照/恢复**：保存实例状态，稍后恢复
- **并行执行**：多个实例可安全并行运行（无共享可变状态）

### 11.4 导入解析对比

```
CLR Assembly Resolution:
  1. 读取 AssemblyRef 表
  2. CLR Fusion/Binder 按名称+版本+PublicKeyToken 搜索
  3. 在 GAC / 应用目录 / 探测路径中查找
  4. 自动加载并递归解析依赖
  → 开发者通常不需要手动干预

Wasm Import Resolution:
  1. 读取 Import Section
  2. 嵌入者（embedder）必须显式提供每个导入
  3. 运行时逐一进行类型匹配检查
  4. 任何不匹配都导致实例化失败
  → 嵌入者完全控制模块能访问什么
```

Wasm 的显式导入解析是其安全模型的基石：模块无法访问任何未被显式授予的资源。

---

## 12. 实现要点与常见陷阱

### 12.1 实现清单

实现模块实例化时，确保覆盖以下要点：

- [ ] **导入数量检查**：提供的 externvals 数量必须与模块的 imports 数量一致
- [ ] **导入顺序**：externvals 的顺序必须与模块 Import Section 中的声明顺序一致
- [ ] **Limits 匹配方向**：是 `actual.min >= expected.min`，不是 `<=`
- [ ] **全局变量 mutability 匹配**：const 和 mut 必须完全一致
- [ ] **Segment 原子性**：所有 active segment 的边界检查必须在任何写入之前完成
- [ ] **Init expr 中的 global.get**：只能引用已导入的不可变全局变量
- [ ] **Start function 签名**：必须是 `() → ()`（无参数无返回值）
- [ ] **导出名唯一性**：同一模块不能有重复的导出名
- [ ] **memory.grow 失败处理**：返回 -1 而非 trap
- [ ] **表元素初始值**：新分配的表元素为 `ref.null`

### 12.2 常见陷阱

#### 陷阱 1：忘记处理导入的索引偏移

模块的函数索引空间中，**导入函数排在前面**：

```
Function Index Space:
  [0]  imported func "env.log"      ← import
  [1]  imported func "env.malloc"   ← import
  [2]  local func $add              ← defined in Code Section[0]
  [3]  local func $main             ← defined in Code Section[1]
```

Code Section 中的第 0 个函数体对应的是函数索引 2（不是 0）。这个偏移量等于导入函数的数量。表、内存、全局变量同理。

```c
// WRONG: directly using code section index
FuncInstance* get_func(ModuleInstance* inst, uint32_t code_idx) {
    return &inst->local_funcs[code_idx];  // ✗
}

// CORRECT: account for import offset
FuncInstance* get_func(ModuleInstance* inst, uint32_t func_idx) {
    if (func_idx < inst->num_imported_funcs) {
        return inst->imported_funcs[func_idx];
    } else {
        return &inst->local_funcs[func_idx - inst->num_imported_funcs];
    }
}
```

#### 陷阱 2：Data Segment 的 memory index

在 MVP 中只有一个内存（index 0），但二进制格式中 active data segment 仍然编码了 memory index。不要硬编码为 0，要正确解析。

#### 陷阱 3：Element Segment 的多种编码格式

Wasm 2.0 的 Element Segment 有 8 种不同的二进制编码格式（由 flags 字段的低 3 位决定），这是解析中最复杂的部分之一。务必参考规范的 §5.5.12 完整实现。

#### 陷阱 4：Start Function 中的 trap

如果 start function 触发 trap，整个实例化应该失败。已分配的资源需要被正确清理。

```c
// In instantiate():
if (module->has_start) {
    WasmError err = call_function(inst, module->start_func_idx, NULL, 0, NULL, 0);
    if (err) {
        // Start function trapped - instantiation fails
        // Must clean up all allocated resources
        free_module_instance(inst);
        return err;
    }
}
```

### 12.3 性能优化建议

1. **延迟编译**：不需要在实例化时编译所有函数，可以在首次调用时再编译（类似 CLR 的 JIT）
2. **共享 Module**：Module 是不可变的，可以被多个 Instance 共享，避免重复解析和验证
3. **内存池**：频繁创建/销毁实例时，使用内存池复用 MemInstance 的缓冲区
4. **Copy-on-Write**：对于 Data Segment 的初始化，可以使用 CoW 页面映射，避免大量数据复制
5. **预计算导出表**：将导出名构建为哈希表，加速导出查找

---

## 下一步

理解了模块实例化与链接机制后，你已经掌握了实现一个完整 Wasm 运行时所需的所有核心概念。接下来：

- → [第六章：wasm3 源码导读](06-wasm3-analysis.md)：看看一个成熟的实现是如何组织这些概念的
- → [第七章：实践项目指南](07-practice-guide.md)：开始动手实现你自己的解释器
