---
title: C++ vs. Rust - 所有权
description: 刚学 Rust，对于 Rust 最重要的所有权，与 C++ 进行一些对比学习与总结。
slug: rust-ownership
date: 2024-03-21 00:00:00+0000
image: 
categories:
    - rust
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

最近因为面试要学习 Rust，突击了几天，感觉 Rust 是真的麻烦。

但确实 Rust 很多地方是 copy 了 C++ 的，然后做出了一些改进。

本文章是建立在我比较懂 C++ 但是不懂 Rust 的情况下写出来的，不保正内容完全正确。仅作为个人记录，为更深入的理解 Rust 做铺垫。

## 堆内存、栈内存

首先要说的是我不知道 C++ 的标准里有没有规定堆内存和栈内存，但大概的实现应该和大部分网上的资料相同。至少目前这么认为应该没问题？

以下为了方便叙述，就假设目前已知的资料是正确的。

Rust 跟 C++ 一样，在堆上使用内存都需要分配（栈内存那当然不用分配……因为已经预先分配好了，不然就不用 stack overflow 了），然后使用指针访问。

那么所有权存在的原因是什么呢？在 Rust 中，**管理 heap 数据，就是所有权存在的原因**。

Rust 怎么回收内存呢？和 C++ 一样，反正不是通过 GC 回收，而是通过所有权。

Rust 在编译时会进行一系列的检查，如果违反了所有权的规则，那么就不能过编译（存在潜在的内存泄漏可能）。

## 所有权

Rust 中关于所有权有三条规则：

1. 每一个值都有一个**所有者**。
2. 每个值在**任何时刻** **有且只有一个所有者**。
3. 所有者**离开作用域**，**值会被 Drop**

> 我思考了一下，Rust 应该是把对象的生命周期和存储周期绑定在一起了？当然在 C++ 中如果你遵循 RAII 的话那大部分时刻也是这样的。

`String` 在 Rust 中是存储在堆上的，以下都用 `String` 来作为示例（当然其他用堆存储数据的结构也同理）。

普通的字符串使用：

```rust
fn main() {
    let mut str = String::from("Hello");

    str.push_str(" World");

    println!("{str}");
}
```

根据上面的规则，你别管堆内存栈内存，离开作用域就会被释放，Rust 在 `}` 自动调用 `drop` 函数（C++ 中的 RAII）

### Rust vs. C++ -> 移动

Rust 中对于存在于**堆上的对象**，**默认语义是 *移动***。

**以下代码无法编译，因为 `str` 的所有权已经移动给了 `str2` 。**

```rust
fn main() {
    let str = String::from("Hello World");

    let str2 = str;

    println!("{str}"); 
    println!("{str2}");
}
```

而在 C++ 中，首先，**C++ 默认语义全是拷贝**，如果要移动需要调用 `std::move`，此外，C++ 标准保证标准库组件被移动后状态为 valid （但未指定）。

**以下代码可以编译，访问 `str.length()` 是合法的**

```cpp
int main() {
    std::string str{ "Hello World" };

    auto str2 = std::move(str);

    std::cout << str.length() << '\n';
    std::cout << str2 << '\n';
}
```

对于存在栈上的对象大家都一样。

> Rust 这么干明显可以避免 double free 这种问题。

当然了，不管是 Rust 还是 C++，移动都不代表浅拷贝。

### Rust vs. C++ -> 克隆

如果真的要**复制**堆上的数据，在 Rust 中，需要调用 `.clone()` 方法，这个就创建了一个副本。

而在 C++ 中当然直接 `=` 赋值默认就会选择复制构造。

但需要注意，**Rust 不允许实现 Drop trait 的同时实现 Copy trait**。

> 真是严格呢~

### Rust vs. C++ -> 所有权 + 函数

根据之前的介绍，`Rust` 默认写值就是移动语义，那么在函数参数中自然也如此。

以下代码并不能通过编译，`str` 的所有权移动给了函数 `takes_ownership` 的参数上，在出函数作用域时已经被释放了。

```rust
fn main() {
    let str = String::from("Hello World");

    takes_ownership(str);

    println!("{str}");
}

fn takes_ownership(str: String) {
    println!("{str}");
}
```

在 C++ 中想实现类似的效果需要：

```cpp
void takes_ownership(std::string&& str) {
    std::cout << str << '\n';
}

int main() {
    std::string str{ "Hello World" };

    takes_ownership(std::move(str));

    std::cout << str.length() << '\n';
}
```

注意，函数参数写的是 `&&`，这样才是直接调用移动构造，然后在函数结尾销毁参数，然后 `str` 是 valid 但是**未指明**状态。

而如果写普通的值类型，那么 `main` 中的 `str` 还是要析构的。

> 这里我也没太明白，我感觉应该是跟解分配和释放有关。总之，&& 才是和 Rust 的语义相同。

---

Rust 中还能返回所有权，这样参数就不会被销毁了，即：

```rust
fn main() {
    let str = String::from("Hello World");

    let str2 = takes_ownership(str);

    println!("{str2}");
}

fn takes_ownership(str: String) -> String {
    println!("{str}");
    str
}
```

> 这里可以利用一下 rust 的 shadowing，直接让返回的变量也叫 str。当然这里 `str` 是不可变引用，所以得 shadowing，不能直接用 str 接返回值。

