---
title: 普通的 std::tuple 技巧
description: 除了我之外人尽皆知的 std::tuple 魔法
slug: tuple-tricks
date: 2024-03-24 00:00:00+0000
image: 
categories:
    - techs
    - cppnotes
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文来自 Raymond Chen 的系列博客

[Mundane std::tuple tricks: Getting started]([Mundane std::tuple tricks: Getting started - The Old New Thing (microsoft.com)](https://devblogs.microsoft.com/oldnewthing/20200622-00/?p=103900))

[Mundane std::tuple tricks: Selecting via an index sequence](https://devblogs.microsoft.com/oldnewthing/20200623-00/?p=103901)

[Mundane std::tuple tricks: Selecting via an index sequence, part 2](https://devblogs.microsoft.com/oldnewthing/20200624-00/?p=103902)

[Mundane std::tuple tricks: Creating interesting index sequences](https://devblogs.microsoft.com/oldnewthing/20200625-00/?p=103903)

[Mundane std::tuple tricks: Creating more interesting index sequences](https://devblogs.microsoft.com/oldnewthing/20200626-00/?p=103904)

[Mundane std::tuple tricks: Finding a type in a tuple](https://devblogs.microsoft.com/oldnewthing/20200629-00/?p=103910)

**未取得授权，私自翻译，仅用于学习目的，如若侵权联系我删除**。

---

## Getting Started

C++ 标准库的 `tuple` 充满魔法，它可以将一堆类型或者值 grab 到一个单个的单位，并且 C++ 标准库也提供了很多帮助函数来辅助操作。

例如，`std::make_tuple` 让你可以从一堆值中构造 tuple，用来解决你捕获参数包之后把它变成可以操作的东西：

```cpp
[](auto... args) {
    auto args_tuple = std::make_tuple(std::move(args)...);
}
```

我们学习过，`std::tuple_element_t` 可以让你从 tuple 中获取单个类型，`std::get` 可以获取单个值。

标准库提供 `std::tuple_cat` 来串联 N 个 tuple 的值，但标准库没有提供串联 N 个 tuple 类型的版本，我们可以自己实现一个：

```cpp
template<typename T1, typename T2> struct tuple_cat_helper;
template<typename... T1, typename... T2> 
struct tuple_cat_helper<std::tuple<T1...>, std::tuple<T2...>> {
    using type = std::tuple<T1..., T2...>;
}

template<typename T1, typename T2>
using tuple_cat_t = typename tuple_cat_helper<T1, T2>::type;

// example is std::tuple<int, char, double>
using example = tuple_cat_t<std::tuple<int>, std::tuple<char, double>>;
```

我们定义了一个特化的模板 `tuple_cat_helper` 来析取所有的 tuple 类型并且生成新的类型，串联两个 tuple 列表。之后定义了 `_t` 的版本。

或者也可以偷懒，让 `std::tuple_cat` 来做：

```cpp
template<typename T1, typename T2>
using tuple_cat_t = decltype(std::tuple_cat(std::declval<T1>(), std::declval<T2>()));
```

然后为了让他支持多个 Tuple：

```cpp
template<typename... Tuples>
using tuple_cat_t = decltype(std::tuple_cat(std::declval<Tuple>()...));
```

写的更少，做的更多。

标准库中有工具把它们组合在一起，但是没有工具把它们分开。

**Bonus chatter**: I wasn’t quite telling the truth when I said that `make_tuple` can capture a template parameter pack. We’ll come back to this issue later.

## 通过 index sequence 进行选择 1

上一次，我们组合了 tuples。组合他们很简单，但想要分开他们有点难。

`std::index_sequence` (C++14) 由**标准库提供的类型**，捕获 0 个或更多的非负整数序列，然后把它们转换为一个类型。它是 `std::integer_sequnence` 的特化。`std::integer_sequence` 会捕获用户提供的类型的整数序列，而 `std::index_sequence` 的用户提供类型是 `std::size_t`。

tuple 的拆分涉及到包含一个 `std::index_sequence` 的参数包展开。（译者：原文用的 fold expression，但这里明显不是 C++17 的折叠表达式，应该是说包展开吧。）

```cpp
// 不要这么做，之后会解释：
template<typename Tuple, std::size_t... Ints>
auto select_tuple(Tuple&& tuple, std::index_sequence<Ints...>) {
    return std::make_tuple( std::get<Ints>(std::forward<Tuple>(tuple))... );
}
```

以上这一段是拆分 tuple 的**核心**，我们来解释一下。

第一个参数是需要操作的 tuple，使用万能引用传入，这样我们就可以对他进行转发了。这会保留右值性（rvalue-ness），这在某些情况比如 tuple 是 move-only 的时候非常好用。（在 both copyable and movable 情况下也有帮助，因为它会选择移动，这样的话开销更小）。

剩下的参数是 `size_t` 数值，代表 index_sequence 的 index。

表达式：

```cpp
(std::get<Ints>(std::forward<Tuple>(tuple))...);
```

是 `std::make_tuple` 的参数列表。表达式会对每个 `Ints` 包中的值调用，结果就是生成一系列的参数然后来提取 tuple 中对应索引的值。

例如：

```cpp
auto res = select_tuple(std::make_tuple('x', 3.14, 'z'), std::index_sequence<2, 1, 1, 2>{});
```

我们提供了一个 3 个元素的 tuple，并且选择 2, 1, 1, 2 对应索引的元素。表达式会展开为：

```cpp
    (std::get<2>(std::forward<Tuple>(tuple)),
     std::get<1>(std::forward<Tuple>(tuple)),
     std::get<1>(std::forward<Tuple>(tuple)),
     std::get<2>(std::forward<Tuple>(tuple)))
```

从 tuple 中提取项 2, 1, 1, 2，然后把它们传递给 `make_tuple` ，重新把它们组成一个 tuple `('z', 3.14, 3.14, 'z')`。注意索引 1 和 2 都被提取了多次，0没有被提取；也要注意结果 tuple 的 size 跟使用的索引匹配，而不是原 tuple 的 size。

注意如果 tuple 里的类型是 movable 类型，那么提取 <2, 1, 1, 2> 会导致它们的项被多次移动。这样的话结果就乱了，所以你通常不应该对一个值提取多次。（虽然并没有阻止这么做）。

不过我们的 `select_tuple` 也有缺陷。

## 通过 index sequence 进行选择 2

上次我们编写了 `select_tuple` 函数接收一个 tuple 和一个 index sequence 为参数，并且产生一个新的基于 index sequence 选择元素的 tuple。我们有：

```cpp
// Don't use this; see discussion.
template<typename Tuple, std::size_t... Ints>
auto select_tuple(Tuple&& tuple, std::index_sequence<Ints...>)
{
 return std::make_tuple(
    std::get<Ints>(std::forward<Tuple>(tuple))...);
}
```

你如果想使用它的话：

```cpp
std::tuple<int, char, float> t{ 1, 'x', 2.0 };
auto t2 = select_tuple(t, std::index_sequence<0, 2>{});
```

`t2` 是 `std::tuple<int, float>{1, 2.0}`

但这个函数有问题。

提问：**什么时候 `std::make_tuple<T>()` 返回的不是 `std::tuple<T>` ？**

| `std::make_tuple<T>`                | Produces `std::tuple<T>` |
| ----------------------------------- | ------------------------ |
| int                                 | int                      |
| const int                           | int                      |
| int&                                | int                      |
| int&&                               | int                      |
| `std::reference_wrapper<int>`       | int&                     |
| `std::reference_wrapper<const int>` | int&                     |
| `std::reference_wrapper<int&>`      | int&                     |
| `std::reference_wrapper<int&&>`     | int&                     |

答案：**当 `T` 是可以退化，或者是 `reference_wrapper` 类型的时候**

退化是一个 C++ 术语，指代的是在**传值给函数时类型发生的变化的行为**：

- 引用会退化为底层数值类型
- cv 会被移除
- 数组退化为指针
- 函数退化为函数指针

但是 `make_tuple` 有额外的规则：**如果退化的类型是一个 `reference_wrapper` ，那么其结果是底层的引用类型**。

我们并不想发生那样的转换。如果你从 tuple 中选择的类型是个引用，那么你想要结果 tuple 也有相同的引用类型。

所以不能使用 `make_tuple`，我们得显式的指明我们要的类型：

```cpp
template<typename Tuple, std::size_t... Ints>
auto select_tuple(Tuple&& tuple, std::index_sequence<Ints...>) {
    return std::tuple<std::tuple_element_t<Ints, Tuple>...>(
        std::get<Ints>(std::forward<Tuple>(tuple))...);
}
```

或者也可以：

```cpp
template<typename Tuple, std::size_t... Ints>
std::tuple<std::tuple_element_t<Ints, Tuple>...>
select_tuple(Tuple&& tuple, std::index_sequence<Ints...>) {
    return { std::get<Ints>(std::forward<Tuple>(tuple))... };
}
```

好了，helper 都有了，我们可以玩更花哨的了。

## 创建有趣的 index sequence

C++ 标准库对于操作 index sequences 只给了一个 helper，`std::make_integer_sequence` 以及它的近亲 `std::make_index_sequence`，也就是 `size_t`。

注意，`std::make_index_sequnece` 的模板参数是结果 index sequence 的大小，并不是最高那个值。

即使只有从 0 开始的 index sequences，我们也能干很多有趣的事情。

```cpp
template<typename Tuple>
auto remove_last(Tuple&& tuple) {
    constexpr auto size = std::tuple_size_v<Tuple>;
    using indices = std::make_index_sequence<size - 1>;
    return select_tuple(std::forward<Tuple>(tuple), indices{});
}
```

`remove_last` 函数移除 tuple 的最后一个元素并且 return 剩下的。我们通过提取源 tuple 的 size 来做到这一点，让他 -1，然后生成一个新的 index sequence（0~size - 2），元素个数就为 size - 1

那么，其他的 index sequence 怎么样呢？我们得自己实现

```cpp
template<std::size_t N, typename Seq> struct offset_sequence;

template<std::size_t N, std::size_t Ints...>
struct offset_sequence<N, std::index_sequence<Ints...>> {
    using type = std::index_sequence<Ints + N...>;
}

template<std::size_t N, typename Seq>
using offset_sequence_t = typename offset_sequence<N, Sqe>::type;

// example = index_sequence<3, 4, 5, 6>
using example = offset_sequence_t<3, std::make_index_sequence<4>>;
```

为了实现 index sequence 的偏移版本，我们生成一个新的 index sequence 它持有的序列是原来的序列 + offset N。魔法发生在参数包展开：

```cpp
 using type = std::index_sequence<Ints + N...>;
```

这会取出原 index sequence 的每个数，然后 +N，之后用来重新生成一个新的序列。

现在我们就可以移除 tuple 的第一个元素了

```cpp
template<typename Tuple>
auto remove_first(Tuple&& tuple) {
    constexpr auto size = std::tuple_size_v<Tuple>;
    using indices = offset_sequence_t<1, std::make_index_sequence<size - 1>>;
    return select_tuple(std::forward<Tuple>(tuple), indices{}); 
}
```

实际上，我们可以移除第 N 个元素

```cpp
template<std::size_t N, typename Tuple> 
auto remove_Nth_element(Tuple&& tuple) {
    constexpr auto size = std::tuple_size_v<Tuple>;
    using first = std::make_index_sequence<N>;
    using rest = offset_sequence<N + 1, std::make_index_sequence<size - N - 1>>;
    return std::tuple_cat(
    	select_tuple(std::forward<Tuple>(tuple), first{});
        select_tuple(std::forward<Tuple>(tuple), rest{});
    );
}
```

我们想要的是提取前 N 个元素，然后跳过第 N 个，之后提取 N + 1 个元素到结尾。

提取前 N 个很简单：直接选择从 0 ~ N - 1的

提取剩余的需要一些思考：我们想要 N + 1 开始，知道 size - 1 结束，长度为 (size - 1) - (N + 1) + 1 = size - N - 1. 好了，现在我们生成了长度为 size - N - 1 的整数序列，起始点就是 N + 1.

我们调用两次 `select_tuple`，一次获取前半部分，一次获取后半部分，之后用 `std::tuple_cat` 组合。

另一种方法是只选择一次，如果这么做的话，我们需要结合两个 index sequences

```cpp
template<typename Sqe1, typename Seq2> struct cat_sequence;

template<std::size_t... Ints1, std::size_t... Ints2>
struct cat_sequence<std::index_sequence<Ints1...>, std::index_sequence<Ints2...>>
{
    using type = std::index_sequence<Ints1..., Ints2...>;
}

template<typename Seq1, typename Seq2>
using cat_sequence_t = typename cat_sequence<Seq1, Seq2>::type;

// example = index_sequence<3, 1, 4, 1, 5, 9>
using example = cat_sequence_t<std::index_sequence<3, 1, 4>,
                               std::index_sequence<1, 5, 9>>;
```

魔法发生在：

```cpp
using type = std::index_sequence<Ints1..., Ints2...>;
```

接收两个 sequence 并且把他们挨着组成一个单个的序列。

我们现在可以这样使用：

```cpp
template<std::size_t N, typename Tuple>
auto remove_Nth_element(Tuple&& tuple)
{
  constexpr auto size = std::tuple_size_v<Tuple>;
  using first = std::make_index_sequence<N>;
  using rest = offset_sequence_t<N+1,
                std::make_index_sequence<size-N-1>>;
  using indices = cat_sequence_t<first, rest>;
  return select_tuple(std::forward<Tuple>(tuple), indices{});
}
```

## 创建更有趣的 index sequence

我们可以泛化之前的版本，

```cpp
template<std::size_t F(std::size_t), typename Seq> struct modify_sequence;

template<std::size_t F(std::size_t), typename std::size_t... Ints>
struct modify_sequence<F, std::index_sequence<Ints...>> {
    using type = std::index_sequence<F(Ints)...>;
};

template<std::size_t F(std::size_t), typename Seq>
using modify_sequence_t = typename modify_sequence<F, Seq>::type;
```

> 未完待续……

