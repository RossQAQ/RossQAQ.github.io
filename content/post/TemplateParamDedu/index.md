---
title: 模板参数推导 
description: C++ 模板参数推导规则
slug: Template-Arg-Deduction
date: 2024-02-13 00:00:00+0000
image: 
categories:
    - cppnotes
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

## C++ 模板实参推导

大致总结一下 C++ 的模板实参推导规则，专注于重点。

C++ 本身有一堆复杂的规则，但我只挑我目前理解中常用的。本节内容只记录 *函数模板* 推导规则。C++17 后支持 *类模板* 参数推导，当然，只在构造时才能推导。

首先关于实参推导，标准的描述是：

> 为了实例化一个[函数模板](https://zh.cppreference.com/w/cpp/language/function_template)需要知晓每个模板实参，但并非每个模板实参都必须予以指定。编译器会尽可能从函数实参推导缺失的模板实参。

从函数推导的规则：

> 模板实参推导试图确定模板实参（类型模板形参 **T**i 的类型，模板模板形参 **TT**i 的模板，和非类型模板形参 **I**i 的值），它们在经过以下列出的调整之后可以代换到各个函数形参 **P** 中，以产生*推导*的类型 **A**，它与函数实参 **A** 类型相同。
>
> 如果有多个形参，那么分别推导每一对 **P**/**A**，然后合并各个推导的模板实参。如果推导失败，或任何一对 **P**/**A** 或有歧义，或如果不同对推导出的模板实参不同，或如果还遗留有任何模板实参既没有被推导也没有被显式指定，那么编译失败。

注意 P/A是一对，对不上也会编译失败。

意思是说，对于：

```cpp
template <typename T>
T max(T, T);

max(1.0, 2);	// 推导失败，编译器不知道要的是 int 还是 double
max<double>(1.0, 2);	// ok，显式指定
// 注：这里的模板参数是 P，调用函数传递的参数是 A
```

基本上推导是符合直觉的，但会做出以下处理：

1. 若 P 不是引用类型，那么若 A 是数组类型、函数类型，都会退化为相应指针；如果 A 有 cv 限定符，也会忽略（意思就是对 A 使用 std::decay_t。 题外话，所有进行按值传递函数实参时都会进行这种转换 ）
2. 如果 P 有cv 限定，推导时会忽略顶层 cv 限定符
3. 若 P 是引用类型，那么用 P 引用的类型推导
4. **如果 P 是万能引用，且对应函数调用实参是左值，那么 A 的左值引用类型会用于 A 的位置进行推导。（std::forward 的基础）**



其次，必须要知道什么时候进行推导，什么时候不推导，即 **不推导语境**

（常见的比如 std::forward，就是个明显的不推导语境）

> 下列情况下，用来组成 **P**的类型、模板和非类型值不会参与模板实参推导，而改为*使用*可以在别处推导出或显式指定的模板实参。如果模板形参只在不推导语境使用且没有被显式指定，那么模板实参推导失败。

也就是说，这种情况下要么已经推导过了，要么要进行指定，否则会推导失败。

1. 作用域解析运算符 `::` 左侧的所有内容（**最常见的情况**）。

   ```cpp
   // in C++20
   template<typename T>
   void bad(std::vector<T> x, T value = 1);
    
   template<typename T>
   void good(std::vector<T> x, std::type_identity<T>::type value = 1);
   
   std::vector<std::complex<double>> x;
   
   bad(x, 1.2);
   // P1/A1: T = std::complex<double>
   // P2/A2: T = double
   // 推导失败
   
   good(x, 1.2);
   // 成功，后者不是推导语境，使用的是前者推导出的 T，即 std::complex<double>
   ```

2. decltype 内的表达式

3. 非类型模板实参或数组边界，**子表达式引用一个模板形参**

4. 形参 P，其实参 A 是花括号初始化器列表，但 P 不是 std::initializer_list ，到他的引用，或者到数组的引用（C++17）

   也比较常见，这个loser_list 纯纯的坑

   ```cpp
   template<class T>
   void g1(std::vector<T>);
    
   template<class T>
   void g2(std::vector<T>, T x);
    
   g1({1, 2, 3});     // P = std::vector<T>，A = {1, 2, 3}：T 在不推导语境中
                      // 错误：T 没有被显式指定或从另一对 P/A 推导出
    
   g2({1, 2, 3}, 10); // P1 = std::vector<T>，A1 = {1, 2, 3}：T 在不推导语境中
                      // P2 = T，A2 = int：推导出 T = int
   ```

## 参考资料

[模板实参推导](https://zh.cppreference.com/w/cpp/language/template_argument_deduction)