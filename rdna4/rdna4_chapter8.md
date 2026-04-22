# 第8章：标量内存操作

标量内存加载（SMEM）指令允许着色器程序通过常量缓存（"Kcache"）将数据从内存加载到 SGPR 中。指令可加载 1 至 16 个 DWORD 的数据。数据直接加载到 SGPR，不进行任何格式转换（不支持数据格式化）。

标量单元将内存中连续的 DWORD 加载到 SGPR 中。SMEM 加载的常见用途包括加载 ALU 常量，以及间接查找 T#/S#/V#。

加载有两种形式：一种仅使用基址指针，另一种使用缓冲区资源（V#）来提供基址、大小和步幅。

## 8.1 微码编码

标量内存加载指令使用 SMEM 微码格式进行编码。

各字段描述见下表：

### 表44. SMEM 编码字段描述

| 字段 | 位宽 | 描述 |
|------|------|------|
| OP | 6 | 操作码。见下一张表。 |
| SDATA | 7 | SDATA 指定用于接收数据的 SGPR。<br>• 加载 2 个 DWORD 时，SDST-sgpr 必须为偶数。<br>• 加载 3 个或更多 DWORD 时，SDST-gpr 必须按 4 个 SGPR 的倍数对齐。<br>• SDATA 必须为 SGPR 或 VCC，不能为 EXEC 或 M0，允许为 NULL。 |
| SBASE | 6 | SGPR 对（SBASE 隐含 LSB 为零），提供基地址；或对于 BUFFER 指令，为 4 个 SGPR（按 4 SGPR 对齐），保存缓冲区资源。<br>对于 BUFFER 指令，仅使用以下资源字段：base、stride、num_records。 |
| IOFFSET | 24 | 指令地址偏移：有符号立即字节偏移。<br>负 IOFFSET 仅适用于 S_LOAD；对 S_BUFFER 使用负 IOFFSET 会导致 MEMVIOL。 |
| SOFFSET | 7 | 包含 32 位无符号字节偏移的 SGPR。只能指定 SGPR、M0，或设置为 "NULL" 表示不使用（偏移=0）。 |
| SCOPE | 2 | 内存作用域 |
| TH | 2 | 内存时序提示。有关 SCOPE 和 TH 位的更多信息，请参阅缓存控制相关章节。 |

### 表45. SMEM 指令

| 操作码# | 名称 | 操作码# | 名称 |
|---------|------|---------|------|
| 0 | S_LOAD_B32 | 19 | S_BUFFER_LOAD_B256 |
| 1 | S_LOAD_B64 | 20 | S_BUFFER_LOAD_B512 |
| 2 | S_LOAD_B128 | 21 | S_BUFFER_LOAD_B96 |
| 3 | S_LOAD_B256 | 24 | S_BUFFER_LOAD_I8 |
| 4 | S_LOAD_B512 | 25 | S_BUFFER_LOAD_U8 |
| 5 | S_LOAD_B96 | 26 | S_BUFFER_LOAD_I16 |
| 8 | S_LOAD_I8 | 27 | S_BUFFER_LOAD_U16 |
| 9 | S_LOAD_U8 | 33 | S_DCACHE_INV |
| 10 | S_LOAD_I16 | 36 | S_PREFETCH_INST |
| 11 | S_LOAD_U16 | 37 | S_PREFETCH_INST_PC_REL |
| 16 | S_BUFFER_LOAD_B32 | 38 | S_PREFETCH_DATA |
| 17 | S_BUFFER_LOAD_B64 | 39 | S_BUFFER_PREFETCH_DATA |
| 18 | S_BUFFER_LOAD_B128 | 40 | S_PREFETCH_DATA_PC_REL |

这些指令从内存加载 1-16 个 DWORD。SDATA 字段指示将数据加载到哪些 SGPR，地址由 SBASE、OFFSET 和 SOFFSET 字段组成。

SMEM 加载可将 NULL 用作目标 SGPR，从而实现"带确认的数据预取"。

### 8.1.1 标量内存寻址

非缓冲区 S_LOAD 指令使用以下公式计算内存地址：

```
ADDR = SGPR[base] + IOFFSET + { M0 或 SGPR[offset] 或零 }
```

地址的所有组成部分（base、offset、IOFFSET、M0）均以字节为单位，但每个组成部分的两个 LSB 均被忽略并视为零，8 位和 16 位加载除外。对于 16 位加载，每个组成部分的一个 LSB 被忽略并视为零。对于 8 位加载，每个组成部分使用完整的字节地址。

