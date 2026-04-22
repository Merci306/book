# 第9章：向量内存缓冲区指令

向量内存（VM）缓冲区操作通过 L0 缓存在 VGPR 和内存中的缓冲区对象之间传输数据。"向量"意味着 wave 中每个线程都有唯一的一份或多份数据被传输，与标量内存加载不同——标量内存加载仅传输一个由 wave 中所有线程共享的值。

指令定义哪些 VGPR 提供操作地址、哪些 VGPR 提供或接收操作数据，以及包含内存缓冲区描述符（V#）的一组 SGPR。缓冲区原子操作可选择将操作前的内存值返回到 VGPR。

缓冲区对象的示例包括顶点缓冲区、原始缓冲区、流输出缓冲区和结构化缓冲区。

缓冲区对象同时支持同质和异质数据，但不支持对加载数据进行过滤。缓冲区指令分为两类：

**无类型缓冲区对象**
- 数据格式在资源常量中指定。
- 支持加载、存储、原子操作，可选是否进行数据格式转换。

**有类型缓冲区对象**
- 数据格式在指令中指定。
- 仅支持加载和存储，均带数据格式转换。

所有缓冲区操作都使用缓冲区资源常量（V#），这是 SGPR 中的一个 128 位值。该常量在指令执行时发送，定义了内存中缓冲区的地址和特性。通常，这些常量在执行 VM 指令之前通过标量内存加载从内存中取得，但也可在着色器内部生成。

不同类型的内存操作（加载、存储）可以乱序完成。

**缓冲区寻址简化视图**

下面的公式展示了缓冲区访问的内存地址计算方式。对于任何不允许未对齐访问的对齐模式，内存指令对未对齐访问返回 MEMVIOL。

## 9.1 缓冲区指令

缓冲区指令允许着色器程序从内存中的线性缓冲区加载数据或向其存储数据。这些操作可处理小至 1 字节、大至每个工作项 4 个 DWORD 的数据。原子操作从 VGPR 中获取数据，并与内存中已有的数据进行算术运算。操作前内存中的值可选择性地返回给着色器。

缓冲区操作的 D16 指令变体将结果与打包的 16 位值相互转换。例如，BUFFER_LOAD_D16_FORMAT_XYZW 使用两个 VGPR 存储 4 个 16 位值。

### 表46. 缓冲区指令

| 无类型缓冲区指令 | 说明 |
|-----------------|------|
| BUFFER_LOAD_\<size\> | 从无类型缓冲区对象加载，\<size\> = I8、U8、I16、U16、B32、B64、B96、B128 |
| BUFFER_STORE_\<size\> | 向无类型缓冲区对象存储 |
| BUFFER_LOAD_FORMAT_{x,xy,xyz,xyzw} | 带格式转换的加载 |
| BUFFER_STORE_FORMAT_{x,xy,xyz,xyzw} | 带格式转换的存储 |
| BUFFER_LOAD_D16_FORMAT_{x,xy,xyz,xyzw} | 带格式转换的 16 位加载 |
| BUFFER_STORE_D16_FORMAT_{x,xy,xyz,xyzw} | 带格式转换的 16 位存储 |
| BUFFER_LOAD_D16_FORMAT_X | 带格式转换的 16 位单分量加载 |
| BUFFER_STORE_D16_FORMAT_X | 带格式转换的 16 位单分量存储 |
| BUFFER_LOAD_D16_HI_FORMAT_X | 带格式转换的 16 位加载到 VGPR 高 16 位 |
| BUFFER_STORE_D16_HI_FORMAT_X | 带格式转换的从 VGPR 高 16 位存储 |
| BUFFER_ATOMIC_\<op\> | 缓冲区对象原子操作，自动全局一致，支持 32 位或 64 位值。若 TH==RET 则返回先前值。 |

| 有类型缓冲区指令 | 说明 |
|-----------------|------|
| TBUFFER_LOAD_FORMAT_{x,xy,xyz,xyzw} | 从有类型缓冲区对象加载 |
| TBUFFER_STORE_FORMAT_{x,xy,xyz,xyzw} | 向有类型缓冲区对象存储 |
| TBUFFER_LOAD_D16_FORMAT_{x,xy,xyz,xyzw} | 加载时将数据转换为 16 位后存入 VGPR |
| TBUFFER_STORE_D16_FORMAT_{x,xy,xyz,xyzw} | 将 16 位数据转换为纹理格式后存储到内存 |

### 表47. 微码格式

