# 第四章：语法（Syntax）

> 来源：PTX ISA 9.2，第 23–32 页（原文 Pages 35–44）

---

PTX 程序是一组文本源模块（text source modules / 文件）的集合。PTX 源模块具有汇编语言风格的语法，包含指令操作码（instruction operation codes）和操作数（operands）。伪操作（Pseudo-operations）用于指定符号（symbol）和地址管理。`ptxas` 优化后端编译器负责优化并汇编 PTX 源模块，生成对应的二进制目标文件（binary object files）。

---

## 4.1 源文件格式（Source Format）

源模块为 ASCII 文本，行与行之间以换行符 `\n` 分隔。所有空白字符等价，空白仅用于分隔语言中的 token，其他情况均被忽略。

可以使用 C 预处理器 `cpp` 处理 PTX 源模块。以 `#` 开头的行是预处理器指令，常见的预处理器指令包括：

```
#include, #define, #if, #ifdef, #else, #endif, #line, #file
```

Harbison 和 Steele 所著的《C: A Reference Manual》对 C 预处理器有详细描述。

**PTX 区分大小写（case sensitive）**，关键字均为小写。每个 PTX 模块必须以 `.version` 指令开头，指定 PTX 语言版本，随后是 `.target` 指令，指定目标架构。更多信息请参阅 PTX Module Directives（模块指令）相关章节。

---

## 4.2 注释（Comments）

PTX 中的注释遵循 C/C++ 语法：

- 使用非嵌套的 `/*` 和 `*/` 编写可跨多行的注释。
- 使用 `//` 开始注释，直到下一个换行符结束当前行。

注释不能出现在字符常量、字符串字面量或其他注释内部。PTX 中的注释被视为空白（whitespace）。

---

## 4.3 语句（Statements）

PTX 语句（statement）分为**指令（directive）**或**指令（instruction）**两种。语句以可选的标签（label）开头，以分号结尾。

**示例：**

```ptx
.reg   .b32   r1, r2;
.global   .f32   array[N];
start:   mov.b32   r1, %tid.x;
         shl.b32   r1, r1, 2;     // shift thread id by 2 bits
         ld.global.b32   r2, array[r1];   // thread[tid] gets array[tid]
         add.f32   r2, r2, 0.5;   // add 1/2
```

### 4.3.1 指令语句（Directive Statements）

指令关键字以点（`.`）开头，因此与用户自定义标识符不会冲突。PTX 中的指令列于下表（表 1），并在 State Spaces, Types, and Variables（状态空间、类型与变量）及 Directives（指令）章节中详细描述。

**表 1：PTX 指令（PTX Directives）**

```
.address_size   .explicitcluster   .maxnreg         .section
.alias          .extern            .maxntid          .shared
.align          .file              .minnctapersm     .sreg
.branchtargets  .func              .noreturn         .target
.callprototype  .global            .param            .tex
.calltargets    .loc               .pragma           .version
.common         .local             .reg              .visible
.const          .maxclusterrank    .reqnctapercluster .weak
.entry          .maxnctapersm      .reqntid
```

### 4.3.2 指令语句（Instruction Statements）

指令（instruction）由一个指令操作码（opcode）后跟以逗号分隔的零个或多个操作数列表构成，以分号结尾。操作数可以是寄存器变量、常量表达式、地址表达式或标签名。

指令具有可选的**守卫谓词（guard predicate）**，用于控制条件执行。守卫谓词位于可选标签之后、操作码之前，写作 `@p`，其中 `p` 是谓词寄存器（predicate register）。守卫谓词可以取反，写作 `@!p`。目标操作数（destination operand）排在第一位，其后为源操作数（source operands）。

指令关键字列于表 2，所有指令关键字均为 PTX 中的保留 token。

**表 2：保留指令关键字（Reserved Instruction Keywords）**

