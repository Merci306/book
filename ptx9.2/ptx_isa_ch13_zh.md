# 第十三章：发布说明（Release Notes）

> 来源：PTX ISA 9.2 — Chapter 13: Release Notes（第 867–901 页）

本章梳理了 PTX ISA 各版本的变更历史。第一节描述当前版本（9.2）的新特性，其余各节按时间倒序记录从 PTX ISA 2.0 起的历次变更。

---

## PTX ISA 版本与 CUDA 发布对应总表

| PTX ISA 版本 | CUDA 版本 / 驱动 | 新增支持的目标架构 |
|:---:|:---:|:---|
| 1.0 | CUDA 1.0 | sm_10, sm_11 |
| 1.1 | CUDA 1.1 | — |
| 1.2 | CUDA 2.0 | sm_12, sm_13 |
| 1.3 | CUDA 2.1 | — |
| 1.4 | CUDA 2.2 | — |
| 1.5 | driver r190 | — |
| **2.0** | CUDA 3.0 / r195 | **sm_20** |
| 2.1 | CUDA 3.1 / r256 | — |
| 2.2 | CUDA 3.2 / r260 | — |
| 2.3 | CUDA 4.0 / r270 | — |
| **3.0** | CUDA 4.1 / r285 | **sm_30** |
| **3.1** | CUDA 5.0 / r302 | **sm_35** |
| 3.2 | CUDA 5.5 / r319 | — |
| **4.0** | CUDA 6.0 / r331 | **sm_32, sm_50** |
| **4.1** | CUDA 6.5 / r340 | **sm_37, sm_52** |
| **4.2** | CUDA 7.0 / r346 | **sm_53** |
| 4.3 | CUDA 7.5 / r352 | — |
| **5.0** | CUDA 8.0 / r361 | **sm_60, sm_61, sm_62** |
| **6.0** | CUDA 9.0 / r384 | **sm_70** |
| **6.1** | CUDA 9.1 / r387 | **sm_72** |
| 6.2 | CUDA 9.2 / r396 | — |
| **6.3** | CUDA 10.0 / r400 | **sm_75** |
| 6.4 | CUDA 10.1 / r418 | — |
| 6.5 | CUDA 10.2 / r440 | — |
| **7.0** | CUDA 11.0 / r445 | **sm_80** |
| **7.1** | CUDA 11.1 / r455 | **sm_86** |
| 7.2 | CUDA 11.2 / r460 | — |
| 7.3 | CUDA 11.3 / r465 | — |
| **7.4** | CUDA 11.4 / r470 | **sm_87** |
| 7.5 | CUDA 11.5 / r495 | — |
| 7.6 | CUDA 11.6 / r510 | — |
| 7.7 | CUDA 11.7 / r515 | — |
| **7.8** | CUDA 11.8 / r520 | **sm_89, sm_90** |
| **8.0** | CUDA 12.0 / r525 | **sm_90a**（架构专属特性） |
| **8.1** | CUDA 12.1 / r530 | — |
| **8.2** | CUDA 12.2 / r535 | — |
| **8.3** | CUDA 12.3 / r545 | — |
| **8.4** | CUDA 12.4 / r550 | — |
| **8.5** | CUDA 12.5 / r555 | — |
| **8.6** | CUDA 12.7 / r565 | **sm_100, sm_100a, sm_101, sm_101a** |
| **8.7** | CUDA 12.8 / r570 | **sm_120, sm_120a** |
| **8.8** | CUDA 12.9 / r575 | **sm_103, sm_103a, sm_121, sm_121a**；引入 family-specific (f 后缀) 架构 |
| **9.0** | CUDA 13.0 / r580 | **sm_88, sm_110, sm_110f, sm_110a** |
| **9.1** | CUDA 13.1 / r590 | — |
| **9.2** | CUDA 13.2 / r595 | — |

---

## 各版本关键特性摘要

### PTX ISA 9.2（CUDA 13.2）

**新特性：**
- 新增 `add`、`sub`、`min`、`max`、`neg` 指令对 `.u8x4` 和 `.s8x4` 打包整数类型的支持。
- 新增 `add.sat.{u16x2/s16x2/u32}` 饱和加法指令。
- `st.async` 指令新增对 `.b128` 类型的支持。
- `cp.async.bulk` 指令新增 `.ignore_oob`（忽略越界）限定符。
- `cvt` 指令新增从 `.e4m3x2`、`.e5m2x2`、`.e3m2x2`、`.e2m3x2`、`.e2m1x2` 到 `.bf16x2` 的转换支持。