| 字段 | 位宽 | 描述 |
|------|------|------|
| OP | 8 | 操作码 |
| SOFFSET | 7 | 提供无符号字节偏移的 SGPR。可为 SGPR、M0 或 NULL。 |
| VADDR | 8 | 提供地址第一个分量（偏移或索引）的 VGPR 地址。当同时使用索引和偏移时，索引在第一个 VGPR，偏移在第二个。 |
| VDATA | 8 | 提供存储数据第一个分量或接收加载数据第一个分量的 VGPR 地址。 |
| IOFFSET | 24 | 有符号 24 位字节偏移，必须为非负数。 |
| RSRC | 9 | 指定在四个连续 SGPR 中提供 V#（资源常量）的 SGPR。必须是 4 的倍数，范围 0-120。 |
| FORMAT | 7 | 有类型缓冲区指令中内存缓冲区的数据格式。参见：缓冲区图像格式表。 |
| OFFEN | 1 | 1=从 VGPR（VADDR）提供偏移，0=不提供（偏移=0）。 |
| IDXEN | 1 | 1=从 VGPR（VADDR）提供索引，0=不提供（索引=0）。 |
| SCOPE | 2 | 内存作用域 |
| TH | 3 | 内存时序提示。对于原子操作，指示是否返回操作前的值。 |
| TFE | 1 | PRT（部分驻留纹理）纹素故障使能。设为 1 时，若取操作返回 NACK，状态将写入最后一个取目标 VGPR 之后的 VGPR。 |

TBUFFER*_FORMAT 指令（见下文）包含指令中指定的数据格式转换。指令指定内存中数据的格式，并将其展开至 VGPR 中每个分量 32 位，"D16"指令除外——D16 指令展开至每个分量 16 位。

### 表48. 有类型缓冲区指令

| 操作码 | 描述（缓冲区操作的所有地址分量均为无符号整数） |
|--------|----------------------------------------------|
| TBUFFER_LOAD_FORMAT_X | 带格式转换加载 X 分量 |
| TBUFFER_LOAD_FORMAT_XY | 带格式转换加载 XY 分量 |
| TBUFFER_LOAD_FORMAT_XYZ | 带格式转换加载 XYZ 分量 |
| TBUFFER_LOAD_FORMAT_XYZW | 带格式转换加载 XYZW 分量 |
| TBUFFER_STORE_FORMAT_X | 带格式转换存储 X 分量 |
| TBUFFER_STORE_FORMAT_XY | 带格式转换存储 XY 分量 |
| TBUFFER_STORE_FORMAT_XYZ | 带格式转换存储 XYZ 分量 |
| TBUFFER_STORE_FORMAT_XYZW | 带格式转换存储 XYZW 分量 |
| TBUFFER_LOAD_D16_FORMAT_X | 带格式转换加载 X 分量，16 位 |
| TBUFFER_LOAD_D16_FORMAT_XY | 带格式转换加载 XY 分量，16 位 |
| TBUFFER_LOAD_D16_FORMAT_XYZ | 带格式转换加载 XYZ 分量，16 位 |
| TBUFFER_LOAD_D16_FORMAT_XYZW | 带格式转换加载 XYZW 分量，16 位 |
| TBUFFER_STORE_D16_FORMAT_X | 带格式转换存储 X 分量，16 位 |
| TBUFFER_STORE_D16_FORMAT_XY | 带格式转换存储 XY 分量，16 位 |
| TBUFFER_STORE_D16_FORMAT_XYZ | 带格式转换存储 XYZ 分量，16 位 |
| TBUFFER_STORE_D16_FORMAT_XYZW | 带格式转换存储 XYZW 分量，16 位 |

BUFFER*_FORMAT 指令（见下文）包含资源常量（V#）中指定的数据格式转换。

下表中，"D16"表示 VGPR 中的数据为 16 位，而非通常的 32 位。"D16_HI"表示使用 VGPR 的高 16 位，而"D16"使用低 16 位。

### 表49. 无类型缓冲区指令

