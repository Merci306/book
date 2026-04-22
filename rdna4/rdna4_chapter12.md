# 第12章：本地数据共享操作

本地数据存储（LDS）是一种低延迟的 RAM 暂存区，用于临时数据存储以及工作组内线程间的数据共享。通过 LDS 访问数据的延迟可能显著低于访问主存，且带宽更高。

对于计算工作负载，LDS 提供了一种简单的方法，用于在同一工作组内不同 wave 的线程之间传递数据。对于图形渲染，LDS 也用于保存顶点参数供像素着色器使用。

LDS 的高带宽不仅来自其与 ALU 的物理接近性，也得益于对其存储 bank 的同时访问能力。因此，可以并发执行多条存取操作。但是，若多个访问同时命中同一 bank，则会发生 bank 冲突。在这种情况下，对于索引操作和原子操作，硬件会将对同一 bank 的并发访问转为串行访问，从而降低 LDS 的有效带宽。因此，为提高吞吐量（最优效率），应尽量避免 bank 冲突。了解请求调度和地址映射机制是实现这一目标的关键。

数据可通过两种方式加载到 LDS：使用"DS"指令将数据从 VGPR 传输到 LDS，或从内存直接加载。从内存加载时，数据可先加载到 VGPR，某些类型的加载也可直接从内存加载到 LDS。将 LDS 数据写回全局内存时，需先从 LDS 读取数据并放入工作项的 VGPR，再写出到全局内存。为有效利用 LDS，着色程序必须对在全局内存与 LDS 之间传输的数据执行大量操作。

LDS 空间按工作组（或在未使用工作组时按 wave）分配，并记录在专用的 LDS-base/size（分配）寄存器中，这些寄存器不可由着色器写入，从而将所有 LDS 访问限制在该工作组或 wave 所拥有的空间内。

## 12.1 概述

每个工作组处理器（WGP）有 128 kB 内存，分为 64 个 DWORD 宽的 bank（RAM）。这 64 个 bank 进一步细分为两组，每组 32 个：32 个 bank 对应 WGP 内的一对 SIMD32，另外 32 个 bank 对应另一对 SIMD32。每个 bank 是一个 512×32 双端口 RAM（每时钟周期 1 读/1 写）。DWORD 按顺序分配到各 bank，但所有 bank 可同时执行存取操作。一个工作组最多可申请 64 kB 内存。

LDS 原子操作在 LDS 硬件内部执行。尽管 ALU 不直接参与这些操作，但 LDS 执行这些功能时仍会产生延迟。

### 12.1.1 LDS 模式与分配：CU 模式 vs WGP 模式

wave 工作组以 CU 或 WGP 两种模式之一进行调度。详情请参阅相关章节：WGP 与 CU 模式。

### 12.1.2 LDS 访问方式

LDS 可通过以下方式访问：

**索引加载/存储与原子操作**
地址来自 VGPR，数据往/返于 VGPR。LDS 操作最多需要 3 个输入：2 个数据 + 1 个地址，以及立即返回的 VGPR。

**直接加载（Direct Load）**
从 LDS 读取单个 DWORD，并将数据广播到所有通道的 VGPR 中。

**参数插值加载（Parameter Interpolation Load）**
按 quad 从 LDS 读取像素参数并加载到一个 VGPR。读取每个 quad 的 3 个参数（P1、P1-P0 和 P2-P0），并将其加载到 quad 内的第 0、1、2 通道（第 4 个通道接收零值）。

## 12.2 像素参数插值

对于像素 wave，顶点属性数据在 wave 启动前预加载到 LDS，重心坐标（I、J）预加载到 VGPR。参数插值通过使用 `DS_PARAM_LOAD` 将属性数据从 LDS 加载到 VGPR，再使用 `V_INTERP` 指令对每个像素进行插值来实现。

LDS 参数加载用于读取顶点参数数据并存入 VGPR，以供参数插值使用。这些指令的工作方式类似于内存指令，区别在于它们使用 EXPcnt 来跟踪未完成的读取操作，并在数据到达 VGPR 时递减 EXPcnt。

