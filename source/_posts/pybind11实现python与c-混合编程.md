---
title: pybind11实现python与c++混合编程
tags:
  - pybind11
  - c++
  - python
categories: 学习笔记
abbrlink: 71181fb7
date: 2023-08-17 10:42:38
---
## 一、前言
最近在尝试写一个简单的[游戏引擎](https://github.com/wokron/Gamo)，我决定用 python 作为脚本，所以了解了一些混合编程的知识。

### （1）python api
从原理来说，根据[文档](https://docs.python.org/3/extending/extending.html)所述，python 提供了 `Python.h` 头文件，能够将 c 或 c++ 代码编译成可供 python 引入的动态链接库。该库中定义的可供 python 调用的函数中所有的入参都是名为 `PyObject` 结构体的指针。在代码中可以通过一系列函数对 `PyObject` 进行操作。

举一个简单的例子，我们希望 python 调用一个由 c 编写的简单的加法函数
```c
int add(int a, int b) {
    return a + b;
}
```

<!-- more -->

我们期望在 python 中这样调用
```py
# test_mymodule.py
from mymodule import add
a = 10
b = 20
c = add(a, b)
assert c == 30
```

那么我们首先需要对该函数进行包装，包装函数 `_add` 的参数和返回值都应该是 `PyObject *`。在包装函数中调用了 `PyArg_ParseTuple` 将传入的参数转换为 `int` 类型，调用原本的 `add` 函数得到返回值，之后又通过 `PyLong_FromLong` 将 `int` 转换为 `PyObject`。
```c
// mymodule.c
#include <Python.h>

int add(int a, int b) {
    return a + b;
}

static PyObject *_add(PyObject *self, PyObject *args) {
    int _a, _b, rt;

    if (!PyArg_ParseTuple(args, "ii", &_a, &_b)) {
        return NULL;
    }

    rt = add(_a, _b);
    return PyLong_FromLong(rt);
}
```

在此之后还需要定义函数和模块，并定义初始化函数
```c
// mymodule.c
// omit...

static PyMethodDef MyModuleMethods[] = {
    {
        "add",
        _add,
        METH_VARARGS,
        "a simple add function"
    },
    {NULL, NULL, 0, NULL}
};

static struct PyModuleDef mymodule = {
    PyModuleDef_HEAD_INIT,
    "mymodule",
    "this is my simple module", 
    -1,
    MyModuleMethods
};

PyMODINIT_FUNC PyInit_mymodule(void) {
    return PyModule_Create(&mymodule);
}
```

最后构建并测试
```sh
# build_test.sh
#!/bin/bash
PYTHON_VER=3.10
MODULE_NAME=mymodule
TEST_FILE=test_mymodule.py

gcc -fPIC -shared ${MODULE_NAME}.c -o ${MODULE_NAME}.so -I/usr/include/python${PYTHON_VER}/ -lpython${PYTHON_VER}

python ${TEST_FILE} || python3 ${TEST_FILE} || ! echo "fail to finish test" || exit
echo "test pass"
```

可是这种直接使用原始 api 的方式太过繁琐了，并不推荐。庆幸的是已经有库封装了这一过程，提供了更简便的实现方法。这就是 pybind11。

### （2）用 pybind11 重写
首先下载 pybind11 库，这里使用 git 的 submodule 的方式。
```sh
git submodule add -b stable https://github.com/pybind/pybind11 extern/pybind11
git submodule update --init
```

之后在项目根目录的 CMakeLists.txt 中写入如下内容，源代码文件为 mymodule.cpp
```c
cmake_minimum_required(VERSION 3.4...3.18)
project(example LANGUAGES CXX)

add_subdirectory(extern/pybind11)
include_directories(extern/pybind11/include)

pybind11_add_module(mymodule mymodule.cpp)
```

此时要实现之前例子中的加法函数就很简单了。
```cpp
// mymodule.cpp
#include <pybind11/pybind11.h>
namespace py = pybind11;
using namespace pybind11::literals;

int add(int a, int b) {
    return a + b;
}

PYBIND11_MODULE(mymodule, m) {
    m.doc() = "this is my simple module";
    m.def("add", &add, "a simple add function", "a"_a, "b"_a);
}
```

当然本文不会详细解释 pybind11 的所有内容，只是列出一些自己用到的方法罢了，详细内容还请参考 [pybind11](https://pybind11.readthedocs.io/en/stable/index.html) 文档。

## 二、函数与结构体
在 pybind11 中，每个模块在一个 `PYBIND11_MODULE` 宏所指定的区域内定义，其中第一个参数是模块名，第二个参数是模块 handler 变量的名称，为 m 即可。
```cpp
PYBIND11_MODULE(module_name, m) {
    // omit...
}
```

函数的绑定很简单，只需要使用 `m.def()` 即可。其中第一个参数是在 python 中调用所使用的函数名，第二个参数是 c++ 代码中要绑定的函数的函数指针，之后还可以添加对函数的描述、变量名、默认值等等内容，这里不再详述。
```cpp
PYBIND11_MODULE(mymodule, m) {
    m.def("add", &add, "a simple add function", "a"_a, "b"_a);
}
```

结构体和类没有本质区别，因此在 pybind11 中都是使用 `pybind11::class_<T>()` 定义。比如说这里有一个结构体。
```cpp
struct Pet {
    std::string name;
    const std::string &getName() const { return name; }
};
```

可以将该结构体绑定到 python
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<Pet>(m, "Pet");
}
```

但是此时 Pet 在 Python 看来不过是一个没有任何属性和方法的类。使用 `class_<T>().def_readwrite()` 定义可读可写的属性、使用 `class_<T>().def()` 定义成员方法。
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<Pet>(m, "Pet")
        .def_readwrite("name", &Pet::name)
        .def("get_name", &Pet::getName);
}
```

> 在 pybind11 中可以使用 lambda 表达式作为函数，此时需要注意，当 lambda 表达式作为成员方法时，需要让第一个参数为类的实例，例如
> ```cpp
>   py::class_<Pet>(m, "Pet")
>       .def("pat", [](const Pet &self, int times){
>           return "pat pet" + self.getName() + std::tostring(times) + "times";
>       })
> ```

## 三、面向对象
### （1）封装
面向对象编程中经常将属性设为 `private`，而使用 getter、setter 方法对值进行读写。c++ 和 java 并没有提供语法糖，可 python 与 c# 类似，提供了 `property` 简化这一过程。

> 当然 python 的这一点可能有些人并不知道，可以尝试一下如下代码
> ```py
> class Pet:
>     def __init__(self, name):
>         self.__name = name
> 
>     @property
>     def name(self):
>         return self.__name
>     
>     @name.setter
>     def name(self, name):
>         if name == self.__name:
>             print("name cannot be same")
>         else:
>             self.__name = name
> 
> p = Pet("tom")
> assert p.name == "tom"
> 
> p.name = "kitty"
> assert p.name == "kitty"
> 
> p.name = "kitty"
> ```

pybind11 中提供了将 c++ 中的 getter、setter 函数改为 python 的 property 的方法。使用 `class_<T>().def_property()`
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<Pet>(m, "Pet")
        .def_property("name", &Pet::getName, &Pet::setName);
}
```

### （2）继承
比如说现在想要为 `Pet` 类增加一个子类 `Dog`。
```cpp
struct Dog : Pet {
    std::string bark() const { return "woof!"; }
};
```

那么在子类绑定的时候就要声明 Pet 为 Dog 的父类，注意此处 `class_<Dog, Pet>`
```cpp
py::class_<Dog, Pet>(m, "Dog")
        .def("bark", &Dog::bark);
```

对于多继承的情况，只需要在同样的位置添加多个父类即可。即 `class_<Child, Father1, Father2, Father3>` 的形式。

大多数情况，只需要父类绑定其方法，子类也能够调用父类所绑定的方法。但是当父类为抽象类的时候就不同了，此时并不能绑定一个不存在的方法。这时需要使用一个辅助函数来实现，具体方法可见[此处](https://pybind11.readthedocs.io/en/stable/advanced/classes.html#overriding-virtual-functions-in-python)

### （3）多态
这里的多态主要指函数重载，分为两个方面。一是如何指定 c++ 侧重载的函数、二是如何在 python 侧实现重载。当然因为有了 pybind11 的帮助，这两点也很简单。

c 语言中是没有函数多态的，因此函数名就是函数地址，但是 c++ 中就不同了，并不能通过函数名明确到底是哪一个函数。pybind11 中提供了根据函数签名获取地址的方法。这需要使用模板 `pybind11::overload_cast<T1, T2, ...>()`。其中 `T1, T2, ...` 是函数参数的类型，之后提供函数地址。例如
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<Pet>(m, "Pet")
        .def("get_name", py::overload_cast<>(&Pet::Name))
        .def("set_name", py::overload_cast<std::string>(&Pet::Name));
}
```

python 中同样是没有函数多态的，因为函数重载的目的就是实现可变参数个数和可变参数类型，而 python 使用 `*args` 和 `**kwargs` 已经从另一个角度解决了这一问题。不过不要担心， pybind11 已经为我们做出处理，我们并不需要关心这一问题。我们只需要大胆的使用相同的函数名即可。在进行调用时，pybind11 会尝试调用所有同名函数，直到成功调用并成功返回或找不到匹配的函数报错。
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<Pet>(m, "Pet")
        .def("name", py::overload_cast<>(&Pet::Name))
        .def("name", py::overload_cast<std::string>(&Pet::Name));
}
```

## 四、单例模式
说实话这不能算是一节内容，但是在 python 中绑定单例类还是很有意思的。首先写一个简单的单例：
```cpp
class MySingleton {
public:
    static MySingleton *GetInstance()
    {
        static MySingleton _instance;
        return &instance;
    }

protected:
    MySingleton() {}
    ~MySingleton() {}
public:
    MySingleton(MySingleton const &) = delete;
    MySingleton &operator=(MySingleton const &) = delete;
};
```

第一种方法当然是绑定静态方法，注意这里需要加上 `std::unique_ptr<MySingleton, py::nodelete>` 不引用析构函数。
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<MySingleton, std::unique_ptr<MySingleton, py::nodelete>>(m, "MySingleton")
        .def_static("get_instance", &MySingleton::GetInstance);
}
```

第二种方法是这样，因为 pybind11 中可以使用 lambda 自定义构造函数，所以可以将 `GetInstance` 包装成构造函数使用。这样可以在 python 中构造“不同”的实例，但这些实例其实都指向一个单例。
```cpp
PYBIND11_MODULE(mymodule, m) {
    py::class_<MySingleton, std::unique_ptr<MySingleton, py::nodelete>>(m, "MySingleton")
        .def(py::init([](){ return MySingleton::GetInstance(); }));
}
```