| 操作码 | 描述（所有地址分量均为无符号整数） |
|--------|---------------------------------|
| BUFFER_LOAD_U8 | 加载无符号字节（在 DWORD VGPR 的高位填 0） |
| BUFFER_LOAD_D16_U8 | 将无符号字节加载到 VGPR[15:0] |
| BUFFER_LOAD_D16_HI_U8 | 将无符号字节加载到 VGPR[31:16] |
| BUFFER_LOAD_I8 | 加载有符号字节（在 DWORD VGPR 的高位进行符号扩展） |
| BUFFER_LOAD_D16_I8 | 将有符号字节加载到 VGPR[15:0] |
| BUFFER_LOAD_D16_HI_I8 | 将有符号字节加载到 VGPR[31:16] |
| BUFFER_LOAD_U16 | 加载无符号短整数（在 DWORD VGPR 的高位填 0） |
| BUFFER_LOAD_I16 | 加载有符号短整数（在 DWORD VGPR 的高位进行符号扩展） |
| BUFFER_LOAD_D16_B16 | 将短整数加载到 VGPR[15:0] |
| BUFFER_LOAD_D16_HI_B16 | 将短整数加载到 VGPR[31:16] |
| BUFFER_LOAD_B32 | 加载 DWORD |
| BUFFER_LOAD_B64 | 每元素加载 2 个 DWORD |
| BUFFER_LOAD_B96 | 每元素加载 3 个 DWORD |
| BUFFER_LOAD_B128 | 每元素加载 4 个 DWORD |
| BUFFER_LOAD_FORMAT_X | 带格式转换加载 X 分量 |
| BUFFER_LOAD_FORMAT_XY | 带格式转换加载 XY 分量 |
| BUFFER_LOAD_FORMAT_XYZ | 带格式转换加载 XYZ 分量 |
| BUFFER_LOAD_FORMAT_XYZW | 带格式转换加载 XYZW 分量 |
| BUFFER_LOAD_D16_FORMAT_X | 带格式转换加载 X 分量，16 位 |
| BUFFER_LOAD_D16_HI_FORMAT_X | 带格式转换加载 X 分量，16 位，存入 VGPR 高 16 位 |
| BUFFER_LOAD_D16_FORMAT_XY | 带格式转换加载 XY 分量，16 位 |
| BUFFER_LOAD_D16_FORMAT_XYZ | 带格式转换加载 XYZ 分量，16 位 |
| BUFFER_LOAD_D16_FORMAT_XYZW | 带格式转换加载 XYZW 分量，16 位 |
| BUFFER_STORE_B8 | 存储字节（忽略 DWORD VGPR 的高位） |
| BUFFER_STORE_D16_HI_B8 | 从 VGPR 的 [23:16] 位存储字节 |
| BUFFER_STORE_B16 | 存储短整数（忽略 DWORD VGPR 的高位） |
| BUFFER_STORE_D16_HI_B16 | 从 VGPR 的 [31:16] 位存储短整数 |
| BUFFER_STORE_B32 | 存储 DWORD |
| BUFFER_STORE_B64 | 每元素存储 2 个 DWORD |
| BUFFER_STORE_B96 | 每元素存储 3 个 DWORD |
| BUFFER_STORE_B128 | 每元素存储 4 个 DWORD |
| BUFFER_STORE_FORMAT_X | 带格式转换存储 X 分量 |
| BUFFER_STORE_FORMAT_XY | 带格式转换存储 XY 分量 |
| BUFFER_STORE_FORMAT_XYZ | 带格式转换存储 XYZ 分量 |
| BUFFER_STORE_FORMAT_XYZW | 带格式转换存储 XYZW 分量 |
| BUFFER_STORE_D16_FORMAT_X | 带格式转换存储 X 分量，16 位 |
| BUFFER_STORE_D16_HI_FORMAT_X | 带格式转换从 VGPR 高 16 位存储 X 分量，16 位 |
| BUFFER_STORE_D16_FORMAT_XY | 带格式转换存储 XY 分量，16 位 |
| BUFFER_STORE_D16_FORMAT_XYZ | 带格式转换存储 XYZ 分量，16 位 |
| BUFFER_STORE_D16_FORMAT_XYZW | 带格式转换存储 XYZW 分量，16 位 |
| BUFFER_ATOMIC_ADD_U32 | 32 位，dst += src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_ADD_F32 | 32 位浮点，dst += src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_PK_ADD_F16 | 32 位打包，dst[15:0] += src[15:0]; dst[31:16] += src[31:16]，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_ADD_U64 | 64 位，dst += src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_AND_B32 | 32 位，dst &= src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_AND_B64 | 64 位，dst &= src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_CMPSWAP_B32 | 32 位，dst = (dst == cmp) ? src : dst，若 TH==RET 则返回先前值。src 来自 vdata，cmp 来自 vdata+1 |
| BUFFER_ATOMIC_CMPSWAP_B64 | 64 位，dst = (dst == cmp) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_DEC_U32 | 32 位，dst = ((dst == 0) \| (dst > src)) ? src : dst-1，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_DEC_U64 | 64 位，同上 |
| BUFFER_ATOMIC_MAX_NUM_F32 | 32 位浮点，dst = (src > dst) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_MIN_NUM_F32 | 32 位浮点，dst = (src < dst) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_INC_U32 | 32 位，dst = (dst >= src) ? 0 : dst+1，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_INC_U64 | 64 位，同上 |
| BUFFER_ATOMIC_OR_B32 | 32 位，dst \|= src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_OR_B64 | 64 位，同上 |
| BUFFER_ATOMIC_MAX_I32 | 32 位有符号，dst = (src > dst) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_MAX_I64 | 64 位有符号，同上 |
| BUFFER_ATOMIC_MIN_I32 | 32 位有符号，dst = (src < dst) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_MIN_I64 | 64 位有符号，同上 |
| BUFFER_ATOMIC_SUB_U32 | 32 位，dst -= src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_SUB_U64 | 64 位，同上 |
| BUFFER_ATOMIC_SWAP_B32 | 32 位，dst = src，若 TH==RET 则返回 dst 的先前值 |
| BUFFER_ATOMIC_SWAP_B64 | 64 位，同上 |
| BUFFER_ATOMIC_MAX_U32 | 32 位无符号，dst = (src > dst) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_MAX_U64 | 64 位无符号，同上 |
| BUFFER_ATOMIC_MIN_U32 | 32 位无符号，dst = (src < dst) ? src : dst，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_MIN_U64 | 64 位无符号，同上 |
| BUFFER_ATOMIC_XOR_B32 | 32 位，dst ^= src，若 TH==RET 则返回先前值 |
| BUFFER_ATOMIC_XOR_B64 | 64 位，同上 |
| BUFFER_ATOMIC_CSUB_U32 | 32 位，dst = (src > dst) ? 0 : dst - src，TH 必须设为 RET，返回先前值 |
| BUFFER_ATOMIC_CONDSUB_U32 | 32 位，dst = (src > dst) ? dst : dst - src，返回先前值 |

