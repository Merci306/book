# PTX ISA 9.2 — 第十章：特殊寄存器（中文翻译）

> 原文来源：NVIDIA PTX ISA Release 9.2, Chapter 10: Special Registers（第 775–806 页）

PTX 包含一组预定义的只读变量，以**特殊寄存器**的形式呈现，可通过 `mov` 或 `cvt` 指令访问。

---

## 目录

1. [%tid — 线程标识符](#1-tid--线程标识符)
2. [%ntid — 每个 CTA 的线程数](#2-ntid--每个-cta-的线程数)
3. [%laneid — Lane 标识符](#3-laneid--lane-标识符)
4. [%warpid — Warp 标识符](#4-warpid--warp-标识符)
5. [%nwarpid — Warp 标识符总数](#5-nwarpid--warp-标识符总数)
6. [%ctaid — Grid 中的 CTA 标识符](#6-ctaid--grid-中的-cta-标识符)
7. [%nctaid — 每个 Grid 的 CTA 数量](#7-nctaid--每个-grid-的-cta-数量)
8. [%smid — SM 标识符](#8-smid--sm-标识符)
9. [%nsmid — SM 标识符总数](#9-nsmid--sm-标识符总数)
10. [%gridid — Grid 标识符](#10-gridid--grid-标识符)
11. [%is_explicit_cluster — 是否显式指定了 Cluster 启动](#11-is_explicit_cluster--是否显式指定了-cluster-启动)
12. [%clusterid — Grid 中的 Cluster 标识符](#12-clusterid--grid-中的-cluster-标识符)
13. [%nclusterid — 每个 Grid 的 Cluster 数量](#13-nclusterid--每个-grid-的-cluster-数量)
14. [%cluster_ctaid — Cluster 中的 CTA 标识符](#14-cluster_ctaid--cluster-中的-cta-标识符)
15. [%cluster_nctaid — 每个 Cluster 的 CTA 数量](#15-cluster_nctaid--每个-cluster-的-cta-数量)
16. [%cluster_ctarank — Cluster 中跨维度的 CTA 秩](#16-cluster_ctarank--cluster-中跨维度的-cta-秩)
17. [%cluster_nctarank — Cluster 中 CTA 总数（跨维度）](#17-cluster_nctarank--cluster-中-cta-总数跨维度)
18. [%lanemask_eq — Lane 掩码（等于）](#18-lanemask_eq--lane-掩码等于)
19. [%lanemask_le — Lane 掩码（小于或等于）](#19-lanemask_le--lane-掩码小于或等于)
20. [%lanemask_lt — Lane 掩码（小于）](#20-lanemask_lt--lane-掩码小于)
21. [%lanemask_ge — Lane 掩码（大于或等于）](#21-lanemask_ge--lane-掩码大于或等于)
22. [%lanemask_gt — Lane 掩码（大于）](#22-lanemask_gt--lane-掩码大于)
23. [%clock / %clock_hi — 32 位周期计数器](#23-clock--clock_hi--32-位周期计数器)
24. [%clock64 — 64 位周期计数器](#24-clock64--64-位周期计数器)
25. [%pm0–%pm7 — 性能监控计数器（32 位）](#25-pm0pm7--性能监控计数器32-位)
26. [%pm0_64–%pm7_64 — 性能监控计数器（64 位）](#26-pm0_64pm7_64--性能监控计数器64-位)
27. [%envreg\<32\> — 驱动程序定义的只读寄存器](#27-envreg32--驱动程序定义的只读寄存器)
28. [%globaltimer / %globaltimer_lo / %globaltimer_hi — 全局纳秒计时器](#28-globaltimer--globaltimer_lo--globaltimer_hi--全局纳秒计时器)
29. [%reserved_smem_offset_* — 保留共享内存偏移](#29-reserved_smem_offset_---保留共享内存偏移)
30. [%total_smem_size — CTA 共享内存总大小](#30-total_smem_size--cta-共享内存总大小)
31. [%aggr_smem_size — CTA 共享内存聚合大小](#31-aggr_smem_size--cta-共享内存聚合大小)
32. [%dynamic_smem_size — 动态分配的共享内存大小](#32-dynamic_smem_size--动态分配的共享内存大小)
33. [%current_graph_exec — 当前执行的 CUDA Device Graph 标识符](#33-current_graph_exec--当前执行的-cuda-device-graph-标识符)

---

## 1. `%tid` — 线程标识符

### 语法

```ptx
.sreg .v4 .u32 %tid;          // 线程 ID 向量
.sreg .u32 %tid.x, %tid.y, %tid.z;  // 线程 ID 各分量
```

### 描述

预定义的只读逐线程特殊寄存器，以 CTA 内的线程标识符初始化。`%tid` 是一个 1D、2D 或 3D 向量，与 CTA 的形状匹配；未使用维度上的值为 `0`，第四个元素始终返回零。每个 CTA 中每个线程的 `%tid` 唯一。

各维度范围保证满足：

```
0 <= %tid.x < %ntid.x
0 <= %tid.y < %ntid.y
0 <= %tid.z < %ntid.z
```

- 1D CTA 中：`%tid.y == %tid.z == 0`
- 2D CTA 中：`%tid.z == 0`

### PTX ISA 版本

- PTX ISA 1.0 引入，类型为 `.v4.u16`
- PTX ISA 2.0 重新定义为 `.v4.u32`
- 为兼容旧版 PTX 代码，可使用 16 位 `mov`/`cvt` 读取 `%tid` 各分量的低 16 位

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u32  %r1, %tid.x;         // 将 tid.x 移入 %r1

// 旧版代码：访问 16 位分量
mov.u16  %rh, %tid.x;
cvt.u32.u16  %r2, %tid.z;     // 将 tid.z 零扩展到 %r2
```

---

## 2. `%ntid` — 每个 CTA 的线程数

### 语法

```ptx
.sreg .v4 .u32 %ntid;               // CTA 形状向量
.sreg .u32 %ntid.x, %ntid.y, %ntid.z;  // CTA 各维度大小
```

### 描述

预定义的只读特殊寄存器，以每个 CTA 维度的线程数初始化。`%ntid` 是一个三维 CTA 形状向量，所有维度值均非零，第四个元素始终返回零。CTA 的总线程数为 `%ntid.x * %ntid.y * %ntid.z`。

- 1D CTA 中：`%ntid.y == %ntid.z == 1`
- 2D CTA 中：`%ntid.z == 1`

各目标架构的 `%ntid.{x,y,z}` 最大值：

| 目标架构 | %ntid.x | %ntid.y | %ntid.z |
|---|---|---|---|
| sm_1x | 512 | 512 | 64 |
| sm_20、sm_3x、sm_5x、sm_6x、sm_7x、sm_8x、sm_9x、sm_10x、sm_12x | 1024 | 1024 | 64 |

### PTX ISA 版本

- PTX ISA 1.0 引入，类型为 `.v4.u16`
- PTX ISA 2.0 重新定义为 `.v4.u32`
- 为兼容旧版 PTX 代码，可使用 16 位 `mov`/`cvt` 读取各分量的低 16 位

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
// 计算 2D CTA 中的统一线程 ID
mov.u32  %r0, %tid.x;
mov.u32  %h1, %tid.y;
mov.u32  %h2, %ntid.x;
mad.u32  %r0, %h1, %h2, %r0;

mov.u16  %rh, %ntid.x;        // 旧版代码
```

---

## 3. `%laneid` — Lane 标识符

### 语法

```ptx
.sreg .u32 %laneid;
```

### 描述

预定义的只读特殊寄存器，返回线程在 warp 中的 lane 编号。lane 标识符范围为 `0` 到 `WARP_SZ-1`。

### PTX ISA 版本

PTX ISA 1.3 引入。

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u32  %r, %laneid;
```

---

## 4. `%warpid` — Warp 标识符

### 语法

```ptx
.sreg .u32 %warpid;
```

### 描述

预定义的只读特殊寄存器，返回线程所在 warp 的标识符。`%warpid` 在 CTA 内唯一，但跨 CTA 不保证唯一。同一 warp 内的所有线程具有相同的 `%warpid` 值。

**注意**：`%warpid` 返回的是读取时刻的线程位置，在执行过程中（如因抢占导致线程重新调度后）其值可能改变。因此，若需要在 kernel 代码中计算虚拟 warp 索引，应使用 `%ctaid` 和 `%tid`。`%warpid` 主要用于性能分析和诊断代码（例如采样工作负载映射与负载分布）。

### PTX ISA 版本

PTX ISA 1.3 引入。

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u32  %r, %warpid;
```

---

## 5. `%nwarpid` — Warp 标识符总数

### 语法

```ptx
.sreg .u32 %nwarpid;
```

### 描述

预定义的只读特殊寄存器，返回最大 warp 标识符数量。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %nwarpid;
```

---

## 6. `%ctaid` — Grid 中的 CTA 标识符

### 语法

```ptx
.sreg .v4 .u32 %ctaid;                      // CTA ID 向量
.sreg .u32 %ctaid.x, %ctaid.y, %ctaid.z;    // CTA ID 各分量
```

### 描述

预定义的只读特殊寄存器，以 CTA Grid 中的 CTA 标识符初始化。`%ctaid` 是一个 1D、2D 或 3D 向量，取决于 CTA Grid 的形状和维度，第四个元素始终返回零。

各维度范围保证满足：

```
0 <= %ctaid.x < %nctaid.x
0 <= %ctaid.y < %nctaid.y
0 <= %ctaid.z < %nctaid.z
```

### PTX ISA 版本

- PTX ISA 1.0 引入，类型为 `.v4.u16`
- PTX ISA 2.0 重新定义为 `.v4.u32`
- 为兼容旧版 PTX 代码，可使用 16 位 `mov`/`cvt` 读取各分量的低 16 位

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u32  %r0, %ctaid.x;
mov.u16  %rh, %ctaid.y;   // 旧版代码
```

---

## 7. `%nctaid` — 每个 Grid 的 CTA 数量

### 语法

```ptx
.sreg .v4 .u32 %nctaid;                       // Grid 形状向量
.sreg .u32 %nctaid.x, %nctaid.y, %nctaid.z;  // Grid 各维度大小
```

### 描述

预定义的只读特殊寄存器，以每个 Grid 维度的 CTA 数量初始化。`%nctaid` 是三维 Grid 形状向量，每个元素值至少为 `1`，第四个元素始终返回零。

各目标架构的 `%nctaid.{x,y,z}` 最大值：

| 目标架构 | %nctaid.x | %nctaid.y | %nctaid.z |
|---|---|---|---|
| sm_1x、sm_20 | 65535 | 65535 | 65535 |
| sm_3x、sm_5x、sm_6x、sm_7x、sm_8x、sm_9x、sm_10x、sm_12x | 2³¹−1 | 65535 | 65535 |

### PTX ISA 版本

- PTX ISA 1.0 引入，类型为 `.v4.u16`
- PTX ISA 2.0 重新定义为 `.v4.u32`
- 为兼容旧版 PTX 代码，可使用 16 位 `mov`/`cvt` 读取各分量的低 16 位

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u32  %r0, %nctaid.x;
mov.u16  %rh, %nctaid.x;  // 旧版代码
```

---

## 8. `%smid` — SM 标识符

### 语法

```ptx
.sreg .u32 %smid;
```

### 描述

预定义的只读特殊寄存器，返回当前线程正在执行的处理器（SM）标识符。SM 标识符范围为 `0` 到 `%nsmid-1`，但不保证连续。

**注意**：`%smid` 返回的是读取时刻的线程位置，在执行过程中（如因抢占导致线程重新调度后）其值可能改变。`%smid` 主要用于性能分析和诊断代码（例如采样工作负载映射与负载分布）。

### PTX ISA 版本

PTX ISA 1.3 引入。

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u32  %r, %smid;
```

---

## 9. `%nsmid` — SM 标识符总数

### 语法

```ptx
.sreg .u32 %nsmid;
```

### 描述

预定义的只读特殊寄存器，返回最大 SM 标识符数量。由于 SM 标识符编号不保证连续，`%nsmid` 可能大于设备上物理 SM 的实际数量。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %nsmid;
```

---

## 10. `%gridid` — Grid 标识符

### 语法

```ptx
.sreg .u64 %gridid;
```

### 描述

预定义的只读特殊寄存器，以每次 Grid 启动的时序标识符初始化。调试器使用 `%gridid` 区分并发（小型）Grid 中的各个 CTA 和 Cluster。在执行过程中，程序可能被反复启动，每次启动都会开始一个 CTA Grid；该变量提供此上下文中 Grid 启动的时序编号。

- sm_1x 目标：`%gridid` 限于范围 `[0..2¹⁶-1]`
- sm_20 目标：`%gridid` 限于范围 `[0..2³²-1]`
- sm_30 及以上：支持完整 64 位范围

### PTX ISA 版本

- PTX ISA 1.0 引入，类型为 `.u16`
- PTX ISA 1.3 重新定义为 `.u32`
- PTX ISA 3.0 重新定义为 `.u64`
- 为兼容旧版 PTX 代码，可使用 16 位或 32 位 `mov`/`cvt` 读取低位

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.u64  %s, %gridid;     // 64 位读取 %gridid
mov.u32  %r, %gridid;     // 旧版代码：32 位 %gridid
```

---

## 11. `%is_explicit_cluster` — 是否显式指定了 Cluster 启动

### 语法

```ptx
.sreg .pred %is_explicit_cluster;
```

### 描述

预定义的只读特殊寄存器，以谓词值初始化，指示用户是否显式指定了 Cluster 启动。

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .pred p;
mov.pred  p, %is_explicit_cluster;
```

---

## 12. `%clusterid` — Grid 中的 Cluster 标识符

### 语法

```ptx
.sreg .v4 .u32 %clusterid;
.sreg .u32 %clusterid.x, %clusterid.y, %clusterid.z;
```

### 描述

预定义的只读特殊寄存器，以 Grid 中各维度的 Cluster 标识符初始化。每个 Cluster 在 Grid 中具有唯一的标识符。`%clusterid` 是 1D、2D 或 3D 向量，取决于 Cluster 的形状和维度，第四个元素始终返回零。

各维度范围保证满足：

```
0 <= %clusterid.x < %nclusterid.x
0 <= %clusterid.y < %nclusterid.y
0 <= %clusterid.z < %nclusterid.z
```

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .b32 %r<2>;
.reg .v4 .b32 %rx;
mov.u32   %r0, %clusterid.x;
mov.u32   %r1, %clusterid.z;
mov.v4.u32  %rx, %clusterid;
```

---

## 13. `%nclusterid` — 每个 Grid 的 Cluster 数量

### 语法

```ptx
.sreg .v4 .u32 %nclusterid;
.sreg .u32 %nclusterid.x, %nclusterid.y, %nclusterid.z;
```

### 描述

预定义的只读特殊寄存器，以每个 Grid 维度中的 Cluster 数量初始化。`%nclusterid` 是三维 Grid 形状向量，以 Cluster 为单位保存 Grid 维度信息，第四个元素始终返回零。`%nclusterid.{x,y,z}` 的最大值请参阅 CUDA 编程指南。

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .b32 %r<2>;
.reg .v4 .b32 %rx;
mov.u32   %r0, %nclusterid.x;
mov.u32   %r1, %nclusterid.z;
mov.v4.u32  %rx, %nclusterid;
```

---

## 14. `%cluster_ctaid` — Cluster 中的 CTA 标识符

### 语法

```ptx
.sreg .v4 .u32 %cluster_ctaid;
.sreg .u32 %cluster_ctaid.x, %cluster_ctaid.y, %cluster_ctaid.z;
```

### 描述

预定义的只读特殊寄存器，以 Cluster 中各维度的 CTA 标识符初始化。每个 CTA 在 Cluster 内具有唯一的 CTA 标识符。`%cluster_ctaid` 是 1D、2D 或 3D 向量，取决于 Cluster 的形状，第四个元素始终返回零。

各维度范围保证满足：

```
0 <= %cluster_ctaid.x < %cluster_nctaid.x
0 <= %cluster_ctaid.y < %cluster_nctaid.y
0 <= %cluster_ctaid.z < %cluster_nctaid.z
```

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .b32 %r<2>;
.reg .v4 .b32 %rx;
mov.u32   %r0, %cluster_ctaid.x;
mov.u32   %r1, %cluster_ctaid.z;
mov.v4.u32  %rx, %cluster_ctaid;
```

---

## 15. `%cluster_nctaid` — 每个 Cluster 的 CTA 数量

### 语法

```ptx
.sreg .v4 .u32 %cluster_nctaid;
.sreg .u32 %cluster_nctaid.x, %cluster_nctaid.y, %cluster_nctaid.z;
```

### 描述

预定义的只读特殊寄存器，以 Cluster 中各维度的 CTA 数量初始化。`%cluster_nctaid` 是三维 Grid 形状向量，以 CTA 为单位保存 Cluster 维度信息，第四个元素始终返回零。`%cluster_nctaid.{x,y,z}` 的最大值请参阅 CUDA 编程指南。

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .b32 %r<2>;
.reg .v4 .b32 %rx;
mov.u32   %r0, %cluster_nctaid.x;
mov.u32   %r1, %cluster_nctaid.z;
mov.v4.u32  %rx, %cluster_nctaid;
```

---

## 16. `%cluster_ctarank` — Cluster 中跨维度的 CTA 秩

### 语法

```ptx
.sreg .u32 %cluster_ctarank;
```

### 描述

预定义的只读特殊寄存器，以 Cluster 内跨所有维度的 CTA 秩（rank）初始化。

保证满足：

```
0 <= %cluster_ctarank < %cluster_nctarank
```

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .b32 %r;
mov.u32  %r, %cluster_ctarank;
```

---

## 17. `%cluster_nctarank` — Cluster 中 CTA 总数（跨维度）

### 语法

```ptx
.sreg .u32 %cluster_nctarank;
```

### 描述

预定义的只读特殊寄存器，以 Cluster 内跨所有维度的 CTA 总数初始化。

### PTX ISA 版本

PTX ISA 7.8 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
.reg .b32 %r;
mov.u32  %r, %cluster_nctarank;
```

---

## 18. `%lanemask_eq` — Lane 掩码（等于）

### 语法

```ptx
.sreg .u32 %lanemask_eq;
```

### 描述

预定义的只读特殊寄存器，以 32 位掩码初始化，该掩码中**仅**在等于线程 lane 编号的位置置位。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %lanemask_eq;
```

---

## 19. `%lanemask_le` — Lane 掩码（小于或等于）

### 语法

```ptx
.sreg .u32 %lanemask_le;
```

### 描述

预定义的只读特殊寄存器，以 32 位掩码初始化，该掩码中**小于或等于**线程 lane 编号的所有位置均置位。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %lanemask_le;
```

---

## 20. `%lanemask_lt` — Lane 掩码（小于）

### 语法

```ptx
.sreg .u32 %lanemask_lt;
```

### 描述

预定义的只读特殊寄存器，以 32 位掩码初始化，该掩码中**严格小于**线程 lane 编号的所有位置均置位。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %lanemask_lt;
```

---

## 21. `%lanemask_ge` — Lane 掩码（大于或等于）

### 语法

```ptx
.sreg .u32 %lanemask_ge;
```

### 描述

预定义的只读特殊寄存器，以 32 位掩码初始化，该掩码中**大于或等于**线程 lane 编号的所有位置均置位。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %lanemask_ge;
```

---

## 22. `%lanemask_gt` — Lane 掩码（大于）

### 语法

```ptx
.sreg .u32 %lanemask_gt;
```

### 描述

预定义的只读特殊寄存器，以 32 位掩码初始化，该掩码中**严格大于**线程 lane 编号的所有位置均置位。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %lanemask_gt;
```

---

## 23. `%clock` / `%clock_hi` — 32 位周期计数器

### 语法

```ptx
.sreg .u32 %clock;
.sreg .u32 %clock_hi;
```

### 描述

`%clock` 和 `%clock_hi` 是无符号 32 位只读周期计数器，溢出时静默回绕。

- `%clock`：32 位周期计数器
- `%clock_hi`：`%clock64` 特殊寄存器的高 32 位

### PTX ISA 版本

- `%clock`：PTX ISA 1.0 引入
- `%clock_hi`：PTX ISA 5.0 引入

### 目标架构要求

- `%clock`：所有目标架构均支持
- `%clock_hi`：需要 sm_20 或更高版本

### 示例

```ptx
mov.u32  r1, %clock;
mov.u32  r2, %clock_hi;
```

---

## 24. `%clock64` — 64 位周期计数器

### 语法

```ptx
.sreg .u64 %clock64;
```

### 描述

无符号 64 位只读周期计数器，溢出时静默回绕。

**注意**：`%clock64` 的低 32 位与 `%clock` 相同，高 32 位与 `%clock_hi` 相同。

### PTX ISA 版本

PTX ISA 2.0 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u64  r1, %clock64;
```

---

## 25. `%pm0`–`%pm7` — 性能监控计数器（32 位）

### 语法

```ptx
.sreg .u32 %pm<8>;
```

### 描述

`%pm0`–`%pm7` 是无符号 32 位只读性能监控计数器，其行为目前未定义。

### PTX ISA 版本

- `%pm0`–`%pm3`：PTX ISA 1.3 引入
- `%pm4`–`%pm7`：PTX ISA 3.0 引入

### 目标架构要求

- `%pm0`–`%pm3`：所有目标架构均支持
- `%pm4`–`%pm7`：需要 sm_20 或更高版本

### 示例

```ptx
mov.u32  r1, %pm0;
mov.u32  r1, %pm7;
```

---

## 26. `%pm0_64`–`%pm7_64` — 性能监控计数器（64 位）

### 语法

```ptx
.sreg .u64 %pm0_64;
.sreg .u64 %pm1_64;
.sreg .u64 %pm2_64;
.sreg .u64 %pm3_64;
.sreg .u64 %pm4_64;
.sreg .u64 %pm5_64;
.sreg .u64 %pm6_64;
.sreg .u64 %pm7_64;
```

### 描述

`%pm0_64`–`%pm7_64` 是无符号 64 位只读性能监控计数器，其行为目前未定义。

**注意**：`%pm0_64`–`%pm7_64` 的低 32 位与 `%pm0`–`%pm7` 相同。

### PTX ISA 版本

PTX ISA 4.0 引入。

### 目标架构要求

需要 sm_50 或更高版本。

### 示例

```ptx
mov.u32  r1, %pm0_64;
mov.u32  r1, %pm7_64;
```

---

## 27. `%envreg<32>` — 驱动程序定义的只读寄存器

### 语法

```ptx
.sreg .b32 %envreg<32>;
```

### 描述

一组 32 个预定义只读寄存器，用于捕获 PTX 虚拟机外部的 PTX 程序执行环境。这些寄存器由驱动程序在 kernel 启动前初始化，可包含 CTA 级或 Grid 级的值。这些寄存器的精确语义由驱动程序文档定义。

### PTX ISA 版本

PTX ISA 2.1 引入。

### 目标架构要求

所有目标架构均支持。

### 示例

```ptx
mov.b32  %r1, %envreg0;   // 将 envreg0 移入 %r1
```

---

## 28. `%globaltimer` / `%globaltimer_lo` / `%globaltimer_hi` — 全局纳秒计时器

### 语法

```ptx
.sreg .u64 %globaltimer;
.sreg .u32 %globaltimer_lo, %globaltimer_hi;
```

### 描述

供 NVIDIA 工具使用的特殊寄存器，提供 64 位全局纳秒级计时器。其行为与目标架构相关，在未来 GPU 上可能发生变化或被移除。JIT 编译到其他目标时，这些寄存器的值未定义。

- `%globaltimer`：完整 64 位全局纳秒计时器
- `%globaltimer_lo`：`%globaltimer` 的低 32 位
- `%globaltimer_hi`：`%globaltimer` 的高 32 位

### PTX ISA 版本

PTX ISA 3.1 引入。

### 目标架构要求

需要 sm_30 或更高版本。

### 示例

```ptx
mov.u64  r1, %globaltimer;
```

---

## 29. `%reserved_smem_offset_*` — 保留共享内存偏移

### 语法

```ptx
.sreg .b32 %reserved_smem_offset_begin;
.sreg .b32 %reserved_smem_offset_end;
.sreg .b32 %reserved_smem_offset_cap;
.sreg .b32 %reserved_smem_offset_<2>;
```

### 描述

预定义的只读特殊寄存器，包含 NVIDIA 系统软件保留共享内存区域的相关信息。该共享内存区域用户不可访问，用户代码访问该区域将导致未定义行为。请参阅 CUDA 编程指南了解详细信息。

- `%reserved_smem_offset_begin`：保留共享内存区域的起始偏移
- `%reserved_smem_offset_end`：保留共享内存区域的结束偏移
- `%reserved_smem_offset_cap`：保留共享内存区域的总大小
- `%reserved_smem_offset_<2>`（即 `%reserved_smem_offset_0` 和 `%reserved_smem_offset_1`）：保留共享内存区域内的偏移量

### PTX ISA 版本

PTX ISA 7.6 引入。

### 目标架构要求

需要 sm_80 或更高版本。

### 示例

```ptx
.reg .b32 %reg_begin, %reg_end, %reg_cap, %reg_offset0, %reg_offset1;
mov.b32  %reg_begin,   %reserved_smem_offset_begin;
mov.b32  %reg_end,     %reserved_smem_offset_end;
mov.b32  %reg_cap,     %reserved_smem_offset_cap;
mov.b32  %reg_offset0, %reserved_smem_offset_0;
mov.b32  %reg_offset1, %reserved_smem_offset_1;
```

---

## 30. `%total_smem_size` — CTA 共享内存总大小

### 语法

```ptx
.sreg .u32 %total_smem_size;
```

### 描述

预定义的只读特殊寄存器，以 kernel 启动时为 CTA 分配的共享内存总大小初始化（包含静态和动态分配部分，不含 NVIDIA 系统软件保留区域）。大小以目标架构支持的共享内存分配单元的倍数返回。

各目标架构的分配单元大小：

| 目标架构 | 分配单元大小 |
|---|---|
| sm_2x | 128 字节 |
| sm_3x、sm_5x、sm_6x、sm_7x | 256 字节 |
| sm_8x、sm_9x、sm_10x、sm_12x | 128 字节 |

### PTX ISA 版本

PTX ISA 4.1 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %total_smem_size;
```

---

## 31. `%aggr_smem_size` — CTA 共享内存聚合大小

### 语法

```ptx
.sreg .u32 %aggr_smem_size;
```

### 描述

预定义的只读特殊寄存器，以共享内存的总聚合大小初始化，包括：
- 用户共享内存（静态分配 + 动态分配）在启动时的大小
- NVIDIA 系统软件保留的共享内存区域大小

### PTX ISA 版本

PTX ISA 8.1 引入。

### 目标架构要求

需要 sm_90 或更高版本。

### 示例

```ptx
mov.u32  %r, %aggr_smem_size;
```

---

## 32. `%dynamic_smem_size` — 动态分配的共享内存大小

### 语法

```ptx
.sreg .u32 %dynamic_smem_size;
```

### 描述

预定义的只读特殊寄存器，以 kernel 启动时为 CTA 动态分配的共享内存大小初始化。

### PTX ISA 版本

PTX ISA 4.1 引入。

### 目标架构要求

需要 sm_20 或更高版本。

### 示例

```ptx
mov.u32  %r, %dynamic_smem_size;
```

---

## 33. `%current_graph_exec` — 当前执行的 CUDA Device Graph 标识符

### 语法

```ptx
.sreg .u64 %current_graph_exec;
```

### 描述

预定义的只读特殊寄存器，以当前正在执行的 CUDA Device Graph 的标识符初始化。若当前执行的 kernel 不属于任何 CUDA Device Graph，则该寄存器值为 `0`。请参阅 CUDA 编程指南了解 CUDA Device Graph 的详细信息。

### PTX ISA 版本

PTX ISA 8.0 引入。

### 目标架构要求

需要 sm_50 或更高版本。

### 示例

```ptx
mov.u64  r1, %current_graph_exec;
```

---

## 附录：特殊寄存器速查表

| 寄存器 | 类型 | 描述 | 引入版本 | 最低目标架构 |
|---|---|---|---|---|
| `%tid` | `.v4.u32` | CTA 内线程标识符（x/y/z） | 1.0 | 全部 |
| `%ntid` | `.v4.u32` | 每个 CTA 的线程数（x/y/z） | 1.0 | 全部 |
| `%laneid` | `.u32` | Warp 内的 Lane 编号 | 1.3 | 全部 |
| `%warpid` | `.u32` | CTA 内的 Warp 标识符 | 1.3 | 全部 |
| `%nwarpid` | `.u32` | 最大 Warp 标识符数量 | 2.0 | sm_20 |
| `%ctaid` | `.v4.u32` | Grid 中的 CTA 标识符（x/y/z） | 1.0 | 全部 |
| `%nctaid` | `.v4.u32` | 每个 Grid 的 CTA 数量（x/y/z） | 1.0 | 全部 |
| `%smid` | `.u32` | SM 标识符 | 1.3 | 全部 |
| `%nsmid` | `.u32` | 最大 SM 标识符数量 | 2.0 | sm_20 |
| `%gridid` | `.u64` | Grid 时序标识符 | 1.0 | 全部 |
| `%is_explicit_cluster` | `.pred` | 是否显式指定 Cluster 启动 | 7.8 | sm_90 |
| `%clusterid` | `.v4.u32` | Grid 中的 Cluster 标识符（x/y/z） | 7.8 | sm_90 |
| `%nclusterid` | `.v4.u32` | 每个 Grid 的 Cluster 数量（x/y/z） | 7.8 | sm_90 |
| `%cluster_ctaid` | `.v4.u32` | Cluster 中的 CTA 标识符（x/y/z） | 7.8 | sm_90 |
| `%cluster_nctaid` | `.v4.u32` | 每个 Cluster 的 CTA 数量（x/y/z） | 7.8 | sm_90 |
| `%cluster_ctarank` | `.u32` | Cluster 内 CTA 的线性秩 | 7.8 | sm_90 |
| `%cluster_nctarank` | `.u32` | Cluster 内 CTA 总数（线性） | 7.8 | sm_90 |
| `%lanemask_eq` | `.u32` | Lane 位掩码（等于当前 Lane） | 2.0 | sm_20 |
| `%lanemask_le` | `.u32` | Lane 位掩码（≤ 当前 Lane） | 2.0 | sm_20 |
| `%lanemask_lt` | `.u32` | Lane 位掩码（< 当前 Lane） | 2.0 | sm_20 |
| `%lanemask_ge` | `.u32` | Lane 位掩码（≥ 当前 Lane） | 2.0 | sm_20 |
| `%lanemask_gt` | `.u32` | Lane 位掩码（> 当前 Lane） | 2.0 | sm_20 |
| `%clock` | `.u32` | 32 位周期计数器 | 1.0 | 全部 |
| `%clock_hi` | `.u32` | `%clock64` 的高 32 位 | 5.0 | sm_20 |
| `%clock64` | `.u64` | 64 位周期计数器 | 2.0 | sm_20 |
| `%pm0`–`%pm3` | `.u32` | 32 位性能监控计数器 | 1.3 | 全部 |
| `%pm4`–`%pm7` | `.u32` | 32 位性能监控计数器（扩展） | 3.0 | sm_20 |
| `%pm0_64`–`%pm7_64` | `.u64` | 64 位性能监控计数器 | 4.0 | sm_50 |
| `%envreg<32>` | `.b32` | 驱动程序定义的 32 个只读寄存器 | 2.1 | 全部 |
| `%globaltimer` | `.u64` | 64 位全局纳秒计时器 | 3.1 | sm_30 |
| `%globaltimer_lo` | `.u32` | `%globaltimer` 的低 32 位 | 3.1 | sm_30 |
| `%globaltimer_hi` | `.u32` | `%globaltimer` 的高 32 位 | 3.1 | sm_30 |
| `%reserved_smem_offset_begin` | `.b32` | 系统保留共享内存起始偏移 | 7.6 | sm_80 |
| `%reserved_smem_offset_end` | `.b32` | 系统保留共享内存结束偏移 | 7.6 | sm_80 |
| `%reserved_smem_offset_cap` | `.b32` | 系统保留共享内存总大小 | 7.6 | sm_80 |
| `%reserved_smem_offset_<2>` | `.b32` | 系统保留共享内存内偏移（0/1） | 7.6 | sm_80 |
| `%total_smem_size` | `.u32` | CTA 共享内存总大小（含静态+动态） | 4.1 | sm_20 |
| `%aggr_smem_size` | `.u32` | CTA 共享内存聚合大小（含系统保留） | 8.1 | sm_90 |
| `%dynamic_smem_size` | `.u32` | 动态分配的共享内存大小 | 4.1 | sm_20 |
| `%current_graph_exec` | `.u64` | 当前执行的 CUDA Device Graph 标识符 | 8.0 | sm_50 |