**语义变更与澄清：**
- 明确了 `wgmma.mma_async` 在 `.atype`/`.btype` 为 `.e4m3`/`.e5m2`、`.dtype` 为 `.f32` 时，累加精度高于半精度但低于单精度的实现行为。

---

### PTX ISA 9.1（CUDA 13.1）

**新特性：**
- `ld`、`st` 指令对 `.local` 状态空间新增 `.volatile` 限定符支持。
- `cvt` 指令新增从 `.f16x2`、`.bf16x2` 到低精度浮点类型（`.e2m1x2`、`.e2m3x2`、`.e3m2x2`、`.e4m3x2`、`.e5m2x2`）的转换。
- `mma`/`mma.sp` 指令新增 `.scale_vec::4X`（配合 `.kind::mxf4nvf4` 与 `.ue8m0` 类型）支持。
- `cvt` 指令新增 `.s2f6x2` 类型支持。
- 新增 `multimem.cp.async.bulk` 和 `multimem.cp.reduce.async.bulk` 指令。

---

### PTX ISA 9.0（CUDA 13.0）

**新特性：**
- 支持 `sm_88` 和 `sm_110` 目标架构（包括 family-specific `sm_110f` 和 architecture-specific `sm_110a`）。
- 新增 pragma `enable_smem_spilling`：将寄存器溢出到共享内存以减少延迟。
- 新增 pragma `frequency`：为基本块指定执行频率提示。
- 新增指令 `.blocksareclusters`：将 CUDA 线程块映射到 cluster。
- `st.bulk` 指令的 `size` 操作数扩展为支持 32 位长度。
- 新增性能调优指令 `.abi_preserve` 和 `.abi_preserve_control`，用于指定调用者应保存的数据寄存器和控制寄存器数量。

**重命名说明：** `sm_{101,101f,101a}` 自 PTX ISA 9.0 起重命名为 `sm_{110,110f,110a}`。

---

### PTX ISA 8.8（CUDA 12.9）

**新特性：**
- 支持 `sm_103`、`sm_103a`、`sm_121`、`sm_121a` 目标架构。
- **引入 family-specific 目标架构**（以 "f" 为后缀），PTX 与同一家族后续目标兼容。新增：`sm_100f`、`sm_101f`、`sm_103f`、`sm_120f`、`sm_121f`。
- `min`、`max` 指令扩展为支持三个输入参数。
- `tcgen05.mma` 指令新增 `.block16`、`.block32` 等 scale_vectorsize 限定符及 K 维度 96 支持。
- `tensormap.replace` 的 `.field3` 新增 96B swizzle 模式支持。
- 新增 `tcgen05.ld.red` 指令。
- `ld`、`ld.global.nc`、`st` 指令扩展支持 256 位加载/存储。

**语义变更：** 澄清了浮点数到整数转换时 NaN 输入的行为。

---

### PTX ISA 8.7（CUDA 12.8）

**新特性：**
- 支持 `sm_120`、`sm_120a` 目标架构。
- `tcgen05.mma` 新增 `.kind::mxf4nvf4` 和 `.scale_vec::4X` 限定符。
- `mma` 指令新增 FP16 类型累加器及 FP8 类型（`.e4m3`/`.e5m2`）支持（形状 `.m16n8k16`）。
- `cvt` 指令新增 `.rs` 舍入模式，以及目标类型 `.e2m1x4`、`.e4m3x4`、`.e5m2x4`、`.e3m2x4`、`.e2m3x4`。
- `st.async`、`red.async` 扩展支持 `.mmio`、`.release`、`.global`、`.scope` 限定符。
- `tensormap.replace` 的 `.elemtype` 限定符值域扩展至 13–15。
- `mma`/`mma.sp::ordered_metadata` 扩展支持 `.e3m2`/`.e2m3`/`.e2m1` 类型及 `.kind`、`.block_scale`、`.scale_vec_size` 限定符。

---

### PTX ISA 8.6（CUDA 12.7）