像素着色器可以在参数数据写入 LDS 之前就被启动。一旦数据在 LDS 中就绪，wave 的 STATUS 寄存器中的"LDS_READY"位会被置为 1。如果在 LDS_READY 被置位之前就发出 `DS_DIRECT_LOAD` 或 `DS_PARAM_LOAD`，像素着色器 wave 会阻塞等待。

最常见的插值形式是用重心坐标"I"和"J"对顶点参数进行加权，其计算公式为：

```
Result = P0 + I * P10 + J * P20
    其中 "P10" = (P1 - P0)，"P20" = (P2 - P0)
```

参数插值涉及两类指令：
- `DS_PARAM_LOAD`：从 LDS 读取打包的参数数据到 VGPR（数据按 quad 打包）
- `V_INTERP_*`：VALU FMA 指令，用于将参数数据在 quad 内各通道间解包

### 12.2.1 LDS 参数加载

参数加载仅在 CU 模式下可用（WGP 模式不支持）。

`DS_PARAM_LOAD` 从 LDS 读取一个 32 位属性的 3 个参数（P0、P10、P20），或两个 16 位属性的 3 个参数，并写入 VGPR。一个 quad 内 4 个像素共用这 3 个参数（P0、P10、P20），这些值分别分布在每个 quad 的 VGPR 通道 0、1 和 2 中。之后使用带 DPP 的 FMA 进行插值，使每个通道以其 I 或 J 值与 quad 共享的 P0、P10 和 P20 组合计算。

**表 69. VDSDIR 指令字段**

| 字段 | 大小（位） | 描述 |
|------|-----------|------|
| OP | 2 | 操作码：0 = DS_PARAM_LOAD；1 = DS_DIRECT_LOAD；2、3：保留 |
| WAITVDST | 4 | 等待之前发出的仍未完成的 VALU 指令数量降至不超过此值，用于避免 VGPR 上的写后读（WAR）冒险 |
| VDST | 8 | 目标 VGPR |
| ATTR_CHAN | 2 | 属性通道：0=X，1=Y，2=Z，3=W。DS_DIRECT_LOAD 中不使用 |
| ATTR | 6 | 属性编号：0 - 32。DS_DIRECT_LOAD 中不使用 |
| WAITVMVS | 1 | 等待之前所有 VMEM 操作读完其 VGPR 源操作数：0=等待，1=不等待（类似 S_WAITCNT） |
| (M0) | 32 | DS_DIRECT_LOAD：`{ 13'b0, DataType[2:0], LDS_address[15:0] }` // 地址单位为字节<br>DS_PARAM_LOAD：M0 自动使用，M0 必须包含：`{ 1'b0, new_prim_mask[15:1], lds_param_offset[15:0] }` |

M0 寄存器被该指令隐式读取，必须在这些指令之前初始化。

- **new_prim_mask**：每个 quad 对应一个位的掩码，表示该 quad 是否开始新图元；0 表示与前一个 quad 属于同一图元。wave 的第一个 quad 默认隐含为"1"（每个 wave 都从新图元开始），因此第 0 位被省略。
- **lds_param_offset**：参数偏移量，指示参数在 LDS 中的起始地址。该地址之前的空间可用作 wave 的临时存储。`lds_param_offset` 的位 [6:0] 必须为零。

**DS_PARAM_LOAD 示例（new_prim_mask[3:0] = 0110）**

```
LDS_ADDR = lds_base + param_offset + attr# * numPrimsInVector * 12DWORDs + prim# * 12 + attr_offset
(attr_offset = 0..11：0 = P0.x，1 = P0.Y，… 11 = P2.W)
```

硬件从 NewPrimMask 中推导出 NumPrimInVec 和 Prim#（0..15）。

若目标 VGPR 超出范围，加载操作仍然执行，但 EXEC 被强制置零。

`DS_PARAM_LOAD` 和 `DS_DIRECT_LOAD` 按 quad 使用 EXEC（若 quad 中任意像素有效，则数据写入该 quad 的所有 4 个像素/线程）。

#### 12.2.1.1 16 位参数数据

