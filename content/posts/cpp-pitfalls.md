+++
title = "C++ 踩坑记录（持续更新！）"
tags = ["C++"]
categories = ["编程语言"]
date = 2025-09-11T22:37:42+08:00
+++

有人说你永远不能自称精通 C++，本文试图为这个观点提供一个例证。下面列出了一些从去年（2024）开始我编写 C++ 代码时犯过的错误。当然，有些可能看上去很蠢，不过谁又能在未知全貌的时候保证自己不会犯错呢？我认为这些错误至少初看上去是反直觉的。

## std 集合操作只能用于有序容器

你要表示两个整数集合，所以你用了 `std::unordered_set<int>`。之后你想要求两个集合的交集，你搜了一下 STL，发现 `std::set_intersection` 似乎正合适。于是你写了一个简单的程序

```cpp
// test_set.cpp
std::unordered_set<int> set1 = {1, 2, 3, 4, 5};
std::unordered_set<int> set2 = {4, 5, 6, 7, 8};

std::unordered_set<int> result;
std::set_intersection(set1.begin(), set1.end(), set2.begin(), set2.end(),
                        std::inserter(result, result.begin()));
for (const auto &elem : result) {
    std::cout << elem << " ";
}
std::cout << std::endl;
```

然后运行，你期望输出的结果是 `4 5` 或是 `5 4`（毕竟你很严谨）。可是实际结果呢

```console
$ clang++ test_set.cpp -o test_set
$ ./test_set

```

什么都没有输出。

正如 `std::set` 实际上表示的是有序集合一样，`std::set_intersection` 实际上也是 `ordered_set_intersection`，只不过函数签名并不告诉你。


不过既然 `std::set_intersection` 已经定义在 `<algorithm>` 中了，那么也就不要期望其能有什么超出算法之外的魔法了。对于 `std::unordered_set`，还是老老实实用最笨的方法吧。

```cpp
std::unordered_set<int> result;
for (const auto& elem : set1) {
    if (set2.find(elem) != set2.end()) {
        result.push_back(elem);
    }
}
```

> 没有魔法，但是模板还是有的。std 集合操作可以用于有序迭代器。

## std::set 作为优先队列丢失元素

你需要这样一个数据结构，希望其内部有序，能够从头尾两端分别取出最大最小的元素，且能够随时插入新的元素。你很懒，所以没有想着自己实现一个数据结构。从 STL 里找来找去，你找到了 `std::set`，想着这是个好东西，从中取出元素和插入元素的时间复杂度都是 O(logn)。

你用 `std::set` 写了一小段代码

```cpp
// test_priority_set.cpp
using Entry = std::pair<int, std::string>;
auto cmp = [](const auto &a, const auto &b) { return a.first < b.first; };

std::set<Entry, decltype(cmp)> pq(cmp);
pq.emplace(3, "Task 1");
pq.emplace(1, "Task 2");
pq.emplace(1, "Task 3");
pq.emplace(2, "Task 4");

while (!pq.empty()) {
    auto it = pq.begin();
    const auto &[_, task] = *it;
    std::cout << task << std::endl;
    pq.erase(it);
}
```

运行一下呢

```console
$ ./test_priority_set
Task 2
Task 4
Task 1
```

`Task 3` 去哪了？

然后你才猛然想起来 `std::set` 并不是队列，而是集合。事后诸葛来看这个问题并不难发现，不过当我遇到这个问题时，它正藏在一个复杂的算法之中。

修复的方式很显然，用 `std::multiset` 代替 `std::set`。

## 引用并非对象

引用并非对象。对象是什么呢？对象是存储在一块内存区域中的东西。

当然，从实现的角度来看，引用总是需要空间来存储的，它和指针没有本质上的区别。但是在 C++ 的语义中，引用并不是对象。你没法访问引用本身，而只能访问引用所引用的对象。

这就意味着你没法存储一个引用。比如说 `std::vector<int&>` 就是非法的。

如果你实在想要存引用的话要怎么办？那当然是用指针了。当然，容器通常是所有权转移的好地方，所以一定要注意引用的生命周期。

除了用指针，还可以用 `std::reference_wrapper`。比如说 `std::vector<std::reference_wrapper<int>>`。但是在应用程序的代码中使用这个并没有意义，因为这个类型只有 `get()` 方法，用于获取对应指针。

`std::reference_wrapper` 是给模板用的，其目的是避免所有权的转移。

比如说我们编写了一个用于构建闭包的函数

```cpp
template <typename Fn, typename... Captured>
auto build_closure(Fn &&fn, Captured &&...captured) {
    return [fn = std::forward<Fn>(fn),
            ... captured = std::forward<Captured>(captured)](auto &&...args) mutable {
        fn(captured..., std::forward<decltype(args)>(args)...);
    };
}
```

之后我们编写一个函数 `func`，该函数传入一个非 `const` 引用，并在函数中修改传入参数的取值。我们将该函数和其参数构建为一个闭包并调用。

```cpp
void func(int& v) { v += 1; }

int a = 1;
auto closure = build_closure(func, a);
closure(); // a = 1
```

按照直觉，调用之后 `a` 的取值应该是 `2`。可实际上其取值依然为 `1`。因为在 `build_closure` 中进行了 `int` 的复制构造。closure 中的 `a` 和原来的 `a` 并不是同一个对象。

因此这里需要使用 `std::ref` 来把 `a` 包装成一个 `std::reference_wrapper`。此时 `build_closure` 中将进行 `std::reference_wrapper<int>` 的构造。

```cpp
int a = 1;
auto closure = build_closure(func, std::ref(a));
closure(); // a = 2
```

