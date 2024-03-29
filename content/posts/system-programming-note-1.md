+++
title = "系统编程之命令行编译"
tags = ["C", "GCC", "Makefile", "Linux", "Shell"]
categories = ["操作系统"]
series = ["系统编程笔记"]
aliases = ["/posts/a5aebc7b"]
date = "2022-09-12T21:18:59+08:00"
+++

## 一、前言
本文将简单介绍在Linux系统下的命令行编译流程。介绍gcc、gdb、make等工具的简单使用。

## 二、GCC
### 基本操作
#### 编译选项
- 无选项编译链接
  - 用法：gcc test.c
  作用：将 test.c 预处理、编译、汇编并链接形成可执行文件。这里未指定输出文件，默认输出为 a.out。
- 选项 -o
  - 用法：gcc test.c -o test
  作用：将 test.c 预处理、编译、汇编并链接形成可执行文件 test。-o 选项用来指定输出文件的文件名。
- 选项 -E
  - 用法：gcc -E test.c -o test.i
  作用：将 test.c 预处理输出 test.i 文件。
- 选项 -S
  - 用法：gcc -S test.i
  作用：将预处理输出文件 test.i 编译成 test.s 文件。
- 选项 -c
  - 用法：gcc -c test.s
  作用：将汇编语言文件 test.s 汇编成目标代码 test.o 文件。
- 无选项链接
  - 用法：gcc test.o -o test
  作用：将目标代码文件 test.o 链接成最终可执行文件 test。
- 选项 -O
  - 用法：gcc -O1 test.c -o test
  作用：使用编译优化级别 1 编译程序。级别为 1~3，级别越大优化效果越好，但编译时间越长。

