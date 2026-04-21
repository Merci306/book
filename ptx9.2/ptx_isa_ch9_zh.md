# 第 9 章：指令集（Instruction Set）

> **说明**：本章是 PTX ISA 9.2 的核心，共 654 页（PDF 第 133–786 页）。本文件以**分类索引**形式整理，每类指令给出中文说明和代表性示例，不逐条翻译每条指令的完整规格表。

---

## 9.1 指令描述格式与语义

每条指令的规格描述包含以下字段：

| 字段 | 含义 |
|---|---|
| Syntax | PTX 汇编语法 |
| Description | 功能描述 |
| Semantics | 伪代码形式的精确语义 |
| Notes | PTX ISA 版本、目标架构要求 |
| Examples | 代码示例 |

---

## 9.2 PTX 指令概述

PTX 指令格式：
```
[@p] opcode{.modifier} d, a, b, c;
```
- `@p` — 可选谓词守卫（predicate guard），`@!p` 为取反
- `opcode` — 操作码
- `.modifier` — 修饰符（类型、舍入、饱和等）
- `d` — 目标操作数；`a, b, c` — 源操作数

---

## 9.3 谓词执行（Predicated Execution）

所有指令都支持可选的谓词守卫：
```ptx
setp.lt.s32  p, a, b;   // p = (a < b)
@p  add.s32  d, a, b;   // 若 p 为真才执行
@!p sub.s32  d, a, b;   // 若 p 为假才执行
```

### 比较操作符

**整数/位大小比较**：`eq, ne, lt, le, gt, ge, lo, ls, hi, hs`（无符号用 lo/ls/hi/hs）

**浮点比较**（含 NaN 处理）：  
- 有序版本（NaN → false）：`eq, ne, lt, le, gt, ge`  
- 无序版本（NaN → true）：`equ, neu, ltu, leu, gtu, geu`  
- 特殊：`num`（两者都不是 NaN），`nan`（至少一个是 NaN）

---

## 9.4 指令的类型信息

- 每条指令有一个**指令类型**（instruction type）决定操作精度
- 操作数可以比指令类型更宽，高位被忽略（`chop`）或扩展
- 类型兼容规则：位大小类型与相同大小的任何类型兼容

---

## 9.5 控制流中的线程分歧（Divergence）

Warp 内线程走不同分支时发生**分歧**：
- 硬件对两个分支路径串行执行，非活跃线程被屏蔽
- 重聚（reconvergence）在后续同步点自动完成
- Volta+ 架构（Independent Thread Scheduling）允许每线程独立 PC

---

## 9.6 语义约定

- 操作以目标类型的精度执行
- 整数溢出：默认回绕（wrapping），`.sat` 修饰符启用饱和（clamp）
- 浮点遵循 IEEE 754，`.ftz` 将次正规数刷零，`.approx` 允许低精度近似

---

## 9.7 指令分类

### 9.7.1 整数算术指令

| 指令 | 说明 |
|---|---|
| `add{.sat}` | 加法，可选饱和；支持 `.u8x4/.s8x4/.u16x2/.s16x2` 打包类型 |
| `sub{.sat}` | 减法 |
| `mul{.hi/.lo/.wide}` | 乘法；`.wide` 产生双倍宽度结果 |
| `mad{.hi/.lo/.wide}` | 乘加（multiply-add） |
| `mad24` / `mul24` | 24 位整数乘法（高性能） |
| `sad` | 绝对差之和（sum of absolute differences） |
| `div` / `rem` | 除法 / 取余 |
| `abs` / `neg` | 绝对值 / 取反 |
| `min` / `max` | 最小值 / 最大值 |
| `popc` | 统计 1 的个数（popcount） |
| `clz` | 统计前导零个数 |
| `bfind` | 查找最高有效位位置 |
| `brev` | 位反转 |
| `bfe` | 位域提取（bit field extract） |
| `bfi` | 位域插入（bit field insert） |
| `szext` | 有符号/零扩展 |
| `bmsk` | 位掩码生成 |
| `dp4a` | 4 元素点积累加（INT8 × INT8 → INT32） |
| `dp2a` | 2 元素点积累加 |

### 9.7.2 扩展精度整数算术指令

用于实现 128 位及更宽的整数算术（通过进位链）：

