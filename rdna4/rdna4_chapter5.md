# 第五章：程序流程控制

程序流程控制使用标量 ALU 指令编程，包括循环、分支、子程序调用和陷阱。程序使用 SGPR 存储分支条件和循环计数器，常量可直接从标量常量缓存获取到 SGPR。

本章的指令控制着色器程序的优先级和终止，并为陷阱处理程序提供支持。

---

## 5.1 程序流程控制指令格式

这些指令以以下微码格式之一编码：

| 名称 | 大小 | 功能 |
|------|------|------|
| SOPP | 32 位 | 带 16 位立即常量的 SALU 程序控制操作 |
| SOP1 | 32 位 | 带 1 个输入的 SALU 操作 |

各格式使用的字段：

| 字段 | 描述 |
|------|------|
| OP | 操作码：要执行的指令 |
| SDST | 目标 SGPR、M0、NULL 或 EXEC |
| SSRC0 | 第一个源操作数 |
| SIMM16 | 有符号 16 位整数立即常量 |

---

## 5.2 程序控制指令

**表 17. Wave 终止与陷阱**

| 指令 | 描述 |
|------|------|
| S_ENDPGM | 终止 wave。可出现在着色器程序的任何位置，可出现多次。 |
| S_ENDPGM_SAVED | 因上下文保存而终止 wave，仅供陷阱处理程序使用。 |
| S_TRAP | 跳转到陷阱处理程序并传入 SIMM[3:0] 中的 4 位陷阱 ID。等待未完成指令完成，然后：`{TTMP1, TTMP0} = {TrapID[3:0], zeros, PC[47:0]}`；`PC = TBA`；`PRIV = 1`。主机陷阱使着色器硬件生成 S_TRAP 指令，保存的 PC 指向 S_TRAP 指令本身。TRAPID 0 保留，不应使用。 |
| S_RFE_B64 | 从异常（陷阱处理程序）返回并继续。`MOVE PC, <src>; STATUS.PRIV = 0`。只能在陷阱处理程序中使用。 |
| S_SETHALT | 将 HALT 或 FATAL_HALT 位设置为 SIMM16[0] 的值。SIMM16[0]: 1=停止, 0=恢复。SIMM16[2]: 1=设置 FATAL_HALT, 0=设置 HALT。用户着色器可设置 HALT 位，但不能设置 FATAL_HALT 位；陷阱处理程序（priv=1）允许设置任一位。HALT==1 在 PRIV==0 时停止 wave，在 PRIV==1 时忽略。FATAL_HALT==1 无论 PRIV 值都停止 wave，一旦 FATAL_HALT 就无法自行解除——只有主机命令可以。 |

**表 18. 依赖、延迟和调度指令**

| 指令 | 描述 |
|------|------|
| S_NOP | NOP，重复 SIMM16[6:0] 次（1..128），类似 S_SLEEP 的简短版本。 |
| S_SLEEP | 使 wave 休眠约 `64*SIMM16[6:0]` 个时钟，或可被 S_WAKEUP 提前唤醒。"s_sleep 0"使 wave 休眠 0 个周期。若 SIMM16[15]==1，则永久休眠（直到唤醒、陷阱或终止）。 |
| S_SLEEP_VAR | 使 wave 休眠约 `SGPR_value[6:0]*64` 个周期，也可被 S_WAKEUP 唤醒。 |
| S_WAKEUP | 使工作组中的一个 wave 向同一工作组中的所有其他 wave 发信号，使其从 S_SLEEP/S_SLEEP_VAR 提前唤醒。若 wave 未在休眠，则此指令对其无效。 |
| S_SETPRIO | 设置 2 位 USER_PRIO（用户可设置 wave 优先级）。0=低，3=高。总 wave 优先级为：`{MIN(3,(SysPrio[1:0] + UserPrio[1:0])), WaveAge[3:0]}`。 |
| S_CLAUSE | 开始由 s_clause 之后的指令类型匹配的指令子句。子句长度为：(SIMM16[5:0] + 1)，子句必须在 2 至 63 条指令之间。SIMM16[5:0] 必须为 1-32，不能为 0 或 63。 |
| S_BARRIER_SIGNAL / S_BARRIER_SIGNAL_ISFIRST | 发信号 wave 已到达屏障，同步工作组内 wave。不在工作组中的 wave（或工作组大小=1 wave）将此视为 S_NOP。ISFIRST 变体返回 SCC，若此为第一个发信号的 wave 则为 1。 |
| S_BARRIER_WAIT | 等待工作组中的所有 wave 发信号屏障后继续。若工作组中的所有 wave 尚未全部创建，等待整个工作组完成。已结束的 wave 不阻止屏障满足。不在工作组中的 wave 将此视为 S_NOP。 |
| S_GET_BARRIER_STATE | 获取当前屏障状态，通常用于上下文切换。 |

