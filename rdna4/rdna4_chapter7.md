# 第七章：向量 ALU 操作

向量 ALU 指令（VALU）对 32 或 64 个通道的数据执行算术或逻辑运算，并将结果写回 VGPR、SGPR 或 EXEC 掩码。

参数插值是一个两步过程，先执行 LDS 指令，再执行 VALU 指令（详见参数插值章节）。

VALU 指令控制 SIMD32 的数学单元，一次对 32 个工作项的数据进行操作。每条指令可从 VGPR、SGPR 或常量获取输入，通常将结果返回到 VGPR。掩码结果和进位输出返回到 SGPR。ALU 支持 16、32 和 64 位整数和浮点类型的操作，以及将两个 16 位值打包到一个 VGPR 或将四个 8 位值打包到一个 VGPR 的"打包"数据类型。

---

## 7.1 微码编码

VALU 指令有 6 种编码方式：

| 名称 | 大小 | 功能 | 修饰符 |
|------|------|------|--------|
| VOP2 | 32 位 | 2 输入 VALU 操作 | — |
| VOP1 | 32 位 | 1 输入 VALU 操作 | — |
| VOPC | 32 位 | 2 输入比较操作，写入 VCC/EXEC | — |
| VOP3 | 64 位 | 3 输入 VALU 操作，或 VOP1/2/C 指令 | abs, neg, omod, clamp |
| VOP3SD | 64 位 | 3 输入 VALU 操作，带标量目标 | neg, omod, clamp |
| VOP3P | 64 位 | 3 输入打包数学或需要较大 VGPR 范围的操作 | neg, clamp |
| VOPD | 64 位 | 双操作码：一条指令编码两个操作 | — |

许多 VALU 指令有两种编码：64 位的 VOP3 和三种限制能力但代码更小的 32 位编码之一。建议尽可能使用 32 位编码。

**VOP3 优势：**
- 源地址更灵活（所有源字段为 9 位）
- 浮点运算的 NEG、ABS 和 OMOD 字段
- 输出范围限制的 CLAMP 字段
- 可为 VCC 选择备用源和目标寄存器

**不能升级到 VOP3 的 VOP1/VOP2 指令：**
- SWAP 和 SWAPREL、PERMLANE64_B32
- FMAMK、FMAAK、PK_FMAC

**VOP3 的两种变体：**
- VOP3：用于大多数指令包括 V_CMP*；有 OPSEL 和 ABS 字段
- VOP3SD：有 SDST 字段取代 OPSEL 和 ABS，仅用于：V_{ADD,SUB,SUBREV}_CO_CI_U32、V_{ADD,SUB,SUBREV}_CO_U32（带进位加法）；V_DIV_SCALE_{F32,F64}、V_MAD_CO_U64_U32、V_MAD_CO_I64_I32

### 指令字段

| 字段 | 大小 | 描述 |
|------|------|------|
| OP | 可变 | 指令操作码 |
| SRC0 | 9 | 第一个参数，可来自：VGPR、SGPR、VCC、M0、EXEC、SCC 或常量 |
| SRC1 | 9 | 第二个参数，可来自：VGPR、SGPR、VCC、M0、EXEC、SCC 或常量 |
| VSRC1 | 8 | 第二个参数，仅 VGPR |
| SRC2 | 9 | 第三个参数，可来自：VGPR、SGPR、VCC、M0、EXEC、SCC 或常量 |
| VDST | 8 | 接收结果的 VGPR。V_READLANE 和 V_CMP 中表示接收结果的 SGPR（不能为 M0 或 EXEC） |
| SDST | 8 | 仅 VOP3SD 编码，接收标量输出的 SGPR，不能是 M0 或 EXEC；支持 NULL |
| OMOD | 2 | 输出修饰符（仅浮点结果）：0=无、1=×2、2=×4、3=÷2 |
| NEG | 3 | 对输入取反（反转符号位），仅浮点输入。bit0=src0，bit1=src1，bit2=src2 |
| ABS | 3 | 对输入取绝对值，仅浮点输入，在 neg 之前应用。bit0=src0，bit1=src1，bit2=src2 |
| CM | 1 | 取决于操作码：V_CMP 时 CM=1 表示检测到 qNaN 时发出信号比较；浮点算术时将结果限制到 [0, 1.0]；有符号整数限制到 [INT_MIN, INT_MAX]；无符号整数限制到 [0, UINT_MAX] |
| OPSEL | 4 | 16 位数学的操作选择：1=选高半部，0=选低半部。[0]=src0，[1]=src1，[2]=src2，[3]=dest |