```
abs           cvta          membar        setp          vabsdiff
activemask    discard       min           shf           vabsdiff2
add           div           mma           shfl          vabsdiff4
addc          dp2a          mov           shl           vadd
alloca        dp4a          movmatrix     shr           vadd2
and           elect         mul           sin           vadd4
applypriority ex2           mul24         slct          vavrg2
atom          exit          multimem      sqrt          vavrg4
bar           fence         nanosleep     st            vmad
barrier       fma           neg           stackrestore  vmax
bfe           fns           not           stacksave     vmax2
bfi           getctarank    or            stmatrix      vmax4
bfind         griddepcontrol pmevent      sub           vmin
bmsk          isspacep      popc          subc          vmin2
bra           istypep       prefetch      suld          vmin4
brev          ld            prefetchu     suq           vote
brkpt         ldmatrix      prmt          sured         vset
brx           ldu           rcp           sust          vset2
call          lg2           red           szext         vset4
clz           lop3          redux         tanh          vshl
cnot          mad           rem           tcgen05       vshr
copysign      mad24         ret           tensormap     vsub
cos           madc          rsqrt         testp         vsub2
clusterlaunchcontrol mapa   sad           tex           vsub4
cp            match         selp          tld4          wgmma
createpolicy  max           set           trap          wmma
cvt           mbarrier      setmaxnreg    txq           xor
```

---

## 4.4 标识符（Identifiers）

用户自定义标识符遵循扩展的 C++ 规则：

- 以字母开头，后跟零个或多个字母、数字、下划线或美元符号字符；
- 或以下划线、美元符号或百分号字符开头，后跟一个或多个字母、数字、下划线或美元符号字符。

```
followsym:    [a-zA-Z0-9_$]
identifier:   [a-zA-Z]{followsym}*   |   [_$%]{followsym}+
```

PTX 未规定标识符的最大长度，建议所有实现至少支持 1024 个字符的最小长度。

许多高级语言（如 C 和 C++）对标识符名称有类似规则，但不允许使用百分号。**PTX 允许百分号作为标识符的第一个字符**，可用于避免命名冲突（例如用户自定义变量名与编译器生成的名称之间的冲突）。

PTX 预定义了一个常量和少量以百分号开头的特殊寄存器，如下表所示。

**表 3：预定义标识符（Predefined Identifiers）**

```
%aggr_smem_size          %dynamic_smem_size       %lanemask_gt      %reserved_smem_offset_begin
%clock                   %envreg<32>              %lanemask_le      %reserved_smem_offset_cap
%clock64                 %globaltimer             %lanemask_lt      %reserved_smem_offset_end
%cluster_ctaid           %globaltimer_hi          %nclusterid       %smid
%cluster_ctarank         %globaltimer_lo          %nctaid           %tid
%cluster_nctaid          %gridid                  %nsmid            %total_smem_size
%cluster_nctarank        %is_explicit_cluster     %ntid             %warpid
%clusterid               %laneid                  %nwarpid          WARP_SZ
%ctaid                   %lanemask_eq             %pm0, ..., %pm7
%current_graph_exec      %lanemask_ge             %reserved_smem_offset_<2>
```

---

## 4.5 常量（Constants）

PTX 支持整数常量和浮点常量以及常量表达式（constant expressions）。这些常量可用于数据初始化和指令操作数。

- 对整数、浮点和位大小（bit-size）类型，类型检查规则保持不变。
- 对谓词类型（predicate-type）数据和指令，允许使用整数常量，其解释方式与 C 相同：零值为 `False`，非零值为 `True`。

### 4.5.1 整数常量（Integer Constants）

整数常量为 64 位大小，可以是有符号或无符号的，即每个整数常量具有类型 `.s64` 或 `.u64`。

整数字面量（integer literals）可以用十进制、十六进制、八进制或二进制表示，语法遵循 C 的规范。整数字面量后可紧跟字母 `U` 表示无符号。

```
hexadecimal literal:   0[xX]{hexdigit}+U?
octal literal:         0{octal digit}+U?
binary literal:        0[bB]{bit}+U?
decimal literal:       {nonzero-digit}{digit}*U?
```

整数字面量为非负值，其类型由数值大小和可选类型后缀决定：
- 字面量默认为有符号（`.s64`），除非该值无法完整表示为 `.s64` 或指定了无符号后缀，此时字面量为无符号（`.u64`）。

预定义的整数常量 `WARP_SZ` 指定目标平台每个 warp 的线程数；目前所有目标架构的 `WARP_SZ` 值均为 32。

### 4.5.2 浮点常量（Floating-Point Constants）