在 C++ 里一般应该不会这么干，如果想干的话：

```cpp
std::string takes_ownership(std::string&& str) {
    std::cout << str << '\n';
    return str;
}

int main() {
    std::string str{ "Hello World" };

    str = takes_ownership(std::move(str));

    std::cout << str << '\n';

    return 0;
}
```

返回的时候值类型就可以，会帮你选择移动的。

## 引用

Rust 中，如果不需要所有权，那么可以给函数传入一个**引用**。

也就是说：**引用不持有所有权，创建一个引用称为 *借用***。

### Rust vs. C++ -> 引用

Rust 给函数传参也需要**加上 `&` 符号**，代表传递引用。

（弹幕说其实是类似于指针，Rust 在函数参数这个地方自动帮你解引用了）

```rust
fn main() {
    let str = String::from("Hello World");

    let str_len = calculate_length(&str);

    println!("{str_len}");
}

fn calculate_length(str: &String) -> usize {
    str.len()
}
```

那这个在 C++ 里就不用说了吧：

```cpp
size_t calculate_length(const std::string& str) {
    return str.length();
}

int main() {
    std::string str{ "Hello World" };

    auto str_len = calculate_length(str);

    std::cout << str_len << '\n';

    return 0;
}
```

---

如果想要在函数里修改字符串，那么在 Rust 中你需要：

```rust
fn main() {
    let mut str = String::from("Hello");

    let str_len = change(&mut str);

    println!("str: {str}, len: {str_len}");
}

fn change(str: &mut String) -> usize {
    str.push_str(" World!");
    str.len()
}
```

如果在 C++ 中，反过来（这俩一个默认可变一个默认不可变真的是）：

```cpp
size_t change(std::string& str) {
    str += " World!";
    return str.length();
}

int main() {
    std::string str{ "Hello" };

    auto str_len = change(str);

    std::cout << str << '\n' << str_len << '\n';

    return 0;
}
```

### Rust vs. C++ -> 可变引用

在 Rust 中，同时创建并使用两个可变引用是错误的。

```rust
fn main() {
    let mut str = String::from("Hello");

    let str2 = &mut str;	// str2 虽然没写 mut，但也是可变引用，s3 同理。

    let str3 = &mut str;

    println!("{str2}, {str3}");
}
```

C++ 不用说了肯定可以。

---

引用借用出去后，原来的引用是不能修改的（可能是防止 Rust 所说的“数据竞争”），例如：

这个代码可以通过编译：

```rust
fn main() {
    let mut str = String::from("Hello");

    let r = &mut str;

    r.push_str("123");

    str.push_str("456");

    println!("{str}");
}
```

而这段代码不能通过编译：

```rust
fn main() {
    let mut str = String::from("Hello");

    let r = &mut str;

    str.push_str("456");

    r.push_str("123");

    println!("{str}");
}
```

---

Rust 中，在不同作用域可以拥有多个可变引用，但不能同时拥有（类似于 lock_guard ? 正常，不然就冲突了，而在同一个大括号作用域内是不会冲突的）

此外，在同时使用可变引用和不可变引用时也有这种规则：

以下代码无法通过编译，其实**类似于读写锁**，就像写锁只能有一个，读锁可以很多，而读写又互斥。

```rust
fn main() {
    let mut str = String::from("Hello");

    let r1 = &str;
    let r2 = &str;

    let r3 = &mut str;

    println!("{r1} {r2} {r3}");
}
```

注意，这里如果你使用了 `r1` `r2` 那么就不能编译，如果你没有使用，那么可以编译。

**引用的作用域**是：**引用声明的地方开始，到最后一次使用为止。**

编译器判断这个的能力称之为：**非词法作用域生命周期，NLL**。

### Rust vs. C++ -> 悬垂引用

在 Rust 中，会阻止你使用悬垂引用，编译器确保引用永远有效。

```rust
fn main() {
    let mut str = dangling();
}

fn dangling() -> &String {
    let str = String::from("dangling ref");
    &str
}
```

这里会报错，需要生命周期标记。

如果是 C++ 中，那必然是可以返回的，你得自己避免。

## Slice

Slice 也是 Rust 中的一种引用，**他引用一段连续的集合**，既然 Slice 也是引用，那么自然也没有所有权。

### Rust vs. C++ -> 字符串 Slice

使用一个左闭右开的 range，当然，也可以写 `&s[0..=4]` 来给一个闭区间。

此外，0可以省略，最后一个 `idx` 也可以省略，比如 `&s[..]`。

**字符串的 slice 类型叫做 `&str`，字符串常量的类型也是 `&str`**

```rust
fn main() {
    let s = String::from("Hello");

    let hello = &s[0..5];
}
```

在 C++ 中 字符串 Slice 明显是 C++17 的 std::string_view，然后字符串字面量的类型明显都是 `const char[]`

```cpp
int main() {
    std::string str{ "Hello" };

    auto view = std::string_view{ str.begin(), str.begin() + 5 };
}
```

其他的结构可以用 C++20 的 `std::span` 以及 C++23 的 `std::mdspan` 来进行切片。

而不管是 Rust 还是 C++，都可以对 Slice 字符串直接传递 `String`/`std::string`。