---

## 7.2 操作数

### 表 34. VALU 指令操作数

| 代码 | 含义 | 标量源（8 位） | 标量目标（7 位） |
|------|------|-------------|----------------|
| 0-105 | SGPR 0 .. 105 | 标量通用寄存器，每个 1 DWORD | ✓ |
| 106 | VCC_LO | VCC[31:0] | ✓ |
| 107 | VCC_HI | VCC[63:32] | ✓ |
| 108-123 | TTMP0 .. TTMP15 | 陷阱处理程序临时 SGPR（特权） | ✓ |
| 124 | NULL | 读取返回零，写入被忽略 | ✓ |
| 125 | M0 | 临时寄存器 | ✓ |
| 126 | EXEC_LO | EXEC[31:0] | ✓ |
| 127 | EXEC_HI | EXEC[63:32] | ✓ |
| 128 | 0 | 整数内联常量零 | — |
| 129-192 | int 1 .. 64 | 整数内联常量 | — |
| 193-208 | int -1 .. -16 | 整数内联常量 | — |
| 233 | DPP8 | 8 通道 DPP（仅在 SRC0 有效） | — |
| 234 | DPP8FI | 带无效获取的 8 通道 DPP | — |
| 235 | SHARED_BASE | 内存孔径定义 | — |
| 236-238 | SHARED_LIMIT/PRIVATE_BASE/PRIVATE_LIMIT | | — |
| 240-248 | 浮点内联常量 0.5, -0.5, 1.0, -1.0, 2.0, -2.0, 4.0, -4.0, 1/(2π) | 浮点内联常量 | — |
| 250 | DPP16 | 数据并行原语（仅在 SRC0 有效） | — |
| 253 | SCC | `{31'b0, SCC}` | — |
| 255 | 字面常量 32 | 来自指令流的 32 位常量 | — |
| 256-511 | VGPR 0 .. 255 | 向量通用寄存器，每个 1 DWORD | ✓ |

### 7.2.1 操作数字段的非标准使用

V_{ADD,SUB,SUBREV}_CO_U32 系列、V_MAD_*_CO、V_DIV_SCALE、V_READLANE、V_READFIRSTLANE、V_WRITELANE、V_CMP*、V_CNDMASK 等指令对 VDST/SDST/VSRC 字段有特殊用法（如 carry-out 写入 SDST，或掩码值来自 VCC）。

读取通道选择（readlane lane-select）通过忽略通道号的高位限制在有效范围（wave32 为 0-31，wave64 为 0-63）。

### 7.2.2 输入操作数

- **VOP1, VOP2, VOPC**：SRC0 为 9 位，可为 VGPR、SGPR（含 TTMP 和 VCC）、M0、EXEC、内联或字面常量；VSRC1 为 8 位，只能是 VGPR
- **VOP3**：所有 3 个源都是 9 位，但仍有限制

**源操作数限制：**
- 指令最多可使用两个标量值（SGPR、VCC、M0、EXEC、SCC、字面常量）
- 所有指令格式（包括 VOP3 和 VOP3P）最多可使用一个字面常量
- 内联常量不计入两个标量值限制
- 字面常量不能与 DPP 一起使用

**输入修饰符 ABS 和 NEG：**
- 适用于浮点输入，对其他类型输入未定义
- 还支持：V_MOV_B32、V_MOV_B16、V_MOVREL*_B32 和 V_CNDMASK
- 不支持：READLANE、READFIRSTLANE、WRITELANE、整数算术或位操作、PERMLANE 等

### 7.2.3 输出操作数

VALU 指令通常将结果写入 VDST 字段指定的 VGPR。只有 EXEC 掩码中对应位为 1 的线程才写入结果。

- V_CMPX 指令将比较结果写入 EXEC 掩码
- V_CMP 和 V_CMPX 都写入完整掩码（wave32 为 32 位，wave64 为 64 位），非活跃通道写入零
- 产生进位输出的指令（整数加减）：VOP2 形式写入 VCC，VOP3 形式写入任意 SGPR 对

**输出修饰符 OMOD：** 适用于半精度、单精度和双精度浮点结果，将结果缩放 0.5、2.0、4.0 或不缩放。整数和打包浮点 16 位结果忽略 OMOD。

