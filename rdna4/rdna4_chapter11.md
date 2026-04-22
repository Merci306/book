# 第十一章：全局、暂存和 Flat 地址空间操作

Flat、Global 和 Scratch 是一组 VMEM 指令，允许每线程访问全局内存、共享内存和私有内存。与缓冲区和图像指令不同，这些指令不使用资源常量。

- **Flat**：最通用的类型，每线程地址可映射到全局、私有或共享内存。以单一平坦地址空间寻址，特定内存地址孔径映射这些区域。Flat 加载/存储/原子指令同时作为 LDS 和 GLOBAL 指令执行。
- **Global**：当所有地址都落入全局内存时使用（而非 LDS 或 Scratch）。尽可能使用 Global（而非 Flat），因为 Global 不占用 LDS 资源。
- **Scratch**：访问暂存（私有）内存空间，是为 wave 或工作组分配的全局内存中的私有区域。

Flat 地址空间指令允许对通用内存地址指针进行加载/存储/原子访问，可解析到以下物理内存：
- 全局内存
- Scratch（私有）
- LDS（共享）
- 无效
- 不包括：GPR 或 LDS 参数

Scratch（线程私有内存）可通过 SCRATCH 内存指令或 FLAT 指令（地址落入孔径寄存器定义区域时）访问。有暂存内存分配的 wave，`SCRATCH_BASE` 寄存器初始化为该 wave 私有暂存内存的指针；无暂存内存的 wave，`SCRATCH_BASE` 初始化为零。

---

## 11.1 指令字段

| 字段 | 大小 | 描述 |
|------|------|------|
| OP | 8 | 操作码 |
| VADDR | 8 | 持有地址或偏移的 VGPR。64 位地址时 ADDR 存低位，ADDR+1 存高位。FLAT_* 指定地址；GLOBAL_* 当 SADDR 为 NULL 时指定地址，否则指定无符号字节偏移；SCRATCH 指定有符号字节偏移（SVE=1 时） |
| VSRC | 8 | 持有存储数据第一个 DWORD 的 VGPR，可用 0-4 个 DWORD |
| VDST | 8 | 加载返回数据或原子前值的 VGPR 目标 |
| SCOPE | 2 | 内存作用域 |
| TH | 3 | 内存时间提示。对于原子操作，TH 指示是否返回前值 |
| IOFFSET | 24 | 地址立即偏移：24 位有符号字节偏移 |
| SADDR | 7 | 提供地址或偏移的 SGPR，设为 NULL 禁用。Flat：未使用；Scratch：提供 32 位有符号偏移；Global：提供 64 位无符号基地址，VGPR 提供 32 位字节偏移 |
| SVE | 1 | Scratch VGPR 使能。1=包含 VGPR 32 位偏移；0=不使用 VGPR 寻址 |

---

## 表 67. 指令列表

