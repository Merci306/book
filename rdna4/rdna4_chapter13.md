# 第13章：浮点内存原子操作

浮点原子操作可作为 LDS、Buffer 以及 Flat/Global/Scratch 指令发出。

内存原子操作不报告任何数值异常（例如信号 NaN）。

## 13.1 舍入

LDS 和内存原子操作的浮点原子加法的舍入模式固定为"就近舍入到偶数"（round to nearest even）。原子操作中 MODE.round 字段被忽略。

## 13.2 非规格化数（Denormals）

当这些操作处理浮点数据时，数据中可能包含非规格化数，或操作结果可能产生非规格化数。浮点原子指令可选择直接传递非规格化值，或将其刷新为零。

LDS 索引指令允许根据 MODE.denormal wave 状态寄存器来选择直通或刷新为零。与 VALU 操作类似，"denorm_single"影响 F32 操作，"denorm_double"影响 F64 和 F16。LDS 指令使用 FP_DENORM 的两个控制位（allow_input_denormal、allow_output_denormal），分别独立控制输入和输出的刷新行为。使用 MODE 寄存器可使 LDS 与 VALU 的浮点操作产生相同的结果。

- 16 位和 32 位浮点加法器同时使用 MODE 中的输入和输出非规格化数刷新控制
- 64 位浮点加法器不刷新非规格化数
- 浮点 MIN 和 MAX 仅使用"输入非规格化数"刷新控制
  - 比较中的每个输入在指数为零且刷新控制有效时，比较前将两个操作数的尾数均刷新为零
  - Min 和 Max 先刷新输入非规格化数（若已启用），再对刷新后的值执行 Min/Max 操作
  - 浮点比较存储（"compare swap"）根据"输入非规格化数刷新"控制同时刷新输入和输出非规格化数

**非规格化数刷新行为汇总表**

| 操作 | LDS (DS) 输入刷新 | LDS (DS) 输出刷新 | LDS (FLAT) 输入刷新 | LDS (FLAT) 输出刷新 | L2 Cache 输入刷新 | L2 Cache 输出刷新 | Data Fabric 输入刷新 | Data Fabric 输出刷新 |
|------|-----------------|-----------------|--------------------|--------------------|-----------------|-----------------|--------------------|--------------------|
| ADD_F32 | Mode.I | Mode.O | NoFlush | NoFlush | NoFlush | NoFlush | Reg | Reg |
| PK_ADD_F16 / _BF16 | Mode.I | Mode.O | NoFlush | NoFlush | NoFlush | NoFlush | Reg | Reg |
| Compare-Store / Compare-Swap | Mode.I | Mode.I | NoFlush | NoFlush | NoFlush | NoFlush | NoFlush | NoFlush |

说明：
- **NoFlush**：不刷新非规格化数
- **Mode.I**：刷新行为由 MODE.denorm（输入非规格化数控制）控制
- **Mode.O**：刷新行为由 MODE.denorm（输出非规格化数控制）控制
- **Reg**：非规格化数刷新由配置寄存器控制（应设置为"不刷新"）

通过 FLAT 指令处理但实际访问 LDS 的操作，无论 MODE.denorm 的设置如何，均不刷新非规格化数。

CompareStore（"compare swap"）在输入非规格化数刷新发生时同时刷新结果。

## 13.3 NaN 处理

NaN（Not a Number，非数字）是 IEEE-754 中表示无法计算的结果的特殊值。

NaN 有两种类型：安静 NaN（Quiet NaN）和信号 NaN（Signaling NaN）：
- **安静 NaN**：指数 = 0xFF，尾数最高位 = 1
- **信号 NaN**：指数 = 0xFF，尾数最高位 = 0，且至少有另一个尾数位 = 1

LDS 不会因信号 NaN 产生任何异常或"信号"。

`DS_ADD_F32` 可以产生安静 NaN，或从其输入中传播 NaN：若任一输入为 NaN，则输出为该 NaN；若两个输入均为 NaN，则选择第一个输入的 NaN 作为输出。信号 NaN 被转换为安静 NaN。

当非规格化数被刷新时（见上节表格），刷新发生在操作之前（即比较之前）。

**FP MAX / MIN 选择规则**

```
if      (Src0==NaN)    result = quiet(Src0)    // 信号 NaN 或安静 NaN
else if (Src1==NaN)    result = quiet(Src1)
else if (Src0>Src1)    larger_of(src0, src1)   // 或 smaller_of（取最小值时）
"larger_of" 从小到大的顺序：-inf, -float, -denorm, -0, +0, +denorm, +float, +inf
-0 < +0，保留 -0。若任意输入为 sNaN，触发无效异常信号。
"smaller_of" 顺序同上。
-0 < +0，保留 -0。若任意输入为 sNaN，触发无效异常信号。
```

**FP MAXNUM / MINNUM 选择规则**

```
if      (src0 == SNaN 或 QNaN) && (src1 == SNaN 或 QNaN)  result = QNaN(src0)
else if (src0 == SNaN 或 QNaN)  result = src1
else if (src1 == SNaN 或 QNaN)  result = src0
else                            result = larger_of(src0, src1)  // 或 smaller_of（取最小值时）
"larger_of" 从小到大的顺序：-inf, -float, -denorm, -0, +0, +denorm, +float, +inf
-0 < +0，保留 -0。若任意输入为 sNaN，触发无效异常信号。
"smaller_of" 顺序同上。
-0 < +0，保留 -0。若任意输入为 sNaN，触发无效异常信号。
```

内存原子操作使用 MINNUM / MAXNUM 风格的规则。

对于 VALU 操作，当 CLAMP=1 时，任何 NaN 结果均被截断为零，异常在应用 CLAMP 之前报告。

**浮点加法规则**

1. 若 SRC0 == NaN：result = QNaN of SRC0（保留符号）
2. 否则若 SRC1 == NaN：result = QNaN of SRC1（保留符号）
3. 否则若 SRC0 == INF 且 SRC1 == INF 且符号相反：-QNAN（FP32：0xFFC00000）
4. 否则若 SRC0 == INF：result = SRC0
5. 否则若 SRC1 == INF：result = SRC1