**CLAMP 位：**
- 对浮点运算：限制到 [0.0, 1.0]
- 对有符号整数：限制到 [INT_MIN, INT_MAX]
- 对无符号整数：限制到 [0, UINT_MAX]
- 对 V_CMP：设为 1 时表示发出信号比较
- CLAMP==1 时，任何 NaN 结果被限制到零

### 7.2.4 非规格化数和舍入模式

### 表 36. 舍入和非规格化模式

| 字段 | 位位置 | 描述 |
|------|--------|------|
| FP_ROUND | 3:0 | [1:0] 单精度舍入模式；[3:2] 双精度和半精度（FP16）舍入模式。0=最近偶数，1=+∞，2=-∞，3=向零 |
| FP_DENORM | 7:4 | [5:4] 单精度非规格化模式；[7:6] 双精度和半精度模式。0=刷新输入和输出非规格化数，1=允许输入，2=允许输出，3=全允许 |

---

## 7.3 指令列表

### VOP3（3 操作数）

`V_ADD3_U32`, `V_ADD_LSHL_U32`, `V_ALIGNBIT_B32`, `V_ALIGNBYTE_B32`, `V_AND_OR_B32`, `V_BFE_I32`, `V_BFE_U32`, `V_BFI_B32`, `V_BFM_B32`, `V_CNDMASK_B16`, `V_CVT_PK_*`, `V_DOT2_BF16_BF16`, `V_DOT2_F16_F16`, `V_FMA_DX9_ZERO_F32`, `V_FMA_F16/F32/F64`, `V_LERP_U8`, `V_LSHL_ADD_U32`, `V_LSHL_OR_B32`, `V_MAD_CO_I64_I32`, `V_MAD_CO_U64_U32`, `V_MAD_I16/U16/I32_I16/U32_U16/U32_U24`, `V_MAX3_*`, `V_MAXIMUM3_*`, `V_MAXIMUMMINIMUM_*`, `V_MAXMIN_*`, `V_MED3_*`, `V_MIN3_*`, `V_MINIMUM3_*`, `V_MINIMUMMAXIMUM_*`, `V_MINMAX_*`, `V_MQSAD_*`, `V_MSAD_U8`, `V_MULLIT_F32`, `V_OR3_B32`, `V_PERMLANE16_B32`, `V_PERMLANEX16_B32`, `V_PERM_B32`, `V_QSAD_PK_U16_U8`, `V_SAD_*`, `V_XAD_U32`, `V_XOR3_B32` 等

### VOP3（2 操作数）

`V_ADD_CO_U32/CI_U32`, `V_ADD_NC_I16/I32/U16/U32`, `V_AND_B16/B32`, `V_ASHRREV_I16/I32/I64`, `V_BCNT_U32_B32`, `V_LDEXP_F32/F64`, `V_LSHLREV_B16/B32/B64`, `V_LSHRREV_B16/B32/B64`, `V_MAXIMUM_F16/F32/F64`, `V_MAX_I16/U16`, `V_MBCNT_HI/LO_U32_B32`, `V_MIN_I16/U16`, `V_MINIMUM_F16/F32/F64`, `V_MUL_HI_I32/U32`, `V_MUL_LO_U16/U32`, `V_OR_B16/B32`, `V_PERMLANE16/X16_VAR_B32`, `V_READLANE_B32`, `V_SUBREV_CO_CI/CO_U32`, `V_SUB_CO_CI/CO_U32`, `V_TRIG_PREOP_F64`, `V_WRITELANE_B32`, `V_XOR_B16/B32` 等

### VOP2

`V_ADD_F16/F32`, `V_ADD_NC_U32`, `V_AND_B32`, `V_ASHRREV_I32`, `V_CNDMASK_B32`, `V_FMAAK_F16/F32`, `V_FMAC_F16/F32`, `V_FMAMK_F16/F32`, `V_LSHLREV_B32`, `V_LSHRREV_B32`, `V_MAX_NUM_F16/F32/F64`, `V_MAX_U32`, `V_MIN_NUM_F16/F32/F64`, `V_MIN_U32`, `V_MUL_DX9_ZERO_F32`, `V_MUL_F16/F32/F64`, `V_MUL_HI_I32_I24/U32_U24`, `V_MUL_I32_I24/U32_U24`, `V_OR_B32`, `V_PK_FMAC_F16`, `V_SUB_F16/F32`, `V_SUBREV_F16/F32/NC_U32`, `V_SUB_NC_U32`, `V_XNOR_B32`, `V_XOR_B32` 等

