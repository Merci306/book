# 第六章：标量 ALU 操作

标量 ALU（SALU）指令对 wave 中所有工作项共用的值进行操作。这些操作包括 32 位整数或浮点算术，以及 32 位或 64 位位操作。SALU 还可以直接对程序计数器执行操作，允许程序在 SGPR 中创建调用栈。

许多操作还设置标量条件码位（SCC），指示比较结果、进位输出或指令结果是否为零。

---

## 6.1 SALU 指令格式

SALU 指令以以下五种微码格式之一编码：

| 名称 | 大小 | 功能 |
|------|------|------|
| SOP1 | 32 位 | 带 1 个输入的 SALU 操作 |
| SOP2 | 32 位 | 带 2 个输入的 SALU 操作 |
| SOPK | 32 位 | 带 1 个有符号 16 位整数常量输入的 SALU 操作 |
| SOPC | 32 位 | SALU 比较操作 |
| SOPP | 32 位 | SALU 程序控制操作 |

指令格式字段：

| 字段 | 描述 |
|------|------|
| OP | 操作码：要执行的指令 |
| SDST | 目标 SGPR、M0、NULL 或 EXEC |
| SSRC0 | 第一个源操作数 |
| SSRC1 | 第二个源操作数 |
| SIMM16 | 有符号 16 位整数立即常量 |

---

## 6.2 标量 ALU 操作数

**表 28. 标量操作数**（0-127 可用作源或目标，128-255 只能用作源）

| 代码 | 含义 |
|------|------|
| 0-105 | SGPR 0 .. 105，标量通用寄存器，每个 1 DWORD |
| 106 | VCC_LO，VCC[31:0] |
| 107 | VCC_HI，VCC[63:32] |
| 108-123 | TTMP0 .. TTMP15，陷阱处理程序临时 SGPR（特权） |
| 124 | NULL，读取返回零，写入被忽略。用作 SALU 目标时使指令无效化 |
| 125 | M0，临时寄存器，用于多种功能 |
| 126 | EXEC_LO，EXEC[31:0] |
| 127 | EXEC_HI，EXEC[63:32] |
| 128 | 整数内联常量 0 |
| 129-192 | 整数内联常量 1 .. 64 |
| 193-208 | 整数内联常量 -1 .. -16 |
| 235 | SHARED_BASE，内存孔径定义 |
| 236 | SHARED_LIMIT |
| 237 | PRIVATE_BASE |
| 238 | PRIVATE_LIMIT |
| 240 | 0.5，浮点内联常量（可用于 16/32/64 位浮点运算） |
| 241-248 | -0.5, 1.0, -1.0, 2.0, -2.0, 4.0, -4.0, 1.0/(2*PI) |
| 253 | SCC，`{31'b0, SCC}` |
| 255 | 字面常量 32，来自指令流的 32 位常量 |

SALU 目标在 0-127 范围内。SALU 指令可使用 32 位或 64 位字面常量（SOPP 和 SOPK 除外，但 `S_SETREG_IMM32_B32` 中允许字面量）。字面常量通过将源指令字段设置为"literal"（255）来使用，然后使用后续指令 DWORD 作为源值。

若目标 SGPR 越界，则不写入任何 SGPR 且不更新 SCC。若指令在 SGPR 中使用 64 位数据，则 SGPR 对必须对齐到偶数边界。

---

## 6.3 标量条件码（SCC）

SCC 由大多数 SALU 指令执行结果写入，用于扩展整数算术中的进位/借位输入。

SCC 设置规则：
- **比较操作**：1 = 真
- **算术操作**：1 = 进位输出。有符号加减的 SCC = 溢出（对于加法：两个操作数符号相同且结果 MSB 与操作数符号不同；对于减法 A-B：A 和 B 符号相反且结果符号与 A 的符号不同）
- **位/逻辑操作**：1 = 结果不为零

---

## 6.4 整数算术指令

**表 29. 整数算术指令**