浮点常量以 64 位双精度值表示，所有浮点常量表达式均使用 64 位双精度算术求值。唯一的例外是用于表示精确单精度浮点值的 32 位十六进制表示法；这类值保留其精确的 32 位单精度值，不能用于常量表达式。

浮点字面量可以带有可选的小数点和可选的有符号指数。与 C/C++ 不同，没有后缀字母来指定大小；字面量始终以 64 位双精度格式表示。

PTX 还提供了第二种浮点常量表示方式，使用十六进制常量来指定精确的机器表示：

```
0[fF]{hexdigit}{8}    // single-precision floating point（单精度浮点）
0[dD]{hexdigit}{16}   // double-precision floating point（双精度浮点）
```

- 指定 IEEE 754 双精度浮点值：常量以 `0d` 或 `0D` 开头，后跟 16 个十六进制数字。
- 指定 IEEE 754 单精度浮点值：常量以 `0f` 或 `0F` 开头，后跟 8 个十六进制数字。

**示例：**

```ptx
mov.f32   $f3, 0F3f800000;   // 1.0
```

### 4.5.3 谓词常量（Predicate Constants）

在 PTX 中，整数常量可以用作谓词（predicate）。对于谓词类型的数据初始值和指令操作数，整数常量的解释与 C 相同：零值为 `False`，非零值为 `True`。

### 4.5.4 常量表达式（Constant Expressions）

在 PTX 中，常量表达式使用运算符构成（与 C 类似），求值规则也与 C 相似，但通过限制类型和大小、去除大多数类型转换（cast）、定义完整语义以消除 C 中实现相关情况而有所简化。

常量表达式由以下部分构成：
- 常量字面量
- 一元加和减
- 基本算术运算符（加、减、乘、除）
- 比较运算符
- 条件三元运算符（`?:`）
- 括号

整数常量表达式还允许：
- 一元逻辑取反（`!`）
- 按位取反（`~`）
- 取余（`%`）
- 移位运算符（`<<` 和 `>>`）
- 位类型运算符（`&`、`|` 和 `^`）
- 逻辑运算符（`&&`、`||`）

PTX 中的常量表达式不支持整数和浮点之间的类型转换（cast）。

常量表达式的求值遵循与 C 相同的运算符优先级。表 4 给出了运算符优先级和结合性。一元运算符的优先级最高，每一行优先级依次降低。同一行的运算符具有相同的优先级：一元运算符从右到左求值，二元运算符从左到右求值。

**表 4：运算符优先级（Operator Precedence）**

| 种类（Kind）   | 运算符符号                  | 运算符名称                          | 结合性（Associates）|
|--------------|--------------------------|------------------------------------|--------------------|
| Primary（主）  | `()`                      | 括号（parenthesis）                 | n/a                |
| Unary（一元）  | `+ - ! ~`                 | 正、负、逻辑非、按位取反              | 右结合（right）     |
|               | `(.s64)(.u64)`            | 类型转换（casts）                   | 右结合（right）     |
| Binary（二元） | `* / %`                   | 乘、除、取余                        | 左结合（left）      |
|               | `+ -`                     | 加、减                              | 左结合             |
|               | `>> <<`                   | 移位                               | 左结合             |
|               | `< > <= >=`               | 有序比较（ordered comparisons）     | 左结合             |
|               | `== !=`                   | 等于、不等于                        | 左结合             |
|               | `&`                       | 按位与（bitwise AND）               | 左结合             |
|               | `^`                       | 按位异或（bitwise XOR）              | 左结合             |
|               | `\|`                      | 按位或（bitwise OR）                | 左结合             |
|               | `&&`                      | 逻辑与（logical AND）               | 左结合             |
|               | `\|\|`                    | 逻辑或（logical OR）                | 左结合             |
| Ternary（三元）| `?:`                      | 条件运算符（conditional）            | 右结合（right）     |

### 4.5.5 整数常量表达式求值（Integer Constant Expression Evaluation）

整数常量表达式在编译时按照一组规则求值，这些规则决定每个子表达式的类型（有符号 `.s64` 与无符号 `.u64`）。这些规则基于 C 中的规则，但简化为仅适用于 64 位整数，并且在所有情况下行为均完全定义（特别是对于取余和移位运算符）。