**新特性：**
- 支持 `sm_100`、`sm_100a`、`sm_101`、`sm_101a` 目标架构（Blackwell GPU 系列）。
- `cp.async.bulk` / `cp.async.bulk.tensor` 扩展：增加 `.shared::cta` 作为目标状态空间。
- `fence` 指令新增 `.acquire`、`.release` 以及 `.sync_restrict` 限定符。
- `ldmatrix` 扩展支持 `.m16n16`、`.m8n16` 形状、`.b8` 类型，以及 `.src_fmt`/`.dst_fmt` 限定符。
- `stmatrix` 扩展支持 `.m16n8` 形状和 `.b8` 类型。
- 新增 `clusterlaunchcontrol` 指令。
- `add`/`sub`/`fma` 扩展支持混合精度浮点（`.f32` 目标 + `.f16`/`.bf16` 源）。
- `add`/`sub`/`mul`/`fma` 新增 `.f32x2` 类型支持。
- `cvt` 对 `.tf32` 类型的 `.rn`/`.rz` 舍入模式增加 `.satfinite` 支持。
- `cp.async.bulk` 新增 `.cp_mask` 限定符和 `byteMask` 操作数。
- `multimem.ld_reduce`/`multimem.st` 扩展支持多种 FP8 类型。
- `cvt` 指令新增到/从 `.e2m1x2`、`.e3m2x2`、`.e2m3x2`、`.ue8m0x2` 类型的转换。
- `cp.async.bulk.tensor` 新增 `.tile::scatter4`、`.tile::gather4`、`.im2col::w`、`.im2col::w::128` 加载模式。
- `tensormap.replace` 新增 `.swizzle_atomicity` 限定符。
- `mbarrier` 系列指令新增 `.relaxed` 限定符。
- 新增 `st.bulk` 指令。
- **引入 tcgen05 指令集**：`tcgen05.alloc`、`tcgen05.dealloc`、`tcgen05.ld`、`tcgen05.st`、`tcgen05.mma`、`tcgen05.mma.sp`、`tcgen05.mma.ws` 等，用于 Blackwell TensorCore 5th Generation。
- `redux.sync` 指令扩展支持 `.f32` 类型及 `.abs`、`.NaN` 限定符。

---

### PTX ISA 8.5（CUDA 12.5）

**新特性：**
- 新增 `mma.sp::ordered_metadata` 指令。

**语义变更：** 明确了 `mma.sp` 稀疏元数据操作数 `e` 的非法值（`0b0000`、`0b0101`、`0b1010`、`0b1111`）。

---

### PTX ISA 8.4（CUDA 12.4）

**新特性：**
- `ld`、`st`、`atom` 指令的 `.b128` 类型新增 `.sys` 作用域支持。
- 整数型 `wgmma.mma_async` 指令新增 `.u8.s8` 和 `.s8.u8` 类型组合支持。
- `mma`、`mma.sp` 扩展支持 FP8 类型 `.e4m3` 和 `.e5m2`。

---

### PTX ISA 8.3（CUDA 12.3）

**新特性：**
- 新增 pragma `used_bytes_mask`：指定加载操作中实际使用字节的掩码。
- `isspacep`、`cvta.to`、`ld`、`st` 指令扩展支持 `.param` 状态空间的 `::entry`/`::func` 子限定符。
- `ld`、`ld.global.nc`、`ldu`、`st`、`mov`、`atom` 指令新增 `.b128` 类型支持。
- 新增 `tensormap.replace`、`tensormap.cp_fenceproxy` 指令，支持修改 tensor-map 对象。

---

### PTX ISA 8.2（CUDA 12.2）

**新特性：**
- `ld`/`st` 指令新增 `.mmio` 限定符支持。
- `lop3` 指令扩展：允许谓词目标操作数。
- `multimem.ld_reduce` 扩展支持 `.acc::f32` 限定符（允许以 f32 精度进行中间累加）。
- `wgmma.mma_async` 扩展支持 `.sp` 修饰符（稀疏矩阵 A 的 WGMMA 操作）。

**语义澄清：** 明确 `cp.async.bulk`/`cp.async.bulk.tensor` 的 `.multicast::cluster` 限定符针对 `sm_90a` 进行了优化，在其他目标上性能可能大幅下降。

---

### PTX ISA 8.1（CUDA 12.1）