| 指令 | 编码 | 设置 SCC？ | 操作 |
|------|------|-----------|------|
| S_ADD_CO_I32 | SOP2 | 溢出 | `D = S0 + S1, SCC = overflow` |
| S_ADD_CO_U32 | SOP2 | 进位 | `D = S0 + S1, SCC = carry out` |
| S_ADD_U64 | SOP2 | 否 | `D = S0 + S1` |
| S_SUB_U64 | SOP2 | 否 | `D = S0 - S1` |
| S_ADD_CO_CI_U32 | SOP2 | 进位 | `D = S0 + S1 + SCC, SCC = overflow` |
| S_SUB_CO_I32 | SOP2 | 溢出 | `D = S0 - S1, SCC = overflow` |
| S_SUB_CO_U32 | SOP2 | 进位 | `D = S0 - S1, SCC = carry out` |
| S_SUB_CO_CI_U32 | SOP2 | 进位 | `D = S0 - S1 - SCC, SCC = carry out` |
| S_ADD_LSH{1,2,3,4}_U32 | SOP2 | D!=0 | `D = S0 + (S1 << {1,2,3,4})` |
| S_ABSDIFF_I32 | SOP2 | D!=0 | `D = abs(S0 - S1), SCC = result not zero` |
| S_MIN_I32 / S_MIN_U32 | SOP2 | D!=0 | `D = (S0 < S1) ? S0 : S1; SCC = (S0 < S1)` |
| S_MAX_I32 / S_MAX_U32 | SOP2 | D!=0 | `D = (S0 > S1) ? S0 : S1; SCC = (S0 > S1)` |
| S_MUL_I32 | SOP2 | 否 | `D = S0 * S1`（低 32 位结果，有符号/无符号相同） |
| S_MUL_U64 | SOP2 | 否 | `D = S0 * S1` |
| S_ADDK_I32 | SOPK | 溢出 | `D = D + SIMM16, SCC = overflow` |
| S_MULK_I32 | SOPK | 否 | `D = D * SIMM16`，返回低 32 位，SIMM16 符号扩展 |
| S_ABS_I32 | SOP1 | D!=0 | `D.i = abs(S0.i), SCC = result not zero` |
| S_SEXT_I32_I8 | SOP1 | 否 | `D = {24{S0[7]}, S0[7:0]}` |
| S_SEXT_I32_I16 | SOP1 | 否 | `D = {16{S0[15]}, S0[15:0]}` |
| S_MUL_HI_I32 | SOP2 | 否 | `D = S0 * S1`（高 32 位有符号结果） |
| S_MUL_HI_U32 | SOP2 | 否 | `D = S0 * S1`（高 32 位无符号结果） |
| S_PACK_LL_B32_B16 | SOP2 | 否 | `D = {S1[15:0], S0[15:0]}` |
| S_PACK_LH_B32_B16 | SOP2 | 否 | `D = {S1[31:16], S0[15:0]}` |
| S_PACK_HL_B32_B16 | SOP2 | 否 | `D = {S1[15:0], S0[31:16]}` |
| S_PACK_HH_B32_B16 | SOP2 | 否 | `D = {S1[31:16], S0[31:16]}` |

---

## 6.5 条件移动指令

条件指令使用 SCC 标志决定是否执行操作，或（对于 CSELECT）使用哪个源操作数。

**表 30. 条件指令**

| 指令 | 编码 | 设置 SCC？ | 操作 |
|------|------|-----------|------|
| S_CSELECT_{B32, B64} | SOP2 | 否 | `D = SCC ? S0 : S1` |
| S_CMOVK_I32 | SOPK | 否 | `if (SCC) D = signext(SIMM16); else NOP` |
| S_CMOV_{B32, B64} | SOP1 | 否 | `if (SCC) D = S0; else NOP` |

---

## 6.6 比较指令

这些指令比较两个值，若比较结果为 TRUE 则设置 SCC = 1。

**表 31. 比较指令**

| 指令 | 编码 | 操作 |
|------|------|------|
| S_CMP_EQ_U64, S_CMP_LG_U64 | SOPC | 比较两个 64 位源值，`SCC = S0 <cond> S1` |
| S_CMP_{EQ,LG,GT,GE,LE,LT}_{I32,U32} | SOPC | 比较两个源值，`SCC = S0 <cond> S1` |
| S_BITCMP0_{B32,B64} | SOPC | 测试某位是否为零，`SCC = !S0[S1]` |
| S_BITCMP1_{B32,B64} | SOPC | 测试某位是否为一，`SCC = S0[S1]` |

---

## 6.7 位操作指令

位操作指令对 32 位或 64 位数据进行操作，不解释其类型。若下表所注，结果不为零时设置 SCC。

**表 32. 位操作指令**