16 位参数在 LDS 中以两个属性配对打包在 DWORD 中：ATTR0.X 与 ATTR1.X 共用一个 DWORD。也存在一种非打包模式，即参数存储在 DWORD 低半部分（一个 16 位参数占用一个 DWORD 的低 16 位）。这些属性可使用相同的 `DS_PARAM_LOAD` 指令读取，并返回包含 2 个属性的打包 DWORD（当它们被打包时）。之后可结合特定的混合精度 FMA 操作码、DPP（用于选择 P0、P10 或 P20）以及 OPSEL（用于选择高/低 16 位）进行插值。重心坐标为 32 位，不是 16 位。

#### 12.2.1.2 参数加载数据冒险规避

以下数据依赖规则同时适用于参数加载和直接加载。

`DS_DIRECT_LOAD` 和 `DS_PARAM_LOAD` 从 LDS 读取数据并写入 VGPR，使用 EXPcnt 跟踪指令的完成状态和 VGPR 写入情况。

着色程序有责任确保避免数据冒险。由于这些指令走的路径与 VALU 指令不同，之前的 VALU 指令可能仍在读取 LDS 指令将要写入的 VGPR，从而可能引发冒险。

EXPcnt 用于跟踪读后写（RAW）冒险，即 `DS_PARAM_LOAD` 写入某 VGPR 后，另一条指令再读取该 VGPR 的情况。着色程序使用 `S_WAIT_EXPCNT` 来等待 `DS_DIRECT_LOAD` 或 `DS_PARAM_LOAD` 的结果在 VGPR 中可用，然后再由后续指令读取。VINTERP 指令提供了"wait_EXPcnt"字段以协助避免此类冒险。

当 EXEC==0 且 EXPcnt==0 时，这些指令被跳过（类似内存操作）。

来自同一 wave 的 export 指令与 LDS direct/param 指令可能不按顺序完成（两者均使用 EXPcnt），若它们交叠执行则需要"S_WAIT_EXPCNT 0"：

```
DS_PARAM_LOAD V2
S_WAIT_EXPCNT 0
```

若某 VALU 指令读取某 VGPR 后，`DS_PARAM_LOAD` 又写入该 VGPR，则存在写后读（WAR）冒险：`DS_PARAM_LOAD` 可能在 VALU 读取其源 VGPR 之前就覆盖了它。用户必须通过设置 `DS_PARAM_LOAD` 指令的"wait_va_vdst"字段来避免此问题。将该字段设为零是最安全的设置，它指示在发出此 `DS_PARAM_LOAD` 时允许仍未完成的 VALU 指令的最大数量。

另一个潜在数据冒险是：`DS_PARAM_LOAD` 覆盖了某个 VGPR，而该 VGPR 尚未被之前的 VMEM 指令（LDS、纹理、缓冲区、Flat）读取为源操作数。为避免此冒险，用户必须确保 VMEM 指令已读取其源 VGPR。可通过在 `DS_PARAM_LOAD` 之前发出任意 VALU 或 export 指令来实现，或使用 WAITVMVS 字段。将该字段设为零是最安全的值，可防止此类冒险。

## 12.3 VALU 参数插值

参数插值使用 FMA 操作实现，其中内置了 DPP 操作，用于将每 quad 的 P0/P10/P20 值解包到各通道中。由于该指令从相邻通道读取数据，隐式 DPP 的行为等同于"fetch invalid = 1"，因此指令可以从 EXEC==0 的相邻通道读取数据，而不会得到零值。标准插值计算：

```
每像素参数 = P0 + I * P10 + J * P20   // I、J 为每像素值；P0/P10/P20 为每图元值
```

参数插值通过一对指令实现：

```
V_INTERP_P10_F32  V5, V0, V1, v2  // tmp = P0 + I*P10  使用 DPP8=1,1,1,1,5,5,5,5；Src2(P0) 使用 DPP8=0,0,0,0,4,4,4,4
V_INTERP_P20_F32  V5, V3, V4, V5  // dst = J*P20 + tmp  使用 DPP8=2,2,2,2,6,6,6,6
```

**表 70. 参数插值指令字段**

