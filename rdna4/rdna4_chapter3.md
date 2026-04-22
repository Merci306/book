# 第三章：Wave 状态

本章描述着色器程序可见的状态变量。除另有说明外，每个 wave 都有这些状态的私有副本。

---

## 3.1 状态概述

下表显示了着色器程序可读写的硬件状态。以下寄存器对每个 wave 唯一，但 TBA 和 TMA 是共享的。

**表 4. 常用着色器 Wave 状态**

| 缩写 | 名称 | 大小（位） | 描述 |
|------|------|-----------|------|
| PC | 程序计数器 | 48 | 指向下一条要执行的着色器指令的内存地址。仅通过标量控制流指令读写，或通过分支间接读写。低 2 位强制为零。 |
| V0-V255 | VGPR | 32 | 向量通用寄存器。（32 位/工作项 × (32 或 64) 工作项/wave） |
| S0-S105 | SGPR | 32 | 标量通用寄存器。所有 wave 分配 106 个 SGPR + 16 个 TTMP。 |
| LDS | 本地数据共享 | 64kB | 带内置算术功能的暂存 RAM，允许工作组内线程间共享数据。 |
| EXEC | 执行掩码 | 64 | 每线程一位的位掩码，应用于向量指令，控制哪些线程执行，哪些忽略该指令。 |
| EXECZ | EXEC 为零 | 1 | 单位标志，指示 EXEC 掩码全为零。对于 wave32，仅考虑 EXEC[31:0]。 |
| VCC | 向量条件码 | 64 | 每线程一位的位掩码，保存向量比较操作的结果或整数进位输出。VCC 物理上存储在特定 SGPR 中。 |
| VCCZ | VCC 为零 | 1 | 单位标志，指示 VCC 掩码全为零。对于 wave32，仅考虑 VCC[31:0]。 |
| SCC | 标量条件码 | 1 | 标量 ALU 比较指令、进位输出或其他用途的结果。 |
| SCRATCH_BASE | 暂存内存地址 | 48 | 该 wave 的暂存内存基地址。由 Flat 和 Scratch 指令使用。用户着色器只读；陷阱处理程序可写。 |
| M0 | 杂项寄存器 | 32 | 具有多种用途的临时寄存器，包括 GPR 索引和边界检查。 |
| STATUS | 状态 | 32 | 着色器状态位（只读）。 |
| STATE_PRIV | 特权状态 | 32 | 仅由陷阱处理程序可写的着色器状态位。 |
| MODE | 模式 | 32 | 可写的着色器模式位。 |
| TRAP_CTRL | 异常掩码 | 32 | 哪些异常触发陷阱的掩码。仅由陷阱处理程序可写；用户着色器可读。 |
| EXCP_FLAG_PRIV | 特权异常标志 | 32 | 已发生异常的掩码。仅由陷阱处理程序可写；用户着色器可读。 |
| EXCP_FLAG_USER | 用户可清除异常标志 | 32 | 已发生异常的掩码。用户着色器可写。 |
| TBA | 陷阱基地址 | 48 | 保存陷阱处理程序的程序地址。每 VMID 寄存器。位[63] 指示是否存在陷阱处理程序（1）或不存在（0），不视为地址的一部分。通过 S_SENDMSG_RTN 访问。 |
| TMA | 陷阱内存地址 | 48 | 着色器操作的临时寄存器，例如可保存陷阱处理程序使用的数据内存指针。 |
| TTMP0-TTMP15 | 陷阱临时 SGPR | 32 | 16 个仅供陷阱处理程序使用的临时存储 SGPR。 |
| LOADcnt | 向量内存加载指令计数 | 6 | 统计已发射但尚未完成的 VMEM 加载（及带返回值的原子）指令数量。 |
| STOREcnt | 向量内存存储指令计数 | 6 | 统计已发射但尚未完成的 VMEM 存储（及无返回值的原子）指令数量。 |
| DScnt | LDS 指令计数 | 6 | 统计已发射但尚未完成的 LDS 指令数量。 |
| KMcnt | 常量和消息计数 | 5 | 统计已发射但尚未完成的常量获取（标量内存读取）和消息指令数量。 |
| SAMPLEcnt | 向量内存采样指令计数 | 6 | 统计已发射但尚未完成的 VMEM 采样/gather/msaa-load/get-lod 指令（VSAMPLE 编码）数量。 |
| BVHcnt | 向量内存 BVH 指令计数 | 3 | 统计已发射但尚未完成的 VMEM BVH 指令数量。 |
| EXPcnt | 导出计数 | 3 | 统计已发射但尚未完成的导出指令数量，也统计未完成的参数加载。 |

---

## 3.2 控制状态：PC 与 EXEC

### 3.2.1 程序计数器（PC）

程序计数器是一个 DWORD 对齐的字节地址，指向下一条要执行的指令。创建 wave 时，PC 初始化为程序中的第一条指令。地址是 DWORD 对齐的，因此最低两位强制为零。

直接与 PC 交互的指令有：`S_GETPC_B64`、`S_SETPC_B64`、`S_CALL_B64`、`S_RFE_B64` 和 `S_SWAPPC_B64`。这些指令将 PC 与偶数对齐的 SGPR 对（零扩展）相互传输。

分支跳转到：`(分支后面一条指令的 PC + offset*4)`。分支、GET_PC 和 SWAP_PC 相对于下一条指令（不是当前指令）计算 PC 偏移。`S_TRAP` 则不同，它保存 `S_TRAP` 指令本身的 PC。

在波形调试期间可以读取程序计数器。PC 指向下一条要发射的指令，此前的所有指令已发射但可能尚未完成执行。

### 3.2.2 EXECute 掩码

执行掩码（64 位）控制向量中哪些线程被执行。每位指示一个线程对向量指令的行为：1 = 执行，0 = 不执行。EXEC 可通过标量指令读写，也可作为向量 ALU 比较的结果写入。EXEC 影响：向量 ALU、向量内存、LDS 和导出指令，不影响标量执行或分支。

Wave64 使用执行掩码的全部 64 位；Wave32 仅使用位 31:0，硬件不对高位进行操作。

有一个汇总位（EXECZ）指示整个执行掩码是否为零，可用作分支条件以在 EXEC 为零时跳过代码。对于 wave32，这反映 EXEC[31:0] 的状态。