### VOP1（单操作数）

`V_BFREV_B32`, `V_CEIL_F16/F32/F64`, `V_CLZ_I32_U32`, `V_COS_F16/F32`, `V_CTZ_I32_B32`, `V_CUBEID/MA/SC/TC_F32`, `V_CVT_F16_*`, `V_CVT_F32_*`, `V_CVT_F64_*`, `V_CVT_I32_*`, `V_CVT_U32_*`, `V_EXP_F16/F32`, `V_FLOOR_F16/F32/F64`, `V_FRACT_F16/F32/F64`, `V_FREXP_*`, `V_LOG_F16/F32`, `V_MOV_B16/B32`, `V_MOVRELD/MOVRELS/MOVRELSD_B32`, `V_NOP`, `V_NOT_B16/B32`, `V_PERMLANE64_B32`, `V_PIPEFLUSH`, `V_RCP_F16/F32/F64/IFLAG_F32`, `V_READFIRSTLANE_B32`, `V_RNDNE_F16/F32/F64`, `V_RSQ_F16/F32/F64`, `V_SAT_PK_U8_I16`, `V_SIN_F16/F32`, `V_SQRT_F16/F32/F64`, `V_SWAP_B16/B32`, `V_TRUNC_F16/F32/F64`

### VOPC 比较操作

```
V_CMP  整数（I16/I32/I64/U16/U32/U64）：F, LT, EQ, LE, GT, LG, GE, T → 写入 VCC
V_CMPX                                                                  → 写入 EXEC
V_CMP  浮点（F16/F32/F64）：F, LT, EQ, LE, GT, LG, GE, T, O, U, NGE, NLG, NGT, NLE, NEQ, NLT → 写入 VCC
V_CMPX                                                                                           → 写入 EXEC
V_CMP_CLASS（F16/F32/F64）：测试 signaling-NaN, quiet-NaN, ±∞, normal, subnormal, zero 的任意组合 → 写入 VCC
V_CMPX_CLASS                                                                                      → 写入 EXEC
```

---

## 7.4 16 位数学和 VGPR

16 位 VGPR 对打包到 32 位 VGPR 中：32 位 VGPR "V0" 包含两个 16 位 VGPR：`V0.L`（V0[15:0]）和 `V0.H`（V0[31:16]）。

**VOP1/2/C 编码中 16 位 VGPR：**
- SRC/DST[6:0] = 32 位 VGPR 地址；SRC/DST[7] = 高/低半部
- 仅能寻址 256 个 16 位 VGPR

**VOP3/VOP3P/VINTERP 编码中 16 位 VGPR：**
- SRC/DST[7:0] = 32 位 VGPR 地址，OPSEL = 高/低
- 可寻址 512 个 16 位 VGPR

---

## 7.5 8 位数学

8 位浮点值有两种格式：
- **FP8**：{Sign, Exp4, Mant3}；ExpBias = 7
- **BF8**：{Sign, Exp5, Mant2}；ExpBias = 15

支持 DOT、WMMA 和 CVT 操作，但 ISA 不支持在 VGPR 中直接寻址单个 8 位量。

---

## 7.6 数据转换操作

**通用限制：**
- 不支持输入修饰符（neg、abs）
- 舍入为 RNE（"SR"操作除外，截断）
- 不使用 OMOD
- BF8/FP8 或更小浮点的转换不支持 CLAMP
- 目标为 FP16 或 BF16 时应用 FP16_OVFL

**转换类型：**
- 普通/单值：将源寄存器的一个值转换为目标寄存器的一个值
- 打包（"PK"）：将源寄存器的两个连续值转换到目标寄存器
- 随机舍入：向每个源值添加伪随机值（每通道唯一），然后截断到较小精度

---

## 7.7 打包数学

打包数学对同一 VGPR 中打包的两个值加速算术。对 DWORD 中两个 16 位值进行操作，就像它们是独立线程一样。使用 VOP3P 微码格式，该格式对高低操作数各有 OPSEL 和 NEG 字段，无 ABS 和 OMOD。

### 表 37. 打包数学操作码