| 字段 | 大小（位） | 描述 |
|------|-----------|------|
| OP | 5 | 指令操作码：<br>`V_INTERP_P10_F32` // tmp = P0 + I*P10，2 个源上硬编码 DPP8<br>`V_INTERP_P2_F32` // D = tmp + J*P20，1 个源上硬编码 DPP8<br>`V_INTERP_P10_F16_F32` // tmp = P0 + I*P10，2 个源上硬编码 DPP8<br>`V_INTERP_P2_F16_F32` // D = tmp + J*P20，1 个源上硬编码 DPP8<br>`V_INTERP_RTZ_P10_F16_F32` // 同上，但舍入模式为向零舍入<br>`V_INTERP_RTZ_P2_F16_F32` // 同上，但舍入模式为向零舍入 |
| SRC0 | 9 | 第一个参数 VGPR：来自 LDS 存储于 VGPR 中的参数数据（P0 或 P20） |
| SRC1 | 9 | 第二个参数 VGPR：I 或 J 重心坐标 |
| SRC2 | 9 | 第三个参数 VGPR："P10"操作存放 P10 数据；"P2"操作存放"P10"操作的中间结果 |
| VDST | 8 | 目标 VGPR |
| NEG | 3 | 对输入取反（翻转符号位）。bit 0 对应 src0，bit 1 对应 src1，bit 2 对应 src2。对于 16 位插值，同时作用于低半部分和高半部分 |
| WaitEXP | 3 | 在发出此指令前等待 EXPcnt 降至不超过该值。用于等待特定的 DS_PARAM_LOAD 完成 |
| OPSEL | 4 | 16 位运算的操作数选择：1=选高半部分，0=选低半部分。[0]=src0，[1]=src1，[2]=src2，[3]=dest<br>dest=0 时：dest_vgpr[31:0] = {prev_dst_vgpr[31:16], result[15:0]}<br>dest=1 时：dest_vgpr[31:0] = {result[15:0], prev_dst_vgpr[15:0]}<br>OPSEL 只能用于 16 位操作数，其他操作数/结果时必须为零 |
| CM | 1 | 将结果截断到 [0, 1.0] |

VINTERP 指令内置了"S_WAIT_EXPCNT"功能，便于解决 `DS_PARAM_LOAD` 产生的数据冒险。

**指令限制：**
- `V_INTERP` 指令不检测或报告异常
- `V_INTERP` 指令不支持对通常来自 LDS 数据的输入进行数据转发（V_INTERP_P10_* 的源 A 和 C，以及 V_INTERP_P2_* 的源 A）

VGPR 预加载内容可包含以下部分或全部：
- `I_persp_sample`、`J_persp_sample`、`I_persp_center`、`J_persp_center`
- `I_persp_centroid`、`J_persp_centroid`
- `I/W`、`J/W`、`1.0/W`
- `I_linear_sample`、`J_linear_sample`
- `I_linear_center`、`J_linear_center`
- `I_linear_centroid`、`J_linear_centroid`

这些指令消耗由 `DS_PARAM_LOAD` 提供的数据。指令内置"S_WAIT_EXPCNT <= N"能力，支持高效的软件流水线：

```
ds_param_load V0,  attr0
ds_param_load V10, attr1
ds_param_load V20, attr2
ds_param_load V30, attr3
v_interp_p0    V1,  V0[1],  Vi, V0[0]    S_WAIT_EXPCNT <=3  // 等待 V0
v_interp_p0    V11, V10[1], Vi, V10[0]   S_WAIT_EXPCNT <=2
v_interp_p0    V21, V20[1], Vi, V20[0]   S_WAIT_EXPCNT <=1
v_interp_p0    V31, V30[1], Vi, V30[0]   S_WAIT_EXPCNT <=0  // 等待 V30
v_interp_p2    V2,  V0[2],  Vj, V1
v_interp_p2    V12, V10[2], Vj, V11
v_interp_p2    V22, V20[2], Vj, V21
v_interp_p2    V32, V30[2], Vj, V31
```

### 12.3.1 16 位参数插值

16 位插值对打包在 16 位 VGPR 中的一对属性值进行操作。这些操作使用相同的 I 和 J 值进行插值。OPSEL 用于选择数据的高半部分或低半部分。

16 位插值指令存在将舍入模式覆盖为"向零舍入"的变体。