| 指令 | 说明 |
|---|---|
| `add.cc` | 加法并设置进位标志（CC.CF） |
| `addc{.cc}` | 带进位加法 |
| `sub.cc` | 减法并设置借位标志 |
| `subc{.cc}` | 带借位减法 |
| `mad.cc` / `madc` | 扩展精度乘加 |

```ptx
// 64位 + 64位 → 64位（通过两个32位寄存器对）
add.cc.u32   rl, al, bl;
addc.u32     rh, ah, bh;
```

### 9.7.3 单精度/双精度浮点指令

| 指令 | 说明 |
|---|---|
| `testp` | 测试浮点属性（infinite/finite/number/notanumber/normal/subnormal） |
| `copysign` | 复制符号位 |
| `add / sub / mul` | 基本算术，支持 `.rn/.rz/.rm/.rp` 舍入 |
| `fma` | 融合乘加（fused multiply-add），单次舍入，精度更高 |
| `mad` | 乘加（非融合，两次舍入，已基本被 fma 取代） |
| `div` | 除法（`.full` 全精度，`.approx` 近似） |
| `abs / neg / min / max` | 绝对值/取反/最小/最大 |
| `rcp` | 倒数（`.approx` 近似，`.rn/.rz` 全精度） |
| `sqrt` | 平方根 |
| `rsqrt` | 平方根倒数（`.approx`） |
| `sin / cos` | 三角函数（近似，`.approx.ftz.f32` 仅） |
| `lg2` | 以 2 为底的对数（近似） |
| `ex2` | 2 的指数（近似） |
| `tanh` | 双曲正切 |

### 9.7.4 半精度浮点指令（f16/bf16）

对 `.f16` 和 `.bf16` 类型的操作，以及打包类型 `.f16x2` 的 SIMD 操作：

`add, sub, mul, fma, neg, abs, min, max, tanh, ex2`

```ptx
add.f16x2   d, a, b;   // 同时对两个 f16 做加法
fma.rn.f16  d, a, b, c;
```

### 9.7.5 比较与选择指令

| 指令 | 说明 |
|---|---|
| `set` | 比较两个值，结果写入整数或浮点寄存器（0 或 1/-1） |
| `setp` | 比较两个值，结果写入谓词寄存器 |
| `selp` | 谓词选择：`d = p ? a : b` |
| `slct` | 符号选择：根据第三个操作数的符号选择 |

### 9.7.6 半精度比较指令

`set.f16x2` 和 `setp.f16x2` — 对打包 f16x2 进行 SIMD 比较。

### 9.7.7 逻辑与移位指令

| 指令 | 说明 |
|---|---|
| `and / or / xor / not` | 位逻辑操作 |
| `cnot` | 逻辑非（0 → 1，非0 → 0） |
| `lop3` | 三操作数任意逻辑运算（由立即数 `immLut` 指定真值表，256 种组合） |
| `shf` | 漏斗移位（funnel shift）：从两个 32 位寄存器拼接后移位 |
| `shl` / `shr` | 左移 / 右移（算术或逻辑） |

```ptx
lop3.b32  d, a, b, c, 0x96;  // d = a XOR b XOR c
```

### 9.7.8–9.7.9 数据移动与转换指令

**寄存器/立即数移动：**

| 指令 | 说明 |
|---|---|
| `mov` | 寄存器间复制，或将符号/地址/立即数载入寄存器 |
| `shfl.sync` | Warp 内 lane 间数据交换（butterfly/broadcast/up/down/idx 模式） |
| `prmt` | 字节排列（permute bytes，任意 4 字节重组） |

**内存 Load/Store：**

| 指令 | 说明 |
|---|---|
| `ld{.space}{.qualifier}` | 从内存加载（支持 .weak/.relaxed/.acquire/.volatile/.mmio 等） |
| `ld.global.nc` | 通过只读数据缓存（non-coherent）加载全局内存 |
| `ldu` | 统一加载（uniform load，所有 lane 访问相同地址） |
| `st{.space}{.qualifier}` | 向内存存储 |
| `st.async` | 异步存储（结合 mbarrier） |
| `st.bulk` | 批量存储（.bulk，面向大块数据） |

**原子与 Multimem：**

