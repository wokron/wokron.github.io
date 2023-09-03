---
title: CMake实用语法教程
tags:
  - cmake
categories: 教程
abbrlink: 38f6cfcb
date: 2023-09-03 11:00:23
---
## 一、前言
最近一段时间在用 c++ 写一个项目，因此学了学 cmake。说实话，cmake 奇怪的语法在一开始实在容易让人望而生畏。但是上手使用的话就会发现平常会用到的不过是其中的一小部分，并且通常有规律可循。掌握这一部分的内容，大概率就可以组织起一个规模较大的项目了。因此本文也就旨在讲述 cmake 的这一部分的内容。

当然，阅读本教程之前需要了解代码编译、链接的相关知识。关于编译相关的命令，可见我的文章[系统编程之命令行编译](https://wokron.github.io/posts/a5aebc7b/)。

本文的所有代码保存在仓库 [practical-cmake](https://github.com/wokron/practical-cmake) 中，欢迎 star `:)`。

<!-- more -->

cmake 下载方式如下（apt）
```sh
sudo apt-get install build-essential
sudo apt-get install cmake
```

## 二、第一步
说到第一个程序，那当然要请出经典的 `hello, world` 了。
```c
#include <stdio.h>

int main()
{
    printf("hello, world\n");
    return 0;
}
```

在本文中我会首先给出使用 gcc 编译的命令，之后再使用 cmake 做同样的事情。那么对于第一步，我们的 gcc 命令如下
```sh
gcc main.cpp -o main
```

当然很简单，而对于 cmake 也类似。要使用 cmake，我们需要添加一个配置文件 CMakeLists.txt，其中包含要执行的操作。本小节中，CMakeLists.txt 的内容是
```cmake
cmake_minimum_required(VERSION 3.10)

project(main)

add_executable(main main.cpp)
```

其中第一条指定了 cmake 版本要求，第二条指定了当前项目名，而第三条 `add_executable` 则实现了和 gcc 命令相同的操作：指定源文件 `main.cpp` 和输出文件名 `main`，生成一个可执行文件。

要进行编译，需要首先使用 `cmake` 命令生成构建文件，这会在当前目录下生成包含 Makefile 在内的许多配置文件。之后再通过 `make` 指令即可完成编译。

为了不影响源文件的结构，我们可以选择新建一个文件夹执行 `cmake` 命令。假设现在位于项目根目录，那么执行如下命令
```sh
mkdir build
cd build
cmake ..
```
即可生成构建文件。

随后执行 `make` 命令完成编译并运行。
```sh
make
./main
```

仅就这一节看来，cmake 和 gcc 并没有什么差距，反而步骤更加繁琐，但是随着项目结构越发复杂，cmake 的优势就会越发明显。

## 三、头文件的引入
大部分实际的项目都不会只有一个文件。如果有多个源文件，编译器可以将其编译成同一个 .o 文件，源文件间的符号通过头文件共享。下面我们举如下的项目为例。
```text
.
|-- CMakeLists.txt
|-- include
|   `-- add.h
`-- src
    |-- add.cpp
    `-- main.cpp
```

add.h 中声明了函数 `int add(int a, int b)`，并在 add.cpp 中实现，而 main.cpp 则调用了该函数，因此需要引入头文件 add.h 获得该函数的声明。

如果使用 gcc，可以使用如下命令编译。注意此处需要通过命令行参数 `-I` 设定头文件所在路径。
```sh
gcc src/add.cpp src/main.cpp -I ./include -o main
```

对于 cmake 来说，配置如下
```cmake
cmake_minimum_required(VERSION 3.10)

project(main)

add_executable(main src/main.cpp src/add.cpp)
target_include_directories(main PRIVATE include)
```
可以看出 `add_executable` 指令之外，我们新增加了一条指令 `target_include_directories`，这条命令将路径 ./include 添加到生成目标文件 main 所需的头文件路径中。

> 注意 `target_include_directories` 中除了目标和 include 路径外，还有一个符号 `PRIVATE`，该符号指定了头文件的作用范围，`PRIVATE` 表示头文件只用于当前的目标，而不用于链接当前目标的文件，类似还有 `PUBLIC`，将头文件向外暴露，和 `INTERFACE`，目标不使用，但向外暴露。当然现在不理解没有关系，在之后我们还会遇到。

## 四、库的编译和链接
有时候可能需要将一个模块编译成一个库，以便用于不同的程序。我们还以前一小节所用的代码为例，但这次我们重新组织代码结构。因为 main 模块调用了 add 模块提供的函数，所以我们可以直接把 add 模块编译成一个库，再将该库与 main 链接。

因为要将 add 作为一个库，所以这里将 add 移到了一个文件夹中。
```text
.
|-- add
|   |-- include
|   |   `-- add
|   |       `-- add.h
|   `-- src
|       `-- add.cpp
|-- CMakeLists.txt
`-- main.cpp
```
> 这里有一个技巧，include 的路径下并不直接放头文件，而是在其中增加一层以库名为名的文件夹，这样引入的时候就需要使用 `#include <add/add.h>` 而不是 `#include <add.h>`，可以避免命名冲突。

gcc 可以选择编译为静态库或动态链接库，对于静态库，使用如下的命令创建 add 模块的静态库
```sh
gcc -c add/src/add.cpp -I add/include # just compile add
ar cr libadd.a add.o # create a static lib add (this name is libadd.a)
```

之后还需要将库文件与程序链接起来
```sh
gcc main.cpp -I add/include -L . -l add -o main
```

同理，对于动态库，有
```sh
gcc -c -fPIC add/src/add.cpp -I add/include -o add.o
gcc -shared -fPIC add.o -o libadd.so
```

之后进行链接
```sh
gcc main.cpp libadd.so -I add/include -o main
```

执行时还需要设定动态库位置
```sh
LD_LIBRARY_PATH=.
./main
```

> 可以看出，随着项目越来越复杂，我们为了完成编译需要进行的操作越来越繁琐了。一般的情况下，多个命令可以写成 shell 脚本；如果依赖关系更加复杂，则需要 makefile 管理编译环节；但如果项目更加庞大，使用 makefile 也需要编写大量代码的话，cmake 之类更加高级的工具就是必然的选择了。

对于 cmake 来说，库的编译和链接同样简单。我们需要引入两条新的指令。
```cmake
cmake_minimum_required(VERSION 3.10)

project(main)

add_library(add add/src/add.cpp)
target_include_directories(add PUBLIC add/include)

add_executable(main main.cpp)
target_link_libraries(main PRIVATE add)
```
注意这里对于 add.cpp，我们不再使用 `add_executable` 而是 `add_library`。默认这将创建一个动态库（对于静态库，添加 `STATIC` 符号即可 `add_library(add STATIC add/src/add.cpp)`）。之后我们需要将该库与 main 模块链接，这需要使用 `target_link_libraries`，和引入头文件时类似，只不过这次参数不是路径而是库名。不论是静态库还是动态库，cmake 都使用统一的方法进行处理，实际上简化了流程。

> 其实还有另一种“库”，就是只包含头文件的库，使用如下方式定义
> ```cmake
> add_library(headerlib INTERFACE)
> target_include_directories(headerlib INTERFACE headerlib/include)
> ```

> 注意这里出现了 `PUBLIC` 和 `PRIVATE`。这次 add.h 被 main.cpp 引用，因此需要使用 `PUBLIC`。试想一下，将 add.cpp 中的 `#include <add/add.h>` 去掉（因为 add.cpp 中定义了 add 函数，所以可以去掉），那么这里就可以使用 `INTERFACE` 或 `PUBLIC`。

## 五、项目组织——子目录
有时候如果你有一个规模较大的项目，将所有源文件堆在一起就不是一个很好的决定了。你可能需要拆分代码，将实现同一功能的源文件放在同一个文件夹中，并将其编译成一个库以便项目其他部分调用。但是一个项目拥有很多模块，如果所有模块的编译操作都在一个 CMakeLists.txt 文件中编写，则这个文件必然变得臃肿杂乱。合适的方法是在子目录中添加 CMakeLists.txt 文件，让该文件管理其目录下的编译操作，再将所有子目录合并到一起。

在 cmake 中只需要一条语句就可以完成子目录的添加，那就是 `add_subdirectory`。为了便于理解，我们以如下的项目作为例子。
```text
.
|-- add
|   |-- CMakeLists.txt
|   |-- include
|   |   `-- add
|   |       `-- add.h
|   `-- src
|       `-- add.cpp
|-- CMakeLists.txt
|-- fibo
|   |-- CMakeLists.txt
|   |-- include
|   |   `-- fibo
|   |       `-- fibo.h
|   `-- src
|       `-- fibo.cpp
`-- main.cpp
```

在这个项目中，我们添加了一个 fibo 模块，用于计算斐波那契数，该模块使用 `add` 函数来完成加法运算。而 main.cpp 现在则添加了一个循环用来计算 1-10 的斐波那契数并输出。因此模块间有如下的引用关系：
```text
fibo -> add
main -> add
main -> fibo
```

我们在 add 和 fibo 文件夹中都添加了 CMakeLists.txt 用于模块内的编译操作。
```cmake
# add/CMakeLists.txt
add_library(add src/add.cpp)
target_include_directories(add PUBLIC include)
```

```cmake
# fibo/CMakeLists.txt
add_library(fibo src/fibo.cpp)
target_include_directories(fibo INTERFACE include)
target_link_libraries(fibo PRIVATE add)
```

而在项目路径下的 CMakeLists.txt 中使用 `add_subdirectory` 将 add 和 fibo 路径设定为子路径。这样 cmake 便会执行 add 和 fibo 的 CMakeLists.txt 中的指令。
```cmake
cmake_minimum_required(VERSION 3.10)

project(main)

add_subdirectory(add)
add_subdirectory(fibo)

add_executable(main main.cpp)
target_link_libraries(main PRIVATE add fibo)
```

> 再一次，注意 `PUBLIC`、`INTERFACE` 和 `PRIVATE` 的使用，fibo 的源代码中没有引用 fibo.h 头文件，因此使用 `INTERFACE`；fibo 模块引用了 add 模块，但又不想向外提供 add 模块的接口，因此使用 `PRIVATE`（这时如果从 `target_link_libraries(main PRIVATE add fibo)` 中删除 add 则会报错）。

使用 `add_subdirectory` 之后，项目结构多么清晰！

## 六、输入输出、变量与流程控制
有时候你可能想要以多种不同的方式编译项目，比如说选择只编译某些模块或者选择是否编译单元测试。因此你希望在编译的时候输入某些参数选项，而 cmake 也确实提供了这一功能。

cmake 实际上提供了输入、输出、变量和流程控制，像是一门编程语言。当然这一部分并不会深入，因为那样的话这篇文章就不能称为“实用”了。

> cmake 中的输入输出、变量与流程控制发生在调用 `cmake` 命令时。

### 输入输出

cmake 中使用 `option` 设置布尔参数作为输入（当然也可以使用 set cache string 设置字符串参数，但不在本文范围内），使用 `message` 输出字符串，其中也可以包含参数。如下的例子中展示了具体的用法。
```cmake
cmake_minimum_required(VERSION 3.10)

project(io)

option(OPTION_VAR "this is help text" OFF)

message("the value of OPTION_VAR is ${OPTION_VAR}")
```

### 变量
用户也可以使用 `set` 设置变量。语法为 `set(name "value")`。另外 cmake 也内置了一些变量，如项目名 `PROJECT_NAME`、项目根路径 `CMAKE_SOURCE_DIR`、当前路径 `CMAKE_CURRENT_DIR` 等等。比如如下的例子
```cmake
cmake_minimum_required(VERSION 3.10)

project(variable)

set(USER_VAR "default")

message("the value of USER_VAR is ${USER_VAR}")

message("the name of project is ${PROJECT_NAME}")

message("the whole project's dir is ${CMAKE_SOURCE_DIR}")
```

运行后应该能看到类似下面的输出
```text
the value of USER_VAR is default
the name of project is variable
the whole project's dir is /home/wokron/Code/Projects/practical-cmake/variable
```

因为有了这些内置变量，我们也可以将之前的 CMakeLists.txt 继续化简，如用 `${PROJECT_NAME}` 替换特定的目标名。

### 流程控制
有了变量当然要有对变量的比较，cmake 提供了分支控制的方法。这里就只介绍一下分支控制。通过例子就可以理解分支控制的语法了。
```cmake
cmake_minimum_required(VERSION 3.10)

project(controlflow)

option(OPTION_VAR "this is option" ON)

if(OPTION_VAR)
    message("OPTION_VAR is on")
else()
    message("OPTION_VAR is off")
endif()

set(STR_VAR "123")

if(STR_VAR MATCHES "123")
    message("is 123!")
elseif(STR_VAR MATCHES "456")
    message("is 456")
else()
    message("is other :(")
endif()
```

通过参数和分支，就可以使我们项目的编译更加灵活，适应更复杂的情况。

### 增加子目录的情况
在使用 `add_subdirectory` 时，cmake 究竟发生了什么？其实很简单，cmake 只是去执行子目录中的指令，等执行完成后再跳转回原本的位置而已。这一点实在类似于函数，因为子目录中的 CMakeLists.txt 中可能包含 `option`，而父目录中的 CMakeLists.txt 也可以通过使用 `option` 设置同名参数值来指定子目录的编译选项。这一点在引入第三方库的时候十分有用。如下的两个 CMakeLists.txt 实例可以很好的模拟这种情况。
```cmake
# ./CMakeLists.txt
cmake_minimum_required(VERSION 3.10)

project(subdirectory)

message("this is ${PROJECT_NAME}")

option(EXTERN_OPTION "" ON)
add_subdirectory(extern)
```

```cmake
# ./extern/CMakeLists.txt
cmake_minimum_required(VERSION 3.8)

project(extern_module)

option(EXTERN_OPTION "this is option in extern" OFF)

message("this is ${PROJECT_NAME}")

if(EXTERN_OPTION)
    message("will do something")
else()
    message("will do other things")
endif()
```

## 七、测试
cmake 提供了一个简单的测试功能。想要使用首先需要开启测试，这需要在根 CMakeLists.txt 中加入 `enable_testing`
```cmake
cmake_minimum_required(VERSION 3.10)

project(main)

enable_testing() # here!!!

add_subdirectory(add)
add_subdirectory(fibo)
add_subdirectory(test)

add_executable(main main.cpp)
target_link_libraries(main PRIVATE add fibo)
```

之后我们需要创建用于测试的可执行文件，并使用 `add_test` 指令将其加入到测试中。这里我们选择使用一个新的子目录存储测试文件，当然也可以在其他位置，如每个模块目录内。test 目录下的 CMakeLists.txt 文件内容如下
```cmake
add_executable(test_add test_add.cpp)
target_link_libraries(test_add add)

add_executable(test_fibo test_fibo.cpp)
target_link_libraries(test_fibo fibo)

add_test(NAME test_add COMMAND test_add)
add_test(NAME test_fibo COMMAND test_fibo)
```

以及测试文件的内容，这里就只是简单编写一下
```cpp
// test_add.cpp
#include "add/add.h"
#include <assert.h>

int main()
{
    assert(add(1, 2) == 3);
    assert(add(1, -1) == 0);
    assert(add(1, -2) == -1);
    assert(add(100, 100) == 200);
    return 0;
}
```

```cpp
// test_fibo.cpp
#include "fibo/fibo.h"
#include <assert.h>

int main()
{
    assert(fibonacci(1) == 1);
    assert(fibonacci(2) == 1);
    assert(fibonacci(3) == 2);
    assert(fibonacci(4) == 3);
    assert(fibonacci(5) == 5);
    assert(fibonacci(6) == 8);
    assert(fibonacci(7) == 13);
    assert(fibonacci(8) == 21);
    assert(fibonacci(9) == 34);
    assert(fibonacci(10) == 55);
    return 0;
}

```

之后在 build 目录下执行如下命令
```sh
cmake ..
make
make test
```

输出结果如下，测试成功！！！
```text
Test project /home/wokron/Code/Projects/practical-cmake/build/test
    Start 1: test_add
1/2 Test #1: test_add .........................   Passed    0.00 sec
    Start 2: test_fibo
2/2 Test #2: test_fibo ........................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 2

Total Test time (real) =   0.00 sec
```

最终，我们的项目结构如下
```text
.
|-- add
|   |-- CMakeLists.txt
|   |-- include
|   |   `-- add
|   |       `-- add.h
|   `-- src
|       `-- add.cpp
|-- CMakeLists.txt
|-- fibo
|   |-- CMakeLists.txt
|   |-- include
|   |   `-- fibo
|   |       `-- fibo.h
|   `-- src
|       `-- fibo.cpp
|-- main.cpp
`-- test
    |-- CMakeLists.txt
    |-- test_add.cpp
    `-- test_fibo.cpp
```
