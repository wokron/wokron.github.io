---
title: 了解LLVM IR，从手写开始
tags:
  - LLVM
  - 编译
  - 汇编
  - 短路求值
categories: 学习笔记
abbrlink: '52312655'
mathjax: true
date: 2023-11-03 21:56:13
---
## 一、总览
如果你想要了解 LLVM，那么有几种可能呢？你或许是想要创造一门新的编程语言的技术爱好者；或是陷入编译课的泥沼不能自拔的大学生，但不管怎么样，你都只希望使用 LLVM 做一件事：生成目标代码。

但是这一过程并不容易。你大概率经过了编译前端的艰辛历程，但是现在在你面前的是另一座大山，LLVM IR。在最开始就看到什么基本块、虚拟寄存器、phi 节点之类的概念使你心烦意乱；各种各样的代码示例中充满了和重点并不相关的细节。这些都使你无从下手……

会不会从一开始就错了？LLVM IR 并不应当被视为一种为了 “适配” LLVM 后端而创造的 “中间表示”，而应理解为一种特殊的编程语言。只不过这种语言站在了高级与低级语言之间，既具备了一定的抽象能力，又反映了底层汇编的工作原理。

<!-- more -->

本篇文章，就将以编程语言的视角粗略介绍 LLVM IR。通过直接手写 LLVM IR，逐步分析其特点和原理。希望能够有所帮助。

### （1）准备工作
LLVM IR 虽然是中间代码，但是 LLVM 也提供了 lli 工具用于解释/即时编译执行 LLVM IR 文件。这样 LLVM IR 又与 Java 字节码有了一些相似之处。使用 lli 可以直接运行我们手写的中间代码，很是方便。使用 lli 这需要安装 LLVM。
```sh
sudo apt-get install llvm
```

另外，LLVM 中是没有内建输入输出的，因为 LLVM 的工作在操作系统之下。这就导致了一个问题，我们不能得知程序的执行结果。一种方法是可以用函数的返回值来输出，但是这样输出只能是单个数值。为了方便起见，可以先用 c 语言编写一个输出库。并使用 clang 将其编译成 LLVM IR。为此需要安装 clang
```sh
sudo apt-get install clang
```

对于输出库，这里只简单定义 `putint` 函数。
```c
// lib.c
#include <stdio.h>

void putint(int i) { printf("%d\n", i); }
```

使用 clang 将其翻译成中间代码。不要担心，我们并不需要关心生成的 `lib.ll` 中的任何内容。
```sh
clang -emit-llvm -S ./lib.c -o lib.ll
```

现在准备工作就全部完成了。接下来我们从个第一个程序开始。

### （2）第一个程序，但是没有 `hello world`
因为懒得去定义字符的输出，所以第一个程序并不会输出 hello world。这里就写一个简单的加法程序吧。只是输出 1、2 以及 1 + 2 的结果。
```llvm
; first.ll
declare void @putint(i32)

define i32 @main() {
  call void @putint(i32 1)
  call void @putint(i32 2)
  %result = add i32 1, 2
  call void @putint(i32 %result)

  ret i32 0
}
```
> LLVM IR 使用 `;` 用作注释。

编写完代码后，将该文件 first.ll 与之前通过 clang 生成的 lib.ll 使用 llvm-link 进行链接。
```sh
llvm-link lib.ll first.ll -o out.ll
```

随后就可以使用 lli 命令执行链接后的 out.ll 文件
```sh
lli out.ll
```

输出结果为
```sh
1
2
3
```

从第一个程序中，你是否感觉到 LLVM IR 有很强的 c 的味道？首先是第一条 declare 语句，很明显对应了 c 语言中头文件的声明。

```llvm
declare void @putint(i32)
```

对于函数定义，LLVM IR 中也保留了和 c 语言类似的结构。
```llvm
define i32 @main() {
  ; omit...
}
```

对于程序语句，函数调用依旧和 c 语言基本没有区别
```llvm
  call void @putint(i32 1)
```

唯一区别较大的是运算的部分，好像是不加定义就将结果赋给了一个**变量**，实际上 `%result` 表示了一个寄存器，而非在内存空间中存在的变量。
```llvm
  %result = add i32 1, 2
```
> 实际上，也不应当将 `%result` 这样的结果称为寄存器。而更应当将其看作用于表示该条指令结果的标识符。因为在 LLVM IR 中，这些 “寄存器” 并不对应任何指令集中的实际寄存器，可以任意定义，不需要进行寄存器的分配。且在同一函数中，每一条指令对应的寄存器名都不同。

另外的区别在于类型上。LLVM IR 中保留了类型，但又与高级语言只在定义时声明类型不同，需要在每一次用到某一值的时候均说明其类型。以我们的第一个程序举例，在使用 add 指令、函数调用传参和函数返回时都显式写出了类型 `i32`。

但是不管怎么样，LLVM IR 依旧与 c 语言有很多相似之处，以至于我们可以直接将这个 LLVM IR 程序转换成 c 语言。

```c
// lib.h
void putint(int);
```

```c
// first.c
#include "lib.h"

int main() {
    putint(1);
    putint(2);
    putint(1 + 2);
    return 0;
}
```

从目前看来，LLVM IR 似乎也没什么难以理解的地方。之后我们就将逐步了解 LLVM IR 中的各部分内容，并编写相应代码。

## 二、类型
LLVM IR 是强类型的，并且每一个 “数值” 或 “变量” 在使用的时候都要指明其类型。这使得类型成为 LLVM IR 中十分重要的部分。因此这里将类型作为首先探讨的内容。不过 LLVM IR 的类型系统比较复杂，因此这里也只介绍和程序关系最紧密的部分。

对我们需要了解的部分来说，LLVM IR 的类型分为基本类型和聚合类型，和 c 语言类似。基本类型是各种运算指令的基本单位，包括整型、浮点型，再加上空（`void`）。聚合类型是基本类型的组合。包括数组和结构体。此外还有特殊的指针类型，各基本类型、聚合类型和指针本身都有其指针类型。

### （1）基本类型
对整型来说，LLVM 只关注整型的长度，而不关系其有无符号。符号的区别在 LLVM 中由指令表明。如无符号整数除法和有符号整数除法分别为 `udiv` 和 `sdiv`。

整型的长度，LLVM 并不明确设定为 32 和 64。而是可以选择 $1 \sim 2^{23} - 1$ 位中的任意数值作为整型所占比特数。