| Flat | GLOBAL | Scratch |
|------|--------|---------|
| FLAT_LOAD_U8 | GLOBAL_LOAD_U8 | SCRATCH_LOAD_U8 |
| FLAT_LOAD_D16_U8 | GLOBAL_LOAD_D16_U8 | SCRATCH_LOAD_D16_U8 |
| FLAT_LOAD_D16_HI_U8 | GLOBAL_LOAD_D16_HI_U8 | SCRATCH_LOAD_D16_HI_U8 |
| FLAT_LOAD_I8 | GLOBAL_LOAD_I8 | SCRATCH_LOAD_I8 |
| FLAT_LOAD_D16_I8 | GLOBAL_LOAD_D16_I8 | SCRATCH_LOAD_D16_I8 |
| FLAT_LOAD_D16_HI_I8 | GLOBAL_LOAD_D16_HI_I8 | SCRATCH_LOAD_D16_HI_I8 |
| FLAT_LOAD_U16 | GLOBAL_LOAD_U16 | SCRATCH_LOAD_U16 |
| FLAT_LOAD_I16 | GLOBAL_LOAD_I16 | SCRATCH_LOAD_I16 |
| FLAT_LOAD_D16_B16 | GLOBAL_LOAD_D16_B16 | SCRATCH_LOAD_D16_B16 |
| FLAT_LOAD_D16_HI_B16 | GLOBAL_LOAD_D16_HI_B16 | SCRATCH_LOAD_D16_HI_B16 |
| FLAT_LOAD_B32 | GLOBAL_LOAD_B32 | SCRATCH_LOAD_B32 |
| FLAT_LOAD_B64 | GLOBAL_LOAD_B64 | SCRATCH_LOAD_B64 |
| FLAT_LOAD_B96 | GLOBAL_LOAD_B96 | SCRATCH_LOAD_B96 |
| FLAT_LOAD_B128 | GLOBAL_LOAD_B128 | SCRATCH_LOAD_B128 |
| FLAT_STORE_B8 | GLOBAL_STORE_B8 | SCRATCH_STORE_B8 |
| FLAT_STORE_D16_HI_B8 | GLOBAL_STORE_D16_HI_B8 | SCRATCH_STORE_D16_HI_B8 |
| FLAT_STORE_B16 | GLOBAL_STORE_B16 | SCRATCH_STORE_B16 |
| FLAT_STORE_D16_HI_B16 | GLOBAL_STORE_D16_HI_B16 | SCRATCH_STORE_D16_HI_B16 |
| FLAT_STORE_B32 | GLOBAL_STORE_B32 | SCRATCH_STORE_B32 |
| FLAT_STORE_B64 | GLOBAL_STORE_B64 | SCRATCH_STORE_B64 |
| FLAT_STORE_B96 | GLOBAL_STORE_B96 | SCRATCH_STORE_B96 |
| FLAT_STORE_B128 | GLOBAL_STORE_B128 | SCRATCH_STORE_B128 |
| — | GLOBAL_LOAD_ADDTID_B32 | — |
| — | GLOBAL_STORE_ADDTID_B32 | — |
| FLAT_ATOMIC_SWAP_B32 | GLOBAL_ATOMIC_SWAP_B32 | — |
| FLAT_ATOMIC_CMPSWAP_B32 | GLOBAL_ATOMIC_CMPSWAP_B32 | — |
| FLAT_ATOMIC_ADD_U32 | GLOBAL_ATOMIC_ADD_U32 | — |
| FLAT_ATOMIC_ADD_F32 | GLOBAL_ATOMIC_ADD_F32 | — |
| FLAT_ATOMIC_PK_ADD_F16 | GLOBAL_ATOMIC_PK_ADD_F16 | — |
| FLAT_ATOMIC_PK_ADD_BF16 | GLOBAL_ATOMIC_PK_ADD_BF16 | — |
| FLAT_ATOMIC_SUB_U32 | GLOBAL_ATOMIC_SUB_U32 | — |
| FLAT_ATOMIC_MIN_I32 | GLOBAL_ATOMIC_MIN_I32 | — |
| FLAT_ATOMIC_MIN_U32 | GLOBAL_ATOMIC_MIN_U32 | — |
| FLAT_ATOMIC_MAX_I32 | GLOBAL_ATOMIC_MAX_I32 | — |
| FLAT_ATOMIC_MAX_U32 | GLOBAL_ATOMIC_MAX_U32 | — |
| FLAT_ATOMIC_AND_B32 | GLOBAL_ATOMIC_AND_B32 | — |
| FLAT_ATOMIC_OR_B32 | GLOBAL_ATOMIC_OR_B32 | — |
| FLAT_ATOMIC_XOR_B32 | GLOBAL_ATOMIC_XOR_B32 | — |
| FLAT_ATOMIC_INC_U32 | GLOBAL_ATOMIC_INC_U32 | — |
| FLAT_ATOMIC_DEC_U32 | GLOBAL_ATOMIC_DEC_U32 | — |
| FLAT_ATOMIC_MIN_NUM_F32 | GLOBAL_ATOMIC_MIN_NUM_F32 | — |
| FLAT_ATOMIC_MAX_NUM_F32 | GLOBAL_ATOMIC_MAX_NUM_F32 | — |
| FLAT_ATOMIC_SWAP_B64 | GLOBAL_ATOMIC_SWAP_B64 | — |
| FLAT_ATOMIC_ADD_U64 | GLOBAL_ATOMIC_ADD_U64 | — |
| FLAT_ATOMIC_SUB_U64 | GLOBAL_ATOMIC_SUB_U64 | — |
| FLAT_ATOMIC_MIN_I64 | GLOBAL_ATOMIC_MIN_I64 | — |
| FLAT_ATOMIC_MIN_U64 | GLOBAL_ATOMIC_MIN_U64 | — |
| FLAT_ATOMIC_MAX_I64 | GLOBAL_ATOMIC_MAX_I64 | — |
| FLAT_ATOMIC_MAX_U64 | GLOBAL_ATOMIC_MAX_U64 | — |
| FLAT_ATOMIC_AND_B64 | GLOBAL_ATOMIC_AND_B64 | — |
| FLAT_ATOMIC_OR_B64 | GLOBAL_ATOMIC_OR_B64 | — |
| FLAT_ATOMIC_XOR_B64 | GLOBAL_ATOMIC_XOR_B64 | — |
| FLAT_ATOMIC_INC_U64 | GLOBAL_ATOMIC_INC_U64 | — |
| FLAT_ATOMIC_DEC_U64 | GLOBAL_ATOMIC_DEC_U64 | — |
| FLAT_ATOMIC_COND_SUB_U32（仅支持"返回前值"） | GLOBAL_ATOMIC_COND_SUB_U32（仅支持"返回前值"） | — |
| — | GLOBAL_ATOMIC_SUB_CLAMP_U32（仅支持"返回前值"） | — |
| — | GLOBAL_ATOMIC_ORDERED_ADD_B64（仅支持"返回前值"） | — |
| — | GLOBAL_INV | — |
| — | GLOBAL_WB | — |
| — | GLOBAL_WBINV | — |