**表 19. 控制指令**

| 指令 | 描述 |
|------|------|
| S_VERSION | 不执行任何操作（视为 S_NOP），可用作代码注释，使用 SIMM16 字段指示着色器编译的硬件版本。 |
| S_CODE_END | 视为非法指令。用于在着色器末尾进行填充。 |
| S_SENDMSG | 向中断处理程序或专用硬件发送消息。SIMM[9:0] 是保存消息类型的立即值，M0 中保存消息负载。 |
| S_SENDMSG_RTN_B32 / S_SENDMSG_RTN_B64 | 发送请求某些数据返回到 SGPR 的消息。使用 KMcnt 跟踪数据何时返回。SDST = 返回目标 SGPR（_B64 为对齐的 SGPR 对）；SSRC0 = 请求数据的枚举（非 SGPR）。若用于写入 VCC，则 VCCZ 未定义。 |
| S_SENDMSGHALT | 执行 S_SENDMSG 然后 HALT。 |
| S_ICACHE_INV | 无效化与此 wave 关联的 WGP 的第一级着色器指令缓存。 |

**表 20. 浮点算术状态指令**

| 指令 | 描述 |
|------|------|
| S_ROUND_MODE | 从立即值设置舍入模式：SIMM16[3:0] |
| S_DENORM_MODE | 从立即值设置非规格化模式：SIMM16[3:0] |

---

## 5.3 指令子句

指令子句是一系列相同类型的指令，将被不间断地连续执行。通常硬件可以交错来自不同 wave 的指令，但子句可用于覆盖此行为，强制硬件在子句持续期间仅为给定指令类型服务一个 wave，即使这使执行硬件空闲。

子句使用 `S_CLAUSE` 指令定义和启动，且必须只包含单一类型的指令。子句类型由紧跟在 s_clause 之后的指令类型隐式定义。

**子句类型：**
- 非 Flat 加载：图像、缓冲区、全局、暂存、BVH 和采样/gather
- 非 Flat 存储：图像、缓冲区、全局、暂存
- 非 Flat 原子：图像、缓冲区、全局、暂存
- Flat 加载、Flat 存储、Flat 原子
- LDS 索引加载、存储、原子、BVH_stack（全为同一类型）
- SMEM
- VALU

**子句内规则：**

| 在子句内非法 | 在任意类型子句内合法（但不能作为第一条指令） |
|-------------|----------------------------------------------|
| 不同类型的指令 | S_NOP |
| S_CLAUSE | S_SETPRIO* |
| S_ENDPGM | S_VERSION |
| SALU | S_WAIT_EVENT |
| 分支/跳转 | S_WAIT_*CNT, S_WAIT_IDLE |
| S_SENDMSG* | S_ICACHE_INV |
| S_SLEEP, S_SLEEP_VAR | — |
| S_SETHALT | — |
| DS_PARAM_LOAD, DS_DIRECT_LOAD | — |
| 导出 | — |
| VALU: FP64 操作 | — |
| S_DELAY_ALU | — |

S_TRAP 在子句内合法，即使作为 S_CLAUSE 之后的第一条指令也是合法的。S_TRAP 结束子句。

