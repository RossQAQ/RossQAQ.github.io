---
title: 智能指针
description: 对 CppCon 2019, Arthur O'Dwyer 演讲的翻译与总结。
slug: CppCon 2019 Smart Pointers
date: 2023-12-25 00:00:00+0000
image: cppcon2019-cover.png
categories:
    - cppcon
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[CppCon 2019: Arthur O'Dwyer “Back to Basics: Smart Pointers”](https://www.youtube.com/watch?v=xGDLkt-jBJ4)

C++11 的面试圣经 —— 智能指针

我发现这个人讲的还是不错的，就是语速太快……不过好在他的 lecture 都有官方字母。

## 智能指针发展历史

`auto_ptr` : C++98 的遗老，C++17被移除

`unique_ptr`: C++11. 用来替代 auto_ptr，C++14 加入了 `std::make_unique()`

`shared_ptr`: C++11. 引用计数，支持`std::make_shared()` 。C++17 加入了 `std::shared_ptr<T[]>`

`weak_ptr` : C++11. 弱引用。

C++20 : `std::make_shared<T[]>`

## std::unique_ptr ：独占所有权

`std::unique_ptr` 可以自动替你管理资源。

原始指针是可以拷贝的，那么如果我拷贝了原始指针，那么谁来清理资源呢？这个不太好说。

`std::unique_ptr` 是 move-only，它的移动构造会将原来的指针置为 nullptr。

只有一个指针指向资源，`std::unique_ptr` 会自动帮你管理资源。

此外，`std::unique_ptr` 有一个对 T[] 的特化：

```cpp
template<typename T>
class unique_ptr<T[]> {
	T* p_ {nullptr};
    
    ~unique_ptr() {
        delete [] p_;
    }
};
```

`std::unique_ptr` 还有一个模板参数：Deleter。你可以显式的传入一个 deleter。

```cpp
template<typename T, typename Deleter = std::default_delete<T>>
class unique_ptr {
	T* p_{nullptr};
    Deleter d_;
    
    ~unique_ptr() {
        if (p_) d_(p_);
    }
};

template<typename T>
struct default_delete {
	void opeartor()(T *p) const {
        delete p;
    }
};
```

假设我们使用一个 FILE*

```cpp
struct FileCloser {
	void operator()(FILE* fp) const {
        assert(fp != nullptr);
        fclose(fp);
    }
};

FILE *fp = fopen("input.txt", 'r');
std::unique_ptr<FILE, FileCloser> uptr(fp);
```

这样的话可以更加异常安全，而且可以完美适配 C API。

如果你使用类似于 OpenSSL 这样的 C API 的话，就可以使用这个用法。unique_ptr 可以作为 low-level (C API), non-RAII, raw resource 和 高级 API 间的粘合剂。

## 使用智能指针时的推荐做法

- 像对待裸指针一样对待智能指针
  - pass by value
  - return by value（当然）
  - 对指针传引用太异味了，自然对智能指针也是
- 如果一个函数接受 `unique_ptr` by value，那么意味着**所有权的转移**
- 智能指针通常作为实现细节以及胶水
  - 在接口中暴露 unique_ptr/shared_ptr 有点 code smell，你应该把他们放在类里。

## std::shared_ptr：共享所有权

### 控制块

std::shared_ptr 代表**共享所有权**，使用 ***引用计数*** 实现。计数归零就会析构对象。引用计数可以使用一个 `std::atomic<int>`

对于一个 std::shared_ptr ，一般有两个成员，一个指向被管理对象的指针，另外一个指向**控制块**（control block）的指针。

控制块包含：引用计数、弱引用计数、自定义 deleter、指向管理对象的指针。

**每个被管理的对象拥有一个控制块。**

拷贝 shared_ptr，会拷贝两个指针，然后引用计数 +1。如果销毁 shared_ptr ，引用计数 -1

shared_ptr 通过控制块参与所有权的管理。

那么为什么控制块要有一个指向控制对象的指针呢？

### 类的布局

考虑以下的结构：

```cpp
struct Fruit {int juice;};
struct Vegetable {int fiber;};
struct Apple : Fruit {int red;};
struct Tomato : Fruit, Vegetable {int sauce;};
```

Apple 继承 Fruit，实际上的布局大概是 |juice|red| 这样。

类似的，Tomato 大概是 |juice|fiber|sauce|

Apple is a Fruit，也就是说我有一个指向 Apple 的指针的同时也代表了指向 Fruit，先是 Fruit 的成员之后才是 Apple 的成员。

Tomato 类似。

就是说，如果我有一个 `std::shared_ptr<Fruit>` 和一个 `std::shared_ptr<Vegetable>`，他们都指向了 `Tomato`。指向 vegetable 的那个指针会有一些偏移。并没有指向直接需要管理的对象。

所以控制块中需要一个指针来决定对谁来执行 delete，在这里就是保存一个 tomato 对象的指针。

### shared_ptr 的 aliasing construct

```cpp
using Vec = std::vector<int>;

std::shared_ptr<int> foo() {
    auto elts = {1, 2, 3, 4, 5};
    std::shared_ptr<Vec> pvec = std::make_shared<Vec>(elts);
    return std::shared_ptr<int>(pvec, &(*pvec)[2]);	// 与 pvec 共享所有权，但指向 &(*pvec)[2]
}
int main() {
    std::shared_ptr<int> ptr = foo();
    for (int i = -2; i < 3; ++i) {
        std::cout << ptr.get()[i] << '\n';
    }
}
```

在以上的代码中，shared_ptr 中指向对象的成员指针指向的是 vec[2]，但控制块中的指针指向的是 vector

最后一个 shared_ptr 销毁时就会销毁 vector

## 优先选择 make_unique()/make_shared()

现代 C++ 的目的之一就是，没有 new/delete 出现，且只调用 new 看起来也很难受。

比如下面这样：

```cpp
std::shared_ptr<Widget> w(new Widget());
use(w)
```

也就是说，如果没调用 delete，那也应该尽量避免 new。标准库所以提供了 make_foo()

```cpp
auto w = std::make_shared<Widget>();
use(w);
```

make_shared 也可以被优化，可以少一次内存分配，现在的库基本都能做到。例如：

```cpp
template<typename T, typename... Args>
std::shared_ptr<T> make_shared(Args&&... args) {
    auto* raw_ptr = new ControlBlockAnd<T>(std::forward<T>(args)...);
    return std::shared_ptr<T>::From(raw_ptr);
}
```

总之：

- 多使用 make_shared/make_unique 避免 new
- 你不 new 就不会内存泄漏
- make_shared 可以优化

顺便，unique_ptr 可以隐式转换为 shared_ptr

```cpp
std::shared_ptr<Widget> sptr = std::make_unique<Widget>();
```