| 指令 | 编码 | 设置 SCC？ | 操作 |
|------|------|-----------|------|
| S_MOV_{B32,B64} | SOP1 | 否 | `D = S0` |
| S_MOVK_I32 | SOPK | 否 | `D = signext(SIMM16)` |
| S_{AND,OR,XOR}_{B32,B64} | SOP2 | D!=0 | `D = S0 & S1, S0 \| S1, S0 ^ S1` |
| S_{AND_NOT1,OR_NOT1}_{B32,B64} | SOP2 | D!=0 | `D = S0 & ~S1, S0 \| ~S1` |
| S_{NAND,NOR,XNOR}_{B32,B64} | SOP2 | D!=0 | `D = ~(S0 & S1), ~(S0 \| S1), ~(S0 ^ S1)` |
| S_LSHL_{B32,B64} | SOP2 | D!=0 | `D = S0 << S1[4:0]`（B64 使用 [5:0]） |
| S_LSHR_{B32,B64} | SOP2 | D!=0 | `D = S0 >> S1[4:0]`（B64 使用 [5:0]） |
| S_ASHR_{I32,I64} | SOP2 | D!=0 | `D = signext(S0 >> S1[4:0])`（I64 使用 [5:0]） |
| S_BFM_{B32,B64} | SOP2 | 否 | 位字段掩码：`D = ((1 << S0[4:0]) - 1) << S1[4:0]` |
| S_BFE_{U32,U64,I32,I64} | SOP2 | D!=0 | 位字段提取，有符号版本符号扩展结果。S0=数据，S1[22:16]=宽度，S1[4:0]/[5:0]=偏移 |
| S_NOT_{B32,B64} | SOP1 | D!=0 | `D = ~S0` |
| S_WQM_{B32,B64} | SOP1 | D!=0 | Whole Quad Mode：每 4 位组，若任意位为 1 则将结果设为 1111 |
| S_QUADMASK_{B32,B64} | SOP1 | D!=0 | 从每像素 1 位掩码创建每四元组 1 位掩码（32 位→8 位，64 位→16 位） |
| S_BITREPLICATE_B64_B32 | SOP1 | 否 | 复制 32 位 S0 中的每位：`D = {...,S0[1],S0[1],S0[0],S0[0]}`（两个此指令是 S_QUADMASK 的逆操作） |
| S_BREV_{B32,B64} | SOP1 | 否 | `D[31:0] = S0[0:31]`，位反转 |
| S_BCNT0_I32_{B32,B64} | SOP1 | D!=0 | `D = CountZeroBits(S0)` |
| S_BCNT1_I32_{B32,B64} | SOP1 | D!=0 | `D = CountOneBits(S0)` |
| S_CTZ_I32_{B32,B64} | SOP1 | 否 | 统计尾随零：从 LSB 开始找第一个 1 的位位置，未找到返回 -1 |
| S_CLZ_I32_{B32,B64} | SOP1 | 否 | 统计前导零：从 MSB 开始第一个 1 之前有多少个零，未找到返回 -1 |
| S_CLS_I32_{B32,B64} | SOP1 | 否 | 统计前导符号位：从 MSB 到 LSB 与符号位相同的连续位数。输入为 0 或全 1（-1）时返回 -1 |
| S_BITSET0_{B32,B64} | SOP1 | 否 | `D[S0[4:0]] = 0`（B64 使用 [5:0]） |
| S_BITSET1_{B32,B64} | SOP1 | 否 | `D[S0[4:0]] = 1`（B64 使用 [5:0]） |
| S_{AND,OR,XOR,...}_SAVEEXEC_{B32,B64} | SOP1 | D!=0 | 保存 EXEC 掩码，然后对其应用位操作：`D = EXEC; EXEC = S0 <op> EXEC; SCC = (EXEC != 0)` |
| S_{AND_NOT{0,1}_WREXEC_B{32,64}} | SOP1 | D!=0 | NOT0: `EXEC = D = ~S0 & EXEC`；NOT1: `EXEC = D = S0 & ~EXEC`。D 和 EXEC 得到相同结果，D 不能是 EXEC |
| S_MOVRELS_{B32,B64} | SOP1 | 否 | `D = SGPR[S0+M0]`，将 M0 相对 SGPR 的值移入 D |
| S_MOVRELD_{B32,B64} | SOP1 | 否 | `SGPR[D+M0] = S0`，将值移入 M0 相对的 SGPR |

---

## 6.8 SALU 浮点

SALU 支持一组浮点操作，用于从 VALU 管线卸载统一值计算。标量浮点指令与 VALU 对应指令产生相同结果，但有一些限制：标量指令不支持操作数修饰符，编译器可用额外指令模拟这些修饰符。

**标量 F32 算术指令：** S_ADD_F32、S_SUB_F32、S_MIN_NUM_F32、S_MAX_NUM_F32、S_MINIMUM_F32、S_MAXIMUM_F32、S_MUL_F32、S_FMAC_F32、S_CEIL_F32、S_FLOOR_F32、S_TRUNC_F32、S_RNDNE_F32、S_FMAAK_F32、S_FMAMK_F32