---

## 11.1 指令类型

### 11.1.1 FLAT

Flat 指令集由加载、存储和内存原子操作组成。Flat 指令不使用资源常量（V#）或采样器（S#），但使用 wave 状态（`SCRATCH_BASE`）存储暂存空间信息（在某些线程地址解析到暂存空间时使用）。

Flat 指令同时作为 LDS 和 Global 指令执行，因此 Flat 指令同时递增 LOADcnt（或 STOREcnt）和 DScnt，两者都递减后才算完成：
- 加载、带返回的原子和 GLOBAL_INV：由 LOADcnt 跟踪
- 存储、无返回原子、GLOBAL_WB 和 GLOBAL_WBINV：由 STOREcnt 跟踪

当 Flat 指令的地址落入暂存（私有）空间时，使用不同的"混洗"寻址机制。地址（每线程）指向该线程拥有的特定 DWORD 暂存数据的内存空间，硬件将此地址映射到持有 wave 中所有线程数据的实际内存地址。

孔径检查在 VGPR 读取的值上执行，无效地址被路由到纹理单元。孔径检查在 IOFFSET 加入地址**之前**执行，因此添加 IOFFSET 将地址推入不同内存孔径时行为未定义。

对于地址落入 LDS 空间的线程，仅做如下地址检查：
```
超出范围 = Logical_ADDR[16:0] >= Wave_allocated_LDS
```

若线程未超出范围，地址通过丢弃上位地址位映射到 LDS 空间（多个虚拟地址可映射到同一物理 LDS 存储）。

### 11.1.2 Global

Global 操作在 VGPR 和全局内存之间传输数据。程序员负责确保没有线程访问 LDS 或私有空间。因此不使用 LDS 带宽，且只使用 LOADcnt（或 STOREcnt），不使用 DScnt。

Global 包含两条不使用任何 VGPR 寻址的指令（仅用 SGPR 和 IOFFSET）：
- `GLOBAL_LOAD_ADDTID_B32`
- `GLOBAL_STORE_ADDTID_B32`

### 11.1.3 Scratch

Scratch 指令类似于 Global，但访问经过混洗的私有（每线程）内存空间。不使用 LDS 带宽，只使用 LOADcnt（或 STOREcnt），不使用 DScnt。支持多 DWORD 访问和非对齐访问（非对齐较慢）。Scratch 指令不能访问 LDS，因此不进行错误检查，也不执行孔径检查。

---

## 11.2 寻址

Global、Flat 和 Scratch 各有自己的寻址模式。64 位地址的低位存储在 ADDR VGPR 中，高位存储在 ADDR+1 VGPR 中。

有 4 种不同的着色器指令：GLOBAL、SCRATCH、LDS、FLAT（基于每线程 VGPR 地址，可加载/存储全局内存、LDS 或暂存内存）。

### 表 68. 寻址模式选择

| 指令类型 | 模式 | SVE | SADDR |
|----------|------|-----|-------|
| Scratch | SV | 1 | NULL |
| Scratch | SS | 0 | ≠ NULL |
| Scratch | ST | 0 | NULL |
| Scratch | SVS | 1 | ≠ NULL |
| Flat 和 Global | GV | 忽略 | NULL |
| Global 专用 | GT | 由操作码指示 | |
| LDS | LDS | 由操作码指示 | |

