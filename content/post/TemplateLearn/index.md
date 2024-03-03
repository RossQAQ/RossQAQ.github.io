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

## 待决名

> 在模板（类模板和函数模板）定义中，某些构造的含义可以在不同的实例化间有所不同。特别是，**类型和表达式可能会取决于类型模板形参的类型和非类型模板形参的值**。

### typename 消除待决名歧义

```cpp
template<typename T>
const T::type& f(const T&) {
	return 0;
}

struct X {
	using type = int;
};

X x;
f(x);
```

无法编译，因为编译器认为 T::type 是标识符，我们需要加上 typename 提示编译器他是类型

```cpp
template<typename T>
const typename T::type& f(const T&) {...}
```

> **在模板（包括别名模版）的声明或定义中，*不是当前实例化的成员且取决于某个模板形参的名字* 不会被认为是类型，除非使用关键词 typename 或它已经被设立为类型名（例如用 typedef 声明或通过用作基类名）**。

```cpp
int p = 1;

template<typename T>
void foo(const std::vector<T>& v){
    // std::vector<T>::const_iterator 是待决名，
    typename std::vector<T>::const_iterator it = v.begin();

    // 下列内容因为没有 'typename' 而会被解析成
    // 类型待决的成员变量 'const_iterator' 和某变量 'p' 的乘法。
    // 因为在此处有一个可见的全局 'p'，所以此模板定义能编译。
    std::vector<T>::const_iterator* p;

    typedef typename std::vector<T>::const_iterator iter_t;
    iter_t* p2; // iter_t 是待决名，但已知它是类型名
}

int main(){
    std::vector<int>v;
    foo(v); // 实例化失败
}
```

### template 消除歧义符

> ***模板定义中 不是当前实例化的成员 的待决名 同样不被认为是模板名*，除非使用消歧义关键词 template，或它已被设立为模板名**

```cpp
template<typename T>
struct S {
	template<typename U>
	void foo() {}
};

template<typename T>
void bar() {
	S<T> s;
	s.foo<T>();
    // s.template foo<T>();
}
```

**`template` 的使用比 `typename` 少，并且 `template` 只能用于 `::`、`->`、`.` 三个运算符 \*之后\*。**

### 非待决名绑定规则

> 非待决名在模版定义点查找并绑定，即使模板实例化点有更好的匹配，也保持此绑定。

### 待决与非待决查找规则

- 有限名字查找？
- 无限名字查找？

> 对于在模板的定义中所使用的**非待决名**，当**检查该模板的定义时将进行无限定的名字查找**。在这个位置与声明之间的绑定并不会受到在实例化点可见的声明的影响。而对于在模板定义中所使用的**待决名**，**它的查找会推迟到得知它的模板实参之时**。此时，ADL 将同时在模板的定义语境和在模板的实例化语境中检查可见的具有外部连接的 (C++11 前)函数声明，而非 ADL 的查找只会检查在模板的定义语境中可见的具有外部连接的 (C++11 前)函数声明。（换句话说，在模板定义之后添加新的函数声明，除非通过 ADL 否则仍是不可见的。）如果在 ADL 查找所检查的命名空间中，在某个别的翻译单元中声明了一个具有外部连接的更好的匹配声明，或者如果当同样检查这些翻译单元时其查找会导致歧义，那么行为未定义。无论哪种情况，**如果某个基类取决于某个模板形参，那么无限定名字查找不会检查它的作用域（在定义点和实例化点都不会）**。

很长，但是看我们加粗的就够：

- 非待决名：检查该模板的定义时将进行无限定的名字查找
- 待决名：它的查找会推迟到得知它的模板实参之时

**这个故事告诉我们，this加不加是真的有区别的（this依赖模板参数是待决名）**

## SFINAE

### SFINAE?

“代换失败不是错误” (Substitution Failure Is Not An Error)