## 9.2 VGPR 使用

VGPR 提供地址和存储数据，也可作为返回数据的目标。

**地址**

根据指令字中的索引使能（IDXEN）和偏移使能（OFFEN），使用零个、一个或两个 VGPR。这些均为无符号整数。对于 64 位地址，低位在 VGPRn，高位在 VGPRn+1。

### 表50. 地址 VGPR

| IDXEN | OFFEN | VGPRn | VGPRn+1 |
|-------|-------|-------|---------|
| 0 | 0 | 无 | — |
| 0 | 1 | uint offset | — |
| 1 | 0 | uint index | — |
| 1 | 1 | uint index | uint offset |

**存储数据**：N 个连续 VGPR，从 VDATA 开始。指令字操作码和 D16 设置中指定的数据格式决定着色器提供多少个 DWORD 用于存储。

**加载数据**：与存储相同。数据返回到连续 VGPR。

**加载数据格式**：加载数据为 32 位或 16 位，由指令或资源中的数据格式以及 D16 决定。浮点数或归一化数据以浮点数返回；整数格式以整数返回（有符号或无符号，与内存存储格式相同）。内存中 32 位或 64 位数据的加载，若 D16 设为 1，则以 16 位返回，否则不进行格式转换。

**带返回的原子操作**：从 VDATA 开始的 VGPR 中读取数据，用于原子操作。若原子操作将值返回给 VGPR，该数据将返回到从 VDATA 开始的相同 VGPR。

### 表51. VGPR 和内存中的数据格式

**加载指令**