```
V_INTERP_P10_F16_F32  dst.f32 = vgpr_hi/lo.f16 * vgpr.f32 + vgpr_hi/lo.f16  // tmp = P10 * I + P0
  允许 OPSEL；Src0 使用 DPP8=1,1,1,1,5,5,5,5；Src2 使用 DPP8=0,0,0,0,4,4,4,4

V_INTERP_P2_F16_F32   dst.f16 = vgpr_hi/lo.f16 * vgpr.f32 + vgpr.f32        // dst = P2 * J + tmp
  允许 OPSEL；Src0 使用 DPP8=2,2,2,2,6,6,6,6
```

## 12.4 LDS 直接加载

直接加载仅在 CU 模式下可用，WGP 模式不支持。

`DS_DIRECT_LOAD` 指令从 LDS 读取单个 DWORD，并将其返回到 VGPR，广播到 wave 中所有活跃通道。M0 提供地址和数据类型。`DS_DIRECT_LOAD` 按 quad 使用 EXEC（非按像素）：若 quad 中任意像素有效，则数据写入该 quad 的所有 4 个像素。

`DS_DIRECT_LOAD` 使用 EXPcnt 跟踪完成情况。

`DS_DIRECT_LOAD` 使用与 `DS_PARAM_LOAD` 相同的指令格式和字段，详见像素参数插值章节。

```
LDS_addr = M0[15:0]（字节地址，必须 DWORD 对齐）
DataType = M0[18:16]
    0  无符号字节
    1  无符号短整型
    2  DWORD
    3  未使用
    4  有符号字节
    5  有符号短整型
    6,7  保留
```

示例：`DS_DIRECT_LOAD  V4`  // 从 M0[15:0] 中的 LDS 地址加载值到 V4

有符号字节和短整型数据在写入 VGPR 之前进行符号扩展到 32 位；无符号字节和短整型数据进行零扩展到 32 位。

## 12.5 数据共享索引与原子访问

数据共享单元可执行索引和原子数据共享操作。

索引和原子操作从 VGPR 为每个工作项提供唯一地址，并向 LDS 提供数据或从 LDS 返回每工作项的唯一数据到 VGPR。由于 LDS 内部的 bank 结构，操作可在最少一个时钟周期内完成，也可能因 bank 冲突（多个地址映射到同一 bank）而需要更多时钟周期。

索引操作是简单的 LDS 加载和存储操作，从 VGPR 读取数据或将数据返回到 VGPR。原子操作是算术操作，将 VGPR 数据与 LDS 中的数据合并，并将结果写回 LDS。原子操作可选择将 LDS 的"操作前"值返回到 VGPR。

LDS 索引和原子指令使用 DScnt 跟踪完成情况。DScnt 在每条指令发出时递增，在完成执行时递减。来自同一 wave 的 LDS 指令保持顺序执行。

**表 71. VDS 指令字段**

| 字段 | 大小（位） | 描述 |
|------|-----------|------|
| OP | 8 | LDS 操作码 |
| OFFSET0 | 8 | 立即地址偏移量。解释方式因操作码而异：<br>单地址指令：将两个偏移量字段合并为 16 位无符号字节偏移：{offset1, offset0}<br>双地址指令（如 {LOAD, STORE, XCHG}_2ADDR）：将两个偏移量分开使用，每个为 8 位无符号偏移。8/16/32 位数据时偏移量乘以 4；64 位数据时乘以 8 |
| OFFSET1 | 8 | 同上（第二地址偏移） |
| VDST | 8 | 写入结果的 VGPR：来自 LDS 加载或原子操作的返回值 |
| ADDR | 8 | 提供字节地址偏移的 VGPR |
| DATA0 | 8 | 提供第一个数据源的 VGPR |
| DATA1 | 8 | 提供第二个数据源的 VGPR |
| M0 | 16 | 无符号字节 Offset[15:0]，用于：`ds_load_addtid_b32`、`ds_write_addtid_b32` |

M0 寄存器不用于大多数 LDS 索引操作；只有"ADDTID"指令读取 M0，并将其解释为字节地址。

**表 72. LDS 索引加载/存储**

