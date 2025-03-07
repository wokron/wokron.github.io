+++
title = "写个编译期排序"
tags = ["C++"]
categories = ["编程语言"]
date = 2025-03-07T16:17:03+08:00
+++

这回摆弄一下模板。

C++ 里有一个 `std::integer_sequence`。可以定义编译期整数序列。比如说

```cpp
#include <utility>
using my_seq = std::integer_sequence<int, 1, 2, 3, 4, 5>;
```

接下来写一个 trait `seq_sort_t`，实现编译期排序。类似于：

```cpp
using my_seq = std::integer_sequence<int, 2, 5, 3, 1, 4>;
using sorted_my_seq = seq_sort_t<my_seq>; // std::integer_sequence<int, 1, 2, 3, 4, 5>
```

首先做一些准备工作。

## 运行时输出

通过**类型萃取**，将 `std::integer_sequence` 转换为运行时的 `std::initializer_list`。方便输出结果。

```cpp
template <typename S> struct seq_to_init_list;

template <typename T, T... Is>
struct seq_to_init_list<std::integer_sequence<T, Is...>> {
    static constexpr std::initializer_list<T> value = {Is...};
};

template <typename S>
constexpr auto seq_to_init_list_v = seq_to_init_list<S>::value;
```

用法如下：

```cpp
int main() {
    using seq = std::integer_sequence<int, 1, 2, 3, 4, 5>;
    for (auto i : seq_to_init_list_v<seq>) {
        // Do something...
    }
}
```

> **类型萃取**：
>
> C++ 中可以定义模板
> ```cpp
> template <typename T>
> class Foo {};
> ```
> 
> 如果对于特定的类型，比如说 `T = bool`，有特殊的实现，则可以使用**模板偏特化**：
> ```cpp
> template <>
> class Foo<bool> {};
> ```
> 
> 偏特化时也可以使用其他模板类型。此处的类型 `U` 源自于传入的 `std::vector<U>` 类型。我们从 `std::vector<U>` 中萃取出了 `U`。
> 
> ```cpp
> template <typename U>
> class Foo<std::vector<U>> {};
> ```
>
> 使用 `typedef`、`using` 可以定义类型别名。例如
> ```cpp
> template <typename U>
> class value_type<std::vector<U>> {
>     using type = U;
> };
> ```
> 类似 `value_type` 的结构体称为 **trait**。
> > 和 Rust 没什么关系。

## 序列拼接

之后还需要实现两个 `std::integer_sequence` 的拼接 `seq_concat_t`。为了方便使用，这个 trait 要支持多个序列的拼接。

```cpp
template <typename S1, typename... Ss> struct seq_concat;
```

首先是最基本的情况，实现两个序列的拼接

```cpp
template <typename T, T... Ns1, T... Ns2>
struct seq_concat<std::integer_sequence<T, Ns1...>,
                  std::integer_sequence<T, Ns2...>> {
    using type = std::integer_sequence<T, Ns1..., Ns2...>;
};
```

之后是通用的多个序列拼接。这时需要将问题归约到拼接两个序列的情况。

```cpp
template <typename S1, typename S2, typename... Ss>
struct seq_concat<S1, S2, Ss...> {
    using type =
        typename seq_concat<S1, typename seq_concat<S2, Ss...>::type>::type;
};
```

最后做一下包装。

```cpp
template <typename S1, typename... Ss>
using seq_concat_t = typename seq_concat<S1, Ss...>::type;
```

用法如下：

```cpp
using seq1 = std::integer_sequence<int, 1, 2, 3>;
using seq2 = std::integer_sequence<int, 4>;
using seq3 = std::integer_sequence<int, 5, 6>;
using result = seq_concat_t<seq1, seq2, seq3>; // std::integer_sequence<int, 1, 2, 3, 4, 5, 6>
```

> **变长模板参数**：
> 
> `typename... Ts` 是变长模板参数，表示数量大于等于 0 的一组类型（或整数）。
> 
> 如果需要接受大于等于 1 个类型，则可以这样表示
> 
> ```cpp
> template <typename T, typename... Ts>
> class Foo {};
> ```
> 
> 可以对变长模板参数采取运算，类似于对其中每个元素进行**映射**，例如 `(Is + 1)...`。
> 
> ```cpp
> template <int... Is>
> struct Foo {
>     static constexpr std::initializer_list<T> value = {(Is + 1)...};
> };
> ```
> 
> 另外可以通过 `sizeof...()` 运算符获取变长模板参数的长度。以及可以通过**折叠表达式**对所有元素进行**归约**。