问题是这里的 `std::reference_wrapper` 和指针有什么区别呢？

如果我们选择传递指针，编译器将会报错。因为不存在从 `int *` 到 `int &` 的隐式转换。在不使用 `std::reference_wrapper` 的情况下，只有修改函数签名，接受 `int *` 才能实现引用的传递。

```cpp
int a = 1;
// int& b = &a; // Wrong
int &c = std::ref(a); // Right
```

## `std::vector<move_only>` 是 copy constructible 的

你刚开始学习模板，所以想写一个简单的模板类。类似下面这样

```cpp
template <typename T> struct Wrapper {
    T value;
};
```

你当然不满足于此，你想再定义一个 `Wrapper<T>::create` 函数。你吧这个函数的签名定义成如下这样

```cpp
template <typename T> struct Wrapper {
    T value;

    static Wrapper<T> create(T v) {
        // ...
    }
};
```

然后你就会发现一个问题，`T` 可能是 move-only 的，也可能是 copy-only 的。你没法确定。不过你很快就找到了解决这一问题的魔法。那就是 `<type_traits>` 头文件。你使用 `std::is_copy_constructible` 来确定类型 `T` 是否是可复制的。如果是，那么使用复制语义，否则使用移动语义。

```cpp
template <typename T> struct Wrapper {
    T value;

    static Wrapper<T> create(T v) {
        if constexpr (std::is_copy_constructible<T>::value) {
            return {v};
        } else {
            return {std::move(v)};
        }
    }
};
```

你试了几个例子

```cpp
auto a = Wrapper<int>::create(42);
auto b = Wrapper<std::unique_ptr<int>>::create(std::make_unique<int>(42));
auto c = Wrapper<std::vector<int>>::create(std::vector<int>{});
```

嗯，都能完美编译。但是下面这个呢

```cpp
auto d = Wrapper<std::vector<std::unique_ptr<int>>>::create(std::vector<std::unique_ptr<int>>{});
```

嗯？为什么会报错？

从报错信息来看，你会发现 `std::vector<std::unique_ptr<int>>` 竟然被 `std::is_copy_constructible` 判定为真！？

具体原理我也不太清楚，总之不要信任 `std::is_copy_constructible` 的判定结果。

> [这里](https://stackoverflow.com/questions/71632098/why-is-stdis-copy-constructible-vstdvectormoveonlytype-true)有答案，但是我不太想看。

至于 `Wrapper<T>::create` 的实现，你应该使用通用引用。

```cpp
template <typename T> struct Wrapper {
    T value;

    template <typename U = T> static Wrapper<T> create(U &&v) {
        return {std::forward<U>(v)};
    }
};
```

## `std::future::wait` 不会转发异常

你有一个异步的任务

```cpp
std::future<void> fut = std::async(std::launch::async, task);
```

你希望用如下操作令其退出。似乎没有问题，是吧？

```cpp
stop_flag = true
auto status = fut.wait_for(std::chrono::seconds(1));
assert(status != std::future_status::deferred);
if (status == std::future_status::timeout) {
    std::cerr << "timeout" << std::endl;
    std::terminate();
}
```

当然，在正常的代码路径下这样做当然没有问题。但是如果 task 将会抛出一个异常呢？

例如，你的 `task` 函数在实现上存在问题。就像是下面这样

```cpp
void task() {
    while (!stop_flag) {
        std::stoi("not a number");
    }
}
```

> 会抛出 `std::invalid_argument`。

在测试的时候，你肯定希望代码在运行时不会抛出任何未处理的异常。你的测试代码（如果有的话）可能有类似下面的内容

```cpp
try {
    stop_flag = true
    auto status = fut.wait_for(std::chrono::seconds(1));
    assert(status != std::future_status::deferred);
    if (status == std::future_status::timeout) {
        std::cerr << "timeout" << std::endl;
        std::terminate();
    }
} catch (const std::exception& e) {
    assert(false);
}
```

但是实际运行时，你会发现上述代码并不会抛出异常。

> 你可能需要在设置 `stop_flag` 和进行 `wait` 之间增加一段时间的等待。

这可是个问题。当然，是你没有认真阅读文档的问题。（不过谁又能一直认真呢？）

`std::future::wait*` 的语义是等待，直到该 `future` 完成。除此之外并不会做任何操作，比如访问 `future` 的取值，或者抛出导致 `future` 完成的异常。

完成 `wait` 所没有做的操作的方案是 `std::future::get`。这里需要注意 `std::future<void>` 的特化。其 `get` 函数不会返回取值，但如果 `future` 中包含一个异常，则会将该异常抛出。

> 有意思的是 `get` 的语义不止包括获取 `future` 的取值，还包括一直阻塞，直到 `future` 完成。

## `size_t` 无符号

`size_t` 是无符号整数。这很自然，因为其至少要能表示地址空间的大小，而地址空间是 32 位或 64 位的。

因为 `size_t` 有明确的语义（用于表示数量），所以大多数情况下我们喜欢在循环中使用 `size_t`。

```cpp
for (size_t i = 0; i < n; i++)
    // ...
```

偶尔我们也需要用到反向的循环

```cpp
for (size_t i = n - 1; i >= 0; i--)
    // ...
```

但是这样就出错了。因为 `size_t` 是无符号的，所以其取值永远 `>= 0`，上面的代码变成了死循环。

这时应该换用 `ssize_t`。当然，你要保证 `n - 1` 在 `ssize_t` 的取值范围内。不过尽管范围减半，但是让 `n` 溢出 `ssize_t` 的场景还是很少的。

```cpp
for (ssize_t i = n - 1; i >= 0; i--)
    // ...
```

## 未完待续

就是未完待续