注意：V_S_* 指令（伪标量超越函数）是 VALU 操作，可在 VALU 子句中使用。

若 VALU 子句中的第一条指令 EXEC==0，则忽略整个子句，指令将单独发射（如同没有子句）。若 VALU 子句以 EXEC!=0 开始但在子句中途 EXEC 变为零，则子句继续直到指定子句的最后一条指令。

若在 S_CLAUSE 之前需要 S_DELAY_ALU，顺序必须为：
```
S_DELAY_ALU   // 不能紧接在 S_CLAUSE 之后——那条指令声明子句类型
S_CLAUSE
<子句中第一条指令>
```

### 5.3.1 子句中断

以下条件可中断子句：
1. VALU 异常（陷阱）中断 VALU 子句
2. 主机命令发送给 wave（停止、恢复、单步等）中断所有活动子句。上下文保存中断受影响 wave 的子句
3. 任何使 wave 跳转到陷阱处理程序的操作都中断子句（包括上下文保存）。进入 HALT（包括主机发起的单步）的 wave 可能中断子句

---

## 5.4 发送消息类型

`S_SENDMSG` 用于向固定功能硬件、主机发送消息，或请求将值返回给 wave。`S_SENDMSG` 在 SIMM16 字段中编码消息类型，在 M0 中编码消息负载。

`S_SENDMSG_RTN_B*` 在 SSRC0 字段（在指令字段中，不在 SGPR 中）编码消息类型，在 M0 中编码负载（如有），在 SDST 中编码目标 SGPR。`S_SENDMSG_RTN_B*` 指令向着色器返回数据：KMcnt 递增 2，然后消息发出时递减 1，数据返回时再递减 1。这允许用户使用 `S_WAIT_KMCNT==0` 等待数据返回。完成通过 KMcnt 跟踪。

**表 21. S_SENDMSG 消息**

| 消息 | SIMM16[7:0] | 负载 |
|------|-------------|------|
| INTERRUPT | 0x01 | 软件生成的中断。M0[23:0] 携带用户数据，还发送 wave_id、wgp_id 等 ID |
| HS_TESSFACTOR | 0x02 | 指示此 HS 工作组中所有 patch 的 HS 曲面细分因子全为零或全为一。M0[0]: 0="所有线程曲面细分因子为零", 1="所有线程曲面细分因子为一" |
| DEALLOC_VGPRS | 0x03 | 释放此 wave 的所有 VGPR 和暂存内存，允许另一个 wave 在此 wave 结束前分配这些 VGPR |
| GS_ALLOC_REQ | 0x09 | 在参数缓存中请求 GS 空间。M0[9:0]=顶点数量, M0[22:12]=图元数量 |

**表 22. S_SENDMSG_RTN 消息**

| 消息 | SIMM16[7:0] | 负载 |
|------|-------------|------|
| RTN_GET_DOORBELL | 0x80 | 获取与此 wave 关联的门铃 ID |
| RTN_GET_DDID | 0x81 | 获取与此 wave 关联的绘制或调度 ID |
| RTN_GET_TMA | 0x82 | 获取陷阱内存地址：[31:0] 或 [63:0]，取决于请求大小 |
| RTN_GET_REALTIME | 0x83 | 获取固定频率（REFCLK）时间计数器的值：[31:0] 或 [63:0] |
| RTN_SAVE_WAVE | 0x84 | 用于上下文切换，指示此 wave 准备好进行上下文保存。只有陷阱处理程序可发送此消息 |
| RTN_GET_TBA | 0x85 | 获取陷阱基地址 [31:0] 或 [63:0] |
| RTN_GET_SE_HW_ID | 0x87 | 获取逻辑着色器引擎 ID：data[3:0]=SE_ID; data[11:8]=AID_ID |
| RTN_ILLEGAL_MSG | 0xFF | 带数据返回的非法消息 |

---

## 5.5 分支

