# 第 8 章：内存一致性模型

在多线程执行中，每个线程的内存操作副作用对其他线程的可见顺序是**部分的、非一致的**。任意两个操作在不同线程看来，可能表现为没有顺序，或顺序相反。内存一致性模型通过公理（axioms）明确规定：不同线程观察到的顺序之间，哪些矛盾是被**禁止的**。

---

## 8.1 模型适用范围

- 适用于任意 PTX ISA 版本、运行在 **sm_70 及以上**架构的程序
- **不适用**于纹理（包括 `ld.global.nc`）和 surface 访问

### 8.1.1 系统范围原子性的限制

与 host CPU 通信时，某些 system scope 的 strong 操作在部分系统上**可能不是原子的**。详见 CUDA Atomicity Requirements。

---

## 8.2 内存操作

PTX 内存模型的**基本存储单位是字节（8 位）**。每个状态空间是连续字节序列，每个字节都有对所有可访问该空间的线程唯一的地址。

每条 PTX 内存指令指定：
- 一个地址操作数（虚拟地址，运行时转换为物理地址）
- 一个数据类型

物理地址 + 数据类型大小共同定义**物理内存位置**（memory location）。

### 8.2.1 重叠（Overlap）

- 两个内存位置**重叠**：一个的起始地址落在另一个的字节范围内
- 两个内存操作**重叠**：指定相同虚拟地址且对应物理位置重叠
- **完全重叠**：两个内存位置完全相同；否则为**部分重叠**

### 8.2.2 别名（Aliases）

两个不同的虚拟地址映射到同一物理内存位置时，称为**别名（aliases）**。

### 8.2.3 Multimem 地址

Multimem 地址是指向**多个设备上不同内存位置**的虚拟地址。只有 `multimem.*` 操作可以访问 multimem 地址，其他访问行为未定义。

### 8.2.4 向量数据类型的内存操作

向量内存操作被建模为一组等价标量内存操作，以**未指定顺序**作用于各元素（标量最大 64 位）。

### 8.2.5 打包数据类型的内存操作

打包类型（如 `.f16x2`）包含两个相同标量类型的值，存储在相邻内存位置。打包内存操作被建模为一对等价标量操作，以**未指定顺序**执行。

### 8.2.6 初始化

内存中每个字节由一个假想写操作 **W0** 在任何线程启动前初始化：
- 若字节属于程序变量且有初始值，W0 写入该初始值
- 否则，W0 写入未知但固定的值

---

## 8.3 状态空间

内存一致性模型中的关系独立于状态空间。因果序（causality order）跨越所有状态空间的所有内存操作。但一个操作的副作用**只能被同一状态空间内的操作直接观察**。

例：`ld.relaxed.shared.sys` 与 `ld.relaxed.shared.cluster` 的同步效果相同，因为 cluster 之外的线程无法访问同一 shared memory 位置。

---

## 8.4 操作类型

**表 20：操作类型**

| 操作类型 | 对应指令/操作 |
|---|---|
| 原子操作（atomic） | `atom` 或 `red` 指令 |
| 读操作（read） | 所有 `ld` 变体及 `atom`（不含 `red`） |
| 写操作（write） | 所有 `st` 变体，以及导致写入的原子操作 |
| 内存操作（memory） | 读或写操作 |
| volatile 操作 | 带 `.volatile` 修饰符的指令 |
| acquire 操作 | 带 `.acquire` 或 `.acq_rel` 修饰符的内存操作 |
| release 操作 | 带 `.release` 或 `.acq_rel` 修饰符的内存操作 |
| mmio 操作 | 带 `.mmio` 修饰符的 `ld` 或 `st` |
| 内存栅栏操作（memory fence） | `membar`、`fence.sc`、`fence.acq_rel` |
| 代理栅栏操作（proxy fence） | `fence.proxy` 或 `membar.proxy` |
| strong 操作 | 内存栅栏，或带 `.relaxed/.acquire/.release/.acq_rel/.volatile/.mmio` 的内存操作 |
| weak 操作 | 带 `.weak` 修饰符的 `ld` 或 `st` |
| 同步操作（synchronizing） | barrier 指令、fence 操作、release 操作、acquire 操作 |

### 8.4.1 mmio 操作