官方文档：[GCC, the GNU Compiler Collection](https://gcc.gnu.org/)

#### 搜索路径控制

- -I **dir**：将 **dir** 增加至头文件搜索路径
- -L **dir**：将 **dir** 增加至库文件搜索路径
- -l **library** 或 -l**library**：指定编译时搜索的库名

### 示例
#### 编译链接
在这样的文件结构下：
``` bash
.
├── include
│   └── dog.h
├── main.c
├── v1
│   └── dog.c
└── v2
    └── dog.c
```

- 将 `./main.c` 编译为 `./main.o`（仅编译）

  > 指令：gcc -I ./include -c main.c

- 将 `./v1/dog.c` 编译为 `./v1/dog.o`（仅编译）

  > 指令：gcc -I ./include -c ./v1/dog.c -o ./v1/dog.o

- 将 `./v2/dog.c` 编译为 `./v2/dog.o`（仅编译）

  > 指令：gcc -I ./include -c ./v2/dog.c -o ./v2/dog.o

- 将 `./v1/dog.o` 与 `./main.o` 链接为 `./dog1`

  > 指令：gcc main.o ./v1/dog.o -o dog1

- 将 `./v2/dog.o` 与 `./main.o` 链接为 `./dog2`

  > 指令：gcc main.o ./v2/dog.o -o dog2

#### 静态和动态库
执行下面的命令：
```shell
ar cr libdog.a ./v1/dog.o
gcc -o main main.o -L. -ldog
./main
```
> 第一条生成了一个名为libdog.a的静态库。`cr`意为“create and replace”，指如果库不存在则创建，如果存在则用新文件替换库中的同名文件。后接要生成的静态库名称，再之后接要添加入库中的文件。
  > 第二条将第一条中生成的静态库与main.o文件链接，生成一个名为main的可执行文件。`-L`后跟地址表示将该地址添加到库文件的搜索路径，`-L.`表示将当前目录添加到搜索路径。`-l`指定编译时搜索的库名。

再执行以下命令：

```sh
ar cr libdog.a ./v2/dog.o
./main
```
两次运行main可执行文件结果相同。
> 因为静态链接，程序在链接完成时就已确定。第二次只是更改了静态库本身，而没有改变链接后生成的可执行文件。

---
执行下面的命令

```shell
gcc -c -fPIC v1/dog.c -o v1/dog.o -I include
gcc -c -fPIC v2/dog.c -o v2/dog.o -I include
gcc -shared -fPIC -o libdog.so v1/dog.o
gcc main.c libdog.so -o main -I include
```
再将库文件路径设置为当前路径：

```sh
LD_LIBRARY_PATH=.
export LD_LIBRARY_PATH
```
执行 `./main`，之后再执行以下命令：

```sh
gcc -shared -fPIC -o libdog.so v2/dog.o
./main
```
两次运行main可执行文件结果不同。
> 引用动态库时，动态库的程序并不保存在可执行文件内部，而是在运行时才会动态地连接到程序中。

## 三、GDB
### 基本操作
- 在使用 gcc 对程序编译时，需要**加上-g 参数**（产生调试信息）才能使 GDB 进行调试。
- 输入 help 命令获得帮助
- 输入 quit 或者按 Ctrl+D 组合键退出 GDB。
- 启动程序准备调试方法
  - 方法一：在执行 GDB 命令时加上要调试的可执行程序名称，如“GDB yourprogram”；
  - 方法二：先输入 GDB，在 GDB 中输入 file yourprogram 加载需要调试的程序。最后使用 run 或者 r 命令开始执行，也可以使用 run parameter 方式传递参数

|   命令    | 命令缩写 |                                           命令说明                                           |
| :-------: | :------: | :------------------------------------------------------------------------------------------: |
|   list    |    l     |                                        显示多行源代码                                        |
|   break   |    b     |                            设置断点，程序运行到断点的位置会停下来                            |
|   info    |    i     |                                      描述程序运行的状态                                      |
|    run    |    r     |                                         开始运行程序                                         |
|  display  |   disp   |                           跟踪查看某个变量，每次停下来都显示它的值                           |
|   step    |    s     |                执行下一条语句，若该语句为函数调用，则进入函数执行其第一条语句                |
|   next    |    n     | 执行下一条语句，若该语句为函数调用，不会进入函数内部执行（即不会一步一步地调试函数内部语句） |
|   print   |    p     |                                         打印内部变量                                         |
| continue  |    c     |                               继续程序的执行直到遇到下一个断点                               |
|  set var  |          |                                         设置变量的值                                         |
|   start   |          |                         开始执行程序，在 main 函数第一条语句前面停下                         |
|   file    |          |                                      装入需要调试的文件                                      |
|   kill    |    k     |                                      终止正在调试的程序                                      |
|   watch   |          |                                       监视变量值的变化                                       |
| backtrace |    bt    |                                      查看函数调用的信息                                      |
|   frame   |    f     |                                           查看栈帧                                           |
|   quit    |    q     |                                        退出 GDB 环境                                         |

GDB 的更多使用方法可以参阅[**GDB User Manual**](http://sourceware.org/gdb/current/onlinedocs/gdb/) ([PDF](http://sourceware.org/gdb/current/onlinedocs/gdb.pdf))
                                                                            
### 示例
当前文件夹下有test.c文件：
```c
#include <stdio.h>
int main() {
    int num;
    do
    {
        printf("Enter a positive integer: ");
        scanf("%d", &num);
    } while (num < 0);

    int factorial;
    for (int i = 1; i <= num; i++)
        factorial = factorial * i;

    printf("%d! = %d\n", num, factorial);
    return 0;
}
```
接下来对该文件进行调试。
按顺序执行如下操作：

（1）执行以下命令进入调试状态
``` bash
gcc -g test.c -o test
gdb test
```
（2）在 main 函数处设置断点
``` bash
break main
```
（3）输入 `run` 命令开始程序
``` bash
Breakpoint 1, main () at test.c:2
2	int main() {
```
（4）多次输入 `next` 命令使程序运行到第 14 行,使用 print 命令打印 num 的值
``` bash
print num
```
``` bash
$1 = 2
```
（5）继续调试至程序第 15 行,使用 `print` 命令打印 factorial 的值
``` bash
print factorial
```
``` bash
$2 = -17088
```
（6）使用 `run` 命令再次调试程序

（7）在程序第 10 行加入断点
``` bash
break test.c:10
```
（8）使用 `continue` 命令使程序运行到断点处

（9）使用 `next` 命令

（10）再次使用 `print` 命令打印 i 和 factorial 的值

（11）使用 `p factorial=1` 命令改变 factorial 的值

（12）使用 `info locals` 查看所有局部变量值
``` bash
i = 1
num = 2
factorial = 1
```
（13）继续调试至程序结束

> 易知程序出错在没有初始化变量factorial

> gdb能很好地展示程序的执行过程，方便查找出错的位置。

## make
### 基本操作
#### Makefile的格式
Makefile文件由一系列规则（rules）构成。每条规则的形式如下。
``` bash
<target> : <prerequisites> 
[tab]  <commands>
```
第一行冒号前面的部分，叫做"目标"（target）；冒号后面的部分叫做"前置条件"（prerequisites）；第二行必须由一个tab键起首，后面跟着"命令"（commands）。

- 目标
  - 一个目标（target）就构成一条规则。目标通常是文件名，指明Make命令所要构建的对象，比如上文的 a.txt 。目标可以是一个文件名，也可以是多个文件名，之间用空格分隔。
  - 除了文件名，目标还可以是某个操作的名字，称为"伪目标"（phony target）。

- 前置条件
  - 前置条件通常是一组文件名，之间用空格分隔。它指定了"目标"是否重新构建的判断标准：只要有一个前置文件不存在，或者有过更新（前置文件的last-modification时间戳比目标的时间戳新），"目标"就需要重新构建。
  
- 命令
  - 命令（commands）表示如何更新目标文件，由一行或多行的Shell命令组成。它是构建"目标"的具体指令，它的运行结果通常就是生成目标文件。

#### 语法选述
- 注释
  - 井号（#）在Makefile中表示注释。

- 模式匹配
  - Make命令允许对文件名，进行类似正则运算的匹配，主要用到的匹配符是%。比如，假定当前目录下有 f1.c 和 f2.c 两个源码文件，需要将它们编译为对应的对象文件。
    ``` bash
    %.o: %.c
    ```
    等同于下面的写法。
    ``` bash
    f1.o: f1.c
    f2.o: f2.c
    ```
    使用匹配符%，可以将大量同类型的文件，只用一条规则就完成构建。

- 通配符
  - 通配符（wildcard）用来指定一组符合条件的文件名。Makefile 的通配符与 Bash 一致

- 依赖路径
  - 大写的VPATH或小写的vpath，表示搜索文件时的路径。写法例如`VPATH src:include:lib`表示在`./src` `./include` `./lib` 三个路径下寻找文件。`vpath %.c ./src`表示以.c结尾的文件在`./src`路径下寻找。
  
### 示例
当前文件夹下有如下结构：
``` bash
.
├── include
│   ├── dylib.h
│   ├── fun1.h
│   └── fun2.h
├── lib
│   └── libdy.so
├── Makefile
└── src
    ├── fun1.c
    ├── fun2.c
    └── main.c
```
将这些资源编译成一个可执行文件，在Makefile文件中写下：
```makefile
vpath %.h ./include
vpath %.so ./lib
vpath %.c ./src

main: main.o fun1.o fun2.o libdy.so
    gcc $^ -o main

%.o: %.c
    gcc $^ -I include -c

.PHONY: clean
clean:
    rm *.o
```
在终端中输入命令`make main`，即生成main文件（以及中间文件）。再输入`make clean`将中间文件删除。