分支使用以下标量 ALU 指令之一完成，影响整个 wave，而不仅是单个工作项。"SIMM16" 是一个符号扩展的 16 位整数常量，对于分支被视为 DWORD 偏移。

PC 值：S_BRANCH、S_CBRANCH、S_SWAPPC、S_CALL 和 S_GETPC 对指向当前指令之后一条指令的 PC 进行操作。因此"S_BRANCH 0"类似于 NOP，而不是无限循环。

**表 23. 分支指令**

| 指令 | 描述 |
|------|------|
| S_BRANCH | 无条件分支。`PC = PC + (SIMM16 * 4)` |
| S_CBRANCH_\<cond\> | 条件分支，仅在条件为真时分支。`if (cond) PC = PC + (SIMM16*4); else NOP`。若 SIMM16=0，分支跳到下一条指令。条件：SCC1, SCC0, VCCZ, VCCNZ, EXECZ, EXECNZ |
| S_SETPC_B64 | 直接从 SGPR 对设置 PC：`PC = SGPR对` |
| S_SWAPPC_B64 | 交换当前 PC（指向此指令后面的指令）与 SGPR 对中的地址。`SWAP(PC, SGPR对)` |
| S_GETPC_B64 | 获取当前 PC 值（零扩展），不引起分支。`SGPR对 = 下一条指令的PC` |
| S_CALL_B64 | 跳转到子程序并保存返回地址。`SGPR对 = PC; PC = PC+SIMM16*4` |

对于条件分支，分支条件可由标量或向量操作确定。标量比较操作设置标量条件码（SCC），可用作条件分支条件；向量比较操作设置 VCC 掩码，VCCZ 或 VCCNZ 可用于确定是否分支。

---

## 5.6 工作组与屏障

工作组是运行在同一 WGP 上的可同步和共享数据的 wave 集合。最多 1024 个工作项（16 个 wave64 或 32 个 wave32）可组合成一个工作组。当多个 wave 在一个工作组中时，屏障指令可使每个 wave 等待所有其他 wave 到达相同指令，然后所有 wave 继续。

屏障操作分为两部分：信号（到达）和等待。每个 wave 先发出 `S_BARRIER_SIGNAL` 表示其他 wave 可以继续，之后 wave 发出 `S_BARRIER_WAIT` 等待工作组中的每个 wave 也已发出 `S_BARRIER_SIGNAL`。当所有成员 wave 发信号后，屏障重置，wave 可通过 `S_BARRIER_WAIT` 继续。

- **屏障有效**：一旦工作组中的所有 wave 都被创建，屏障即有效。在此之前，工作组中 wave 的数量未知
- **屏障完成**：当工作组中的每个 wave 都发信号（或终止）时，屏障完成
- 屏障在有效之前无法完成，但 wave 可以在屏障有效之前发信号
- wave 允许提前终止而不是发信号，剩余 wave 仍可使用屏障；当剩余 wave 都发信号后，可通过 `S_BARRIER_WAIT`
- 单 wave 工作组和不在工作组中的 wave 将所有屏障指令视为 S_NOP

**所有工作组 wave 的屏障类型：**
- **工作组屏障**：当工作组中的每个 wave 都发信号或终止时完成
- **陷阱屏障**：与工作组屏障相同，但专供陷阱处理程序使用。若用户着色器尝试发信号或等待，操作被忽略

### 5.6.1 屏障状态

屏障由两个计数组成：
- **成员计数**：完成屏障需要发信号的 wave 数量。工作组和陷阱屏障初始化为工作组大小
- **已信号计数**：已发信号屏障的 wave 数量。第一个工作组 wave 创建时重置为零，屏障完成后也重置为零

当最后一个 wave 发信号使屏障完成时，屏障向工作组广播完成，然后重置 signaledCount=0。陷阱和工作组屏障只有在有效后才能完成。