对于 S_LOAD（非缓冲区）指令，若（IOFFSET + (M0、SGPR[offset] 或零)）为负数，则属于非法且未定义的操作。

对于 S_BUFFER_LOAD，IOFFSET 为负数是非法且未定义的操作。

### 8.1.2 使用缓冲区资源（V#）的加载

S_BUFFER_LOAD 指令使用类似的公式，但基地址来自缓冲区资源的 base_address 字段。

使用的缓冲区资源字段：base_address、stride、num_records；其他字段被忽略。

标量内存加载不支持"swizzled"缓冲区。stride 仅用于内存地址边界检查，不用于计算访问地址。stride 和缓冲区大小必须是 4 的倍数。

SMEM 仅提供 SBASE 地址（字节）和偏移（字节）。任何"index × stride"必须在着色器代码中手动计算，并在 SMEM 之前加入偏移。IOFFSET 必须为非负数——IOFFSET 为负值会导致 MEMVIOL。

V#.base 的两个 LSB 被忽略并视为零，8 位和 16 位加载除外。对于 16 位加载，V#.base 的一个 LSB 被忽略并视为零。对于 8 位加载，使用 V#.base 的完整字节地址。

`m_*` 组成部分来自缓冲区资源（V#）：

```
def alignToSize(addr): return (dataSize == 8bit ? addr :
                               dataSize == 16bit ? addr & ~1 :
                               addr & ~3)
resource  = { SGPR[SBASE*2+3], SGPR[SBASE*2+2], SGPR[SBASE*2+1], SGPR[SBASE*2] }
m_base        = alignToSize(resource[47:0], dataSize)
m_stride      = resource[61:48]
m_num_records = resource[95:64]
offset     = alignToSize(IOFFSET) + alignToSize(SOFFSET)
m_size     = alignToSize((m_stride == 0 ? 1 : m_stride) * m_num_records)
addr       = m_base + offset
SGPR[SDST] = load_dword_from_dcache(addr, m_size)
```

若加载超过 1 个 DWORD，数据依次返回到 SDST+1、SDST+2 等，偏移每个 DWORD 递增 4 字节。

注：V#.stride_scale 字段被忽略。

### 8.1.3 8 位和 16 位数据加载

- `S_LOAD_U8`：加载 8 位无符号数据，零扩展至 32 位
- `S_LOAD_I8`：加载 8 位有符号数据，符号扩展至 32 位
- `S_LOAD_U16`：加载 16 位无符号数据，零扩展至 32 位
- `S_LOAD_I16`：加载 16 位有符号数据，符号扩展至 32 位

S_BUFFER_LOAD 同上。

16 位加载在内存中必须按 16 位对齐。

### 8.1.4 S_DCACHE_INV

这些指令使整个标量常量缓存失效，不向 SDST 返回任何内容。S_DCACHE_INV 没有地址或数据参数。

## 8.2 依赖性检查

标量内存加载返回数据的顺序可能与发出指令的顺序不同；当加载跨越两个缓存行时，可在不同时间返回部分结果。着色器程序使用 KMcnt 计数器来判断数据何时已返回到 SDST SGPR，方法如下：

- 每次取单个 DWORD 或缓存失效，KMcnt 加 1。
- 每次取两个或更多 DWORD，KMcnt 加 2。
- 每条指令完成时，KMcnt 减去相同数量。

由于指令可能乱序返回，使用此计数器的唯一合理方式是执行"S_WAIT_KMCNT 0"；这会等待所有先前 SMEM 的数据返回后再继续执行。

缓存失效指令在着色器等待 KMcnt==0 之前不视为已完成。

## 8.3 标量内存子句与分组

**子句（clause）** 是从 S_CLAUSE 开始，持续 2-63 条指令的序列。子句将指令仲裁器锁定在当前 wave 上，直到子句执行完毕。

**分组（group）** 是代码中出现的相同类型指令的集合，不一定以子句形式执行。遇到非 SMEM 指令时，分组结束。标量内存指令以分组方式发出，硬件不强制单个 wave 在其他 wave 发出指令之前执行完整个分组。

分组限制：
- INV 必须单独成组，不能出现在子句中。

## 8.4 对齐与边界检查

**SDST**

对于取 2 个 DWORD，SDST 的值必须为偶数；对于更大的取操作，必须是 4 的倍数。若不遵守此规则，可能导致数据无效。

**SBASE**

对于 S_BUFFER_LOAD（指定 SGPR 地址为 4 的倍数），SBASE 值必须为偶数。若 SBASE 超出范围，则使用 SGPR0 的值。

**OFFSET**

