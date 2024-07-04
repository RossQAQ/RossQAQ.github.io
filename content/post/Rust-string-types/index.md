---
title: 理解 Rust 的各种字符串类型
description: Rust 的各种字符串类型确实让初学者十分迷惑
slug: rust-string-types
date: 2024-07-04 00:00:00+0000
image: 
categories:
    - rust
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文是文章 Understanding String and &str in Rust 的阅读翻译，[查看原文](https://blog.logrocket.com/understanding-rust-string-str)。

根据你的编程背景，Rust 的各种字符串类型可能会迷惑你，比如 `String`, `str`。这篇文章中，我们会澄清 `String` 和 `str` 的区别，更细致的说就是 `String`, `&String`, 以及 `&str`，尤其是决定好什么时候该用什么。

理解了这篇文章中的内容有助于你更高效的使用 Rust 的字符串，也有助于你更好的理解他人的代码，看看别人是怎么处理字符串的。

首先，我们需要从理论层面上来看，例如不同字符串类型的结构，以及他们在内存位置、可变性的不同之处。之后，我们会看一下实践中他们的区别，讨论如何正确使用这些类型。最后，我们会简单举例说明。

如果你是 Rust 刚刚起步，那么这篇文章应该会很好的帮助你。它会解释为什么有的时候你的关于字符串的代码无法编译。

## 理解 Rust 字符串类型们

这一部分中，我们会解释在语言层面这些类型的区别以及他们的 implications。泛泛地说，就是他们在所有权和内存层面的区别。

**Rust 的所有的字符串类型总是保证为有效的 UTF-8.**

### 什么是 `String`

`String` 是所有权类型，需要被分配。它具有动态大小，因而编译器无法知道它的大小，但它内部的数组的容量可以随时改变，这个类型自己基本是以下形式：

```rust
pub struct String {
  vec: Vec<u8>,
}
```

因为它包含了一个 `Vec`，我们知道它有一个指向一个区块的指针，一个 size 以及一个 capcity。size 是字符串的有效长度，capcity 告诉我们字符串的长度最大在多少时需要重新分配内存。指针指向一个连续的字符数组，`capcity` `size` 都在它内部了。

**`String`s 非常的灵活，我们总是可以创建一个新的，动态的可修改的字符串。但这也一些开销，我们总是需要分配内存。**

---

### 什么是 `&String`

`&String` 类型是 `String` 的引用。这意味着它不是一个拥有所有权的类型，它的大小在编译期是可知的，因为它只是一个指向 `String` 的指针。

关于 `&String` 没有太多可以说的内容。但因为它没有所有权，我们可以到处传递 `&String`，只要我们引用的内容没有出作用域，我们并不需要担心内存分配。

大多数情况下，`String` 和 `&String` 的区别基本是在[借用](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)上。

有意思的一件事是，`&String` 可以在编译期被 Rust deref 为 `&str`。这在 API 的灵活性上很有用。但反之是不行的。

```rust
fn coerce_success(data: &str) {
    println!("{}", data);
}

fn coerce_fail(data: &String) {
    println!("{}", data);
}

fn main() {
    let s = "hello_world";
    let mut mutable_string = String::from("Hello");

    coerce_success(&mutable_string);
    coerce_fail(s);
}
```

---

### 什么是 `&str`

最终，我们看看 `&str`，因为 `&str` 由指向内存的指针组成（包括一个 size），它的 size 在编译期是已知的。

内存可以是 heap，stack，或者是二进制可执行文件的 static 内存。**它也不是一个所有权类型，而是一个对字符串切片的只读引用。**Rust 保证当 `&str` 在作用域内时，底层的内存不会改变，即使跨线程。

就像上文说的，`&String` 可以转换为 `&str`，**这意味着 `&str` 作为函数的参数很好用（如果不需要所有权和可变形的话）。**

`&str` 在需要一个字符串切片（视图）时很好用。然而，**记住 `&str` 只是一个指向 `str` 的指针，而 `str` 的大小在编译期不可知，而它天然时不可变的，所以 capcity 也不可变。**

很重要的是，像上文提到过的，**`&str` 指向的内存中 `&str` 存活期间是不可变的，即使是 `str` 的所有者也不行。**

## 实践中 `&str` 的含义

理解了上面的概念那么你就可以让代码过编译了。

此外，还有一些 API 的语义表达，以及一些性能方面的考虑。

在性能方面，你需要考虑知道创建一个 `String` 总是需要内存分配。如果你可以避免额外的分配，那你就应该这么做，因为他们需要一些时间以及对你的运行时造成一些负担。

### 嵌套循环的示例

考虑一个情况，在一个嵌套的循环中，你总是需要某一个字符串的不同部分。如果你每次创建一个 `String`，那么你每次都要为子串分配内存，以及还要做一大堆其他事情。而实际上你仅仅需要 `&str` 的字符串切片。

同样的事情在传递数据时也会发生，如果你到处传递拥有所有权的 `String` 实例而不是可变借用（`&mut String`）或者是他们的只读视图，那么就会发生很多本可以避免的内存分配。

所以为了性能，知道什么时候发生内存分配以及你什么时候不需要分配是很重要的，因为你可以重用字符串的切片。

---

### API 设计

谈到 API 的设计，那么事情就有一些复杂了，你需要在心中明确的了解你的 API 目标，并且为你的用户选择一个正确的类型来达成这个目标。

例如，**因为 `&String` 可以转换为 `&str`，但不能反过来**，所以如果只需要一个只读的字符串视图，那么在参数中使用 `&str` 一般是正确的。

此外，如果一个函数需要修改给定的 `String`，就完全不能传递 `&str` 了，因为它是不可变的，你需要创建一个新的 `String` 之后它会返回。如果你需要修改，那么使用 `&mut String`

---

### 拥有所有权的字符串

在你需要具有所有权类型的字符串时需要思考这些事，例如你需要把字符串传给一个线程，或者需要创建一个成员中具有所有权字符串的结构体时。这些情况下，你需要直接使用 `String`，因为 `&String` 和 `&str` 都是借用的类型。

在某些情况下，所有权和可变性都很重要，在 `String` 和 `&str` 之间的选择很重要。在大部分情况下，你用了不正确的类型都会无法编译，如果你对这些类型的属性以及它们之间的转换没有正确的理解的话，你可能会写出迷惑的 API。

我们看看例子。

#### `String` 和 `&str` 用法的例子

以下的示例展示了上面提到的一些情况，以及包含一些解决的方法。

记住，这些都是独立的、人为设置的例子，在实际中还要考虑其他的因素，然而，你可以把这个当作基本的宗旨。

1. 常量字符串

   **这是最简单的例子，如果你需要一个字符串常量，那么推荐采用以下的方式**

   ```rust
   const CONST_STRING: &'static str = "some constant string";
   ```

   `CONST_STRING` 是只读且满足静态生命周期的字符串，在执行中被直接装入内存。

2. 字符串可变性

   **如果你有一个 `String`，你想在函数中修改它，那么你可以使用 `&mut String` 作为参数**

   ```rust
   fn main() {
     let mut mutable_str = String::from("hello");
     do_some_mutation(&mut mutable_string);
     println!("{}", mutable_string);
   }
   
   fn do_some_mutation(input: &mut String) {
     input.push_str("add this to the end");
   }
   ```

   但要注意的是如果长度超过 capcity，那么也会重新分配内存。

3. 具有所有权的字符串

   当你想从函数中返回字符串，或者你想把所有权也一并传递给其他线程等

   ```rust
   fn main() {
     let s = "Hello World";
     println!("{}", do_something(s));
   }
   
   fn do_something(intput: &str) -> String {
     input.to_ascii_uppercase();
   }
   ```

   ```rust
   struct Owned {
     bla: String,
   }
   
   fn create_owned(other_bla: String) -> Owned {
     Owned {bla: other_bla};
   }
   
   ```

## 只读参数/切片

如果不需要修改字符串，用 `&str` 作为参数类型即可，`String` 也可以用，因为 `&String` 可以被 deref 为 `&str`。

```rust
const CONST_STRING: &'static str = "some constant string";

fn main() {
    let s = "hello_world";
    let mut mutable_string = String::from("hello");

    print_something(&mutable_string);
    print_something(s);
    print_something(CONST_STRING);
}

fn print_something(something: &str) {
    println!("{}", something);
}
```

如你所见，我们可以使用 `&String`, `&'static str`，以及 `&str` 作为输入的参数。

## 在结构体中使用 Rust 的字符串类型

结构体中我们可以使用 `String` 和 `&str`，主要的问题是看你的结构体是否需要拥有某个字符串的所有权。如果你使用 `&str`，那么你需要使用 Rust 的生命周期标注，**保证结构体对象存活时间不长于它借用的字符串，否则就会无法编译。**

```rust
struct Owned {
  bla: String,
}

struct Borrowed<'a> {
  bla: &'a str,
}

fn create_something() -> Borrowed {
  let o = Owned {
    bla: String::from("bla"),
  };
  
  let b = Borrowed {bla: &o.bla};
  b
}
```

这个代码无法通过编译，因为 `Borrowed` 中的值存活的时间比 `O` 更长。

这样写即可：

```rust
struct Owned {
    bla: String,
}

struct Borrowed<'a> {
    bla: &'a str,
}

fn main() {
    let o = Owned {
        bla: String::from("bla"),
    };

    let b = create_something(&o.bla);
}

fn create_something(other_bla: &str) -> Borrowed {
    let b = Borrowed { bla: other_bla };
    b
}
```

## Rust 的字符串操作

Rust 提供了很多 built-in 方法来操作字符串，我们会在这一部分中探索一下。

> Roses：终于到重点了，想必上面的内容对 C++ 程序员来说不成问题。

### Rust 的字符串切片

可以使用字符串切片来引用字符串的子集。使用两个中括号+数字的形式来切片，自然也是左闭右开。

```rust
let hello_world = String::from("hello world");

let ello = &hello_world[1..5];
let orld = &hello_world[7..11];
```

省略左侧数字可以从 `0` 开始，省略右侧数字可以直接索引到最后一个字符。

可以这样引用整个 String，

```rust
let hello_world_ref = &hello_world[..];
```

如前文所说，Rust 把存储的字符串是 UTF-8 编码的顺序字节，也就是说，**上面的例子只对 单字节 的字符有效。如果你的字符由多个字节组成，你就必须中字符的边缘切片。否则 Rust 就会 panic，因为你中多个字节组成的字符的中间切片来。**

例如，❤️这个emoji，就是6个字节编码而成，6～11 之间的索引都用来表示这个 emoji，也就是说，`start_index` 和 `end_index` 在 6～12 之间都会 panic。

```rust
let hello_love = String::from("hello ❤️");

let heart = &hello_love[6..12];
let broken_heart = &hello_love[6..8]; //error
```

---

### 使用 `contains` 来进行模式匹配

`contains` 方法，就像它的名字表示的一样，是检查一个字符串切片的位置。你可以使用它来检查一个字符串切片是否是另一个字符串的子集。返回值是 bool。

传递给它的参数必须是 `&str` ，`char`，或者 `char`s 的切片。

---

### 使用 `starts_with` 来进行模式匹配

在 Rust 中，你可以使用 `start_with` 方法来检查一个字符串切片是否以另一个字符串开头。返回值也是 bool。

注意这个方法大小写敏感。

---

### 使用 `find` 来进行模式匹配

`find` 方法会查找模式在字符串切片中第一次出现的位置，返回值是 `Option`。

注意这个方法也是大小写敏感。

---

### 使用 `rfind` 来进行模式匹配

`rfind` 和 `find` 非常像，它是查找一个字符串最后出现的位置，返回值也是 Option。

## Rust 中的字符串转换

### Rust 中转换为字符串

你可以使用 `to_string` 方法将任何实现了 `ToString` trait 的类型的值转换为字符串。

```rust
let my_bool: bool = true;
let my_int: i32 = 23;
let my_float: f32 = 3.14;
let my_char: char = 'a';


println!("{}", my_bool.to_string());
println!("{}", my_int.to_string());
println!("{}", my_float.to_string());
println!("{}", my_char.to_string());
```

**实现了 `Display` trait 的类型天然实现了 `ToString`，你就不需要再实现一次了。**

**前文提到 Rust 中主要有两种字符串类型。有时你需要把一种转换为另外一种。你可以使用 `String::from` 把字符串切片转换为 `String`**

```rust
let my_string = String::from("Hello World");
```

**相反，可以使用 `as_str` 方法把 `String` 转换为字符串切片，`as_str` 方法借用了底层的数据。**

```rust
let my_string = String::from("Hello World");
let my_string_slice = my_string.as_str();
```

---

### 解析字符串

你可能会写把字符串类型解析为其他类型，他们需要实现 `FromStr` trait。`parse` 方法就可以使用了。

```rust
let my_string = "10";

let parsed_string: u32 = my_string.parse().unwrap();
```

因为你要把字符串解析为其他类型，所以你需要显式指明类型，就像例子一样。Rust 文档中提供了 [实现 `FromStr` trait 的 built-in 类型](https://doc.rust-lang.org/std/str/trait.FromStr.html)，关于 trait 可以看 [Rust traits: A deep dive](https://blog.logrocket.com/rust-traits-a-deep-dive/)

> 之后会尝试翻译。

除了中前面指明类型，你还可以用 Rust 的 "turbofish" 语法，即：

```rust
let my_string = "10";

let parsed_string = my_string.parse::<u32>().unwrap();
```

如果你使用 parse 方法，需要确保字符串提供了有效的字符。你正在解析的字符串可能包含无效字符，**比如 units，尾随的空格，locale 的格式，例如千分隔符。**

**你可以使用 built-in 的 `trim`, `trim_start`, `trim_matches`, `replace` 来移除这些字符，这样你就可以正确解析了**，就像例子：

```rust
let price = " 200,000 ";
println!("{}", price.trim().replace(",", "").parse::<u32>().unwrap());
```

类似的，你解析的字符表示的数字可能会溢出，注意潜在的错误。

`parse` 返回的是一个错误，如果不能解析成制定类型的话。注意正确的进行错误处理。

如果有更加复杂的解析需求，使用 `regex` crate，它有一些 rust build-in 没有的能力。

```rust
use regex::Regex;

fn main() {
  let price = "$ 1,500";
  let regx = Regex::new(r"[^0-9.]").unwrap();

  println!("{}", regx.replace_all(price, ""));
}
```

## 总结

本文探索了 Rust 中 `String` `str` 两个类型的字符串，看了看应该如何正确使用它们。我们也提供了很多示例代码来解释两个字符串类型常规的使用情况。

我希望这篇文章可以解决你的关于 Rust 字符串的困扰，帮助你写出更高效的 Rust 代码。