| 指令 | 说明 |
|---|---|
| `multimem.ld_reduce` | 跨设备多内存 reduce 读取 |
| `multimem.st` | 广播存储到多设备 |
| `multimem.red` | 跨设备 reduce |

**缓存控制：**

| 指令 | 说明 |
|---|---|
| `prefetch` / `prefetchu` | 预取（L1/L2 层次可选） |
| `applypriority` | 设置缓存行优先级（evict-first/evict-normal/evict-last） |
| `discard` | 废弃缓存行（无需写回） |
| `createpolicy` | 创建缓存访问策略对象 |

**地址空间操作：**

| 指令 | 说明 |
|---|---|
| `isspacep` | 测试指针是否属于特定状态空间（.global/.shared/.local/.const） |
| `cvta` | 地址转换（通用地址 ↔ 状态空间地址） |
| `mapa` | 将 shared 变量地址映射到同 cluster 内另一个 CTA 的地址 |
| `getctarank` | 获取 cluster 内 CTA 的 rank |

**类型转换：**

| 指令 | 说明 |
|---|---|
| `cvt` | 标量类型转换（整数↔浮点↔各种 FP 格式，支持舍入和饱和） |
| `cvt.pack` | 打包转换（将两个宽类型值打包为一个窄打包类型） |

**异步拷贝（TMA/cp.async）：**

| 指令 | 说明 |
|---|---|
| `cp.async` | 异步全局→共享内存拷贝（绕过寄存器，sm_80+） |
| `cp.async.commit_group` | 提交当前 cp.async 组 |
| `cp.async.wait_group` | 等待指定数量的 cp.async 组完成 |
| `cp.async.bulk` | 大块异步拷贝（TMA，sm_90+；支持 global→shared/shared→global/shared→shared） |
| `cp.reduce.async.bulk` | 异步 reduce 并写入目标（TMA reduce） |
| `cp.async.bulk.tensor` | TMA 多维张量拷贝（Tiled/Im2col 模式） |

---

### 9.7.10 纹理指令（Texture Instructions）

通过 `.texref` / `.samplerref` 访问纹理内存，支持 1D/2D/3D/Cube/Array：

| 指令 | 说明 |
|---|---|
| `tex` | 纹理读取（自动滤波/插值/边界处理） |
| `tld4` | 双线性滤波的 4 个相邻纹素采样 |
| `txq` | 纹理属性查询（宽度/高度/通道数等） |
| `istypep` | 测试 opaque 类型对象是否为指定类型 |

### 9.7.11 表面指令（Surface Instructions）

通过 `.surfref` 进行读写访问：

| 指令 | 说明 |
|---|---|
| `suld` | 表面加载（surface load） |
| `sust` | 表面存储（surface store） |
| `sured` | 表面原子操作（surface reduction） |
| `suq` | 表面属性查询 |

### 9.7.12 控制流指令

| 指令 | 说明 |
|---|---|
| `bra{.uni}` | 无条件/条件跳转；`.uni` 提示 warp 中所有线程取相同分支 |
| `brx.idx` | 跳转表（indexed branch，通过寄存器选择跳转目标） |
| `call{.uni}` | 函数调用（直接或间接） |
| `ret{.uni}` | 函数返回 |
| `exit` | 线程退出（终止当前线程） |

### 9.7.13 并行同步与通信指令

**CTA 内 Barrier：**

| 指令 | 说明 |
|---|---|
| `bar{.cta}.sync` | CTA 内 barrier，所有线程到达后继续（等价于 `__syncthreads()`） |
| `bar{.cta}.red` | Barrier + 对谓词做 reduction（and/or/popc） |
| `bar{.cta}.arrive` | 到达 barrier 但不等待（用于分步同步） |
| `bar.warp.sync` | Warp 级 barrier |

**Cluster Barrier：**

| 指令 | 说明 |
|---|---|
| `barrier.cluster.arrive{.release}` | 到达 cluster barrier |
| `barrier.cluster.wait{.acquire}` | 等待 cluster barrier |

**内存栅栏：**

| 指令 | 说明 |
|---|---|
| `membar.{cta/gl/sys}` | 内存序 barrier（旧式，推荐用 fence） |
| `fence.{sc/acq_rel/acquire/release}.{scope}` | 细粒度内存栅栏 |
| `fence.proxy.*` | 跨 proxy（generic/tensormap/alias）的 proxy fence |

