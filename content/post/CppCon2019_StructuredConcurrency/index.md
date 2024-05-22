---
title: 结构化并发与协程
description: 对 CppCon 2019, Lewis Baker 演讲的翻译与总结。
slug: CppCon 2019 Strutured Concurrency
date: 2024-05-14 00:00:00+0000
image: cppcon2019-cover.png
categories:
    - cppcon
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Structured Concurrency: Writing Safer Concurrent Code with Coroutines... - Lewis Baker - CppCon 2019](https://www.youtube.com/watch?v=1Wy5sq3s2rg)

Lewis Baker 不用介绍了，以前写过很多关于协程的文章，github还有库 `cppcoro`.

这次介绍协程与结构化并发的关系，专注于协程；而 Roses 之前翻译的文章更多介绍的是结构化并发本身。

## 安全性

- 异常安全
- 生命周期安全（悬垂引用/指针）
- 资源泄露
- 数据竞争/ API 竞争
- forward progress / 死锁

本 lecture 主要介绍前三个。