#### 3.2.2.1 指令跳过：EXEC == 0

当 EXEC == 0 时，着色器硬件可以跳过向量指令。

被跳过的指令产生与以 EXEC == 0 执行相同的 wave 状态结果：不改变 wave 状态。

**可能被跳过的指令：**
- **VALU**：EXEC == 0 时跳过
  - 若指令写入 SGPR/VCC，则不跳过
  - 不跳过 WMMA 或 SWMMAC 操作
  - 此跳过取决于时序，在 V_CMPX 之后可能不发生

**以下指令无论 EXEC 掩码值均不跳过，在 wave64 模式下只发射一次：**
- `V_NOP`、`V_PIPEFLUSH`、`V_READLANE`、`V_READFIRSTLANE`、`V_WRITELANE`
- `GLOBAL_INV`、`GLOBAL_WB`、`GLOBAL_WBINV`

**以下指令不跳过，在 wave64 模式下无论 EXEC 掩码值均发射两次：**
- 写入 SGPR 或 VCC 的 V_CMP（不是 V_CMPX——可能跳过一次但不跳过两次）
- 写入 SGPR 的任何 VALU

**其他规则：**
- **导出请求**：跳过，除非：Done == 1 或导出目标为 POS0；若 wave 以 SKIP_EXPORT=1 创建则跳过
- **DS_param_load / DS_direct_load**：EXEC == 0 且 EXPcnt == 0 时跳过
- **LDS、内存**：通常不跳过
  - 仅当 EXEC == 0 且 LOADcnt/STOREcnt/BVHcnt/SAMPLEcnt == 0 时，VMEM 可跳过
    - FLAT 还需要 DScnt == 0 才能跳过
    - 否则对于 wave64，若某半部分的 EXEC == 0 则可跳过该半部分，但不能同时跳过两个半部分
  - 仅当 DScnt == 0 且 EXEC == 0 时，LDS 可跳过

---

## 3.3 存储状态：SGPR、VGPR、LDS

### 3.3.1 SGPR

#### 3.3.1.1 SGPR 分配与存储

每个 wave 分配固定数量的 SGPR：
- 106 个普通 SGPR
- VCC_HI 和 VCC_LO（存储在 SGPR 106 和 107 中）
- 16 个陷阱临时 SGPR，供陷阱处理程序使用

**VCC（向量条件码）**

VCC 是一个命名的 SGPR 对，可由 `V_CMP` 和整数向量 ADD/SUB 指令写入。VCC 被 `V_ADD_CI`、`V_SUB_CI`、`V_CNDMASK` 和 `V_DIV_FMAS` 隐式读取。VCC 受与其他任何 SGPR 相同的依赖性检查约束。

#### 3.3.1.2 SGPR 对齐

以下情况需要偶数对齐的 SGPR：
1. 使用 64 位数据时（包括与 64 位寄存器（含 PC）的移动）
2. 地址基来自 SGPR 对的标量内存加载

当 64 位 SGPR 数据值用作 VALU 操作的源时，无论大小均需偶数对齐。相比之下，当 32 位 SGPR 数据值用作 VALU 操作的源时，无论 wave 大小均可任意对齐。

超过 64 位的操作以及标量内存操作（读、写或原子）操作超过 2 个 DWORD 的数据 SGPR 需要四字节对齐。

当 64 位量存储在 SGPR 中时，低有效位在 SGPR[n]，高有效位在 SGPR[n+1]。

当 SGPR 用作 VALU 操作的进位输入、进位输出或掩码值时，wave64 着色器中必须偶数对齐，但 wave32 着色器中可任意对齐。

对大于 32 位的数据使用未对齐的源或目标 SGPR 是非法的，结果不可预测。

#### 3.3.1.3 SGPR 越界行为

标量源和目标使用 7 位编码：

标量 0-105 = SGPR；106, 107 = VCC；108-123 = TTMP0-15；124-127 = {NULL, M0, EXEC_LO, EXEC_HI}。

非法使用 GPR 索引或多 DWORD 操作数跨越 SGPR 区域。区域划分：
- SGPR 0-107（含 VCC）
- 陷阱临时 SGPR
- 所有其他 SGPR 和标量源地址不能被索引，单个操作数不能引用多个寄存器范围

**一般规则：**
- 越界的源 SGPR 返回零（不允许 NULL、M0 或 EXEC 的位置返回零）
- 写入越界 SGPR 被忽略
- TTMP0-15 只能在陷阱处理程序中（STATUS.PRIV=1）写入，用户着色器（STATUS.PRIV=0）可读取。在陷阱处理程序外写入 TTMP 被忽略。尝试但失败写入 TTMP 的 SALU 指令也不更新 SCC。

**特殊规则：**
- WREXEC 和 SAVEEXEC：即使 SDST 越界也写入 EXEC 掩码
- VMEM：S#、T#、V# 必须包含在一个区域内
- SMEM 操作：不写入越界目标 SGPR，读取 M0/NULL/EXEC* 返回零且无法加载到这些寄存器
- S_MOVREL：索引仅在 SGPR 和 TTMP 内允许，不能跨两者。越界时源用 S0，目标被忽略

### 3.3.2 VGPR

#### 3.3.2.1 VGPR 分配与对齐

VGPR 以 wave32 的 16 块或 wave64 的 8 块为单位分配，着色器最多可有 256 个 VGPR。换句话说，VGPR 以（16×32 或 8×64 = 512 DWORD）为单位分配。不能以零 VGPR 创建 wave。具有每 SIMD 1536 VGPR 的设备以 wave32 的 24 块和 wave64 的 12 块分配。

Wave 可以通过 `S_SENDMSG` 主动释放其所有 VGPR。一旦这样做，wave 将无法重新分配它们，唯一有效的操作是终止 wave。这在 wave 已向内存发出存储并等待写入确认之前终止时很有用，在等待期间释放 VGPR 可能允许新 wave 更早分配它们并开始执行。

#### 3.3.2.2 VGPR 越界行为

给定使用一个或多个 VGPR 数据 DWORD 的指令操作数：
- Vs = 第一个 VGPR DWORD（起始）
- Ve = 最后一个 VGPR DWORD（结束）

操作数越界条件：`Vs < 0 || Vs >= VGPR_SIZE` 或 `Ve < 0 || Ve >= VGPR_SIZE`