## 快速排序：分类函数

快排的关键是根据基准元素将其他元素分为两部分。因此首先定义 `seq_partition` trait。

`seq_partition` 的原理是，对于给定的取值 `V` 和序列 `S`，我们将序列中的元素根据其与 `V` 的大小关系分为两部分 `left` 和 `right`。我们首先声明 `seq_partition`。

```cpp
template <typename T, template <T, T> class Comparator, T V, typename S>
struct seq_partition;
```

这里我们希望定义排序结果是升序还是降序，所以传入了一个**模板模板参数** `Comparator`。我们规定该参数的用法为 `Comparator<1, 2>::value`。

`seq_partition` 是递归的。我们首先考虑一般情况。此时序列中有一个或多个元素。

```cpp
template <typename T, template <T, T> class Comparator, T V, T Head, T... Tail>
struct seq_partition<T, Comparator, V,
                     std::integer_sequence<T, Head, Tail...>> {
    using next_ =
        seq_partition<T, Comparator, V, std::integer_sequence<T, Tail...>>;
    using left = std::conditional_t<
        Comparator<Head, V>::value,
        seq_concat_t<std::integer_sequence<T, Head>, typename next_::left>,
        typename next_::left>;
    using right = std::conditional_t<
        Comparator<Head, V>::value, typename next_::right,
        seq_concat_t<std::integer_sequence<T, Head>, typename next_::right>>;
};
```

> **偏特化实现条件选择**：
> 
> `std::conditional_t` 是偏特化的一个例子。我们可以实现自己的 `my_conditional_t`
> 
> ```cpp
> template<bool cond, typename IfTrue, typename IfFalse>
> struct my_conditional;
> 
> template<typename IfTrue, typename IfFalse>
> struct my_conditional<true, IfTrue, IfFalse> {
>     using type = IfTrue;
> };
> 
> template<typename IfTrue, typename IfFalse>
> struct my_conditional<false, IfTrue, IfFalse> {
>     using type = IfFalse;
> };
> 
> template<bool cond, typename IfTrue, typename IfFalse>
> using my_conditional_t = typename my_conditional<cond, IfTrue, IfFalse>::type;
> ```


接着定义终止条件。此时序列中没有任何元素

```cpp
template <typename T, template <T, T> class Comparator, T V>
struct seq_partition<T, Comparator, V, std::integer_sequence<T>> {
    using left = std::integer_sequence<T>;
    using right = std::integer_sequence<T>;
};
```

之后我们定义比较 trait。我们要考虑不同的整数类型 `T`。因此定义的 trait 应当能接受类型 `T`，并产生该类型对应的比较 trait。如下所示：

```cpp
template <typename T> struct less_than {
    template <T V1, T V2>
    using type = std::integral_constant<bool, (V1 < V2)>;
};

template <typename T> struct greater_than {
    template <T V1, T V2>
    using type = std::integral_constant<bool, (V1 > V2)>;
};
```

结合 `seq_partition` 和比较 trait，给出一个例子：

```cpp
using seq = std::integer_sequence<int, 1, 2, 3, 4, 5, 6>;
using result = seq_partition<int, less_than<int>, 3, seq>;
using left = typename result::left; // std::integer_sequence<int, 1, 2>
using right = typename result::right; // std::integer_sequence<int, 3, 4, 5, 6>
```



## 快速排序：主体

快速排序的主体部分就比较简单了。`seq_sort` 传入原始序列和比较 trait，返回排序好的序列

```cpp
template <typename T, template <T, T> class Comparator, typename S>
struct seq_sort;
```

排序主体部分，通过 `partition` 将序列分为两部分，分别排序，最后拼接在一起。

```cpp
template <typename T, template <T, T> class Comparator, T Head, T... Tail>
struct seq_sort<T, Comparator, std::integer_sequence<T, Head, Tail...>> {
    using partition_ =
        seq_partition<T, Comparator, Head, std::integer_sequence<T, Tail...>>;
    using left_ = typename partition_::left;
    using right_ = typename partition_::right;
    using type = seq_concat_t<typename seq_sort<T, Comparator, left_>::type,
                              std::integer_sequence<T, Head>,
                              typename seq_sort<T, Comparator, right_>::type>;
};
```

递归终止条件是序列为空。此时也返回空序列

```cpp
template <typename T, template <T, T> class Comparator>
struct seq_sort<T, Comparator, std::integer_sequence<T>> {
    using type = std::integer_sequence<T>;
};
```

最后包装一下