在 LLVM IR 中，整型由 `i` 加上整型所占比特数表示，因此 `i1`、`i2`、`i32`、`i64` 等都是合法的。特别的是 `i1` 在 LLVM IR 中明确表示 `bool` 类型，其常量值为 `true` 和 `false` 而非 `0` 和 `1`。

```llvm
; int.ll
; i2 依旧可以使用，返回值应为 2

define i2 @main() {
  %t1 = add i2 1, 1
  ret i2 %t1
}
```

浮点类型包括 float、double 等等。根据标准的不同有不一样的具体类型。运算方式与整型类似。比如下面的例子，运行结果应为 7.555000。
```c
// lib.c
#include <stdio.h>

void putint(int i) { printf("%d\n", i); }
void putfloat(float f) { printf("%f\n", f); } // new
```
```llvm
; float.ll
declare void @putfloat(float)

define i32 @main() {
  %t0 = fadd float 2.0, 0x40163851E0000000
  call void @putfloat(float %t0)

  ret i32 0
}
```

需要注意的是，由于并非所有的小数都能用浮点数表示，因此有许多浮点数常量并不能直接被使用，如 2.0 可以由浮点数表示，因此可以作为常量；而 0.2 不能由浮点数表示，因此不能作为常量。更好的方式是直接用 16 进制表示浮点数。如上述例子中的 `0x40163851E0000000`，实际表示 0.555。

### （2）聚合类型
数组用于表示一定连续地址空间中存储的类型相同的数据。在 LLVM IR 中用 `[N x T]` 的形式表示。其中 `N` 为数组的长度，`T` 为数据类型，可以是基本类型、结构体、指针或是另一个数组。因此 LLVM IR 中可以表示多维数组。举个例子，如果要表示类似于 `int arr[2][3]` 的二维数组，则其类型为 `[2 x [3 x i32]]`。

特别需要提一下数组常量的表示方法。这对于初始化全局变量来说十分重要。在 LLVM IR 中数组常量值用 `[T V, T V, ..., T V]` 的形式表示。如果是一位数组如 `{1, 2, 3}`，则其值为 `[i32 1, i32 2, i32 3]`。如果是二维数组如 `{{1, 2}, {3, 4}}`，则其值为 `[[2 x i32] [i32 1, i32 2], [2 x i32] [i32 3, i32 4]]`，更高维度则类似。当然在 LLVM IR 中一般在值前都要加上类型，因此在使用常量时还需在最前面加上常量本身的类型，比如说 `[i32 1, i32 2, i32 3]`，常以 `[3 x i32] [i32 1, i32 2, i32 3]` 的方式使用。

接下来举一例子来说明。当然因为数组涉及到地址的计算和从地址中取值，所以代码的 `main` 函数中用到了还未提及的指令，现在可以先忽略不看。只看定义全局变量时使用到的初始值 `[3 x i32] [i32 1, i32 2, i32 3]`。

```llvm
; array.ll
declare void @putint(i32)

; 定义全局变量 @arr，这里用到了数组常量进行初始化
@arr = global [3 x i32] [i32 1, i32 2, i32 3]

define i32 @main() {
  ; 输出 arr[0] 的值
  %arr0 = getelementptr [3 x i32], [3 x i32]* @arr, i32 0, i32 0
  %arr0val = load i32, i32* %arr0
  call void @putint(i32 %arr0val)

  ; 输出 arr[1] 的值
  %arr1 = getelementptr [3 x i32], [3 x i32]* @arr, i32 0, i32 1
  %arr1val = load i32, i32* %arr1
  call void @putint(i32 %arr1val)

  ; 输出 arr[2] 的值
  %arr2 = getelementptr [3 x i32], [3 x i32]* @arr, i32 0, i32 2
  %arr2val = load i32, i32* %arr2
  call void @putint(i32 %arr2val)

  ret i32 0
}
```

现在运行该程序，应该得到如下输出
```sh
1
2
3
```

结构体与数组类似，但是用于表示连续地址空间中一组类型不同的数据。结构体在使用前需要先定义，其定义方式和 c 语言类似，同时也必须在函数体外定义。如下给出一个结构体定义的例子。例子中定义了结构体名为 `MyStruct`，其中包含一个整数、一个浮点数和一个大小为 3 的整数数组。注意 `%` 是必须的
```llvm
%MyStruct = type {
    i32,
    float,
    [3 x i32]
}
```

定义完结构体后，使用该结构体类型和使用其他类型的方式基本相同。只不过需要使用定义时的结构体名而已，还以上面的代码为例子，在使用该类型时应当使用 `%MyStruct`。

结构体常量和数组常量类似，在定义全局变量的时候使用。在 LLVM IR 中用 `{T V, T V, ..., T V}` 的形式表示。在使用时也常在前面加上常量本身的类型。如对于 `%MyStruct` 可以定义常量 `%MyStruct { i32 10, float 2.0, [3 x i32] [i32 3, i32 2, i32 1]}`。

如下是一个使用结构体的例子，同样现在并不需要关注 `main` 函数中的内容。只需要看前面对于 `%MyStruct` 结构体类型的定义和对全局变量 `@mystruct` 定义时结构体常量的声明即可。

```llvm
; struct.ll
declare void @putint(i32)
declare void @putfloat(float)

%MyStruct = type {
    i32,
    float,
    [3 x i32]
}

@mystruct = global %MyStruct { i32 10, float 2.0, [3 x i32] [i32 3, i32 2, i32 1]}

define i32 @main() {
  ; 取 i32
  %mystruct0 = getelementptr %MyStruct, %MyStruct* @mystruct, i32 0, i32 0
  %mystruct0val = load i32, i32* %mystruct0
  call void @putint(i32 %mystruct0val)
  
  ; 取 float
  %mystruct1 = getelementptr %MyStruct, %MyStruct* @mystruct, i32 0, i32 1
  %mystruct1val = load float, float* %mystruct1
  call void @putfloat(float %mystruct1val)

  ; 取 [3 x i32] 各元素
  ; 取 [0]
  %mystruct2 = getelementptr %MyStruct, %MyStruct* @mystruct, i32 0, i32 2
  %arr0 = getelementptr [3 x i32], [3 x i32]* %mystruct2, i32 0, i32 0
  %arr0val = load i32, i32* %arr0
  call void @putint(i32 %arr0val)

  ; 取 [1]
  %arr1 = getelementptr [3 x i32], [3 x i32]* %mystruct2, i32 0, i32 1
  %arr1val = load i32, i32* %arr1
  call void @putint(i32 %arr1val)

  ; 取 [2]
  %arr2 = getelementptr [3 x i32], [3 x i32]* %mystruct2, i32 0, i32 2
  %arr2val = load i32, i32* %arr2
  call void @putint(i32 %arr2val)

  ret i32 0
}

```

