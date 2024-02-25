---
title: 现代 C++ 模板应该学习什么？
description: 阅读 mq白 现代 C++ 模板教程知识点记录。
slug: Modern-Cxx-Template
date: 2024-02-12 09:00:00+0000
image: 
categories:
    - cppnotes
    - continuous
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[教程原文](https://github.com/Mq-b/Modern-Cpp-templates-tutorial/blob/main/md/)

## 函数模板

### 使用函数模板

1. 函数模板使用时才会实例化（隐式实例化）

### 函数模板参数推导规则

1. 函数模板参数推导规则、无法推导的情况
2. ADL 对于函数模板调用时的影响
3. 万能引用、引用折叠，特殊推导规则

### 函数模板默认实参

1. 可以给模板类型实参默认值，既然是类型，那么默认值也要类型

   （func<>() 代表使用默认值）

2. 推导时，P/A 对无法推导（例如3P 2A对不上的情况）下的处理

   默认实参？部分指明？通过 decltype 三目表达式配合 decay_t 获取 common_type？

   后置返回值类型配合 decltype

   对于 C++20，甚至可以使用简写函数模板。注意 auto 和 decltype 的推导规则

### 非类型模板实参

模板参数接收值或者对象，而不是类型

```cpp
template <std::size_t N = 10>
void func() { std::cout << N << '\n'; }

func<5>();
```

### 重载函数模板

函数模板与普通函数都能重载，重载决议规则非常复杂。

一般而言，会优先选择非模板的函数。（毕竟隐式实例化也有开销）

### 可变参数模板

C 语言的可变参数，最常见的例子就是 printf ，[参见](https://github.com/Mq-b/Loser-HomeWork/blob/main/C++CoreGuidelines/第4章-函数.md#f55-不要使用-va_arg-参数)。

C++ 中若想使用可变参数，则必须使用模板。

[**形参包**](https://zh.cppreference.com/w/cpp/language/parameter_pack)

- 如何实现支持任何类型，任何参数的调用？

- 类型形参包：存类型

- 函数形参包：存参数

- 如何使用？形参包展开

- 什么是**模式**？（[形参包名]... 中的形参包名是模式，其会被展开为 0 或多个逗号分隔的模式实例）

举个例子

```cpp
template<typename... Args>
void print(const Args&... args) {
    int _[] { (std::cout << args << ' ', 0)... };
}
// (std::cout << arg0 << ' ', 0), (std::cout << arg1 << ' ', 0), (std::cout << arg2 << ' ', 0) 会展开成这种形式
// 
// (std::cout << args << ' ', 0) 是模式
// 逗号表达式，从左往右顺序执行。
// , 0，会返回0 然后初始化数组，数组没啥用，只是用于写成这个形式，因为 {} 可以进行包展开。

template<typename T, std::size_t N, typename... Args>
void f(const T(&array)[N], Args... index) {
    print(array[index]...);	// 模式 array[index]
}
// const T(&array)[N] 数组引用
// 内建数组，其 size 是他类型的一部分，所以可以被推导

// const char (&)[5], const int&, const double&
// const char (&)[5] -> const char*
print("luse", 1, 1.2);

int array[10] {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
f(array, 1, 3, 5);
```

那么如何写一个函数 sum，支持任何类型，任意参数调用

```cpp
template <typename... Args, typename RT = std::common_type_t<Args...>>
RT sum(const Args&... args) {
    RT _[] { args... };
    RT n{};
    for (int i = 0; i < sizeof...(Args); ++i) {
        n += _[i];
    }
    return n;
}
```

### 模板分文件

**显然不可能**（当然显式实例化除外）

## 类模板

### 初始类模板

类模板不是类，实例化类模板才能生成类。

函数模板中形参列表可以写的类模板都能写。

C++17 后，CTAD 可以根据传入的参数推导类模板

### 用户定义推导指引

```cpp
template_name(deduced_type) -> template_name<the type you want>;
// e.g.
Test(int) -> Test<size_t>;
```

比较复杂的：

```cpp
template <typename Ty, std::size_t size>
struct array {
	Ty arr[size];
};

template <typename T, typename... Args>
array(T t, Args...) -> array<T, sizeof...(Args) + 1>;
```

### 类模板有默认实参的模板形参

```cpp
template <typename T = int>
struct X {};

struct Test {
    X x;	// C++17 也不行，类中的不允许省略 <>
};

X x; 		// C++17 ok
X<> x; 		// before C++17
```

应用很多，例如 vector，string 等等，分配器。

### 模板模板形参

如果想要模板参数接受一个类模板怎么办？

```cpp
template<typename T>
struct X {};

// before C++17 template<template<typename T> class C>
template<template<typename T> typename C>
struct Test {};

Test<X>arr;
```

- template \<typename T> typename C 是模板模板形参语法 

- typename C 是模板模板形参，可以自定义

模板模板形参包也可以

```cpp
template<template<typename T> class... Args>

test<X, Y, Z> t;
```

```cpp
template <std::size_t>
struct Y{};

template <template<std::size_t> class T>
struct X{};
```

```cpp
template <typename... Args>
struct Y{};

template <template<typename... Args> class T>
struct X{};

X<Y> t;
```

基本就是，要接受什么，就复制其模板声明，之后再声明一个 T

### 成员函数模板

```cpp
// 模板类和函数
template<typename T>
struct X {
    void f(T arg) {}
};
X<int> x;
x.f(123);

// 模板类和模板函数
template<typename T>
struct X {
    template<typename... Args>
    void f(Args&&... args) {}
};
X<int, double, float> x;
x.f(1, 2, 3);

// 普通类和模板函数
struct X {
    template<typename... Args>
    void f(Args&&... args) {}
};
X x;
x.f(1, 2, 3);
```

### 可变参数类模板

```cpp
#include <iostream>
#include <tuple>

template<typename... Args>
struct X {
	X(Args... args) : value{ args... } {}
	std::tuple<Args...> value;
};

int main() {
	X x{ 1, 2.1, "2", '3' };
	std::cout << std::get<2>(x.value);
}
```

可以使用 cpp insights 辅助。

## 变量模板

### 初识变量模板

```cpp
template <typename T>
T v;
```

跟函数模板、类模板同理，变量模板也不是变量，实例化之后是全局变量。自然也可以 cv 修饰等等。

### 变量模板默认实参与非类型模板实参

跟函数、类同理。

### 可变参数变量模板

```cpp
template <typename... Args>
size_t N = sizeof...(Args);

template <std::size_t... value>
constexpr std::size_t array[] {value...};

for (const auto& i : array<1, 2, 3, 4, 5>) {
    std::cout << i << ' ';
}
```

### 类静态数据成员模板

首先说类的普通静态成员

```cpp
struct X {
    static int n;	// 声明，没有定义
};
int X::n;			// 类外定义，当然也可以给初始值
```

```cpp
struct X {
	static const int n = 10;	// 不是定义，还是声明
};
// 这种情况能用是因为
// 读取编译时常量，不是 ODR 调用，没有违反 ODR
// 如果要单一使用，就必须要定义
```

```cpp
struct X {
	static inline int n = 10;	// C++17, 定义，可以 ODR 使用
};
```

```cpp
struct X {
	static constexpr int n = 10;	// C++17, 定义
};
// static constexpr 自带 inline 属性，所以可以 ODR 使用
```

来看看类的静态变量模板

```cpp
struct limits {
	template<typename T>
    static const T min;	// 声明
};
template<typename T>
const T limits::min = {};	// 静态数据成员模板定义
```

## 模板全特化

### 函数模板全特化

特化，对某些类型进行定制的操作

```cpp
template<typename T,typename T2>
auto f(const T& a, const T2& b)  {
    return a + b;
}

template<>
auto f<double, int>(const double& a, const int& b) {
    return a - b;
}
```

### 类模板全特化

实现个 std::is_void

```cpp
template<typename T>
struct is_void {
    static constexpr bool value {false};
};

template<>
struct is_void<void> {
    static constexpr bool value {true};
}
```

C++17 还引入了 _v 版本

使用的是变量模板

```cpp
template<typename T>
constexpr bool is_void_v = is_void<T>::value;
```

注意不同实例化的模板类之间没有任何关系，他们是互相独立的

```cpp
template <typename T>
struct X {};

template<>
struct X<int> {
    void f() {}
};
```

### 变量模板全特化

没什么好说。

### 模板全特化细节

- 特化必须在发生隐式实例化之前，在使用到其的翻译单元中声明
- 只有声明没有定义的模板特化可以像其他不完整类型一样使用（例如使用到他的指针或引用

- 函数模板和变量模板的显式特化是否为 inline/constexpr/constinit/consteval 只与特化自身有关。主模版的声明是否带有对应说明符对其没有影响。模板声明中出现的属性在它的显式特化中也没有效果。

### 特化的成员与总结

```cpp
template<typename T>
struct A {
	struct B {};

	template<typename U>
	struct C {};
};
```

特化后类内成名，类外定义：

```cpp
template<>
struct A<void> {
	void f();
};

void A<void>::f() {}

A<void> a;
a.f();
```

特化 A\<char> 情况下的 B 成员类

```cpp
template<>
struct A<char>::B {
	void f();
};

void A<char>::B::f() {}

A<char>::B b_c;
b_c.f();
```

特化成员类模板 A\<int> C 的定义

```cpp
template<>
template<typename U>
struct A<int>::C {
	void f();
};

template<>
template<typename U>
void A<int>::C<U>::f() {}

A<int>::C<void> c_v;
c_v.f();
```

**特化普通类的成员函数模板**

```cpp
struct X {
	template<typename T>
	void f(T) {}

	template<>
	void f<int>(int) {}	// 可以类内直接特化
};

template<>
void X::f<double>(double) {}	// 类外特化
```

**特化类模板的成员函数模板**

```cpp
template<typename T>
struct X {
	template<typename U>
	void f(U) {}
};

template<>
template<>
void X<int>::f<char>(char) {}
```

## 模板偏特化

对有共同一类特征的类模板、变量模板进行定制行为（函数模板不能偏特化）

### 变量模板偏特化

```cpp
template<typename T>
const char* s = "?";

template<typename T>
const char* s<T*> = "pointer";	// 对指针这一类类型特化

template<typename T>
const char* S<T[]> = "array";	// 针对 T[] 进行偏特化，而不是数组类型
								// T[] 和 T[N] 肯定不是一个类型

template<typename T, typename T2>
const char* s = "?";

template<typename T2>
const char* s<int, T2> = " T == int ";
```

### 类模板偏特化

```cpp
template<typename T, typename T2>
struct X {
	void f() const {}
};

template<typename T>
struct X<void, T> {
    void g() const;
}

template<typename T>
void X<void, T>::g() const {}	// 不建议写到类外
```

```cpp
template<typename T, std::size_t N>
struct X {
	template<typename U, typename V> 
    struct Y{};
};

template<>
template<typename V>
struct X<int, 5>::Y<int, V> {
    void f() const {}
}
```

### 实现 is_same_v

```cpp
template<class, class>
struct is_same {
	static constexpr bool value{ false };
};

template<class Ty>
struct is_same<Ty, Ty> {
	static constexpr bool value{ true };
};

// 变量模板
template <class T, class T2>
constexpr bool is_same_v = is_same<T, T2>::value;
```

当然也可以用变量模板直接写

```cpp
template <class, class>
constexpr bool is_same_v = false;

template <class Ty>
constexpr bool is_same_v<Ty, Ty> = true;
```

## 函数模板显式实例化解决分文件问题

### 函数模板

```cpp
template<typename T>
void f(T) {
	std::cout << typeid(T).name() << "\n";
}

template void f(int);	// 编译器会实例化 f<int>(int)
```

之后就可以把模板放在头文件，在cpp文件内显式实例化，然后其他 cpp 引入头文件就可以使用了。

### 类模板

类的完整定义不包含成员函数的完整定义，所以可以创建对象，但是不能调用成员函数。

需要显式实例化成员函数

```cpp
template<typename T>
struct X {
    void f();
};

template void X<int>::f();
```

## 折叠表达式

### 一元

C++17 引入折叠表达式，更方便进行**形参包展开**

之前在新参包中的写法非常愚蠢，在C++17之后就可以使用折叠表达式了。

折叠表达式是左折叠还是右折叠，取决于 `...` 的位置

```cpp
template<typename... Args>
void print(const Args&... args) {
    ((std::cout << args << ' '), ...);
}
// 一元 右折叠 -> (E 运算符 ...) : E-> (std::cout << args << ' ')；运算符-> , 
// 剩下点和括号就不说了，括号是折叠表达式语法的一部分
// 展开为 (E1 运算符 (... 运算符 (EN-1 运算符 EN)))
```

```cpp
template<typename... Args>
void print(const Args&... args) {
    (..., (std::cout << args << ' '));
}
// 一元 左折叠
// (... 运算符 E) -> (((E1 运算符 E2) 运算符 ...) 运算符 EN)
```

那打印顺序会变吗？不会！因为逗号从左到右执行，不管左折叠还是右折叠都不影响。当然其他运算符会有一些区别。

```cpp
template<int... I>
constexpr int v_r = (I - ...);	// 一元右折叠

template<int... I>
constexpr int v_l = (... - I);	// 一元左折叠

int main() {
	std::cout << v_r<4, 5, 6> << '\n';
	std::cout << v_l<4, 5, 6> << '\n';
}
// v_r (4-(5-6)) = 5
// v_l ((4-5)-6) = -7
```

大部分会求值的表达式会影响结果。

### 二元

二元会长成:  **运算符 ... 运算符** 

二元左折叠 (... 在形参包左边)

`(I 运算符 ... 运算符 E)`，I 是初值表达式

```cpp
template<typename... Args>
void print(Args&&... args) {
	(std::cout << ... << args);
}
// 这样写，在打印的情况下会比较抽象。
```

- I：std::cout

- 运算符: << .. <<
- E: args

展开形式和一元的很像 `((((I 运算符 E1) 运算符 E2) 运算符 ...) 运算符 EN)`，只不过多了一个 I，右折叠同理，最后多了 I

```cpp
template<int... I>
constexpr int v_r = (I + ... + 10);	//二元右折叠

template<int... I>
constexpr int v_l = (10 + ... + I);

int main() {
	std::cout << v_r<1, 2, 3, 4> << '\n';
	std::cout << v_l<1, 2, 3, 4> << '\n';
}
```

结果显然都是 20

v_r:  `(1+(2+(3+(4+10))))`

v_l:  `((((10+1)+2)+3)+4)`

初值无论如何都是第一个计算。
