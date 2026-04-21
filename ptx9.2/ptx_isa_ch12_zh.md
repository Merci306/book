# 第十二章：`.pragma` 字符串说明

> 来源：PTX ISA 9.2 — Chapter 12: Descriptions of .pragma Strings（第 861–866 页）

本章说明由 `ptxas`（PTX 汇编器/优化编译器）定义的各个 `.pragma` 指令字符串的语义与用法。

---

## 12.1 Pragma 字符串：`"nounroll"`

### 功能

禁止优化后端编译器对循环进行展开（loop unrolling）。

### 语法

```ptx
.pragma "nounroll";
```

### 说明

`"nounroll"` pragma 是一条指令，用于禁止优化后端编译器展开循环。该 pragma 可以出现在以下三个作用域：

| 作用域 | 效果 |
|--------|------|
| **模块作用域（module scope）** | 禁止模块中所有循环的展开，包括出现在 `.pragma` 之前的循环。 |
| **入口函数作用域（entry-function scope）** | 禁止该入口函数体中所有循环的展开。 |
| **语句级（statement level）** | 仅禁止以当前基本块为循环头（loop header）的那个循环的展开。 |

> **注意：** 若要在语句级起效，`"nounroll"` 指令必须出现在目标循环头基本块中、任何指令语句之前。  
> 循环头基本块定义为：支配循环体中所有基本块，且为循环回边（loop backedge）目标的基本块。  
> 出现在非循环头基本块中的语句级 `"nounroll"` 将被静默忽略。

### PTX ISA 版本说明

- 在 PTX ISA 2.0 中引入。

### 目标架构说明

- 需要 `sm_20` 或更高架构；对 `sm_1x` 目标将被忽略。

### 示例

```ptx
.entry foo (...)
.pragma "nounroll";   // 禁止此函数中的所有循环展开
{
    ...
}

.func bar (...)
{
    ...
L1_head:
    .pragma "nounroll";   // 禁止此循环展开
    ...
    @p bra L1_end;
L1_body:
    ...
L1_continue:
    bra L1_head;
L1_end:
    ...
}
```

---

## 12.2 Pragma 字符串：`"used_bytes_mask"`

### 功能

通过掩码（mask）指明一次加载（`ld`）操作中实际被使用的字节。

### 语法

```ptx
.pragma "used_bytes_mask mask";
```

其中 `mask` 为一个 32 位整数，置位的比特对应于加载数据中实际被使用的字节。

### 说明

`"used_bytes_mask"` pragma 是一条指令，用于告知编译器某次加载操作中哪些字节被实际使用。

- 该 pragma 必须紧接在目标加载指令**之前**声明；若其后紧跟的不是加载指令，则该 pragma 被忽略。
- 若加载指令前没有此 pragma，则默认认为加载的所有字节均被使用。
- `mask` 操作数是一个 32 位整数，每个置位的比特表示加载数据中对应字节被使用。最高有效位（MSB）对应数据的最高有效字节（MSB）。

### 语义示例

```ptx
// 4 字节加载，仅低 3 字节被使用
.pragma "used_bytes_mask 0x7";
ld.global.u32 %r0, [gbl];     // %r0 的高 1 字节未被使用

// 向量加载 16 字节，仅低 12 字节被使用
.pragma "used_bytes_mask 0xfff";
ld.global.v4.u32 {%r0, %r1, %r2, %r3}, [gbl];   // %r3 未被使用
```

### PTX ISA 版本说明

- 在 PTX ISA 8.3 中引入。

### 目标架构说明

- 需要 `sm_50` 或更高架构。

### 示例

```ptx
.pragma "used_bytes_mask 0xfff";
ld.global.v4.u32 {%r0, %r1, %r2, %r3}, [gbl];   // 仅低 12 字节被使用
```

---

## 12.3 Pragma 字符串：`"enable_smem_spilling"`

### 功能

为 CUDA kernel 启用将寄存器溢出（spilling）到共享内存（shared memory）的机制。

### 语法

```ptx
.pragma "enable_smem_spilling";
```

### 说明

`"enable_smem_spilling"` pragma 是一条指令，用于开启寄存器向共享内存的溢出。

溢出过程如下：
1. 寄存器首先溢出到共享内存；
2. 一旦分配的共享内存已满，后续溢出将转向本地内存（local memory）。

由于共享内存的访问延迟低于本地内存，此机制可提升性能。

该 pragma 仅允许在**函数作用域**内使用。

#### 不允许使用此 pragma 的情形（否则可能导致编译错误）：

- 逐函数编译模式（Per-function compilation mode），例如：
  - 分离编译（Separate Compilation）
  - 设备调试（Device-debug）
  - 包含递归函数调用的全程序模式（Whole program with recursive function calls）
  - 可扩展全程序模式（Extensible-whole-program）
- 使用了动态分配共享内存的 kernel
- 使用了 `setmaxnreg` 指令的 kernel

> **注意：** 若未显式指定 launch bounds，编译器将假设最大可能的每 CTA 线程数来估算共享内存分配量及溢出大小。若实际启动时每 CTA 线程数少于估算值，实际每 CTA 共享内存分配量可能超出编译器估算值，从而可能限制 SM 上可同时运行的 CTA 数量，进而导致性能下降。因此，**建议仅在显式指定 launch bounds 时使用此 pragma**。

### PTX ISA 版本说明

- 在 PTX ISA 9.0 中引入。

### 目标架构说明

- 需要 `sm_75` 或更高架构。

### 示例

```ptx
.entry foo (...)
{
    ...
    .pragma "enable_smem_spilling";   // 为此函数启用共享内存溢出
    ...
}
```

---

## 12.4 Pragma 字符串：`"frequency"`

### 功能

为基本块（basic block）的执行指定频率提示。

### 语法

```ptx
.pragma "frequency n";
```

其中 `n` 为一个 64 位非负整数常量，表示执行频率。

### 说明

`"frequency"` pragma 是一条指令，用于告知优化编译器后端某个基本块被一个执行线程执行的次数。编译器将此 pragma 视为**提示**，用于指导优化。

- `n` 为 64 位非负整数常量，指定执行频率。
- 为确保生效，该 pragma 应声明在目标基本块的**开头**。
- 基本块定义为：具有唯一入口点和唯一出口点的连续指令序列。

### PTX ISA 版本说明

- 在 PTX ISA 9.0 中引入。

### 目标架构说明

- 支持所有目标架构。

### 示例

```ptx
.entry foo (...)
{
    .pragma "frequency 32";
    ...
}
```

---

*本文档为 PTX ISA 9.2 第十二章中文翻译，仅供参考。*