**原子操作：**

| 指令 | 说明 |
|---|---|
| `atom` | 原子 RMW（add/min/max/inc/dec/and/or/xor/exch/cas），支持整数和浮点 |
| `red` | 原子 Reduction（只写，无返回值，吞吐更高） |

**Warp 通信：**

| 指令 | 说明 |
|---|---|
| `vote.sync` | Warp 内谓词投票（all/any/uni/ballot） |
| `match.sync` | Warp 内值匹配（返回哪些 lane 有相同值的掩码） |
| `activemask` | 返回当前活跃 lane 的掩码 |
| `redux.sync` | Warp 内 reduction（add/min/max/and/or/xor） |
| `elect.sync` | 从活跃线程中选出一个 |

**Grid 与 Cluster 控制：**

| 指令 | 说明 |
|---|---|
| `griddepcontrol` | 控制从 GPU 端发起子 kernel 的依赖（CUDA Dynamic Parallelism） |
| `clusterlaunchcontrol.try_cancel` | 尝试取消 cluster 启动 |
| `clusterlaunchcontrol.query_cancel` | 查询 cluster 取消状态 |

**mbarrier（异步 Barrier）：**

核心操作（sm_80+）：

| 指令 | 说明 |
|---|---|
| `mbarrier.init` | 初始化 mbarrier，设置参与线程数 |
| `mbarrier.arrive{.release}` | 到达并减少期望计数 |
| `mbarrier.arrive_drop` | 到达并减少期望计数（同时减少参与者数量） |
| `mbarrier.complete_tx` | 通知异步 TX 字节数完成 |
| `mbarrier.test_wait{.acquire}` | 非阻塞查询是否完成 |
| `mbarrier.try_wait{.acquire}` | 尝试等待（带超时提示） |
| `mbarrier.pending_count` | 查询剩余未完成计数 |

```ptx
// 典型 cp.async + mbarrier 使用模式
mbarrier.init.shared.b64    [mb], N;
cp.async.bulk.tensor.2d.shared::cluster.global.mbarrier::complete_tx::bytes
                             [smem], [gmem], {H, W}, [mb];
mbarrier.try_wait.shared.b64 complete, [mb], phase;
```

**TensorMap Proxy Fence：**

| 指令 | 说明 |
|---|---|
| `tensormap.cp_fenceproxy` | 在 tensor map 写操作后设置 proxy fence |
| `tensormap.replace` | 在设备端动态修改 tensor map 属性 |

---

### 9.7.14 Warp 级矩阵乘累加（wmma / mma）

**wmma 指令**（sm_70+，以 warp 为单位的 MMA）：

```ptx
wmma.load.a.sync.aligned.row.m16n16k16.f16  {ra0,...,ra7}, [ptr];
wmma.load.b.sync.aligned.col.m16n16k16.f16  {rb0,...,rb7}, [ptr];
wmma.load.c.sync.aligned.row.m16n16k16.f32  {rc0,...,rc7}, [ptr];
wmma.mma.sync.aligned.row.col.m16n16k16.f32.f32  {rd0,...,rd7}, {ra}, {rb}, {rc};
wmma.store.d.sync.aligned.row.m16n16k16.f32  [ptr], {rd0,...,rd7};
```

**支持形状**（M×N×K）：
- `.f16/.bf16`：m16n16k16, m32n8k16, m8n32k16
- `.tf32`：m16n16k8
- `.f64`：m8n8k4
- `.s8/.u8`：m16n16k16, m32n8k16, m8n32k16
- `.s4/.u4`：m8n8k32 等

**mma 指令**（sm_80+，更精细控制）：

```ptx
mma.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32
    {rd0,...,rd3}, {ra0,ra1,ra2,ra3}, {rb0,rb1}, {rc0,...,rc3};
```

**mma.sp 稀疏矩阵**（sm_80+，50% 结构化稀疏）：

```ptx
mma.sp.sync.aligned.m16n8k16.row.col.f32.f16.f16.f32
    {d}, {a}, {b}, {c}, meta, 0x0;
```

---

### 9.7.15 异步 Warp Group 级 MMA（wgmma）

sm_90+ 专属，4 个 warp（128 线程）协同完成 MMA：