**新特性：**
- 新增 `st.async`（异步存储）和 `red.async`（异步规约）指令，作用于共享内存。
- 半精度 `fma` 指令新增 `.oob` 修饰符。
- `cvt` 指令的 `.f16`、`.bf16`、`.tf32` 格式新增 `.satfinite` 饱和修饰符。
- `cvt` 对 `.e4m3`/`.e5m2` 的支持扩展到 `sm_89`。
- `atom`/`red` 指令扩展支持向量类型。
- 新增特殊寄存器 `%aggr_smem_size`。
- `sured` 指令扩展支持 64 位 `min`/`max` 操作。
- kernel 参数大小上限增至 32764 字节。
- 新增对 multimem 地址的内存一致性模型支持。
- 新增 `multimem.ld_reduce`、`multimem.st`、`multimem.red` 指令。

---

### PTX ISA 8.0（CUDA 12.0）

**新特性：**
- 支持 `sm_90a`（架构专属特性）目标架构（Hopper GPU）。
- 引入异步 Warpgroup 级矩阵乘加操作 `wgmma`（`wgmma.mma_async`）。
- 扩展异步复制操作，支持针对大数据和张量数据的 bulk 操作（包括 `cp.async.bulk.tensor`）。
- 引入打包整数类型 `.u16x2`、`.s16x2`。
- `add`/`min`/`max` 等整数指令扩展支持打包整数类型及 `.relu` 饱和修饰符。
- 新增特殊寄存器 `%current_graph_exec`（标识当前执行的 CUDA 设备图）。
- 新增 `elect.sync` 指令。
- 函数和变量新增 `.unified` 属性。
- 新增 `setmaxnreg` 指令（设置最大寄存器数）。
- `barrier.cluster` 指令新增 `.sem` 限定符。
- `fence` 指令扩展支持 `op_restrict` 限定符。
- `mbarrier` 系列扩展：支持 `.cluster` 作用域；支持 `.expect_tx`/`.complete_tx` 事务计数操作。

---

### PTX ISA 7.8（CUDA 11.8）

**新特性：**
- 支持 `sm_89`（Ada Lovelace）和 `sm_90`（Hopper）目标架构。
- `bar`/`barrier` 指令新增可选 `.cta` 作用域限定符。
- `.shared` 状态空间扩展：新增 `::cta`、`::cluster` 子限定符。
- 新增 `movmatrix` 指令（在 warp 内跨寄存器转置矩阵）。
- 新增 `stmatrix` 指令（将矩阵存储到共享内存）。
- `mma` 的 `.f64` 浮点类型扩展新形状：`.m16n8k4`、`.m16n8k8`、`.m16n8k16`。
- `add`/`sub`/`mul`/`set`/`setp`/`cvt`/`tanh`/`ex2`/`atom`/`red` 等指令扩展支持 `bf16` 格式。
- 引入新的替代浮点格式 `.e4m3` 和 `.e5m2`（FP8）。
- 新增 `griddepcontrol` 指令（控制依赖网格的执行）。
- `mbarrier` 新增 `try_wait` 阶段完成检查操作。
- 引入新线程作用域 `.cluster`（一组 CTA 构成的集群）。
- `fence`/`membar`、`ld`、`st`、`atom`、`red` 等指令扩展支持 `.cluster` 作用域。
- 新增 `mapa` 指令（将共享内存地址映射到集群内其他 CTA 的对应地址）。
- 新增 `getctarank` 指令。
- 新增 `barrier.cluster` 集群级屏障同步指令。
- 新增与集群相关的特殊寄存器：`%is_explicit_cluster`、`%clusterid`、`%nclusterid`、`%cluster_ctaid`、`%cluster_nctaid`、`%cluster_ctarank`、`%cluster_nctarank`。
- 新增集群维度指令：`.reqnctapercluster`、`.explicitcluster`、`.maxclusterrank`。

---

### PTX ISA 7.7（CUDA 11.7）

**新特性：**
- `isspacep`/`cvta` 指令扩展：支持 kernel 函数参数的 `.param` 状态空间。

---

### PTX ISA 7.6（CUDA 11.6）

**新特性：**
- 新增 `szext` 指令（对指定值进行符号扩展或零扩展）。
- 新增 `bmsk` 指令（从指定位置起创建指定宽度的位掩码）。
- 新增特殊寄存器：`%reserved_smem_offset_begin`、`%reserved_smem_offset_end`、`%reserved_smem_offset_cap`、`%reserved_smem_offset<2>`（用于描述系统软件保留的共享内存区域）。

---

### PTX ISA 7.5（CUDA 11.5）

