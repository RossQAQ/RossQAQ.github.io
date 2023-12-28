---
title: C++20 Ranges overview
description: 初学 C++20 Ranges
slug: Ranges overview
date: 2023-12-27 00:00:00+0000
image: ranges-cover.png
categories:
    - cppnotes
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文翻译自：[A beginner's guide to C++ Ranges and Views.  —— Hannes Hauswedell](https://hannes.hauswedell.net/post/2019/11/30/range_intro/#lazy-evaluation)

C++ Ranges 是 C++20 的新特性之一，"views" 是 ranges 的大头。这篇文章是给刚接触 C++ ranges 的程序员的一篇介绍。

## 前言

你不需要拥有任何关于 C++ ranges 的前置知识，但你**需要理解 C++ 迭代器，以及了解一些 C++ Concept 特性**，这里有[老爷子的文章](https://www.stroustrup.com/good_concepts.pdf)。

这篇文章基于写给 [SeqAn3 library](https://github.com/seqan/seqan3/) 的文档，原文[在这](https://docs.seqan.de/seqan3/3-master-user/tutorial_ranges.html)，也有关于 [Concept 的文章](http://docs.seqan.de/seqan/3-master-user/tutorial_concepts.html)

> 之后都会尝试搬运翻译。

现在的标准库也都是 ranges v3 了，[原始链接](https://github.com/ericniebler/range-v3/)。

## 动机

传统的 STL 泛型算法，例如 `std::sort` ，接受一对迭代器（如 `begin()` 的返回值），你需要调用

```cpp
std::sort(vec.begin(), vec.end());
// 而不是
std::sort(vec);
```

为什么这么设计？因为更加灵活，举个例子：

- 只排序第五个元素之后的元素

```cpp
std::sort(vec.begin() + 5, vec.end());
```

- 使用反向迭代器排序

```cpp
std::sort(vec.rbegin(), vec.rend());
```

- 结合二者（排序，除最后5个元素）

```cpp
std::sort(vec.rbegin() + 5, vec.rend());
```

但这样的接口相比直接整体调用 `std::sort` 来排序你希望的部分更不直观，它也更可能出错，例如，混淆了两种不兼容的迭代器。

C++20 引入了 *ranges* 并在 `std::ranges::` 下重写了所有算法，例如 `std::ranges::sort(vec)`。现在，假如 vec 是 range，算法就可以正常运行。而 vector 就是 ranges！

那么使用 ranges 版本算法的优点在哪呢？在 C++20，你可以：

- 只排序第五个元素之后的元素

```cpp
std::ranges::sort(std::views::drop(vec, 5));
```

- 反向排序

```cpp
std::ranges::sort(std::views::reverse(vec));
```

- 结合

```cpp
std::ranges::sort(std::views::drop(std::views::reverse(vec), 5));
```

稍后我们讨论 `std::views::reverse(vec)` 的行为。现在只需要记住**它返回某些 `std::ranges::sort` 可以排序的容器**。之后你就会看到这种方式相较于传统迭代器更加灵活的地方。

## 范围

范围 (*Ranges*) 是 “一些元素” 或者 “可迭代对象” 的抽象。最最基础的 range 定义**只要求提供 `begin()` 和 `end()`**。

## 范围概念

有很多种方法解释 ranges，最重要的一种是根据其迭代器的能力。

ranges 通常可以是输入范围（可以读取）、输出范围（可以写入）二者之一，也可以都是。

例如，`std::vector<int>` 就既是输入范围又是输出范围，`std::vector<int> const` 只能是输入范围（只读）。

输入迭代器有不同的 *强度*，其通过不同的更精细的概念实现(即，满足更强概念的类型总是满足更弱的概念):

| Concept                          | 描述                                   |
| -------------------------------- | -------------------------------------- |
| std::ranges::input_range         | 可以**至少一次**从头迭代至尾           |
| std::ranges::forward_range       | 可以**多次**从头迭代至尾               |
| std::ranges::bidirectional_range | 迭代器可以通过 `--` 反向移动           |
| std::ranges::random_access_range | 可以通过 `[]` **在常量时间内**访问元素 |
| std::ranges::contiguous_range    | 元素在内存中总是连续存储               |

这些概念直接从迭代器各自的概念继承而来。例如，如果一个 ranges 模型的迭代器满足 `std::forward_iterator` ，那么他就是一个 `std::ranges::forward_range`。

对于常见的标准库容器，这里展示了他们满足的 ranges 模型：

|                                  | std::forward_list | std::list | std::deque | std::array | std::vector |
| -------------------------------- | ----------------- | --------- | ---------- | ---------- | ----------- |
| std::ranges::input_range         | √                 | √         | √          | √          | √           |
| std::ranges::forward_range       | √                 | √         | √          | √          | √           |
| std::ranges::bidirectional_range |                   | √         | √          | √          | √           |
| std::ranges::random_access_range |                   |           | √          | √          | √           |
| std::ranges::contiguous_range    |                   |           |            | √          | √           |

也有一些概念独立于以上的输入输出要求。例如，`std::ranges::sized_range` 要求常量时间内通过 `std::ranges::size()` 获取 range 的容量。

## 存储行为

**容器**是 ranges 最熟悉的，他们拥有自己的所有元素，STL 已经提供了许多容器，参上。

**视图** 指经常通过另一个 range 以及其底层 range 执行算法、操作等变换得来的 ranges。**视图不拥有除其算法外的任何数据**，且其构造、析构、拷贝都不应依赖于其可以表示的元素的数量。算法要求**惰性求值**，所以结合多个 views 是可行的。

存储行为与通过迭代器定义的范围概念是正交的。**例如，你可以拥有一个容器，满足 `std::ranges::random_access_range` ，但其视图可以满足也可以不满足。**

## 视图

### 惰性求值

视图一个关键的特性就是，**无论对其应用什么变换，他们总是在你需要一个元素时才执行，而并不是在视图创建时执行**。

```cpp
std::vector vec{1, 2, 3, 4, 5};
auto v = std::views::reverse(vec);
```

这段代码中，v 是一个视图；创建他并不会改变 `vec` 的值，且 `v` 也不存储任何元素，构造 `v` 以及其在内存中的大小都和 `vec` 的大小无关。

```cpp
std::vector vec{1, 2, 3, 4, 5, 6};
auto v = std::views::reverse(vec);
std::cout << *v.begin() << '\n';
```

会打印 "6"，但重要的是**将 vec 的最后一个元素解析为 v 的第一个元素是按需发生的**。这保证视图可以作为更灵活的迭代器来使用，但他意味着 view 有一些额外开销，如果一个相同元素需要使用多次，那么就得重复计算。

## 组合能力

你可能想知道为什么文章中写

```cpp
auto v = std::views::reverse(vec);
```

而不是

```cpp
std::views::reverse v{vec};
```

这是因为 `std::views::reverse` 并不是视图本身，他是一个底层视图（我们的例子中，是 vector）的 ***适配器 (adaptor)***，并且返回一个基于 vector 的视图对象。视图的具体类型隐藏在 auto 身后。这有一些好处，我们不需要关心视图类型的模板参数，但是更重要的是适配器有一个额外的特性：**适配器可以和其他适配器链起来**

```cpp
std::vector vec{1, 2, 3, 4, 5, 6};
auto v = vec | std::views::reverse | std::views::drop(2);

std::cout << *v.begin() << '\n';
```

会打印什么呢？

会打印 4，因为 4 是反向后去掉 2个元素后的第一个元素。

在上面的例子中，vector 是 piped（类似于 unix/linux 命令行）传入 reverse 适配器，之后传入 drop 适配器，最后组合出的视图被返回。管道是一种提升可读性的形式。例如 

`vec | foo | bar(3) | baz(7)` 等同于 `baz(bar(foo(vec), 3), 7)`

注意，访问视图的 0th 元素仍然是惰性的，这也决定了访问时其映射的元素会发生什么。

----

### 练习

基于 `std::vector vec{1, 2, 3, 4, 5, 6}` 创建一个视图，过滤掉所有的奇数，平方剩下的值。

```cpp
std::vector vec{1, 2, 3, 4, 5, 6};
auto v = vec | // ...?

std::cout << *v.begin() << '\n'; // 应该打印 4
```

完成这个练习，你可以使用 `std::views::transform` 以及 `std::views::filter` 。他们都接受一个可调用对象。`std::views::transform` 会在底层 range 的元素上应用可调用对象，`std::views::filter` 会移除不满足可调用对象的元素。

答案（我写的）：

```cpp
std::vector vec{ 1, 2, 3, 4, 5, 6 };
auto square = [](const int value) {return value * value; };
auto odd = [](const int value) {return value % 2 == 0; };

auto v = vec | std::views::filter(odd) | std::views::transform(square);

std::cout << *v.begin();
```

## 视图概念

视图是特殊的 range，其实例化于 `std::ranges::view` 概念，每个通过视图适配器返回的视图都是符合该概念的模型，还有哪些范围概念是由视图建模的？

这取决于底层 range 以及视图本身。很容易想到，视图并不会建模比其底层 range 更强的 range 概念（除了他们总是 `std::ranges::view` ），他会尽可能多地保留  range 的概念。

**例如，`std::views::reverse` 返回的视图满足 `std::ranges::random_access_range`（以及更弱的模型），前提是底层视图满足该概念，但他永远不会满足 `std::ranges::contiguous_range`，因为视图中的第三个元素在内存中并不位于第二个元素之后，而是其之前。**

这可能会震惊一些人，许多视图满足 `std::ranges::output_range`，如果其底层 range 满足的话。例如：**非只读视图：**

```cpp
std::vector vec{1, 2, 3, 4, 5, 6};
auto v = vec | std::views::reverse | std::views::drop(2);

*v.begin() = 42; // now vec == {1, 2, 3, 42, 5, 6 } !!
```

----

### 练习

考虑上一道习题（filter + transform），v 可能满足哪些概念？

其满足：

| Concept                          | 满足与否 |
| -------------------------------- | -------- |
| std::ranges::input_range         | √        |
| std::ranges::forward_range       | √        |
| std::ranges::bidirectional_range | √        |
| std::ranges::random_access_range |          |
| std::ranges::contiguous_range    |          |
| std::ranges::view                | √        |
| std::ranges::sized_range         |          |
| std::ranges::output_range        |          |

filter 无法保证元素随机访问，因此并不连续。因为无法在常数时间内知道底层 range 的元素究竟是第几个元素。它不能“跳”过去，他需要遍历底层 range，这当然也代表我们无法在常数时间内知道大小。

transform 视图可以 jump，因为它总是对每个独立元素执行相同操作；它也可以保证 sized-ness 因为留下的 size 相同。在这些情况下，filter 都会丢弃这两只属性。换句话说，transform视图 在每次访问时产生新的元素，因此 `v` 不是输出范围，你不能向他的元素赋值。这也阻止了 contiguous-range 的建立（如果没有被 filter 处理掉的话，因为按需创建的值根本不在内存中存储）

理解这些需要更多练习，在 SeqAn3 中提供了[详细的文档](https://docs.seqan.de/seqan/3-master-user/group__views.html)。