`V_PK_MUL_F16`, `V_PK_FMA_F16`, `V_PK_ADD_F16`, `V_PK_FMAC_F16`, `V_PK_MIN_NUM_F16`, `V_PK_MAX_NUM_F16`, `V_PK_MINIMUM_F16`, `V_PK_MAXIMUM_F16`, `V_PK_ADD_I16`, `V_PK_MIN_I16`, `V_PK_MAD_I16`, `V_PK_ADD_U16`, `V_PK_MIN_U16`, `V_PK_MAD_U16`, `V_PK_SUB_I16`, `V_PK_MAX_I16`, `V_PK_MUL_LO_U16`, `V_PK_SUB_U16`, `V_PK_MAX_U16`, `V_FMA_MIX_F32`, `V_FMA_MIXHI_F16`, `V_FMA_MIXLO_F16`, `V_PK_LSHLREV_B16`, `V_PK_LSHRREV_B16`, `V_PK_ASHRREV_I16`, `V_DOT2_F32_BF16`, `V_DOT2_F32_F16`, `V_DOT4_F32_FP8_BF8`, `V_DOT4_F32_BF8_FP8`, `V_DOT4_F32_FP8_FP8`, `V_DOT4_F32_BF8_BF8`, `V_DOT4_I32_IU8`, `V_DOT4_U32_U8`, `V_DOT8_I32_IU4`, `V_DOT8_U32_U4`

### VOP3P 字段

| 字段 | 大小 | 描述 |
|------|------|------|
| OP | 7 | 指令操作码 |
| SRC0/1/2 | 9 | 第1/2/3操作数 |
| VDST | 8 | 接收结果的 VGPR |
| NEG | 3 | 对低 16 位操作数取反（仅浮点）；IU 类型中重新用于指示有符号/无符号 |
| NEG_HI | 3 | 对高 16 位操作数取反（仅浮点）；MIX 指令中用作 ABS |
| OPSEL | 3 | 选择低/高操作数作为生成低半部结果的输入 |
| OPSEL_HI | 3 | 选择低/高操作数作为生成高半部结果的输入 |
| CM | 1 | 限制结果 |

---

## 7.8 双发射 VALU（VOPD）

VOPD 指令编码允许单条着色器指令编码两个并行执行的独立 VALU 操作。两个操作必须相互独立。**此指令格式仅适用于 wave32，不得用于 wave64。**

两个操作分别称为"X"和"Y"：
- OpcodeX：从 SRC0X（VGPR、SGPR 或常量）和 SRC1X（VGPR）获取数据
- OpcodeY：从 SRC0Y（VGPR、SGPR 或常量）和 SRC1Y（VGPR）获取数据

**限制：**
- 每条指令最多使用 2 个 VGPR
- 每条指令最多使用 1 个 SGPR，或共享同一个字面常量
- SRC0 可为 VGPR 或 SGPR（或常量）；VSRC1 只能为 VGPR
- SRC0x 和 SRC0y 必须来自不同 VGPR bank（bank# = SRC % 4）
- 目标 VGPR：一个必须为偶数，另一个为奇数
- 两条指令必须相互独立（OPX 不能覆盖 OPY 的源，OPY 可以覆盖 OPX 的源）
- 不能使用 DPP；必须是 wave32

**VOPD 操作码（OPX/OPY）：**
`V_DUAL_FMAC_F32`, `V_DUAL_FMAAK_F32`, `V_DUAL_FMAMK_F32`, `V_DUAL_MUL_F32`, `V_DUAL_ADD_F32`, `V_DUAL_SUB_F32`, `V_DUAL_SUBREV_F32`, `V_DUAL_MUL_DX9_ZERO_F32`, `V_DUAL_MOV_B32`, `V_DUAL_CNDMASK_B32`, `V_DUAL_MAX_NUM_F32`, `V_DUAL_MIN_NUM_F32`, `V_DUAL_DOT2ACC_F32_F16`, `V_DUAL_DOT2ACC_F32_BF16`；OPY 还有：`V_DUAL_ADD_NC_U32`, `V_DUAL_LSHLREV_B32`, `V_DUAL_AND_B32`

---

## 7.9 跨通道和数据并行处理（DPP）

VALU 提供多种功能在 wave 的不同通道之间传输数据：