**新特性：**
- 调试信息增强：`.section` 调试指令支持标签差值和负值。
- `cp.async` 指令新增 `ignore-src` 操作数。
- 内存一致性模型扩展：引入内存代理（memory proxy）概念和虚拟别名（virtual aliases）。
- 新增 `fence.proxy`、`membar.proxy` 指令，用于通过虚拟别名进行内存访问同步。

---

### PTX ISA 7.4（CUDA 11.4）

**新特性：**
- 支持 `sm_87` 目标架构。
- 新增 `.level::eviction_priority` 限定符（用于 `ld`/`st`/`prefetch` 指令的缓存驱逐优先级提示）。
- 新增 `.level::prefetch_size` 限定符（用于 `ld`/`cp.async` 指令的数据预取大小提示）。
- 新增 `createpolicy` 指令（构建缓存驱逐策略）。
- 新增 `.level::cache_hint` 限定符（供 `ld`/`st`/`atom`/`red`/`cp.async` 使用）。
- 新增 `applypriority`、`discard` 操作（用于缓存数据管理）。

---

### PTX ISA 7.3（CUDA 11.3）

**新特性：**
- `mask()` 运算符扩展：支持整数常量表达式。
- 新增栈操作指令：`stacksave`、`stackrestore`（栈保存与恢复）和 `alloca`（每线程栈分配）。

---

### PTX ISA 7.2（CUDA 11.2）

**新特性：**
- `.loc` 指令增强，用于表示内联函数信息。
- 支持在调试节（debug sections）中定义标签。
- `min`/`max` 指令新增 `.xorsign` 和 `.abs` 修饰符。

---

### PTX ISA 7.1（CUDA 11.1）

**新特性：**
- 支持 `sm_86` 目标架构（Ampere GA102 系列）。
- 新增 `mask()` 运算符（从变量地址提取特定字节，用于初始化器）。
- `tex`/`tld4` 指令扩展：可返回可选谓词，指示指定坐标处的数据是否驻留内存。
- 单比特 `wmma`/`mma` 指令扩展支持 `.and` 操作。
- `mma` 指令新增 `.sp` 修饰符（稀疏矩阵 A 的 MMA 操作）。
- `mbarrier.test_wait` 扩展：支持测试特定阶段奇偶的完成情况。

---

### PTX ISA 7.0（CUDA 11.0）

**新特性：**
- 支持 `sm_80` 目标架构（Ampere A100）。
- 引入异步复制指令（`cp.async`），支持跨状态空间的异步数据搬运。
- 引入 `mbarrier` 指令族：用于线程与异步复制操作的同步。
- 新增 `redux.sync` 指令（warp 内跨线程规约）。
- 引入新浮点格式 `.bf16`（BFloat16）和 `.tf32`（TensorFloat-32）。
- `wmma`/`mma` 指令大幅扩展：支持 `.f64`、`.bf16`、`.tf32` 类型及众多新形状。
- `abs`/`neg`/`min`/`max`/`fma`/`cvt`/`ex2` 等指令扩展支持 `.bf16`/`.bf16x2` 格式。
- 新增 `tanh` 指令（双曲正切）。

---

### PTX ISA 6.5（CUDA 10.2）

**新特性：**
- `set` 半精度比较指令支持整数目标类型。
- `abs` 指令扩展支持 `.f16`/`.f16x2`。
- 新增 `cvt.pack` 指令（将两个整数值转换并打包）。
- `mma` 指令新增形状 `.m16n8k8`、`.m8n8k16`、`.m8n8k32`。
- 新增 `ldmatrix` 指令（从共享内存加载矩阵，用于 `mma`）。

**移除特性：** 移除 `wmma.mma` 指令对 `.satfinite` 限定符的支持（此前于 6.4 已废弃）。

---

### PTX ISA 6.4（CUDA 10.1）

**新特性：**
- 新增 `.noreturn` 指令（标记函数不返回）。
- 新增 `mma` 指令（矩阵乘加操作，Tensor Core）。

**废弃特性：** `wmma.mma` 浮点指令的 `.satfinite` 限定符废弃。  
**移除特性：** `sm_70` 及更高架构上的 `shfl`/`vote` 无 `.sync` 用法被移除。

---

### PTX ISA 6.3（CUDA 10.0）