运行程序的结果应是
```sh
10
2.000000
3
2
1
```

> 最后还需要说明一个小知识点。那就是可以使用 `zeroinitializer` 来统一对全局量所对应的地址区域的所有位置零。如一个全为 0 的大小为 3 的整数数组，可以使用如下形式作为初始值 `[3 x i32] zeroinitializer` 这表示一个 3 x i32 大小的全 0 区域。类似的，对于二维，如 `{{0, 0}, {0, 1}}`，也可以用如下的形式初始化：`[2 x [2 x i32]] [[2 x i32] zeroinitializer, [2 x i32] [i32 0, i32 1]]`。结构体使用 `zeroinitializer` 类似。

### （3）指针
指针就是地址，所有类型都有其对应的指针。这一点和 c 相同。地址当然主要用于存取数据，有时也需要直接对地址进行运算操作。前一种操作将在下一章讲解，而后一种操作主要通过 `ptrtoint .. to` 和 `inttoptr .. to` 将指针与整型相互转化来完成。举个例子，可以使用 `%addr_int = ptrtoint i32* %some_addr to i64` 将指针转换为 64 位整数。这样就可以对指针采用整数的运算了。

指针常常作为函数参数传递。比如说对于一个数组，我们并不会将其全部内容都复制到栈上，而是只传递该数组的指针，这一点很好理解。与 c 语言类似，一个特殊的情况是在传递多维数组时，需要明确指定大于 1 的维度的具体数值。在 c 语言这表示为
```c
void func(int arr[][10]) { }
```

而在 LLVM IR 中，则以如下形式表示
```c
define void @func([10 x i32]* %arr) {
  ret void
}
```

如下是一个使用指针的例子，请关注函数体中对 `ptrtoint` 的使用，以及函数的参数中的指针。通过注释，应该可以理解函数间数组指针的传递方式。

```c
// lib.c
#include <stdio.h>

void putint(int i) { printf("%d\n", i); }
void putfloat(float f) { printf("%f\n", f); }
void putaddr(size_t addr) { printf("%#zx\n", addr); } // new

```

```llvm
; pointer.ll
declare void @putaddr(i64)

@arr = global [2 x [2 x i32]] [[2 x i32] [i32 1, i32 2], [2 x i32] [i32 3, i32 4]]

define void @func1([2 x i32]* %arr) {
  %addr = ptrtoint [2 x i32]* %arr to i64
  call void @putaddr(i64 %addr)

  ; arr[1]
  %arr1addr = getelementptr [2 x i32], [2 x i32]* %arr, i32 0, i32 1

  ; func2(arr[1])
  call void @func2(i32* %arr1addr)
  ret void
}

define void @func2(i32* %arr) {
  %addr = ptrtoint i32* %arr to i64
  call void @putaddr(i64 %addr)
  ret void
}

define i32 @main() {
  %addr = ptrtoint [2 x [2 x i32]]* @arr to i64
  call void @putaddr(i64 %addr)
  
  ; arr[1]
  %arr1addr = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* @arr, i32 0, i32 1

  ; func1(arr[1])
  call void @func1([2 x i32]* %arr1addr)

  ret i32 0
}
```

在我的设备上，该程序的运行结果如下
```sh
0x7f03162ff000
0x7f03162ff008
0x7f03162ff00c
```
可以看出全局数组 `@arr` 的基地址为 `0x7f381bd79000`。之后调用 `func1(arr[1])`，在 `func1` 中传入的地址为 `0x7f03162ff008`，偏移了 8 个字节，即两个 i32 长度。再之后在 `func1` 中调用 `func2(arr[1])`，传入 `func2` 的地址变为 `0x7f03162ff00c`，与 `0x7f03162ff008` 相比偏移了 4 个字节，正是一个 i32 长度。

## 三、变量
类型定义了数据的表示，而变量定义了数据的位置。接下来我们就要了解 LLVM IR 中的变量。

众所周知，按其存储位置，变量分为三类，分别是栈上变量、堆上变量和静态存储区的变量。然而堆上变量由操作系统（或者说程序员）分配，在编译中并不考虑，所以实际上在 LLVM IR 中只有另外两种变量。

### （1）全局量
静态存储区的变量，实际上我们已经在之前的章节中看到过了。就是那些定义在函数体以外的全局变量。此种变量可以使用 `global` 或 `constant` 定义。如 `@a = global i32 1` 或 `@two = constant i32 2`，其中以 `constant` 定义的全局量只读而不可写。这些变量都需要为其设置初始值，如果不想设定（或者说默认设定为 0），那么可以使用 `zeroinitializer` 将其置零，这在上一节中已经说明了。

> 对于全局量还需要注意一点，那就是全局量在定义的同时都为其设定了一个名字，形如 `@xxx`。此标签和我们在函数体中使用的 “虚拟寄存器” 不同。实际上应当将其看作汇编中的 “标签”。也就是说，其本质应当是一个地址；更进一步来说，也可以看作是一个指向所定义全局量的指针。正因如此，像是 `%t = add i32 @a, 1` 这样的表述是错误的，因为 `@a` 的类型应为 `i32*`。

### （2）局部量
除了全局量就只剩下栈上变量了，或者可以称为局部变量。栈上变量的空间是动态增减的，在 LLVM 中使用一条特殊的指令 `alloca` 表示函数中的栈上空间申请操作。而当函数结束返回时，则会自动释放栈上空间。

> 需要注意局部变量没有真正的 constant。因为对于全局量，可以分别将变量和常量划分到不同的地址区域中。使得变量所在地址区域可读写，而常量区域只可读。关于这一点，可以参见操作系统的页表管理和程序段加载相关的知识。

`alloca` 指令的具体形式是这样的 `%xxx = alloca T`，表示分配了一块用于存储 T 类型的栈上空间，**其地址**存储在虚拟寄存器 `%xxx` 中。注意这里 `%xxx` 的类型是 `T*` 而非 `T`，这一点与全局变量时类似。

> `alloca` 与全局变量的定义有不同之处。`alloca` 指令不会带有初始值，因此如果想要设定初始值的话只能在此命令之后使用另外一条指令 `store` 设定其值。