| 指令 | 功能 |
|------|------|
| V_PERM_B32 | 64 位源数据内字节混洗（每通道唯一，非跨通道） |
| V_PERMLANE16_B32 | 16 通道组内任意聚集混洗（0-15、16-31 等），统一混洗控制 |
| V_PERMLANE16_VAR_B32 | 同上，但每通道唯一通道选择 |
| V_PERMLANEX16_B32 | 访问相反 16 通道组：0-15 读 16-31，16-31 读 0-15 |
| V_PERMLANEX16_VAR_B32 | 同上，但每通道唯一通道选择 |
| V_PERMLANE64_B32 | 交换上 32 通道和下 32 通道，wave32 时为 NOP |
| DPP8 | 带任意 8 通道组内混洗的 ALU 指令 |
| DPP16 | 带 16 通道组内预定义菜单混洗的 ALU 指令 |
| DS_SWIZZLE | LDS 操作：在固定菜单的 32 通道组内混洗 |
| DS_PERMUTE / BPERMUTE | LDS 操作：在所有通道间正向/反向排列 |

DPP 操作通过在 SRC0 操作数中使用内联常量 DPP8、DPP8FI 或 DPP16 来指定。

### DPP 支持情况（表 38）

| 编码 | 操作码 | 规则 |
|------|--------|------|
| VOP1 | 64 位操作码、READFIRSTLANE_B32、SWAP、V_NOP、PERMLANE | 不支持 DPP |
| VOP1 | 其他 | 支持 DPP |
| VOP2 | 64 位操作码、FMAMK/AK_F32/F16 | 不支持 DPP |
| VOP2 | 其他 | 支持 DPP |
| VOP3P | V_DOT4_I32_IU8/U32_U8、V_DOT8_*、V_PK_*、WMMA | 不支持 DPP |
| VOP3P | V_FMA_MIX* | 支持 DPP |
| VOP3 | 64 位操作码、CVT_PK_F32_*、MUL_LO/HI、QSAD、MQSAD、READLANE*、WRITELANE*、PERMLANE*、SWAP* | 不支持 DPP |
| VOP3 | 其他 | 支持 DPP |
| VINTERP/VOPD | 所有 | 不支持 DPP |

### 7.9.1 DPP16

DPP16 允许在 16 通道组内使用固定混洗模式选择数据。

**DPP16 DWORD 字段：**

| 字段 | 位 | 描述 |
|------|-----|------|
| row_mask | 31:28 | 适用于 VGPR 目标写入，bit 28-31 分别对应通道 0-15、16-31、32-47、48-63 |
| bank_mask | 27:24 | 适用于 VGPR 目标写入，按 bank 禁用通道 |
| src1_imod | 23:22 | bit23=ABS，bit22=NEG（NEG 在 ABS 之后） |
| src0_imod | 21:20 | bit21=ABS，bit20=NEG |
| BC | 19 | 边界控制：0=源无效时禁用写入；1=源无效时使用零 |
| FI | 18 | 获取非活跃通道：0=依靠 BC；1=从禁用通道获取数据 |
| dpp_ctrl | 16:8 | 混洗控制：QUAD_PERM(000-0FF)、ROW_SL(1-15)、ROW_SR(1-15)、ROW_RR(1-15)、ROW_MIRROR(140)、ROW_HALF_MIRROR(141)、ROW_SHARE(0-15)、ROW_XMASK(0-15) |
| Src0 | 7:0 | SRC0 的 VGPR 地址 |

### BC 和 FI 行为（表 39）

| BC | FI | 源通道超出范围 | 源通道在范围内但禁用 | 源通道活跃 |
|----|-----|--------------|---------------------|-----------|
| 0 | 0 | 禁用写入 | 禁用写入 | 正常 |
| 1 | 0 | Src0=0 | Src0=0 | 正常 |
| 0 | 1 | Src0=0 | 正常 | 正常 |
| 1 | 1 | 正常 | 正常 | 正常 |

### 7.9.2 DPP8

DPP8 允许在 8 通道组内任意跨通道混洗。有两种形式：普通（从 EXEC 掩码位为零的通道读取零）和 DPP8FI（从非活跃通道获取数据而非使用零）。

**DPP8 字段：**
- **SRC**（8 位）：SRC0 VGPR 地址（VOP1/2 SRC0 槽被 DPP 常量占用）
- **SEL0-SEL7**（各 3 位）：选择每个通道（0-7）从哪个通道（0-7）拉取数据

---

## 7.10 伪标量超越 ALU 操作

这是一组在单个通道上操作的 VALU 指令，源和目标都是 SGPR，使用 VOP3 格式编码：