**新特性：**
- 支持 `sm_75` 目标架构（Turing GPU）。
- 新增 `nanosleep` 指令（使线程休眠指定时长）。
- 新增 `.alias` 指令（为函数符号定义别名）。
- `atom`/`red` 指令扩展：支持 `.f16` 加法和 `.cas.b16` 操作。
- `wmma` 指令扩展：支持 `.s8`、`.u8`、`.s4`、`.u4`、`.b1` 乘数矩阵类型和 `.s32` 累加器类型。

---

### PTX ISA 6.2（CUDA 9.2）

**新特性：**
- 新增 `activemask` 指令（查询 warp 中活跃线程）。
- `atom`/`red` 指令扩展：支持 `.f16x2` 加法（强制 `.noftz` 限定符）。

**废弃特性：** `shfl`/`vote` 无 `.sync` 的用法（已追溯废弃至 PTX ISA 6.0）。

---

### PTX ISA 6.1（CUDA 9.1）

**新特性：**
- 支持 `sm_72` 目标架构。
- `wmma` 指令新增矩阵形状 `32x8x16` 和 `8x32x16`。

---

### PTX ISA 6.0（CUDA 9.0）

**新特性：**
- 支持 `sm_70` 目标架构（Volta GPU）。
- 为 `sm_70` 及更高架构引入**内存一致性模型**（包含同步语义和作用域）。
- 引入 `wmma` 指令族（矩阵加载/乘加/存储操作，初代 Tensor Core）。
- 新增 `barrier` 指令（增强版屏障同步）。
- 新增 `fns`（查找第 n 个置位比特）、`bar.warp.sync`、`match.sync`、`brx.idx` 等指令。
- `vote`/`shfl` 指令新增 `.sync` 修饰符。
- 支持 `.func` 的无大小数组参数（用于实现变参函数）。

---

### PTX ISA 5.0（CUDA 8.0）

**新特性：**
- 支持 `sm_60`、`sm_61`、`sm_62` 目标架构（Pascal GPU）。
- `atom`/`red` 指令扩展：支持双精度加法操作和 `scope` 修饰符。
- 新增 `.common` 指令（允许不同大小的同名符号跨目标文件链接）。
- 新增 `dp4a`（4 路点积累加）和 `dp2a`（2 路点积累加）指令。
- 新增特殊寄存器 `%clock_hi`。

---

### PTX ISA 4.3（CUDA 7.5）

**新特性：**
- 新增 `lop3` 指令（对 3 个输入执行任意逻辑操作）。
- `bfe`/`bfi` 等扩展精度算术指令扩展支持 64 位计算。
- `tex.grad`/`tld4` 扩展支持 `cube`、`acube` 几何形状。

---

### PTX ISA 4.2（CUDA 7.0）

**新特性：**
- 支持 `sm_53` 目标架构。
- 支持 `.f16`/`.f16x2` 类型的算术、比较、纹理指令。
- Surface 新增 `memory_layout` 字段，`suq` 指令支持查询。

---

### PTX ISA 4.1（CUDA 6.5）

**新特性：**
- 支持 `sm_37`、`sm_52` 目标架构。
- Texture/Surface 新增多个字段（`array_size`、`num_mipmap_levels`、`num_samples`），`txq`/`suq` 指令支持查询。
- 新增特殊寄存器 `%total_smem_size`、`%dynamic_smem_size`。

---

### PTX ISA 4.0（CUDA 6.0）

**新特性：**
- 支持 `sm_32`、`sm_50` 目标架构。
- 新增 64 位性能计数器特殊寄存器 `%pm0_64..%pm7_64`。
- 新增 `istypep` 指令（类型测试）。
- 新增 `rsqrt.approx.ftz.f64` 指令（快速平方根倒数近似）。
- 新增 `.attribute` 指令（指定变量特殊属性）和 `.managed` 变量属性。

---

### PTX ISA 3.2（CUDA 5.5）

**新特性：**
- 纹理指令支持多采样纹理（multi-sample / multisample array textures）。
- `.section` 调试指令扩展：支持标签 + 立即数表达式。
- `.file` 指令扩展：包含时间戳和文件大小信息。

---

### PTX ISA 3.1（CUDA 5.0）