在**函数模板的重载决议**[1](https://github.com/Mq-b/Modern-Cpp-templates-tutorial/blob/main/md/第一部分-基础知识/10了解与利用SFINAE.md#user-content-fn-1-bb04513483baf397e0e69541c096e9f5)中会应用此规则：当模板形参在替换成显式指定的类型或推导出的类型失败时，从重载集中丢弃这个特化，*而非导致编译失败*。

此特性被用于模板元编程。

```cpp
template<typename T, typename T2 = typename T::type>
void f(int) { std::puts("int"); }

f<int>(5);
```

会报错：未找到匹配的重载函数

因为这里显然 int::type 非良构（不符合语法），代换失败，会丢弃特化，但又没有找到其他重载函数。

```cpp
template<typename T>
void f(double) { std::puts("double"); }
```

这里会选择到 double 版本。

可以用 typename + decltype 写条件，即可对传入的类型做出要求，比如 operator+ 等等

对模板形参会进行两次代换（推导前指定，推导后）

### 代换失败与硬错误

> **只有在函数类型或其模板形参类型或其 explicit 说明符 (C++20 起)的 *立即语境* 中的类型与表达式中的失败，才是 *SFINAE 错误*。**
>
> **如果对代换后的类型/表达式的 *求值导致副作用*，例如实例化某模板特化、生成某隐式定义的成员函数等，那么这些副作用中的错误都被当做 *硬错误* **。

SFINAE 可以影响重载决议。

尤其注意，进行实例化是硬错误。

```cpp
template<typename T>
struct B {
	using type = typename T::type;
};

template<typename T>
void foo(double) { std::puts("SFINAE"); }

template<
	class T,
	class U = typename T::type,		// 如果T 没有 type，就代换失败，如果没有这一行会因为下一行的 B<T> 硬错误编译失败
	class V = typename B<T>::type	// 这里就是实例化
>
void foo(int) { std::puts("SFINAE T::type, B<T>::type"); }

int main() {
	foo<void>(1);
}
```

### SFINAE 基础示例

需要写一个 add，要求其类型支持 operator+

```cpp
// 自然可以这里写 decltype，但是这样比较蛋疼
template<typename T, typename = decltype(T{} + T{}) >
auto add(const T& a, const T& b) {
	return a + b;
}

// 使用 C++11 后置返回类型，此时知道 a 和 b 的类型
template<typename T>
auto add(const T& a, const T& b) -> decltype(a + b) {
	return a + b;
}
```

多用 SFINAE 等约束才能有更友好的报错和编译速度。

### std::enable_if

如何要求模板提供的类型？

第一个参数接受一个返回 bool 值的表达式

```cpp
template<bool B, class T = void>
struct enable_if {};
 
template<class T> // 类模板偏特化
struct enable_if<true, T> { typedef T type; };     // 只有 B 为 true，才有 type，即 ::type 才合法

template< bool B, class T = void >
using enable_if_t = typename enable_if<B,T>::type; // C++14 引入
```

这是一个模板类，在 C++11 引入，它的用法很简单，就是第一个模板参数为 true，此模板类就有 `type`，不然就没有，以此进行 SFINAE。

为 false，那么会因为 SFINAE 不选择 true 的那个模板，从而错误。



C++11，要求提供类型为 int，17之后就可以写的很简单了

这里第二个typename 纯粹用来做 SFINAE，所以没有名字也无所谓

```cpp
template<typename T, typename = typename std::enable_if<std::is_same<T, int>::value>::type>	//C++11
void f(T) {}

template<typename T, typename = std::enable_if_t<std::is_same_v<T, int>>>					//C++17
void f(T) {}
```

再例如 array，之前推导指引时候写过的，升级版本

```cpp
template <class Type, class... Args>
array(Type, Args...) -> array<std::enable_if_t<(std::is_same_v<Type, Args> && ...), Type>, sizeof...(Args) + 1>;
```

`(std::is_same_v<Type, Args> && ...)` 做 std::enable_if 的第一个模板实参，这里是一个一元右折叠，使用了 **`&&`** 运算符，也就是必须 std::is_same_v 全部为 true，才会是 true。**简单的说就是要求类型形参包 Args 中的每一个类型全部都是一样的，不然就是替换失败。**

### std::void_t

```cpp
template<class...>
using void_t = void;
```

用此元函数检测 SFINAE 语境中的非良构类型

void_t 防止你写一堆模版参数，这样你就可以写在一起只写一个 typename 了

需求：函数模板 add，需要传入的对象支持 operator+，有别名 type，成员value f

```cpp
template<typename T, typename = 
	std::void_t<decltype(T{} + T{}), typename T::type, decltype(&T::value), decltype(&T::f) >>
auto add(const T& a, const T& b) {
	return a + b;
}
```

### std::declval

在上面的 SFINAE 中，decltype(T{}+T{}) 要求 T 能默认构造，显然不正确，我们要的是operator+

我们需要使用 std::declval

```cpp
decltype(std::declval<T>() + std::declval<T>());
```

此时就没有问题了。

```cpp
template<class T>
typename std::add_rvalue_reference<T>::type declval() noexcept;
```

这个函数特殊，只能用于不求值语境，不要求 T 有定义

### 偏特化中的 SFINAE

在确定一个类或变量 (C++14 起)模板的特化是由部分特化还是主模板生成的时候也会出现推导与替换。在这种确定期间，**部分特化的替换失败不会被当作硬错误，而是像函数模板一样\*代换失败不是错误\*，只是忽略这个部分特化**。

## 约束与概念

### 前言

C++20 的约束与概念，再也不用写蛋疼的 SFINAE

### 约束与概念

类模板，函数模板，以及非模板函数（通常是类模板的成员），可以与一项约束（constraint）相关联，它指定了对模板实参的一些要求，这些要求可以被用于选择最恰当的函数重载和模板特化。

这种**要求的具名集合**被称为***概念（concept）***。每个概念都是一个谓词，它在**编译时求值**，并在将之用作约束时成为模板接口的一部分。

```cpp
// 还是add，要求传入的对象支持 operator+

// 定义概念，概念是模板
// 概念，要求约束表达式成立
template<typename T>
concept Add = requires(T a) {
    a + a;
};

// 使用概念
template<Add T>
auto add(const T& a, const T& b) {
    return a + b;
}

// 自然也可以
constexpr bool v = Add<int>; //true
```

概念定义, **约束表达式只要求在编译期求值，返回 bool 即可**

```cpp
template < 模板形参列表 >
concept 概念名 属性 (可选) = 约束表达式;
```

### 简写函数模板与标准概念库

简写函数模板：

```cpp
decltype(auto) max(const auto& a, const auto& b) {
    return a > b ? a : b;
}
```

如果约束传入的对象怎么写？我们可以使用标准库设置，位于 `concepts`

```cpp
#include <concepts>
decltype(auto) max(const std::integral auto& a, const std::integral auto& b) {
    return a > b ? a : b;
}
```

**此外，概念可以在所有使用 auto 的前面使用。**当然也可以要求个普通变量。

变量模板、类模板都同理。

### requires 子句

requires **只要求编译期求值的表达式**

```cpp
template<typename T>
concept Add = requires(T a) {
    a + a;
};

template<typename T>
	requires add<T>
void f(T) {}

template<typename T>
	requires(sizeof(T) >= 4)
void g(T) {}

// 甚至可以
template<typename T>
	requires requires(T a) { a + a; }
void h(T) {}
// 第一个是 requires 子句，为 true 才会选择这个模板
// 第二个是 requires 表达式，恰好编译器求值
```

### 约束 - 合取析取

约束是逻辑操作和操作数的序列，它指定了对模板实参的要求。它们可以在 requires 表达式（见下文）中出现，也可以直接作为概念的主体。

有三种类型的约束：

1. 合取（conjunction）(&&)
2. 析取（disjunction） (||)

### requires 表达式

**产生描述约束的 bool 类型的纯右值表达式**。

> *注意，`requires` 表达式 和 `requires` 子句，没关系*。

```cpp
requires { 要求序列 }
requires ( 形参列表 (可选) ) { 要求序列 }
```

requires 表达式可以检测表达式是否合法，语句不合法会返回 false，而不会认为程序非脸狗

```cpp
template<typename T>
void f(T) {
    constexpr bool v = requires { typename T::type;};	// 待决名，不加 typename 会认为他是变量
}

int main() {
    f(1); 	// v 此时为 false
}
```

### 简单要求

简单要求是任何不以关键词 requires 开始的表达式语句。它断言该表达式是有效的。表达式是不求值的操作数；只检查语言的正确性。

例如前文的 Add 概念的 requires 表达式

```cpp
template<typename T, typename U>
concept Swappable = requires(T&& t, U&& u) {
    swap(std::forward<T>(t), std::forward<U>(u));
    swap(std::forward<U>(u), std::forward<T>(t));
}
```

### 类型要求

类型要求是关键词 **`typename`** 后面接一个可以被限定的**类型名称**。该要求是，所指名的类型是有效的。

可以用来验证：

1. 某个**指名的嵌套类型**是否存在。(typename T::type)
2. 某个**类模板特化**是否指名了某个类型。(typename S\<T>)
3. 某个**别名模板特化**是否指名了某个类型。(using)

### 复合要求

```
{ 表达式 } noexcept(可选) 返回类型要求 (可选) ;
```

> 返回类型要求：-> 类型约束（*概念* concept）

并断言所指名表达式的属性。替换和语义约束检查按以下顺序进行：

1. 模板实参 (若存在) 被替换到 表达式 中；
2. 如果使用了`noexcept`，表达式 一定不能潜在抛出；
3. 如果返回类型要求存在，则：
   - 模板实参被替换到*返回类型要求* 中；
   - `decltype((表达式))` 必须满足*类型约束* 蕴含的约束。否则，被包含的 requires 表达式是 **`false`**。

```cpp
template<typename T>
concept C2 = requires(T x) {
    // 首先 *x 得合法
    // 类型要合法，这里要有嵌套类型
    // 并且 *x 的结果必须可以转换为 T::inner
    {*x}->std::convertible_to<typename T::inner>;
    
    // 表达式 x + 1 必须合法
    // 并且 std::same_as<decltype((x + 1)), int> 必须满足
    // 即, (x + 1) 必须为 int 类型的纯右值
    // std::same_as 是个概念
    {x + 1} -> std::same_as<int>;

    // 表达式 x * 1 必须合法
    // 并且 它的结果必须可以转换为 T
    {x * 1} -> std::convertible_to<T>;
        
    // 复合："x.~T()" 是不会抛出异常的合法表达式
    { x.~T() } noexcept;
}
```

### 嵌套要求

嵌套要求具有如下形式

```
requires 约束表达式 ;
```

就是在 requires 里再写一个 requires

```cpp
template<typename T>
concept C3 = requires(T a, std::size_t n) {
    requires std::is_same_v<T*, decltype(&a)>;     // 要求 is_same_v          求值为 true
    requires std::same_as<T*, decltype(new T[n])>; // 要求 same_as            求值为 true
    requires requires{ a + a; };                   // 要求 requires{ a + a; } 求值为 true
    requires sizeof(a) > 4;                        // 要求 sizeof(a) > 4      求值为 true
};
std::cout << std::boolalpha << C3<int> << '\n';    // false
std::cout << std::boolalpha << C3<double> << '\n'; // true
```

在上面示例中 `requires requires{ a + a; }` 其实是更加麻烦的写法，目的只是为了展示 `requires` 表达式是编译期产生 `bool` 值的表达式，所以有可能会有**两个 `requires`连用的情况**；我们完全可以直接改成 `a + a`，效果完全一样。

这里用 `std::is_same_v` 和 `std::same_as` 其实毫无区别，因为它们都是编译时求值，返回 `bool` 值的表达式。

### 总结

总之记住：

可以连用 `requires requires` 的情况，都是因为第一个 `requires` 期待一个可以编译期产生 `bool` 值的表达式；而 **`requires` 表达式就是产生描述约束的 bool 类型的纯右值表达式**。