**越界后果：**
- 若目标 VGPR 越界，指令被忽略（视为 NOP）。VMEM 指令（含 LDS）以 EXEC==0 发射以保持 LOADcnt/STOREcnt/… 正确
- V_SWAP & V_SWAPREL：由于两个参数都是目标，若任一越界则丢弃指令
- 源 VGPR 越界时：VMEM 或导出指令使用 VGPR0；VALU 指令的源字段被设置为 VGPR0（大于 32 位的操作数使用连续 VGPR：V0, V1, V2, …）
  - VOPD 有不同规则：源地址强制为（VGPRaddr % 4）
- 多目标指令（如 V_ADD_CO）：若任何目标越界，不写入任何结果

### 3.3.3 动态 VGPR 分配与释放

计算着色器可在允许动态分配和释放 VGPR 的模式下启动；图形着色器不支持动态 VGPR。Wave 必须在"动态 VGPR"模式下启动才能使用此功能；否则请求更改 VGPR 分配大小的指令被忽略。

**限制：**
- 动态 VGPR 仅支持 wave32，不支持 wave64
- 动态 VGPR 工作组独占 WGP（不允许动态与非动态 VGPR wave 混用）
- VGPR 以 16 或 32 块（可配置）为单位分配/释放，从最高编号的 VGPR 增减，保持从 VGPR0 开始的连续逻辑 VGPR 范围
- Wave 最多可分配 8 块 VGPR，最少一块

**块大小：**

VGPR 块大小可配置为 16-VGPR（最大分配 128 VGPR/wave）或 32-VGPR（最大分配 256 VGPR/wave）。块大小是芯片级配置，不能按每次绘制或调度修改。使用 16-VGPR 块大小的 wave 不得访问 127 以上的 VGPR。

动态 VGPR 模式的 wave 以分配一个 VGPR 块启动。

**指令：**

```
S_ALLOC_VGPR <NumVgprs>   // Wave 现在希望拥有的 VGPR 数量。可以是内联常量或 SGPR
                           // NumVgprs 向上舍入到最近的 BlockSize
```

`S_ALLOC_VGPR` 尝试分配（NumVgprs > currentVgprs 时）或释放（NumVgprs < currentVgprs 时）。分配请求可能失败并返回 SCC=1（成功）或 SCC=0（失败）。分配不部分成功——要么全部成功，要么全部失败。若分配失败，着色器可以重试直到成功。释放不会失败，不改变分配大小的 S_ALLOC_VGPR 也不会失败。仅考虑 NumVgprs 的位 [8:0]。

**死锁规避：**

动态分配 VGPR 可能在所有 VGPR 都已分配但每个 wave 都需要分配更多 VGPR 才能继续时导致死锁。硬件通过保留足够 VGPR 使至少一个 wave 始终能达到最大 VGPR 分配来缓解这一问题，但当多个 wave 需要最大分配才能继续时无法防止死锁。

### 3.3.4 内存对齐与越界行为

本节定义源或目标 GPR 或内存地址超出 wave 合法范围时的行为。除另有说明外，这些规则适用于 LDS、缓冲区、全局、flat 和暂存内存访问。

**内存、LDS：加载和带返回值的原子：**
- 若任何源 VGPR 或 SGPR 越界，数据值未定义
- 若任何目标 VGPR 越界，操作通过以 EXEC 掩码清零来发射指令以使其无效化
- 图像加载和存储在确定越界时考虑 DMASK 位

**VMEM 内存对齐规则**由配置寄存器 `SH_MEM_CONFIG.alignment_mode` 定义，也影响 LDS、Flat/Scratch/Global 操作：
- **DWORD**：自动对齐到元素大小或 DWORD 中较小者的倍数
- **UNALIGNED**：没有对齐要求（原子除外）

格式化操作（如 `BUFFER_LOAD_FORMAT_*`）必须按如下方式对齐：1 字节格式需 1 字节对齐；2 字节格式需 2 字节对齐；4 字节及更大格式需 4 字节对齐。原子必须对齐到数据大小，否则触发 MEMVIOL。

### 3.3.5 本地数据共享（LDS）

LDS 是分配给 wave 或工作组的暂存内存（有时称为"共享内存"）。Wave 可被分配 LDS 内存，工作组中的所有 wave 共享同一 LDS 内存分配。对 LDS 的所有访问仅限于分配给该 wave/工作组的空间。

Wave 可分配 0 至 64kB 的 LDS 空间，以 1024 字节块为单位分配。

LDS 内部由两块各 64kB 的内存组成，分别与其中一个 CU 关联：字节地址 0-65535 与 CU0 关联，65536-131071 与 CU1 关联。LDS 空间的分配不会回绕：分配起始地址小于结束地址。

- **CU 模式**：wave 的整个 LDS 分配位于与 wave 加载的 SIMD 同侧的 LDS。不允许访问越过或回绕到另一侧的数据
- **WGP 模式**：wave 的 LDS 分配可完全位于 CU0 或 CU1 部分，或跨越两者边界

像素参数加载到 wave 所在的同侧 CU，不跨越到 LDS 存储的另一侧。像素着色器仅在 CU 模式下运行，可以请求除顶点参数所需空间外的额外 LDS 空间。

**LDS 对齐与越界：**

在"未对齐"模式下，任何大小的 DS_LOAD 或 DS_STORE 都可以字节对齐。其他对齐模式下，LDS 通过将地址低有效位清零来强制对齐：

- 32 位原子必须 4 字节对齐；64 位原子必须 8 字节对齐，否则返回 MEMVIOL
- LDS 地址越界时报告 MEMVIOL

**越界行为：**
- LDS 地址越界（addr < 0 或 >= LDS_size）：加载返回零；存储中在范围内的部分被写入，越界部分被丢弃
- 源 VGPR 越界：使用 VGPR0 的值作为 LDS 地址或数据
- 目标 VGPR 越界：使指令无效化（以 EXEC=0 发射）

**LDS 原生对齐：**
- B8：字节对齐；B16 或 D16：2 字节对齐；B32：4 字节对齐；B64：8 字节对齐；B128 和 B96：16 字节对齐

### 3.3.6 暂存（私有）内存

