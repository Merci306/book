# 第十五章：微码格式

本章规定微码格式的定义，可用于通过提供各种指令格式的标准模板和枚举名称来简化编译。

**字节序**：RDNA4 架构使用小端字节序和位序寻址内存和寄存器。多字节值以最低有效（低阶）字节存储在最低字节地址，说明时最低有效字节在右侧。

SALU 和 VALU 指令可选择性地包含 32 位字面常量，某些 VALU 指令可在末尾包含 32 位 DPP 控制 DWORD。**任何指令不能同时使用 DPP 和字面常量。**

任何标记为"保留"的指令字段必须设置为零。

---

## 表 73. 微码格式摘要

| 微码格式 | 宽度（位） |
|----------|-----------|
| **标量 ALU 和控制格式** | |
| SOP2 | 32 |
| SOP1 | 32 |
| SOPK | 32 |
| SOPP | 32 |
| SOPC | 32 |
| **标量内存格式** | |
| SMEM | 64 |
| **向量 ALU 格式** | |
| VOP1 | 32 |
| VOP2 | 32 |
| VOPC | 32 |
| VOP3 | 64 |
| VOP3SD | 64 |
| VOP3P | 64 |
| VOPD | 64 |
| DPP16 | 32 |
| DPP8 | 32 |
| **向量参数插值格式** | |
| VINTERP | 64 |
| **LDS 参数加载和直接加载** | |
| VDSDIR | 32 |
| **数据共享格式** | |
| VDS | 64 |
| **向量内存缓冲区格式** | |
| VBUFFER | 96 |
| **向量内存图像格式** | |
| VIMAGE | 96 |
| VSAMPLE | 96 |
| **Flat、Global 和 Scratch 格式** | |
| VFLAT | 96 |
| VGLOBAL | 96 |
| VSCRATCH | 96 |
| **导出格式** | |
| VEXPORT | 64 |

---

## 指令后缀

大多数指令包含一个表示指令处理数据类型的后缀：
- **B** = 二进制（Binary）
- **F** = 浮点（Floating point）
- **BF** = 脑浮点（"brain-float"）
- **U** = 无符号整数（Unsigned）
- **I** = 有符号整数（Signed）
- **IU** = 有符号或无符号整数（Signed or Unsigned）

当出现多个数据类型说明符时，第一个是结果类型和大小，后续的是输入数据类型和大小。例如：`V_CVT_F32_I32` 读取整数并写入浮点。

---

## 15.1 标量 ALU 和控制格式

### 15.1.1 SOP2

**描述**：两输入一输出的标量指令，可后跟 32 位字面常量。

#### 表 74. SOP2 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SSRC0 | [7:0] | 源 0，第一个操作数：SGPR0-105、VCC_LO/HI、TTMP0-15、NULL、M0、EXEC_LO/HI、整数内联常量 0-64/-1 to -16、浮点内联常量 0.5/-0.5/1.0/-1.0/2.0/-2.0/4.0/-4.0/1/(2π)、SCC、64/32 位字面常量 |
| SSRC1 | [15:8] | 第二个标量源操作数，与 SSRC0 编码相同 |
| SDST | [22:16] | 标量目标，与 SSRC0 相同但只有代码 0-127 有效 |
| OP | [29:23] | 操作码（见表 75） |
| ENCODING | [31:30] | 'b10 |

