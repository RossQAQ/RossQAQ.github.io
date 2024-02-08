---
title: C++ Best Practice - Chapter 3
description: "C++ Best Practice Chapter 3 - Use the Tools: Automated Tests"
slug: C++ Best Prac Cp3
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

## 使用：支持自动测试的工具

你需要一键运行所有测试，如果你仍然没有的话，以下有一些推荐

- [Catch2](https://github.com/catchorg/Catch2) -  [Phil Nash](https://twitter.com/phil_nash) 以及 [Martin Hořeňovský](https://twitter.com/horenmar_ctu) 的测试框架
- [doctest](https://github.com/onqtam/doctest) - 与 Catch2 类似，优化了编译期性能
- [Google Test](https://github.com/google/googletest)
- [Boost.Test](https://www.boost.org/doc/libs/1_84_0/libs/test/doc/html/index.html) - boost 风格的测试框架

[ctest](https://cmake.org/cmake/help/latest/manual/ctest.1.html) 是 CMake 的 test runner，支持以上所有框架。通过 CMake 的 [`add_test`](https://cmake.org/cmake/help/latest/command/add_test.html) 使用。

如果项目设计正确，那么很容易进行测试。

## 练习

### 你能以一个命令就运行所有 tests 吗？

- Yes: Excellent! Run the tests and make sure they all pass!
- No: Does your program produce output? 
  - Yes: Start with “Approval Tests,” which will give you the foundation you need to get started with testing. 
  - No: Develop a strategy for how to implement some minimal form of testing.

