# PTX ISA 9.2 — 第一章：简介

> 原文来源：PTX ISA, Release 9.2，第 21–24 页

---

## 第一章 简介

本文档介绍 PTX（Parallel Thread Execution，并行线程执行），一种低层级的并行线程执行虚拟机与指令集架构（ISA，Instruction Set Architecture）。PTX 将 GPU 作为一种数据并行计算设备对外暴露。

---

## 1.1 使用 GPU 实现可扩展的数据并行计算

受到市场对实时高清 3D 图形的强烈需求驱动，可编程 GPU 已发展为一种高度并行、多线程、众核处理器，具备强大的计算能力和极高的内存带宽。GPU 尤其适合解决那些可以表达为**数据并行计算**（data-parallel computation）的问题——即同一程序对大量数据元素并行执行——以及具备**高算术强度**（high arithmetic intensity，即算术操作与内存操作之比）的场景。

由于每个数据元素执行相同的程序，对复杂流程控制的需求较低；又由于程序在大量数据元素上执行且具备高算术强度，内存访问延迟可以通过计算来掩盖，而不需要依赖大容量数据缓存。

数据并行处理将数据元素映射到并行处理线程。许多处理大型数据集的应用可以利用数据并行编程模型来加速计算。在 3D 渲染中，大量像素和顶点被映射到并行线程上。类似地，图像与媒体处理应用（如渲染后图像的后处理、视频编解码、图像缩放、立体视觉、模式识别等）也可以将图像块和像素映射到并行处理线程。事实上，许多图像渲染与处理领域之外的算法也从数据并行处理中获益，涵盖通用信号处理、物理仿真、计算金融乃至计算生物学。

**PTX** 为通用并行线程执行定义了一套虚拟机与 ISA。PTX 程序在安装时被翻译为目标硬件指令集。PTX-to-GPU 翻译器和驱动程序使 NVIDIA GPU 可作为可编程的并行计算机使用。

---

## 1.2 PTX 的设计目标

**PTX** 为通用并行编程提供了稳定的编程模型和指令集。它被设计为在支持 NVIDIA Tesla 架构所定义计算特性的 NVIDIA GPU 上高效运行。CUDA、C/C++ 等高级语言的编译器会生成 PTX 指令，这些指令经过优化后被翻译为原生目标架构指令。

PTX 的设计目标包括：

- 提供跨越多代 GPU 的稳定 ISA。
- 使编译应用的性能与原生 GPU 性能相当。
- 为 C/C++ 及其他编译器提供与机器无关的 ISA 目标。
- 为应用与中间件开发者提供代码分发用的 ISA。
- 为优化代码生成器和翻译器提供公共的源码级 ISA，以将 PTX 映射到具体目标机器。
- 方便手工编写库、性能内核和架构测试代码。
- 提供可扩展的编程模型，覆盖从单一单元到大规模并行单元的各种 GPU 规模。

---

## 1.3 PTX ISA 9.2 版本新特性

PTX ISA 9.2 版本引入了以下新特性：

- 为 `add`、`sub`、`min`、`max`、`neg` 指令新增 `.u8x4` 和 `.s8x4` 指令类型支持。
- 新增对 `add.sat.{u16x2/s16x2/u32}` 指令的支持。
- 为 `st.async` 指令新增 `.b128` 类型支持。
- 为 `cp.async.bulk` 指令新增 `.ignore_oob` 限定符支持。
- 为 `cvt` 指令新增 `.bf16x2` 目标类型支持，源类型包括 `.e4m3x2`、`.e5m2x2`、`.e3m2x2`、`.e2m3x2`、`.e2m1x2`。

---

## 1.4 文档结构

本文档内容按以下章节组织：

- **编程模型（Programming Model）**：概述编程模型。
- **PTX 机器模型（PTX Machine Model）**：概述 PTX 虚拟机模型。
- **语法（Syntax）**：描述 PTX 语言的基本语法。
- **状态空间、类型与变量（State Spaces, Types, and Variables）**：描述状态空间、类型以及变量声明。
- **指令操作数（Instruction Operands）**：描述指令操作数。
- **抽象 ABI（Abstracting the ABI）**：描述函数与调用语法、调用约定，以及 PTX 对抽象应用二进制接口（ABI，Application Binary Interface）的支持。
- **指令集（Instruction Set）**：描述指令集。
- **特殊寄存器（Special Registers）**：列出特殊寄存器。
- **指令伪操作（Directives）**：列出 PTX 支持的汇编伪操作。
- **发行说明（Release Notes）**：提供 PTX ISA 2.x 及后续版本的发行说明。

---

## 参考资料

- **754-2008 IEEE 浮点运算标准**（IEEE Standard for Floating-Point Arithmetic）。ISBN 978-0-7381-5752-8，2008 年。http://ieeexplore.ieee.org/servlet/opac?punumber=4610933
- **OpenCL 规范**，版本 1.1，文档修订版 44，2011 年 6 月 1 日。http://www.khronos.org/registry/cl/specs/opencl-1.1.pdf
- **CUDA 编程指南（CUDA Programming Guide）**。https://docs.nvidia.com/cuda/cuda-programming-guide/index.html
- **CUDA 动态并行编程指南（CUDA Dynamic Parallelism Programming Guide）**。https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/dynamic-parallelism.html
- **CUDA 原子性要求（CUDA Atomicity Requirements）**。https://nvidia.github.io/cccl/libcudacxx/extended_api/memory_model.html#atomicity
- **PTX 互操作性编写指南（PTX Writers Guide to Interoperability）**。https://docs.nvidia.com/cuda/ptx-writers-guide-to-interoperability/index.html