> 需要注意一点，`alloca` 只应在函数的最开始使用，因为 `alloca` 本质上是向下移动栈指针，在栈上多分配一块空间。如果 `alloca` 不在最开始使用，则可能会出现在循环中，这会导致不停的分配栈空间，可能导致栈溢出。
>
> 如下代码能证明循环中的局部变量都占据相同的栈空间：
> ```c
> #include <stdio.h>
> 
> int main() {
>     for (int i = 0; i < 10; i++) {
>         int a;
>         printf("%d ", a);
>         a++;
>     }
>     return 0;
> }
> ```
> 输出为 `0 1 2 3 4 5 6 7 8 9 `

现在我们已经知道了分配的内存空间的地址，想要对内存进行存取操作。又需要两条指令 `store` 和 `load`。（时刻记住，LLVM IR 中操作的是寄存器而非内存。这是 LLVM IR 类似于汇编而不同于高级语言的地方）。

对于 `store`，目的当然是将寄存器中的值存储到目的内存空间中。其格式为 `store T %val, P %ptr`，`%val` 为存储值的寄存器，T 为其类型；`%ptr` 为存储地址的寄存器，当然也可以是全局变量的 `@ptr`，P 为其类型，应当满足 T* = P。值得注意的是，`store` 指令是一条 没有“返回值” 的指令。

而 `load` 则是 `store` 的反操作，唯一的不同是 `load` 具有 ”返回值”。其格式为 `%val = load T, P %ptr`。

接下来就结合上述内容举一例子。

```llvm
; val.ll

declare void @putint(i32)

@a = global i32 1
@two = constant i32 2

define i32 @main() {
  ; alloca 必须在最开始使用
  %var = alloca i32
  
  ; 错误用法：call void @putint(@a)
  %a_val = load i32, i32* @a
  call void @putint(i32 %a_val)
  
  ; 错误用法：store i32 3, i32* @two
  store i32 100, i32* @a
  %a_val2 = load i32, i32* @a
  call void @putint(i32 %a_val2)
  
  ; 未初始化，取值未定义
  %var_val = load i32, i32* %var
  call void @putint(i32 %var_val)
  
  ; 初值设定为 123
  store i32 123, i32* %var
  %var_val2 = load i32, i32* %var
  call void @putint(i32 %var_val2)
  
  ret i32 0
}
```

输出结果如下，其中第三行的输出实际不确定。
```sh
1
100
0
123
```
### （3）古怪的 getelementptr
现在我们已经了解了如何定义变量，以及如何存取其值了。所以可以进入下一章了吗？其实还不能。假如我现在想要定义一个数组，并且想要获取其每个元素的值；或者定义一个结构体，访问其中的各个属性，需要怎么做呢？

不管是全局变量还是栈上，定义的话很是简单，只需要 `@arr = global [3 x i32] [i32 1, i32 2, i32 3]` 或 `%arr = alloca [3 x i32]` 即可。但问题是我们要如何取每一个元素的值呢？毕竟现在我们只知道数组的首地址。

你可能会想要使用指针运算，将当前的地址加上 4 得到下一个元素的地址。但是这样需要用到 `ptrtoint`，而得到地址之后，因为 `load` 和 `store` 只接受对应类型的指针，所以还需要用 `inttoptr` 转换回对应类型指针。这样的操作实在太过复杂，很明显不是正解。

真正的解决方法是使用指令 `getelementptr`。此指令在 LLVM IR 严格的类型系统下有着很大的作用，能够很方便的得到聚合类型各个元素的地址，同时实现指针类型的转换。

`getelementptr` 在我看来有着三种使用形式，接下来就一一解释这三种形式

#### 计算数组指针的 getelementptr
`getelementptr` 最简单的形式如下 `%ptr = getelementptr T, T* %baseptr, V %idx`。这实际上等价于 c 语言中的指针加法 `baseptr + idx`。旨在求以 T 类型的大小为单位，从 `%base` 所在地址开始，偏移 `%idx` 得到的地址。因此经过此命令后，指针的类型并不发生改变。

假设现在有一函数，其 c 语言形式为 `void func(int arr[])`，则我们现在就可以使用 `getelmentptr` 取得其值。
```llvm
define void @func(i32* %arr) {
  ; putint(arr[0])
  %arr0 = getelementptr i32, i32* %arr, i32 0
  %arr0val = load i32, i32* %arr0
  call void @putint(i32 %arr0val)

  ; putint(arr[1])
  %arr1 = getelementptr i32, i32* %arr, i32 1
  %arr1val = load i32, i32* %arr1
  call void @putint(i32 %arr1val)

  ret void
}
```

所以现在我们访问数组变量各元素的地址了吗？还不能。因为对于一个变量，比如说 `%arr = alloca [3 x i32]`，`%arr` 的类型实际上是 `[3 x i32]*` 而不是 `i32*`，所以此种形式的 `getelementptr` 依旧不能访问数组各元素的地址

#### 获取成员地址的 getelementptr
我们希望地址的计算不 “浮于表面” 而是要 “深入内部”。这就需要第二种形式的 `getelementptr`。形如 `%ptr = getelementptr T, T* %baseptr, V %idx1, V %idx2`。这其实只是在前一种形式的 `getelementptr` 后再加上一项偏移，而其他部分保持不变。其中 `%idx1` 的含义不变，而 `%idx2` 则意味着在取聚合类型 T 中第 `%idx2` 项的地址。其返回的类型即该项对应类型的指针。

例如 `getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* @arr, i32 0, i32 1`，用 c 语言表示即 `(arr+0)[1]`，其返回类型为 `[2 x i32]*`。结构体与此类似，只不过各属性所对应的指针类型并不相同。

此种使用方式在之前的实例代码中使用颇多。各位可以回过头看一看之前我们忽略的代码部分，现在是否就已经明白其含义了？

#### 多层嵌套成员的 getelementptr
假如现在有如下的结构体，并有一个此结构体类型的全局变量：
```llvm
%MyStruct = type {
  i32,
  [2 x [2 x i32]]
}

@a = global %MyStruct {i32 10, [2 x [2 x i32]] [[2 x i32] [i32 1, i32 2], [2 x i32] [i32 3, i32 4]]}
```

如果想要取结构体中的数组的第 1 行第 0 列的数值，对应的 `getelementptr` 指令是什么呢？