#### 表 75. SOP2 操作码

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
| 0 | S_ADD_CO_U32 | 38 | S_BFE_U32 |
| 1 | S_SUB_CO_U32 | 39 | S_BFE_I32 |
| 2 | S_ADD_CO_I32 | 40 | S_BFE_U64 |
| 3 | S_SUB_CO_I32 | 41 | S_BFE_I64 |
| 4 | S_ADD_CO_CI_U32 | 42 | S_BFM_B32 |
| 5 | S_SUB_CO_CI_U32 | 43 | S_BFM_B64 |
| 6 | S_ABSDIFF_I32 | 44 | S_MUL_I32 |
| 8 | S_LSHL_B32 | 45 | S_MUL_HI_U32 |
| 9 | S_LSHL_B64 | 46 | S_MUL_HI_I32 |
| 10 | S_LSHR_B32 | 48 | S_CSELECT_B32 |
| 11 | S_LSHR_B64 | 49 | S_CSELECT_B64 |
| 12 | S_ASHR_I32 | 50 | S_PACK_LL_B32_B16 |
| 13 | S_ASHR_I64 | 51 | S_PACK_LH_B32_B16 |
| 14 | S_LSHL1_ADD_U32 | 52 | S_PACK_HH_B32_B16 |
| 15 | S_LSHL2_ADD_U32 | 53 | S_PACK_HL_B32_B16 |
| 16 | S_LSHL3_ADD_U32 | 64 | S_ADD_F32 |
| 17 | S_LSHL4_ADD_U32 | 65 | S_SUB_F32 |
| 18 | S_MIN_I32 | 66 | S_MIN_NUM_F32 |
| 19 | S_MIN_U32 | 67 | S_MAX_NUM_F32 |
| 20 | S_MAX_I32 | 68 | S_MUL_F32 |
| 21 | S_MAX_U32 | 69 | S_FMAAK_F32 |
| 22 | S_AND_B32 | 70 | S_FMAMK_F32 |
| 23 | S_AND_B64 | 71 | S_FMAC_F32 |
| 24 | S_OR_B32 | 72 | S_CVT_PK_RTZ_F16_F32 |
| 25 | S_OR_B64 | 73 | S_ADD_F16 |
| 26 | S_XOR_B32 | 74 | S_SUB_F16 |
| 27 | S_XOR_B64 | 75 | S_MIN_NUM_F16 |
| 28 | S_NAND_B32 | 76 | S_MAX_NUM_F16 |
| 29 | S_NAND_B64 | 77 | S_MUL_F16 |
| 30 | S_NOR_B32 | 78 | S_FMAC_F16 |
| 31 | S_NOR_B64 | 79 | S_MINIMUM_F32 |
| 32 | S_XNOR_B32 | 80 | S_MAXIMUM_F32 |
| 33 | S_XNOR_B64 | 81 | S_MINIMUM_F16 |
| 34 | S_AND_NOT1_B32 | 82 | S_MAXIMUM_F16 |
| 35 | S_AND_NOT1_B64 | 83 | S_ADD_NC_U64 |
| 36 | S_OR_NOT1_B32 | 84 | S_SUB_NC_U64 |
| 37 | S_OR_NOT1_B64 | 85 | S_MUL_U64 |

---

### 15.1.2 SOPK

**描述**：一个 16 位有符号立即数（SIMM16）输入、单个目标的标量指令。使用两个输入的指令将目标用作第一个输入，SIMM16 作为第二个输入（如 `S_ADDK_I32 S0, 1` 表示 `S0 = S0 + 1`）。

#### 表 76. SOPK 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SIMM16 | [15:0] | 有符号 16 位立即数 |
| SDST | [22:16] | 标量目标（兼作第二源操作数）：SGPR0-105、VCC_LO/HI、TTMP0-15、M0、NULL、EXEC_LO/HI |
| OP | [27:23] | 操作码（见表 77） |
| ENCODING | [31:28] | 'b1011 |

#### 表 77. SOPK 操作码

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
| 0 | S_MOVK_I32 | 17 | S_GETREG_B32 |
| 1 | S_VERSION | 18 | S_SETREG_B32 |
| 2 | S_CMOVK_I32 | 19 | S_SETREG_IMM32_B32 |
| 15 | S_ADDK_CO_I32 | 20 | S_CALL_B64 |
| 16 | S_MULK_I32 | | |

---

### 15.1.3 SOP1

**描述**：两输入一输出的标量指令，可后跟 32 位字面常量。

