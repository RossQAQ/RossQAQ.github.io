---
title: C++ 类型擦除的设计思想
description: 对 CppCon 2021, Klaus Iglberger 演讲的翻译与总结。
slug: CppCon 2021 type-erasure
date: 2024-02-23 20:00:00+0000
image: cppcon2021-cover.png
categories:
    - cppcon
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Breaking Dependencies: Type Erasure - A Design Analysis - Klaus Iglberger - CppCon 2021](https://www.youtube.com/watch?v=4eeESJQk-mw)

讲解类型擦除的，更重要的是考虑其背后的设计思想。

也是熟人，之前讲解 class design 的，而这次的类型擦除也跟那期有关。