这当然可以求，只需要不断的使用 `getelementptr` 不就可以了吗？于是你写下了如下语句
```llvm
define i32 @main() {
  %t0 = getelementptr %MyStruct, %MyStruct* @a, i32 0, i32 1
  %t1 = getelementptr [2 x [2 x i32]], [2 x [2 x i32]]* %t0, i32 0, i32 1
  %t2 = getelementptr [2 x i32], [2 x i32]* %t1, i32 0, i32 0
  %val = load i32, i32* %t2
  call void @putint(i32 %val)

  ret i32 0
}
```

这当然是正确的代码，但问题是这样实在是过于繁琐了，有没有一种更方便的写法呢？实际上是有的，只需要将第一条之后的 `getelementptr` 语句中的第二个 idx 不断追加到第一条 `getelementptr` 指令之后即可。对于如上的代码，其等价形式即：
```llvm
define i32 @main() {
  %t = getelementptr %MyStruct, %MyStruct* @a, i32 0, i32 1, i32 1, i32 0
  %val2 = load i32, i32* %t
  call void @putint(i32 %val2)

  ret i32 0
}
```

当然，你也可以认为 `getelementptr` 本来就只有一种形式而已。确实，只不过是我认为这样说应该会更加方便理解而已。
 
## 四、运算
计算是 CPU 的根本职责。关于这一部分实际上不需要多加赘述。因此这里只是稍微概述一下。在此事先提醒一下，运算指令的两输入都必须是相同的类型。运算指令的大致格式为 `%xxx = <op> T %yyy, %zzz`。比如说一个简单的整数加法，表示为 `%t2 = add i32 %t0, %t1`。

### （1）算数运算
运算包括算术运算和逻辑运算。算术运算当然包括加减乘除等基本运算，以 c 为例，对整数来说，其各类操作对应的 llvm 算术运算如下

|c|llvm ir|
|---|---|
|`+`|`add`|
|`-`|`sub`|
|`*`|`mul`|
|`/`|`sdiv` 或 `udiv`|
|`%`|`srem` 或 `urem`|
| &#124; | `or`|
|`&`|`and`|
|`<<`|`shl`|
|`>>`|`ashr`|

> 需要注意，对于除除法以外的其他运算，整数有无符号并无区别，但在使用除法的时候却需要指定是否有符号，这就有了 `sdiv` 和 `udiv` 的区别；取模操作实际上就是除法时求余数，因此也需要指定有无符号。另外 c 中 `>>` 为算数右移，因此对应的 llvm ir 为 `ashr`（Arithmetic Shift Right） 而非 `lshr`（Logical Shift Right）。

对于浮点数来说，LLVM 也提供了与整数运算相对应的指令，只是在指令前加上 `f` 而已。

|c|llvm ir|
|---|---|
|`+`|`fadd`|
|`-`|`fsub`|
|`*`|`fmul`|
|`/`|`fdiv`|
|`%`|`frem`|

> 因为浮点数当然都是有符号的，所以对于除法来说就不再有 `sdiv` 和 `udiv` 的区别。

对于整数运算来说，如下是一个简单的例子。浮点数的运算与其类似。
```llvm
; calc.ll
declare void @putint(i32)

define i32 @main() {
  %t0 = add i32 5, 8
  call void @putint(i32 %t0) ; 13
  
  %t1 = sub i32 5, 8
  call void @putint(i32 %t1) ; -3

  %t2 = mul i32 5, 8
  call void @putint(i32 %t2) ; 40

  %t3 = sdiv i32 8, -5
  call void @putint(i32 %t3) ; -1

  ; 注意这里，将 -5 看作无符号整数的话这个数远比 8 大，因此结果为 0
  %t4 = udiv i32 8, -5
  call void @putint(i32 %t4) ; 0 

  %t5 = udiv i32 8, 5
  call void @putint(i32 %t5) ; 1

  %t6 = srem i32 8, -5
  call void @putint(i32 %t6) ; 3

  ; 与上一条注释同理
  %t7 = urem i32 8, -5
  call void @putint(i32 %t7) ; 8

  %t8 = urem i32 8, 5
  call void @putint(i32 %t8) ; 3

  ; 0101 | 1000 = 1101
  %t9 = or i32 5, 8
  call void @putint(i32 %t9) ; 13

  ; 0101 & 0001 = 0001
  %t10 = and i32 5, 1
  call void @putint(i32 %t10) ; 1

  ; 0101 << 1 = 1010
  %t11 = shl i32 5, 1
  call void @putint(i32 %t11) ; 10

  ; 0101 >> 1 = 0010
  %t12 = lshr i32 5, 1
  call void @putint(i32 %t12) ; 2

  ; 符号右移，对负数前面补 1
  %t13 = ashr i32 -5, 1
  call void @putint(i32 %t13) ; -3

  ret i32 0
}

```

### （2）逻辑运算
逻辑运算用于比较两数的大小，对于整数和浮点数，分别用 `icmp` 和 `fcmp` 指令进行计算。`icmp` 或 `fcmp` 在使用时需要指定比较时两操作数的关系。指令的基本形式为 `%xxx = (icmp|fcmp) <cond> T %yyy, %zzz`。举个例子，比如说想要比较两整数是否相等，则应该使用指令 `%t2 = icmp eq i32 %t0, %t1`。

对于条件，同样以 c 为例，对应的 llvm ir 如下

|c|llvm ir|
|---|---|
|`==`|`eq`|
|`!=`|`ne`|
|`>`|`gt`|
|`>=`|`ge`|
|`<`|`lt`|
|`<=`|`le`|

> 需要注意的是，`icmp` 或 `fcmp` 的运算结果的类型为 `i1`，即 `bool` 类型。

### （3）类型转换
尽管 LLVM IR 中各运算指令要求操作数的类型保持一致，可是实际编写代码时并不一定满足这一条件。在执行运算时我们会进行一系列的隐式或显式的类型转换。LLVM IR 中也提供了一系列类型转换指令。不过类型转换的指令十分繁多，因此这里只提一些较为简单的类型转换指令。

对于整数来说，类型的转换就只是扩充或截断一些比特位而已。对扩充和截断，LLVM IR 提供了指令 `zext`（Zero Extend）`sext`（Sign Extend）和 `trunc`（Truncate）。这些指令的使用形式为 `%xxx = zext T %yyy to Y`，其中 Y 为要转换到的类型。因此对于指令的结果 `%xxx`，其类型为 `Y`。