**标量 F32 比较指令：** S_CMP_LT_F32、S_CMP_EQ_F32、S_CMP_LE_F32、S_CMP_GT_F32、S_CMP_LG_F32、S_CMP_GE_F32、S_CMP_O_F32、S_CMP_U_F32、S_CMP_NGE_F32、S_CMP_NLG_F32、S_CMP_NGT_F32、S_CMP_NLE_F32、S_CMP_NEQ_F32、S_CMP_NLT_F32

**标量浮点转换指令：** S_CVT_F32_I32、S_CVT_F32_U32、S_CVT_I32_F32、S_CVT_U32_F32、S_CVT_F16_F32、S_CVT_PK_RTZ_F16_F32、S_CVT_F32_F16、S_CVT_HI_F32_F16

**标量 F16 算术指令：** S_ADD_F16、S_SUB_F16、S_MIN_NUM_F16、S_MAX_NUM_F16、S_MINIMUM_F16、S_MAXIMUM_F16、S_FMAC_F16、S_MUL_F16、S_CEIL_F16、S_FLOOR_F16、S_TRUNC_F16、S_RNDNE_F16

**标量 F16 比较指令：** S_CMP_LT_F16、S_CMP_EQ_F16、S_CMP_LE_F16、S_CMP_GT_F16、S_CMP_LG_F16、S_CMP_GE_F16、S_CMP_O_F16、S_CMP_U_F16 等

S_CMP 操作正常设置 SCC（SCC=1 表示比较条件满足），其他操作不设置 SCC。

`S_CVT_HI_F32_F16` 没有对应的 VALU 指令，它是 `S_CVT_F32_F16` 的变体，将 SGPR 源的高 16 位从 F16 转换为 F32。

舍入和非规格化处理遵循 MODE.round 和 MODE.denorm 设置。这些标量浮点算术指令可触发浮点异常，异常处理方式与 VALU 管线中发生的异常相同。

标量 F16 指令不支持在其源/目标操作数字段中编码半 SGPR。所有标量 F16 指令对指定 SGPR 的低位（bit[15:0]）进行操作，并将其目标 SGPR 的高位（bit[31:16]）设置为 0。

---

## 6.9 状态访问指令

这些指令访问硬件内部寄存器。

**表 33. 硬件内部寄存器**

| 指令 | 编码 | 设置 SCC？ | 操作 |
|------|------|-----------|------|
| S_GETREG_B32 | SOPK | 否 | 将硬件寄存器读入 SDST 的低有效位 |
| S_SETREG_B32 | SOPK | 否 | 将 SDST 的低有效位写入硬件寄存器（注意 SDST 用作源 SGPR） |
| S_SETREG_IMM32_B32 | SOPK | 否 | S_SETREG，32 位数据来自字面常量（64 位指令格式） |

`GETREG/SETREG`：`#SIMM16 = { Size[4:0], Offset[4:0], hwRegId[5:0] }`，Offset 为 0..31，Size 为 1..32。

注意：`S_SETREG` 只应写入完整的寄存器字段，而不是部分字段。但 MODE.FP_ROUND 和 FP_DENORM 的每一位可以单独设置。

---

## 6.10 内存孔径查询

着色器可以通过标量操作数查询共享和私有空间的内存孔径基地址和大小：`PRIVATE_BASE`、`PRIVATE_LIMIT`、`SHARED_BASE`、`SHARED_LIMIT`。

这些值来自 `SH_MEM_BASES` 寄存器（"SMB"），主要与 FLAT 内存指令一起使用。将共享基地址或私有（暂存）基地址设置为零会禁用该孔径。

地址空间布局：

```
Seg1: 基地址 0xFFFF_8000_0000_0000，限制 0xFFFF_FFFF_FFFF_FFFF
Hole: 基地址 0x0000_8000_0000_0000，限制 0xFFFF_7FFF_FFFF_FFFF（非法地址）
Seg0: 基地址 0x0000_0000_0000_0000，限制 0x0000_7FFF_FFFF_FFFF
```

这些常量可由 SALU 和 VALU 操作使用，为 64 位无符号整数：

```
SHARED_BASE   = {SMB.shared_base[15:0],  48'h000000000000}
SHARED_LIMIT  = {SMB.shared_base[15:0],  48'h0000FFFFFFFF}
PRIVATE_BASE  = {SMB.private_base[15:0], 48'h000000000000}
PRIVATE_LIMIT = {SMB.private_base[15:0], 48'h0000FFFFFFFF}
```

"Hole"（非法地址段）：`addr[63:47] != 全零或全一` 且不在共享或私有孔径内。