Wave 的屏障相关状态位：
- **barrierComplete**：记录工作组屏障是否之前已完成但 wave 尚未等待。若 S_BARRIER_WAIT 执行时此位为 1，则立即执行并将此位重置为零；若此位为零则 wave 等待直到屏障完成
- **trapBarrierComplete**：同上，但用于陷阱屏障

### 5.6.2 屏障指令

屏障由 `S_BARRIER_SIGNAL{_ISFIRST}`、`S_BARRIER_WAIT` 和 `S_GET_BARRIER_STATE` 控制。

**表 24. 屏障 ID**

| 屏障 # | 屏障 | 描述 |
|--------|------|------|
| -2 | 陷阱屏障 | 专供陷阱处理程序使用的屏障 |
| -1 | 工作组屏障 | 同步工作组中所有 wave 的屏障 |

**表 25. 屏障指令**

| 指令 | 参数 | 描述 |
|------|------|------|
| S_BARRIER_SIGNAL / S_BARRIER_SIGNAL_ISFIRST | Bar# | 发信号屏障 \<Bar#\>。Bar# 可以是内联常量或 M0（M0[4:0] 提供屏障 ID#）。S_BARRIER_SIGNAL_ISFIRST 还向 wave 返回 SCC：若为此轮第一个发信号 wave 则为 1，否则为零。使用 KMcnt 跟踪工作组的 SCC 何时返回。SCC 只应在屏障完成后（S_BARRIER_WAIT 之后）读取。 |
| S_BARRIER_WAIT | Bar# | 等待屏障完成，Bar# 必须为内联常量。 |
| S_GET_BARRIER_STATE | Bar#, SDST | 获取屏障状态并返回到 SGPR：`SDST = { 0, signalCnt[6:0], 5'b0, memberCnt[6:0], 3'b0, valid }`。使用 KMcnt 跟踪完成（+/-1）。Bar# 来自 M0[4:0] 或内联常量。 |

### 5.6.3 陷阱与异常中的屏障

陷阱和异常在屏障使用期间可能发生，表现正常：wave 等待空闲并跳转到陷阱处理程序。若 wave 在 `S_BARRIER_WAIT` 处等待时发生异常，wave 保存 `S_BARRIER_WAIT` 的 PC 并跳转到陷阱处理程序，从陷阱处理程序返回时重新执行 `S_BARRIER_WAIT`。

### 5.6.4 上下文切换

陷阱处理程序负责保存和恢复 wave 和屏障单元状态。陷阱处理程序以 `S_BARRIER_SIGNAL` 和等待（陷阱屏障）开始，确保来自用户着色器的任何先前 `S_BARRIER_SIGNAL` 已完成。陷阱处理程序然后可读出并保存屏障单元状态（信号计数），并在通过一系列 `S_BARRIER_SIGNAL` 恢复上下文时稍后恢复。

---

## 5.7 数据依赖解析

着色器硬件可以解析大多数非内存数据依赖，但内存和导出依赖需要着色器程序显式处理。本节描述着色器程序员为获得正确结果必须遵循的规则。

`S_WAIT_*CNT` 指令用于解析数据冒险。每个 wave 有计数器统计每个 wave 未完成的内存指令数量，计数器在指令发射时递增，在指令完成时递减。着色器编写者可调度长延迟指令，执行不相关工作，并指定何时需要长延迟操作的结果。

**示例：**

```
GLOBAL_LOAD_B32 V0, V[4:5], 0x0   // 加载 memory[{V5,V4}] 到 V0，LOADcnt++
GLOBAL_LOAD_B32 V1, V[4:5], 0x8   // 加载 memory[{V5,V4}+8] 到 V1，LOADcnt++
S_WAIT_LOADcnt <= 1               // 等待第一个 global_load 完成
V_MOV_B32  V9, V0                 // 将 V0 移到 V9
```

同类型指令按发射顺序返回（SMEM 除外），但不同类型的指令可以乱序完成。导出请求在每类目标内按顺序完成，但不同类型之间可以乱序完成（类别为：MRT 和 Z、位置、图元数据、双源混合）。