`zext` 实现了零扩展，在扩展整数位数的时候会用 0 填充高位，用于对无符号整数进行扩展。`sext` 实现了符号扩展，在扩展整数位数的时候会根据之前最高位的符号来进行填充，用于对有符号整数进行扩展。`trunc` 则用于对高位进行截断。

对于浮点数来说，也有类似的指令 `fpext` 和 `fptrunc`。另外也有命令 `fptoui`、`fptosi` `uitofp` `sitofp` 用于实现有符号和无符号整数与浮点数间的相互转换。这里不再赘述。

## 五、函数调用
在之前的许多例子中，我们已经见到过函数调用的使用方式了。其形式为 `%xxx = call T @fff(T1 %p1, T2 %p2)`。当返回值类型为 `void` 的时候，不需要前面的”赋值“。

值得提一句的实际上是函数的定义方式。如果留意前面的例子的话，应该能发现函数定义时，参数实际上被表示为虚拟寄存器。也就是说在 LLVM IR 中函数参数被保存在寄存器中而不是栈中。

```llvm
; 之前章节中的代码，参数 arr 实际上是寄存器 `%arr`
define void @func2(i32* %arr) {
  %addr = ptrtoint i32* %arr to i64
  call void @putaddr(i64 %addr)
  ret void
}
```
> 当然，“参数保存在栈上”其实也并不是什么必须的要求。只不过是根本没有那么多寄存器来存储大小可变的那么多参数而已。

为保持一致性，可以在函数体中使用 `alloca` 指令为参数分配栈上空间，再使用 `store` 指令将参数寄存器中的值保存在栈上。

## 六、流程控制
最后，来谈一谈最重要的流程控制。对于像 LLVM IR 或更底层的语言来说，流程控制仅仅表现为跳转指令。在 LLVM IR 中的跳转指令为 `br`。

`br` 存在两种形式，其一是条件跳转，接受一个 bool 类型（`i1`）的值，根据其真假分别跳转到不同的标签的位置。指令的具体形式为 `br i1 %cond, label %iftrue, label %iffalse`。表示的含义很简单，如果 `%cond` 为真，则跳转到 `%iftrue` 标签所在位置；如果为假，则跳转到 `%iffalse` 所在位置。

`br` 的另一种形式是无条件跳转。形式为 `br label %target`，执行此条指令，就会跳转到 `%target` 所在位置。

而所谓标签，就是一个名称加上冒号，形如 `labelname:`。在汇编中，实际上指代紧跟冒号后的第一条指令的地址。

LLVM 对跳转语句和标签的使用做出了约束。具体来说，两个标签间的最后一条语句必须为 `br` 或 `ret`，标签间不能没有语句。如下的例子展示了错误的情况。
```llvm
; errbr.ll
define i32 @main() {
one:
  %t0 = add i32 1, 2
  ; 错误，需要 br 或 ret
two:
  ; 错误，l2 和 l3 间需要包含指令
three:
  ret i32 %t0
}
```

> 为什么这样规定？这就涉及到基本块的概念了。基本块是只有第一条语句作为入口，最后一条语句作为出口的语句序列。通过将程序划分为基本块，能够为编译器的代码优化提供便利。LLVM IR 在设计时就规定要以基本块的形式生成代码。因为基本块只有一个入口，所以只可能在最开始有一个标签；基本块只有一个出口，所以跳转语句之后不能再跟其他语句。这样我们就可以理解 LLVM IR 对跳转语句和标签的约束了：将两标签间的语句作为基本块的内容，在满足约束的情况下，最开始的标签是其唯一的入口，末尾的跳转语句是其唯一的出口。

了解了标签和跳转语句，现在我们可以使用这两者实现各种各样的程序流程。但是不管标签和跳转语句的组合多么复杂，总是逃不出三种控制流程，那就是顺序、分支和循环。（关于这一点似乎存在某种数学证明。）

### （1）分支语句
对于**顺序**不必赘述，基本块内的指令就是按照顺序执行的。对于**分支**，以我们熟悉的 c 语言举例，存在两种类型的分支。

**if 类型**：
```c
if (cond) {
  // do something if cond is true
}
```

**if-else 类型**：
```c
if (cond) {
  // do something if cond is true
} else {
  // do something if cond is false
}
```

> “还有第三种呢，就是带有 `if else` 的分支语句。”实际上这样的说法并不正确。因为 `if else` 实际上只是分支语句的嵌套而已。对于如下两种带有 `if else` 的语句
> ```c
> if (cond1) {
>   // do something if cond1 is true
> } else if (cond2) {
>   // do something if cond1 is false and cond2 is true
> }
>
> if (cond1) {
>   // do something if cond1 is true
> } else if (cond2) {
>   // do something if cond1 is false and cond2 is true
> } else {
>   // do something if cond1 and cond2 are false
> }
> ```
> 实际上应当将 `else` 之后的 `if` 看作另一条语句，为其加上括号的话，就是
> ```c
> if (cond1) {
>   // do something if cond1 is true
> } else {
>   // enter if cond1 is false
>   if (cond2) {
>     // do something if cond2 is true
>   }
> }
> 
> if (cond1) {
>   // do something if cond1 is true
> } else {
>   // enter if cond1 is false
>   if (cond2) {
>     // do something if cond2 is true
>   } else {
>     // do something if cond2 is false
>   }
> }
> ```
> > 这也是为什么 c 语言中存在“可以在 `if` `else` 后跟单条语句而不加大括号”的规则。

两种分支语句完全可以用跳转语句表示，以下就是用 LLVM IR 表示的两种控制流。尽管什么都没做，但是这个例子是可以正常编译的。

```llvm
; ifelse.ll

define void @dosomething() {
  ret void
}

define void @if(i1 %cond) {
  br i1 %cond, label %true, label %fi

true:
  call void @dosomething() ; do something if cond is true
  br label %fi

fi:
  ret void
}

define void @ifelse(i1 %cond) {
  br i1 %cond, label %true, label %false

true:
  call void @dosomething() ; do something if cond is true
  br label %fi

false:
  call void @dosomething() ; do something if cond is false
  br label %fi

fi:
  ret void
}

define i32 @main() {
  call void @if(i1 true)
  call void @ifelse(i1 true)

  ret i32 0
}
```

> 为了向 shell 的语法致敬，这里我特地用 `fi` 来表示退出分支语句时的标签。

如果写过其他汇编应该会感觉上面的代码中有看似冗余的语句，跳转语句紧跟跳转到的目标标签。
```llvm
  br label %fi
fi:
```

