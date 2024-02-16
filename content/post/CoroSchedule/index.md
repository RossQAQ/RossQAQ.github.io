---
title: C++ 协程，以及其调度
description: 对 Dian-Lun Lin CppNow 2023 的演讲的翻译与总结。
slug: coro-schedule
date: 2024-02-16 00:00:00+0000
image: 
categories:
    - techs
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Introduction to C++ Coroutines Through a Thread Scheduling Demonstration](https://www.youtube.com/watch?v=kIPzED3VD3w)

由于并不知道如何写个调度器，所以找到了这个资料来学习。

协程的基础使用方法已经在文章 [Writing custom C++20 coroutine systems](https://rossqaq.github.io/article/coroutine-system/) 中介绍的足够详细，所以视频中关于协程基础的用法我会选择性翻译或者直接略过。

重点摘录后半部分对于调度的讲述内容。