SALU 相关等待规则：若 64 位 SGPR 对（如 s[0:1]）的一个子集被 VALU 指令读取，则对同一 SGPR 对的任何子集的后续写入之后读取必须按如下方式保护：
- VALU 写入 VCC 后 VALU 读取 VCC：`s_wait_alu 0xFFFD`
- VALU 写入 SGPR 后 VALU 读取 SGPR：`s_wait_alu 0xF1FF`
- SALU 写入 SGPR 或 VCC 后 SALU 或 VALU 读取：`s_wait_alu 0xFFFE`

```
v_mov_b32 v0, s1        // VALU 读取 s[0:1] 对
s_mov_b32 s0, 0x42      // SALU 写入 s0
s_wait_alu 0xFFFE       // 读取 s0 必须等待 SALU 完成
s_mov_b32 s2, s0        // SALU 读取 s0
```

### 5.7.1 内存依赖计数器

`S_WAIT_*CNT` 等待使用指定计数器的未完成指令完成以避免数据冒险。硬件通过在任何指令使计数器溢出时停止该指令的发射，来防止计数器溢出。

| 计数器 | 大小 | 描述 |
|--------|------|------|
| LOADcnt | 6 | 向量内存计数：图像、缓冲区、flat、暂存、全局（加载和带返回值的原子）、global_inv。每次发射 VIMAGE/VBUFFER/FLAT 格式的向量内存加载或带返回原子指令时递增 1 |
| SAMPLEcnt | 6 | 向量内存采样计数。每个采样指令发射时递增 1，按序完成时递减 |
| BVHcnt | 3 | 向量内存 BVH 指令计数，每个 BVH 指令递增/递减 1 |
| STOREcnt | 6 | 向量内存存储计数：图像、缓冲区、flat、暂存、全局（存储和无返回原子）、global_wb/wbinv。数据写入 SCOPE 位指定的内存层次结构级别时递减 |
| DScnt | 6 | 数据存储计数（LDS 和 Flat）。LDS 指令发射时递增，LDS 加载或带返回原子数据返回到 VGPR 时递减，LDS 存储数据写入 LDS 时递减；Flat 指令每次发射时递增，LDS 部分完成时递减 |
| KMcnt | 5 | 常量缓存和消息计数。32 位及更小加载递增 1，更大加载递增 2；每个 S_SENDMSG 递增 1；每个从数据缓存返回的 DWORD 递减 1（SMEM 乱序返回，因此只有 S_WAIT_KMcnt 0 是合法值） |
| EXPcnt | 3 | VGPR 导出计数和 LDS direct/param 加载计数。导出发射时递增，最后一个导出周期授权执行时递减；DS_PARAM_LOAD 和 DS_DIRECT_LOAD 发射时递增，完成写入 VGPR 时递减 |

#### 5.7.1.1 标量内存

SMEM 指令使用 KMcnt 计数器。仅读/写单个 DWORD 时递增 1，否则递增 2。标量数据缓存是只读的，与第一或第二级向量内存缓存不一致；必须手动无效化缓存以保持一致性，发送缓存无效化后必须使用 `S_WAIT_KMcnt<=0` 确认完成。

#### 5.7.1.2 向量内存——图像、缓冲区、暂存、全局

- LOADcnt：统计类型为加载、带返回原子和缓存无效化的未完成指令
- SAMPLEcnt：统计类型为采样（含 gather）的未完成指令
- STOREcnt：统计已发射但未完成的内存存储和无返回内存原子数据指令

对 STOREcnt 进行等待只在着色器必须知道先前写入已提交到内存才能继续时才有必要。存储后立即覆盖源 GPR 没有冒险，从一个 wave 向一个地址存储后从同一地址加载（同一 wave）也没有冒险——加载和存储到同一地址保持顺序。

#### 5.7.1.3 向量内存——Flat

由于 Flat 指令可能有工作项访问向量内存，而其他工作项访问 LDS，Flat 指令同时使用 DScnt 和 LOADcnt/STOREcnt，发射时两者都递增，完成时都递减，但在不同时间：DScnt 在 LDS 完成其工作项时递减，LOADcnt/STOREcnt 在向量内存完成其工作项时递减。