#### 表 78. SOP1 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SSRC0 | [7:0] | 源 0（与 SOP2 SSRC0 编码相同） |
| OP | [15:8] | 操作码（见表 79） |
| SDST | [22:16] | 标量目标，与 SSRC0 相同但仅代码 0-127 有效 |
| ENCODING | [31:23] | 'b10_1111101 |

#### 表 79. SOP1 操作码（部分）

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
| 0 | S_MOV_B32 | 50 | S_OR_NOT1_SAVEEXEC_B32 |
| 1 | S_MOV_B64 | 51 | S_OR_NOT1_SAVEEXEC_B64 |
| 2 | S_CMOV_B32 | 52 | S_AND_NOT0_WREXEC_B32 |
| 3 | S_CMOV_B64 | 53 | S_AND_NOT0_WREXEC_B64 |
| 4 | S_BREV_B32 | 54 | S_AND_NOT1_WREXEC_B32 |
| 5 | S_BREV_B64 | 55 | S_AND_NOT1_WREXEC_B64 |
| 8 | S_CTZ_I32_B32 | 64 | S_MOVRELS_B32 |
| 9 | S_CTZ_I32_B64 | 65 | S_MOVRELS_B64 |
| 10 | S_CLZ_I32_U32 | 66 | S_MOVRELD_B32 |
| 11 | S_CLZ_I32_U64 | 67 | S_MOVRELD_B64 |
| 12 | S_CLS_I32 | 68 | S_MOVRELSD_2_B32 |
| 13 | S_CLS_I32_I64 | 71 | S_GETPC_B64 |
| 14 | S_SEXT_I32_I8 | 72 | S_SETPC_B64 |
| 15 | S_SEXT_I32_I16 | 73 | S_SWAPPC_B64 |
| 16 | S_BITSET0_B32 | 74 | S_RFE_B64 |
| 17 | S_BITSET0_B64 | 76 | S_SENDMSG_RTN_B32 |
| 18 | S_BITSET1_B32 | 77 | S_SENDMSG_RTN_B64 |
| 19 | S_BITSET1_B64 | 78 | S_BARRIER_SIGNAL |
| 20 | S_BITREPLICATE_B64_B32 | 79 | S_BARRIER_SIGNAL_ISFIRST |
| 21 | S_ABS_I32 | 80 | S_GET_BARRIER_STATE |
| 22 | S_BCNT0_I32_B32 | 83 | S_ALLOC_VGPR |
| 23 | S_BCNT0_I32_B64 | 88 | S_SLEEP_VAR |
| 24 | S_BCNT1_I32_B32 | 96 | S_CEIL_F32 |
| 25 | S_BCNT1_I32_B64 | 97 | S_FLOOR_F32 |
| 26 | S_QUADMASK_B32 | 98 | S_TRUNC_F32 |
| 27 | S_QUADMASK_B64 | 99 | S_RNDNE_F32 |
| 28 | S_WQM_B32 | 100 | S_CVT_F32_I32 |
| 29 | S_WQM_B64 | 101 | S_CVT_F32_U32 |
| 30 | S_NOT_B32 | 102 | S_CVT_I32_F32 |
| 31 | S_NOT_B64 | 103 | S_CVT_U32_F32 |
| 32 | S_AND_SAVEEXEC_B32 | 104 | S_CVT_F16_F32 |
| 33 | S_AND_SAVEEXEC_B64 | 105 | S_CVT_F32_F16 |
| 34 | S_OR_SAVEEXEC_B32 | 106 | S_CVT_HI_F32_F16 |
| 35 | S_OR_SAVEEXEC_B64 | 107 | S_CEIL_F16 |
| 36 | S_XOR_SAVEEXEC_B32 | 108 | S_FLOOR_F16 |
| 37 | S_XOR_SAVEEXEC_B64 | 109 | S_TRUNC_F16 |
| 38 | S_NAND_SAVEEXEC_B32 | 110 | S_RNDNE_F16 |
| 39 | S_NAND_SAVEEXEC_B64 | | |
| 40 | S_NOR_SAVEEXEC_B32 | | |
| 41 | S_NOR_SAVEEXEC_B64 | | |
| 42 | S_XNOR_SAVEEXEC_B32 | | |
| 43 | S_XNOR_SAVEEXEC_B64 | | |
| 44 | S_AND_NOT0_SAVEEXEC_B32 | | |
| 45 | S_AND_NOT0_SAVEEXEC_B64 | | |
| 46 | S_OR_NOT0_SAVEEXEC_B32 | | |
| 47 | S_OR_NOT0_SAVEEXEC_B64 | | |
| 48 | S_AND_NOT1_SAVEEXEC_B32 | | |
| 49 | S_AND_NOT1_SAVEEXEC_B64 | | |