mmio 操作通常用于访问映射到**对等 I/O 设备控制寄存器**的内存位置。其精确语义由底层 I/O 设备定义。从一致性模型角度等价于 strong 操作，但有如下实现特性：
- 写操作总是被执行，在指定 scope 内**不会合并**
- 读操作总是被执行，**不会被转发、预取、合并或命中缓存**
- 例外：某些实现可能额外加载周围位置（32~128 字节）

### 8.4.2 volatile 操作

volatile 操作等价于 system scope 的 relaxed 内存操作，并附加以下实现约束：
- volatile **指令**的执行数量被保留（但硬件可以合并多个 volatile **操作**）
- volatile 指令之间不会重排序，但其**操作**可以相互重排

> **注意**：volatile 操作不适合用于**线程间同步**（推荐用 `ld.relaxed.sys`/`st.relaxed.sys`），也不适合 **MMIO**（应使用 `.mmio` 操作）。

---

## 8.5 作用域（Scope）

每个 strong 操作必须指定一个**作用域**，即可以与该操作直接交互、建立一致性模型关系的线程集合。

**表 21：作用域**

| 作用域 | 描述 |
|---|---|
| `.cta` | 与当前线程在同一 CTA 中执行的所有线程 |
| `.cluster` | 与当前线程在同一 cluster 中执行的所有线程 |
| `.gpu` | 当前程序在同一计算设备上执行的所有线程（包括 host 发起的其他 kernel grid） |
| `.sys` | 当前程序的所有线程，包括所有设备上的所有 kernel grid 以及 host 程序本身的所有线程 |

> warp **不是**作用域；CTA 是内存一致性模型中最小的线程集合作用域。

---

## 8.6 代理（Proxies）

**内存代理（memory proxy）** 是应用于内存访问方法的抽象标签。当两个内存操作使用不同的访问方法时，称为**不同代理**。

- 普通内存操作使用 **generic proxy**
- 纹理、surface 等使用各自独立的代理
- 跨不同代理同步需要 **proxy fence**
- 虚拟别名（virtual aliases）虽然都使用 generic proxy，但访问不同虚拟地址的行为类似不同代理，也需要 proxy fence 建立内存序

---

## 8.7 道义强（Morally Strong）操作

两个操作满足以下**所有**条件时，称为相互 **morally strong**：

1. 它们在**程序序**中相关（同一线程执行），**或**双方都是 strong 操作且各自指定的 scope 包含对方线程
2. 两者通过**相同的 proxy** 执行
3. 若两者都是内存操作，则它们**完全重叠**

### 8.7.1 冲突与数据竞争

- **冲突**：两个重叠内存操作中至少有一个是写操作
- **数据竞争（data-race）**：两个冲突操作既不在因果序中相关，又不是 morally strong

### 8.7.2 混合大小数据竞争的限制

- **均匀大小数据竞争**：完全重叠的操作之间的数据竞争
- **混合大小数据竞争**：部分重叠的操作之间的数据竞争

若程序包含混合大小数据竞争，一致性模型公理**不适用**。

**混合大小 RMW 原子性**：对任意一对重叠原子操作 A1 和 A2（各自 scope 包含对方），A1 的 RMW 要么完全在 A2 启动前完成，要么反之。无论是完全重叠还是部分重叠均成立。

---

## 8.8 Release 与 Acquire 模式

**Release 模式**使当前线程的先前操作对其他线程的某些操作可见。

**Release 模式**作用于位置 M 的形式（满足其一即可）：
1. 对 M 的 release 操作：`st.release [M];`
2. 对 M 的 release/acq_rel 操作 + 程序序中之后对 M 的 strong 写：`st.release [M]; st.relaxed [M];`
3. release/acq_rel 内存栅栏 + 程序序中之后对 M 的 strong 写：`fence.release; st.relaxed [M];`

> release 模式建立的内存同步只影响该模式**第一条指令之前**（程序序）的操作。

**Acquire 模式**使其他线程的某些操作对当前线程的后续操作可见。

**Acquire 模式**作用于位置 M 的形式（满足其一即可）：
1. 对 M 的 acquire 操作：`ld.acquire [M];`
2. 对 M 的 strong 读 + 程序序中之后对 M 的 acquire 操作：`ld.relaxed [M]; ld.acquire [M];`
3. 对 M 的 strong 读 + 程序序中之后的 acquire 内存栅栏：`ld.relaxed [M]; fence.acquire;`

> acquire 模式建立的内存同步只影响该模式**最后一条指令之后**（程序序）的操作。

