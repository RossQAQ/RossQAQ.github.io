---
title: Rust for Rustaceans - Chapter-1 基石
description: 阅读 Rust for Rustaceans 的记录，第一章。
slug: rfr-intro
date: 2024-05-15 00:00:00+0000
image: 
categories:
    - rust
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本系列博客并不是书《Rust for Rustaceans》的全文翻译，不保证内容准确，仅作为个人阅读后的学习记录。

支持正版：[Rust for Rustaceans by Jon Gjengset (rust-for-rustaceans.com)](https://rust-for-rustaceans.com/)

作者油管：[Jon Gjengset - YouTube](https://www.youtube.com/@jonhoo)

# 第一章 — 基石

Rust 中的各种概念非常重要，所以有必要保证你的基础坚挺。

这一章会讨论变量、值、所有权、借用、生命周期等等，你需要完全了解后才能继续深入本书。

## 内存

内存的每个部分并不相同，在多数语言环境下，你的程序都会访问：栈、堆、寄存器、text segments、memory-mapped 寄存器，memory-mapped 文件，还有 non volatile RAM。

### 内存术语

在深入了解各个内存区域之前，首先需要知道值、变量、指针的区别。在 Rust 中，**值**是类型和这个类型值域中的元素的结合。值可以被转换为使用某种类型代表的字节序列。

例如，6u8，就代表整数 6，以及内存中的字节是 0x06. 值跟变量存储的地址是无关的。

值需要存储在一个地方，不管是栈还是堆，最常见的存储值的地方叫做 **变量**，栈上的一个 slot。

指针存储了某个内存区域的地址值，所以指针指向一个地方，指针通过解引用可以访问其存储的地址值中的值。可以拥有多个指向同一块内存的指针变量。

```rust
let x = 42;
let y = 43;
let var1 = &x;
let mut var2 = &x;
var2 = &y;
```

值：42、43、x的地址、y的地址

变量：x，y，var1，var2

与普通的变量不同的是字符串变量

```rust
let string = "Hello World";
```

`string` 存储的还是一个指针，虽然看似我们把字符串字面量赋值给它了。但关于 `string` 实际指向了谁，以后再说。

---

### 深入理解变量

之前给出的变量的定义可能确实没什么用，更重要的是你需要有一个更准确的 mental model 来帮助你理解程序的行为。有很多种模型，但总得来说，分两种 —— High-level models 和 Low-level models。High-level models 在你讨论代码本身时更有用，比如讨论生命周期、借用等。Low-level 模型在你使用 unsafe 时有用，比如使用裸指针。

对阅读本书来说，知道这两种模型足够了。

> 我大概懂了作者的意思了，放在 C++ 里就是说别老一天天想着你那引用的本质就是指针，不同的时候要从不同的抽象层级看待你的代码。

#### High-Level Model

在 High-Level 角度，我们会把变量看成是值的名字，他们被初始化、被移动、被使用。在这种模型中，变量只有在拥有合法值的时候存在，你不能使用未初始化的变量，也不能使用值被移动的变量。

想象一下，你的变量在被访问时，会从先前一次访问的地方向这里画一条线，这条线称之为 *flow*，它描述了两次访问之间的依赖关系。你的程序会有许多这样的线，他们跟踪变量的生命周期，编译器跟踪这些 flows，每个 flow 只能在与其他 flows 兼容时才能存在。例如，不能同时有两个平行的 flows 指向同一个可变变量，也不能跟没有值的变量借用。

```rust
let mut x;

// 非法。x 没有值
assert_eq!(x, 42);

x = 42;		//(1)

// ok. flow 从第一次赋值时开始，
let y = &x;	//(2)	

// 第二个 mutable flow from x
x = 43;		//(3)

// 非法。 这个 flow 从 y 开始，但跟 x 的 flow 冲突。
assert_eq!(*y, 42);	//(4)
```

程序有两个 flows，一条是从(1) 到 (3) 的独占 flow（&mut），另外一条是从 (1) 穿过 (2) 到 (4) 的共享 flow。borrow checker 会对这些 flows 进行检查。你不能让独占 flow 和共享 flow 并存（3），所以编译会报错。如果没有 (4)，那么可以通过编译。

注意，如果声明了一个新的变量和之前的变量命名相同，它们仍然会被认为是不同的变量，这称为 *shadowing*。后面的变量以相同的名字 shadows 前面的变量。

#### Low-Lovel Model

变量名字的内存位置不一定拥有合法的值。你可以把变量想象成是：value slot. 你赋值，slot 被填充，旧的值会被 drop. 当你访问时，编译器会对这个 slot 进行检查。

这个层级的模型与 C/C++ 以及其他一些低级语言的模型是类似的，当你需要直面内存时你要这么考虑。

> 当然，这里我们忽略的 CPU 寄存器的存在。

这两种模型是并存的，你需要在你的脑子中建立起这种概念，而不是说谁比谁更好。如果你能在两个层面同时理解一段代码，那么你就会发现一段复杂的代码是如何工作、如何通过编译、如何按你的预期执行的。

---

### 内存区域

我们已经知道如何与内存交互，我们需要讨论内存究竟是什么？
