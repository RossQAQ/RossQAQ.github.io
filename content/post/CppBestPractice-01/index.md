---
title: C++ Best Practice - 前言
description: C++ Best Practice Intro
slug: C++ Best Prac Intro
date: 2024-01-23 00:00:00+0000
image: 
categories:
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

**声明：本博客是对 C++ Best Practice (By Jason Turner) 的翻译与注释，仅供个人学习用途。**

**如有问题，请联系 rossqaq@outlook.com 修改或删除。**

**有能力请购买电子版正版**：[C++ Best Practices](http://leanpub.com/cppbestpractices)

> 支持正版，从你我做起。

以下是原书 Intro 部分译文。

## 前言

> 这本书为 CppCon 2021 而作，包含电子版上一版以及一些新的章节。

我（Jason Turner）作为老师希望每个人：

1. 学习如何自己动手实验
2. 不要只相信我，而要多测试
3. 学习语言是怎么工作的
4. 不要再犯上一代的错误

I’m thinking about changing my title from “C++ Trainer” to “C++ Guide.” I always adapt my courses and material to the class I currently have. We might agree on a class about X, but I change it to Y halfway through the first day to meet the organization’s needs. 

Along the way, we experiment and learn as a group. I often also learn while teaching. Every group is unique; every class has new questions. 

Many of the questions I get in classes are the same ones repeatedly to the point where I get to look like a mind reader as I anticipate the next question that will be asked. 

Hence, this book, and the Twitter thread that it came from, to help spread the word on the long-standing best practices.

I wrote the book I wanted to read. It is intentionally straightforward, short, to the point, and has specific action items.

## 关于 Best Practice

Best Practice，其目的是：

1. 减少普遍错误。
2. 快速找出错误。
3. 不要牺牲（通常反而会提升）性能。

### Why Best Practice

首先，我们需要弄清楚：

#### 你的项目并不特殊

如果你使用 C++，代表你，或者你公司的某些人非常在意性能表现。否则，大概率会用一些其他语言。我去过很多公司， 他们都认为自己是特殊的，需要更快的性能。

注意：他们都因为共同的原因做出了相同的决定。

There are very few exceptions. The outliers who make different decisions: they are the organizations that are already following the advice in this book.

#### 可能预见的坏事

如果你的项目有 critical 缺陷，那么最坏的情况会是？

- 游戏

  严重的缺陷会导致远程攻击 or attack vector

- 金融

  严重的缺陷会导致经济损失，加速交易，市场崩溃

- 航空航天

  严重的缺陷导致失去飞船甚至是人的生命

- 工业

  经济损失？失业？hacks？更糟糕？

### 例子

本书的例子全部使用 struct 代替 class，二者的唯一区别是可见性，struct 内的成员默认可见性是 public。使用 struct 让例子更加简洁。

### 练习

每一节会有一或多个练习，大多数都没有完全正确或完全错误的答案。

### 练习：寻找练习们

看看下面的章节，你会找到类似于这里的练习。

练习是为了：

- 确保你通过自己的努力加深了对语言的理解

### 链接和引用

会放出参考的文章。