OFFSET 的值没有对齐限制。

### 8.4.1 地址与 GPR 范围检查

硬件同时检查地址是否超出范围（仅 BUFFER 指令）以及源或目标 SGPR 是否超出范围。

内存地址强制对齐：基地址强制为 DWORD 对齐；DWORD 或更大的加载强制内存地址为 DWORD 对齐；16 位数据的加载强制地址为 2 字节对齐；字节加载无强制对齐。

| 情况 | 说明 |
|------|------|
| 地址超出范围（条件）| `offset >= ((stride==0 ? 1 : stride) * num_records)`，其中 "offset" 为：`IOFFSET + {M0 或 sgpr-offset}` |
| 缓冲区加载中内存地址超出范围的 DWORD | 返回零。若多 DWORD 请求（如 S_BUFFER_LOAD_B256）部分超出范围，范围内的 DWORD 正常返回数据，超出范围的 DWORD 返回零。 |
| 源 SGPR 超出范围 | 若任何源数据超出 SGPR 范围（部分或全部），则使用值"零"代替。 |
| 目标 SGPR 超出范围 | 若目标 SGPR 部分或全部超出范围，则该指令不向 SGPR 写回任何数据。 |

## 8.5 标量预取指令

着色器可通过着色器指令请求将指令或数据预取到第一级缓存中。这些指令不使用 KMcnt。

缓存重用策略通过 SCOPE 和 TH 位控制，适用于数据预取，不适用于指令预取。

若 `MODE.SCALAR_PREFETCH_EN == 0`，这些指令将被跳过（视为 S_NOP）。

**预取长度**：SOFFSET 保存一个带有长度的 SGPR/M0（int），或设置为 NULL 表示零；SDATA 携带一个立即常量值（设为 0 表示忽略）。两者之和为 0-31（模 32，高位舍去），表示预取大小为 1-32 个 128 字节块。

地址的所有组成部分（base、offset、IOFFSET、M0）均以字节为单位，但每个组成部分的两个 LSB 被忽略并视为零。

### 8.5.1 预取数据

这些指令使用简单地址、缓冲区资源（V#）或 PC 相对地址，将数据预取到常量缓存中。

```
S_PREFETCH_DATA   SBASE(addr)  IOFFSET  SOFFSET(length)  SDATA(as immediate value)
MemAddr = (SBASE[63:0].u64 + IOFFSET.i24) & ~0x07f    // 强制 128B 对齐
Length  = SOFFSET + #SDATA : SGPR/M0 + immediate, limit to 1-32 chunks, units of 128B

S_BUFFER_PREFETCH_DATA   SBASE(V#)  IOFFSET  SOFFSET(length)
MemAddr = (SBASE[47:0].u64 + IOFFSET.i24) & ~0x07f
Length  = SOFFSET + #SDATA : SGPR/M0 + immediate, limit to 1-32 cacheline-pairs, units of 128B
// 注：S_BUFFER_PREFETCH_DATA 不对预取地址执行边界检查。

S_PREFETCH_DATA_PC_REL  IOFFSET  SOFFSET(length)
Addr   = (PC + IOFFSET.i24) & ~0x07f.    // 强制 128B 对齐
Length = SOFFSET + #SDATA : SGPR/M0 + immediate, limit to 1-32 chunks, units of 128B
// "PC" 在计算地址时指向该指令的后一条指令。
```

注：S_BUFFER_PREFETCH_DATA 不对预取地址执行边界检查。S_BUFFER_PREFETCH_DATA 不支持负 IOFFSET，但若 IOFFSET 为负数，不会返回 MEMVIOL（预取请求被丢弃）。

### 8.5.2 预取指令

这些指令使用简单地址或 PC 相对地址，将指令预取到指令缓存中。

```
S_PREFETCH_INST   SBASE(addr)  IOFFSET  SOFFSET(length)  SDATA(as immediate value)
MemAddr = (SBASE[63:0].u64 + IOFFSET.i24) & ~0x07f    // 强制 128B 对齐
Length  = SOFFSET + #SDATA : SGPR/M0 + immediate, limit to 1-32 chunks, units of 128B

S_PREFETCH_INST_PC_REL  IOFFSET  SOFFSET(length)   SDATA(as immediate value)
Addr   = (PC + IOFFSET.i24) & ~0x07f.    // 强制 128B 对齐；SBASE 被忽略
Length = SOFFSET + #SDATA : SGPR/M0 + immediate, limit to 1-32 chunks, units of 128B
// "PC" 在计算地址时指向该指令的后一条指令。
```