**Write-After-Write 冒险示例（应避免）：**
```
A: flat_load_B32 v10, v[0:1]
B: flat_load_B32 v10, v[2:3]
```
若"A"读向量内存而"B"读 LDS，B 可能先完成，然后 A 覆盖 B 的数据，导致 Write-After-Write 冒险。

#### 5.7.1.4 LDS

LDS 操作使用 DScnt：发射时递增，完成时递减。LDS 参数和直接加载使用 EXPcnt 跟踪完成（VGPR 已写入）。

#### 5.7.1.5 导出

导出使用 EXPcnt 计数器，防止 Write-after-Read 冒险。导出在每种类型内按序完成，但不同类型之间可能乱序。着色器程序必须使用 `S_WAIT_EXPcnt 0` 才能覆盖 EXEC 掩码。发出导出请求时读取 M0，因此 M0 上没有 WAR 冒险。

#### 5.7.1.6 消息

消息使用 KMcnt：发射 `S_SENDMSG` 时递增，消息发出 WGP 时递减。使用 `S_WAIT_KMcnt 0` 确保消息已发出。

### 5.7.2 特定依赖情况

以下情况需要在 `S_GETREG` 之前有 S_NOP 或其他任何指令：
- `S_SETHALT` 之后 `S_GETREG`（STATE_PRIV 或 STATUS）
- `S_SETPRIO` 之后 `S_GETREG STATE_PRIV`

---

## 5.8 ALU 指令软件调度

着色器程序可包含指令，延迟 ALU 指令发射，以尝试避免因过近发射相关指令而导致的流水线停顿。

这通过 `S_DELAY_ALU` 指令完成："在相对于之前的 VALU 指令插入延迟"。编译器可插入 `S_DELAY_ALU` 指令，指示可能受益于在相关指令之间插入额外空闲周期的数据依赖。

此指令在想要延迟的指令之前插入，指定此指令依赖于哪些先前指令，硬件然后确定添加的延迟周期数。

此指令是可选的——不是正确操作所必需的。只有在必要时才应插入以避免依赖停顿。

`S_DELAY_ALU` 打包了两个延迟值到一条指令中，带有"跳过"指示符，使两个延迟的指令不需要背靠背：

```
S_DELAY_ALU InstID1[4], Skip[3], InstID0[4]  // 打包到 SIMM16
```

**INSTID** 向后计数 N 条已发射的 VALU 指令。EXEC==0 因而被跳过的 VALU 指令在计算 INSTID 时确实计入。

**SKIP** 计算在具有第二个依赖的指令之前跳过的指令数量，所有类型的所有指令都计入跳过计数。

`S_DELAY_ALU` 不应在 VALU 子句中使用。

**表 27. S_DELAY_ALU 指令代码**

| DEP 代码 | 含义 | SKIP 代码 | 含义 |
|---------|------|----------|------|
| 0 | 无依赖 | 0 | 相同操作，两个 DEP 代码均应用于下一条指令 |
| 1-4 | 依赖于之前 1-4 条 VALU 指令 | 1 | 无跳过，DEP0 应用于下一条指令，DEP1 应用于其后一条指令 |
| 5-7 | 依赖于之前 1-4 条超越函数 VALU | 2 | 跳过 1，DEP0 应用于下一条指令，DEP1 应用于其后 2 条指令 |
| 8 | 保留 | 3-5 | DEP0 和 DEP1 之间跳过 2-4 条指令 |
| 9-11 | 等待之前 1-3 个 SALU 操作的周期 | 6 | 保留 |

代码 9-11：SALU 操作通常在单个周期内完成，因此等待 1 个周期大致相当于在继续之前等待 1 个 SALU 操作执行。

超越函数 VALU 操作包括：SIN、COS、SQRT、RCP、RSQ、EXP 和 LOG。