| 指令 | 内存格式 | VGPR 格式 | 备注 |
|------|---------|-----------|------|
| BUFFER_LOAD_U8 | ubyte | V0[31:0] = {24'b0, byte} | — |
| BUFFER_LOAD_D16_U8 | ubyte | V0[15:0] = {8'b0, byte} | 仅写入 16 位 |
| BUFFER_LOAD_D16_HI_U8 | ubyte | V0[31:16] = {8'h0, byte} | 仅写入 16 位 |
| BUFFER_LOAD_S8 | sbyte | V0[31:0] = { 24{sign}, byte} | — |
| BUFFER_LOAD_D16_S8 | sbyte | V0[15:0] = {8{sign}, byte} | 仅写入 16 位 |
| BUFFER_LOAD_D16_HI_S8 | sbyte | V0[31:16] = {8{sign}, byte} | 仅写入 16 位 |
| BUFFER_LOAD_U16 | uint16 | V0[31:0] = { 16'b0, short} | — |
| BUFFER_LOAD_S16 | int16 | V0[31:0] = { 16{sign}, short} | — |
| BUFFER_LOAD_D16_B16 | short | V0[15:0] = short | 仅写入 16 位 |
| BUFFER_LOAD_D16_HI_B16 | short | V0[31:16] = short | 仅写入 16 位 |
| BUFFER_LOAD_B32 | DWORD | DWORD | — |
| BUFFER_LOAD_FORMAT_X | FORMAT 字段 | float、uint 或 sint | 将 X 加载到 V0[31:0]；VGPR 中数据类型基于 FORMAT 字段（D16_X 和 D16_HI_X 仅写入 16 位） |
| BUFFER_LOAD_FORMAT_XY | FORMAT 字段 | float、uint 或 sint | 将 X、Y 加载到 V0[31:0]、V1[31:0] |
| BUFFER_LOAD_FORMAT_XYZ | FORMAT 字段 | float、uint 或 sint | 将 X、Y、Z 加载到 V0[31:0]、V1[31:0]、V2[31:0] |
| BUFFER_LOAD_FORMAT_XYZW | FORMAT 字段 | float、uint 或 sint | 将 X、Y、Z、W 加载到 V0[31:0]、V1[31:0]、V2[31:0]、V3[31:0] |
| BUFFER_LOAD_D16_FORMAT_X | FORMAT 字段 | float、uint 或 sint | 将 X 加载到 V0[15:0] |
| BUFFER_LOAD_D16_HI_FORMAT_X | FORMAT 字段 | float、uint16 或 int16 | 将 X 加载到 V0[31:16] |
| BUFFER_LOAD_D16_FORMAT_XY | FORMAT 字段 | float、uint16 或 int16 | 将 X、Y 加载到 V0[15:0]、V0[31:16] |
| BUFFER_LOAD_D16_FORMAT_XYZ | FORMAT 字段 | float、uint16 或 int16 | 将 X、Y、Z 加载到 V0[15:0]、V0[31:16]、V1[15:0] |
| BUFFER_LOAD_D16_FORMAT_XYZW | FORMAT 字段 | float、uint16 或 int16 | 将 X、Y、Z、W 加载到 V0[15:0]、V0[31:16]、V1[15:0]、V1[31:16] |

其中"V0"为 VDATA VGPR；V1 为 VDATA+1 VGPR，以此类推。

**存储指令**

| 指令 | VGPR 格式 | 内存格式 | 备注 |
|------|-----------|---------|------|
| BUFFER_STORE_B8 | [7:0] 字节 | byte | — |
| BUFFER_STORE_D16_HI_B8 | [23:16] 字节 | byte | — |
| BUFFER_STORE_B16 | [15:0] 短整数 | short | — |
| BUFFER_STORE_D16_HI_B16 | [31:16] 短整数 | short | — |
| BUFFER_STORE_B32 | [31:0] 数据 | DWORD | — |
| BUFFER_STORE_FORMAT_X | float、uint 或 sint，数据在 V0[31:0] | FORMAT 字段 | VGPR 中数据类型基于 FORMAT 字段 |
| BUFFER_STORE_D16_FORMAT_X | float、uint16 或 int16，数据在 V0[15:0] | — | — |
| BUFFER_STORE_D16_FORMAT_XY | float、uint16 或 int16，数据在 V0[15:0]、V0[31:16] | — | — |
| BUFFER_STORE_D16_FORMAT_XYZ | float、uint16 或 int16，数据在 V0[15:0]、V0[31:16]、V1[15:0] | — | — |
| BUFFER_STORE_D16_FORMAT_XYZW | float、uint16 或 int16，数据在 V0[15:0]、V0[31:16]、V1[15:0]、V1[31:16] | — | — |
| BUFFER_STORE_D16_HI_FORMAT_X | float、uint16 或 int16，数据在 V0[31:16] | — | — |

## 9.3 缓冲区数据

加载或存储的数据量和类型由以下因素控制：资源格式字段、目标分量选择（dst_sel）和操作码。

数据格式可来自资源、指令字段或操作码本身。有类型缓冲区操作从指令的 FORMAT 字段获取数据格式；无类型"format"指令从资源获取 FORMAT；其他缓冲区操作码从指令本身获取数据格式。DST_SEL 来自资源，但对许多操作会被忽略。

### 表52. 缓冲区指令数据来源

| 指令 | 数据格式 | DST SEL |
|------|---------|---------|
| TBUFFER_LOAD_FORMAT_* | 指令 | identity |
| TBUFFER_STORE_FORMAT_* | 指令 | identity |
| BUFFER_LOAD_FORMAT_* | 资源 | 资源 |
| BUFFER_STORE_FORMAT_* | 资源 | 资源 |
| BUFFER_LOAD_\<type\> | 派生 | identity |
| BUFFER_STORE_\<type\> | 派生 | identity |
| BUFFER_ATOMIC_* | 派生 | identity |

- **指令**：使用指令的格式字段，而非资源字段。
- **数据格式派生**：数据格式从操作码推导，忽略资源定义。例如，BUFFER_LOAD_U8 将数据格式设置为 uint-8。

> 资源的数据格式不得为 INVALID；该格式具有特定含义（未绑定资源），此时数据格式不会被指令隐含的数据格式替代。

- **DST_SEL identity**：根据数据格式中的分量数，分别为：X000、XY00、XYZ0 或 XYZW。

当着色器提供的分量数少于表面格式所需数量时，第一个分量将被复制填充缺失分量。例如，若表面有 4 个分量，但着色器只提供 X 和 Y，则写入结果为 XYXX。

### 9.3.1 D16 指令

加载和存储指令也有"D16"变体。D16 缓冲区指令允许着色器程序在 VGPR 和内存之间每个工作项只加载或存储 16 位数据。对于存储，每个 32 位 VGPR 持有两个 16 位数据元素，这些数据被传递给纹理单元，纹理单元在写入内存之前将其转换为缓冲区格式。对于加载，纹理单元返回的数据被转换为 16 位，每对数据存储在一个 32 位 VGPR 中（先低位，后高位）。整数与浮点的控制由 FORMAT 决定。

float32 到 float16 的转换使用截断；其他输入数据格式使用最近偶数舍入。

这些指令有两种变体：
- **D16**：将数据加载到或从 VGPR 的低 16 位存储数据。
- **D16_HI**：将数据加载到或从 VGPR 的高 16 位存储数据。

例如，BUFFER_LOAD_D16_U8 从内存中每个工作项加载一个字节，将其转换为 16 位整数，然后加载到数据 VGPR 的低 16 位。

### 9.3.2 LOAD/STORE_FORMAT 与数据格式不匹配

"format"指令指定元素数量（x、xy、xyz 或 xyzw），这与指令或资源数据格式字段中指定的元素数量可能不匹配。发生不匹配时：

- `buffer_load_format_x` 且 dfmt 为 "32_32_32_32"：从内存加载 4 个 DWORD，但只将第一个加载到着色器。
- `buffer_store_format_x` 且 dfmt 为 "32_32_32_32"：若 "x" 从着色器发出，则将其存储到内存中所有非常量通道；否则存储 0。
- `buffer_load_format_xyzw` 且 dfmt 为 "32"：从内存加载 1 个 DWORD，通过 dst_sel 向着色器返回 4 个值。
- `buffer_store_format_xyzw` 且 dfmt 为 "32"：将 1 个 DWORD（X）存储到内存，忽略 YZW。

## 9.4 缓冲区寻址

缓冲区是内存中以索引和偏移寻址的数据结构。索引指向大小为 stride 字节的特定记录，偏移是记录内的字节偏移。stride 来自资源，索引来自 VGPR（或为零），偏移来自 SGPR 或 VGPR，也来自指令本身。

### 表53. 缓冲区指令寻址字段

| 字段 | 位宽 | 描述 |
|------|------|------|
| IOFFSET | 24 | 来自指令的字面字节偏移。 |
| IDXEN | 1 | 布尔值：为 true 时从 VGPR 获取每通道索引，为 false 时无索引。 |
| OFFEN | 1 | 布尔值：为 true 时从 VGPR 获取每通道偏移，为 false 时无偏移。注意 IOFFSET 无论此位如何都存在。 |

缓冲区指令的"元素大小"是指令传输的数据量。对于有类型缓冲区指令，由 FORMAT 字段决定；对于无类型缓冲区指令，由操作码决定，可为 1、2、4、8、12 或 16 字节。例如，格式"16_16"的元素大小为 4 字节。

### 表54. 缓冲区资源常量寻址字段

| 字段 | 位宽 | 描述 |
|------|------|------|
| const_base | 48 | 缓冲区资源的基地址，以字节为单位。 |
| const_stride | 14 | 记录步幅，以字节为单位，然后乘以 stride_scale。 |
| const_num_records | 32 | 缓冲区中的记录数。若 const_stride ≤ 1，单位为字节；否则以"stride"为单位。 |
| const_add_tid_enable | 1 | 布尔值。为 true 时将 wave 内的线程 ID 加入索引。 |
| const_swizzle_enable | 2 | 根据 stride、index_stride 和 element_size 对 AOS（结构数组）进行 Swizzle：0=禁用，1=以 4 字节元素大小启用，2=保留，3=以 16 字节元素大小启用。 |
| const_index_stride | 2 | 仅在 const_swizzle_en=true 时使用。单个元素（const_element_size=4 或 16 字节）切换到下一元素前的连续索引数：8、16、32 或 64 个索引。 |

### 表55. 来自 GPR 的地址分量

| 字段 | 位宽 | 描述 |
|------|------|------|
| SGPR_offset | 32 | 无符号字节偏移，来自 SGPR 或 M0。 |
| VGPR_offset | 32 | 可选的无符号字节偏移，每线程，来自 VGPR。 |
| VGPR_index | 32 | 可选的索引值，每线程，来自 VGPR。 |

最终缓冲区内存地址由三部分组成：
- 来自缓冲区资源（V#）的基地址
- 来自 SGPR 的偏移
- 缓冲区偏移，其计算方式取决于缓冲区是线性寻址（简单的结构数组计算）还是 Swizzled

**线性缓冲区地址计算**

### 9.4.1 范围检查

缓冲区地址会与内存缓冲区的大小进行检查。超出范围的加载返回零，超出范围的存储和原子操作被丢弃。对于大于一个 DWORD 的非格式化加载和存储，范围检查针对每个分量进行。load/store_B64、B96 和 B128 被视为"2/3/4 DWORD 加载/存储"，每个 DWORD 单独进行边界检查。夹紧方式由缓冲区资源中的 2 位字段 OOB_SELECT（越界选择）控制。

下表中 Payload 为指令传输的字节数。

### 表56. 缓冲区越界选择

| OOB_SELECT | 越界条件 | 描述或用途 |
|-----------|---------|-----------|
| 0 | (index >= NumRecords) \|\| (offset+payload > stride) | 结构化缓冲区 |
| 1 | (index >= NumRecords) | 原始缓冲区 |
| 2 | (NumRecords == 0) | 不检查边界（空缓冲区除外） |
| 3 | 若 swizzle_en 且 const_stride != 0：OOB = (index >= NumRecords \|\| (offset+payload > stride))；否则 OOB = (offset+payload > NumRecords)。在此模式下，"num_records"减去"sgpr_offset"。 | 原始（Raw）模式 |

注：
1. 超出范围的加载返回零（V#.dst_sel = SEL_1 的分量除外，返回 1）。
2. 超出范围的存储不写入任何数据。
3. Load/store-format-* 指令和原子操作进行"全部或全不"的范围检查——要么完全在范围内，要么完全在范围外。
4. Load/store-B{64,96,128} 针对每个分量进行范围检查。

对于有类型缓冲区，若线程的任何分量越界，则整个线程视为越界并返回零。对于无类型缓冲区，只有越界的分量返回零。

#### 9.4.1.1 结构化缓冲区

swizzle_en==0 时的地址计算（未 swizzled 的结构化缓冲区）：

```
ADDR = Base  + baseOff + Ioff +  Stride * Vidx  + (OffEn ? Voff : 0)
       V#      SGPR      INST      V#     VGPR     INST    VGPR
```

结构化缓冲区的 NumRecords 以 stride 为单位。

#### 9.4.1.2 原始缓冲区

```
ADDR = Base  + baseOff + Ioff +  (OffEn ? Voff : 0)
       V#      SGPR      INST      INST   VGPR
```

原始缓冲区的 NumRecords 以字节为单位。这是精确的范围检查，包括 payload 大小，可正确处理多 DWORD 和未对齐的情况。stride 字段被忽略。

#### 9.4.1.3 Scratch 缓冲区

swizzle_en=0 时的地址计算（未 swizzled 的 scratch 缓冲区）：

```
ADDR = Base  + baseOffset + Ioff +  Stride * TID +  (OffEn ? Voff : 0)
       V#      SGPR          INST      V#    0..63    INST    VGPR
```

Scratch 缓冲区也支持 Swizzle（且通常使用）。TID 的高位（TID / 64）被折叠进 baseOffset。不进行范围检查（使用 OOB 模式 2）。

#### 9.4.1.4 标量内存

标量内存适用于 RAW 缓冲区和未 swizzled 的结构化缓冲区：

```
Addr = Base  +   offset
       V#       SGPR or Inst
```

越界条件：`offset >= ((stride==0 ? 1 : stride) * num_records)`

注：
1. 超出范围的加载返回零。超出范围的存储不写入任何数据。
2. 原子操作和 Load/store-format-* 指令进行"全部或全不"的范围检查。
3. Load/store-DWORD-x{2,3,4} 针对每个分量进行范围检查。

### 9.4.2 Swizzled 缓冲区寻址

Swizzled 寻址重排缓冲区中的数据，可以提升结构数组的缓存局部性。单次取操作的数据单元不得大于 const_element_size。缓冲区的 STRIDE 必须是 const_element_size 的倍数。

const_element_size 为 4 或 16 字节，取决于 V#.swizzle_enable 的设置。

```
Index        = (IDXEN ? vgpr_index : 0) + (const_add_tid_enable ? thread_id[5:0] : 0)
index_msb      = index / const_index_stride
index_lsb      = index % const_index_stride
total_offset   = ((OFFEN ? vgpr_offset : 0) + IOFFSET + sgpr_offset) & 32'hffffffff
offset_msb     = total_offset / const_element_size
offset_lsb     = total_offset % const_element_size
buffer_offset  = (index_msb * const_stride + offset_msb * const_element_size) * const_index_stride +
                  index_lsb * const_element_size + offset_lsb
Final Address = const_base + buffer_offset
```

Swizzled 缓冲区的限制和行为：
- const_swizzle_en==3（16 字节）时不支持 16 字节元素。
- 对于仅 DWORD 对齐的 16 字节元素，高位 DWORD 的地址从第一个 DWORD 的地址线性偏移。

## 9.5 对齐

格式化操作（如 BUFFER_LOAD_FORMAT_*）的对齐要求如下：
- 1 字节格式需要 1 字节对齐
- 2 字节格式需要 2 字节对齐
- 4 字节及更大格式需要 4 字节对齐

非格式化操作的内存对齐要求由配置寄存器 `SH_MEM_CONFIG.alignment_mode` 控制。

原子操作必须按数据大小对齐，否则触发 MEMVIOL。

对齐模式选项：
- **0: DWORD** — 硬件自动将请求对齐到元素大小和 DWORD 中较小的那个。对于 DWORD 或更大的非格式化操作（如 BUFFER_LOAD_DWORD），字节地址的两个 LSB 被忽略，从而强制 DWORD 对齐。对于原子操作，通过忽略 LSB 强制所需对齐，这意味着 8 字节原子操作被强制到比 DWORD 更大的对齐。
- **1: DWORD_STRICT** — 必须对齐到元素大小和 DWORD 中较小的那个。
- **2: STRICT** — 访问必须对齐到数据大小。
- **3: UNALIGNED** — 允许任意对齐（但原子操作仍必须对齐）。

## 9.6 缓冲区资源

缓冲区资源（V#）描述内存中缓冲区的位置和数据格式。它由四个连续 SGPR（按 4 SGPR 对齐）指定，并随每条缓冲区指令一起发送到纹理缓存。

下表详细描述了构成缓冲区资源描述符的各字段。

### 表57. 缓冲区资源描述符

| 位 | 位宽 | 名称 | 描述 |
|----|------|------|------|
| 47:0 | 48 | 基地址 | 字节地址。 |
| 61:48 | 14 | Stride | 0 到 16383 字节（经 Stride Scale 修改）。 |
| 63:62 | 2 | Swizzle 使能 | 根据 stride、index_stride 和 element_size 对 AOS 进行 Swizzle，否则为线性。0=禁用；1=以 4 字节元素大小启用；2=保留；3=以 16 字节元素大小启用。 |
| 95:64 | 32 | Num_records | 若 stride ≥ 1，以 stride 为单位；否则以字节为单位。 |
| 98:96 | 3 | Dst_sel_x | 目标通道选择：0=0，1=1，4=R，5=G，6=B，7=A |
| 101:99 | 3 | Dst_sel_y | 同上 |
| 104:102 | 3 | Dst_sel_z | 同上 |
| 107:105 | 3 | Dst_sel_w | 同上 |
| 113:108 | 6 | Format | 内存数据类型。仅用于无类型缓冲区"FORMAT"指令。 |
| 115:114 | 2 | Stride Scale | stride 字段的乘数：0=×1；1=×4；2=×8；3=×32。 |
| 118:117 | 2 | Index stride | 0=8，1=16，2=32，3=64。用于 swizzled 缓冲区寻址。 |
| 119 | 1 | Add tid enable | 将线程 ID 加入索引以计算地址。 |
| 120 | 1 | 写压缩使能 | 1=启用写压缩，0=禁用。 |
| 121 | 1 | 压缩使能 | 0=绕过压缩（资源不可压缩）；1=不绕过压缩。 |
| 123:122 | 2 | 压缩访问模式 | 0=正常；1=强制现有数据压缩；2=压缩数据访问；3=元数据访问。 |
| 125:124 | 2 | OOB_SELECT | 越界选择。 |
| 127:126 | 2 | Type | 缓冲区值为 0。与 128 位 V# 资源中四位 TYPE 字段的高两位重叠。 |

**未绑定资源**

通过将"数据格式"设为零（INVALID）来标识，对于无类型缓冲区还需 add_tid_en=false。

**资源-指令不匹配**

若资源类型与指令不匹配（例如缓冲区资源配合图像指令，或图像资源配合缓冲区指令），该指令将被忽略（加载不返回任何数据，存储不改变内存）。
