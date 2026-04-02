# 附录 B：LEB128 变长整数编码详解

> LEB128（Little Endian Base 128）是 WebAssembly 二进制格式中最基础的编码方式。几乎所有的整数值——Section ID、类型索引、函数索引、指令立即数、向量长度等——都使用 LEB128 编码。在实现 Wasm 解析器之前，你必须先搞定它。
>
> **wasm3 源码参考**：[m3_core.c - ReadLebUnsigned()](../wasm3/source/m3_core.c#L356) — 无符号 LEB128 解码；[m3_core.c - ReadLebSigned()](../wasm3/source/m3_core.c#L390) — 有符号 LEB128 解码；[m3_core.c - ReadLEB_u32()](../wasm3/source/m3_core.c#L435) / [ReadLEB_i32()](../wasm3/source/m3_core.c#L465) / [ReadLEB_i64()](../wasm3/source/m3_core.c#L475) — 便捷包装函数；[m3_core.h#L291](../wasm3/source/m3_core.h#L291) — 函数声明。

## 1. 为什么用 LEB128？

如果你实现过 CLR，你会记得 CLR 元数据中的 Blob 堆使用压缩整数编码（Compressed Integer），规则是：

- 1 字节：`0xxxxxxx`（0~127）
- 2 字节：`10xxxxxx xxxxxxxx`（128~16383）
- 4 字节：`110xxxxx ...`（更大值）

LEB128 的设计目标类似——**用尽可能少的字节表示小整数**——但编码方案不同。LEB128 来自 DWARF 调试信息格式，被 WebAssembly 直接采用。

**核心思想**：每个字节的低 7 位存储数据，最高位（bit 7）作为**延续标志**（continuation bit）：
- `1` = 后面还有更多字节
- `0` = 这是最后一个字节

字节按**小端序**排列（低位在前），因此得名 "Little Endian Base 128"。

## 2. 无符号 LEB128（Unsigned LEB128）

### 2.1 编码原理

将一个无符号整数拆分为 7-bit 一组，从低位到高位依次输出，每组占一个字节。除最后一个字节外，所有字节的最高位设为 1。

**示例：编码 `624485`**

```
624485 的二进制：0010 0110 0001 0000 1110 0101

按 7-bit 分组（从低位开始）：
  group 0: 1100101  (低 7 位)
  group 1: 0001000
  group 2: 0100110  (高位组)

添加延续位：
  byte 0: 1_1100101 = 0xE5  (还有后续 → bit7=1)
  byte 1: 1_0001000 = 0x88  (还有后续 → bit7=1)
  byte 2: 0_0100110 = 0x26  (最后一个 → bit7=0)

输出字节流：E5 88 26
```

### 2.2 解码原理

从字节流中逐字节读取，每个字节取低 7 位，按位置左移后累加。遇到最高位为 0 的字节时停止。

```
输入：E5 88 26

byte 0: 0xE5 → data=0x65(1100101), shift=0,  result = 0x65 << 0  = 0x65
byte 1: 0x88 → data=0x08(0001000), shift=7,  result += 0x08 << 7  = 0x65 + 0x400 = 0x465
byte 2: 0x26 → data=0x26(0100110), shift=14, result += 0x26 << 14 = 0x465 + 0x98000 = 0x98465

0x98465 = 624485 ✓
```

### 2.3 Wasm 中的约束

WebAssembly 规范对 LEB128 有严格的**最大字节数限制**：

| 目标类型 | 最大字节数 | 最大有效位数 |
|---------|-----------|-------------|
| u32     | 5 字节    | 32 位       |
| u64     | 10 字节   | 64 位       |

**关键细节**：对于 u32，第 5 个字节的高 4 位必须为 0（因为 4×7=28，还剩 4 位，第 5 字节只能使用低 4 位）。如果第 5 字节的值 > 0x0F，则为非法编码，解析器必须报错。

## 3. 有符号 LEB128（Signed LEB128）

### 3.1 编码原理

有符号 LEB128 使用**二进制补码**表示负数。编码过程与无符号类似，但终止条件不同：

- 对于**正数**：当剩余值为 0，且最后一个字节的 bit 6 为 0 时停止
- 对于**负数**：当剩余值为 -1（全 1），且最后一个字节的 bit 6 为 1 时停止

这确保了解码时符号扩展的正确性。

**示例：编码 `-123456`**

```
-123456 的 32 位补码：1111 1111 1111 1110 0001 1101 1100 0000

按 7-bit 分组（从低位开始）：
  group 0: 1000000
  group 1: 1011100
  group 2: 1000111  (注意 bit6=1，表示负数)

添加延续位：
  byte 0: 1_1000000 = 0xC0
  byte 1: 1_1011100 = 0xBC
  byte 2: 0_1000111 = 0x47  (bit6=1，符号位正确，停止)

输出字节流：C0 BC 47
```

### 3.2 解码原理

与无符号解码类似，但在最后一个字节处理完后，需要检查是否需要**符号扩展**：如果最后一个字节的 bit 6 为 1，则将结果的高位全部填充为 1。

```
输入：C0 BC 47

byte 0: 0xC0 → data=0x40, shift=0,  result = 0x40
byte 1: 0xBC → data=0x3C, shift=7,  result += 0x3C << 7 = 0x40 + 0x1E00 = 0x1E40
byte 2: 0x47 → data=0x47, shift=14, result += 0x47 << 14 = 0x1E40 + 0x11C000 = 0x11DE40

最后一个字节 0x47 的 bit6 = 1 → 需要符号扩展
shift = 14 + 7 = 21, 对于 32 位整数：result |= (~0 << 21)
result = 0x11DE40 | 0xFFE00000 = 0xFFFE1DC0

解释为有符号 32 位：0xFFFE1DC0 = -123456 ✓
```

### 3.3 Wasm 中的约束

| 目标类型 | 最大字节数 | 最大有效位数 |
|---------|-----------|-------------|
| s32     | 5 字节    | 32 位       |
| s33     | 5 字节    | 33 位（用于 block type）|
| s64     | 10 字节   | 64 位       |

**特殊类型 s33**：Wasm 的 block type 使用 s33 编码。这是因为 block type 可以是：
- 负数 `0x40`（empty type，编码为单字节 `0x40`）
- 负数 `0x7F`~`0x7C`（valtype 编码）
- 非负数（type index，u32 范围）

使用 s33 可以在一个编码空间中同时容纳负数标记和正数索引。

## 4. C 语言实现

### 4.1 无符号 LEB128 解码

```c
typedef enum {
    WASM_OK = 0,
    WASM_ERR_LEB_OVERFLOW,      // exceeded max bytes
    WASM_ERR_LEB_UNUSED_BITS,   // non-zero unused bits in final byte
    WASM_ERR_UNEXPECTED_END,    // ran out of bytes
} wasm_error_t;

// Decode an unsigned LEB128 value.
// - buf:     pointer to the byte stream
// - buf_end: one-past-the-end pointer for bounds checking
// - max_bits: maximum number of value bits (32 or 64)
// - out_val: decoded value output
// - out_len: number of bytes consumed
wasm_error_t decode_uleb128(
    const uint8_t *buf,
    const uint8_t *buf_end,
    uint32_t       max_bits,
    uint64_t      *out_val,
    uint32_t      *out_len)
{
    uint64_t result = 0;
    uint32_t shift  = 0;
    uint32_t max_bytes = (max_bits + 6) / 7;  // ceil(max_bits / 7)

    for (uint32_t i = 0; ; i++) {
        // bounds check
        if (buf + i >= buf_end) {
            return WASM_ERR_UNEXPECTED_END;
        }

        // max bytes check
        if (i >= max_bytes) {
            return WASM_ERR_LEB_OVERFLOW;
        }

        uint8_t byte = buf[i];
        uint64_t slice = byte & 0x7F;

        // For the last allowed byte, check that unused high bits are zero.
        // e.g., for u32 (max_bits=32), byte 4 (i=4) can only use low 4 bits.
        if (i == max_bytes - 1) {
            uint32_t remaining_bits = max_bits - shift;
            if (remaining_bits < 7) {
                uint8_t mask = (uint8_t)(0x7F << remaining_bits) & 0x7F;
                if (slice & mask) {
                    return WASM_ERR_LEB_UNUSED_BITS;
                }
            }
        }

        result |= (slice << shift);
        shift += 7;

        // continuation bit check
        if ((byte & 0x80) == 0) {
            *out_val = result;
            *out_len = i + 1;
            return WASM_OK;
        }
    }
}
```

### 4.2 有符号 LEB128 解码

```c
// Decode a signed LEB128 value.
// - max_bits: maximum number of value bits (32, 33, or 64)
wasm_error_t decode_sleb128(
    const uint8_t *buf,
    const uint8_t *buf_end,
    uint32_t       max_bits,
    int64_t       *out_val,
    uint32_t      *out_len)
{
    int64_t  result = 0;
    uint32_t shift  = 0;
    uint32_t max_bytes = (max_bits + 6) / 7;
    uint8_t  byte = 0;

    for (uint32_t i = 0; ; i++) {
        if (buf + i >= buf_end) {
            return WASM_ERR_UNEXPECTED_END;
        }
        if (i >= max_bytes) {
            return WASM_ERR_LEB_OVERFLOW;
        }

        byte = buf[i];
        uint64_t slice = byte & 0x7F;

        // Validate unused bits in the final byte.
        // For signed encoding, unused bits must all be copies of the sign bit.
        if (i == max_bytes - 1) {
            uint32_t remaining_bits = max_bits - shift;
            if (remaining_bits < 7) {
                // The sign bit is bit (remaining_bits - 1) of the slice.
                // All bits above it must match the sign bit.
                uint8_t sign_bit = (slice >> (remaining_bits - 1)) & 1;
                uint8_t mask = (0x7F << remaining_bits) & 0x7F;
                uint8_t expected = sign_bit ? mask : 0;
                if ((slice & mask) != expected) {
                    return WASM_ERR_LEB_UNUSED_BITS;
                }
            }
        }

        result |= ((int64_t)slice << shift);
        shift += 7;

        if ((byte & 0x80) == 0) {
            // Sign extend if the sign bit (bit 6 of last byte) is set
            // and we haven't filled all 64 bits yet.
            if (shift < 64 && (byte & 0x40)) {
                result |= -(((int64_t)1) << shift);
            }
            *out_val = result;
            *out_len = i + 1;
            return WASM_OK;
        }
    }
}
```

### 4.3 便捷包装函数

```c
// Convenience wrappers for common Wasm types

static inline wasm_error_t read_u32_leb128(
    const uint8_t *buf, const uint8_t *buf_end,
    uint32_t *out_val, uint32_t *out_len)
{
    uint64_t val;
    wasm_error_t err = decode_uleb128(buf, buf_end, 32, &val, out_len);
    if (err == WASM_OK) *out_val = (uint32_t)val;
    return err;
}

static inline wasm_error_t read_s32_leb128(
    const uint8_t *buf, const uint8_t *buf_end,
    int32_t *out_val, uint32_t *out_len)
{
    int64_t val;
    wasm_error_t err = decode_sleb128(buf, buf_end, 32, &val, out_len);
    if (err == WASM_OK) *out_val = (int32_t)val;
    return err;
}

static inline wasm_error_t read_s33_leb128(
    const uint8_t *buf, const uint8_t *buf_end,
    int64_t *out_val, uint32_t *out_len)
{
    return decode_sleb128(buf, buf_end, 33, out_val, out_len);
}

static inline wasm_error_t read_s64_leb128(
    const uint8_t *buf, const uint8_t *buf_end,
    int64_t *out_val, uint32_t *out_len)
{
    return decode_sleb128(buf, buf_end, 64, out_val, out_len);
}
```

### 4.4 编码实现（用于测试/调试）

```c
// Encode an unsigned 32-bit value as LEB128.
// Returns the number of bytes written.
uint32_t encode_uleb128(uint64_t value, uint8_t *buf) {
    uint32_t len = 0;
    do {
        uint8_t byte = value & 0x7F;
        value >>= 7;
        if (value != 0) {
            byte |= 0x80;  // set continuation bit
        }
        buf[len++] = byte;
    } while (value != 0);
    return len;
}

// Encode a signed 64-bit value as LEB128.
// Returns the number of bytes written.
uint32_t encode_sleb128(int64_t value, uint8_t *buf) {
    uint32_t len = 0;
    int more = 1;
    while (more) {
        uint8_t byte = value & 0x7F;
        value >>= 7;  // arithmetic shift (sign-extending)

        // If the remaining value is 0 and sign bit of byte is clear,
        // or the remaining value is -1 and sign bit of byte is set,
        // then we're done.
        if ((value == 0 && (byte & 0x40) == 0) ||
            (value == -1 && (byte & 0x40) != 0)) {
            more = 0;
        } else {
            byte |= 0x80;
        }
        buf[len++] = byte;
    }
    return len;
}
```

## 5. 边界情况与陷阱

### 5.1 最大值编码

```
u32 最大值 (4294967295 = 0xFFFFFFFF):
  FF FF FF FF 0F  (5 bytes, 最后一个字节 = 0x0F, 高 3 位为 0)

s32 最大值 (2147483647 = 0x7FFFFFFF):
  FF FF FF FF 07  (5 bytes, 最后一个字节 = 0x07)

s32 最小值 (-2147483648 = 0x80000000):
  80 80 80 80 78  (5 bytes, 最后一个字节 = 0x78)

u32 零:
  00  (1 byte)

s32 负一:
  7F  (1 byte, bit6=1 → sign extend to all 1s)
```

### 5.2 常见实现错误

| 错误 | 说明 | 后果 |
|------|------|------|
| **未检查最大字节数** | 恶意输入可以用无限多的 `0x80` 字节 | 无限循环或缓冲区溢出 |
| **未检查最后字节的高位** | u32 的第 5 字节为 `0xFF` 应该报错 | 静默截断，值错误 |
| **有符号解码忘记符号扩展** | 解码 `0x7F` 得到 127 而非 -1 | 所有负数解码错误 |
| **shift 溢出** | 在 32 位平台上 `1 << 32` 是未定义行为 | 平台相关的诡异 bug |
| **算术右移假设** | C 标准不保证 `>>` 对负数是算术右移 | 编码负数时可能出错 |

### 5.3 与 CLR 压缩整数的对比

| 特性 | Wasm LEB128 | CLR Compressed Integer |
|------|-------------|----------------------|
| 字节序 | 小端（低位在前） | 大端（高位在前） |
| 每字节有效位 | 7 位 | 6/14/29 位（分段） |
| 编码方案 | 统一的 7-bit 分组 | 前缀位决定长度（1/2/4 字节） |
| 有符号支持 | 原生支持（signed LEB128） | 需要额外的 SignedCompressedInt |
| 最大长度 | 可变（取决于目标位宽） | 固定（最多 4 字节） |
| 解码复杂度 | 需要循环 | 一次前缀判断即可确定长度 |

**实现建议**：如果你之前实现过 CLR 的压缩整数解码，LEB128 的核心区别在于它是**循环驱动**的（不知道要读几个字节，直到遇到终止字节），而 CLR 是**前缀驱动**的（第一个字节就能确定总长度）。这意味着 LEB128 解码器天然需要一个循环，而 CLR 的可以用 if-else 链。

## 6. 在 Wasm 解析器中的使用模式

> **wasm3 实现参考**：wasm3 的解析器中大量使用 `ReadLEB_u32`/`ReadLEB_i7` 等函数，参见 [m3_parse.c](../wasm3/source/m3_parse.c) 中的调用方式。wasm3 的游标模式使用 `bytes_t *` （即 `const uint8_t **`）作为可移动的读取指针，每次解码后自动前进。

在实际的 Wasm 解析器中，你会频繁调用 LEB128 解码。建议封装一个带游标（cursor）的读取器：

```c
typedef struct {
    const uint8_t *data;    // start of buffer
    const uint8_t *end;     // one past end
    const uint8_t *pos;     // current read position
} wasm_reader_t;

static wasm_error_t reader_read_u32(wasm_reader_t *r, uint32_t *out) {
    uint32_t len;
    uint64_t val;
    wasm_error_t err = decode_uleb128(r->pos, r->end, 32, &val, &len);
    if (err != WASM_OK) return err;
    r->pos += len;
    *out = (uint32_t)val;
    return WASM_OK;
}

static wasm_error_t reader_read_s32(wasm_reader_t *r, int32_t *out) {
    uint32_t len;
    int64_t val;
    wasm_error_t err = decode_sleb128(r->pos, r->end, 32, &val, &len);
    if (err != WASM_OK) return err;
    r->pos += len;
    *out = (int32_t)val;
    return WASM_OK;
}

// Read a vector count (u32) — used everywhere in Wasm binary
static wasm_error_t reader_read_vec_count(wasm_reader_t *r, uint32_t *out) {
    return reader_read_u32(r, out);
}

// Read raw bytes (for names, custom sections, etc.)
static wasm_error_t reader_read_bytes(
    wasm_reader_t *r, uint32_t len, const uint8_t **out)
{
    if (r->pos + len > r->end) return WASM_ERR_UNEXPECTED_END;
    *out = r->pos;
    r->pos += len;
    return WASM_OK;
}

// Read a UTF-8 name (length-prefixed byte vector)
static wasm_error_t reader_read_name(
    wasm_reader_t *r, const uint8_t **out_data, uint32_t *out_len)
{
    wasm_error_t err = reader_read_u32(r, out_len);
    if (err != WASM_OK) return err;
    return reader_read_bytes(r, *out_len, out_data);
}
```

这个 `wasm_reader_t` 结构将贯穿你整个解析器的实现。后续文档中提到的 "读取一个 u32" 或 "读取一个向量" 都是基于这个模式。

## 7. 测试建议

实现 LEB128 后，用以下测试用例验证：

```c
// Unsigned test cases
assert(decode_u32("\x00") == 0);
assert(decode_u32("\x01") == 1);
assert(decode_u32("\x7F") == 127);
assert(decode_u32("\x80\x01") == 128);
assert(decode_u32("\xE5\x8E\x26") == 624485);
assert(decode_u32("\xFF\xFF\xFF\xFF\x0F") == UINT32_MAX);

// Should fail: too many bytes
assert(decode_u32("\x80\x80\x80\x80\x80\x00") == ERROR);
// Should fail: unused bits set in final byte
assert(decode_u32("\xFF\xFF\xFF\xFF\x1F") == ERROR);

// Signed test cases
assert(decode_s32("\x00") == 0);
assert(decode_s32("\x7F") == -1);
assert(decode_s32("\x40") == -64);
assert(decode_s32("\xC0\xBB\x78") == -123456);
assert(decode_s32("\xFF\xFF\xFF\xFF\x07") == INT32_MAX);
assert(decode_s32("\x80\x80\x80\x80\x78") == INT32_MIN);
```

---

**下一步**：掌握 LEB128 后，你就具备了阅读 [01-binary-format.md](../01-binary-format.md) 的基础。二进制格式文档中的每一个整数字段都会标注它使用的是 `u32`（unsigned LEB128）还是 `s32`/`s33`/`s64`（signed LEB128），你将频繁回顾本文档中的解码实现。
