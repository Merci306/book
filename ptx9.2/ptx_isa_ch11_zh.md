# PTX ISA 9.2 — 第十一章：伪指令（Directives）

> 原文来源：*NVIDIA PTX ISA Reference, Release 9.2*，第 819–860 页（第十一章）。

---

## 目录

- [11.1 PTX 模块伪指令](#111-ptx-模块伪指令)
  - [11.1.1 `.version`](#1111-version)
  - [11.1.2 `.target`](#1112-target)
  - [11.1.3 `.address_size`](#1113-address_size)
- [11.2 内核入口点与函数伪指令](#112-内核入口点与函数伪指令)
  - [11.2.1 `.entry`](#1121-entry)
  - [11.2.2 `.func`](#1122-func)
  - [11.2.3 `.alias`](#1123-alias)
- [11.3 控制流伪指令](#113-控制流伪指令)
  - [11.3.1 `.branchtargets`](#1131-branchtargets)
  - [11.3.2 `.calltargets`](#1132-calltargets)
  - [11.3.3 `.callprototype`](#1133-callprototype)
- [11.4 性能调优伪指令](#114-性能调优伪指令)
  - [11.4.1 `.maxnreg`](#1141-maxnreg)
  - [11.4.2 `.maxntid`](#1142-maxntid)
  - [11.4.3 `.reqntid`](#1143-reqntid)
  - [11.4.4 `.minnctapersm`](#1144-minnctapersm)
  - [11.4.5 `.maxnctapersm`（已弃用）](#1145-maxnctapersm已弃用)
  - [11.4.6 `.noreturn`](#1146-noreturn)
  - [11.4.7 `.pragma`](#1147-pragma)
  - [11.4.8 `.abi_preserve`](#1148-abi_preserve)
  - [11.4.9 `.abi_preserve_control`](#1149-abi_preserve_control)
- [11.5 调试伪指令](#115-调试伪指令)
  - [11.5.1 `@@dwarf`](#1151-dwarf)
  - [11.5.2 `.section`](#1152-section)
  - [11.5.3 `.file`](#1153-file)
  - [11.5.4 `.loc`](#1154-loc)
- [11.6 链接伪指令](#116-链接伪指令)
  - [11.6.1 `.extern`](#1161-extern)
  - [11.6.2 `.visible`](#1162-visible)
  - [11.6.3 `.weak`](#1163-weak)
  - [11.6.4 `.common`](#1164-common)
- [11.7 集群维度伪指令](#117-集群维度伪指令)
  - [11.7.1 `.reqnctapercluster`](#1171-reqnctapercluster)
  - [11.7.2 `.explicitcluster`](#1172-explicitcluster)
  - [11.7.3 `.maxclusterrank`](#1173-maxclusterrank)
- [11.8 杂项伪指令](#118-杂项伪指令)
  - [11.8.1 `.blocksareclusters`](#1181-blocksareclusters)

---

## 11.1 PTX 模块伪指令

以下伪指令用于声明模块中 PTX ISA 的版本号、代码所针对的目标架构，以及 PTX 模块中地址的大小。

- `.version`
- `.target`
- `.address_size`

---

### 11.1.1 `.version`

**功能**：指定 PTX ISA 版本号。

#### 语法

```
.version major.minor   // major、minor 均为整数
```

#### 描述

指定 PTX 语言的版本号。当 PTX 语言发生不兼容变更（如语法或语义的修改）时，主版本号（`major`）递增；PTX 编译器使用主版本号来确保旧版 PTX 代码的正确执行。当 PTX 增加新功能时，次版本号（`minor`）递增。

#### 语义

表示本模块必须使用支持**等于或更高版本号**的工具进行编译。每个 PTX 模块必须以 `.version` 伪指令开头，模块内其他位置不允许再出现 `.version` 伪指令。

#### PTX ISA 说明

在 PTX ISA version 1.0 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.version 3.1
.version 3.0
.version 2.3
```

---

### 11.1.2 `.target`

**功能**：指定目标架构与平台选项。

#### 语法

```
.target stringlist   // 逗号分隔的目标说明符列表
string = {
    sm_120a, sm_120f, sm_120,   // sm_12x 目标架构
    sm_121a, sm_121f, sm_121,   // sm_12x 目标架构
    sm_110a, sm_110f, sm_110,   // sm_11x 目标架构
    sm_100a, sm_100f, sm_100,   // sm_10x 目标架构
    sm_101a, sm_101f, sm_101,   // sm_10x 目标架构
    sm_103a, sm_103f, sm_103,   // sm_10x 目标架构
    sm_90a,  sm_90,             // sm_9x 目标架构
    sm_80, sm_86, sm_87, sm_88, sm_89,  // sm_8x 目标架构
    sm_70, sm_72, sm_75,        // sm_7x 目标架构
    sm_60, sm_61, sm_62,        // sm_6x 目标架构
    sm_50, sm_52, sm_53,        // sm_5x 目标架构
    sm_30, sm_32, sm_35, sm_37, // sm_3x 目标架构
    sm_20,                      // sm_2x 目标架构
    sm_10, sm_11, sm_12, sm_13, // sm_1x 目标架构
    texmode_unified, texmode_independent,  // 纹理模式
    debug,                      // 平台选项
    map_f64_to_f32              // 平台选项
}
```

#### 描述

指定当前 PTX 代码所针对目标架构中可用的功能集合。通常，SM 架构各代遵循"洋葱层"模型：每一代在保留前代所有功能的基础上新增功能，因此为特定目标生成的 PTX 代码可以在更高代的设备上运行。

带后缀 `a` 的目标架构（如 `sm_90a`）包含仅在该特定架构上受支持的架构专属功能，**不遵循**洋葱层模型，因此为此类目标生成的 PTX 代码无法在更高代设备上运行。架构专属功能只能与支持这些功能的目标配合使用。

带后缀 `f` 的目标架构（如 `sm_100f`）包含仅在同一架构族内受支持的族专属功能，为此类目标生成的 PTX 代码只能在同族中更高代的设备上运行。族专属功能可与 f 目标以及同族更高代的 a 目标配合使用。

**架构族定义（表 64）：**

| 族 | 包含的目标 SM 架构 |
|---|---|
| sm_10x 族 | sm_100f, sm_103f, 以及 sm_10x 族的未来目标 |
| sm_11x 族 | sm_110f, sm_101f, 以及 sm_11x 族的未来目标 |
| sm_12x 族 | sm_120f, sm_121f, 以及 sm_12x 族的未来目标 |

**各目标架构功能说明：**

| 目标 | 描述 |
|---|---|
| `sm_120` | sm_120 架构的基础功能集 |
| `sm_120f` | 新增对 sm_120f 族专属功能的支持 |
| `sm_120a` | 新增对 sm_120a 架构专属功能的支持 |
| `sm_121` | sm_121 架构的基础功能集 |
| `sm_121f` | 新增对 sm_121f 族专属功能的支持 |
| `sm_121a` | 新增对 sm_121a 架构专属功能的支持 |
| `sm_110` | sm_110 架构的基础功能集 |
| `sm_110f` | 新增对 sm_110f 族专属功能的支持 |
| `sm_110a` | 新增对 sm_110a 架构专属功能的支持 |
| `sm_100` | sm_100 架构的基础功能集 |
| `sm_100f` | 新增对 sm_100f 族专属功能的支持 |
| `sm_100a` | 新增对 sm_100a 架构专属功能的支持 |
| `sm_101` | sm_101 架构基础功能集（已更名为 sm_110） |
| `sm_101f` | sm_101f 族专属功能（已更名为 sm_110f） |
| `sm_101a` | sm_101a 架构专属功能（已更名为 sm_110a） |
| `sm_103` | sm_103 架构的基础功能集 |
| `sm_103f` | 新增对 sm_103f 族专属功能的支持 |
| `sm_103a` | 新增对 sm_103a 架构专属功能的支持 |
| `sm_90` | sm_90 架构的基础功能集 |
| `sm_90a` | 新增对 sm_90a 架构专属功能的支持 |
| `sm_80` | sm_80 架构的基础功能集 |
| `sm_86` | 在 `min` 和 `max` 指令上新增 `.xorsign` 修饰符支持 |
| `sm_87` | sm_87 架构的基础功能集 |
| `sm_88` | sm_88 架构的基础功能集 |
| `sm_89` | sm_89 架构的基础功能集 |
| `sm_70` | sm_70 架构的基础功能集 |
| `sm_72` | 在 `wmma` 指令中新增整数乘法数和累加器矩阵支持；新增 `cvt.pack` 指令支持 |
| `sm_75` | 在 `wmma` 指令中新增次字节整数和单比特乘法数矩阵支持；新增 `ldmatrix`、`movmatrix`、`tanh` 指令支持 |
| `sm_60` | sm_60 架构的基础功能集 |
| `sm_61` | 新增 `dp2a` 和 `dp4a` 指令支持 |
| `sm_62` | sm_61 架构的基础功能集 |
| `sm_50` | sm_50 架构的基础功能集 |
| `sm_52` | sm_50 架构的基础功能集 |
| `sm_53` | 新增 `.f16` 和 `.f16x2` 类型的算术、比较和纹理指令支持 |
| `sm_30` | sm_30 架构的基础功能集 |
| `sm_32` | 新增 64 位 `{atom,red}.{and,or,xor,min,max}` 指令、`shf` 指令、`ld.global.nc` 指令 |
| `sm_35` | 新增 CUDA 动态并行（Dynamic Parallelism）支持 |
| `sm_37` | sm_35 架构的基础功能集 |
| `sm_20` | sm_20 架构的基础功能集 |
| `sm_10` | sm_10 架构基础功能集。若使用任何 `.f64` 指令，则需要 `map_f64_to_f32` |
| `sm_11` | 新增 64 位 `{atom,red}.{and,or,xor,min,max}` 指令。若使用 `.f64` 指令，需要 `map_f64_to_f32` |
| `sm_12` | 新增 `{atom,red}.shared`、64 位 `{atom,red}.global`、`vote` 指令。若使用 `.f64` 指令，需要 `map_f64_to_f32` |
| `sm_13` | 新增双精度支持，包括扩展的舍入修饰符。禁止使用 `map_f64_to_f32` |

#### 语义

每个 PTX 模块必须以 `.version` 伪指令开头，紧随其后是包含目标架构及可选平台选项的 `.target` 伪指令。`.target` 伪指令每次只能指定一个目标架构，但后续的 `.target` 伪指令可用于更改解析期间允许的目标功能集合。包含多个 `.target` 伪指令的程序，只能在支持程序中所列最高编号架构全部功能的设备上编译和运行。PTX 功能会依据指定的目标架构进行检查，若使用了不受支持的功能则会报错。

纹理模式针对整个模块指定，不能在模块内部更改。

`.target debug` 选项声明 PTX 文件包含 DWARF 调试信息，后续 PTX 编译将保留源码级调试所需的映射信息。若声明了 `debug` 选项但文件中未找到 DWARF 信息，则会生成错误消息。`debug` 选项需要 PTX ISA version 3.0 或更高版本。

`map_f64_to_f32` 表示所有双精度指令都映射到单精度，与目标架构无关。这使高级语言编译器可将含有 `double` 类型的程序编译到不支持双精度操作的目标设备。注意 `.f64` 存储仍占 64 位，但从 `.f64` 转换为 `.f32` 的指令只使用其中的一半。

#### 注意

`compute_xx` 形式的目标也可作为 `sm_xx` 目标的同义词。`sm_{101,101f,101a}` 从 PTX ISA version 9.0 起已更名为 `sm_{110,110f,110a}`。

#### PTX ISA 说明

在 PTX ISA version 1.0 中引入。各目标字符串的引入版本：
- `sm_10`、`sm_11`：version 1.0
- `sm_12`、`sm_13`：version 1.2
- 纹理模式：version 1.5
- `sm_20`：version 2.0
- `sm_30`：version 3.0；`debug` 平台选项：version 3.0
- `sm_35`：version 3.1
- `sm_32`、`sm_50`：version 4.0
- `sm_37`、`sm_52`：version 4.1
- `sm_53`：version 4.2
- `sm_60`、`sm_61`、`sm_62`：version 5.0
- `sm_70`：version 6.0；`sm_72`：version 6.1；`sm_75`：version 6.3
- `sm_80`：version 7.0；`sm_86`：version 7.1；`sm_87`：version 7.4
- `sm_88`：version 9.0；`sm_89`：version 7.8；`sm_90`：version 7.8
- `sm_90a`：version 8.0
- `sm_100`、`sm_100a`、`sm_101`、`sm_101a`：version 8.6
- `sm_100f`、`sm_101f`、`sm_103`、`sm_103f`、`sm_103a`：version 8.8
- `sm_110`、`sm_110f`、`sm_110a`：version 9.0
- `sm_120`、`sm_120a`：version 8.7；`sm_120f`：version 8.8
- `sm_121`、`sm_121f`、`sm_121a`：version 8.8

#### 目标 ISA 说明

所有目标架构均支持 `.target` 伪指令。

#### 示例

```ptx
.target sm_10              // 基础目标架构
.target sm_13              // 支持双精度
.target sm_20, texmode_independent
.target sm_90              // 基础目标架构
.target sm_90a             // 使用架构专属功能的 PTX
.target sm_100f            // 使用族专属功能的 PTX
```

---

### 11.1.3 `.address_size`

**功能**：指定整个 PTX 模块中使用的地址大小。

#### 语法

```
.address_size address-size
address-size = { 32, 64 };
```

#### 描述

指定 PTX 代码及 PTX 中二进制 DWARF 信息所假定的地址大小（作用于整个模块）。不允许在同一模块内重复定义此伪指令。在独立编译（separate compilation）的情况下，所有模块必须指定（或默认为）相同的地址大小。

`.address_size` 伪指令是可选的，但若存在，则必须紧随 `.target` 伪指令之后。

#### 语义

若省略 `.address_size` 伪指令，地址大小默认为 32。

#### PTX ISA 说明

在 PTX ISA version 2.3 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
// 示例伪指令
.address_size 32   // 地址为 32 位
.address_size 64   // 地址为 64 位

// 在模块中的放置位置示例
.version 2.3
.target  sm_20
.address_size 64
...
.entry foo () { ... }
```

---

## 11.2 内核入口点与函数伪指令

以下伪指令用于指定内核入口点和函数：

- `.entry`
- `.func`

---

### 11.2.1 `.entry`

**功能**：带可选参数的内核入口点与函数体。

#### 语法

```
.entry kernel-name ( param-list ) kernel-body
.entry kernel-name kernel-body
```

#### 描述

定义内核函数的入口点名称、参数列表以及函数体。参数通过 `.param` 空间内存传递，列在可选的圆括号参数列表中。可以在内核体中按名称引用参数，并使用 `ld.param{::entry}` 指令将其加载到寄存器中。

除普通参数外，不透明类型的 `.texref`、`.samplerref` 和 `.surfref` 变量也可作为参数传递。这些参数只能在纹理和表面加载、存储及查询指令中按名称引用，不能通过 `ld.param` 指令访问。

执行内核的 CTA 的形状和大小可通过专用寄存器获取。

#### 语义

指定内核程序的入口点。内核启动时，内核维度和属性被确定，并通过专用寄存器（如 `%ntid`、`%nctaid` 等）提供访问。

#### PTX ISA 说明

- PTX ISA version 1.4 及更高版本：参数变量在内核参数列表中声明。
- PTX ISA version 1.0 至 1.3：参数变量在内核体中声明。

普通（非不透明类型）参数的最大内存大小 PTX 支持为 32764 字节，具体限制随 PTX ISA 版本而变化：

| PTX ISA 版本 | 最大参数大小（字节） |
|---|---|
| version 8.1 及以上 | 32764 |
| version 1.5 及以上 | 4352 |
| version 1.4 及以上 | 256 |

CUDA 和 OpenCL 驱动支持的参数内存限制：

| 驱动 | 参数内存大小 |
|---|---|
| CUDA | sm_1x 为 256 字节；sm_2x 及以上为 4096 字节；sm_70 及以上为 32764 字节 |
| OpenCL | sm_70 及以上为 32764 字节；sm_6x 及以下为 4352 字节 |

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.entry cta_fft

.entry filter ( .param .b32 x, .param .b32 y, .param .b32 z )
{
    .reg .b32 %r<99>;
    ld.param.b32 %r1, [x];
    ld.param.b32 %r2, [y];
    ld.param.b32 %r3, [z];
    ...
}

.entry prefix_sum ( .param .align 4 .s32 pitch[8000] )
{
    .reg .s32 %t;
    ld.param::entry.s32 %t, [pitch];
    ...
}
```

---

### 11.2.2 `.func`

**功能**：函数定义。

#### 语法

```
.func {.attribute(attr-list)} fname {.noreturn} {.abi_preserve N} {.abi_preserve_control N} function-body
.func {.attribute(attr-list)} fname (param-list) {.noreturn} {.abi_preserve N} {.abi_preserve_control N} function-body
.func {.attribute(attr-list)} (ret-param) fname (param-list) {.abi_preserve N} {.abi_preserve_control N} function-body
```

#### 描述

定义函数，包括输入和返回参数及可选的函数体。

- 可选的 `.noreturn` 伪指令表示函数不会返回到调用方函数。`.noreturn` 不能用于有返回参数的函数。详见 §11.4.6。
- 可选的 `.attribute` 伪指令指定与函数关联的附加信息。
- 可选的 `.abi_preserve` 和 `.abi_preserve_control` 伪指令用于指定通用寄存器和控制寄存器的数量。详见 §11.4.8 和 §11.4.9。

函数的任何伪指令（如 `.noreturn`）在函数声明和定义之间必须保持一致。

不带函数体的 `.func` 定义提供函数原型。参数列表定义函数体中局部范围的变量。参数必须是寄存器或参数状态空间中的基本类型。寄存器状态空间的参数可在函数体指令中直接引用；`.param` 空间的参数通过 `ld.param{::func}` 和 `st.param{::func}` 指令访问。参数传递采用**按值传递**方式。

参数列表中最后一个参数可以是不指定大小的 `.b8` 类型 `.param` 数组，用于将任意数量的参数打包成单个数组对象传递给函数。

#### 语义

PTX 语法隐藏了底层调用约定和 ABI 的所有细节。参数传递的具体实现由优化转换器决定，可能使用寄存器和栈位置的组合。

#### 发布说明

- PTX ISA version 1.x：参数必须在寄存器状态空间，无栈，禁止递归。
- PTX ISA version 2.0 及更高，目标 sm_20 或以上：允许在 `.param` 状态空间使用参数，实现带栈的 ABI，并支持递归，最多支持一个返回值。

#### PTX ISA 说明

- 在 version 1.0 中引入
- 不定大小数组参数：version 6.0
- `.noreturn` 伪指令：version 6.4
- `.attribute` 伪指令：version 8.0
- `.abi_preserve` 和 `.abi_preserve_control`：version 9.0

#### 目标 ISA 说明

- 无不定大小数组参数的函数：所有目标架构均支持
- 不定大小数组参数：需要 sm_30 或更高
- `.noreturn`：需要 sm_30 或更高
- `.attribute`：需要 sm_90 或更高
- `.abi_preserve` 和 `.abi_preserve_control`：需要 sm_80 或更高

#### 示例

```ptx
// 带返回值的函数
.func (.reg .b32 rval) foo (.reg .b32 N, .reg .f64 dbl)
{
    .reg .b32 localVar;
    ... use N, dbl; other code;
    mov.b32 rval, result;
    ret;
}
...
call (fooval), foo, (val0, val1);   // 返回值在 fooval 中

// 带 .noreturn 的函数
.func foo (.reg .b32 N, .reg .f64 dbl) .noreturn
{
    .reg .b32 localVar;
    ... use N, dbl; other code;
    mov.b32 rval, result;
    ret;
}
...
call foo, (val0, val1);

// 带不定大小数组参数的函数
.func (.param .u32 rval) bar (.param .u32 N, .param .align 4 .b8 numbers[])
{
    .reg .b32 input0, input1;
    ld.param.b32 input0, [numbers + 0];
    ld.param.b32 input1, [numbers + 4];
    ... other code;
    ret;
}
...
.param .u32 N;
.param .align 4 .b8 numbers[8];
st.param.u32 [N], 2;
st.param.b32 [numbers + 0], 5;
st.param.b32 [numbers + 4], 10;
call (rval), bar, (N, numbers);
```

---

### 11.2.3 `.alias`

**功能**：为现有函数符号定义别名。

#### 语法

```
.alias fAlias, fAliasee;
```

#### 描述

`.alias` 是模块级伪指令，定义标识符 `fAlias` 为 `fAliasee` 所指定函数的别名。`fAlias` 和 `fAliasee` 均为非入口函数符号。

- `fAlias` 是不带函数体的函数声明。
- `fAliasee` 是必须在与 `.alias` 声明同一模块中定义的函数符号，且不能具有 `.weak` 链接。
- `fAlias` 与 `fAliasee` 的函数原型必须匹配。
- 程序可以使用 `fAlias` 或 `fAliasee` 标识符来引用由 `fAliasee` 定义的函数。

#### PTX ISA 说明

`.alias` 伪指令在 PTX ISA version 6.3 中引入。

#### 目标 ISA 说明

需要 sm_30 或更高。

#### 示例

```ptx
.visible .func foo (.param .u32 p) { ... }
.visible .func bar (.param .u32 p);
.alias bar, foo;

.entry test() {
    .param .u32 p;
    ...
    call foo, (p);   // 直接调用 foo
    ...
    .param .u32 p;
    call bar, (p);   // 通过别名调用 foo
}
```

---

## 11.3 控制流伪指令

PTX 提供用于指定 `brx.idx` 和 `call` 指令潜在目标的伪指令：

- `.branchtargets`
- `.calltargets`
- `.callprototype`

---

### 11.3.1 `.branchtargets`

**功能**：声明潜在分支目标列表。

#### 语法

```
Label: .branchtargets list-of-labels ;
```

#### 描述

为后续 `brx.idx` 指令声明潜在分支目标列表，并将该列表与行首的标签关联。列表中所有控制流标签必须出现在与声明相同的函数中。标签列表可使用紧凑的简写语法来枚举具有公共前缀的一组标签，类似于参数化变量名的语法。

#### PTX ISA 说明

在 PTX ISA version 2.1 中引入。

#### 目标 ISA 说明

需要 sm_20 或更高。

#### 示例

```ptx
.function foo () {
    .reg .u32 %r0;
    ...
L1: ...
L2: ...
L3: ...
ts: .branchtargets L1, L2, L3;
@p  brx.idx %r0, ts;
...

.function bar() {
    .reg .u32 %r0;
    ...
N0: ... N1: ... N2: ... N3: ... N4: ...
ts: .branchtargets N<5>;
@p  brx.idx %r0, ts;
```

---

### 11.3.2 `.calltargets`

**功能**：声明潜在调用目标列表。

#### 语法

```
Label: .calltargets list-of-functions ;
```

#### 描述

为后续间接调用声明潜在调用目标列表，并将该列表与行首的标签关联。列表中命名的所有函数必须在 `.calltargets` 伪指令之前声明，且所有函数必须具有相同的类型签名。

#### PTX ISA 说明

在 PTX ISA version 2.1 中引入。

#### 目标 ISA 说明

需要 sm_20 或更高。

#### 示例

```ptx
calltgt: .calltargets fastsin, fastcos;
...
@p call (%f1), %r0, (%x), calltgt;
```

---

### 11.3.3 `.callprototype`

**功能**：声明用于间接调用的函数原型。

#### 语法

```
// 无输入参数或返回参数
label: .callprototype _ .noreturn {.abi_preserve N} {.abi_preserve_control N};

// 有输入参数，无返回参数
label: .callprototype _ (param-list) .noreturn {.abi_preserve N} {.abi_preserve_control N};

// 无输入参数，有返回参数
label: .callprototype (ret-param) _ {.abi_preserve N} {.abi_preserve_control N};

// 有输入参数和返回参数
label: .callprototype (ret-param) _ (param-list) {.abi_preserve N} {.abi_preserve_control N};
```

#### 描述

定义无特定函数名的原型，并将其与标签关联。该原型可用于对可能调用目标不完全已知的间接调用指令。参数可以是寄存器或参数状态空间中的基本类型，或参数状态空间中的数组类型。可使用占位符符号 `_` 来避免命名虚拟参数名称。

可选的 `.noreturn` 指明函数不会返回到调用方（不能与有返回参数的函数一起使用）。可选的 `.abi_preserve` 和 `.abi_preserve_control` 用于指定通用寄存器和控制寄存器数量。

#### PTX ISA 说明

- 在 version 2.1 中引入
- `.noreturn`：version 6.4
- `.abi_preserve` 和 `.abi_preserve_control`：version 9.0

#### 目标 ISA 说明

- 需要 sm_20 或更高
- `.noreturn`：需要 sm_30 或更高
- `.abi_preserve` 和 `.abi_preserve_control`：需要 sm_80 或更高

#### 示例

```ptx
Fproto1: .callprototype _ ;
Fproto2: .callprototype _ (.param .f32 _);
Fproto3: .callprototype (.param .u32 _) _ ;
Fproto4: .callprototype (.param .u32 _) _ (.param .f32 _);
...
@p call (%val), %r0, (%f1), Fproto4;

// 数组参数示例
Fproto5: .callprototype _ (.param .b8 _[12]);

// .noreturn 示例
Fproto6: .callprototype _ (.param .f32 _) .noreturn;
...
@p call %r0, (%f1), Fproto6;

// .abi_preserve 示例
Fproto7: .callprototype _ (.param .b32 _) .abi_preserve 10;
...
@p call %r0, (%r1), Fproto7;
```

---

## 11.4 性能调优伪指令

PTX 支持以下伪指令，将信息传递给优化后端编译器以实现低级性能调优：

- `.maxnreg`
- `.maxntid`
- `.reqntid`
- `.minnctapersm`
- `.maxnctapersm`（已弃用）
- `.pragma`
- `.abi_preserve`
- `.abi_preserve_control`

- `.maxnreg`：指定分配给单个线程的最大寄存器数量。
- `.maxntid`：指定线程块（CTA）中的最大线程数。
- `.reqntid`：指定线程块（CTA）中的精确线程数（必须满足）。
- `.minnctapersm`：指定映射到单个多处理器（SM）上的最小线程块数量。

这些伪指令可用于限制资源需求（如寄存器数），从而提高总线程数并提供更多隐藏内存延迟的机会。`.minnctapersm` 可与 `.maxntid` 或 `.reqntid` 一起使用，在不直接指定最大寄存器数的情况下权衡每线程寄存器数与多处理器利用率。

设备函数伪指令 `.abi_preserve` 和 `.abi_preserve_control` 指定函数必须为其调用者保留的数据寄存器和控制寄存器数量（即调用函数时存储在被调用者保存寄存器中的调用者活跃数据和控制变量数量）。

目前，`.maxnreg`、`.maxntid`、`.reqntid` 和 `.minnctapersm` 可按入口点应用，必须出现在 `.entry` 伪指令和函数体之间。这些伪指令优先于传递给优化后端的任何模块级约束。若约束不一致或无法满足则会生成警告。

`.pragma` 伪指令可出现在模块（文件）作用域、入口点作用域或内核/设备函数体的语句级别。

---

### 11.4.1 `.maxnreg`

**功能**：指定每个线程可分配的最大寄存器数量。

#### 语法

```
.maxnreg n
```

#### 描述

声明 CTA 中每线程使用的最大寄存器数量。

#### 语义

编译器保证不超过此限制。实际使用的寄存器数可能更少；例如，后端可能能够编译为更少的寄存器，或者最大寄存器数可能进一步受到 `.maxntid` 和 `.maxctapersm` 的约束。

#### PTX ISA 说明

在 PTX ISA version 1.3 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.entry foo .maxnreg 16 { ... }   // 每线程最大寄存器数 = 16
```

---

### 11.4.2 `.maxntid`

**功能**：指定线程块（CTA）中的最大线程数。

#### 语法

```
.maxntid nx
.maxntid nx, ny
.maxntid nx, ny, nz
```

#### 描述

通过给出 1D、2D 或 3D CTA 各维度的最大范围，声明线程块中的最大线程数。最大线程总数是各维度最大范围的乘积。

#### 语义

线程块中线程总数（各维度最大范围的乘积）保证在此伪指令出现的内核的任何调用中不会超过。超过最大线程数会导致运行时错误或内核启动失败。注意，此伪指令保证**总**线程数不超过最大值，但不保证任意单一维度的限制不被超过。

#### PTX ISA 说明

在 PTX ISA version 1.3 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.entry foo .maxntid 256        { ... }   // 最大线程数 = 256
.entry bar .maxntid 16, 16, 4  { ... }   // 最大线程数 = 1024
```

---

### 11.4.3 `.reqntid`

**功能**：指定线程块（CTA）中的精确线程数。

#### 语法

```
.reqntid nx
.reqntid nx, ny
.reqntid nx, ny, nz
```

#### 描述

通过指定 1D、2D 或 3D CTA 各维度的范围来声明线程块中的线程数。线程总数是各维度线程数的乘积。

#### 语义

任何内核调用中每个 CTA 维度的大小必须等于此伪指令中指定的值。以不同的 CTA 维度启动内核将导致运行时错误或内核启动失败。

#### 注意

`.reqntid` 不能与 `.maxntid` 一起使用。

#### PTX ISA 说明

在 PTX ISA version 2.1 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.entry foo .reqntid 256        { ... }   // 线程数 = 256
.entry bar .reqntid 16, 16, 4  { ... }   // 线程数 = 1024
```

---

### 11.4.4 `.minnctapersm`

**功能**：指定每个 SM 的最小 CTA 数量。

#### 语法

```
.minnctapersm ncta
```

#### 描述

声明从内核网格（grid）映射到单个多处理器（SM）的最小 CTA 数量。

#### 注意

基于 `.minnctapersm` 的优化需要同时指定 `.maxntid` 或 `.reqntid`。若 `.minnctapersm` 和 `.maxntid`/`.reqntid` 导致单个 SM 上的总线程数超过 SM 支持的最大线程数，则 `.minnctapersm` 将被忽略。在 PTX ISA version 2.1 或更高版本中，若指定了 `.minnctapersm` 但未指定 `.maxntid` 或 `.reqntid`，则会生成警告。

#### PTX ISA 说明

在 PTX ISA version 2.0 中引入，用于替代 `.maxnctapersm`。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.entry foo
.maxntid      256
.minnctapersm 4
{ ... }
```

---

### 11.4.5 `.maxnctapersm`（已弃用）

**功能**：指定每个 SM 的最大 CTA 数量（已弃用）。

#### 语法

```
.maxnctapersm ncta
```

#### 描述

声明从内核网格映射到单个多处理器（SM）的最大 CTA 数量。

#### 注意

基于 `.maxnctapersm` 的优化通常需要同时指定 `.maxntid`。优化后端编译器使用 `.maxntid` 和 `.maxnctapersm` 来计算每线程寄存器使用量的上限，以便将指定数量的 CTA 映射到单个多处理器。然而，若后端实际使用的寄存器数量远低于此上限，则可能有更多 CTA 映射到单个多处理器。因此，`.maxnctapersm` 在 PTX ISA version 2.0 中更名为 `.minnctapersm`。

#### PTX ISA 说明

在 version 1.3 中引入，在 version 2.0 中弃用。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.entry foo
.maxntid      256
.maxnctapersm 4
{ ... }
```

---

### 11.4.6 `.noreturn`

**功能**：指示函数不会返回到其调用方函数。

#### 语法

```
.noreturn
```

#### 描述

指示函数不会返回到调用方函数。

#### 语义

可选的 `.noreturn` 伪指令只能用于设备函数，且必须出现在 `.func` 伪指令和函数体之间。不能用于有返回参数的函数。若带有 `.noreturn` 的函数在运行时实际返回到调用方，则行为未定义。

#### PTX ISA 说明

在 PTX ISA version 6.4 中引入。

#### 目标 ISA 说明

需要 sm_30 或更高。

#### 示例

```ptx
.func foo .noreturn { ... }
```

---

### 11.4.7 `.pragma`

**功能**：向 PTX 后端编译器传递指令字符串。

#### 语法

```
.pragma list-of-strings ;
```

#### 描述

向 PTX 后端编译器传递模块级、入口点级或语句级指令。`.pragma` 伪指令可出现在模块作用域、入口点作用域或语句级别。

#### 语义

`.pragma` 字符串的解释取决于具体实现，不影响 PTX 语义。有关 `ptxas` 中定义的 pragma 字符串描述，请参阅相关文档。

#### PTX ISA 说明

在 PTX ISA version 2.0 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.pragma "nounroll";                    // 在后端禁用循环展开

// 为当前内核禁用循环展开
.entry foo .pragma "nounroll"; { ... }
```

---

### 11.4.8 `.abi_preserve`

**功能**：指定此函数的调用者应保留的通用寄存器数量。

#### 语法

```
.abi_preserve N
```

#### 描述

这是一个与架构无关的值，指定实际的通用寄存器数量。ABI 内部将某些通用寄存器定义为被保留（被调用者保存）寄存器。整数 N 指定函数需要保留的实际通用寄存器数量。

`.abi_preserve` 只能用于设备函数，且必须出现在 `.func` 伪指令和函数体之间。

#### 语义

指定此伪指令后，编译器后端修改底层 ABI 组件，以确保此函数调用者中存储在被调用者保存寄存器中的活跃数据变量数量少于指定值。

#### PTX ISA 说明

在 PTX ISA version 9.0 中引入。

#### 目标 ISA 说明

需要 sm_80 或更高。

#### 示例

```ptx
.func bar() .abi_preserve 8

// 通过调用原型进行间接调用
.func (.param .b32 out[30]) foo (.param .b32 in[30]) .abi_preserve 10 { ... }
...
mov.b64 lpfoo, foo;
prot: .callprototype (.param .b32 out[30]) _ (.param .b32 in[30]) .abi_preserve 10;
call (out), lpfoo, (in), prot;
```

---

### 11.4.9 `.abi_preserve_control`

**功能**：指定此函数的调用者应保留的控制寄存器数量。

#### 语法

```
.abi_preserve_control N
```

#### 描述

这是一个与架构无关的值，指定在导致当前函数调用的调用树中发生的发散程序点数量。ABI 内部将某些控制寄存器定义为被保留（被调用者保存）寄存器。整数 N 指定函数需要保留的实际控制寄存器数量。

`.abi_preserve_control` 只能用于设备函数，且必须出现在 `.func` 伪指令和函数体之间。

#### 语义

指定此伪指令后，编译器后端修改底层 ABI 组件，以确保此函数调用者中存储在被调用者保存控制寄存器中的活跃控制变量数量少于指定值。

#### PTX ISA 说明

在 PTX ISA version 9.0 中引入。

#### 目标 ISA 说明

需要 sm_80 或更高。

#### 示例

```ptx
.func foo() .abi_preserve_control 14

// 通过调用原型进行间接调用
.func (.param .b32 out[30]) bar (.param .b32 in[30]) .abi_preserve_control 10 { ... }
...
mov.b64 lpbar, bar;
prot: .callprototype (.param .b32 out[30]) _ (.param .b32 in[30]) .abi_preserve_control 10;
call (out), lpbar, (in), prot;
```

---

## 11.5 调试伪指令

DWARF 格式的调试信息通过以下伪指令传入 PTX 模块：

- `@@DWARF`
- `.section`
- `.file`
- `.loc`

`.section` 伪指令在 PTX ISA version 2.0 中引入，取代了 `@@DWARF` 语法。`@@DWARF` 语法在 PTX ISA version 2.0 中弃用，但为兼容旧版 PTX ISA version 1.x 代码仍予以支持。

从 PTX ISA version 3.0 起，包含 DWARF 调试信息的 PTX 文件应包含 `.target debug` 平台选项，该前向声明指示 PTX 编译保留源码级调试所需的映射信息。

---

### 11.5.1 `@@dwarf`

**功能**：DWARF 格式信息（旧版语法）。

#### 语法

```
@@DWARF dwarf-string
```

`dwarf-string` 可为以下形式之一：

```
.byte  byte-list           // 逗号分隔的十六进制字节值
.4byte int32-list          // 逗号分隔的十六进制整数，范围 [0..2^32-1]
.quad  int64-list          // 逗号分隔的十六进制整数，范围 [0..2^64-1]
.4byte label
.quad  label
```

#### PTX ISA 说明

在 PTX ISA version 1.2 中引入。自 PTX ISA version 2.0 起弃用，由 `.section` 伪指令替代。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
@@DWARF .section .debug_pubnames, "", @progbits
@@DWARF .byte 0x2b, 0x00, 0x00, 0x00, 0x02, 0x00
@@DWARF .4byte .debug_info
@@DWARF .4byte 0x000006b5, 0x00000364, 0x61395a5f, 0x5f736f63
@@DWARF .4byte 0x6e69616d, 0x63613031, 0x6150736f, 0x736d6172
@@DWARF .byte  0x00, 0x00, 0x00, 0x00, 0x00
```

---

### 11.5.2 `.section`

**功能**：PTX 节（section）定义。

#### 语法

```
.section section_name {
    dwarf-lines
}
```

`dwarf-lines` 支持以下格式：

```
.b8  byte-list                  // 逗号分隔的整数，范围 [-128..255]
.b16 int16-list                 // 逗号分隔的整数，范围 [-2^15..2^16-1]
.b32 int32-list                 // 逗号分隔的整数，范围 [-2^31..2^32-1]
label:                          // 在 debug 节内定义标签
.b64 int64-list                 // 逗号分隔的整数，范围 [-2^63..2^64-1]
.b32 label
.b64 label
.b32 label+imm                  // 标签地址加常量整数字节偏移（有符号，32 位）
.b64 label+imm                  // 标签地址加常量整数字节偏移（有符号，64 位）
.b32 label1-label2              // 同一 dwarf 节中两标签地址之差（32 位）
.b64 label3-label4              // 同一 dwarf 节中两标签地址之差（64 位）
```

#### PTX ISA 说明

- 在 version 2.0 中引入，取代 `@@DWARF` 语法
- `label+imm` 表达式：version 3.2
- `dwarf-lines` 中的 `.b16` 整数支持：version 6.0
- 支持在 DWARF 节内定义标签：version 7.2
- `label1-label2` 表达式：version 7.5
- dwarf-lines 中的负数：version 7.5

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.section .debug_pubnames {
    .b32 LpubNames_end0-LpubNames_begin0
LpubNames_begin0:
    .b8  0x2b, 0x00, 0x00, 0x00, 0x02, 0x00
    .b32 .debug_info
info_label1:
    .b32 0x000006b5, 0x00000364, 0x61395a5f, 0x5f736f63
    .b32 0x6e69616d, 0x63613031, 0x6150736f, 0x736d6172
    .b8  0x00, 0x00, 0x00, 0x00, 0x00
LpubNames_end0:
}

.section .debug_info {
    .b32 11430
    .b8  2, 0
    .b32 .debug_abbrev
    .b8  8, 1, 108, 103, 101, 110, 102, 101, 58, 32, 69, 68, 71, 32, 52, 46, 49
    .b8  0
    .b32 3, 37, 176, -99
    .b32 info_label1
    .b32 .debug_loc+0x4
    .b8  -11, 11, 112, 97
    .b32 info_label1+12
    .b64 -1
    .b16 -5, -65535
}
```

---

### 11.5.3 `.file`

**功能**：源文件名。

#### 语法

```
.file file_index "filename" {, timestamp, file_size}
```

#### 描述

将源文件名与整数索引关联。`.loc` 伪指令通过索引引用源文件。

`.file` 伪指令允许可选地指定一个无符号数（表示最后修改时间）和一个无符号整数（表示源文件的字节大小）。`timestamp` 和 `file_size` 值可为 0，表示该信息不可用。`timestamp` 值采用 C/C++ `time_t` 数据类型的格式。`file_size` 是无符号 64 位整数。

`.file` 伪指令只允许出现在最外层作用域，即与内核和设备函数声明相同的层级。

#### 语义

若未指定 `timestamp` 和 `file_size`，则默认为 0。

#### PTX ISA 说明

- 在 version 1.0 中引入
- `timestamp` 和 `file_size`：version 3.2

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.file 1 "example.cu"
.file 2 "kernel.cu"
.file 1 "kernel.cu", 1339013327, 64118
```

---

### 11.5.4 `.loc`

**功能**：源文件位置。

#### 语法

```
.loc file_index line_number column_position
.loc file_index line_number column_position, function_name label {+ immediate}, inlined_at file_index2 line_number2 column_position2
```

#### 描述

声明与其后的 PTX 指令关联的源文件位置（源文件、行号和列位置）。`.loc` 引用由 `.file` 伪指令定义的 `file_index`。

为指示由内联函数生成的 PTX 指令，可在 `.loc` 伪指令中指定 `.inlined_at` 属性，用于指定该函数被内联的源位置。`file_index2`、`line_number2` 和 `column_position2` 指定函数被内联的位置。作为 `.inlined_at` 指定的源位置必须在词法上先于 `.loc` 伪指令中的源位置。

`function_name` 属性指定 `.debug_str` 命名的 DWARF 节中的偏移量，以 `label` 表达式或 `label + immediate` 表达式指定（`label` 定义在 `.debug_str` 节中）。`debug_str` 节包含以 ASCII null 结尾的字符串，指定被内联函数的名称。

注意，PTX 指令可能有一个关联的源位置（由最近在词法上先于该指令的 `.loc` 伪指令确定），若之前没有 `.loc` 伪指令则没有关联的源位置。PTX 中的标签继承词法上紧随其后的最近指令的位置；没有后续 PTX 指令的标签没有关联的源位置。

#### PTX ISA 说明

- 在 version 1.0 中引入
- `function_name` 和 `inlined_at` 属性：version 7.2

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.loc 2 4237 0
L1:                           // 继承 mov 的位置：文件 #2 第 4237 行第 0 列
mov.u32 %r1, %r2;             // 文件 #2 第 4237 行第 0 列
add.u32 %r2, %r1, %r3;        // 文件 #2 第 4237 行第 0 列
...
L2:                           // 继承 sub 的位置：文件 #2 第 4239 行第 5 列
.loc 2 4239 5
sub.u32 %r2, %r1, %r3;        // 文件 #2 第 4239 行第 5 列

// 内联函数示例
.loc 1 21 3
.loc 1 9 3, function_name info_string0, inlined_at 1 21 3
ld.global.u32 %r1, [gg];      // 第 9 行的函数，内联于第 21 行
setp.lt.s32   %p1, %r1, 8;
.loc 1 27 3
.loc 1 10 5, function_name info_string1, inlined_at 1 27 3
.loc 1 15 3, function_name .debug_str+16, inlined_at 1 10 5
setp.ne.s32 %p2, %r1, 18;
@%p2 bra BB2_3;

.section .debug_str {
info_string0:
    .b8 95   // _
    .b8 90   // z
    .b8 51   // 3
    .b8 102  // f
    .b8 111  // o
    .b8 111  // o
    .b8 118  // v
    .b8 0
info_string1:
    .b8 95   // _
    .b8 90   // z
    .b8 51   // 3
    .b8 98   // b
    .b8 97   // a
    .b8 114  // r
    .b8 118  // v
    .b8 0
    ...
}
```

---

## 11.6 链接伪指令

- `.extern`
- `.visible`
- `.weak`
- `.common`

---

### 11.6.1 `.extern`

**功能**：外部符号声明。

#### 语法

```
.extern identifier
```

#### 描述

声明 `identifier` 在当前模块外部定义。定义该标识符的模块必须在单个目标文件中将其定义为 `.weak` 或 `.visible`（且仅定义一次）。外部声明可多次出现，对该符号的引用将解析到那个唯一定义。

#### PTX ISA 说明

在 PTX ISA version 1.0 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.extern .global .b32 foo;   // foo 在另一个模块中定义
```

---

### 11.6.2 `.visible`

**功能**：可见（外部可访问）符号声明。

#### 语法

```
.visible identifier
```

#### 描述

声明 `identifier` 全局可见。与 C 语言（标识符默认全局可见，除非声明为 `static`）不同，PTX 标识符仅在当前模块内可见，除非声明为 `.visible`。

#### PTX ISA 说明

在 PTX ISA version 1.0 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.visible .global .b32 foo;   // foo 将对外部可见
```

---

### 11.6.3 `.weak`

**功能**：弱符号声明。

#### 语法

```
.weak identifier
```

#### 描述

声明 `identifier` 全局可见但为**弱符号**。弱符号类似于全局可见符号，但在链接时，弱符号仅在全局可见符号之后才被选择（用于符号解析）。与全局可见符号不同，多个目标文件可以声明同名弱符号；只有在没有同名全局符号时，对该符号的引用才会解析到弱符号。

#### PTX ISA 说明

在 PTX ISA version 3.1 中引入。

#### 目标 ISA 说明

所有目标架构均支持。

#### 示例

```ptx
.weak .func (.reg .b32 val) foo;   // foo 将对外部可见（弱符号）
```

---

### 11.6.4 `.common`

**功能**：公共符号声明。

#### 语法

```
.common identifier
```

#### 描述

声明 `identifier` 全局可见但为**公共符号**。公共符号类似于全局可见符号，但多个目标文件可以声明同名公共符号（且可具有不同类型和大小），对该符号的引用将解析到大小最大的公共符号定义。只有一个目标文件可以初始化公共符号，且该文件中的定义大小必须是所有同名公共符号定义中最大的。

`.common` 链接伪指令只能用于 `.global` 存储的变量，不能用于函数符号或不透明类型的符号。

#### PTX ISA 说明

在 PTX ISA version 5.0 中引入。

#### 目标 ISA 说明

需要 sm_20 或更高。

#### 示例

```ptx
.common .global .u32 gbl;
```

---

## 11.7 集群维度伪指令

以下伪指令指定有关集群（cluster）的信息：

- `.reqnctapercluster`
- `.explicitcluster`
- `.maxclusterrank`

- `.reqnctapercluster`：指定集群中的 CTA 数量。
- `.explicitcluster`：指定内核必须以明确的集群细节启动。
- `.maxclusterrank`：指定集群中 CTA 的最大数量。

集群维度伪指令只能应用于内核函数。

---

### 11.7.1 `.reqnctapercluster`

**功能**：声明集群中的 CTA 数量。

#### 语法

```
.reqnctapercluster nx
.reqnctapercluster nx, ny
.reqnctapercluster nx, ny, nz
```

#### 描述

通过指定 1D、2D 或 3D 集群各维度的范围来设置集群中线程块（CTA）的数量。CTA 总数是各维度 CTA 数量的乘积。对于指定了 `.reqnctapercluster` 伪指令的内核，若在启动时未指定集群维度，运行时将使用此处指定的值进行配置。

#### 语义

若在启动时显式指定了集群维度，则其值必须等于此伪指令中指定的值。以不同的集群维度启动将导致运行时错误或内核启动失败。

#### PTX ISA 说明

在 PTX ISA version 7.8 中引入。

#### 目标 ISA 说明

需要 sm_90 或更高。

#### 示例

```ptx
.entry foo .reqnctapercluster 2     { . . . }
.entry bar .reqnctapercluster 2,2,1 { . . . }
.entry ker .reqnctapercluster 3,2   { . . . }
```

---

### 11.7.2 `.explicitcluster`

**功能**：声明内核必须以明确指定的集群维度启动。

#### 语法

```
.explicitcluster
```

#### 描述

声明此内核应以明确指定的集群维度启动。

#### 语义

带有 `.explicitcluster` 伪指令的内核必须以明确指定的集群维度启动（在启动时或通过 `.reqnctapercluster` 指定），否则程序将以运行时错误或内核启动失败终止。

#### PTX ISA 说明

在 PTX ISA version 7.8 中引入。

#### 目标 ISA 说明

需要 sm_90 或更高。

#### 示例

```ptx
.entry foo .explicitcluster { . . . }
```

---

### 11.7.3 `.maxclusterrank`

**功能**：声明集群中可包含的最大 CTA 数量。

#### 语法

```
.maxclusterrank n
```

#### 描述

声明集群中允许的最大线程块（CTA）数量。

#### 语义

内核任何调用中每个集群各维度 CTA 数量的乘积必须小于或等于此伪指令中指定的值，否则将导致运行时错误或内核启动失败。

`.maxclusterrank` 不能与 `.reqnctapercluster` 一起使用。

#### PTX ISA 说明

在 PTX ISA version 7.8 中引入。

#### 目标 ISA 说明

需要 sm_90 或更高。

#### 示例

```ptx
.entry foo .maxclusterrank 8 { . . . }
```

---

## 11.8 杂项伪指令

PTX 提供以下杂项伪指令：

- `.blocksareclusters`

---

### 11.8.1 `.blocksareclusters`

**功能**：指定 CUDA 线程块被映射到集群。

#### 语法

```
.blocksareclusters
```

#### 描述

CUDA API 的默认行为是通过指定线程块数量和每块线程数来配置网格启动参数。当指定了 `.blocksareclusters` 伪指令时，意味着对应 `.entry` 函数的网格启动配置指定的是**集群数量**（而非线程块数量）。此时，每个集群中的线程块数量由 `.reqnctapercluster` 伪指令指定，线程块大小由 `.reqntid` 伪指令指定。

`.blocksareclusters` 伪指令只允许用于 `.entry` 函数，且需要同时指定 `.reqntid` 和 `.reqnctapercluster` 伪指令。更多详情请参阅《CUDA 编程指南》。

#### PTX ISA 说明

在 PTX ISA version 9.0 中引入。

#### 目标 ISA 说明

需要 sm_90 或更高。

#### 示例

```ptx
.entry foo
.reqntid          32, 32, 1
.reqnctapercluster 32, 32, 1
.blocksareclusters
{ ... }
```

---

*文档结束 — PTX ISA 9.2 第十一章：伪指令（中文版）*