**新特性：**
- 支持 `sm_35` 目标架构。
- 支持 CUDA 动态并行（Dynamic Parallelism），允许 kernel 创建并同步新任务。
- 新增 `ld.global.nc`（通过非相干纹理缓存加载只读全局数据）。
- 新增 `shf`（漏斗移位）指令。
- `atom`/`red` 扩展支持 64 位 `{and,or,xor}` 和 64 位整数 `{min,max}` 操作。
- 支持 mipmap 和间接纹理/Surface 访问。
- `.const` 状态空间扩展支持通用寻址；新增 `generic()` 运算符。
- 新增 `.weak` 指令（允许多个目标文件包含同名符号声明）。

---

### PTX ISA 3.0（CUDA 4.1）

**新特性：**
- 支持 `sm_30` 目标架构（Kepler GPU）。
- 引入 SIMD 视频指令（视频处理加速）。
- 新增 warp shuffle 指令（warp 内数据广播与交换）。
- 新增 `mad.cc`/`madc` 指令（高效扩展精度整数乘法）。
- Surface 指令扩展支持 3D 和数组几何形状。
- 纹理指令扩展支持 cubemap 和 cubemap 数组纹理。
- 性能计数器特殊寄存器扩展至 `%pm4..%pm7`。
- `pmevent.mask` 支持同时触发多个性能监控事件。

**语义变更：** `%gridid` 从 32 位扩展至 64 位；模块级 `.reg`/`.local` 变量在 ABI 编译模式下废弃。

---

### PTX ISA 2.3（CUDA 4.0）

**新特性：**
- 支持纹理数组（1D/2D 纹理数组）。
- 新增 `.address_size` 指令（指定地址大小）。
- `.const`/`.global` 状态空间变量默认初始化为零。

---

### PTX ISA 2.2（CUDA 3.2）

**新特性：**
- 新增 kernel 参数属性指令（指定参数为指针及其目标状态空间/对齐）。
- `.samplerref` 新增 `force_unnormalized_coords` 字段（支持 OpenCL 采样器语义）。
- `.const` 状态空间废弃显式常量块，改为统一大型平坦地址空间。
- 新增 `tld4` 指令（从双线性插值足迹的四个纹素加载一个分量）。

---

### PTX ISA 2.1（CUDA 3.1）

**新特性：**
- 为 `sm_2x` 目标实现了基于栈的 ABI。
- 实现了间接调用（Indirect Call）支持（`sm_2x`）。
- 新增 `.branchtargets`、`.calltargets`、`.callprototype` 指令。
- 全局/常量变量名可用于初始化器中表示其地址。
- 新增 32 个驱动专属执行环境特殊寄存器 `%envreg0..%envreg31`。
- 纹理/Surface 新增通道数据类型和通道顺序字段，`txq`/`suq` 支持查询。
- 新增 `.reqntid` 指令（指定精确的 CTA 维度）；`.maxnctapersm` 替换为 `.minnctapersm`。
- 新增 `rcp.approx.ftz.f64` 指令（快速双精度倒数近似）。

---

### PTX ISA 2.0（CUDA 3.0）

**新特性（浮点扩展）：**
- `sm_20` 单精度指令默认支持非规格化数（subnormal numbers）；`.ftz` 修饰符保持向后兼容。
- `add`/`sub`/`mul` 新增 `.rm`/`.rp` 舍入修饰符支持。
- 新增 IEEE 754 兼容的单精度 FMA 指令（`fma.f32`）。
- `div`/`rcp`/`sqrt` 新增 IEEE 754 兼容版本（带舍入修饰符）。
- 新增 `testp`、`copysign` 指令。

**新增指令：**
- `ldu`（加载统一数据）、`clz`（前导零计数）、`bfind`（查找前导非符号位）、`brev`（位反转）、`bfe`/`bfi`（位域提取/插入）、`popc`（人口计数）、`vote.ballot.b32`、`isspacep`（通用地址状态空间查询）、`cvta`（通用地址转换）等。
- `bar` 指令扩展：新增 `bar.arrive`、`bar.red.popc.u32`、`bar.red.{and,or}.pred`。
- 新增标量视频指令（包含 `prmt`）。
- 新增特殊寄存器：`%nwarpid`、`%nsmid`、`%clock64`、`%lanemask_{eq,le,lt,ge,gt}`。
- `ld`/`st` 等指令扩展支持通用寻址。
- 新增缓存操作修饰符（用于预取）。
- 新增 `.pragma nounroll` 指令。
- `.section` 指令替代 `@@DWARF` 语法。

---

*本文档为 PTX ISA 9.2 第十三章中文总结，仅供参考。*
