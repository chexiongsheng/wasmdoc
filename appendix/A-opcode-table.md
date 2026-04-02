# 附录 A：WebAssembly 操作码速查表

> 本表基于 WebAssembly Core Specification 2.0，涵盖 MVP 及主要扩展提案中的所有操作码。
>
> **wasm3 源码参考**：[m3_compile.c - c_operations](../wasm3/source/m3_compile.c#L2260) — 完整的操作码到 M3OpInfo 映射表（包含栈偏移、类型、operation 函数指针、编译器函数）；[m3_compile.c - c_operationsFC](../wasm3/source/m3_compile.c#L2527) — `0xFC` 前缀扩展操作码映射表；[m3_exec.h](../wasm3/source/m3_exec.h) — 所有 M3 operation 的运行时实现。
>
> **约定说明：**
> - **立即数**：跟随操作码之后的固定格式参数，在解码阶段读取
> - **栈签名**：`[inputs] → [outputs]`，描述指令对操作数栈的影响
> - `t` 表示任意值类型，`t*` 表示零或多个值
> - LEB128 编码的整数标记为 `u32`（无符号）或 `s32`/`s64`（有符号）

---

## 1. 控制流指令（Control Instructions）

> **wasm3 实现参考**：编译阶段见 [m3_compile.c - Compile_LoopOrBlock()](../wasm3/source/m3_compile.c#L1851)、[Compile_If()](../wasm3/source/m3_compile.c#L1869)、[Compile_Branch()](../wasm3/source/m3_compile.c#L1533)、[Compile_Call()](../wasm3/source/m3_compile.c#L1705)；运行时见 [m3_exec.h - op_Loop()](../wasm3/source/m3_exec.h#L864)、[op_Branch()](../wasm3/source/m3_exec.h#L1240)、[op_BranchTable()](../wasm3/source/m3_exec.h#L1275)、[op_Call()](../wasm3/source/m3_exec.h#L554)、[op_CallIndirect()](../wasm3/source/m3_exec.h#L580)。

这是 Wasm 最独特的部分——结构化控制流。与 CIL 的任意偏移跳转不同，Wasm 的跳转目标只能是嵌套块的边界。

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0x00` | `unreachable` | — | `[] → [t*]` | 无条件触发 trap，栈多态 |
| `0x01` | `nop` | — | `[] → []` | 空操作 |
| `0x02` | `block` | `bt:blocktype` | `[t1*] → [t2*]` | 开始一个块，`br` 跳转到块**末尾** |
| `0x03` | `loop` | `bt:blocktype` | `[t1*] → [t2*]` | 开始一个循环块，`br` 跳转到块**开头** |
| `0x04` | `if` | `bt:blocktype` | `[t1* i32] → [t2*]` | 弹出 i32，非零执行 then 分支 |
| `0x05` | `else` | — | `[t*] → [t*]` | if 的 else 分支开始 |
| `0x0B` | `end` | — | `[t*] → [t*]` | 块/函数体结束标记 |
| `0x0C` | `br` | `l:labelidx(u32)` | `[t1* t*] → [t2*]` | 无条件跳转到第 l 层外围块的边界 |
| `0x0D` | `br_if` | `l:labelidx(u32)` | `[t* i32] → [t*]` | 弹出 i32，非零则跳转 |
| `0x0E` | `br_table` | `l*:vec(labelidx), ln:labelidx` | `[t1* t* i32] → [t2*]` | 弹出 i32 作为索引，查表跳转，越界用 ln |
| `0x0F` | `return` | — | `[t1* t*] → [t2*]` | 从当前函数返回 |
| `0x10` | `call` | `x:funcidx(u32)` | `[t1*] → [t2*]` | 调用函数 x |
| `0x11` | `call_indirect` | `y:typeidx(u32), x:tableidx(u32)` | `[t1* i32] → [t2*]` | 从表 x 中取索引（栈顶 i32），按类型 y 间接调用 |

> **blocktype 编码：**
> - `0x40` = 空（无参数无返回值）
> - `0x7F`/`0x7E`/`0x7D`/`0x7C` = 单返回值类型（i32/i64/f32/f64）
> - 正整数（s33 LEB128）= 类型索引，引用 Type Section 中的函数签名

---

## 2. 引用指令（Reference Instructions）

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0xD0` | `ref.null` | `t:reftype` | `[] → [t]` | 压入类型 t 的空引用 |
| `0xD1` | `ref.is_null` | — | `[t] → [i32]` | 弹出引用，空则压 1，否则压 0 |
| `0xD2` | `ref.func` | `x:funcidx(u32)` | `[] → [funcref]` | 压入函数 x 的引用 |

---

## 3. 参数指令（Parametric Instructions）

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0x1A` | `drop` | — | `[t] → []` | 弹出并丢弃栈顶值 |
| `0x1B` | `select` | — | `[t t i32] → [t]` | 弹出 i32 条件值，非零选第一个，零选第二个 |
| `0x1C` | `select t*` | `t*:vec(valtype)` | `[t t i32] → [t]` | 带类型注解的 select（用于引用类型） |

---

## 4. 变量指令（Variable Instructions）

> **wasm3 实现参考**：编译阶段见 [m3_compile.c - Compile_GetLocal()](../wasm3/source/m3_compile.c#L1322)、[Compile_SetLocal()](../wasm3/source/m3_compile.c#L1327)；全局变量见 [m3_compile.c - Compile_GetSetGlobal()](../wasm3/source/m3_compile.c#L1383)。

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0x20` | `local.get` | `x:localidx(u32)` | `[] → [t]` | 读取局部变量 x 压栈 |
| `0x21` | `local.set` | `x:localidx(u32)` | `[t] → []` | 弹出栈顶写入局部变量 x |
| `0x22` | `local.tee` | `x:localidx(u32)` | `[t] → [t]` | 写入局部变量 x 但**不弹出**栈顶（peek + set） |
| `0x23` | `global.get` | `x:globalidx(u32)` | `[] → [t]` | 读取全局变量 x 压栈 |
| `0x24` | `global.set` | `x:globalidx(u32)` | `[t] → []` | 弹出栈顶写入全局变量 x（必须是 mutable） |

---

## 5. 表指令（Table Instructions）

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0x25` | `table.get` | `x:tableidx(u32)` | `[i32] → [t]` | 以栈顶 i32 为索引，从表 x 读取元素 |
| `0x26` | `table.set` | `x:tableidx(u32)` | `[i32 t] → []` | 以 i32 为索引，将 t 写入表 x |
| `0xFC 12` | `table.init` | `y:elemidx(u32), x:tableidx(u32)` | `[i32 i32 i32] → []` | 从 element segment y 初始化表 x |
| `0xFC 13` | `elem.drop` | `x:elemidx(u32)` | `[] → []` | 丢弃 element segment x |
| `0xFC 14` | `table.copy` | `x:tableidx(u32), y:tableidx(u32)` | `[i32 i32 i32] → []` | 表间复制 |
| `0xFC 15` | `table.grow` | `x:tableidx(u32)` | `[t i32] → [i32]` | 增长表 x，返回旧大小（失败返回 -1） |
| `0xFC 16` | `table.size` | `x:tableidx(u32)` | `[] → [i32]` | 返回表 x 当前大小 |
| `0xFC 17` | `table.fill` | `x:tableidx(u32)` | `[i32 t i32] → []` | 用值 t 填充表 x 的指定范围 |

---

## 6. 内存指令（Memory Instructions）

> **wasm3 实现参考**：[m3_exec.h - d_m3Load](../wasm3/source/m3_exec.h#L1321) — load 操作宏（生成所有 load 变体）；[m3_exec.h - d_m3Store](../wasm3/source/m3_exec.h#L1390) — store 操作宏（生成所有 store 变体）；[m3_exec.h - op_MemSize()](../wasm3/source/m3_exec.h#L696)、[op_MemGrow()](../wasm3/source/m3_exec.h#L725) — 内存管理指令；[m3_compile.c - Compile_Load_Store()](../wasm3/source/m3_compile.c#L2200) — load/store 编译。

### 6.1 加载指令

所有 load 指令的立即数格式：`align:u32, offset:u32`（对齐指数 + 偏移量）。
有效地址 = `i32_base + offset`，越界触发 trap。

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x28` | `i32.load` | `[i32] → [i32]` | 加载 4 字节为 i32 |
| `0x29` | `i64.load` | `[i32] → [i64]` | 加载 8 字节为 i64 |
| `0x2A` | `f32.load` | `[i32] → [f32]` | 加载 4 字节为 f32 |
| `0x2B` | `f64.load` | `[i32] → [f64]` | 加载 8 字节为 f64 |
| `0x2C` | `i32.load8_s` | `[i32] → [i32]` | 加载 1 字节，符号扩展为 i32 |
| `0x2D` | `i32.load8_u` | `[i32] → [i32]` | 加载 1 字节，零扩展为 i32 |
| `0x2E` | `i32.load16_s` | `[i32] → [i32]` | 加载 2 字节，符号扩展为 i32 |
| `0x2F` | `i32.load16_u` | `[i32] → [i32]` | 加载 2 字节，零扩展为 i32 |
| `0x30` | `i64.load8_s` | `[i32] → [i64]` | 加载 1 字节，符号扩展为 i64 |
| `0x31` | `i64.load8_u` | `[i32] → [i64]` | 加载 1 字节，零扩展为 i64 |
| `0x32` | `i64.load16_s` | `[i32] → [i64]` | 加载 2 字节，符号扩展为 i64 |
| `0x33` | `i64.load16_u` | `[i32] → [i64]` | 加载 2 字节，零扩展为 i64 |
| `0x34` | `i64.load32_s` | `[i32] → [i64]` | 加载 4 字节，符号扩展为 i64 |
| `0x35` | `i64.load32_u` | `[i32] → [i64]` | 加载 4 字节，零扩展为 i64 |

### 6.2 存储指令

所有 store 指令的立即数格式同 load：`align:u32, offset:u32`。

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x36` | `i32.store` | `[i32 i32] → []` | 存储 i32（4 字节） |
| `0x37` | `i64.store` | `[i32 i64] → []` | 存储 i64（8 字节） |
| `0x38` | `f32.store` | `[i32 f32] → []` | 存储 f32（4 字节） |
| `0x39` | `f64.store` | `[i32 f64] → []` | 存储 f64（8 字节） |
| `0x3A` | `i32.store8` | `[i32 i32] → []` | 截断存储低 1 字节 |
| `0x3B` | `i32.store16` | `[i32 i32] → []` | 截断存储低 2 字节 |
| `0x3C` | `i64.store8` | `[i32 i64] → []` | 截断存储低 1 字节 |
| `0x3D` | `i64.store16` | `[i32 i64] → []` | 截断存储低 2 字节 |
| `0x3E` | `i64.store32` | `[i32 i64] → []` | 截断存储低 4 字节 |

### 6.3 内存管理指令

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0x3F` | `memory.size` | `0x00`（memidx） | `[] → [i32]` | 返回当前内存大小（单位：页，1 页 = 64KB） |
| `0x40` | `memory.grow` | `0x00`（memidx） | `[i32] → [i32]` | 增长 n 页，返回旧页数（失败返回 -1） |
| `0xFC 8` | `memory.init` | `x:dataidx(u32), 0x00` | `[i32 i32 i32] → []` | 从 data segment x 复制到内存 |
| `0xFC 9` | `data.drop` | `x:dataidx(u32)` | `[] → []` | 丢弃 data segment x |
| `0xFC 10` | `memory.copy` | `0x00, 0x00` | `[i32 i32 i32] → []` | 内存区域复制（dst, src, len） |
| `0xFC 11` | `memory.fill` | `0x00` | `[i32 i32 i32] → []` | 内存区域填充（dst, val, len） |

---

## 7. 数值指令（Numeric Instructions）

### 7.1 常量指令

| 操作码 | 助记符 | 立即数 | 栈签名 | 语义 |
|--------|--------|--------|--------|------|
| `0x41` | `i32.const` | `n:i32(s32 LEB128)` | `[] → [i32]` | 压入 i32 常量 |
| `0x42` | `i64.const` | `n:i64(s64 LEB128)` | `[] → [i64]` | 压入 i64 常量 |
| `0x43` | `f32.const` | `z:f32(4 bytes IEEE754)` | `[] → [f32]` | 压入 f32 常量 |
| `0x44` | `f64.const` | `z:f64(8 bytes IEEE754)` | `[] → [f64]` | 压入 f64 常量 |

### 7.2 i32 比较指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x45` | `i32.eqz` | `[i32] → [i32]` | 等于零测试 |
| `0x46` | `i32.eq` | `[i32 i32] → [i32]` | 相等 |
| `0x47` | `i32.ne` | `[i32 i32] → [i32]` | 不等 |
| `0x48` | `i32.lt_s` | `[i32 i32] → [i32]` | 有符号小于 |
| `0x49` | `i32.lt_u` | `[i32 i32] → [i32]` | 无符号小于 |
| `0x4A` | `i32.gt_s` | `[i32 i32] → [i32]` | 有符号大于 |
| `0x4B` | `i32.gt_u` | `[i32 i32] → [i32]` | 无符号大于 |
| `0x4C` | `i32.le_s` | `[i32 i32] → [i32]` | 有符号小于等于 |
| `0x4D` | `i32.le_u` | `[i32 i32] → [i32]` | 无符号小于等于 |
| `0x4E` | `i32.ge_s` | `[i32 i32] → [i32]` | 有符号大于等于 |
| `0x4F` | `i32.ge_u` | `[i32 i32] → [i32]` | 无符号大于等于 |

### 7.3 i64 比较指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x50` | `i64.eqz` | `[i64] → [i32]` | 等于零测试 |
| `0x51` | `i64.eq` | `[i64 i64] → [i32]` | 相等 |
| `0x52` | `i64.ne` | `[i64 i64] → [i32]` | 不等 |
| `0x53` | `i64.lt_s` | `[i64 i64] → [i32]` | 有符号小于 |
| `0x54` | `i64.lt_u` | `[i64 i64] → [i32]` | 无符号小于 |
| `0x55` | `i64.gt_s` | `[i64 i64] → [i32]` | 有符号大于 |
| `0x56` | `i64.gt_u` | `[i64 i64] → [i32]` | 无符号大于 |
| `0x57` | `i64.le_s` | `[i64 i64] → [i32]` | 有符号小于等于 |
| `0x58` | `i64.le_u` | `[i64 i64] → [i32]` | 无符号小于等于 |
| `0x59` | `i64.ge_s` | `[i64 i64] → [i32]` | 有符号大于等于 |
| `0x5A` | `i64.ge_u` | `[i64 i64] → [i32]` | 无符号大于等于 |

### 7.4 f32 比较指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x5B` | `f32.eq` | `[f32 f32] → [i32]` | 相等（NaN ≠ NaN） |
| `0x5C` | `f32.ne` | `[f32 f32] → [i32]` | 不等 |
| `0x5D` | `f32.lt` | `[f32 f32] → [i32]` | 小于 |
| `0x5E` | `f32.gt` | `[f32 f32] → [i32]` | 大于 |
| `0x5F` | `f32.le` | `[f32 f32] → [i32]` | 小于等于 |
| `0x60` | `f32.ge` | `[f32 f32] → [i32]` | 大于等于 |

### 7.5 f64 比较指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x61` | `f64.eq` | `[f64 f64] → [i32]` | 相等（NaN ≠ NaN） |
| `0x62` | `f64.ne` | `[f64 f64] → [i32]` | 不等 |
| `0x63` | `f64.lt` | `[f64 f64] → [i32]` | 小于 |
| `0x64` | `f64.gt` | `[f64 f64] → [i32]` | 大于 |
| `0x65` | `f64.le` | `[f64 f64] → [i32]` | 小于等于 |
| `0x66` | `f64.ge` | `[f64 f64] → [i32]` | 大于等于 |

### 7.6 i32 算术与位运算指令

> **wasm3 实现参考**：所有 i32 算术运算的 M3 operation 实现见 [m3_exec.h](../wasm3/source/m3_exec.h)，通过 `d_m3CommutativeOp_i`/`d_m3Op_i` 等宏批量生成。例如除法和取余包含除零检查：[m3_exec.h - d_m3OpMacro_i(i32, Divide)](../wasm3/source/m3_exec.h#L241)。

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x67` | `i32.clz` | `[i32] → [i32]` | 前导零计数 |
| `0x68` | `i32.ctz` | `[i32] → [i32]` | 尾随零计数 |
| `0x69` | `i32.popcnt` | `[i32] → [i32]` | 置位位计数 |
| `0x6A` | `i32.add` | `[i32 i32] → [i32]` | 加法（模 2³²） |
| `0x6B` | `i32.sub` | `[i32 i32] → [i32]` | 减法（模 2³²） |
| `0x6C` | `i32.mul` | `[i32 i32] → [i32]` | 乘法（模 2³²） |
| `0x6D` | `i32.div_s` | `[i32 i32] → [i32]` | 有符号除法（除零/溢出 trap） |
| `0x6E` | `i32.div_u` | `[i32 i32] → [i32]` | 无符号除法（除零 trap） |
| `0x6F` | `i32.rem_s` | `[i32 i32] → [i32]` | 有符号取余（除零 trap） |
| `0x70` | `i32.rem_u` | `[i32 i32] → [i32]` | 无符号取余（除零 trap） |
| `0x71` | `i32.and` | `[i32 i32] → [i32]` | 按位与 |
| `0x72` | `i32.or` | `[i32 i32] → [i32]` | 按位或 |
| `0x73` | `i32.xor` | `[i32 i32] → [i32]` | 按位异或 |
| `0x74` | `i32.shl` | `[i32 i32] → [i32]` | 左移（移位量 mod 32） |
| `0x75` | `i32.shr_s` | `[i32 i32] → [i32]` | 算术右移（移位量 mod 32） |
| `0x76` | `i32.shr_u` | `[i32 i32] → [i32]` | 逻辑右移（移位量 mod 32） |
| `0x77` | `i32.rotl` | `[i32 i32] → [i32]` | 循环左移 |
| `0x78` | `i32.rotr` | `[i32 i32] → [i32]` | 循环右移 |

### 7.7 i64 算术与位运算指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x79` | `i64.clz` | `[i64] → [i64]` | 前导零计数 |
| `0x7A` | `i64.ctz` | `[i64] → [i64]` | 尾随零计数 |
| `0x7B` | `i64.popcnt` | `[i64] → [i64]` | 置位位计数 |
| `0x7C` | `i64.add` | `[i64 i64] → [i64]` | 加法（模 2⁶⁴） |
| `0x7D` | `i64.sub` | `[i64 i64] → [i64]` | 减法（模 2⁶⁴） |
| `0x7E` | `i64.mul` | `[i64 i64] → [i64]` | 乘法（模 2⁶⁴） |
| `0x7F` | `i64.div_s` | `[i64 i64] → [i64]` | 有符号除法（除零/溢出 trap） |
| `0x80` | `i64.div_u` | `[i64 i64] → [i64]` | 无符号除法（除零 trap） |
| `0x81` | `i64.rem_s` | `[i64 i64] → [i64]` | 有符号取余（除零 trap） |
| `0x82` | `i64.rem_u` | `[i64 i64] → [i64]` | 无符号取余（除零 trap） |
| `0x83` | `i64.and` | `[i64 i64] → [i64]` | 按位与 |
| `0x84` | `i64.or` | `[i64 i64] → [i64]` | 按位或 |
| `0x85` | `i64.xor` | `[i64 i64] → [i64]` | 按位异或 |
| `0x86` | `i64.shl` | `[i64 i64] → [i64]` | 左移（移位量 mod 64） |
| `0x87` | `i64.shr_s` | `[i64 i64] → [i64]` | 算术右移（移位量 mod 64） |
| `0x88` | `i64.shr_u` | `[i64 i64] → [i64]` | 逻辑右移（移位量 mod 64） |
| `0x89` | `i64.rotl` | `[i64 i64] → [i64]` | 循环左移 |
| `0x8A` | `i64.rotr` | `[i64 i64] → [i64]` | 循环右移 |

### 7.8 f32 算术指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x8B` | `f32.abs` | `[f32] → [f32]` | 绝对值 |
| `0x8C` | `f32.neg` | `[f32] → [f32]` | 取反 |
| `0x8D` | `f32.ceil` | `[f32] → [f32]` | 向上取整 |
| `0x8E` | `f32.floor` | `[f32] → [f32]` | 向下取整 |
| `0x8F` | `f32.trunc` | `[f32] → [f32]` | 向零取整 |
| `0x90` | `f32.nearest` | `[f32] → [f32]` | 四舍五入到最近偶数 |
| `0x91` | `f32.sqrt` | `[f32] → [f32]` | 平方根 |
| `0x92` | `f32.add` | `[f32 f32] → [f32]` | 加法 |
| `0x93` | `f32.sub` | `[f32 f32] → [f32]` | 减法 |
| `0x94` | `f32.mul` | `[f32 f32] → [f32]` | 乘法 |
| `0x95` | `f32.div` | `[f32 f32] → [f32]` | 除法 |
| `0x96` | `f32.min` | `[f32 f32] → [f32]` | 最小值（NaN 传播） |
| `0x97` | `f32.max` | `[f32 f32] → [f32]` | 最大值（NaN 传播） |
| `0x98` | `f32.copysign` | `[f32 f32] → [f32]` | 复制符号位 |

### 7.9 f64 算术指令

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0x99` | `f64.abs` | `[f64] → [f64]` | 绝对值 |
| `0x9A` | `f64.neg` | `[f64] → [f64]` | 取反 |
| `0x9B` | `f64.ceil` | `[f64] → [f64]` | 向上取整 |
| `0x9C` | `f64.floor` | `[f64] → [f64]` | 向下取整 |
| `0x9D` | `f64.trunc` | `[f64] → [f64]` | 向零取整 |
| `0x9E` | `f64.nearest` | `[f64] → [f64]` | 四舍五入到最近偶数 |
| `0x9F` | `f64.sqrt` | `[f64] → [f64]` | 平方根 |
| `0xA0` | `f64.add` | `[f64 f64] → [f64]` | 加法 |
| `0xA1` | `f64.sub` | `[f64 f64] → [f64]` | 减法 |
| `0xA2` | `f64.mul` | `[f64 f64] → [f64]` | 乘法 |
| `0xA3` | `f64.div` | `[f64 f64] → [f64]` | 除法 |
| `0xA4` | `f64.min` | `[f64 f64] → [f64]` | 最小值（NaN 传播） |
| `0xA5` | `f64.max` | `[f64 f64] → [f64]` | 最大值（NaN 传播） |
| `0xA6` | `f64.copysign` | `[f64 f64] → [f64]` | 复制符号位 |

### 7.10 类型转换指令

> **wasm3 实现参考**：截断转换（可能 trap）见 [m3_exec.h - d_m3TruncMacro](../wasm3/source/m3_exec.h#L353) — 包含溢出和 NaN 检查的截断宏；位模式重解释通过 C 类型双关实现。

这是实现中容易出错的部分，特别注意 trap 条件和 saturating 变体的区别。

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0xA7` | `i32.wrap_i64` | `[i64] → [i32]` | 截断低 32 位 |
| `0xA8` | `i32.trunc_f32_s` | `[f32] → [i32]` | f32→i32 有符号截断（溢出/NaN trap） |
| `0xA9` | `i32.trunc_f32_u` | `[f32] → [i32]` | f32→i32 无符号截断（溢出/NaN trap） |
| `0xAA` | `i32.trunc_f64_s` | `[f64] → [i32]` | f64→i32 有符号截断（溢出/NaN trap） |
| `0xAB` | `i32.trunc_f64_u` | `[f64] → [i32]` | f64→i32 无符号截断（溢出/NaN trap） |
| `0xAC` | `i64.extend_i32_s` | `[i32] → [i64]` | i32→i64 符号扩展 |
| `0xAD` | `i64.extend_i32_u` | `[i32] → [i64]` | i32→i64 零扩展 |
| `0xAE` | `i64.trunc_f32_s` | `[f32] → [i64]` | f32→i64 有符号截断（溢出/NaN trap） |
| `0xAF` | `i64.trunc_f32_u` | `[f32] → [i64]` | f32→i64 无符号截断（溢出/NaN trap） |
| `0xB0` | `i64.trunc_f64_s` | `[f64] → [i64]` | f64→i64 有符号截断（溢出/NaN trap） |
| `0xB1` | `i64.trunc_f64_u` | `[f64] → [i64]` | f64→i64 无符号截断（溢出/NaN trap） |
| `0xB2` | `f32.convert_i32_s` | `[i32] → [f32]` | i32 有符号→f32 |
| `0xB3` | `f32.convert_i32_u` | `[i32] → [f32]` | i32 无符号→f32 |
| `0xB4` | `f32.convert_i64_s` | `[i64] → [f32]` | i64 有符号→f32 |
| `0xB5` | `f32.convert_i64_u` | `[i64] → [f32]` | i64 无符号→f32 |
| `0xB6` | `f32.demote_f64` | `[f64] → [f32]` | f64→f32 精度降低 |
| `0xB7` | `f64.convert_i32_s` | `[i32] → [f64]` | i32 有符号→f64 |
| `0xB8` | `f64.convert_i32_u` | `[i32] → [f64]` | i32 无符号→f64 |
| `0xB9` | `f64.convert_i64_s` | `[i64] → [f64]` | i64 有符号→f64 |
| `0xBA` | `f64.convert_i64_u` | `[i64] → [f64]` | i64 无符号→f64 |
| `0xBB` | `f64.promote_f32` | `[f32] → [f64]` | f32→f64 精度提升 |
| `0xBC` | `i32.reinterpret_f32` | `[f32] → [i32]` | 位模式重解释（不改变位） |
| `0xBD` | `i64.reinterpret_f64` | `[f64] → [i64]` | 位模式重解释 |
| `0xBE` | `f32.reinterpret_i32` | `[i32] → [f32]` | 位模式重解释 |
| `0xBF` | `f64.reinterpret_i64` | `[i64] → [f64]` | 位模式重解释 |

### 7.11 符号扩展指令（Sign Extension Operators 提案）

| 操作码 | 助记符 | 栈签名 | 语义 |
|--------|--------|--------|------|
| `0xC0` | `i32.extend8_s` | `[i32] → [i32]` | 低 8 位符号扩展到 i32 |
| `0xC1` | `i32.extend16_s` | `[i32] → [i32]` | 低 16 位符号扩展到 i32 |
| `0xC2` | `i64.extend8_s` | `[i64] → [i64]` | 低 8 位符号扩展到 i64 |
| `0xC3` | `i64.extend16_s` | `[i64] → [i64]` | 低 16 位符号扩展到 i64 |
| `0xC4` | `i64.extend32_s` | `[i64] → [i64]` | 低 32 位符号扩展到 i64 |

---

## 8. 0xFC 前缀扩展指令

`0xFC` 前缀后跟一个 u32 LEB128 编码的子操作码。

### 8.1 饱和截断指令（Nontrapping Float-to-Int Conversions）

> **wasm3 实现参考**：饱和截断变体见 [m3_exec.h - d_m3TruncMacro(TruncSat)](../wasm3/source/m3_exec.h#L390) — 溢出时返回最近可表示值而非 trap。

与普通 trunc 不同，这些指令在溢出时**不 trap**，而是返回最近的可表示值。

| 子操作码 | 助记符 | 栈签名 | 语义 |
|----------|--------|--------|------|
| `0xFC 0` | `i32.trunc_sat_f32_s` | `[f32] → [i32]` | 饱和截断 f32→i32 有符号 |
| `0xFC 1` | `i32.trunc_sat_f32_u` | `[f32] → [i32]` | 饱和截断 f32→i32 无符号 |
| `0xFC 2` | `i32.trunc_sat_f64_s` | `[f64] → [i32]` | 饱和截断 f64→i32 有符号 |
| `0xFC 3` | `i32.trunc_sat_f64_u` | `[f64] → [i32]` | 饱和截断 f64→i32 无符号 |
| `0xFC 4` | `i64.trunc_sat_f32_s` | `[f32] → [i64]` | 饱和截断 f32→i64 有符号 |
| `0xFC 5` | `i64.trunc_sat_f32_u` | `[f32] → [i64]` | 饱和截断 f32→i64 无符号 |
| `0xFC 6` | `i64.trunc_sat_f64_s` | `[f64] → [i64]` | 饱和截断 f64→i64 有符号 |
| `0xFC 7` | `i64.trunc_sat_f64_u` | `[f64] → [i64]` | 饱和截断 f64→i64 无符号 |

### 8.2 批量内存与表操作

已在第 5 节（表指令）和第 6.3 节（内存管理指令）中列出：
- `0xFC 8` ~ `0xFC 11`：memory.init / data.drop / memory.copy / memory.fill
- `0xFC 12` ~ `0xFC 17`：table.init / elem.drop / table.copy / table.grow / table.size / table.fill

---

## 9. 实现者备注

### 9.1 解码策略

> **wasm3 实现参考**：[m3_compile.c - CompileBlockStatements()](../wasm3/source/m3_compile.c#L2568) — 主解码循环，逐条读取操作码并分发到对应的编译函数；[m3_compile.c - GetOpInfo()](../wasm3/source/m3_compile.c#L2550) — 操作码到 M3OpInfo 的查找（含 0xFC 前缀二级分发）。

```
// Pseudocode: instruction decoder main loop
uint8_t opcode = read_byte(stream);
switch (opcode) {
    case 0x00: /* unreachable */ break;
    case 0x01: /* nop */ break;
    case 0x02: /* block */
        blocktype bt = decode_blocktype(stream);
        // ...
        break;
    // ... single-byte opcodes 0x03 ~ 0xC4 ...
    case 0xFC: {
        uint32_t sub = read_leb128_u32(stream);
        switch (sub) {
            case 0: /* i32.trunc_sat_f32_s */ break;
            // ... sub-opcodes 0 ~ 17 ...
        }
        break;
    }
    case 0xFD: {
        // SIMD prefix (v128 instructions, not covered here)
        uint32_t sub = read_leb128_u32(stream);
        // ...
        break;
    }
    default:
        // invalid opcode → validation error
        break;
}
```

### 9.2 与 CLR 操作码的关键差异

| 特性 | Wasm | CIL |
|------|------|-----|
| 操作码长度 | 1 字节 + 可选前缀（0xFC/0xFD） | 1 字节 + 可选前缀（0xFE） |
| 跳转目标 | 结构化标签索引（相对深度） | 任意字节偏移 |
| 类型区分 | 每种类型有独立操作码（i32.add ≠ i64.add） | 栈多态（add 同时处理 int32/int64/float） |
| 溢出行为 | 整数运算静默回绕，浮点→整数截断可 trap | 有 checked 变体（add.ovf） |
| 内存访问 | 显式 load/store + 偏移立即数 | 隐式通过引用/指针（ldind/stind） |
| 间接调用 | call_indirect + 运行时类型检查 | callvirt + vtable 分发 |

### 9.3 操作码编码空间总览

```
0x00 - 0x11  : Control flow (18 opcodes)
0x1A - 0x1C  : Parametric (3 opcodes)
0x20 - 0x26  : Variable + Table (7 opcodes)
0x28 - 0x40  : Memory (25 opcodes)
0x41 - 0x44  : Constants (4 opcodes)
0x45 - 0xC4  : Numeric (128 opcodes)
0xD0 - 0xD2  : Reference (3 opcodes)
0xFC + u32   : Extended (saturating trunc + bulk memory + table ops)
0xFD + u32   : SIMD v128 (not covered, 200+ opcodes)
0xFE + u32   : Threads/Atomics (not covered)
```

> **提示**：实现解释器时，可以用一个 256 元素的函数指针数组（dispatch table）处理单字节操作码，
> 对 0xFC/0xFD/0xFE 前缀再用二级 dispatch。这正是 wasm3 的基本思路——参见 [m3_compile.c - c_operations](../wasm3/source/m3_compile.c#L2260)（主表）和 [c_operationsFC](../wasm3/source/m3_compile.c#L2527)（0xFC 子表）。