---

### 15.1.4 SOPC

**描述**：两输入标量比较指令，结果产生 SCC，可后跟 32 位字面常量。

#### 表 80. SOPC 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SSRC0 | [7:0] | 源 0（与 SOP2 SSRC0 编码相同） |
| SSRC1 | [15:8] | 第二个标量源操作数，与 SSRC0 相同 |
| OP | [22:16] | 操作码（见表 81） |
| ENCODING | [31:23] | 'b10_1111110 |

#### 表 81. SOPC 操作码（部分）

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
| 0 | S_CMP_EQ_I32 | 65 | S_CMP_LT_F32 |
| 1 | S_CMP_LG_I32 | 66 | S_CMP_EQ_F32 |
| 2 | S_CMP_GT_I32 | 67 | S_CMP_LE_F32 |
| 3 | S_CMP_GE_I32 | 68 | S_CMP_GT_F32 |
| 4 | S_CMP_LT_I32 | 69 | S_CMP_LG_F32 |
| 5 | S_CMP_LE_I32 | 70 | S_CMP_GE_F32 |
| 6 | S_CMP_EQ_U32 | 71 | S_CMP_O_F32 |
| 7 | S_CMP_LG_U32 | 72 | S_CMP_U_F32 |
| 8 | S_CMP_GT_U32 | 73 | S_CMP_NGE_F32 |
| 9 | S_CMP_GE_U32 | 74 | S_CMP_NLG_F32 |
| 10 | S_CMP_LT_U32 | 75 | S_CMP_NGT_F32 |
| 11 | S_CMP_LE_U32 | 76 | S_CMP_NLE_F32 |
| 12 | S_BITCMP0_B32 | 77 | S_CMP_NEQ_F32 |
| 13 | S_BITCMP1_B32 | 78 | S_CMP_NLT_F32 |
| 14 | S_BITCMP0_B64 | 81 | S_CMP_LT_F16 |
| 15 | S_BITCMP1_B64 | 82 | S_CMP_EQ_F16 |
| 16 | S_CMP_EQ_U64 | 83 | S_CMP_LE_F16 |
| 17 | S_CMP_LG_U64 | 84 | S_CMP_GT_F16 |
| — | — | 85-94 | S_CMP_{LG,GE,O,U,NGE,NLG,NGT,NLE,NEQ,NLT}_F16 |

---

### 15.1.5 SOPP

**描述**：一个 16 位有符号立即数（SIMM16）输入的标量指令。

#### 表 82. SOPP 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SIMM16 | [15:0] | 有符号 16 位立即数 |
| OP | [22:16] | 操作码（见表 83） |
| ENCODING | [31:23] | 'b10_1111111 |

