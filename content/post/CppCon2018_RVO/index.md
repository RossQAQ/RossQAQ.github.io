---
title: 返回值优化
description: 对 CppCon 2018, Arthur O'Dwyer 演讲的翻译与总结。
slug: CppCon 2018 RVO
date: 2024-02-16 00:00:00+0000
image: cppcon2018-cover.png
categories:
    - cppcon
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Lecture: Return Value Optimization: Harder Than It Looks - Arthur O'Dwyer - CppCon 2018](https://www.youtube.com/watch?v=hA1WNtNyNbo)

[Slides PDF](https://github.com/CppCon/CppCon2018/blob/master/Presentations/return_value_optimization_harder_than_it_looks/return_value_optimization_harder_than_it_looks__arthur_odwyer__cppcon_2018.pdf)

C++17 开始标准强制要求复制消除，这也是 C++17 最重要的特性之一。

来看看 **返回值优化**，即 *RVO* 吧。

主讲依然是 Arthur O'Dwyer。

## return slot

对于 x86，单纯返回个 int 会存放在 %eax 中。

那么如果我们返回一个比较大的对象呢？例如一个结构体，总不能把结构体塞到 %eax

这时 caller 会提前分配好 stack 空间供其调用的函数的返回值填充，这也就是所谓的 "return slot"

可能遇到的情况是，你需要将返回值 move 到 return slot

例如下面的情况，在你调用函数之前并不知道该返回哪个，只能在返回的时候进行判断再“移动”（在返回时将其*复制* 到 return slot）

```cpp
struct Fruit {
    int data[5];
    Fruit(Fruit&&)
};

Fruit apples_and_oranges(bool condition) {
    Fruit x = ...;
    Fruit y = ...;
    return std::move(condition ? x : y);
}
```

但如果他是一个只返回 x 的 `nothing_but_apples()` ：

```cpp
struct Fruit {
    int data[5];
};

Fruit nothing_but_apples() {
    Fruit x = ...;
    return x;
}
```

那么标准允许**将 x 与 return slot 整合**，现在我们就完全不需要拷贝了。

## 复制消除

在 C++17 之后，复制消除在一些场合下是*强制的*。以前的标准并不强制。

当然，也有复制消除不能应用的场景，例如之前的 `apples_and_oranges()`，我们并不知道要返回哪个对象。

自然，也有一些其他的情景，比如以下的例子

### 返回值是参数的情况

```cpp
Fruit apples_to_apples(int i, Fruit x, int j) {
    return x;
}
```

这种情况，x 自然被分配在一个位置，而 caller 自然也给 return slot 分配了另外一个位置，这种情况下自然不能消除复制，因为我们没有实际 x 的位置。

### 返回值是全局变量

```cpp
static Fruit x;
Fruit apples_to_apples() {
    return x;
}
```

现在 x 是全局变量，显然更不能复制消除了。

### 切片为基类

举个例子，榴莲也是水果，所以是 Is-A 的关系

```cpp
struct Durian : Fruit {
    double smell;
}

Fruit slapchop() {
    Durian x = ...;
    return x;
}
```

上面的情形中，我们创建了一个 Durian，但返回的是 Fruit，二者占用的内存大小并不相同。

如果把榴莲返回的话，就会被 slice。

这种情况下自然不能复制消除，我们不能在先分配的 return slot 中直接分配 x，因为 x 比 return slot **大**（return slot 是按照 Fruit 的大小分配的）。

## Rules of thumb for RVO

以下是两条常见的 RVO 规则：

- Unnamed RVO (URVO) : 返回 xvalue/prvalue 会触发复制消除；（常见的例如函数调用是纯右值，临时对象是亡值）
- Named RVO (NRVO) : 除了我们上面举的例子，假设 x 是 local 具名变量，那么返回时也会触发复制消除。
- 如果*没有发生* 复制消除的话，那么编译器会 *隐式* 选择移动

> 实际上似乎标准中提到的 RVO 指的是 URVO，保证的也是 URVO。可以自行参考 [Copy elision](https://en.cppreference.com/w/cpp/language/copy_elision)
>
> 此外，类成员变量 *不是* 隐式可移动对象，想想也很正常，不然调用个成员函数直接就把数据清空了。

关于 隐式移动：

- 返回一个不会触发 RVO 的具名 local 变量，编译器重载决议会自动将 x 当成 *亡值* 处理

  ```cpp
  std::string identity(std::string x) {
      return x;	// 自动触发移动，不会被复制
  }
  ```

- 因为 C++11 的隐式移动，你写 `return std::move(x)` 是纯负优化，因为会强制编译器移动它，编译器就没法触发 NRVO 了。

## 其他

> 后面还有一堆看起来非常复杂的内容和规则，因为时间不够还跳过了一部分。

总结起来就是，为了避免切片，以及可以重载决议到 move

```cpp
std::unique_ptr<ConfigManager> create() {
    auto p = std::make_unique<ConfigManagerImpl>();
    return p;
}
```

对于这种类型，一定要多实现 `explicit ctor(typename&&)` 的版本

标准库的这些组件都是这么做的。

还有一些额外内容，建议有能力的看原视频。