| 操作 | F32 | F16 |
|------|-----|-----|
| 指数 | V_S_EXP_F32 | V_S_EXP_F16 |
| 对数 | V_S_LOG_F32 | V_S_LOG_F16 |
| 倒数 | V_S_RCP_F32 | V_S_RCP_F16 |
| 倒数平方根 | V_S_RSQ_F32 | V_S_RSQ_F16 |
| 平方根 | V_S_SQRT_F32 | V_S_SQRT_F16 |

注意：
- 16 位数据不支持半 SGPR，数据在 bits[15:0]，完整 32 位写入，高半部为零
- EXEC 值被忽略，即使 EXEC==0 时指令也执行
- VCC 不能作为目标
- 目标在 VDST 字段中指定（不在 SDST 字段）

---

## 7.11 VGPR 索引

VALU 提供一组通过 M0 寄存器值对源、目标或两者进行索引的移动/交换 VGPR 指令：

### 表 40. VGPR 索引指令

| 指令 | 索引 | 功能 |
|------|------|------|
| V_MOVRELD_B32 | M0[31:0] | 相对目标移动：`VGPR[dst + M0] = VGPR[src]` |
| V_MOVRELS_B32 | M0[31:0] | 相对源移动：`VGPR[dst] = VGPR[src + M0]` |
| V_MOVRELSD_B32 | M0[31:0] | 相对源和目标移动：`VGPR[dst + M0] = VGPR[src + M0]` |
| V_MOVRELSD_2_B32 | 源 M0[9:0]，目标 M0[25:16] | 双独立相对移动 |
| V_SWAPREL_B32 | — | 交换两个 VGPR（各自相对独立索引） |

---

## 7.12 Wave 矩阵乘法累加（WMMA）

WMMA 指令为矩阵乘法运算提供加速。每条 WMMA 或 SWMMAC（稀疏 WMMA）指令执行一次矩阵乘法，VGPRs 中分别持有 A、B、C 和 D 矩阵。矩阵条带化分布在所有通道上，不是每通道一个矩阵。使用 VOP3P 编码。

执行：**A × B + C ⇒ D**

SWMMAC 指令使用稀疏矩阵作为矩阵 A（矩阵 B 必须为密集）。稀疏矩阵通过要求每 4 个元素中有 2 个为零来节省一半存储空间。

### 表 41. WMMA 指令

| 指令 | 矩阵 A | 矩阵 B | 矩阵 C | 结果矩阵 |
|------|--------|--------|--------|---------|
| V_WMMA_F32_16X16X16_F16 | 16×16 F16 | 16×16 F16 | 16×16 F32 | 16×16 F32 |
| V_WMMA_F32_16X16X16_BF16 | 16×16 BF16 | 16×16 BF16 | 16×16 F32 | 16×16 F32 |
| V_WMMA_F16_16X16X16_F16 | 16×16 F16 | 16×16 F16 | 16×16 F16 | 16×16 F16 |
| V_WMMA_BF16_16X16X16_BF16 | 16×16 BF16 | 16×16 BF16 | 16×16 BF16 | 16×16 BF16 |
| V_WMMA_I32_16X16X16_IU8 | 16×16 IU8 | 16×16 IU8 | 16×16 I32 | 16×16 I32 |
| V_WMMA_I32_16X16X16_IU4 | 16×16 IU4 | 16×16 IU4 | 16×16 I32 | 16×16 I32 |
| V_WMMA_I32_16X16X32_IU4 | 16×32 IU4 | 32×16 IU4 | 16×16 I32 | 16×16 I32 |
| V_WMMA_F32_16X16X16_FP8_FP8 | 16×16 FP8 | 16×16 FP8 | 16×16 F32 | 16×16 F32 |
| V_WMMA_F32_16X16X16_FP8_BF8 | 16×16 FP8 | 16×16 BF8 | 16×16 F32 | 16×16 F32 |
| V_WMMA_F32_16X16X16_BF8_FP8 | 16×16 BF8 | 16×16 FP8 | 16×16 F32 | 16×16 F32 |
| V_WMMA_F32_16X16X16_BF8_BF8 | 16×16 BF8 | 16×16 BF8 | 16×16 F32 | 16×16 F32 |
| **稀疏矩阵操作（A 矩阵大小为稀疏展开后）** | | | | |
| V_SWMMAC_F32_16X16X32_F16 | 16×32 F16 | 32×16 F16 | 16×16 F32 | 16×16 F32 |
| V_SWMMAC_F32_16X16X32_BF16 | 16×32 BF16 | 32×16 BF16 | 16×16 F32 | 16×16 F32 |
| V_SWMMAC_F16_16X16X32_F16 | 16×32 F16 | 32×16 F16 | 16×16 F16 | 16×16 F16 |
| V_SWMMAC_BF16_16X16X32_BF16 | 16×32 BF16 | 32×16 BF16 | 16×16 BF16 | 16×16 BF16 |
| V_SWMMAC_I32_16X16X32_IU8 | 16×32 IU8 | 32×16 IU8 | 16×16 I32 | 16×16 I32 |
| V_SWMMAC_I32_16X16X32_IU4 | 16×32 IU4 | 32×16 IU4 | 16×16 I32 | 16×16 I32 |
| V_SWMMAC_I32_16X16X64_IU4 | 16×64 IU4 | 64×16 IU4 | 16×16 I32 | 16×16 I32 |
| V_SWMMAC_F32_16X16X32_FP8_FP8 | 16×32 FP8 | 32×16 FP8 | 16×16 F32 | 16×16 F32 |
| V_SWMMAC_F32_16X16X32_FP8_BF8 | 16×32 FP8 | 32×16 BF8 | 16×16 F32 | 16×16 F32 |
| V_SWMMAC_F32_16X16X32_BF8_FP8 | 16×32 BF8 | 32×16 FP8 | 16×16 F32 | 16×16 F32 |
| V_SWMMAC_F32_16X16X32_BF8_BF8 | 16×32 BF8 | 32×16 BF8 | 16×16 F32 | 16×16 F32 |