#### 表 83. SOPP 操作码

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
| 0 | S_NOP | 37 | S_CBRANCH_EXECZ |
| 1 | S_SETKILL | 38 | S_CBRANCH_EXECNZ |
| 2 | S_SETHALT | 48 | S_ENDPGM |
| 3 | S_SLEEP | 49 | S_ENDPGM_SAVED |
| 5 | S_CLAUSE | 52 | S_WAKEUP |
| 7 | S_DELAY_ALU | 53 | S_SETPRIO |
| 8 | S_WAIT_ALU | 54 | S_SENDMSG |
| 9 | S_WAITCNT | 55 | S_SENDMSGHALT |
| 10 | S_WAIT_IDLE | 56 | S_INCPERFLEVEL |
| 11 | S_WAIT_EVENT | 57 | S_DECPERFLEVEL |
| 16 | S_TRAP | 60 | S_ICACHE_INV |
| 17 | S_ROUND_MODE | 64 | S_WAIT_LOADCNT |
| 18 | S_DENORM_MODE | 65 | S_WAIT_STORECNT |
| 20 | S_BARRIER_WAIT | 66 | S_WAIT_SAMPLECNT |
| 31 | S_CODE_END | 67 | S_WAIT_BVHCNT |
| 32 | S_BRANCH | 68 | S_WAIT_EXPCNT |
| 33 | S_CBRANCH_SCC0 | 70 | S_WAIT_DSCNT |
| 34 | S_CBRANCH_SCC1 | 71 | S_WAIT_KMCNT |
| 35 | S_CBRANCH_VCCZ | 72 | S_WAIT_LOADCNT_DSCNT |
| 36 | S_CBRANCH_VCCNZ | 73 | S_WAIT_STORECNT_DSCNT |

---

## 15.2 标量内存格式

### 15.2.1 SMEM

**描述**：标量内存数据加载。

#### 表 84. SMEM 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SBASE | [5:0] | 提供基地址的 SGPR 对，或提供 V# 的 SGPR 四元组（LSB 的 SGPR 地址被省略） |
| SDATA | [12:6] | 提供写入数据或接收返回数据的 SGPR |
| OP | [18:13] | 操作码（见表 85） |
| SCOPE | [22:21] | 内存作用域 |
| TH | [24:23] | 内存时间提示 |
| ENCODING | [31:26] | 'b111101 |
| IOFFSET | [55:32] | 有符号字节立即偏移，缓存无效化时忽略 |
| SOFFSET | [63:57] | 提供无符号字节偏移的 SGPR，设为 NULL 禁用 |

#### 表 85. SMEM 操作码

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
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

---

## 15.3 向量 ALU 格式

### 15.3.1 VOP2

**描述**：两输入向量 ALU 格式，当指令允许时可后跟 32 位字面常量或 DPP 指令 DWORD。

#### 表 86. VOP2 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SRC0 | [8:0] | 源 0（与 SOP2 SSRC0 编码相同，另加 DPP8/DPP8FI/DPP16 和 VGPR 256-511） |
| VSRC1 | [16:9] | 提供第二个操作数的 VGPR |
| VDST | [24:17] | 目标 VGPR |
| OP | [30:25] | 操作码（见表 87） |
| ENCODING | [31] | 'b0 |

#### 表 87. VOP2 操作码（部分）

| 操作码 | 名称 | 操作码 | 名称 |
|--------|------|--------|------|
| 1 | V_CNDMASK_B32 | 29 | V_XOR_B32 |
| 2 | V_ADD_F64 | 30 | V_XNOR_B32 |
| 3 | V_ADD_F32 | 31 | V_LSHLREV_B64 |
| 4 | V_SUB_F32 | 32 | V_ADD_CO_CI_U32 |
| 5 | V_SUBREV_F32 | 33 | V_SUB_CO_CI_U32 |
| 6 | V_MUL_F64 | 34 | V_SUBREV_CO_CI_U32 |
| 7 | V_MUL_DX9_ZERO_F32 | 37 | V_ADD_NC_U32 |
| 8 | V_MUL_F32 | 38 | V_SUB_NC_U32 |
| 9 | V_MUL_I32_I24 | 39 | V_SUBREV_NC_U32 |
| 10 | V_MUL_HI_I32_I24 | 43 | V_FMAC_F32 |
| 11 | V_MUL_U32_U24 | 44 | V_FMAMK_F32 |
| 12 | V_MUL_HI_U32_U24 | 45 | V_FMAAK_F32 |
| 13 | V_MIN_NUM_F64 | 47 | V_CVT_PK_RTZ_F16_F32 |
| 14 | V_MAX_NUM_F64 | 48 | V_MIN_NUM_F16 |
| 17 | V_MIN_I32 | 49 | V_MAX_NUM_F16 |
| 18 | V_MAX_I32 | 50 | V_ADD_F16 |
| 19 | V_MIN_U32 | 51 | V_SUB_F16 |
| 20 | V_MAX_U32 | 52 | V_SUBREV_F16 |
| 21 | V_MIN_NUM_F32 | 53 | V_MUL_F16 |
| 22 | V_MAX_NUM_F32 | 54 | V_FMAC_F16 |
| 24 | V_LSHLREV_B32 | 55 | V_FMAMK_F16 |
| 25 | V_LSHRREV_B32 | 56 | V_FMAAK_F16 |
| 26 | V_ASHRREV_I32 | 59 | V_LDEXP_F16 |
| 27 | V_AND_B32 | 60 | V_PK_FMAC_F16 |
| 28 | V_OR_B32 | | |