```cpp
template <typename T, template <T, T> class Comparator, typename S>
using seq_sort_t = typename seq_sort<T, Comparator, S>::type;
```

大功告成：

```cpp
int main() {
    using seq = std::integer_sequence<int, 3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5>;
    using sorted1 = seq_sort_t<int, greater_than<int>::type, seq>;
    using sorted2 = seq_sort_t<int, less_than<int>::type, seq>;
    for (auto i : to_initializer_list_v<sorted1>) {
        std::cout << i << ' ';
    }
    std::cout << std::endl;
    for (auto i : to_initializer_list_v<sorted2>) {
        std::cout << i << ' ';
    }
    std::cout << std::endl;
    return 0;
}
```

> 试试结果是什么 :)

## 整体代码

```cpp
#include <initializer_list>
#include <iostream>
#include <type_traits>
#include <utility>

template <typename S> struct to_initializer_list;

template <typename T, T... Ns>
struct to_initializer_list<std::integer_sequence<T, Ns...>> {
    static constexpr std::initializer_list<T> value = {Ns...};
};

template <typename S>
constexpr std::initializer_list<typename S::value_type> to_initializer_list_v =
    to_initializer_list<S>::value;

template <typename S1, typename... Ss> struct seq_concat;

template <typename T, T... Ns1, T... Ns2>
struct seq_concat<std::integer_sequence<T, Ns1...>,
                  std::integer_sequence<T, Ns2...>> {
    using type = std::integer_sequence<T, Ns1..., Ns2...>;
};

template <typename S1, typename S2, typename... Ss>
struct seq_concat<S1, S2, Ss...> {
    using type =
        typename seq_concat<S1, typename seq_concat<S2, Ss...>::type>::type;
};

template <typename S1, typename... Ss>
using seq_concat_t = typename seq_concat<S1, Ss...>::type;

template <typename T, template <T, T> class Comparator, T V, typename S>
struct seq_partition;

template <typename T, template <T, T> class Comparator, T V, T Head, T... Tail>
struct seq_partition<T, Comparator, V,
                     std::integer_sequence<T, Head, Tail...>> {
    using next_ =
        seq_partition<T, Comparator, V, std::integer_sequence<T, Tail...>>;
    using left = std::conditional_t<
        Comparator<Head, V>::value,
        seq_concat_t<std::integer_sequence<T, Head>, typename next_::left>,
        typename next_::left>;
    using right = std::conditional_t<
        Comparator<Head, V>::value, typename next_::right,
        seq_concat_t<std::integer_sequence<T, Head>, typename next_::right>>;
};

template <typename T, template <T, T> class Comparator, T V>
struct seq_partition<T, Comparator, V, std::integer_sequence<T>> {
    using left = std::integer_sequence<T>;
    using right = std::integer_sequence<T>;
};

template <typename T> struct less_than {
    template <T V1, T V2> using type = std::integral_constant<bool, (V1 < V2)>;
};

template <typename T> struct greater_than {
    template <T V1, T V2> using type = std::integral_constant<bool, (V1 > V2)>;
};

template <typename T, template <T, T> class Comparator, typename S>
struct seq_sort;

template <typename T, template <T, T> class Comparator, T Head, T... Tail>
struct seq_sort<T, Comparator, std::integer_sequence<T, Head, Tail...>> {
    using partition_ =
        seq_partition<T, Comparator, Head, std::integer_sequence<T, Tail...>>;
    using left_ = typename partition_::left;
    using right_ = typename partition_::right;
    using type = seq_concat_t<typename seq_sort<T, Comparator, left_>::type,
                              std::integer_sequence<T, Head>,
                              typename seq_sort<T, Comparator, right_>::type>;
};

template <typename T, template <T, T> class Comparator>
struct seq_sort<T, Comparator, std::integer_sequence<T>> {
    using type = std::integer_sequence<T>;
};

template <typename T, template <T, T> class Comparator, typename S>
using seq_sort_t = typename seq_sort<T, Comparator, S>::type;

int main() {
    using seq = std::integer_sequence<int, 3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5>;
    using sorted1 = seq_sort_t<int, greater_than<int>::type, seq>;
    using sorted2 = seq_sort_t<int, less_than<int>::type, seq>;
    for (auto i : to_initializer_list_v<sorted1>) {
        std::cout << i << ' ';
    }
    std::cout << std::endl;
    for (auto i : to_initializer_list_v<sorted2>) {
        std::cout << i << ' ';
    }
    std::cout << std::endl;
    return 0;
}
```