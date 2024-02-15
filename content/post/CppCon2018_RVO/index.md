---
title: 返回值优化
description: 对 CppCon 2018, Arthur O'Dwyer 演讲的翻译与总结。
slug: CppCon 2018 RVO
date: 2024-02-15 09:00:00+0000
image: cppcon2018-cover.png
categories:
    - cppcon
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Lecture: Return Value Optimization: Harder Than It Looks - Arthur O'Dwyer - CppCon 2018](https://www.youtube.com/watch?v=hA1WNtNyNbo)

[Slides PDF](https://github.com/CppCon/CppCon2018/blob/master/Presentations/return_value_optimization_harder_than_it_looks/return_value_optimization_harder_than_it_looks__arthur_odwyer__cppcon_2018.pdf)

C++17 开始标准强制要求复制消除，这也是 C++17 最重要的特性之一。

来看看 **返回值优化**，即 *RVO* 吧。

主讲依然是 Arthur O'Dwyer。

