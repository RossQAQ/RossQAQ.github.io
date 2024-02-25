---
title: emplace_back vs. push_back
description: C++ Weekly Ep 108 and Ep 278.
slug: cppweekly-emplace-back
date: 2024-02-23 00:00:00+0000
image: 
categories:
    - cppweekly
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

C++ Weekly with Jason Turner.

[C++ Weekly - Ep 108 - Understanding emplace_back](https://www.youtube.com/watch?v=uwv1uvi1OTU)

[C++ Weekly - Ep 278 - `emplace_back` vs `push_back`](https://www.youtube.com/watch?v=jKS9dSHkAZY)

两期的总结。

## 理解 emplace_back

emplace_back 代表的是**原地构造**，直接上代码：

```cpp
struct S {
    S() {puts("S()");}
    S(const S&) {puts("S(const S&)");}
    S(S&&) {puts("S(S&&)");}
    S& operator=(const S&) {puts("S(const S&)="); return *this;}
    S& operator=(S&&) {puts("S(S&&)="); return *this;}
    ~S() {puts("~S()");}
};

int main() {
    std::vector<S> vec;
    vec.push_back(S{});
    // S() S(S&&) ~S() ~S()
    // 如果使用 emplace_back
    vec.emplace_back();
    // S() ~S()
}
```

观察更细致一些：

```cpp
struct S {
    S(int) {puts("S(int)");}
    ...
};

int main() {
    std::vector<S> vec;
    vec.emplace_back(5);
    // S(int) ~S()
}
```

## emplace_back vs. push_back

```cpp
#include <vector>
#include <string>

int main() {
    std::vector<std::string> vec;
    // 如果需要构造对象，就使用 emplace_back
    vec.emplace_back(100, 'c');
    
    // 1. 给 string 分配空间 (resize)
    // 2. placement new() into new space (args...)
    
    // 如果已经有对象，那么就使用 push_back
    vec.push_back(std::string(100, 'c'));
    
    // 1. 在栈上创建临时对象
    // 2. resize vector
    // 3. std::move 来移动到新的位置
    
    // vec.emplace_back(std::string(100, 'c'));
    // emplace_back() 的错误用法
    // 先创建临时 string
    // 给 string 分配空间
    // placement new() （移动构造）
}
```

此外 emplace_back 是个成员函数模板

总之，push_back(std::move(obj)) 和 emplace_back 差不多

如果 push_back with copy 就很慢。