### 地址计算公式

**Global 寻址：**
```
GV:   mem_addr = VGPR[U64] + IOFFSET[I24]
GVS:  mem_addr = SGPR[U64] + VGPR[U32] + IOFFSET[I24]
GT:   mem_addr = SGPR[U64] + IOFFSET[I24] + ThreadID*4
```

**LDS 寻址（DS 操作）：**
```
LDS:  LDS_ADDR = VGPR_addr[U32] + IOFFSET[U16]
LDS 地址相对于分配给此 wave 的 LDS 空间
```

**Scratch 寻址：**
```
SV:   mem_addr = SCRATCH_BASE[U64] + SWIZZLE(VGPR_offset[I32] + IOFFSET[I24], ThreadID)
SS:   mem_addr = SCRATCH_BASE[U64] + SWIZZLE(SGPR_offset[I32] + IOFFSET[I24], ThreadID)
SVS:  mem_addr = SCRATCH_BASE[U64] + SWIZZLE(SGPR_offset[I32] + VGPR_offset[I32] + IOFFSET[I24], ThreadID)
ST:   mem_addr = SCRATCH_BASE[U64] + SWIZZLE(IOFFSET[I24], ThreadID)
```

SWIZZLE() 内部的组合偏移必须为非负数。来自 SGPR 和 VGPR 的偏移值为有符号 32 位字节偏移。

Scratch ST 模式下，IOFFSET 必须与负载大小对齐：1-DWORD 为 4 字节对齐，4-DWORD 为 16 字节对齐。

### Flat 寻址

Flat 指令使用"GV"寻址模式，孔径检查基于地址确定每线程的全局/LDS/Scratch。孔径检查仅在基地址上执行，不含 IOFFSET。

```
普通（GV）：  addr[63:0] = VGPR[U64] + IOFFSET[I24]
```

孔径检查（基于 VGPR 读取的 64 位地址）：
```
isLDS     = (ADDR[63:32] == { SH_MEM_BASES.SHARED_BASE[15:0],  16'h0000 })
isScratch = (ADDR[63:32] == { SH_MEM_BASES.PRIVATE_BASE[15:0], 16'h0000 })
isHole    = ADDR[63:47] != (全零或全一) && !isLDS && !isScratch
isGlobal  = !isLDS && !isScratch && !isHole
```

根据每线程所属内存空间应用不同地址计算：

| 孔径 | 地址计算 |
|------|----------|
| GLOBAL | `mem_addr = addr` |
| SCRATCH (SV) | `mem_addr = SCRATCH_BASE[U64] + SWIZZLE(addr[31:0], ThreadID)` |
| LDS | `LDS_ADDR[U17] = VGPR(addr)[16:0] + IOFFSET[16:0]`（截断，可环绕）；范围检查：LDS_ADDR[U17] < LDS_SIZE |
| Hole | 内存违规 |

---

## 11.3 内存错误检查

缓存和 LDS 均可报告由错误地址引起的错误，原因包括：
- 无效地址（超出任何孔径）
- 向只读全局内存地址写入
- 非对齐数据（暂存访问可能非对齐）
- 超出范围地址：LDS 访问地址超出范围 [0, LDS_SIZE-1]

错误线程的策略：范围外的存储不写入值，读取返回零。孔径检查（无效地址）在添加地址偏移之前执行，其他检查在添加偏移后执行。

来自 LDS 或 VMEM 的寻址错误设置 wave 的 MEMVIOL 位并导致异常（陷阱）。

---

## 11.4 数据

FLAT 指令可在 VGPR 和/或内存中使用零到四个连续 DWORD 的数据。不进行数据格式转换。

"D16" 指令仅使用 VGPR 的 16 位而非完整 32 位：
- "D16_HI"：读写仅高 16 位
- "D16"：读写低 16 位

---

## 11.5 块 VGPR 加载与存储

这些指令可高效地将最多 32 个连续 VGPR 移入/移出内存：

```
GLOBAL_LOAD_BLOCK   VDST, VADDR, SADDR, IOFFSET, (M0)
GLOBAL_STORE_BLOCK  VSRC, VADDR, SADDR, IOFFSET, (M0)
SCRATCH_LOAD_BLOCK  VDST, VADDR, SADDR, IOFFSET, (M0)
SCRATCH_STORE_BLOCK VSRC, VADDR, SADDR, IOFFSET, (M0)
```