Wave 可在 wave 启动时分配一块全局内存，用于线程私有数据。此内存对每个线程私有（虽然这不被强制执行），内存交错针对每个线程具有相同索引的情况进行了优化。

暂存内存以 64-DWORD 粒度分配，每个 wave 可分配 [0 - (16M-64)] DWORD。

---

## 3.4 Wave 状态寄存器

以下寄存器访问频率低，仅通过 `S_GETREG_B32` 和 `S_SETREG` 指令读写。部分只读，部分可写，部分仅在陷阱处理程序中可写（"PRIV"）。

| 索引 | 可写？ | 寄存器 | 描述 |
|------|--------|--------|------|
| 1 | 是 | MODE | Wave 模式位 |
| 2 | 否 | STATUS | Wave 状态位 |
| 4 | PRIV | STATE_PRIV | 陷阱处理程序可写的 wave 状态位 |
| 10 | 否 | PERF_SNAPSHOT_DATA | 性能快照期间捕获的数据 |
| 11 | 否 | PERF_SNAPSHOT_PC_LO | 性能快照的 PC[31:0] |
| 12 | 否 | PERF_SNAPSHOT_PC_HI | 性能快照的 PC[39:32] |
| 15 | 否 | PERF_SNAPSHOT_DATA1 | 性能快照期间捕获的数据 |
| 16 | 否 | PERF_SNAPSHOT_DATA2 | 性能快照期间捕获的数据 |
| 17 | PRIV | EXCP_FLAG_PRIV | 仅陷阱处理程序可写的异常标志 |
| 18 | 是 | EXCP_FLAG_USER | 用户可写的异常标志 |
| 19 | PRIV | TRAP_CTRL | 陷阱异常使能；仅陷阱处理程序可写 |
| 20 | PRIV | SCRATCH_BASE_LO | 用户只读；仅陷阱处理程序可写 |
| 21 | PRIV | SCRATCH_BASE_HI | 用户只读；仅陷阱处理程序可写 |
| 23 | 否 | HW_ID1 | 只读，仅调试用——值不可预测 |
| 24 | 否 | HW_ID2 | 只读，仅调试用——值不可预测 |
| 28 | 否 | IB_STS2 | — |
| 29 | 否 | SHADER_CYCLES_LO | 获取当前着色器时钟计数器值 |
| 30 | 否 | SHADER_CYCLES_HI | 获取当前着色器时钟计数器值 |

### 3.4.1 STATUS 寄存器

状态寄存器字段可由着色器读取但不能写入。这些位在 wave 创建时初始化或在执行期间更新。

**表 5. 状态寄存器字段**

| 字段 | 位 | 描述 |
|------|-----|------|
| PRIV | 5 | 特权模式。指示 wave 处于陷阱处理程序中，允许写入 TTMP 寄存器。 |
| TRAP_EN | 6 | 指示陷阱处理程序已存在。为零时不触发陷阱。 |
| EXPORT_RDY | 8 | 仅限像素着色器：此状态位指示导出缓冲区空间已分配。着色器会暂停任何导出指令直到此位变为 1。 |
| EXECZ | 9 | 执行掩码为零。 |
| VCCZ | 10 | 向量条件码为零。 |
| IN_WG | 11 | Wave 是超过一个 wave 的工作组的成员。 |
| TRAP | 14 | Wave 被标记为尽快进入陷阱处理程序。 |
| TRAP_BARRIER_COMPLETE | 15 | 陷阱屏障已完成但 wave 尚未等待该屏障。 |
| VALID | 16 | Wave 有效（已创建且尚未结束）。 |
| SKIP_EXPORT | 18 | 仅限像素和顶点着色器：1 表示此着色器未分配导出缓冲区空间，导出指令被忽略（视为 NOP）。 |
| FATAL_HALT | 23 | 指示 wave 因致命错误而停止（如非法指令）。与 HALT 的区别在于 FATAL_HALT 即使 PRIV=1 也停止 wave。 |
| NO_VGPRS | 24 | 指示此 wave 已释放所有 VGPR。 |
| LDS_PARAM_RDY | 25 | 仅限像素着色器：指示 LDS 已写入顶点属性数据，着色器现在可以执行 DS_PARAM_LOAD 指令。 |
| MUST_GS_ALLOC | 26 | 指示 GS 着色器在终止前必须发出 GS_ALLOC_REQ 消息，发送后清除此位。 |
| MUST_EXPORT | 27 | 仅限像素着色器：此 wave 必须在终止前导出颜色（"export-done"）。除非 skip_export==1，否则 PS wave 设为 1；PS 以导出的 Done 位为 1 导出数据时清除。其他 wave 类型为零。 |
| IDLE | 28 | Wave 空闲（没有未完成的指令）。 |
| WAVE64 | 29 | Wave 为 64（0 = wave32）。 |
| DYN_VGPR_EN | 30 | 指示 wave 使用动态 VGPR 运行。 |

### 3.4.2 STATE_PRIV 寄存器

STATE_PRIV 寄存器字段可由用户着色器读取，并在陷阱处理程序中（PRIV=1）写入。

**表 6. STATE_PRIV 寄存器字段**

| 字段 | 位 | 描述 |
|------|-----|------|
| WG_RR_EN | 0 | 请求工作组轮询仲裁。 |
| SLEEP_WAKEUP | 1 | Wave 有一个信用，由于之前收到 S_WAKEUP 而跳过下一条 S_SLEEP 指令。 |
| BARRIER_COMPLETE | 2 | 屏障已完成但 wave 尚未等待该屏障。 |
| SCC | 9 | 标量条件码，用作进位输出位。 |
| SYS_PRIO | 11:10 | wave 创建时设置的 wave 优先级。0 为最低，3 为最高。 |
| USER_PRIO | 13:12 | 由着色器程序本身设置的 wave 优先级。 |
| HALT | 14 | Wave 已停止或被调度停止。可由主机通过 wave 控制消息或着色器设置。在陷阱处理程序中（PRIV=1）忽略 HALT 位。 |
| SCRATCH_EN | 18 | 指示 wave 已分配暂存内存。若 wave 已初始化 SCRATCH_BASE 则为 1，否则为零。 |

### 3.4.3 MODE 寄存器

模式寄存器字段可由着色器通过标量指令读写。

**表 7. 模式寄存器字段**

