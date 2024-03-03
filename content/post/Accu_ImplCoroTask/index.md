---
title: C++ 协程手把手实现 Task
description: 对 Dietmar Kühl ACCU 2023 的演讲的翻译与总结。
slug: coro-tasks
date: 2024-02-25 00:00:00+0000
image: 
categories:
    - coroutine
    - techs
    - cppnotes
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Implementing a C++ Coroutine Task from Scratch - Dietmar Kühl - ACCU 2023](https://www.youtube.com/watch?v=Npiw4cYElng)

众所周知，直到 C++23 标准库都没有什么可用的协程组件，所以我们来自己实现一个 Task 封装。

协程的基础知识不会再介绍，可以参考之前的博文。