| 加载/存储指令 | 描述 |
|-------------|------|
| `DS_LOAD_{B32,B64,B96,B128,U8,I8,U16,I16}` | 将每线程一个值加载到 VGPR；有符号则进行符号扩展，无符号则进行零扩展 |
| `DS_LOAD_2ADDR_{B32,B64}` | 在两个唯一地址加载两个值 |
| `DS_LOAD_2ADDR_STRIDE64_{B32,B64}` | 在两个唯一地址加载 2 个值；偏移量 *= 64 |
| `DS_STORE_{B8,B16,B32,B64,B96,B128}` | 将一个值从 VGPR 存储到 LDS |
| `DS_STORE_2ADDR_{B32,B64}` | 存储两个值 |
| `DS_STORE_2ADDR_STRIDE64_{B32,B64}` | 存储两个值，偏移量 *= 64 |
| `DS_STOREXCHG_RTN_{B32,B64}` | 将 GPR 与 LDS 内存进行交换 |
| `DS_STOREXCHG_2ADDR_RTN_{B32,B64}` | 将两个独立 GPR 与 LDS 内存进行交换 |
| `DS_STOREXCHG_2ADDR_STRIDE64_RTN_{B32,B64}` | 将 GPR 与 LDS 内存进行交换；偏移量 *= 64 |
| "D16 操作"：加载操作只写 VGPR 的 16 位（低或高）；存储操作使用 VGPR 的 16 位 | |
| `DS_STORE_{B8, B16}_D16_HI` | 使用 VGPR 高 16 位存储 8 或 16 位 |
| `DS_LOAD_{U8, I8, U16}_D16` | 将无符号或有符号 8 或 16 位加载到 VGPR 低半部分 |
| `DS_LOAD_{U8, I8, U16}_D16_HI` | 将无符号或有符号 8 或 16 位加载到 VGPR 高半部分 |
| `DS_LOAD_ADDTID_B32` / `DS_STORE_ADDTID_B32` | 使用线程 ID 作为地址偏移的一部分进行 B32 加载和存储 |
| `DS_PERMUTE_B32` / `DS_BPERMUTE_B32` / `DS_BPERMUTE_FI_B32` | 前向和后向置换，不写入任何 LDS 内存（详见 LDS 通道置换操作） |
| `DS_SWIZZLE_B32` | DWORD 混洗；不向 LDS 写入任何数据 |
| `DS_CONSUME` | 从内存减去 countBits(EXEC)，返回操作前值 |
| `DS_APPEND` | 向内存加上 countBits(EXEC)，返回操作前值 |

**单地址指令**
```
LDS_Addr = LDS_BASE + VGPR[ADDR] + {OFFSET1, OFFSET0}
```

**双地址指令**
```
LDS_Addr0 = LDS_BASE + VGPR[ADDR] + OFFSET0 * ADJ
LDS_Addr1 = LDS_BASE + VGPR[ADDR] + OFFSET1 * ADJ
    其中 ADJ = 4（8、16、32 位数据类型）；ADJ = 8（64 位数据类型）
```

双地址指令包括：`LOAD_2ADDR*`、`STORE_2ADDR*` 和 `STOREXCHG_2ADDR_*`。地址来自 VGPR，VGPR[ADDR] 和 OFFSET 均为字节地址。在 wave 创建时，LDS_BASE 被赋值为该 wave 或工作组所拥有的物理 LDS 区域地址。

**DS_{LOAD,STORE}_ADDTID 寻址**
```
LDS_Addr = LDS_BASE + {OFFSET1, OFFSET0} + TID(0..63)*4 + M0
  注意：地址的任何部分都不来自 VGPR。M0 必须 DWORD 对齐。
```

"ADDTID"（添加线程 ID）是一种特殊形式，其基地址对所有线程相同，但每个线程根据其在 wave 内的线程 ID 添加固定偏移量。这提供了一种无需 VGPR 提供地址即可在 VGPR 和 LDS 之间高效传输数据的便捷方式。

### 12.5.1 LDS 原子操作

原子操作将 VGPR 中的数据与 LDS 中的数据合并，将结果写回 LDS 内存，并可选择将 LDS 内存中的"操作前"值返回到 VGPR。当 wave 中多个通道访问同一 LDS 位置时，各通道执行操作的顺序未指定，但保证每个通道在另一个通道操作该数据之前完成完整的读-改-写操作。