| 字段 | 位 | 描述 |
|------|-----|------|
| FP_ROUND | 3:0 | 控制数学运算的舍入模式。[1:0] 单精度舍入模式；[3:2] 双精度和半精度（FP16）舍入模式。舍入模式：0=最近偶数，1=+无穷，2=-无穷，3=向零。影响 VALU 中的浮点操作，但不影响 LDS 或内存。 |
| FP_DENORM | 7:4 | 控制浮点非规格化数是否刷新。[5:4] 单精度；[7:6] 双精度和 FP16。模式：00=刷新输入和输出；01=允许输入非规格化，刷新输出；10=刷新输入非规格化，允许输出；11=允许输入和输出非规格化。影响 VALU 和 LDS 原子，但不影响 flat 和内存原子。 |
| FP16_OVFL | 23 | 若设置，溢出的 FP16 VALU 结果被夹到 +/- MAX_FP16，无论舍入模式，同时保留真正的 INF 值。 |
| SCALAR_PREFETCH_EN | 24 | 1=启用标量预取指令（指令和数据）；0=忽略（S_NOP）。 |
| DISABLE_PERF | 27 | 1=暂时禁用此 wave 的性能计数。 |

### 3.4.4 M0：杂项寄存器

每个 wave 有一个 32 位 M0 寄存器，用于以下用途：

**表 8. M0 寄存器字段**

| 操作 | M0 内容 | 注意事项 |
|------|---------|---------|
| S/V_MOVREL | GPR 索引 | 参见 S_MOVREL 和 V_MOVREL 指令 |
| LDS ADDTID | `{ 16'h0, lds_offset[15:0] }` | 偏移量以字节为单位，必须 4 字节对齐 |
| DS_PARAM_LOAD | `{ 1'b0, new_prim_mask[15:1], parameter_offset[15:0] }` | 偏移量以字节为单位，offset[6:0] 必须为零。Wave32: new_prim_mask 为 `{8'b0, mask[7:1]}` |
| DS_DIRECT_LOAD | `{ 13'b0, DataType[2:0], LDS_address[15:0] }` | 地址以字节为单位 |
| EXPORT | 网格着色器 POS 和参数导出的行号 | 参见导出章节 |
| S_SENDMSG / _RTN | 各种 | 发送消息数据，参见发送消息类型 |
| 各种 | 临时数据 [31:0] | 可用作通用临时数据存储 |

M0 只能由标量 ALU 写入。

### 3.4.5 NULL

NULL 是标量源和目标。读取 NULL 返回零，写入 NULL 无效（写入数据被丢弃）。

NULL 可用在通常可使用标量源的任何地方：
- 当 NULL 用作 SALU 指令的目标时，指令执行：SDST 不被写入但 SCC 被更新（如果该指令正常更新 SCC）
- 指令如 S_SWAP_PC，以 NULL 作为 DEST 仍然执行，将零加载到 PC 且不写入任何其他结果
- NULL 不能用作 S#、V# 或 T#

### 3.4.6 SCC：标量条件码

许多标量 ALU 指令设置标量条件码（SCC）位，指示操作结果：
- **比较操作**：1 = 真
- **算术操作**：1 = 进位输出
- **位/逻辑操作**：1 = 结果不为零
- **移动操作**：不改变 SCC

SCC 可用作扩展精度整数算术的进位输入，以及条件移动和分支的选择器。

### 3.4.7 向量比较：VCC 与 VCCZ

向量 ALU 比较指令（V_CMP）比较两个值并返回结果的位掩码，其中每位代表一个 lane（工作项）：1=通过，0=失败。此结果掩码即为向量条件码（VCC）。VCC 也为选定的整数 ALU 操作（进位输出）设置。

这些指令将此掩码写入 VCC、SGPR 或 EXEC，但不同时写入 EXEC 和 SGPR。Wave64 写入 64 位的 VCC、EXEC 或对齐的 SGPR 对；Wave32 仅写入 VCC、EXEC 或单个 SGPR 的低 32 位。

任何指令向 VCC 写入值时，硬件自动更新称为 VCCZ 的"VCC 摘要"位，指示当前 wave 大小的整个 VCC 掩码是否为零。Wave32 忽略 VCC[63:32]，只有位 [31:0] 对 VCCZ 有贡献。这对早期退出分支测试很有用。

VCC 物理上位于 SGPR 寄存器文件中的特定 SGPR 对中，因此当指令源自 VCC 时，这将计入给定指令可源自的 SGPR 总数限制。

Wave32 wave 可将任何 SGPR 用于掩码/进位/借位操作，但不能使用 VCC_HI 或 EXEC_HI。

### 3.4.8 SCRATCH_BASE

SCRATCH_BASE 是一个 64 位寄存器，保存此 wave 的暂存内存基地址指针。对于已分配暂存空间的 wave，wave 启动硬件用该 wave 唯一的暂存基地址初始化 SCRATCH_BASE。此寄存器只读，但在陷阱处理程序中可写。该值为字节地址且必须 256 字节对齐。若 wave 没有分配暂存空间，读取 SCRATCH_BASE 返回零。

### 3.4.9 硬件内部寄存器

这些寄存器是只读的，可通过 `S_GETREG_B32` 指令访问。它们返回硬件分配和状态信息。

**HW_ID1**

| 字段 | 位 | 描述 |
|------|-----|------|
| WAVE_ID | 4:0 | SIMD 内的 wave ID |
| SIMD_ID | 9:8 | WGP 内的 SIMD_ID：[0] = WGP 内的 CU，[1] = CU 内的 SIMD |
| WGP_ID | 13:10 | 物理 WGP ID |
| SA_ID | 16 | 着色器阵列 ID |

**HW_ID2**

| 字段 | 位 | 描述 |
|------|-----|------|
| QUEUE_ID | 3:0 | 队列 ID（也编码着色器阶段） |
| PIPE_ID | 5:4 | 管线 ID |
| ME_ID | 9:8 | 微引擎 ID：0=图形，1和2=ACE 计算 |
| STATE_ID | 14:12 | 状态上下文 ID |
| WG_ID | 20:16 | WGP 内的工作组 ID（0-31） |
| VM_ID | 27:24 | 虚拟内存 ID |

**IB_STS2** — 调试用指令缓冲区状态

**PERF_SNAPSHOT_DATA**