---

### 15.3.2 VOP1

**描述**：单输入向量 ALU 格式，当指令允许时可后跟 32 位字面常量或 DPP 指令 DWORD。

#### 表 88. VOP1 字段

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SRC0 | [8:0] | 源 0（与 VOP2 SRC0 编码相同） |
| OP | [16:9] | 操作码 |
| VDST | [24:17] | 目标 VGPR |
| ENCODING | [31:25] | 'b0_1111110 |

---

### 15.3.3 VOPC

**描述**：两输入向量 ALU 比较操作，将比较结果写入 VCC 或 EXEC。

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| SRC0 | [8:0] | 源 0 |
| VSRC1 | [16:9] | 第二个操作数 VGPR |
| OP | [24:17] | 操作码 |
| ENCODING | [31:25] | 'b0_1111110（不同于 VOP1） |

---

### 15.3.4 VOP3

**描述**：三输入向量 ALU 操作，或提升为 VOP3 的 VOP1/2/C 指令，64 位。

| 字段名 | 位 | 格式或描述 |
|--------|-----|-----------|
| OP | [25:16] | 操作码 |
| CM | [15] | 限制/信号比较位 |
| OPSEL | [14:11] | 操作选择（16 位操作） |
| ABS | [10:8] | 对 src0/1/2 应用绝对值 |
| ENCODING | [31:26] | 'b110101 |
| VDST | [39:32] | 目标 VGPR |
| NEG | [63:61] | 对 src0/1/2 取反 |
| OMOD | [60:59] | 输出修饰符 |
| SRC2 | [58:50] | 第三个源操作数 |
| SRC1 | [49:41] | 第二个源操作数 |
| SRC0 | [40:32]（注：后 32 位） | 第一个源操作数 |

---

### 15.3.5 VOP3SD

与 VOP3 类似，但有 SDST 字段（取代 OPSEL 和 ABS），用于产生标量输出的操作（如带进位加法、DIV_SCALE、MAD_CO）。

---

### 15.3.6 VOP3P

打包数学操作的 64 位格式，有独立的 OPSEL 和 OPSEL_HI（高/低操作数选择）以及 NEG 和 NEG_HI 字段。

---

### 15.3.7 VOPD（双发射）

64 位格式，编码两个并行执行的独立 VALU 操作（X 和 Y），仅适用于 wave32。包含 opX（4位）、opY（5位）、src0X、src0Y、vsrc1X、vsrc1Y、vdstX、vdstY 字段。

---

### 15.3.8 DPP16 和 DPP8

DPP（数据并行原语）指令字跟随在使用 DPP 操作的 VOP1/2/C/3/3P 指令后面。

**DPP16 DWORD**（32 位）：包含 row_mask、bank_mask、src1_imod、src0_imod、BC、FI、dpp_ctrl（混洗控制）和 SRC0 VGPR 字段。

