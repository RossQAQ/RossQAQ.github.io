---
title: io_uring P2 - 实现 cp
description: 一起学习使用 liburing 一次提交多个 requests 来实现 cp 命令
slug: io_uring-cp
date: 2024-02-16 00:00:00+0000
image: 
categories:
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文是文章 [io_uring by example: Part 2 – Queuing multiple requests](https://unixism.net/2020/04/io-uring-by-example-part-2-queuing-multiple-requests/) 的翻译与总结。

同样只摘录重要的部分。

这一篇文章使用 liburing 来构造类似于 cp 命令的程序，但区别于 P1，由于 P1 的重点在于理解 io_uring 本身，所以并没有用上很多特性。

## 简介
