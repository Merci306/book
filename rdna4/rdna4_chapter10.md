# 第10章：向量内存图像指令

向量内存（VMEM）图像操作通过纹理缓存在 VGPR 和内存之间传输数据。图像操作支持访问图像对象，例如纹理贴图和有类型表面。图像采样操作从表面读取多个元素并将其合并，每个通道产生一个结果。

图像对象使用一到四维地址进行访问；它们由同质采样点组成，每个采样点包含一到四个元素。这些图像对象通过 IMAGE_* 或 SAMPLE_* 指令读写，这些指令均使用 VIMAGE 或 VSAMPLE 指令格式。

IMAGE_LOAD 指令将图像缓冲区中的元素直接加载到 VGPR；SAMPLE 指令使用采样器常量（S#）并在读取数据后应用过滤。IMAGE_ATOMIC 指令将 VGPR 中的数据与内存中已有数据进行运算，并可选择性地返回操作前内存中的值。

VMEM 图像操作使用图像资源常量（T#），这是 SGPR 中的 128 位或 256 位值。该常量在指令执行时发送到纹理缓存，定义了内存中表面的地址、数据格式和特性。某些图像指令还使用 SGPR 中的 128 位采样器常量。通常，这些常量在执行 VM 指令之前通过标量内存加载从内存中取得，但也可在着色器内部生成。

纹理取操作有数据掩码（DMASK）字段。DMASK 指定返回多少个数据分量。若 DMASK 小于纹理中的分量数，纹理单元仅发送 DMASK 个分量，顺序为 R、G、B、A。若 DMASK 指定的数量超过纹理格式的分量数，着色器基于 T#.DST_SEL 接收缺失分量的数据。图像操作不生成 MemViol——若地址超出范围，则应用夹紧模式。

不同类型的内存操作（如加载、存储和采样）可以乱序完成。

## 10.1 图像指令

本节描述图像指令集及其可用的微码字段。图像指令根据指令编码分为两类：VIMAGE 和 VSAMPLE。

**VIMAGE 指令**