**NEG 字段用法：**
- IU8/IU4 类型：NEG[0] 表示 A 矩阵有符号/无符号，NEG[1] 表示 B 矩阵
- F16/BF16 类型：NEG[1:0] 用于 SRC0/SRC1 低 16 位；NEG_HI[1:0] 用于高 16 位；{NEG_HI[2], NEG[2]} 用于 SRC2（作为 {ABS, NEG}）

### 表 42. ISA 操作数字段

| 指令类型 | 操作数 | 含义 |
|----------|--------|------|
| WMMA / SWMMAC | SRC0 | A 矩阵 |
| WMMA / SWMMAC | SRC1 | B 矩阵 |
| WMMA | SRC2 | C 矩阵 |
| SWMMAC | SRC2 | 稀疏索引数据 |
| WMMA | VDST | D 矩阵 |
| SWMMAC | VDST | C 矩阵和 D 矩阵（SWMMAC 读取 VDST 并累加到其中） |

### 7.12.1 WMMA 数据冒险要求

| 第一条指令 | 第二条指令 | 两条指令之间的要求 |
|-----------|-----------|------------------|
| WMMA | WMMA（A/B/索引与前一 WMMA 的 D 矩阵相同或重叠） | 至少 1 条 V_NOP 或独立 VALU 指令 |
| WMMA | WMMA（使用前一 WMMA 的 D 矩阵 VGPR 作为 C 矩阵） | 若类型不同或对 SRC2 使用 ABS/NEG，则停顿 |
| WMMA | WMMA（与前一 WMMA D 矩阵 VGPR 重叠的 C 矩阵） | 硬件可能停顿 |
| WMMA | VALU（读取前一 WMMA 的 D 矩阵） | 硬件可能停顿 |
| WMMA | WMMA（读取 VALU 结果作为 A/B/C） | 硬件可能停顿 |

### 7.12.2 矩阵元素在 VGPR 中的存储

矩阵元素存储规则：A 矩阵每行条带化分布在一个通道的多个 VGPR 中；B、C 和 D 矩阵每行条带化分布在一个 VGPR 的多个通道中。

映射公式：`Memory[row][col] → VGPR[lane][vgpr][startPosn*dataSize + dataSize-1 : startPosn*dataSize]`

### 表 43. 矩阵大小和 VGPR 使用量

| 数据大小 | Wave 大小 | A 矩阵（密集） | 密集 A、B VGPR 数 | 稀疏 VGPR 数 |
|----------|-----------|----------------|------------------|-------------|
| 16 位 | 32 | 16×16 | A=B=4 | A=4, B=8, S=0.5 |
| 16 位 | 64 | 16×16 | A=B=2 | A=2, B=4, S=0.25 |
| 8 位 | 32 | 16×16 | A=B=2 | A=2, B=4, S=0.5 |
| 8 位 | 64 | 16×16 | A=B=1 | A=1, B=2, S=0.25 |