- **字面量**：默认为有符号，除非无符号是为了防止溢出，或字面量使用了 `U` 后缀。
  - 例如：`42`、`0x1234`、`0123` 为有符号。
  - 例如：`0xfabc123400000000`、`42U`、`0x1234U` 为无符号。
- **一元加和减**：保留输入操作数的类型。
  - 例如：`+123`、`-1`、`-(-42)` 为有符号。
  - 例如：`-1U`、`-0xfabc123400000000` 为无符号。
- **一元逻辑取反（`!`）**：产生值为 `0` 或 `1` 的有符号结果。
- **一元按位取反（`~`）**：将源操作数解释为无符号并产生无符号结果。
- **某些二元运算符**需要对源操作数进行规范化，称为**通常算术转换（usual arithmetic conversions）**：若任一操作数为无符号，则将两个操作数均转换为无符号类型。
- **加、减、乘、除**：执行通常算术转换，结果类型与转换后的操作数类型相同（若任一源操作数为无符号则结果为无符号，否则为有符号）。
- **取余（`%`）**：将操作数解释为无符号。注意这与 C 不同，C 允许负除数但行为由实现决定。
- **左移和右移**：将第二个操作数解释为无符号，结果类型与第一个操作数相同。右移行为由第一个操作数的类型决定：有符号值的右移是算术右移（保留符号位），无符号值的右移是逻辑右移（移入零位）。
- **AND（`&`）、OR（`|`）、XOR（`^`）**：执行通常算术转换，产生与转换后操作数类型相同的结果。
- **AND_OP（`&&`）、OR_OP（`||`）、Equal（`==`）、Not_Equal（`!=`）**：产生有符号结果，结果值为 0 或 1。
- **有序比较（`<`、`<=`、`>`、`>=`）**：对源操作数执行通常算术转换并产生有符号结果，结果值为 0 或 1。
- **类型转换（cast）**：支持使用 `(.s64)` 和 `(.u64)` 将表达式转换为有符号或无符号。
- **条件运算符（`?:`）**：第一个操作数必须是整数，第二和第三个操作数要么都是整数，要么都是浮点数。对第二和第三个操作数执行通常算术转换，结果类型与转换后的类型相同。

### 4.5.6 常量表达式求值规则总结（Summary of Constant Expression Evaluation Rules）

**表 5：常量表达式求值规则**

| 种类（Kind）  | 运算符（Operator） | 操作数类型（Operand Types）        | 操作数解释（Operand Interpretation） | 结果类型（Result Type）        |
|-------------|-------------------|----------------------------------|-----------------------------------|-----------------------------|
| Primary     | `()`              | 任意类型                          | 与源相同                           | 与源相同                      |
|             | 常量字面量（literal）| n/a                            | n/a                               | `.u64`、`.s64` 或 `.f64`     |
| Unary       | `+ -`             | 任意类型                          | 与源相同                           | 与源相同                      |
|             | `!`               | integer（整数）                   | 零或非零                           | `.s64`                       |
|             | `~`               | integer                          | `.u64`                            | `.u64`                       |
| Cast        | `(.u64)`          | integer                          | `.u64`                            | `.u64`                       |
|             | `(.s64)`          | integer                          | `.s64`                            | `.s64`                       |
| Binary      | `+ - * /`         | `.f64`                           | `.f64`                            | `.f64`                       |
|             |                   | integer                          | 使用通常转换（usual conversions）    | 转换后的类型                   |
|             | `< > <= >=`       | `.f64`                           | `.f64`                            | `.s64`                       |
|             |                   | integer                          | 使用通常转换                        | `.s64`                       |
|             | `== !=`           | `.f64`                           | `.f64`                            | `.s64`                       |
|             |                   | integer                          | 使用通常转换                        | `.s64`                       |
|             | `%`               | integer                          | `.u64`                            | `.s64`                       |
|             | `>> <<`           | integer                          | 第一操作数不变，第二操作数为 `.u64`   | 与第一操作数相同                |
|             | `& \| ^`          | integer                          | `.u64`                            | `.u64`                       |
|             | `&& \|\|`         | integer                          | 零或非零                           | `.s64`                       |
| Ternary     | `?:`              | `int ? .f64 : .f64`              | 与源相同                           | `.f64`                       |
|             |                   | `int ? int : int`                | 使用通常转换                        | 转换后的类型                   |