```ptx
wgmma.mma_async.sync.aligned.m64n128k16.f32.f16.f16
    {d0,...,d63}, descA, descB, 1, 1, 1;
wgmma.wait_group.sync.aligned  0;
wgmma.fence.sync.aligned;
wgmma.commit_group.sync.aligned;
```

**固定 M=64，N=8~256（步长 8），K 依数据类型**：
- `.f16/.bf16`：K=16
- `.tf32`：K=8
- `.f8`（e4m3/e5m2）：K=32
- `.f4`（e2m1）：K=64

矩阵 B **必须在 smem**（通过 64 位 smem descriptor 传递）；矩阵 A 可在寄存器或 smem；累加器在寄存器（每线程 32 个 f32）。

**wgmma.sp** — 稀疏 A 矩阵的 wgmma（sm_90+）。

---

### 9.7.16 Tensor 内存指令（tcgen05，sm_100+/Blackwell）

tcgen05 是 Blackwell 新增的 Tensor Memory（TC Memory）访问指令组：

| 指令 | 说明 |
|---|---|
| `tcgen05.alloc` | 分配 Tensor Memory（TC Mem）块 |
| `tcgen05.dealloc` | 释放 TC Mem 块 |
| `tcgen05.ld` | 从 TC Mem 加载（支持多种形状，M=16/32/64/128/256 等） |
| `tcgen05.st` | 向 TC Mem 存储 |
| `tcgen05.cp` | 从 smem 拷贝到 TC Mem |
| `tcgen05.reduce` | TC Mem 内 reduce 操作 |
| `tcgen05.shift` | TC Mem 行移位 |
| `tcgen05.fence` | TC Mem 操作的 fence |
| `tcgen05.commit_group` | 提交 TC Mem 操作组 |
| `tcgen05.wait_group` | 等待 TC Mem 操作组完成 |
| `tcgen05.mma` | 使用 TC Mem 的 MMA（类似 wgmma 但使用 TC Mem 作为累加器） |

---

### 9.7.17 栈操作指令（Stack Manipulation）

| 指令 | 说明 |
|---|---|
| `alloca` | 在线程本地栈上运行时分配内存，返回 `.local` 空间指针 |
| `stacksave` | 将栈指针保存到寄存器 |
| `stackrestore` | 从寄存器恢复栈指针 |

---

### 9.7.18 其他杂项指令

| 指令 | 说明 |
|---|---|
| `trap` | 生成运行时错误（触发 GPU 异常） |
| `brkpt` | 调试断点（需调试器支持） |
| `setmaxnreg` | 动态调整当前 warpgroup 使用的寄存器数量上限（sm_90+） |
| `nanosleep` | 线程休眠指定纳秒数（sm_70+） |
| `grp.arrive` | Warpgroup Arrive 操作 |

---

## 附：指令快查表（按功能分组）

| 功能 | 关键指令 |
|---|---|
| 整数四则运算 | `add`, `sub`, `mul`, `mad`, `div`, `rem` |
| 整数位操作 | `and`, `or`, `xor`, `not`, `shl`, `shr`, `bfe`, `bfi`, `lop3` |
| 浮点运算 | `fma`, `rcp`, `sqrt`, `rsqrt`, `sin`, `cos`, `ex2`, `lg2` |
| 类型转换 | `cvt`, `cvt.pack` |
| 内存访问 | `ld`, `st`, `atom`, `red` |
| 异步拷贝 | `cp.async`, `cp.async.bulk`, `cp.async.bulk.tensor` |
| Warp 通信 | `shfl.sync`, `vote.sync`, `match.sync`, `redux.sync` |
| CTA 同步 | `bar.sync`, `mbarrier.*`, `fence.*` |
| Cluster 同步 | `barrier.cluster.*`, `fence.proxy.*` |
| 矩阵乘法（Warp） | `wmma.*`, `mma.sync` |
| 矩阵乘法（Warpgroup） | `wgmma.mma_async`, `wgmma.wait_group` |
| Tensor Memory | `tcgen05.*` |
| 控制流 | `bra`, `call`, `ret`, `exit`, `setp`, `selp` |
| 特殊目的 | `alloca`, `setmaxnreg`, `nanosleep`, `trap` |
