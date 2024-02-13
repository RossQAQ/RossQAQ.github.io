---
title: 现代 C++ 模板应该学习什么？
description: 阅读 mq白 现代 C++ 模板教程知识点记录。
slug: Modern-Cxx-Template
date: 2024-02-12 09:00:00+0000
image: 
categories:
    - cppnotes
    - continuous
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

## 函数模板

### 使用函数模板

1. 函数模板使用时才会实例化

### 函数模板参数推导规则

1. 函数模板参数推导规则、无法推导的情况
2. ADL 对于函数模板调用时的影响
3. 万能引用、引用折叠，特殊推导规则

### 函数模板默认实参

1. 可以给模板类型实参默认值，既然是类型，那么默认值也要类型

   （func<>() 代表使用默认值）

2. 推导时，P/A 对无法推导（例如3P 2A对不上的情况）下的处理

   默认实参？部分指明？通过 decltype 三目表达式配合 decay_t 获取 common_type？

   后置返回值类型配合 decltype

   对于 C++20，甚至可以使用简写函数模板。注意 auto 和 decltype 的推导规则

### 非类型模板实参

```cpp
template <std::size_t N>
void func() { std::cout << N << '\n'; }

func<5>();
```