| 指令 | 说明 |
|------|------|
| IMAGE_LOAD_{-, PCK, PCK_SGN} | 从图像对象加载数据 |
| IMAGE_LOAD_MIP_{-, PCK, PCK_SGN} | 从图像对象的指定 mip 级别加载数据 |
| IMAGE_STORE | 向图像对象存储数据 |
| IMAGE_STORE_PCK | 向图像对象存储数据（压缩） |
| IMAGE_STORE_MIP | 向图像对象的特定 mipmap 级别存储数据 |
| IMAGE_STORE_MIP_PCK | 向图像对象的特定 mipmap 级别存储数据（压缩） |
| IMAGE_ATOMIC_SWAP | 图像原子交换 |
| IMAGE_ATOMIC_CMPSWAP | 图像原子比较交换 |
| IMAGE_ATOMIC_ADD | 图像原子加 |
| IMAGE_ATOMIC_SUB | 图像原子减 |
| IMAGE_ATOMIC_SMIN | 图像原子有符号最小值 |
| IMAGE_ATOMIC_UMIN | 图像原子无符号最小值 |
| IMAGE_ATOMIC_SMAX | 图像原子有符号最大值 |
| IMAGE_ATOMIC_UMAX | 图像原子无符号最大值 |
| IMAGE_ATOMIC_AND | 图像原子与 |
| IMAGE_ATOMIC_OR | 图像原子或 |
| IMAGE_ATOMIC_XOR | 图像原子异或 |
| IMAGE_ATOMIC_INC | 图像原子自增 |
| IMAGE_ATOMIC_DEC | 图像原子自减 |
| IMAGE_ATOMIC_ADD_F32 | 图像原子浮点加 |
| IMAGE_ATOMIC_MIN_F32 | 图像原子浮点最小值 |
| IMAGE_ATOMIC_MAX_F32 | 图像原子浮点最大值 |
| IMAGE_ATOMIC_PK_ADD_F16 | 图像原子打包 F16 加 |
| IMAGE_ATOMIC_PK_ADD_BF16 | 图像原子打包 BF16 加 |
| IMAGE_GET_RESINFO | 返回指定 MIP 级别的资源信息到 4 个 VGPR，均为 32 位整数：VDATA3-0 = {#mipLevels, depth, height, width}。对于 cubemap，depth = 6 × Number_of_array_faces。（DX 期望 cube 数量，但实际获取的是 face 数量） |
| BVH ops | 所有光线追踪 BVH 指令。参见：光线追踪指令章节。 |

**VSAMPLE 指令**

| 指令 | 说明 |
|------|------|
| IMAGE_SAMPLE_* | 从图像对象加载并过滤数据 |
| IMAGE_SAMPLE_*_G16 | 使用 16 位梯度进行采样 |
| IMAGE_GATHER4_* | 从 4 个纹素加载并返回采样值，用于软件过滤。返回单个分量，从左下纹素开始，按逆时针顺序。 |
| IMAGE_GATHER4H | 4H：从 4×1 纹素中每个纹素取 1 个分量。"DMASK"选择要加载的分量（R、G、B、A），且只能有一个位设为 1。 |
| IMAGE_MSAA_LOAD | 使用用户指定的片段 ID，从 MSAA 资源加载最多 4 个采样的 1 个分量。使用 DMASK 作为分量选择——行为类似 gather4 操作并返回 4 个 VGPR（D16=1 时为 2 个）。SAMP 应设为 NULL，因为此操作不使用采样器。 |
| IMAGE_GET_LOD | 返回计算得到的 LOD。视为采样指令。将"原始"LOD 和"夹紧"LOD 以两个 32 位浮点数返回到 VDATA：第一个 VGPR = clampLOD，第二个 VGPR = rawLOD。 |

### 表58. 指令字段

| 字段 | 位宽 | 描述 |
|------|------|------|
| OP | 8 | 操作码 |
| DIM | 3 | 表面维度：0=1D，1=2D，2=3D，3=cube，4=1D数组，5=2D数组，6=2D MSAA，7=2D MSAA数组 |
| DMASK | 4 | 数据 VGPR 使能掩码：1..4 个连续 VGPR。加载时定义返回哪些分量（0=红，1=绿，2=蓝，3=透明度）；存储时定义将哪些分量写入数据（缺失分量取 X 分量的值）。已启用的分量来自连续 VGPR。例：DMASK=1001，红色在 VGPRn，透明度在 VGPRn+1。对于 D16 加载，DMASK 指示返回哪些分量；对于 D16 存储，DMASK 掩码指示存储哪些分量，但有限制：数据从连续 VGPR 依次读出，与 DMASK 位位置无关，只与位数有关，位的位置控制在内存中写入哪个分量。若 DMASK==0，TA 强制 DMASK=1 并在 VGPR 中写零，若存在 LWE 状态也随之写入；不生成 TFE 状态（取操作被丢弃）。对于 IMAGE_GATHER4* 指令，DMASK 指示分量（RGBA），使用的 VGPR 数量由硬件自动确定（D16=0 时为 4 个 VGPR，D16=1 时为 2 个 VGPR）。 |
| R128 | 1 | 纹理资源大小：1=128 位，0=256 位。 |
| A16 | 1 | 地址分量为 16 位（而非通常的 32 位）。设置时，所有地址分量均为 16 位（每 DWORD 打包两个），但以下情况除外：纹素偏移（3 个 6 位无符号整数打包成 1 DWORD），PCF 参考（用于"_C"指令）。对于无采样器的图像操作，地址分量为 16 位无符号整数；有采样器时为 16 位浮点数。 |
| D16 | 1 | VGPR 数据 16 位。加载时，将内存中的数据转换为 16 位格式后存入 VGPR；存储时，将 VGPR 中的 16 位数据转换为内存格式后写入内存。数据被视为浮点数还是整数由 NFMT 决定。仅允许用于以下操作码：IMAGE_SAMPLE*、IMAGE_GATHER4、IMAGE_MSAA_LOAD、IMAGE_LOAD、IMAGE_LOAD_MIP、IMAGE_STORE、IMAGE_STORE_MIP。 |
| VDATA | 8 | 提供存储数据第一个分量或接收加载数据第一个分量的 VGPR 地址，范围 0-255。 |
| RSRC | 9 | 指定在四个连续 SGPR 中提供 T#（资源常量）的 SGPR，必须是 4 的倍数，范围 0-120。 |
| SCOPE | 2 | 内存作用域 |
| TH | 3 | 内存时序提示 |
| TFE | 1 | PRT（部分驻留纹理）纹素故障使能。设置时，取操作可能返回一个额外 VGPR 写入（DST+1），其中包含 NACK 信息。 |

**仅 VIMAGE 可用的字段**

| 字段 | 位宽 | 描述 |
|------|------|------|
| VADDR0-4 | 5×8 | 五个 8 位 VGPR 地址字段，每个提供一个或一组连续地址 VGPR。 |

**仅 VSAMPLE 可用的字段**

| 字段 | 位宽 | 描述 |
|------|------|------|
| VADDR0-3 | 4×8 | 四个 8 位 VGPR 地址字段，每个提供一个或一组连续地址 VGPR。 |
| SAMP | 9 | 指定在四个连续 SGPR 中提供 S#（采样器常量）的 SGPR，必须是 4 的倍数，范围 0-120。 |
| UNRM | 1 | 强制地址为非归一化。图像存储和原子操作必须设为 1。0：对于带采样器的图像操作，S、T、R 在 [0.0, 1.0] 范围内覆盖整个纹理贴图；1：对于带采样器的图像操作，S、T、R 在 [0.0, N] 范围内覆盖纹理贴图（N 为宽度、高度或深度）。数组/cube 切片、lod、bias 等不受影响。无采样器的图像操作不受影响。UINT 输入为"非归一化"。此位与 S#.force_unnormalized 位进行逻辑或运算。 |
| LWE | 1 | LOD 警告使能。设为 1 时，纹理取操作可能返回"LOD_CLAMPED = 1"，并引起 VGPR 写入（DST+1，即所有取目标 VGPR 之后的第一个 VGPR）。LWE 仅适用于采样器操作；非采样器操作中 LWE 被忽略。 |

### 10.1.1 纹素故障使能（TFE）和 LOD 警告使能（LWE）

这与"部分驻留纹理"相关。

当指令中设置了这两个位中的任何一个时，任何纹理取操作都可能在所有数据返回 VGPR 之后多返回一个额外 VGPR。此数据每线程唯一返回，指示该线程的错误/警告状态；若没有线程经历纹素故障或 LOD 警告，则不返回任何内容。

返回的数据为：`TEXEL_FAIL | (LOD_WARNING << 1) | (LOD << 16)`

- **TEXEL_FAIL**：1 位，指示该像素的 1 个或多个纹素产生了 NACK（"失败"意味着访问了未映射的页面）。
  - TFE==0 时：TEX 将未 NACK 线程的数据写入 VGPR DST；NACK 采样结果写零或混合零的结果到 VGPR DST。
  - TFE==1 时：VGPR DST 写入方式同上；TEX 还向 VGPR DST+1 写入状态，NACK 线程对应位设为 1。
- **LOD_WARNING**：1 位，指示像素尝试访问过小 LOD 的纹素：`warn = (LOD < T#.min_lod_warning)`
- **LOD**：指示触发 NACK 的 LOD 请求级别，返回所请求 LOD 的向下取整值。

一个像素不能同时收到 TEXEL_FAIL 和 LOD_WARNING：TEXEL_FAIL 优先级更高。

### 10.1.2 D16 指令

加载格式和存储格式指令也有"D16"变体。对于存储，每个 32 位 VGPR 保存两个 16 位数据元素并传递给纹理单元，纹理单元在写入内存之前将其转换为纹理格式。对于加载，纹理单元返回的数据被转换为 16 位，每对数据存储在一个 32 位 VGPR 中（先低位，后高位）。DMASK 位代表独立的 16 位元素；因此，当图像加载的 DMASK=0011 时，两个 16 位分量被加载到单个 32 位 VGPR 中。

### 10.1.3 A16 指令

A16 指令位指示地址分量为 16 位，而非通常的 32 位。分量按如下方式打包：第一个地址分量进入低 16 位（[15:0]），下一个进入高 16 位（[31:16]）。

### 10.1.4 G16 指令

名称中带"G16"的指令表示用户提供的导数为 16 位，而非通常的 32 位。导数按如下方式打包：第一个导数进入低 16 位（[15:0]），下一个进入高 16 位（[31:16]）。

## 10.2 无采样器的图像操作码

对于无采样器的图像操作码，所有 VGPR 地址值均视为无符号整数。

对于 cubemap，`face_id = slice * 6 + face`。

MSAA 表面仅支持加载、存储和原子操作，不支持 load-mip 或 store-mip。

下表展示了各种图像操作码对应的地址 VGPR 内容（a16=0 表示 32 位地址，a16=1 表示 16 位地址打包两个分量每 DWORD，acnt 为地址分量数）：

| 操作码 | a16[0] 类型 | acnt | VGPRn[31:0] | VGPRn+1[31:0] | VGPRn+2[31:0] | VGPRn+3[31:0] |
|--------|------------|------|-------------|--------------|--------------|--------------|
| GET_RESINFO | — | 任意 | 0（mipid） | — | — | — |
| LOAD / LOAD_PCK / LOAD_PCK_SGN / STORE / STORE_PCK（a16=0） | 0 | 1D:0 | s | — | — | — |
| | | 2D:1 | s | t | — | — |
| | | 3D:2 | s | t | r | — |
| | | Cube/Cube Array:2 | s | t | face | — |
| | | 1D Array:1 | s | slice | — | — |
| | | 2D Array:2 | s | t | slice | — |
| | | 2D MSAA:2 | s | t | fragid | — |
| | | 2D Array MSAA:3 | s | t | slice | fragid |
| LOAD / LOAD_PCK（a16=1） | 1 | 1D:0 | -, s | — | — | — |
| | | 2D:1 | t, s | — | — | — |
| | | 3D:2 | t, s | -, r | — | — |
| | | Cube/Cube Array:2 | t, s | -, face | — | — |
| | | 1D Array:1 | slice, s | — | — | — |
| | | 2D Array:2 | t, s | -, slice | — | — |
| | | 2D MSAA:2 | t, s | -, fragid | — | — |
| | | 2D Array MSAA:3 | t, s | fragid, slice | — | — |
| ATOMIC（a16=0） | 0 | 1D:0 | s | — | — | — |
| | | 2D:1 | s | t | — | — |
| | | 3D:2 | s | t | r | — |
| | | 1D Array:1 | s | slice | — | — |
| | | 2D Array:2 | s | t | slice | — |
| | | 2D MSAA:2 | s | t | fragid | — |
| | | 2D Array MSAA:3 | s | t | slice | fragid |
| ATOMIC（a16=1） | 1 | 1D:0 | -, s | — | — | — |
| | | 2D:1 | t, s | — | — | — |
| | | 3D:2 | t, s | -, r | — | — |
| | | 1D Array:1 | slice, s | — | — | — |
| | | 2D Array:2 | t, s | -, slice | — | — |
| | | 2D MSAA:2 | t, s | -, fragid | — | — |
| | | 2D Array MSAA:3 | t, s | fragid, slice | — | — |
| LOAD_MIP / LOAD_MIP_PCK / LOAD_MIP_PCK_SGN / STORE_MIP / STORE_MIP_PCK（a16=0） | 0 | 1D:1 | s | mipid | — | — |
| | | 2D:2 | s | t | mipid | — |
| | | 3D:3 | s | t | r | mipid |
| | | Cube/Cube Array:3 | s | t | face | mipid |
| | | 1D Array:2 | s | slice | mipid | — |
| | | 2D Array:3 | s | t | slice | mipid |
| LOAD_MIP（a16=1） | 1 | 1D:1 | mipid, s | — | — | — |
| | | 2D:2 | t, s | -, mipid | — | — |
| | | 3D:3 | t, s | mipid, r | — | — |
| | | Cube/Cube Array:3 | t, s | mipid, face | — | — |
| | | 1D Array:2 | slice, s | -, mipid | — | — |
| | | 2D Array:3 | t, s | mipid, slice | — | — |

- **Image_Load**：image_load、image_load_mip、image_load_{pck, pck_sgn, mip_pck, mip_pck_sgn}
- **Image_Store**：image_store、image_store_mip
- **Image_Atomic_***：swap、cmpswap、add、sub、{u,s}{min,max}、and、or、xor、inc、dec、add_f32、min_f32、max_f32

"ACNT"为地址计数：由指令的 DIM 字段和操作码派生，表示提供地址"主体"的 VGPR 数。

## 10.3 带采样器的图像操作码

带采样器的操作码：所有 VGPR 地址值均视为浮点数，纹素偏移除外（为无符号整数）。

对于 cubemap，`face_id = slice * 8 + face`。（注意此处乘以 8，与无采样器情况乘以 6 不同。）

某些采样和 gather 操作码需要从 VGPR 获取下表中未显示的额外值，包括：offset、bias、z-compare 和梯度。MSAA 表面不支持采样或 gather4 操作。MSAA_LOAD 不使用采样器，但使用 VSAMPLE 指令编码（用户应将 SAMP 设为 NULL）。

| 操作码 | a16[0] | acnt 类型 | VGPRn[31:0] | VGPRn+1[31:0] | VGPRn+2[31:0] | VGPRn+3[31:0] |
|--------|--------|----------|-------------|--------------|--------------|--------------|
| Sample / GetLod（a16=0） | 0 | 1D:0 | s | — | — | — |
| | | 2D:1 | s | t | — | — |
| | | 3D:2 | s | t | r | — |
| | | Cube(Array):2 | s | t | face | — |
| | | 1D Array:1 | s | slice | — | — |
| | | 2D Array:2 | s | t | slice | — |
| Sample（a16=1） | 1 | 1D:0 | -, s | — | — | — |
| | | 2D:1 | t, s | — | — | — |
| | | 3D:2 | t, s | -, r | — | — |
| | | Cube(Array):2 | t, s | -, face | — | — |
| | | 1D Array:1 | slice, s | — | — | — |
| | | 2D Array:2 | t, s | -, slice | — | — |
| Sample "_L"（a16=0） | 0 | 1D:1 | s | lod | — | — |
| | | 2D:2 | s | t | lod | — |
| | | 3D:3 | s | t | r | lod |
| | | Cube(Array):3 | s | t | face | lod |
| | | 1D Array:2 | s | slice | lod | — |
| | | 2D Array:3 | s | t | slice | lod |
| Sample "_L"（a16=1） | 1 | 1D:1 | lod, s | — | — | — |
| | | 2D:2 | t, s | -, lod | — | — |
| | | 3D:3 | t, s | lod, r | — | — |
| | | Cube(Array):3 | t, s | lod, face | — | — |
| | | 1D Array:2 | slice, s | -, lod | — | — |
| | | 2D Array:3 | t, s | lod, slice | — | — |
| Sample "_CL"（a16=0） | 0 | 1D:1 | s | clamp | — | — |
| | | 2D:2 | s | t | clamp | — |
| | | 3D:3 | s | t | r | clamp |
| | | Cube(Array):3 | s | t | face | clamp |
| | | 1D Array:2 | s | slice | clamp | — |
| | | 2D Array:3 | s | t | slice | clamp |
| Sample "_CL"（a16=1） | 1 | 1D:1 | clamp, s | — | — | — |
| | | 2D:2 | t, s | -, clamp | — | — |
| | | 3D:3 | t, s | clamp, r | — | — |
| | | Cube(Array):3 | t, s | clamp, face | — | — |
| | | 1D Array:2 | slice, s | -, clamp | — | — |
| | | 2D Array:3 | t, s | clamp, slice | — | — |
| Gather（a16=0） | 0 | 2D:1 | s | t | — | — |
| | | Cube(Array):2 | s | t | face | — |
| | | 2D Array:2 | s | t | slice | — |
| Gather（a16=1） | 1 | 2D:1 | t, s | — | — | — |
| | | Cube(Array):2 | t, s | -, face | — | — |
| | | 2D Array:2 | t, s | -, slice | — | — |
| Gather "_L"（a16=0） | 0 | 2D:2 | s | t | lod | — |
| | | Cube(Array):3 | s | t | face | lod |
| | | 2D Array:3 | s | t | slice | lod |
| Gather "_L"（a16=1） | 1 | 2D:2 | t, s | -, lod | — | — |
| | | Cube(Array):3 | t, s | lod, face | — | — |
| | | 2D Array:3 | t, s | lod, slice | — | — |
| Gather "_CL"（a16=0） | 0 | 2D:2 | s | t | clamp | — |
| | | Cube(Array):3 | s | t | face | clamp |
| | | 2D Array:3 | s | t | slice | clamp |
| Gather "_CL"（a16=1） | 1 | 2D:2 | t, s | -, clamp | — | — |
| | | Cube(Array):3 | t, s | clamp, face | — | — |
| | | 2D Array:3 | t, s | clamp, slice | — | — |
| MSAA_LOAD（a16=0） | 0 | 2D MSAA:2 | s | t | fragid | — |
| | | 2D Array MSAA:3 | s | t | slice | fragid |
| MSAA_LOAD（a16=1） | 1 | 2D MSAA:2 | t, s | -, fragid | — | — |
| | | 2D Array MSAA:3 | t, s | fragid, slice | — | — |

### 表59. 采样指令后缀说明

| 后缀 | 含义 | 额外地址 | 描述 |
|------|------|---------|------|
| _L | LOD | — | 使用提供的 LOD 代替计算出的 LOD。 |
| _B | LOD BIAS | 1: lod bias | 将此 BIAS 加到计算出的 LOD。 |
| _CL | LOD CLAMP | — | 将计算出的 LOD 夹紧到不超过此值。 |
| _D | 导数 | 2、4 或 6: 斜率 | 发送 dx/dv、dx/dy 等斜率用于 LOD 计算。 |
| _LZ | 第 0 级 | — | 强制使用 MIP 级别 0。 |
| _C | PCF | 1: z-comp | 百分比接近过滤。 |
| _O | 偏移 | 1: offsets | 发送 X、Y、Z 整数偏移（打包成 1 DWORD）以偏移 XYZ 地址。 |
| _G16 | 梯度 16 位 | — | 梯度为 16 位而非 32 位，每 VGPR 打包 2 个梯度（dX 在低 16 位，dY 在高 16 位）。 |

## 10.4 VGPR 使用

**地址**：地址由最多 5 个部分组成：{ offset } { bias } { z-compare } { derivative } { body }，均在 VGPR 中，最多包含 12 个值。

- **Offset**：SAMPLE*O*、GATHER*O*，1 DWORD 的 'offset_xyz'。偏移为 6 位有符号整数：X=[5:0]，Y=[13:8]，Z=[21:16]。
- **Bias**：SAMPLE*B*、GATHER*B*，1 DWORD 浮点数。
- **Z-compare**：SAMPLE*C*、GATHER*C*，1 DWORD。
- **导数（SAMPLE_D）**：2、4 或 6 个 DWORD，每个导数打包 1 DWORD（F32），如下所示。
- **Body**：1 到 4 个 DWORD，由"带采样器的图像操作码"表定义。

地址分量 X、Y、Z、W 依次存放，X 在 VGPR[M]，Y 在 VGPR[M]+1，以此类推。

"body"中的分量数为表中 ACNT 字段值加 1。

注：Bias 和导数互斥——着色器只能使用其中一个。

**32 位导数：**

| 图像维度 | VGPR N | N+1 | N+2 | N+3 | N+4 | N+5 |
|---------|--------|-----|-----|-----|-----|-----|
| 1D | dx/dh | dx/dv | — | — | — | — |
| 2D/cube | dx/dh | dy/dh | dx/dv | dy/dv | — | — |
| 3D | dx/dh | dy/dh | dz/dh | dx/dv | dy/dv | dz/dv |

**16 位导数：**

| 图像类型 | VGPR_N | VGPR_N+1 | VGPR_N+2 | VGPR_N+3 |
|---------|--------|---------|---------|---------|
| 1（1D、1D Array） | 16'hx, dx/dh | 16'hx, dx/dv | — | — |
| 2（2D、2D Array、Cubemap） | dy/dh, dx/dh | dy/dv, dx/dv | — | — |
| 3（3D） | dy/dh, dx/dh | 16'hx, dz/dh | dy/dv, dx/dv | 16'hx, dz/dv |

### 10.4.1 地址 VGPR

图像和采样指令支持多个地址 VGPR，允许着色器进行地址分量的 gather 操作，最多可指定 5 个唯一的地址 VGPR：

- **VADDR** 提供第一个地址分量
- **VADDR1** 提供第二个地址分量
- **VADDR2** 提供第三个地址分量
- **VADDR3**（VIMAGE）提供第四个地址分量；对于 VSAMPLE，提供所有额外分量（VADDR3、VADDR3+1 等）
- **VADDR4**（VIMAGE）提供所有额外分量（VADDR4、VADDR4+1 等）

A16 指令位指定地址分量为 16 位而非通常 32 位。使用 16 位地址时，每个 VGPR 保存一对地址，且不能分散在不同 VGPR 中。低编号的 16 位值在 VGPR 的低位。

对于光线追踪，VGPR 被分为 5 组，每组内必须连续，但各组可以分散存放。当 A16=1 时，打包方式不同，因为 RayDir.Z 和 RayInvDir.x 在同一 DWORD 中。A16 模式下，RayDir 和 RayInvDir 被合并进 3 个 VGPR，但顺序不同：每个分量的 RayDir 和 RayInvDir 共享一个 VGPR。

### 10.4.2 数据 VGPR

**数据**：数据存储于或返回到 1-4 个连续 VGPR。加载或存储的数据量完全由指令的 DMASK 字段决定。

**加载**

DMASK 指定将资源的哪些元素返回到连续 VGPR。纹理系统从内存加载数据，根据数据格式将其扩展为标准 RGBA 形式，根据 T#.dst_sel 填充缺失分量，然后应用 DMASK，只将选定分量返回给着色器。

**存储**

写入图像对象时，只能存储完整的元素（所有分量），而不能只写个别分量。分量来自连续 VGPR；若表面的分量数多于着色器提供的分量数，缺失分量由复制第一个 VGPR 填充。例如，若表面有 4 个分量，着色器只提供 X 和 Y，则表面写入 XYXX。

例如，若 DMASK=1001，着色器将 VGPR_N 的红色和 VGPR_N+1 的透明度发送给纹理单元。若图像对象为 RGB，纹素将被覆盖为：红色来自 VGPR_N，绿色和蓝色设为红色值，透明度来自着色器的值被忽略。对于 D16=1，DMASK 中每 1 位代表从 VGPR 写入内存的 16 位数据，位的位置无关紧要，只有设为 1 的位数才有意义。

**"D16"指令**

加载和存储指令也有"D16"变体。对于存储，每个 32 位 VGPR 保存两个 16 位数据元素，传递给纹理单元，纹理单元在写入内存之前将其转换为纹理格式。对于加载，纹理单元返回的数据被转换为 16 位，每对数据存储在一个 32 位 VGPR（先低位，后高位）。若只有一个分量，数据进入 VGPR 的下半部分，除非使用"HI"指令变体——此时 VGPR 的上半部分被加载。

**原子操作**

图像原子操作仅支持 32 位或 64 位每像素的表面，表面数据格式在资源常量中指定。原子操作将元素视为单个 32 位或 64 位分量。对于原子操作，DMASK 设置为发送给纹理单元的 VGPR 数（DWORD 数）。

原子图像操作的 DMASK 合法值（其他值均非法）：
- `0x1`：32 位原子，cmpswap 除外
- `0x3`：32 位原子 cmpswap
- `0x3`：64 位原子，cmpswap 除外
- `0xf`：64 位原子 cmpswap

带返回的原子操作：从 VDATA 开始的 VGPR 中读取数据，供原子操作使用。若原子操作将值返回 VGPR，数据返回到从 VDATA 开始的相同 VGPR。

DMASK 必须与资源的数据格式兼容。

**浮点数中的非正规值**

采样操作将非正规数刷新为零，加载操作不修改非正规数。

### 10.4.3 VGPR 中的数据格式

发送到纹理（存储）或从纹理返回（加载）的 VGPR 数据采用几种标准格式之一，纹理单元负责与内存格式之间的转换。

| FORMAT | VGPR 数据格式 | D16==1 时 |
|--------|------------|---------|
| SINT | 有符号 32 位整数 | 16 位有符号整数 |
| UINT | 无符号 32 位整数 | 16 位无符号整数 |
| 其他 | 32 位浮点数 | 16 位浮点数 |
| 原子操作 | 取决于操作码：uint 或 float | — |

## 10.5 图像资源

图像资源（也称为 T#）定义了图像缓冲区在内存中的位置、维度、平铺方式和数据格式。这些资源存储在 4 或 8 个连续 SGPR 中，由图像指令读取。所有未定义或保留位必须设为零，除非另有说明。

### 表60. 图像资源定义

**128 位资源：1D 纹理、2D 纹理、2D MSAA（多重采样抗锯齿）**

| 位 | 位宽 | 名称 | 说明 |
|----|------|------|------|
| 39:0 | 40 | 基地址 | 256 字节对齐（表示位 47:8）。 |
| 48:44 | 5 | max mip | MSAA 资源：保存 log2(采样数)；其他：最大 mip 级别（MipLevels-1）。描述的是资源而非资源视图（如 base_level/last_level）。 |
| 56:49 | 8 | format | 内存数据格式 |
| 61:57 | 5 | base level | 资源视图中最大的 mip 级别。MSAA 应设为 0。 |
| 77:62 | 16 | width | mip 0 的宽度减 1（以纹素为单位） |
| 93:78 | 16 | height | mip 0 的高度减 1（以纹素为单位） |
| 98:96 | 3 | dst_sel_x | 目标通道选择：0=0，1=1，4=R，5=G，6=B，7=A |
| 101:99 | 3 | dst_sel_y | 同上 |
| 104:102 | 3 | dst_sel_z | 同上 |
| 107:105 | 3 | dst_sel_w | 同上 |
| 115:111 | 5 | last level | 资源视图中最小的 mip 级别。MSAA 资源保存 log2(采样数)。 |
| 123:121 | 3 | BC Swizzle | 指定边界颜色数据的通道排列，独立于 T# dst_sel_* 之外。内部 xyzw 通道从内存中获取以下边界颜色通道：0=xyzw，1=xwyz，2=wzyx，3=wxyz，4=zyxw，5=yxwz |
| 127:124 | 4 | type | 0=buf，8=1D，9=2D，10=3D，11=cube，12=1D数组，13=2D数组，14=2D-MSAA，15=2D-MSAA数组。1-7 保留。 |

**256 位资源：1D 数组、2D 数组、3D、Cubemap、MSAA**

| 位 | 位宽 | 名称 | 说明 |
|----|------|------|------|
| 141:128 | 14 | depth | 3D 资源：mip 0 的深度减 1。1D 或 2D 数组、cube 资源：最后一个数组切片（见类型表）。最大 8k，第 14 位不使用。1D 或 2D、MSAA 资源：pitch-1 的低位（即 mip 0 的 (pitch-1)[13:0]，当 pitch > width 时）；pitch_msb 字段包含 pitch 字段的高位。其他资源：必须为零。 |
| 143:142 | 2 | pitch_msb | 1D 或 2D、MSAA 资源：pitch-1 的高位（即 mip 0 的 (pitch-1)[15:14]，当 pitch > width 时）。其他资源：必须为零。 |
| 156:144 | 13 | base array | 资源视图数组的第一个切片。 |
| 164 | 1 | UAV3D | 3D 资源：位 0 指示 SRV 或 UAV：0=SRV（base_array 被忽略，depth 相对于 base map）；1=UAV（base_array 和 depth 为视图的第一层和最后一层，相对于指定的 mip 级别）。其他资源：不使用。 |
| 177:165 | 13 | min_lod_warn | LOD 反馈触发器，u5.8 格式。 |
| 183 | 1 | corner samples mod | 描述资源中纹素的生成方式：0=中心采样，1=角点采样。 |
| 198:186 | 13 | min_lod | PRT 允许的最小 LOD，U5.8 格式。 |

全零的资源视为"未绑定"：返回全零，不生成内存事务。检查"全零"时忽略"resource-level"字段。

## 10.6 图像采样器

采样器资源（也称为 S#）定义了对采样指令加载的纹理贴图数据执行哪些操作，主要包括地址夹紧和过滤选项。采样器资源由四个连续 SGPR 定义，并随每条采样指令一起提供给纹理缓存。

### 表61. 图像采样器定义

| 位 | 位宽 | 名称 | 描述 |
|----|------|------|------|
| 2:0 | 3 | clamp x | 夹紧/环绕模式：0=Wrap，1=Mirror，2=ClampLastTexel，3=MirrorOnceLastTexel，4=ClampHalfBorder，5=MirrorOnceHalfBorder，6=ClampBorder，7=MirrorOnceBorder |
| 5:3 | 3 | clamp y | 同上 |
| 8:6 | 3 | clamp z | 同上 |
| 11:9 | 3 | max aniso ratio | 最大各向异性比率：0=1:1，1=2:1，2=4:1，3=8:1，4=16:1 |
| 14:12 | 3 | depth compare func | 深度比较函数：0=Never，1=Less，2=Equal，3=Less than or equal，4=Greater，5=Not equal，6=Greater than or equal，7=Always |
| 15 | 1 | force unnormalized | 强制地址坐标非归一化：0=坐标已归一化，范围 [0,1)；1=坐标未归一化，范围 [0,dim)。 |
| 18:16 | 3 | aniso threshold | 各向异性阈值，低于此值时 floor(各向异性比率) 决定采样数和步长。 |
| 19 | 1 | mc coord trunc | 启用双线性混合分数截断为 1 位，用于运动补偿。 |
| 20 | 1 | force degamma | 若 data_format 允许，强制格式为 sRGB。 |
| 26:21 | 6 | aniso bias | 6 位，u1.5 格式。 |
| 27 | 1 | trunc coord | 选择纹素坐标舍入或截断。 |
| 28 | 1 | disable cube wrap | 禁用无缝 DX10 cubemap，允许 cubemap 按 clamp_x 和 clamp_y 字段夹紧。 |
| 30:29 | 2 | filter_mode | 0=Blend（线性插值）；1=min；2=max。 |
| 31 | 1 | skip degamma | 禁用 degamma（sRGB→Linear）转换。 |
| 44:32 | 13 | min lod | 资源视图空间中的最小 LOD（0.0 = T#.base_level），u5.8 格式。 |
| 57:45 | 13 | max lod | 资源视图空间中的最大 LOD。 |
| 77:64 | 14 | lod bias | LOD 偏差，s6.8 格式。 |
| 83:78 | 6 | lod bias sec | 加到计算 LOD 的偏差（s2.4 格式）。 |
| 85:84 | 2 | xy mag filter | 放大过滤器：0=point，1=bilinear，2=aniso-point，3=aniso-linear。 |
| 87:86 | 2 | xy min filter | 缩小过滤器：0=point，1=bilinear，2=aniso-point，3=aniso-linear。 |
| 89:88 | 2 | z filter | 体积过滤器：0=无（使用 XY min/mag 过滤器），1=point，2=linear。 |
| 91:90 | 2 | mip filter | Mip 级别过滤器：0=无（禁用 mipmapping，使用基础级别），1=point，2=linear。 |
| 125:114 | 12 | border color ptr | 边界颜色空间的索引。 |
| 127:126 | 2 | border color type | 不透明黑、透明黑、白、使用 border color ptr。0=透明黑，1=不透明黑，2=不透明白，3=寄存器（用户边界颜色，由 border_color_ptr 指向）。 |

## 10.7 数据格式

下表详细列出了图像和缓冲区资源可使用的所有数据格式。

### 表62. 缓冲区和图像数据格式

| # | 格式 | # | 格式 | # | 格式 |
|---|------|---|------|---|------|
| 0 | INVALID | 32 | 10_10_10_2_UNORM | 64 | 8_SRGB |
| 1 | 8_UNORM | 33 | 10_10_10_2_SNORM | 65 | 8_8_SRGB |
| 2 | 8_SNORM | 34 | 10_10_10_2_UINT | 66 | 8_8_8_8_SRGB |
| 3 | 8_USCALED | 35 | 10_10_10_2_SINT | 67 | 5_9_9_9_FLOAT |
| 4 | 8_SSCALED | 36 | 2_10_10_10_UNORM | 68 | 5_6_5_UNORM |
| 5 | 8_UINT | 37 | 2_10_10_10_SNORM | 69 | 1_5_5_5_UNORM |
| 6 | 8_SINT | 38 | 2_10_10_10_USCALED | 70 | 5_5_5_1_UNORM |
| 7 | 16_UNORM | 39 | 2_10_10_10_SSCALED | 71 | 4_4_4_4_UNORM |
| 8 | 16_SNORM | 40 | 2_10_10_10_UINT | 72 | 4_4_UNORM |
| 9 | 16_USCALED | 41 | 2_10_10_10_SINT | 73 | 1_UNORM |
| 10 | 16_SSCALED | 42 | 8_8_8_8_UNORM | 74 | 1_REVERSED_UNORM |
| 11 | 16_UINT | 43 | 8_8_8_8_SNORM | 75 | 32_FLOAT_CLAMP |
| 12 | 16_SINT | 44 | 8_8_8_8_USCALED | 76 | 8_24_UNORM |
| 13 | 16_FLOAT | 45 | 8_8_8_8_SSCALED | 77 | 8_24_UINT |
| 14 | 8_8_UNORM | 46 | 8_8_8_8_UINT | 78 | 24_8_UNORM |
| 15 | 8_8_SNORM | 47 | 8_8_8_8_SINT | 79 | 24_8_UINT |
| 16 | 8_8_USCALED | 48 | 32_32_UINT | 80 | X24_8_32_UINT |
| 17 | 8_8_SSCALED | 49 | 32_32_SINT | 81 | X24_8_32_FLOAT |
| 18 | 8_8_UINT | 50 | 32_32_FLOAT | 82 | GB_GR_UNORM |
| 19 | 8_8_SINT | 51 | 16_16_16_16_UNORM | 83 | GB_GR_SNORM |
| 20 | 32_UINT | 52 | 16_16_16_16_SNORM | 84 | GB_GR_UINT |
| 21 | 32_SINT | 53 | 16_16_16_16_USCALED | 85 | GB_GR_SRGB |
| 22 | 32_FLOAT | 54 | 16_16_16_16_SSCALED | 86 | BG_RG_UNORM |
| 23 | 16_16_UNORM | 55 | 16_16_16_16_UINT | 87 | BG_RG_SNORM |
| 24 | 16_16_SNORM | 56 | 16_16_16_16_SINT | 88 | BG_RG_UINT |
| 25 | 16_16_USCALED | 57 | 16_16_16_16_FLOAT | 89 | BG_RG_SRGB |
| 26 | 16_16_SSCALED | 58 | 32_32_32_UINT | — | — |
| 27 | 16_16_UINT | 59 | 32_32_32_SINT | — | — |
| 28 | 16_16_SINT | 60 | 32_32_32_FLOAT | — | — |
| 29 | 16_16_FLOAT | 61 | 32_32_32_32_UINT | — | — |
| 30 | 10_11_11_FLOAT | 62 | 32_32_32_32_SINT | — | — |
| 31 | 11_11_10_FLOAT | 63 | 32_32_32_32_FLOAT | — | — |

**压缩格式**

| # | 格式 | # | 格式 |
|---|------|---|------|
| 109 | BC1_UNORM | 116 | BC4_SNORM |
| 110 | BC1_SRGB | 117 | BC5_UNORM |
| 111 | BC2_UNORM | 118 | BC5_SNORM |
| 112 | BC2_SRGB | 119 | BC6_UFLOAT |
| 113 | BC3_UNORM | 120 | BC6_SFLOAT |
| 114 | BC3_SRGB | 121 | BC7_UNORM |
| 115 | BC4_UNORM | 122 | BC7_SRGB |
| 205 | YCBCR_UNORM | 206 | YCBCR_SRGB |
| 227 | 6E4_FLOAT | — | — |

## 10.8 向量内存指令数据依赖

当 VM 指令发出时，它会调度从 VGPR 读取地址和存储数据以发送给纹理单元。任何试图在这些数据发送给纹理单元之前写入这些数据的 ALU 指令都会被阻塞。

着色器开发者负责避免与 VMEM 指令相关的数据危害，包括在读取从 TC 取回的数据之前等待 VMEM 读指令完成（LOADcnt 和 STOREcnt）。光线追踪图像 BVH 指令通过 BVHcnt 跟踪。

详细说明参见"数据依赖解析"章节。

## 10.9 光线追踪

光线追踪支持以下指令：

| 指令 | 说明 |
|------|------|
| IMAGE_BVH_INTERSECT_RAY | 测试单个 QBVH 节点，每通道由一个 32 位节点指针引用。 |
| IMAGE_BVH64_INTERSECT_RAY | 测试单个 QBVH 节点，每通道由一个 64 位节点指针引用。 |
| IMAGE_BVH_DUAL_INTERSECT_RAY | 测试两个 QBVH 节点，由一个 64 位 BVH 基地址和两个 32 位节点偏移引用。 |
| IMAGE_BVH8_INTERSECT_RAY | 测试一个 BVH8 节点，由一个 64 位 BVH 基地址和一个 32 位节点偏移引用。 |

光线追踪指令支持还包括 LDS 中的 BVH 栈操作，参见"光线追踪栈操作"章节。

这些指令从 VGPR 接收光线数据，并从内存中取 BVH（包围体层次结构）数据。

- **Box BVH 节点**：执行 4 次 Ray/Box 相交测试，根据相交距离对 4 个子节点排序，并返回子节点指针和命中状态。
- **Triangle 节点**：执行 1 次 Ray/Triangle 相交测试，返回交点和三角形 ID。

IMAGE_BVH_INTERSECT_RAY 和 IMAGE_BVH64_INTERSECT_RAY 的区别仅在于地址宽度："64"版本支持 64 位地址，普通版本仅支持 32 位地址。这些指令可使用"A16"指令字段将部分（非全部）地址分量从 32 位减少到 16 位，但 image_bvh_dual_intersect_ray 和 image_bvh8_intersect_ray 不支持 A16=1。可为 16 位的地址为：ray_dir 和 ray_inv_dir。

### 10.9.1 指令定义与字段

```
image_bvh_intersect_ray       vgpr_d[4],  vgpr_a[11], sgpr_r[4]
image_bvh_intersect_ray       vgpr_d[4],  vgpr_a[8],  sgpr_r[4]   A16=1
image_bvh64_intersect_ray     vgpr_d[4],  vgpr_a[12], sgpr_r[4]
image_bvh64_intersect_ray     vgpr_d[4],  vgpr_a[9],  sgpr_r[4]   A16=1
image_bvh_dual_intersect_ray  vgpr_d[10], vgpr_a[12], sgpr_r[4]
image_bvh8_intersect_ray      vgpr_d[10], vgpr_a[11], sgpr_r[4]
```

发出后，这些指令对 wave 中每个活跃通道执行以下操作：
1. 从 SP 接收描述待测光线的数据（存储在 vgpr_a 中）
2. 从 SP 接收应被取并测试的 BVH 节点指针（存储在 vgpr_a 中）
3. 从着色器接收 BVH 资源描述符（存储在 sgpr_r 中）
4. 计算正在测试的 BVH 节点的节点类型、数据大小和地址
5. 从内存中取 BVH 节点
6. 根据 BVH 节点类型执行相交测试
7. 将相交结果返回给着色器（存储在 vgpr_d 中）
8. 若测试了实例节点，则更新光线原点和方向（更新 vgpr_a 中的值）

wave 中每个通道可以针对不同的 BVH 节点测试不同的光线，甚至在单个 wave 的一次指令发出中混合多种 BVH 节点测试类型。此外，image_bvh 指令可以使用标准的 s_waitcnt 同步机制，与其他图像、缓冲区和 flat 内存指令流水线执行。

但硬件不在返回控制权给着色器之前执行任何递归或内部循环——它只测试 BVH 节点与每条光线的相交关系并返回，着色器必须自行实现完整 BVH 遍历所需的遍历循环。

### 表63. image_bvh_intersect_ray 和 image_bvh64_intersect_ray 的光线追踪 VGPR 内容

| VGPR_A | BVH A16=0 | BVH A16=1 | BVH64 A16=0 | BVH64 A16=1 |
|--------|-----------|-----------|-------------|-------------|
| 0 | node_pointer (u32) | node_pointer (u32) | node_pointer [31:0] (u32) | node_pointer [31:0] (u32) |
| 1 | ray_extent (f32) | ray_extent (f32) | node_pointer [63:32] (u32) | node_pointer [63:32] (u32) |
| 2 | ray_origin.x (f32) | ray_origin.x (f32) | ray_extent (f32) | ray_extent (f32) |
| 3 | ray_origin.y (f32) | ray_origin.y (f32) | ray_origin.x (f32) | ray_origin.x (f32) |
| 4 | ray_origin.z (f32) | ray_origin.z (f32) | ray_origin.y (f32) | ray_origin.y (f32) |
| 5 | ray_dir.x (f32) | [15:0]=ray_dir.x (f16), [31:16]=ray_dir.y (f16) | ray_origin.z (f32) | ray_origin.z (f32) |
| 6 | ray_dir.y (f32) | [15:0]=ray_dir.z (f16), [31:16]=ray_inv_dir.x (f16) | ray_dir.x (f32) | [15:0]=ray_dir.x (f16), [31:16]=ray_dir.y (f16) |
| 7 | ray_dir.z (f32) | [15:0]=ray_inv_dir.y (f16), [31:16]=ray_inv_dir.z (f16) | ray_dir.y (f32) | [15:0]=ray_dir.z (f16), [31:16]=ray_inv_dir.x (f16) |
| 8 | ray_inv_dir.x (f32) | unused | ray_dir.z (f32) | [15:0]=ray_inv_dir.y (f16), [31:16]=ray_inv_dir.z (f16) |
| 9 | ray_inv_dir.y (f32) | unused | ray_inv_dir.x (f32) | unused |
| 10 | ray_inv_dir.z (f32) | unused | ray_inv_dir.y (f32) | unused |
| 11 | unused | unused | ray_inv_dir.z (f32) | unused |

vgpr_d[4] 为相交测试结果的目标 VGPR。对于 Box 节点，结果包含按相交时间排序的 4 个子节点指针；对于 Triangle BVH 节点，结果包含相交时间和被测三角形的 ID。

sgpr_r[4] 为操作的纹理描述符，指令编码中 use_128bit_resource=1。

### IMAGE_BVH_DUAL_INTERSECT_RAY

### 表64. image_bvh_dual_intersect_ray 和 image_bvh8_intersect_ray 的光线追踪 VGPR 内容

| VGPR 组 | 组内 VGPR | 内容 |
|---------|---------|------|
| 0 | 0 | bvh_base[31:0]（uint64 的第一部分，输入，bits[2:0] 必须为 0） |
| 0 | 1 | bvh_base[63:32]（uint64 的第二部分，输入） |
| 1 | 0 | ray_extent（float32，输入） |
| 1 | 1 | instance_mask（8 位 uint，输入） |
| 2 | 0 | ray_origin.x（float32，输入和输出参数） |
| 2 | 1 | ray_origin.y（float32，输入和输出参数） |
| 2 | 2 | ray_origin.z（float32，输入和输出参数） |
| 3 | 0 | ray_dir.x（float32，输入和输出参数） |
| 3 | 1 | ray_dir.y（float32，输入和输出参数） |
| 3 | 2 | ray_dir.z（float32，输入和输出参数） |
| 4 | 0 | image_bvh_dual_intersect_ray: node_pointer0（第一个待测节点的偏移，uint32，输入）；image_bvh8_intersect_ray: node_pointer（待测 BVH8 节点的偏移，uint32） |
| 4 | 1 | image_bvh_dual_intersect_ray: node_pointer1（第二个待测节点的偏移，uint32，输入）；image_bvh8_intersect_ray: unused |

vgpr_d[10] 为相交测试结果的目标 VGPR。前 4 个值对应第一个节点的结果，后 4 个值对应第二个节点的结果。若两个节点都是 Box 节点且启用了宽排序，则 8 个值在两个测试节点之间交错排列（如 8 路宽排序所指定）。返回值因所取 BVH 节点的类型而不同。

对于 Box 节点，结果包含按相交时间排序的 4 个子节点指针。对于 Triangle BVH 节点，结果包含相交时间和三角形 ID 或重心坐标。对于 Instance 节点，数据包含 64 位 BVH 基地址、BVH 中的节点指针偏移、24 位用户数据字段和 8 位实例掩码，详见"相交引擎返回数据"章节。最后两个 DWORD 包含每个被测三角形的 ShapeID/GeoID。

### IMAGE_BVH8_INTERSECT_RAY

vgpr_d[10] 为相交测试结果的目标 VGPR，返回值因所取 BVH 节点的类型而不同。

对于 Box 节点，结果包含按相交时间排序的 8 个子节点指针。当宽排序被禁用时，前 4 个值对应 0-3 号 Box 的结果，后 4 个值对应 4-7 号 Box 的结果；若启用宽排序，则 8 个值完全交错，包含 0-7 号 Box 的结果（如 8 路宽排序所指定）。对于 Triangle BVH 节点，结果包含两个被测三角形的相交时间和三角形 ID 或重心坐标。对于 Instance 节点，数据包含 64 位 BVH 基地址、BVH 中的节点指针偏移、24 位用户数据字段和 8 位实例掩码，详见"相交引擎返回数据"章节。最后两个 DWORD 包含每个被测三角形的 ShapeID/GeoID。

**image_bvh 指令的限制**

- DMASK 必须设为 0xf（指令返回全部四个 DWORD）
- D16 必须设为 0（不支持 16 位返回数据）
- R128 必须设为 1（不支持 256 位 T#）
- UNRM 必须设为 1（仅支持非归一化坐标）
- DIM 必须设为 0（BVH 纹理为 1D）
- LWE 必须设为 0（不支持 LOD 警告）
- TFE 必须设为 0（不支持为 PRT 命中状态写出额外 DWORD）
- SSAMP 必须设为 0（仅为占位，采样器不用于此指令）

BVH 操作的返回顺序设置被忽略，改用按序加载返回队列。

### 10.9.2 VGPR_A 字段组织

VIMAGE 指令编码指定 5 个 VADDR 字段，BVH 指令使用方式如下：

```
node pointer  — VADDR0: 1 个 vgpr
ray extent    — VADDR1: 1 个 vgpr
ray origin    — VADDR2: 3 个连续 vgpr
ray dir       — VADDR3: 3 个连续 vgpr
ray inv dir   — VADDR4: 3 个连续 vgpr（与 ray-dir 共享 16 位地址）
```

使用 A16=1 模式时，ray-dir 和 ray-inv-dir 共享同一组 VGPR，ADDR4 不使用。

### 10.9.3 BVH 纹理资源定义

用于 BVH 指令的 T# 与其他图像指令的 T# 不同。

### 表65. BVH 资源定义

| 位 | 位宽 | 字段 | 描述 |
|----|------|------|------|
| 39:0 | 40 | base_address[47:8] | BVH 纹理的基地址，256 字节对齐。 |
| 51:40 | 12 | 保留 | 必须为零。 |
| 52 | 1 | sort_triangles_first | 0：三角形节点指针在子节点排序时不特殊处理；1：三角形节点指针（image_bvh8 的类型 0 和 1，其他 image_bvh 操作的类型 0、1、2、3）在有效 Box 节点之前排序。 |
| 54:53 | 2 | box_sorting_heuristic | 指定启用 Box 排序时的子节点排序启发式算法：0=Closest（优先进入最近的相交子节点）；1=LargestFirst（优先进入相交区间最大的子节点）；2=ClosestMidpoint（优先进入相交区间中点最小的子节点）；3=未定义（保留）。 |
| 62:55 | 8 | box_grow_value | UINT，用于扩展 Box 相交的 MAX 平面。 |
| 63 | 1 | box_sort_en | 启用 Box 相交结果排序的布尔值。 |
| 105:64 | 42 | size[47:6] | 以 64 字节为单位，表示 BVH 纹理节点数减 1，用于边界检查。 |
| 115:106 | 10 | 保留 | 必须为零。 |
| 116 | 1 | box_node_64B | 0：类型 4 节点为 FP16 Box 节点；1：类型 4 节点为 64 字节高精度 Box 节点。 |
| 117 | 1 | wide_sort_en | 0：对 4 个 Box 子节点排序；1：对 8 个 Box 子节点排序。 |
| 118 | 1 | instance_en | 0：节点 6 为用户节点；1：节点 6 为实例节点。 |
| 119 | 1 | pointer_flags | 0：不使用指针标志或其支持的特性；1：使用指针标志以启用 HW 绕向、背面剔除、不透明/非不透明剔除和基于基元类型的剔除。 |
| 120 | 1 | triangle_return_mode | 0：返回命中/未命中以及三角形测试结果 dword[3:0] = {t_num, t_denom, triangle_id, hit_status}；1：返回重心坐标以及三角形测试结果 dword[3:0] = {t_num, t_denom, I_num, J_num}。 |
| 123:121 | 3 | 保留 | 必须为零。 |
| 127:124 | 4 | type | 必须设为 0x08。 |

**重心坐标**

光线追踪硬件设计支持直接在硬件中计算重心坐标，使用上表中 T# 描述符的"triangle_return_mode"。

### 表66. 光线追踪返回模式

| DWORD | 返回模式=0 字段名 | 类型 | 返回模式=1 字段名 | 类型 |
|-------|----------------|------|----------------|------|
| 0 | t_num | float32 | t_num | float32 |
| 1 | t_denom | float32 | t_denom | float32 |
| 2 | triangle_id | uint32 | I_num | float32 |
| 3 | hit_status | uint32（布尔值） | J_num | float32 |

## 10.10 部分驻留纹理

"部分驻留纹理"（PRT）为内存中并非所有细节层次都驻留的纹理贴图提供支持。着色器编译器在资源中将纹理贴图声明为 PRT，但着色器程序也必须知晓这一点——若纹理取操作访问的 MIP 级别不在内存中，纹理单元会额外返回一个 DWORD 状态到 VGPR，指示取操作失败。若任何纹素不在内存中，纹理缓存返回 NACK，导致每个失败线程的 DST_VGPR+1 中写入非零值，该值可能代表所请求的 LOD。着色器程序必须为所有 PRT 纹理取操作分配这个额外 VGPR，并在取操作后检查其是否为零。用户应在发出可能返回 PRT 结果的纹理取操作之前，将此 PRT VGPR 初始化为零。

PRT 在纹理资源 MIN_LOD_WARN 值非零时启用。普通纹理不会产生 NACK，因此只有 PRT 才会收到 NACK；NACK 会导致写入 DST_VGPR+Num_VGPRS。例如，若 SAMPLE 将 4 个值加载到 4 个 VGPR（4、5、6、7），则 PRT 可能将 NACK 状态返回到 VGPR_8。