| 字段 | 位 | 描述 |
|------|-----|------|
| VALID | 0 | 1：快照数据已写入且被采样 wave 读取；0：无效 |
| WAVE_ISSUE | 1 | 1：wave 在快照循环发射了指令；0：未发射 |
| INST_TYPE | 5:2 | 已发射或 wave 想要发射的指令类型（0=VALU, 1=标量, 2=VMEM, 3=LDS, 4=LDS direct/Param, 5=导出, 6=消息, 7=屏障, 8=分支未跳转, 9=分支跳转, 10=跳转, 13=双 VALU, 14=Flat, 15=VALU-矩阵） |
| NO_ISSUE_REASON | 8:6 | 若指令未发射的原因（0=无可用指令, 1=等待 ALU 依赖, 2=等待 s_waitcnt, 3=仲裁失败, 4=sleep, 5=屏障等待, 6=其他, 7=内部操作） |
| WAVE_ID | 13:9 | 被快照的 wave 的 wave ID |

**PERF_SNAPSHOT_DATA2**

| 字段 | 位 | 描述 |
|------|-----|------|
| LOADcnt | 5:0 | wave 的 LOADcnt 值 |
| STOREcnt | 11:6 | wave 的 STOREcnt 值 |
| BVHcnt | 14:12 | wave 的 BVHcnt 值 |
| SAMPLEcnt | 20:15 | wave 的 SAMPLEcnt 值 |
| DScnt | 26:21 | wave 的 DScnt 值 |
| KMcnt | 31:27 | wave 的 KMcnt 值 |

> 所有 PERF_SNAPSHOT 寄存器只能由被快照的 wave 读取（其他 wave 读取为零）。应最后读取 PERF_SNAPSHOT_PC_HI，因为读取此寄存器会重置（解锁）性能快照寄存器以供下一次快照使用，wave 终止时也会重置。

### 3.4.10 陷阱与异常寄存器

每种类型的异常可通过设置或清除 TRAP_CTRL 寄存器中的位来独立启用或禁用。

陷阱临时 SGPR（TTMP*）对写入是特权的——只能在陷阱处理程序中（STATUS.PRIV=1）写入，用户着色器可读取。TMA 和 TBA 是只读的，可通过 `S_SENDMSG_RTN` 访问。

当触发陷阱时（用户发起、异常或主机发起），着色器硬件生成一条 `S_TRAP` 指令，将陷阱信息加载到一对 SGPR 中：

```
<等待未完成指令完成>
{ TTMP1, TTMP0 } = {TrapID[3:0], zeros, PC[47:0]}  // TrapID=0 除非异常是"S_TRAP"
```

**STATUS.TRAP_EN**

此位告诉着色器是否存在陷阱处理程序。若不存在，无论是浮点、用户还是主机发起的陷阱均不会触发。若 trap_en==0，所有陷阱和异常（致命异常除外）被忽略，`S_TRAP` 被硬件转换为 NOP。

**TRAP_CTRL（异常使能掩码）**

定义哪些异常来源在发生时使着色器跳转到陷阱处理程序。1=启用陷阱；0=禁用陷阱。MEMVIOL 和非法指令始终跳转到陷阱处理程序，不能被屏蔽。

| 位 | 异常 | 原因 | 结果 |
|----|------|------|------|
| 0 | alu_invalid | 无效：操作数对操作无效（0×∞, 0/0, sqrt(-x), 任何输入为 SNaN） | QNaN |
| 1 | alu_input_denorm | 输入非规格化：一个或多个操作数为次正规数 | 正常结果 |
| 2 | alu_float_div0 | 浮点除以零：浮点 X / 0 | 正确的有符号无穷大 |
| 3 | alu_overflow | 溢出：舍入结果大于最大有限数 | 取决于舍入模式 |
| 4 | alu_underflow | 下溢：精确或舍入结果小于最小正规数 | 次正规数或零 |
| 5 | alu_inexact | 不精确：有效操作的舍入结果与无限精度结果不同 | 操作结果 |
| 6 | alu_int_div0 | 整数除以零：整数 X / 0 | 未定义 |
| 7 | addr_watch | 地址监视：VMEM 或 SMEM 线程访问了"感兴趣的地址" | — |
| 8 | trap_on_wave_end | 在执行 S_ENDPGM 之前陷阱 | — |
| 9 | trap_after_inst | 每条指令后陷阱（除 S_ENDPGM 外） | — |

**EXCP_FLAG_PRIV 寄存器**

包含已发生异常的标志，为粘性标志（无论 TRAP_CTRL 设置如何都会累积），仅陷阱处理程序可写，用户着色器可读。

| 字段 | 位 | 描述 |
|------|-----|------|
| addr_watch | 3:0 | 地址监视 0、1、2 或 3 是否被命中的四个位 |
| memviol | 4 | 发生了内存违规 |
| save_context | 5 | 主机命令设置的位，指示此 wave 必须跳转到陷阱处理程序并保存上下文，陷阱处理程序应使用 S_SETREG 清除此位 |
| illegal_inst | 6 | 检测到非法指令 |
| host_trap | 7 | 陷阱处理程序被调用以处理主机陷阱 |
| wave_start | 8 | 陷阱处理程序在新 wave 的第一条指令前被调用 |
| wave_end | 9 | 陷阱处理程序在 wave 的最后一条指令后被调用 |
| perf_snapshot | 10 | 陷阱处理程序因随机性能快照被调用 |
| trap_after_inst | 11 | 陷阱处理程序因"指令后陷阱"模式被调用 |
| first_memviol_source | 31:30 | 第一个 MEMVIOL 的来源：0=指令缓存, 1=SMEM, 2=LDS, 3=VMEM |

**EXCP_FLAG_USER 寄存器**

包含已发生异常的标志，为粘性标志，用户着色器可写。

| 字段 | 位 | 描述 |
|------|-----|------|
| alu_invalid | 0 | ALU 浮点无效 |
| alu_input_denormal | 1 | ALU 浮点输入非规格化 |
| alu_float_div0 | 2 | ALU 浮点除以零 |
| alu_overflow | 3 | ALU 浮点溢出 |
| alu_underflow | 4 | ALU 浮点下溢 |
| alu_inexact | 5 | ALU 浮点不精确结果 |
| alu_int_div0 | 6 | 整数除以零 |
| buffer_oob | 30 | 缓冲区越界指示器（不触发陷阱） |
| lod_clamped | 31 | 纹理细节层次被夹（不触发陷阱） |

