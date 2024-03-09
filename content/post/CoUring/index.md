---
title: io_uring + coroutine
description: 一起阅读文章，学习 io_uring 以及 协程 以及 多线程如何组合成为强力的武器。
slug: io_uring-coro
date: 2024-03-09 00:00:00+0000
image: 
categories: 
    - io-uring
    - coroutine
    - techs
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

终于让我发现了 io_uring 与协程结合的教程，后面还包括与多线程结合。让我来验证一下我的猜想究竟对不对…

本文是系列文章：

[C++20 Coroutines and io_uring - Part 1/3](https://pabloariasal.github.io/2022/11/12/couring-1/)

[C++20 Coroutines and io_uring - Part 2/3](https://pabloariasal.github.io/2022/11/13/couring-2/)

[C++20 Coroutines and io_uring - Part 3/3](https://pabloariasal.github.io/2022/11/13/couring-3/)

的**翻译与总结**。文章间会使用分割线分开。

关于作者

I’m Pablo, a software engineer living in Munich. Please write me an email if you have feedback or have discovered an error in a post.

-------

# Part 1/3

## 前言

在这个系列中，我们会写使用 `io_uring` 和 C++20 协程的读取很多硬盘上的文件的程序。主要的目的是因为，我发现有很多关于 `io_uring` 和 C++20 协程的单独资料，也有讲的比较深入的，但是基本没有展示如何结合二者的。我们会揭开异步 IO 和 协程结合的谜底，就像黄油和面包一样。

这个系列会分成三部分，P1 我们首先仅用 `io_uring` 来解决问题。P2 我们会重构实现，并且加入协程。P3 我们会使用多线程优化，这才能展现出协程的真正力量。

## Async I/O ❤ Coroutines

异步 IO，跟同步 IO 相反，描述一种**不阻塞当前 calling 线程**的 IO 操作。跟等待操作完成不同，当前线程会立刻 release，然后执行其他操作，而此时后台正在执行你请求的 IO 操作。待一段时间后，calling 线程（或者其他的线程）可以回来并且收集请求操作的结果。这就好像你对一个 pizza guy 说：“将 margherita 放到烤箱，我去药房取药，一会回来。”

异步 IO 可以比同步 IO 更有效率，线程不需要等待资源可用，但同时程序也会变得更加复杂。程序需要异步工作，必须记得回来取走他们点的 margherita。

**协程允许我们以同步的形式编写异步代码**，如果你使用协程来从硬盘读取一个文件，它可以挂起自己然后将控制流返回给 caller，此时文件正在后台读取，等数据读取完后恢复协程，然后获取数据。所有的代码写起来就和 good-old 同步调用类似。

结合异步 IO 和协程，允许我们编写异步程序，但并不用写繁琐的异步代码。取了二者的长处。

## 目标

这个系列中，我们会写一个程序，从硬盘中读取并且 parse 几百个 [wavefront OBJ](https://en.wikipedia.org/wiki/Wavefront_.obj_file) 文件。

了解读取和 parsing 的区别很重要。读取代表**把文件从内存中装载到内存中**。parsing 代表**从内存中取数据，然后把它翻译为应用程序可以理解的结构**。

OBJ 文件使用 ASCII 编码，描述的是 3D 三角形组成的网格图形。该文件对构成三角形的网络、顶点、颜色等信息编码。parsing 一个 OBJ 意味着将 ASCII 表现形式转换为有更方便访问网格属性 API 的  C++ 对象形式。

> 玩过 3dmax 这种建模软件的应该都知道 .obj 吧。不是你程序编译中间生成的目标文件。是记录 3d 图形的。

为了 parsing obj 文件，我们使用一个三方库 [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader)。它接受一个 string，然后把它 parse 成 `ObjReader` 对象。