但是要考虑到 LLVM IR 中的约束，每个基本块都要以 `br` 或 `ret` 结尾。这样的话就不难理解了。

### （2）循环语句
循环语句有三种形式。还以 c 举例。

**while 类型**：
```c
while (cond) {
  // do something
}
```

**do-while 类型**：
```c
do {
  // do something
} while (cond);
```

**for 类型**：
```c
for (a = b, cond, a = a + c) {
  // do something
}
```

> do-while 并不常用，为什么一些编程语言还保留这种形式呢？因为 do-while 是最接近底层汇编的，其次为 while，最复杂的为 for。

循环在汇编中表示为向前的跳转，下面用 LLVM IR 分别表示三种循环类型。

```llvm
; loop.ll

define void @dosomething() {
  ret void
}

define void @while(i1 %cond) {
  br label %while

while:
  br i1 %cond, label %do, label %done

do:
  call void @dosomething() ; do something while cond is true
  br label %while

done:
  ret void
}

define void @dowhile(i1 %cond) {
  br label %do

do:
  call void @dosomething() ; do something until cond is false
  br label %while

while:
  br i1 %cond, label %do, label %done

done:
  ret void
}

define void @for(i1 %cond) {
  call void @dosomething() ; initial stmt
  br label %for

for:
  br i1 %cond, label %do, label %done

do:
  call void @dosomething() ; do something while cond is true
  br label %update

update:
  call void @dosomething() ; update stmt
  br label %for

done:
  ret void
}

define i32 @main() {
  call void @while(i1 false)
  call void @dowhile(i1 false)
  call void @for(i1 false)

  ret i32 0
}
```

> `do` 和 `done` 同样充满了 shell 的味道。

从代码中就可以看出 do-while 的实现最为简单，因为 do-while 就只是简单的跳回之前的标签而已。

在代码中还需要注意的是 for 的实现中添加了 `update:` 标签，标签之后的是每次循环结束后的更新语句。此标签实际上是考虑到 `continue` 语句的翻译。对于 `continue` 来说，需要跳过本次循环，跳转到的标签位置应该是更新语句之前，因此应当跳转到 `update:` 而非 `for:`。同理对于 do-while 我们也额外添加了 `while` 标签，在 do-while 中使用 `continue` 也应当跳转到 `while:` 而非 `do:`

### （3）短路求值
对于流程控制我们还有一点没有考虑，那就是跳转语句的条件值究竟从哪里来。当然，你可以用 `icmp`/`fcmp` 加上 `and` 和 `or` 来实现所有的布尔运算，但是这样并不满足编程语言中的一个基本设计，那就是短路求值。

> 取反操作可以这样表示，`icmp eq i1 0, i1 %cond`。其中 `%cond` 为要取反的值

`&&` 和 `||` 与 `&` 和 `|` 是不一样的，不管学什么编程语言，在最开始都会强调这一点。可是实际上如果只是计算条件值的话，也并没有什么区别。可以试试如下的例子。
```c
#include <stdio.h>

int main() {
    int a = 10;

    if ((a > 20) && (a < 40)) {
        printf("20 < a < 40\n");
    }

    if ((a > 20) & (a < 40)) {
        printf("20 < a < 40\n");
    }

    if ((a > 5) || (a < 0)) {
        printf("a < 0 or a > 5\n");
    }

    if ((a < 0) | (a > 5)) {
        printf("a < 0 or a > 5\n");
    }
}
```

唯一的问题在于求值过程中的”副作用“。很明显我们并没有在讨论函数式编程，所以副作用是不可避免的。这会使得位运算和逻辑运算的结果不同。
```c
#include <stdio.h>

int main() {
    int a, tmp;

    tmp = a = 10;
    if ((a++ > 20) && (a++ < 40)) {
        printf("20 < a < 40\n");
    }
    printf("change:%d\n", a - tmp);

    tmp = a = 10;
    if ((a++ > 20) & (a++ < 40)) {
        printf("20 < a < 40\n");
    }
    printf("change:%d\n", a - tmp);

    tmp = a = 10;
    if ((a++ > 5) || (a++ < 0)) {
        printf("a < 0 or a > 5\n");
    }
    printf("change:%d\n", a - tmp);

    tmp = a = 10;
    if ((a++ < 0) | (a++ > 5)) {
        printf("a < 0 or a > 5\n");
    }
    printf("change:%d\n", a - tmp);
}
```

**结果**：
```text
change:1
change:2
a < 0 or a > 5
change:1
a < 0 or a > 5
change:2
```

因此需要采用位运算之外的方式实现 `&&` 和 `||`。

> 实际上，因为位运算对于无副作用的表达式来说，其结果等同于短路求值，所以 LLVM IR 对于无副作用的表达式也可能采用位运算计算条件表达式。

短路求值的重点在于提前跳转，对于之前的流程控制来说，我们只是对单一的比较语句的结果进行判断，分别跳转到不同的标签位置。而对于包含了逻辑运算符的条件表达式，则相当于存在多个不同的比较语句。我们所要做的就是为每个比较语句设置合适的跳转位置。

举个简单的例子：
```c
if (a < b && c < d) {
  // do something
} else {
  // do something
}
```

对如上的代码，其流程应该是这样的：
- 首先计算 `a < b` 的值，如果为真则执行 `if` 语句块的内容，如果为假则转而计算 `c < d`；
- 对于计算 `c < d` 的情况，如果其值为真则执行 `if` 语句块的内容，如果为假则执行 `else` 语句块的内容

以 LLVM 的方式叙述，只需要将“执行”、“计算”改为跳转；将 `a < b` 和 `c < d` 视作两个不同基本块中的指令序列即可。

这样我们就可以将问题抽象化了。假设现在有一系列基本块，`cond1`、`cond2`、... 、`condn`，这些基本块分别表示条件表达式中被 `||` 和 `&&` 分隔开的不同表达式。那么问题就是：这些基本块间的跳转关系是怎样的？

对于这一问题，首先要考虑所谓的条件表达式究竟是什么。以一种不严谨的方式定义，条件表达式要么是由 `||` 分隔的序列，序列成员是由 `&&` 分隔的序列或表达式；要么是由 `&&` 分隔的序列，序列成员是由 `||` 分隔的序列或表达式。这样避免引入对 `||` 和 `&&` 优先级的讨论。