```
LDS_Addr0 = LDS_BASE + VGPR[ADDR] + {OFFSET1, OFFSET0}
```

VGPR[ADDR] 为字节地址。对于双精度数据，VGPR 0、1 和 dst 为双 GPR。VGPR 数据源只能是 VGPR 或常量值，不能是 SGPR。浮点原子操作使用 MODE 寄存器控制非规格化数的刷新行为。

**原子操作码**

指令字段：op、offset0、offset1、vdst、addr、data0、data1

| 32 位无返回值 | 32 位有返回值 | 64 位无返回值 | 64 位有返回值 |
|-------------|-------------|-------------|-------------|
| `ds_add_u32` | `ds_add_rtn_u32` | `ds_add_u64` | `ds_add_rtn_u64` |
| `ds_sub_u32` | `ds_sub_rtn_u32` | `ds_sub_u64` | `ds_rsub_rtn_u64` |
| `ds_rsub_u32` | `ds_rsub_rtn_u32` | `ds_rsub_u64` | `ds_rsub_rtn_u64` |
| `ds_inc_u32` | `ds_inc_rtn_u32` | `ds_inc_u64` | `ds_inc_rtn_u64` |
| `ds_dec_u32` | `ds_dec_rtn_u32` | `ds_dec_u64` | `ds_dec_rtn_u64` |
| `ds_min_{u32,i32}` | `ds_min_rtn_{u32,i32}` | `ds_min_{u64,i64}` | `ds_min_rtn_{u64,i64}` |
| `ds_max_{u32,i32}` | `ds_max_rtn_{u32,i32}` | `ds_max_{u64,i64}` | `ds_max_rtn_{u64,i64}` |
| `ds_min_num_f32` | `ds_min_num_rtn_f32` | `ds_min_num_f64` | `ds_min_num_rtn_f64` |
| `ds_max_num_f32` | `ds_max_num_rtn_f32` | `ds_max_num_f64` | `ds_max_num_rtn_f64` |
| `ds_and_b32` | `ds_and_rtn_b32` | `ds_and_b64` | `ds_and_rtn_b64` |
| `ds_or_b32` | `ds_or_rtn_b32` | `ds_or_b64` | `ds_or_rtn_b64` |
| `ds_xor_b32` | `ds_xor_rtn_b32` | `ds_xor_b64` | `ds_xor_rtn_b64` |
| `ds_mskor_b32` | `ds_mskor_rtn_b32` | `ds_mskor_b64` | `ds_mskor_rtn_b64` |
| `ds_cmpstore_b32` | `ds_cmpstore_rtn_b32` | `ds_cmpstore_b64` | `ds_cmpstore_rtn_b64` |
| `ds_add_f32` | `ds_add_rtn_f32` | | |
| `ds_pk_add_f16` | `ds_pk_add_rtn_f16` | | |
| `ds_pk_add_bf16` | `ds_pk_add_rtn_bf16` | | |
| `ds_storexchg_rtn_b32` | | `ds_storexchg_rtn_b64` | |
| `ds_storexchg_2addr_rtn_b32` | | `ds_storexchg_2addr_rtn_b64` | |
| `ds_storexchg_2addr_stride64_rtn_b32` | | `ds_storexchg_2addr_stride64_rtn_b64` | |
| | | `ds_condxchg32_rtn_b64` | |
| `ds_clampsub_u32` | `ds_clampsub_rtn_u32` | | |
| `ds_condsub_u32` | `ds_condsub_rtn_u32` | | |

### 12.5.2 LDS 通道置换操作

`DS_PERMUTE` 指令允许对 wave32 的 32 个通道或 wave64 的所有 64 个通道进行任意数据混洗。提供两个版本：前向（scatter，散播）和后向（gather，聚集）。

这些指令使用 LDS 硬件，但不使用任何内存存储，可被未分配任何 LDS 空间的 wave 使用。指令每通道提供一个数据值和一个索引值。

- `ds_permute_b32`：`Dst[index[0..31]] = src[0..31]`（[0..31] 为通道编号）
- `ds_bpermute_b32`：`Dst[0..31] = src[index[0..31]]`
- 对于 wave64，将"31"替换为"63"