### 3.4.11 时间

有两种测量着色器时间的方法：

- **"TIME"**：以着色器核心时钟（64 位计数器）测量周期
- **"REALTIME"**：基于固定频率的持续运行时钟（通常为 100MHz）测量时间，提供 64 位值

**着色器时钟计数器：**

可通过以下方式读取：`S_GETREG_B32 S0, SHADER_CYCLES_LO` 或 `_HI`，返回计数器值的两半。

由于这是 64 位计数器但分两次 32 位读取，计数器可能在两次读取之间发生溢出。推荐步骤：
1. 用 S_GETREG_B32 读取计数器高位
2. 用 S_GETREG_B32 读取计数器低位
3. 重新读取计数器高位并与第一次读取的值比较，若不同则重复两次读取

此计数器不在不同 SIMD 间同步，只应用于测量同一 wave 内的时间差。读取计数器通过 SALU 处理，延迟约 8 个周期。

**固定频率时间：**

对于测量不同 wave 或 SIMD 之间的时间，或引用芯片空闲时也不停止计数的时钟，使用"REALTIME"。实时是来自时钟生成器的时钟计数器，以恒定速度运行，无论着色器或内存时钟速度如何。可通过以下方式读取：

```
S_SENDMSG_RTN_B64 S[2:3] REALTIME
S_WAIT_KMcnt == 0
```

---

## 3.5 初始 Wave 状态

在 wave 开始执行前，包括 SGPR 和 VGPR 在内的部分状态寄存器以来自状态数据、动态或派生数据（如插值或唯一的每 wave 数据）的值进行初始化。

未在本节明确列为已初始化的着色器状态不被初始化，这包括 LDS、此处未列出的 VGPR 和 SGPR。

### 3.5.1 EXEC 初始化

通常，EXEC 以 wave 中活动线程的掩码初始化。但有些情况下 EXEC 掩码初始化为零，表示此 wave 应不做任何工作并立即退出，称为"Null wave"（EXEC==0）。

### 3.5.2 SCRATCH_BASE 初始化

已分配暂存内存空间的 wave，其 SCRATCH_BASE 寄存器被初始化为指向全局内存中地址的指针。没有暂存空间的 wave 此寄存器初始化为零。

### 3.5.3 SGPR 初始化

SGPR 基于各种 `SPI_PGM_RSRC*` 或 `COMPUTE_PGM_*` 寄存器设置进行初始化。只加载已启用的值，并将其紧缩到连续 SGPR 中，跳过已禁用的值，无论加载了多少用户常量。不为对齐跳过任何 SGPR。

#### PS（像素着色器）SGPR 加载

**表 9. PS SGPR 加载**

| SGPR 顺序 | 描述 | 启用 |
|-----------|------|------|
| 前 0..32 | 用户数据寄存器 | SPI_SHADER_PGM_RSRC2_PS.user_sgpr |
| 然后 | `{bc_optimize, prim_mask[14:0], lds_offset[15:0]}` | N/A |
| 然后 | `{ps_wave_id[9:0], ps_wave_index[5:0]}` | SPI_SHADER_PGM_RSRC2_PS.wave_cnt_en |
| 然后 | Provoking 顶点信息：`{prim15[1:0], prim14[1:0], ..., prim0[1:0]}` | SPI_SHADER_PGM_RSRC1_PS.LOAD_PROVOKING_VTX |

#### GS（几何着色器）SGPR 加载

ES 和 GS 作为 GS 类型的组合 wave 启动。前 8 个 SGPR 自动初始化（未使用的值写为零）。

**表 10. GS SGPR 加载**

| SGPR # | GS（FAST_LAUNCH != 2） | GS（FAST_LAUNCH == 2） | 启用 |
|--------|------------------------|------------------------|------|
| 0 | GS 程序地址 [31:0] | GS 程序地址 [31:0] | 自动加载 |
| 1 | GS 程序地址 [63:32] | GS 程序地址 [63:32] | 自动加载 |
| 2 | `{1'b0, gsAmpPrimPerGrp[8:0], 1'b0, esAmpVertPerGrp[8:0], ordered_wave_id[11:0]}` | `32'h0` | 自动加载 |
| 3 | `{TGsize[3:0], WaveInGroup[3:0], 8'h0, gsInputPrimCnt[7:0], esInputVertCnt[7:0]}` | `{TGsize[3:0], WaveInGroup[3:0], 24'h0}` | 自动加载 |
| 4 | Off-chip LDS 基地址 [31:0] | `{TGID_Y[15:0], TGID_X[15:0]}` | SPI_SHADER_PGM_RSRC2_GS.oc_lds_en |
| 5 | `{17'h0, attrSgBase[14:0]}` | `{TGID_Z[15:0], 1'b0, attrSgBase[14:0]}` | — |
| 8 - (至多) 39 | GS 着色器用户数据寄存器 | GS 着色器用户数据寄存器 | SPI_SHADER_PGM_RSRC2_GS.user_sgpr |

#### HS（曲面细分着色器）SGPR 加载

LS 和 HS 作为 HS 类型的组合 wave 启动，前 8 个 SGPR 自动初始化。

**表 11. HS（LS）SGPR 加载**

| SGPR # | 描述 | 启用 |
|--------|------|------|
| 0 | HS 程序地址低位 [31:0] | SPI_SHADER_USER_DATA_LO_HS |
| 1 | HS 程序地址高位 [63:32] | SPI_SHADER_USER_DATA_HI_HS |
| 2 | Off-chip LDS 基地址 [31:0] | 自动加载 |
| 3 | `{first_wave[0], lshs_TGsize[6:0], lshs_PatchCount[7:0], HS_vertCount[7:0], LS_vertCount[7:0]}` | 自动加载 |
| 4 | TF 缓冲区基地址 [17:0] | 自动加载 |
| 5 | `{27'b0, wave_id_in_group[4:0]}` | SPI_SHADER_PGM_RSRC2_HS.scratch_en |
| 8 - (至多) 39 | HS 着色器用户数据寄存器 | SPI_SHADER_PGM_RSRC2_HS.user_sgpr |