数据从连续 VGPR 传输到内存中连续 DWORD（每线程），可跳过某些 VGPR（但仍跳过内存位置，不在内存中压缩）。

M0 携带一个位掩码，指示哪些 VGPR 需要加载/存储（1）或跳过（0），M0 的 LSB 对应第一个 VGPR。

寻址模式限制：
- GLOBAL_LOAD/STORE：支持"GV"和"GVS"模式，不支持"GT"
- SCRATCH_LOAD/STORE：支持"SS"、"SV"和"SVS"模式，不支持"ST"

伪代码：
```
for (n = 0..31)
    if (M0[n])
        使用 IOFFSET 和 VGPR[vdst/vdata + n] 进行加载/存储
    IOFFSET += 4 字节
```

若 M0==0，不传输任何数据。整个块加载/存储用 LOADcnt 或 STOREcnt 跟踪：整个块传输递增 1，传输完成时递减。

### 11.5.1 错误处理

- 超出范围的地址 VGPR：遵循与 Global 和 Scratch 相同的规则
- 加载时超出范围的目标 VGPR：每个 DWORD 单独检查，在范围内的正常执行，超出范围的被忽略
- 存储时超出范围的 VSRC VGPR：每个 DWORD 单独检查，在范围内的按规定提供数据，超出范围的从 VGPR0 读取数据

---

## 11.6 带转置的 WMMA 矩阵加载操作

矩阵通常以平铺方式存储在内存中。RDNA4 提供指令简化将 16×16 矩阵块加载到 VGPR 的过程，内存中矩阵的主序（行 vs. 列）与 WMMA 操作所需的相反。这些指令加载并转置内存中的矩阵块到 VGPR。

### 11.6.1 密集矩阵

**行主序 A 矩阵（M 行 × K 列）：**
```
Memory_address = (col# + row# * K) * ElementSize
```

**列主序 B 矩阵（K × N）：**
```
Memory_address = (col# * K + row#) * ElementSize
```

### 11.6.2 WMMA 加载转置指令

若 EXEC==0，指令等同于 S_NOP；否则要求 EXEC 全置 1，否则操作未定义。

| 指令 | 描述 |
|------|------|
| GLOBAL_LOAD_TR_B128 | 将 16×16 的 16 位数据矩阵加载到 VGPR 并在行主序和列主序之间转置。wave32：加载到 4 个连续 VGPR；wave64：加载到 2 个连续 VGPR，仅使用通道 0-31 的地址（128 位数据），忽略通道 32-63 的地址 |
| GLOBAL_LOAD_TR_B64 | 将 16×16 的 8 位数据矩阵加载到 VGPR 并转置。wave32：加载到 2 个连续 VGPR；wave64：加载到 1 个 VGPR，仅使用通道 0-31 的地址（64 位数据），忽略通道 32-63 的地址 |

两条指令的所有字段与 `GLOBAL_LOAD_B64` 和 `_B128` 相同，作为加载由 LOADcnt 跟踪。

**矩阵加载指令选择表：**

| 内存顺序 | 元素大小 | Wave 大小 | VGPR 布局 | 使用指令 |
|----------|----------|-----------|-----------|---------|
| 行主序 | 16 位 | 32 | 行主序 | GLOBAL_LOAD_B128 |
| 行主序 | 16 位 | 64 | 行主序 | GLOBAL_LOAD_B64 |
| 列主序 | 16 位 | 32 | 行主序 | GLOBAL_LOAD_TR_B128（每通道写 128 位 × 32 通道） |
| 列主序 | 16 位 | 64 | 行主序 | GLOBAL_LOAD_TR_B128（每通道写 64 位 × 64 通道） |
| 行主序 | 8 位 | 32 | 行主序 | GLOBAL_LOAD_B64 |
| 行主序 | 8 位 | 64 | 行主序 | GLOBAL_LOAD_B32 |
| 列主序 | 8 位 | 32 | 行主序 | GLOBAL_LOAD_TR_B64（每通道写 64 位 × 32 通道） |
| 列主序 | 8 位 | 64 | 行主序 | GLOBAL_LOAD_TR_B64（每通道写 32 位 × 64 通道） |

> 注：当"VGPR 布局"为列主序时，上表同样适用，只需将"内存顺序"中行/列的含义互换。