我们的问题是，对于每一个表达式的基本块，要分别设定为真和为假时跳转的目标。即对于基本块 $B$，要设定 $\text{iftrue}_B$ 和 $\text{iffalse}_B$。

那么对于任意与序列 $S_\text{and}$ 、或序列 $S_\text{or}$ 和基本块 $B$ ，我们分别定义两个集合 $\text{TRUE}$ 和 $\text{FALSE}$ 以及一个函数 $\text{first}$ 。假设与序列和或序列中的成员为 $M_i$ ，序列的长度 $|S| = n$ ，则有：

$$
\begin{align*}
  \text{first}(S) &= \text{first}(M_1) \\
  \text{first}(B) &= B
\end{align*}
$$

另外对于 $S_\text{and}$：
$$
\begin{align*}
  \text{TRUE}(S_\text{and}) &= \text{TRUE}(M_n) \\
  \text{FALSE}(S_\text{and}) &= \bigcup_{M_i \in S_\text{and}} \text{FALSE}(M_i)
\end{align*}
$$

对于 $S_\text{or}$：
$$
\begin{align*}
  \text{TRUE}(S_\text{or}) &= \bigcup_{M_i \in S_\text{or}} \text{TRUE}(M_i) \\
  \text{FALSE}(S_\text{or}) &= \text{FALSE}(M_n) \\
\end{align*}
$$

对于基本块 $B$：
$$
\begin{align*}
  \text{TRUE}(B) &= \{B\} \\
  \text{FALSE}(B) &= \{B\}
\end{align*}
$$

这样，对于条件表达式中的任意基本块，其 $\text{iftrue}_B$ 和 $\text{iffalse}_B$ 可以按照如下方式得出：

对条件表达式中出现的任意与序列和或序列 $S$ 应用如下规则：

- 若 $S$ 为与序列， $\forall M_i \in S, \forall B_j \in \text{TRUE}(M_i), \text{iftrue}_{B_j} = \text{first}(M_{i+1})$

- 若 $S$ 为或序列， $\forall M_i \in S, \forall B_j \in \text{FALSE}(M_i), \text{iffalse}_{B_j} = \text{first}(M_{i+1})$

- 若 $S$ 为组成整个条件表达式的序列，则额外有 $\forall B_i \in \text{TRUE}(S), \text{iftrue}_{B_i} = \text{T}$ ; $\forall B_j \in \text{FALSE}(S), \text{iffalse}_{B_j} = \text{F}$ ，其中 $T$ 和 $F$ 分别为循环或条件语句为真和为假时跳转到的基本块。

对于上述的方法，这里用 python 写了一个简单的例子，应该可以更好的表达上述意思。
```py
# short_circuit_evaluation.py

class CondExp:
    def get_first_block(self):
        pass

    def get_true_set(self):
        pass

    def get_false_set(self):
        pass


class AndSeq(CondExp):
    def __init__(self, cond_list: list[CondExp]):
        super().__init__()
        self.cond_list = cond_list

        self.__set_jump_target()

    def __set_jump_target(self):
        idx = 0
        while idx < len(self.cond_list) - 1:
            cond = self.cond_list[idx]
            true_set = cond.get_true_set()
            next_cond = self.cond_list[idx + 1]
            jump_target = next_cond.get_first_block()

            for true_block in true_set:
                true_block.iftrue = jump_target

            idx += 1

    def get_first_block(self):
        return self.cond_list[0].get_first_block()

    def get_true_set(self):
        return self.cond_list[-1].get_true_set()

    def get_false_set(self):
        false_set = set()
        for cond in self.cond_list:
            false_set.update(cond.get_false_set())
        return false_set

    def __repr__(self) -> str:
        return "(" + " && ".join([str(cond) for cond in self.cond_list]) + ")"


class OrSeq(CondExp):
    def __init__(self, cond_list: list[CondExp]):
        super().__init__()
        self.cond_list = cond_list

        self.__set_jump_target()

    def __set_jump_target(self):
        idx = 0
        while idx < len(self.cond_list) - 1:
            cond = self.cond_list[idx]
            false_set = cond.get_false_set()
            next_cond = self.cond_list[idx + 1]
            jump_target = next_cond.get_first_block()

            for false_block in false_set:
                false_block.iffalse = jump_target

            idx += 1

    def get_first_block(self):
        return self.cond_list[0].get_first_block()

    def get_true_set(self):
        true_set = set()
        for cond in self.cond_list:
            true_set.update(cond.get_true_set())
        return true_set

    def get_false_set(self):
        return self.cond_list[-1].get_false_set()

    def __repr__(self) -> str:
        return "(" + " || ".join([str(cond) for cond in self.cond_list]) + ")"


class Block(CondExp):
    def __init__(self, exp_str="exp"):
        self.iftrue = None
        self.iffalse = None
        self.exp_str = exp_str

    def get_first_block(self):
        return self

    def get_true_set(self):
        return {self}

    def get_false_set(self):
        return {self}

    def __repr__(self) -> str:
        return self.exp_str


if __name__ == "__main__":
    blocks = [
        Block("a < b"),
        Block("c > d"),
        Block("d > 10"),
        Block("d < 1"),
        Block("e > 10"),
    ]

    and_seq1 = AndSeq([blocks[2], blocks[3]])
    or_seq1 = OrSeq([blocks[0], blocks[1]])
    or_seq2 = OrSeq([and_seq1, blocks[4]])

    cond = AndSeq([or_seq1, or_seq2])
    print(cond)
    print()

    target_true = Block("True Stmt")
    target_false = Block("False Stmt")

    for true_block in cond.get_true_set():
        true_block.iftrue = target_true

    for false_block in cond.get_false_set():
        false_block.iffalse = target_false

    for block in blocks:
        print(f"{block}:")
        print(f"\ttrue  -> {block.iftrue}")
        print(f"\tfalse -> {block.iffalse}")
        print()

```

运行结果如下，可以看到程序正确得到了短路求值的跳转顺序。
```text
((a < b || c > d) && ((d > 10 && d < 1) || e > 10))

a < b:
        true  -> d > 10
        false -> c > d

c > d:
        true  -> d > 10
        false -> False Stmt

d > 10:
        true  -> d < 1
        false -> e > 10

d < 1:
        true  -> True Stmt
        false -> e > 10

e > 10:
        true  -> True Stmt
        false -> False Stmt
```

> 这一部分就不再手写 LLVM IR 了。阅读了上述内容后，短路求值只是一个体力活。

> 终于写完了...