对于读源和写目标，EXEC 掩码均被遵守，但 `DS_BPERMUTE_FI_B32` 例外，它读取所有通道，EXEC 只控制写入哪些通道。越界的索引值会回卷：对于 wave32 只使用索引的 [6:2] 位，其他位被忽略；对于 wave64 使用 [7:2] 位。读取禁用通道返回零。

在指令字中：VDST 为目标 VGPR，ADDR 为索引 VGPR，DATA0 为源数据 VGPR。索引值以字节为单位（需乘以 4），并在使用前加上"offset0"字段的值。

### 12.5.3 光线追踪的 DS 栈操作

这些 LDS 指令可用于管理 LDS 中每线程的浅栈，该栈用于光线追踪的 BVH（边界体层次）遍历。BVH 结构由盒节点和三角节点组成。对于给定光线（线程），一个盒节点最多有四个子节点指针可返回到着色器（VGPR）。遍历着色器每次迭代每条光线跟随一个指针，多余的指针被压入 LDS 中的每线程栈。注意：返回的指针是有序排列的。

这个"短栈"容量有限，超出容量时会回卷覆盖旧条目。当栈耗尽时，着色器切换到无栈模式，通过查询内存中的表来获取当前节点的父节点，并跟踪最后访问的地址以避免重复遍历子树。

```
DS_BVH_STACK_PUSH4_POP1_RTN_B32   压 4 出 1
    vgpr(dst), vgpr(stack_addr), vgpr(lvaddr), vgpr[4](data)

DS_BVH_STACK_PUSH8_POP1_RTN_B32   压 8 出 1
    dst, stack_addr, last_node_ptr, data[8], stack_size

DS_BVH_STACK_PUSH8_POP2_RTN_B64   压 8 出 1 并三角出栈
    dst[2], stack_addr, last_node_ptr, data[8], stack_size
```

**指令字段**

| 字段 | 大小（位） | 描述 |
|------|-----------|------|
| OP | 8 | DS_STORE_STACK |
| OFFSET0 | 8 | bits[4:0] 携带 StackSize |
| OFFSET1 | 8 | bit[1]：图元范围使能；bit[0]：三角对优化 |
| VDST | 9 | 结果地址的目标 VGPR（如 X 或栈顶）。必须为 0-255。返回下一个"LV addr" |
| ADDR | 8 | STACK_VGPR：既是源也是目标 VGPR，提供 LDS 栈地址，并写回更新后的地址。<br>stack_addr[31:18] = stack_base[15:2]：栈基地址（相对于已分配 LDS 空间）<br>stack_addr[17:16] = stack_size[1:0]：0=8 DWORD，1=16，2=32，3=64 DWORD/线程<br>stack_addr[15:0] = stack_index[15:0]（bits[1:0] 必须为零） |
| DATA0 | 9 | LVADDR：最后访问地址。与数据值（下一字段）比较以确定下一个要访问的节点。必须为 0-255 |
| DATA1 | 9 | 4 个 VGPR（X、Y、Z、W）。必须为 0-255 |
| M0 | 16 | 未使用 |

**地址字段**

| 字段 | 位 | 大小（位） | 类型 | 描述 |
|------|-----|-----------|------|------|
| valid_entries | 4:0 | 5 | uint | 栈上有效（未被覆盖）条目的数量 |
| entries_to_tlas | 9:5 | 5 | uint | 发生 BLAS→TLAS 转换前需要弹出的条目数量 |
| ring_addr | 14:10 | 5 | uint | 环形缓冲区中压入新条目或弹出条目的滚动地址 |
| stack_base_addr | 28:15 | 14 | uint | LDS 中该通道环形缓冲区存储的 DWORD 偏移量 |
| has_tlas_in_stack | 29 | 1 | boolean | 0：栈中所有有效条目来自同一 BVH；1：栈中部分有效条目来自父 BVH |
| has_overflowed | 30 | 1 | boolean | 0：滚动栈未丢失任何压入条目；1：部分压入条目已丢失（例如被覆盖） |
| blas_to_tlas_pop | 31 | 1 | boolean | 0：最后一次弹出与前一次来自同一 BVH；1：最后一次弹出来自前一次的父 BVH（需恢复父光线） |