> **注意**：原子 reduction（如 `red.add`）虽然概念上包含 strong read，但该 read **不构成** acquire 模式。

---

## 8.9 内存操作的顺序

- **程序序（program order）**：同一线程内操作按顺序处理器执行顺序的全序关系
- **因果序（causality order）**：通过同步操作跨线程建立的操作可见性顺序
- **通信序（communication order）**：捕获内存操作副作用对其他操作可见性的顺序

### 8.9.1 程序序

程序序是同一线程内所有操作的全序，不跨越线程。

#### 异步操作

以下指令的操作**异步于**发出该指令的线程：

| 指令 | 说明 |
|---|---|
| `cp.async` | 各 cp.async 操作之间无序，但相对于 `cp.async.commit_group` / `cp.async.mbarrier.arrive` 有序 |
| `cp.async.bulk` | 隐式 mbarrier complete-tx 仅与同一异步指令的内存操作有序 |
| `cp.reduce.async.bulk` | 同上 |
| `wgmma.mma_async` | 异步于发出线程 |

### 8.9.2 观察序（Observation Order）

写操作 W 通过可选的原子 RMW 操作序列先于读操作 R：
1. R 与 W 是 morally strong 且 R 读取 W 写入的值
2. 存在原子操作 Z，使 W→Z→R 在观察序中成立（传递）

### 8.9.3 Fence-SC 序

`fence.sc` 操作之间由运行时确定的**无环偏序**，称为 Fence-SC 序。

### 8.9.4 内存同步（synchronizes-with）

不同线程的同步操作在运行时相互 **synchronize**，从而建立因果序：

1. `fence.sc` X synchronizes-with `fence.sc` Y：若 X 在 Fence-SC 序中先于 Y
2. `bar.cta.sync/red/arrive` synchronizes-with `bar.cta.sync/red`：作用于同一 barrier
3. `barrier.cluster.arrive` synchronizes-with `barrier.cluster.wait`
4. Release 模式 X synchronizes-with acquire 模式 Y：若 X 中的写操作在观察序中先于 Y 中的读操作，且 X 的首操作与 Y 的末操作是 morally strong

**CUDA API 同步**：以下 API 操作也建立 synchronizes-with 关系：
- CUDA stream 中相邻任务（前者完成 → 后者开始）
- CUDA event 记录 → event 等待
- kernel 开始 → kernel 内所有线程开始；所有线程结束 → kernel 结束
- CUDA graph 节点依赖关系
- `cudaStreamSynchronize` / `cudaEventSynchronize` 返回

### 8.9.5 因果序（Causality Order）

**基础因果序（Base Causality Order）**——传递关系，运行时确定：

操作 X 先于 Y（满足其一）：
1. X 先于 Y 在**程序序**中
2. X **synchronizes-with** Y
3. 存在 Z，使 X→Z→Y（通过程序序和基础因果序的任意组合传递）

**代理保留基础因果序（Proxy-preserved Base Causality Order）**：  
X 先于 Y 在基础因果序中，且满足：
- 两者访问同一地址，使用 generic proxy；或
- 两者访问同一地址，使用相同 proxy，且在同一 thread block；或
- 两者是别名且因果路径上有 alias proxy fence

**因果序** = proxy-preserved 基础因果序 + 观察序的非传递扩展。

### 8.9.6 一致序（Coherence Order）

对重叠写操作的偏序（传递）：两个重叠写操作若是 morally strong 或在因果序中相关，则在一致序中有序。数据竞争中的两个写操作在一致序中**无序**（体现"偏"序的原因）。

### 8.9.7 通信序（Communication Order）

非传递顺序，建立写操作与其他重叠内存操作之间的可见性：

1. 写 W 先于读 R：R 返回了 W 写入的某个字节的值
2. 写 W 先于写 W'：W 在一致序中先于 W'
3. 读 R 先于写 W：对 R 和 W 都访问的每个字节，R 返回了在一致序中先于 W 的某个写 W' 的值

---

## 8.10 公理（Axioms）

### 8.10.1 一致性（Coherence）

若写 W 在**因果序**中先于重叠写 W'，则 W 必须在**一致序**中先于 W'。

### 8.10.2 Fence-SC

Fence-SC 序不能与因果序矛盾：若 morally strong 的 `fence.sc` F1 在因果序中先于 F2，则 F1 必须在 Fence-SC 序中先于 F2。