#### CS（计算着色器）SGPR 加载

计算着色器初始化用户 SGPR 以及陷阱临时 SGPR。

**表 12. CS SGPR 加载**

| SGPR # | 描述 | 启用 |
|--------|------|------|
| 前 0..16 | 用户数据寄存器 | COMPUTE_PGM_RSRC2.user_sgpr |
| 然后 | `{14'h0, ordered_append_term[11:0], work_group_size_in_waves[5:0]}` | COMPUTE_PGM_RSRC2.tg_size_en |
| TTMP6 | 未使用 | — |
| TTMP7 | 工作组网格 `{Z[15:0], Y[15:0]}` | GridYZvalid==1 时加载 |
| TTMP8 | `{DebugMark[0], GridYZvalid[0], waveIDinGroup[4:0], dispatchIndex[24:0]}` | 所有 CS wave 初始化此项 |
| TTMP9 | 工作组网格 X[31:0] | 所有 CS wave 初始化此项 |

- 若着色器引用 GridY 或 GridZ，两者均初始化且 GridYZvalid=1；若两者均未使用则均不初始化
- 线程位置计算：`thread_id_x = work_group_grid_X * work_group_dim_x + thread_idx_in_workgroup_X`（VGPR0）

### 3.5.4 VGPR 初始化

以下表格显示 wave 启动前可能被初始化的 VGPR。

| 阶段 | HS（+LS）组合 | GS（+ES） ES 是 DS 组合 | GS（+ES） ES 是 VS 组合 | GS（+ES）Fast Launch 组合 | CS |
|------|--------------|-------------------------|-------------------------|--------------------------|-----|
| VGPR0 | HS: Patch ID；LS: 顶点索引 | GS: `[7:0]=顶点0相对索引, [8]=边标志0, [16:9]=顶点1相对索引, [17]=边标志1, [25:18]=顶点2相对索引, [26]=边标志2, [31:27]=GS 实例ID` | 同左 | X, Y, Z | `{2'b0, Z[9:0], Y[9:0], X[9:0]}` — 工作组内工作项 X/Y/Z 位置 |
| VGPR1 | HS: `[7:0]=相对 patch ID(0..255), [12:8]=控制点 ID` | GS: Primitive ID | GS: Primitive ID | 未使用 | — |
| VGPR2 | LS: 顶点缓冲区内当前顶点索引 | 未使用 | GS 及相邻图元类型（顶点 3-5 信息） | 未使用 | — |
| VGPR3 | LS: 实例 ID | ES: U[fp32] | ES: 顶点索引 | 未使用 | — |
| VGPR4 | 未使用 | ES: V[fp32] | ES: 实例 ID | 未使用 | — |
| VGPR5 | 未使用 | ES: 相对 Patch ID | 未使用 | 未使用 | — |
| VGPR6 | 未使用 | ES: Patch ID | 未使用 | 未使用 | — |

#### 像素着色器 VGPR 输入控制

像素着色器 VGPR 输入加载相当复杂。SPI 中有一个 CAM，将 VS 输出映射到 PS 输入。需要加载的 PS 输入按以下顺序加载：

I persp sample → J persp sample → I persp center → J persp center → I persp centroid → J persp centroid → I/W → J/W → 1/W → I linear sample → J linear sample → I linear center → J linear center → I linear centroid → J linear centroid → Line stipple → X float → Y float → Z float → W float → Facedness → Ancillary (RTA, ISN, PT, eye-id) → Sample mask → X/Y fixed

两个寄存器（`SPI_PS_INPUT_ENA` 和 `SPI_PS_INPUT_ADDR`）控制 IJ 计算的启用和 PS wave 的 VGPR 初始化规范。

**完整加载时的 PS VGPR 映射：**

| 字段名 | IJ/VGPR 项 | 位数 | VGPR 目标 |
|--------|-----------|------|----------|
| PERSP_SAMPLE_ENA | PERSP_SAMPLE I / J | 32 | VGPR0 / VGPR1 |
| PERSP_CENTER_ENA | PERSP_CENTER I / J | 32 | VGPR2 / VGPR3 |
| PERSP_CENTROID_ENA | PERSP_CENTROID I / J | 32 | VGPR4 / VGPR5 |
| PERSP_PULL_MODEL_ENA | PERSP_PULL_MODEL I/W / J/W / 1/W | 32 | VGPR6 / VGPR7 / VGPR8 |
| LINEAR_SAMPLE_ENA | LINEAR_SAMPLE I / J | 32 | VGPR9 / VGPR10 |
| LINEAR_CENTER_ENA | LINEAR_CENTER I / J | 32 | VGPR11 / VGPR12 |
| LINEAR_CENTROID_ENA | LINEAR_CENTROID I / J | 32 | VGPR13 / VGPR14 |
| LINE_STIPPLE_TEX_ENA | LINE_STIPPLE_TEX | 32 | VGPR15 |
| POS_X_FLOAT_ENA | POS_X_FLOAT | 32 | VGPR16 |
| POS_Y_FLOAT_ENA | POS_Y_FLOAT | 32 | VGPR17 |
| POS_Z_FLOAT_ENA | POS_Z_FLOAT | 32 | VGPR18 |
| POS_W_FLOAT_ENA | POS_W_FLOAT | 32 | VGPR19 |
| FRONT_FACE_ENA | FRONT_FACE | 32 | VGPR20 |
| ANCILLARY_ENA | RTA_Index[28:16], Sample_Num[11:8], Eye_id[7], VRSrateY[5:4], VRSrateX[3:2], Prim Typ[1:0] | 29 | VGPR21 |
| SAMPLE_COVERAGE_ENA | SAMPLE_COVERAGE | 16 | VGPR22 |
| POS_FIXED_PT_ENA | Position {Y[16], X[16]} | 32 | VGPR23 |

若 PS_INPUT_ADDR == PS_INPUT_ENA，则禁用的项被跳过，PS VGPR 向 VGPR0 紧缩。

### 3.5.5 LDS 初始化

LDS 不被初始化；其内容未定义。像素着色器的顶点属性由 SPI 写入 LDS，并在 `STATUS.LDS_PARAM_RDY` 设置后可通过 `DS_PARAM_LOAD` 和 `DS_DIRECT_LOAD` 读取。
