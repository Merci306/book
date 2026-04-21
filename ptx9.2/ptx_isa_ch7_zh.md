# 第 7 章：抽象 ABI

PTX 不暴露特定调用约定、栈布局和应用二进制接口（ABI）的细节，而是提供一个略高层的抽象，并支持多种 ABI 实现。本章描述实现 ABI 隐藏所需的 PTX 特性，包括：函数定义、函数调用、参数传递，以及栈上内存分配（`alloca`）。

---

## 7.1 函数声明与定义

PTX 用 `.func` 指令声明和定义函数。

- **声明（declaration）**：指定可选的返回参数列表、函数名、可选的输入参数列表（即函数原型）
- **定义（definition）**：在声明基础上加上函数体

函数必须在被调用前先声明或定义。

最简单的函数（无参数、无返回值）：

```ptx
.func foo {
    ...
    ret;
}
...
call foo;
...
```

`call` 指令将控制权转移到 `foo`，隐式保存返回地址；`ret` 指令将控制权交还给调用处的下一条指令。

### 标量/向量参数

标量和向量基础类型的输入/返回参数可直接用寄存器变量表示：

```ptx
.func (.reg .u32 %res) inc_ptr (.reg .u32 %ptr, .reg .u32 %inc) {
    add.u32 %res, %ptr, %inc;
    ret;
}
...
call (%r1), inc_ptr, (%r1, 4);
...
```

> 使用 ABI 时，`.reg` 状态空间参数**至少为 32 位**。源语言中的亚字（subword）对象应提升为 32 位寄存器，或使用 `.param` 字节数组。

### 结构体参数（`.param` 空间）

C 结构体和联合体在 PTX 中被**展平**为寄存器或字节数组，用 `.param` 空间内存表示。

示例 C 结构体（按值传递）：

```c
struct { double dbl; char c[4]; };
```

该结构体在 PTX 中展平为 12 字节数组（8 字节对齐，保证 `.f64` 字段对齐）：

```ptx
.func (.reg .s32 out) bar (.reg .s32 x, .param .align 8 .b8 y[12]) {
    .reg .f64 f1;
    .reg .b32 c1, c2, c3, c4;
    ld.param.f64 f1,  [y+0];
    ld.param.b8  c1,  [y+8];
    ld.param.b8  c2,  [y+9];
    ld.param.b8  c3,  [y+10];
    ld.param.b8  c4,  [y+11];
    // 使用 x, f1, c1, c2, c3, c4 进行计算
}

// 调用方
{
    .param .b8 .align 8 py[12];
    st.param.b64 [py+0],  %rd;
    st.param.b8  [py+8],  %rc1;
    st.param.b8  [py+9],  %rc2;
    st.param.b8  [py+10], %rc1;
    st.param.b8  [py+11], %rc2;
    call (%out), bar, (%x, py);
}
```

### `.param` 空间的两种用途

| 角色 | 用途 |
|---|---|
| **调用方（caller）** | 设置传递给被调函数的值；接收被调函数的返回值 |
| **被调方（callee）** | 接收参数值；将返回值传回调用方 |

### 参数传递约束

**调用方（caller）：**
- 参数可以是 `.param` 变量、`.reg` 变量或常量
- `.param` 字节数组参数必须与形参类型、大小、对齐一致，且必须在调用方的局部作用域内声明
- 所有用于传参的 `st.param` 指令必须**紧接在** `call` 指令之前；接收返回值的 `ld.param` 指令必须**紧接在** `call` 指令之后，不得有控制流变化
- 传参用的 `st.param` 和 `ld.param` 不能带谓词

**被调方（callee）：**
- 输入和返回参数可以是 `.param` 或 `.reg` 变量
- `.param` 内存中的参数必须对齐到 1、2、4、8 或 16 字节的倍数
- `.reg` 状态空间中的参数至少为 32 位
- `.reg` 状态空间用于接收和返回基础类型标量/向量值（非 ABI 模式下支持亚字对象）

> **注意**：选择 `.reg` 还是 `.param` 传参，**不影响**参数最终是放在物理寄存器还是栈上。实际映射取决于 ABI 定义以及参数的顺序、大小和对齐。

---

### 7.1.1 与 PTX ISA 1.x 的变化

- **PTX 1.x**：形参只能在 `.reg` 空间；无数组参数支持；C 结构体被展平为多个寄存器，支持多返回值
- **PTX 2.0+**：形参可在 `.reg` 或 `.param` 空间；`.param` 支持数组；`sm_20` 及以上限制为单一返回值，不适合放入寄存器的对象用 `.param` 字节数组返回
- **PTX 3.0+**：编译使用 ABI 时，禁止模块作用域的 `.reg` 和 `.local` 变量，仅允许在函数作用域内使用

> PTX 的栈式 ABI 仅适用于 `sm_20` 及以上目标。

---

## 7.2 可变参数函数（Variadic Functions）

> **注意**：原先未实现的可变参数函数支持已从规范中移除。

PTX 6.0 支持向函数传递**无大小数组参数（unsized array parameter）**，可用于实现可变参数函数。详见 `.func` 指令说明。

---

## 7.3 Alloca

PTX 提供 `alloca` 指令，用于在运行时在**线程本地内存栈**上分配存储空间。

- 分配的栈内存通过 `alloca` 返回的指针，用 `ld.local` / `st.local` 访问
- `stacksave` — 将栈指针当前值读入局部变量
- `stackrestore` — 用保存的值恢复栈指针

这三条指令详见"栈操作指令（Stack Manipulation Instructions）"章节。