**DPP8 DWORD**（32 位）：包含 SRC（SRC0 VGPR 地址）和 SEL0-SEL7（8 个 3 位字段，各选择 8 通道组内的源通道）。

---

## 15.4 向量参数插值格式（VINTERP）

用于参数插值的 64 位向量格式。

---

## 15.5 数据共享格式（VDS）

用于 LDS 操作的 64 位向量格式：

| 字段名 | 位 | 描述 |
|--------|-----|------|
| ADDR | [7:0] | 提供 LDS 地址的 VGPR |
| DATA0 | [15:8] | 提供第一个源数据 DWORD 的 VGPR |
| DATA1 | [23:16] | 提供第二个源数据 DWORD 的 VGPR |
| VDST | [31:24] | 接收返回数据的 VGPR |
| OP | [57:50] | 操作码 |
| ENCODING | [63:58] | 'b110110 |
| OFFSET0 | [7:0]（第二 DWORD） | 地址偏移 0 |
| OFFSET1 | [15:8] | 地址偏移 1 |

---

## 15.6 向量内存缓冲区格式（VBUFFER）

96 位（3 个 DWORD）向量内存缓冲区操作格式：

| 字段名 | 位 | 描述 |
|--------|-----|------|
| VADDR | [7:0] | 提供地址的 VGPR |
| VDATA | [15:8] | 提供数据的 VGPR（第一个 DWORD）|
| SRSRC | [20:16] | 提供缓冲区资源（V#）的 SGPR 四元组（LSB 隐含为零） |
| VDST | [31:24] | 接收返回数据的 VGPR |
| OP | [49:40] | 操作码 |
| SCOPE | [53:52] | 内存作用域 |
| TH | [56:54] | 内存时间提示 |
| ENCODING | [63:58] | 'b111100 |
| SOFFSET | [71:64] | 提供 32 位无符号字节偏移的 SGPR，或设为 NULL |
| OFFSET | [83:72] | 12 位有符号立即字节偏移 |
| IDXEN | [86] | 启用索引（VADDR 包含索引） |
| OFFEN | [87] | 启用偏移（VADDR 包含偏移） |
| SW | [89:88] | 混洗模式 |

---

## 15.7 向量内存图像格式（VIMAGE 和 VSAMPLE）

96 位（3 个 DWORD）向量内存图像操作格式，分为图像加载/存储/原子（VIMAGE）和图像采样（VSAMPLE）。

---

## 15.8 Flat/Global/Scratch 格式

96 位（3 个 DWORD）格式，分为 VFLAT（Flat）、VGLOBAL（Global）和 VSCRATCH（Scratch）三种变体。

| 字段名 | 位 | 描述 |
|--------|-----|------|
| VADDR | [7:0] | 地址或偏移 VGPR |
| VSRC | [15:8] | 存储数据 VGPR |
| VDST | [23:16] | 目标 VGPR |
| OP | [31:24] | 操作码 |
| SCOPE | [33:32] | 内存作用域 |
| TH | [36:34] | 内存时间提示 |
| ENCODING | [63:58] | 格式编码 |
| IOFFSET | [87:64] | 24 位有符号立即字节偏移 |
| SADDR | [94:88] | 提供地址/偏移的 SGPR |
| SVE | [95] | Scratch VGPR 使能 |

---

## 15.9 导出格式（VEXPORT）

64 位导出指令格式：

| 字段名 | 位 | 描述 |
|--------|-----|------|
| EN | [3:0] | 导出使能掩码（每位对应一个分量） |
| TARGET | [9:4] | 导出目标（位置、颜色 MRT、深度等） |
| COMPR | [10] | 压缩导出（两个 16 位值打包到一个 VGPR） |
| DONE | [11] | 最后一次导出信号 |
| VM | [12] | 有效掩码使能 |
| VSRC0-3 | [39:8]（第二 DWORD） | 提供导出数据的 VGPR（最多 4 个） |
| ENCODING | [63:58] | 'b111110 |