### 8.10.3 原子性（Atomicity）

**单副本原子性**：冲突的 morally strong 操作以单副本原子方式执行。当读 R 和写 W 是 morally strong 时，以下两个通信关系不能同时存在：
1. R 从 W 读取某字节
2. R 从在一致序中先于 W 的某写 W' 读取某字节

**RMW 原子性**：当原子操作 A 与写 W 重叠且 morally strong 时，以下两个通信关系不能同时存在：
1. A 从在一致序中先于 W 的某写 W' 读取某字节
2. A 在一致序中跟随 W

```
// Litmus Test 1（保证原子性）
// T1: atom.sys.inc [x]
// T2: atom.sys.inc [x]
// 最终状态：x == 2 ✓（morally strong，保证原子性）

// Litmus Test 2（不保证原子性）
// T1 (CTA A): atom.cta.inc [x]
// T2 (CTA B): atom.gpu.inc [x]
// 最终状态：x == 1 OR x == 2（不是 morally strong）
```

### 8.10.4 无凭空值（No Thin Air）

值不能"凭空出现"：执行过程中不能以推测性的、通过指令依赖链和线程间通信自我满足的方式产生值。

```
// Litmus Test：带依赖的 Load Buffering（LB+deps）——禁止 x==1 AND y==1
// T1: r0 = x; y = r0;
// T2: r1 = y; x = r1;
// 若初始 x==y==0，最终 x==y==0（自我满足的循环被禁止）

// Litmus Test：不带依赖的 Load Buffering——允许 x==1 AND y==1
// T1: r0 = x; y = 1;
// T2: r1 = y; x = 1;
// 无自我满足循环，允许
```

### 8.10.5 按位置顺序一致（Sequential Consistency Per Location）

在任意一组两两 morally strong 的重叠内存操作中，通信序不能与程序序矛盾——即程序序与通信序中 morally strong 关系的拼接不能形成环。

```
// Litmus Test：CoRR（相干读-读）
// T1: st.relaxed.sys [x], 1
// T2: r0 = ld.relaxed.sys [x]; r1 = ld.relaxed.sys [x]
// 若 r0==1 则 r1==1（不能"看到又没看到"同一个写）
```

### 8.10.6 因果性（Causality）

通信序不能与因果序矛盾：
1. 若读 R 在因果序中先于重叠写 W，则 R 不能从 W 读取
2. 若写 W 在因果序中先于重叠读 R，则对两者都访问的字节，R 不能从在一致序中先于 W 的写 W' 读取

```
// Litmus Test：消息传递（MP）
// T1: st [data], 1; fence.sys; st.relaxed.sys [flag], 1;
// T2: r0 = ld.relaxed.sys [flag]; fence.sys; r1 = ld [data];
// 若 r0==1 则 r1==1（经典生产者-消费者模式）

// Litmus Test：CoWR（虚拟别名）
// st [alias_1], 1; fence.proxy.alias; r1 = ld [alias_2];
// r1 == 1（别名需要 alias proxy fence）

// Litmus Test：存储缓冲（SB）
// T1: st [x], 1; fence.sc.sys; r0 = ld [y]
// T2: st [y], 1; fence.sc.sys; r1 = ld [x]
// 保证：r0==1 OR r1==1（fence.sc 保证全局顺序一致性）
// 若改为 fence.acq_rel，则 r0==0 AND r1==0 是允许的
```

---

## 8.11 特殊情况

### 8.11.1 Reduction 不构成 Acquire 模式

原子 reduction（`red`）不与 acquire fence 构成 acquire 模式。

```
// Litmus Test：使用 red 的消息传递
// T1: st [x], 42; st.release.gpu [flag], 1;
// T2: red.sys.global.add [flag], 1; fence.acquire.gpu; r1 = ld.weak [x];
// 可能观察到：r1==0 AND flag==2
// （T1 的 release 模式不 synchronize T2 的 acquire 模式）
// 将 red 改为 atom 则禁止此结果
```

---

**本章小结**：第 8 章定义了 PTX 的正式内存模型（适用于 sm_70+），核心概念链为：
**作用域 → morally strong → release/acquire 模式 → synchronizes-with → 因果序 → 通信序 → 公理约束**。
实际编程中最重要的模式是**消息传递（MP）**：写数据 → `fence.release` → 写 flag → 读 flag → `fence.acquire` → 读数据。
