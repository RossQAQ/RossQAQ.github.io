---
title: Rust 的生命周期
description: Rust 另外一个重要的概念，生命周期
slug: rust-lifetime
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

好复杂的规则……

## 什么是生命周期

1. Rust 每个**引用**都有自己的生命周期
2. 生命周期是指：引用保持有效的作用域
3. 大多数情况下生命周期是隐式可被推断的
4. **当引用的生命周期可能互相关联时，需要手动指定生命周期**

生命周期存在的目标就是**避免悬垂引用**。

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }

    println!("{r}");
}
```

这段代码明显不能过编译，因为 r 是 x 的引用，使用 r 时，x 的生命周期已经结束了。

rust 使用 **borrow-checker** 来检查所有的作用域，判断借用是否合法。

## 生命周期参数

考虑这个函数：

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";
    
    let res = longest(string1.as_str(), string2);
    println!("The longest string is {}", res);
}

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

函数很简单，但编译会报错，rust 编译器会提示你：期望一个具名的生命周期参数。

因为函数签名体现不出借用的值的生命周期，我们得加一个生命周期参数：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

## 生命周期标注

首先注意，生命周期的标注并不能改变引用的生命周期长度。**指定了泛型生命周期参数之后，函数就可以接收带有任何生命周期的引用。**

生命周期标注是用于：**没描述多个引用的生命周期的关系**。

### 语法

- 生命周期参数名
  - `'` 开头
  - 全小写，短
  - 基本使用 `'a`
- 生命周期标注的位置
  - 在引用的 `&` 后
  - 使用空格将标注和引用分开
- 函数签名中的生命周期标注
  - 和泛型一样，在 `<>` 中声明
  - 返回值中也要声明

### 例子

```rust
&i32	
&'a i32	// 带生命周期的引用
&'a mut i32	// 带生命周期的可变引用
```

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

**这段代码中返回的生命周期是两个参数中较短的那个。**

## 深入理解生命周期

1. 指定生命周期参数的方式依赖于函数的行为

   ```rust
   fn longest<'a>(x: &'a str, y: &str) -> &'a str {
       x
   }
   ```

   例如这个函数只返回 x，那么就不用依赖于 y 的生命周期了，自然也不用标注

2. 从**函数返回引用时，返回类型的生命周期参数要与其中一个参数的生命周期匹配**

   如果返回的引用没有指向任何参数，那么暗示它引用的是函数内部创建的值，那明显是悬垂引用。

## 结构体定义中的生命周期标注

- 结构体可以包括
  - 自持有类型（i32）
  - 引用：**需要在每个引用上添加生命周期标注**

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}
```

代表**结构体内的引用必须比结构体的实例本身活得长。**

## 输入、输出生命周期

出现在函数的参数：输入生命周期

出现在函数返回值：输出生命周期

## 生命周期省略的三个规则

规则 1 应用于输入生命周期，规则 2、3 应用于输出生命周期。

如果规则应用完仍然无法确定生命周期，那么会报错。此外，这些规则适用于 fn 定义以及 impl 块。

1. 每个引用类型的参数都有自己的生命周期
2. 如果只有 1 个输入生命周期参数，那么就会把它赋给所有的输出生命周期参数
3. 如果有多个输入生命周期参数，但其中一个是 `&self` / `&mut self` ，那么 self 的生命周期会赋给所有的输出生命周期参数。

### 例子

假设我们自己是编译器：

```rust
 fn first_word(s: &str) -> &str
```

先应用第一条规则：

```rust
fn first_word<'a>(s: &'a str) -> &str
```

函数只有一个输入生命周期参数，那么应用第二条规则：

```rust
fn first_word<'a>(s: &'a str) -> &'a str
```

这种情况下显然编译器就可以处理。



```rust
fn longest(x: &str, y: &str) -> &str 
```

应用规则 1：

```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str 
```

显然有多个输入生命周期参数，那么规则2不能用，也没有 self，所以规则3也不行。

此时编译器并不能计算出所有引用的生命周期参数，故报错。

## 方法定义中的生命周期标注

在结构体上使用生命周期实现方法，语法和泛型一样。在哪声明和使用生命周期参数，取决于其是否和字段、方法的参数或者返回值有关。

`struct` 字段的生命周期名：在 `impl` 后声明，在 `struct` 后使用，**生命周期也是结构体类型的一部分**

`impl` 块中的方法签名中：引用必须绑定与 struct 字段引用的生命周期，或者引用是独立的。此外经常可以省略。

## 静态生命周期

`'static` 是特殊的生命周期：持续整个程序

默认所有字符串字面量都是静态生命周期。

```rust
let s: &'static str = "I have a static lifetime."; 
```

