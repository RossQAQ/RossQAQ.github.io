---
title: Writing custom C++20 coroutine systems
description: 跟随 Simon Tatham 一起学习 C++20 的协程。
slug: Coroutine System
date: 2024-02-08 00:00:00+0000
image: 
categories:
    - cppnotes
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

# 编写你自己的 C++20 协程系统

本文翻译自 [Writing custom C++20 coroutine systems](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/)

## 介绍

在 2023 年，我给使用过 C++ 旧版本的人做了一个三天的 C++20 新特性介绍讲座。

其中的一个新特性那就是 built-in language 支持的协程。我个人非常期待讲那个部分，因为我已经把协程当做我的代码风格一部分了。我甚至花费了大量精力通过使用 C 的预处理来实现写成，PuTTY 在它的 SSH 协议实现中也大量使用了该技术。另外一个我的程序，spigot（交互式计算器）也在 C++ 中用了类似的技术。

所以我非常期待使用 C++ 的原生协程，*至少* 我需要知道是否可以将使用预处理器版本的 spigot 使用原生协程重构，以及其他一切我可能会用上的内容。

不幸的是，C++20 的协程系统非常庞大、复杂，需要很多笔墨来解释。所以那三天的讲座中并没有完全讲完 C++20 的新特性，事实证明，甚至没有时间解释太多细节。（更别提你还要从解释协程是什么开始，因为有很多人并不知道协程）

于是我写了这篇文章。

努力学习并把它写完后，似乎自己藏着不太好。所以这篇文章是我的学习笔记，经过精心打磨后希望对其他人也有用。

文章中我会展示所有的代码来阐述我自己的目的。大多数都不能工作：他们会缺失一部分，以免完全不相干的细节吸引了读者的注意力。有时为了清晰，我甚至会故意犯错（例如把函数定义放在类中，你应该避免这么做来防止前向声明）。大多数部分都有可下载的可以编译运行版本的代码。

## C++ 协程概览

C++ 协程有许多其他语言（例如 python generator）不具备的灵活性，但灵活也代表着你需要自定许多内容才能让其正常工作。他不像是协程系统，而更像是给你一个工具，让你自己搭建自己的协程系统。

函数和协程之间原点比较像。你不需要在函数体外将其声明为协程。（即，调用者不需要知道他们调用的是否是协程，他们只需要知道当调用函数，会返回一个对象）

在 Python 中，在函数中写一个 yield 即可，作用就是当你调用函数时，其内部的代码暂时不会执行，你只会得到一个 'generator' 对象，函数的代码是暂停的，直到你在 generator 对象上进行一些操作时才会执行一点。只有一种 generator，提供的操作也是固定的（next()和 send()），且他们做的内容也差不多。

C++ 中同理，不同的是有三个关键字：`co_yield`, `co_await` and `co_return`。函数中使用任何一个关键字都会让编译器认为其是一个协程。但你可以控制其下一步执行的流程。

举个例子，假如你的协程大部分情况下都是 *接收数据*，当其接收第一个 item 后你就希望立刻运行他。这样的话，当某人调用协程并创建它的实例时，这个实例已经处于准备好接收一个值的状态。Python 并不能这么做，因为 generator 总是在一开始暂停，由此正常的用法是在返回的对象上调用一次 `next()` 来获得第一项数据。但在 C++ 中，你可以按你的想法来定义协程的行为。

在 Python 中，在协程内调用 `yield` 会暂停这部分代码的执行并且将控制权返回给调用 `next()` 的调用者。所以如果你想生成一个值并且把它传递给另外一个协程（就是说，从一个生产者协程将对象传递给一个消费者协程），你需要手动在调用者方实现。在 C++ 中，你可以以你喜欢的方式实现，但是你*也可以* 让协程自动在它们之间转移控制权。因此，在调用方的所需要的值实际可用之前，控制权不会返回给调用方。或者你也可以 co_yield 生成一个值但是完全不暂停协程，继续运行。

在 Python 中，同样 yield 语句会生成一个值 *且* 接收一个输入值（如果协程需要）。在 C++ 中，`co_yield` 和 `co_await` 是分开的，你可以将其用于不同的用途：`co_yield` 向外传值，`co_await` 等待输入，或者（如果你喜欢的话）等待某些事件发生。所以如果你的协程需要从 “这里” 读取一个对象并把它写入 “那里”，那么这些语句可以帮在语义上你区分功能。 

在 Python 中，协程实际不存在与生成值流可区分的 “返回值” 的概念。在 C++ 中，是：你可以使用 co_return 来结束整个协程，你可以给他一个值返回，可以跟你 yield 的值不同类型不同语义。

更复杂一点的例子，假设使用协程来实现一个网络协议，通过从 event loop 中重复调用回调来执行底层的网络 I/O 以发送或接收单独的协议消息或数据包。你可以 set it up 这样就可以通过 co_await 来获取下一个输入的数据包（可能不需要暂停协程，如果队列中已经存在收到的数据的话）可以调用 co_yield 来向外传递数据包；当其完成时，可以调用 co_return 来给整个流程发信号，例如整个事务是否成功。

然而，**这些都得你自己完成**。为了使用 C++20 的协程，你必须编写大量的前置代码（造轮子），然后回答以下问题：

- 协程在启动时需要暂停吗？
- co_yield, co_await, co_return 需要的数据类型是什么？每一个关键字是干什么的？传递给他们的值会发生什么？如果有的话，协程恢复时从 co_await, co_yield *传回* 的数据是什么？
- co_await, co_yield 的具体哪一步会暂停协程？哪一个会立刻运行？
- 协程暂停时，会立刻将控制权返回给上次恢复他的 caller 吗？还是切换到另外暂停一个协程？

Python 中，这些问题的答案是固定的。在 C++ 中，答案是：**这些全部取决于你**。你可以根据你的程序来决定具体采用哪种方式。但换句话说，你需要自己做完所有的工作才能找出答案。

至少，在C++20标准中你只能自己完成这些工作。C++23 引入了一些协程系统，如 `std::generator`，类似 Python 的 generator。如果你需要的是类似于 Python 的 generator，那么你可以使用 C++23。

但如果你想要额外的灵活性，或者无法使用 C++23，那么你需要自己完成全部工作，这也是这篇文章介绍的内容。

## 数据类型的总结

C++ 的协程系统包含了许多不同的数据类型。在深入理解之前，这一部分我们先区分并且理解一下他们在内部是怎么交互的。

C++ 实现自身提供了一个数据结构：**coroutine handle**。所有魔法都来自于他。它定义了你协程代码的一个特殊实例，包含了它内部的变量以及执行状态（例如它现在是否正在运行，如果不是的话，它下次会从哪里恢复）。**它提供了一个 resume 方法，通过调用他就可以恢复协程执行**。

其他的数据类型都由你也就是实现者提供，因此你可以根据你自己想让协程执行的方式来定义他们。

最明显的类型就是协程定义为返回值的类型。*用户* 唯一能看到的就是这个类型（不管他们是根据你的前置代码写协程，还是调用协程）。所以，在这篇文章我会把它称为：**用户感知类型（user-facing type）**，它的实例则为：**用户感知对象（user-facing object）**

用户感知类型是协程的 caller 实际交互的类型。所以你对其定义的方法取决于你希望 *如何* 与用户交互。例如，你可能想让他可以迭代（提供 begin() and end()，并且迭代器可以和自增运算符交互），这样你就可以使用 range-based for loop 了。（C++23 已经提供了 std::generator）或者你也可以选择像使用 istream/ostream 一样使用他，当然也可以是其他任何的方法，这都取决于你。

实际上，C++对于用户感知类型 *没有* 什么特殊需求。不需要有任何特定名称的方法或者字段。甚至不需要是一个类类型，如果你不想的话。你可以让他是一个平凡类型，比如 int，让他扮演一个 index 来访问当前激活协程的一个巨大的表格。当然，让他成为类类型更常见，但不是*必须*。

在这个实现中，你的特殊协程首要的 data type 叫做 **promise type**。

> 我个人更喜欢称之为 'policy type', 因为它主要的作用是定义你的协程策略。C++ 称他为 'promise' 是因为他也可以代表异步计算模型中的 'future/promise'。但其语义上，policy type always, promise type sometimes，所以我经常叫他 'policy type'，更接近于设计初衷。但是 'promise' 是标准的叫法，而且作为标识，所以以下我们也会称之为 promise。

This is inferred from the user-facing type (and, optionally, the rest of the coroutine’s arguments). 它必须是一个类类型，必须提供一些具有特殊名称的方法来控制执行策略，例如开始时是否暂停，结束时是否将控制权转移给其他协程，如果遇到 co_await, co_yield, co_return 后该如何执行。每个协程的实例只有一个（编译器在第一次调用协程时创建他），自然也方便存储协程有关的数据。例如，在 stacked 协程启动时，你可能需要一个指针在栈中来存储 promise object。在协程中通过 co_yield 向外传递值，放置 yielded value 最方便的地方就是在 promise object 中，之后用户感知对象就可以将其传递 caller。

最后，有一种叫做 **awaiters** 的类型（有时也称之为 awaitable）。他为每个事件来设置策略以暂停协程。特别地，每次协程执行 co_yield, co_await, co_return，promise type 都会构造一个新的 awaiter 用于特殊事件，调用 awaiter 的方法来决定发生什么（例如是否暂停协程本身；如果暂停的话，是否将控制权返回给 caller 或是其他协程；协程什么时候恢复；通过 co_yield 或者 co_await 返回什么）。Awaiter 类型可以指定你的特殊行为，或者在多个类型的协程中共享。总之，标准库提供了一些定义好的 Awaiter 以完成最简单的功能，他们是：`std::suspend_always` 以及 `std::suspend_never`

## 如何编写协程的 promise class

### 快速开始：让他至少能编译

在深入理解所有的方法是做什么的之前，我们先从最简单的协程开始，让他至少能够编译，并且执行一些代码。之后我会列出实现所必须的内容，你可以将他们与例子进行关联。

事不宜迟，以下是协程的 Hello World：

```cpp
#include <coroutine>
#include <iostream>

class UserFacing {
public:
    struct promise_type {
        UserFacing get_return_object() { return {}; }
        std::suspend_never initial_suspend() { return {}; }
        void return_void() {}
        void unhandled_exception() {}
        std::suspend_always final_suspend() noexcept { return {}; }
    };
};

UserFacing demo_coro() {
    std::cout << "Hello World\n";
    co_return;
}

int main() {
    UserFacing demo = demo_coro();
}
```

以上代码片段是协程的最小实现，满足了其最小要求。

包括：



**用户感知类型中包含 promise_type**。编译器看到你定义了协程的返回类型 `UserFacing` 时，**首先要做的就是查找其关联的 promise_type**

*默认* （但不是唯一）实现其的方式是实现 UserFacing::promise_type ，在这个简单的例子里我会直接把 promise type 写在 UserFacing 类中。第二种方法是分开定义，在 UserFacing 中包含其声明即可

```cpp
class UserFacing {
public:
    using promise_type = TheActualPromiseType;
};
```

此外还有第三种方法，无需在用户感知类型中实现 *任何* 定义。这需要你提供你自己的 `std::coroutine_traits` 模板特化（重载默认寻找 `UserFacing::promise_type` 的版本）。我们之后会举例说明，例子中协程的返回值是 `std::unique_ptr`



**构造一个用户感知类型返回的实例**。你的协程初始化时，C++实现会创建 promise_type 的实例，并为其分配内存。不过 caller 接受到的返回对象取决于你。

这一步通过 promise_type 中的 `get_return_object()` 实现。在里子中，`UserFacing` 没有任何数据成员，所以 `get_return_object()` 通过完全平凡的方式来进行构造。

如果不是平凡类型，那么你可能需要给用户感知对象更多的信息。例如，**用户感知对象几乎都需要访问 coroutine handle 以及 promise type，这样才能跟其他暂停的协程通信**。由于现在是最小实现，所以目前我们还没有实现他，下一部分会看到的。



**指定协程启动时是否暂停，或者直接运行**。通过 `initial_suspend()` 实现，必须返回一个 awaiter。我们之后会看到 awaiter 的全部细节。目前，我们只是使用了 C++ 标准库提供的：`std::suspend_never`。这样协程就 *不会* 在开始执行之前暂停自己，意味着在控制流返回给 main() 前，协程会一直运行，并且输出 Hello World。

如果我们让 initial_suspend() 返回另一个标准类型：`std::suspend_always`，那么新创建的协程会在函数体开始执行前就暂停，*before* 打印 hello world。所以直到 main 中的某一部分调用 resume 前都不会执行协程。但我们目前还没展示如何做。目前来看，我们的协程开始执行前不暂停，因为只有这样他才能打印正确的信息。

(std::suspend_always 和 std::suspend_never 都不包含任何有趣的数据，他们只是返回固定值的方法。所以你不需要在你构造时提供任何信息，只是单纯调用 `return {}` 即可)



**指定在协程正常返回时的行为**。在这个例子中，协程不返回任何值（co_return 语句没有参数）。所以我们在 promise_type 中实现一个 `return_void()` 函数，这是 `co_return` 或者协程结束时实际执行的函数（因为没有返回值）。

如果我想让协程拥有一个 non-void 返回值，那么就应该实现 `return_value()` ，接受一个参数，之后会在执行 co_return 时被调用。

注意，你***必须*** 在二者中选择一个实现！都不实现，或者都实现都是错误。

（如果你的协程想要返回值，而函数末尾不执行 co_return，那么这是ub，就像返回具体值的函数没有 non-void return 语句一样）



**指定异常从协程传出时的行为**。如果协程内的代码抛出异常，并且没有东西 catch 住它，那么 promise 对象的 `unhandled_exception()` 方法会被调用。它会接受异常并存储它，之后做一些有用的事情。

在之后的部分我们会看到其执行细节，并且展示一些复杂的案例。在这个最小示例中，unhandled_exception 什么都没干，这意味着异常会被无视，协程会保持相同状态，仿佛其正常终止一样。



**指定协程结束时的行为**。通过 final_suspend 方法指定，和 initial_suspend 一样，除了他被声明为 noexcept（如果这时候有异常，就像在析构函数有异常一样）

final_suspend 会在协程被以 *任何* 方式终止时调用，不管是 return 还是抛出异常。

示例中，final_suspend 返回 std::suspend_always，意味着协程结束时（不管是 co_return，还是单纯执行完函数体），它都会暂停当前状态并返回控制权。

你 *不可以* 通过 final_suspend 让协程继续运行来返回值，这会导致崩溃或者其他 ub。这里唯一的用途就是不直接将控制权返回给 caller，而是转给其他协程。后面会有示例。

Full source code for this section: [`co_demo.minimal.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.minimal.cpp).

----

### 使用 co_await 暂停协程

目前，我们的协程功能很少，因为你不能暂停和恢复它，而暂停和恢复才是协程的关键。

C++ 提供了两个关键字来暂停协程：co_yield, co_await，co_await 更基础更常用。co_yield 是它的一个语法糖。所以我们先来介绍 co_await。

提供两个关键字的点在于：await 用于提醒你想要 *等待* 一个东西，而 yield 则是表明你想要给某些人传递一个值。你并不强制需要为实现不同的目的使用两个关键字，你写代码是为了做事情，所以你可以按你的想法来。但通常的语义是这样。

总的来说，不管你给 co_await 什么操作数，编译器都需要把它转换成 awaiter 这样才能管理暂停。

*最简单* 的方法就是提供一个 *已经* 是 awaiter 的对象。我们前文已经提到了，std::suspend_always 和 std::suspend_never。所以我们可以简单改改 demo：

```cpp
UserFacing demo_coro() {
    std::cout << "we're about to suspend this coroutine" << std::endl;
    co_await std::suspend_always{};
    std::cout << "this won't be printed until after we resume" << std::endl;
    co_return;
}
```

这是最简单的暂停协程的方式，但更常见的，如果你的协程正等待什么事情，你需要在事件发生后恢复协程，所以你需要一些 handler。

有两个地方可以插入 handler：

- 如果你的 promise type 有 `await_transform()` 方法，接受的参数类型为你传递给 co_await 的参数的类型，那么会调用这个方法，co_await 的参数也会变成该函数的返回值。（你当然可以重载这个函数）
- 如果在作用域中有一个函数叫做 `operator co_await()` 接收相应的类型，那么会调用这个函数，同样参数也会被替换为返回值。

如果两个都存在，那么会按顺序调用，所以 `co_await foo;` 也许会通过 `operator co_await(promise.await_transform(foo))` 来构造 awaiter

> 不太明白使用 operator co_await() 的意义，因为他不能是类成员，只能是全局函数，所以不能访问 promise type 的对象。看起来灵活性不高。在我的例子中，我只会使用 await_transform()。

所以你可以把想要等待得到的结果传递给 co_await，然后其在背后通过你的 promise type 的 await_transform() 来实际返回合适的 awaiter。

实际的使用中，你大概会让 await_transform() 来做一些工作。**例如，你的程序基于某些监听 I/O channel 的时间循环，比如网络连接，然后它可能持有一些数据结构来告诉它对于每个 IO 事件，当某些事件发生时该恢复哪个协程。**（For example, if your program was based around some kind of event loop that was monitoring I/O channels like network connections, then it would have some data structure that told it what to do about each possible I/O event, perhaps including what coroutine(s) to resume when an event happened. ）

所以如果一个协程想要在做其他工作前等待 IO，你可能会写 `co_await event` （可能你会实现一些方便在协程内部操作代表 IO 事件的类型），然后相应的 await_transform() 会将调用协程的代码插入 event loop 的数据结构中来管理好实际发生的事情。

但我们在这个例子中不需要复杂的事件，我们尽量简单。我们假设存在一个 dummy event 类型，不包含数据，然后 await_transform() 接受他：

```cpp
struct Event {
    // you could put a description of a specific event in here
};

class UserFacing {
    // ...
    class promise_type {
      public:
        // ...
        std::suspend_always await_transform(Event) {
            // you could write code here that adjusted the main
            // program's data structures to ensure the coroutine would
            // be resumed at the right time
            return {};
        }
    };
};
```

然后你就可以 co_await 你的事件描述符了：

```cpp
UserFacing demo_coroutine() {
    std::cout << "we're about to suspend this coroutine" << std::endl;
    co_await Event{};
    std::cout << "this won't be printed until after we resume" << std::endl;
    co_return;
}
```

这个例子中，我的 await transform 返回的是 std::suspend_always。你也可以返回自定义 awaiter 类型，这样更加灵活。特别的，在某些时候，可能你等待的事件已经完成了，这时你自定义的 awaiter 类型最好不要暂停，而是继续执行。

自定义 awaiter 也可以控制 co_await 表达式的*返回值*，例如，假如你要等待网络事件并输出一个值，或者传出一个成功或者失败的状态。那么你的代码可能就会像：

```cpp
UserFacing demo_coro() {
    // You might set up co_await to return actual data
    ip_address addr = co_await async_dns_lookup("hostname.example.com");
    
    // Or a boolean indicating success or failure
    if (co_await attempt_some_network_operation(addr)) {
        std::cout << "succuess\n";
    } else {
        std::cout << "failure.\n";
    }
}
```

之后我们会实现一个自定义 awaiter。本节已经够长了。

Full source code for this section: [`co_demo.await.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.await.cpp).

---

### 恢复协程

现在我们几乎已经接近可用的协程了。我们可以构造一个协程，在里面运行代码，之后暂停它。但到这里协程依然没有用，我们需要在暂停后 *恢复* 协程。

恢复协程的办法是在它的 *coroutine handle* 上调用 *resume()* ，那么首先我们需要获取协程的句柄。

coroutine handle 是一个泛型类型，**参数是promise type**。所以如果你的 Promise type 叫做 P（举例），那么使用 P 作为 promise type 的协程其句柄就叫做 `std::coroutine_handle<P>`

（还有泛型 `std::coroutine_handle<>`，是 `std::coroutine_handle<void>` 的简写。这个句柄使用的是类型擦除后的 void* 泛型指针：可以存储 *任何* 类型的协程句柄，不管其promise type是什么）

协程句柄和 promise type 在同一时间由编译器帮你构造，他们二者互相转化非常简单：

- 通过 promise 得到 handle，调用 coroutine handle 的静态函数：from_promise() 即可，将你的 promise 对象传给他
- 通过 handle 得到 promise，调用 handle 的 promise() 方法，会返回响应 promise 对象的引用

我能构造协程实例时，实现会调用 Promise type 的 get_return_object() ，他已经跟 promise 对象的引用关联了（即*this），我们可以通过他来构造协程句柄。

我们得到他之后要干什么呢？**我们把它传递给用户感知类型的构造函数，因为那是用户通过操作对象来恢复协程**，用户需要知道哪里来寻找 coroutine handle。

`std::coroutine_handle<P>` 非常明确而且有点长，所以以后会通过 alias 来将他称为 handle_type。

```cpp
class UserFacing {
  public:
    class promise_type;
    using handle_type = std::coroutine_handle<promise_type>;

    class promise_type {
      public:
        UserFacing get_return_object() {
            auto handle = handle_type::from_promise(*this);
            return UserFacing{handle};
        }
        // ...
    };

    // ...
};
```

当然，你的构造函数也需要接受 coroutine handle 作为参数，然后把它作为数据保存。至少得是：

```cpp
class UserFacing {
	// ...
  private:
    handle_type handle;

    UserFacing(handle_type handle) : handle(handle) {}

  public:
    void resume() {
        handle.resume();
    }
};
```

实现了一个最简单的版本，通过给用户感知类型一个 resume 函数，单纯调用 handle 的 resume。

注意：构造函数现在是 private。我觉得这是一个使用 private 方法的不错例子：唯一合法的调用是在 promise_type 中调用的那个构造函数，这样的话 API 也完全是 promise type 和 用户感知类型之间内部的。因此你可以随心所欲的对其进行修改，而不用担心改变其他地方的用途。

在这种场景下，构造函数的调用在 promise_type 中，其定义在 UserFacing 内部，因此他自动是 UserFacing 的 private 成员。如果我们想把二者分开定义，我们需要显式声明 promise_type 为 friend，或者让构造函数 public。

现在我们就可以恢复我们的协程了，调用刚才实现的方法：

```cpp
UserFacing demo_coroutine() {
    std::cout << "we're about to suspend this coroutine" << std::endl;
    co_await Event{};
    std::cout << "we've successfully resumed the coroutine" << std::endl;
}

int main() {
    UserFacing demo_instance = demo_coroutine();
    std::cout << "we're back in main()" << std::endl;
    demo_instance.resume();
}
```

结果：

```
we're about to suspend this coroutine
we're back in main()
we've successfully resumed the coroutine
```

终于……讲了三节，我们终于可以暂停和恢复协程了！

（注意这里我移除了 co_return，之前他存在的意义是让函数变为协程，但是现在有 co_await 了，也能干一样的事，且我们不需要 co_return 来返回，我们可以让协程执行到底来自动返回）

Full source code for this section: [`co_demo.resume.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.resume.cpp).

---

### 处理协程状态

现在我们把协程句柄存储在用户感知对象里了，现在是时候处理无聊的部分：内存管理。

我们没有自己手动写分配 promise 对象的代码。C++ 实现帮我们在背后做了。所以我们需要担心的是他会如何被回收。如果我们不做的话，那么协程就会有 built-in 内存泄露。

并不能自动回收协程，你需要手动管理，通过调用 handle 的 destroy()

最简单的办法是先删除复制构造函数和复制赋值运算符，防止用户不小心赋值对象。其次，实现移动构造函数和移动赋值运算符，来移动 coroutine handle，这样的话如果用户移动了对象也不会造成 double free。

```cpp
class UserFacing {
    // ...

  private:
    handle_type handle;

    UserFacing(handle_type handle) : handle(handle) {}

    UserFacing(const UserFacing &) = delete;
    UserFacing &operator=(const UserFacing &) = delete;

  public:
    UserFacing(UserFacing &&rhs) : handle(rhs.handle) {
        rhs.handle = nullptr;
    }
    UserFacing &operator=(UserFacing &&rhs) {
        if (handle)
            handle.destroy();
        handle = rhs.handle;
        rhs.handle = nullptr;
        return *this;
    }

    ~UserFacing() {
        if (handle)
            handle.destroy();
    }
};
```

如果你 *需要* 允许用户感知对象复制的话，那么你就得更小心谨慎的设计结构防止double free，最简单的方式可能就是使用 shared_ptr 来进行管理。

---

### 通过 co_yield 传递值

现在我们理解了基础后，就可以进阶了。

第一件事是 co_yield

作为例子，我们让我们的 demo 协程生成一个值，并交给 caller，最后我们大致会写出类似：

```cpp
UserFacing demo_coro() {
    co_yield 100;
    for (int i = 1; i <= 3; ++i) 
        co_yield i;
    co_yield 200;
}
```

然后提供一个接口 `next_value()` 会按顺序返回 100,1,2,3,200

为了让协程中使用 co_yield 合法，promise type 必须提供方法 `yield_value()`,接受你想要生成的值的类型作为参数。这个例子中，我们定义 `yield_value(int)`

yield_value 的返回值接着会像你正常传给 co_await 一样，所以他必须是一个 awaiter，或者某些可以通过 await_transform 或者 operator co_await 转换为 awaiter 的类型。

这种情况中，最简单的办法是单纯让他返回一个 awaiter，使用平凡的 std::suspend_always 即可。然后，每个 co_yield 都会将控制权传递给 caller，以便于 caller 使用生成的值。

但是 co_yield 也会对他的参数做一些处理。C++实现本身不关心 yielded value 和协程的 caller 的交互。我们需要自己实现一些代码。

最简单的办法是将其存储在 promise 对象中，单独给一个字段存储：

```cpp
class UserFacing {
	class promise_type {
    	public:
        int yielded_value;
        std::suspend_always yield_value(int value) {
            yieleded_value = value;
            return {};
        }
    };
};
```

现在，当协程执行到 co_yield 100时，promise 对象会调用 yield_value(100)，根据上面的实现，100会被存储到成员变量中，之后返回 awaiter 来暂停协程。

暂停协程意味着控制流交还给将会调用 handle.resume() 的东西，在前一部分，这个通过用户感知类型的方法调用。所以我们应该修改方法让其返回 int 而不是 void，并且取出存在于 promise type 中的值。

```cpp
class UserFacing {
    public:
    int next_value() {
        handle.resume();
        return handle.promise().yielded_value;
    }
};
```

之后，前五次调用 next_value 会返回示例协程生成的值了。

但 *之后再* 调用一次会返回什么呢？

第六次调用 next_value() 协程会在 co_yield 200 语句后恢复，这是协程函数体中最后一个语句了，所以控制流会结束，之后协程执行完毕，并通过 final_suspend() 返回的 std::suspend_always 来暂停自己。

但什么都没有生成，没有东西调用 yield_value，也没有东西写入成员变量，仍然会获得跟之前一样的值。

换句话说，第六次调用会 *再返回* 一次 200，单纯是因为上一次调用产生的结果。

修复这个最简单的办法是在恢复协程 *之前* 写一些 dummy value，之后，他代表没有生成任何值，在 resume 后 dummy value 仍然在这。

我们可以考虑一些特殊的 int 值来代表 “没有被生成的值”，例如 0，-1，INT_MIN 或其他的东西。但是那不是 C++20 该做的，更好的办法是使用 `std::optional<int>`，

```cpp
class UserFacing {
    // ...

  public:
    std::optional<int> next_value() {
        auto &promise = handle.promise();
        promise.yielded_value = std::nullopt;
        handle.resume();
        return promise.yielded_value;
    }
};
```

现在的话，next_value 的返回值也是 `std::optional<int>`，这样 caller 就可以正确找到实际的值。

Full source code for this section: [`co_demo.yield.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.yield.cpp) (the simpler version without `std::optional`), and [`co_demo.yield_optional.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.yield_optional.cpp) (the full version that uses `std::optional` so it can can signal end of stream).

----

### 检测协程是否执行完毕

在上一部分，我们假设你调用 next_value() 6次，你依次接收 `std::optional<int>` 5次，一次包括 100 1 2 3 200，之后返回代表空序列的 optional

那继续调用 next_value 会发生什么？

这时，协程已经到达结束的位置了；final_suspend() 已经被调用。在这之后恢复协程会发生错误，导致崩溃。

如果你调用的代码 100% 会判断 std::nullopt，那么大概可以避免这个问题，因为你可能不会再次调用 next_value()，但如果你的代码不那么有条理，你想让多次调用 next_value 也安全，那么你可能需要在 *所有* 调用 next_value 之前先把 value 清空。

我们需要了解协程什么时候完成，然后不应该再恢复他了。幸运的是这个很简单，因为 handle 提供了 done() 函数，返回 bool

```cpp
class UserFacing {
    // ...

  public:
    std::optional<int> next_value() {
        auto &promise = handle.promise();
        promise.yielded_value = std::nullopt;
        if (!handle.done())
            handle.resume();
        return promise.yielded_value;
    }
};
```

好多了，现在你的协程就不会因为调用错误次数的 next_value() 而崩溃。

Full source code for this section: [`co_demo.done.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.done.cpp).

----

### 通过 co_return 返回最终结果

如何使用 co_return 返回一些额外的数据？

大多数情况下你大概率不会需要这个，但如果协程在执行一些网络任务，每次数据到达时 event loop 执行回调，那么可能使用 co_yield 和 co_await 跟网络设备交互（收发数据），之后使用 co_return 和 *程序* 中的其他需要该数据的部分交互，确定任务是否完成，或者查询的结果

如果你想 co_return 一个值，那么你需要定义 promise type 的 return_value 方法，接受一个你想 co_return 的类型的值。

```cpp
class UserFacing {
    // ...
  public:
    class promise_type {
      public:
        std::optional<std::string> returned_value;

        void return_value(std::string value) {
            returned_value = value;
        }
    };

    // ...
    std::optional<std::string> final_result() {
        return handle.promise().returned_value;
    }
};
```

常规来讲，return value 的行为取决于你，我把它存到了另外一个成员变量，并且提供了一个根据要求返回它的方法。所以你可以实例化这种类型的协程，一直调用 next_value 直到协程完成，最后调用 final_result 来得到当前状态的信息。

如果你这么做最重要的事情是：**你必须移除 return_void 方法**，这是标准规定，你 *只能* 实现 return_void 和 return_value 其一。

Full source code for this section: [`co_demo.return.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.return.cpp).

---

### 处理协程抛出的异常

我们已经实现了原始例子中的大部分方法，但是我们还有一个没有讲到的：`unhandled_exception()`

你可以按往常一样在协程内部抛出、捕获异常（虽然你不可以在 catch 块中 co_await 或者 co_yield），如果你抛出的异常 *没有* 在协程内被捕获，那么会发生什么？

首先，协程会自动关闭，就像普通函数抛出异常一样。你无法再恢复协程：因为你*无从恢复*。

但是在那之前，unhandled_exception() 会被 promise 对象调用。它实际上不接收任何参数，但是可以通过调用 std::current_exception() 来获取现在发生的异常，它的返回值是 std::exception_ptr。然后你可以向外传播异常，通过 std::rethrow_exception 来重新抛出。

许多情况下，最方便的方法是将协程中的异常传播到上次恢复时的调用点。例如，在 generator 式的协程中，你比较能接受的方式可能是将协程传播给调用 next_value 的人

（如果没有其他副作用的话，这也是 python 中生成器的工作方式，所以如果你想要那种方式的话，可以参考这个做法）

你的代码可能会是：

```cpp
class UserFacing {
    // ...
  public:
    class promise_type {
        // ...

      public:
        std::exception_ptr exception = nullptr;
        void unhandled_exception() {
            exception = std::current_exception();
        }
    };

    std::optional<int> next_value() {
        auto &promise = handle.promise();
        promise.yielded_value = std::nullopt;
        promise.exception = nullptr;
        if (!handle.done())
            handle.resume();
        if (promise.exception)
            std::rethrow_exception(promise.exception);
        return promise.yielded_value;
    }
};
```

注意我们让 promise.exception 初始化为 nullptr，原因与清除 promise.yielded_value 相同。如果不这么做的话，调用 next_value 时就会抛出跟上次相同的异常。

如果 unhandled_exception 什么都不做，那么异常会在协程退出后被丢弃。就好像协程体包含在隐式 try/catch 中一样，其中 catch 调用 unhandled_exception，然后假设这就是它需要执行的全部操作。

在协程抛出异常时，你可能*不仅仅* 想做这些事情，如果你的系统中的协程调用另外一个子协程（并且根据行为生成它），那么你可能会希望从子协程中抛出的异常传递给调用者协程，就像 caller/callee 普通函数一样。这种情况下，你可能仍然希望 unhandled_exception 存储异常，但是你需要在不同的场景下抛出。之后我们探讨这个问题。

Full source code for this section: [`co_demo.exception.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.exception.cpp).

---

### 编写自定义 awaiter

每次你的协程暂停时，甚至是 *潜在* 暂停，都会构造一个 *awaiter object*，并且用它来控制是否发生暂停以及其影响。

目前，我们都是使用标准库提供的两个 Awaiter，std::suspend_always and std::suspend_never。终于可以自己实现了。

awaiter 类型不需要继承任何特殊的类，只是单纯实现三个方法（某些的类型并不是固定的）

```cpp
class Awaiter {
  public:
    bool await_ready();
    SuspendReturnType await_suspend(std::coroutine_handle<OurPromiseType> handle);
    ResumeReturnType await_resume();
};
```

第一个 await_ready(), 控制协程是否暂停。如果其返回 true，那么协程会继续执行，如果返回 false 那么线程暂停。

（为什么这么设计？你执行 co_await 时大概是需要等待一些东西，例如，可能协程需要其他操作的返回结果，所以需要等待。那么这里的思想就是 await_ready() *一直测试* 你等待的东西是否完成，如果完成，返回 true，不然你也知道不需要继续等待了。）

如果 await_ready 返回 false，那么会调用 await_suspend，他接受协程 handle（这意味着他也可以访问 promise对象了，通过 handle.promise()），它的返回类型有几种选择：

- void，协程暂停并且将控制权返回给上次恢复它的东西
- bool，返回 true 时暂停，false 代表协程不需要暂停
- 返回 *另外一个协程句柄*。这种情况下协程会暂停，但是控制权不会返回给上次恢复它的东西，而是恢复执行返回的 handle 对应的那个协程。当然那个协程 *也* 可以在暂停后转换到其他协程。只有协程暂停且 *没有* 指定需要恢复的另外一个协程时，控制权才会被返回给 resumer。

如果让 await_suspend 在*某些条件下* 将控制权传给另外一个协程怎么样？那么你需要将其 handle 声明为函数的返回值（否则根本就无法获得其他协程了），但他最后还是需要返回一个值，用来不恢复其他东西，只是返回调用者。

为了实现这个，标准库提供了 'no-op coroutine' 总是暂停自己并且不做其他事情。所以如果你声明 await_suspend 返回协程 handle，然后在某些情况下你想返回给调用者，那么你通过返回 std::noop_coroutine 实现。

好了，那么怎么才能让 await_suspend 也可以 *在某些条件下* 不暂停呢？

这种情况，你需要返回你传入作为参数的那个协程 handle。然后同一个协程会被立马恢复，就好像一开始就没有暂停一样。

所以返回协程 handle 版本的 await_suspend 是最通用的形式：可以选择不暂停（返回其参数），暂停并且返回给 caller（返回 std::noop_coroutine），*或者* 转移到其他协程。

void 和 bool 版本是单纯上面行为的简化版子集。

最后，await_resume() 会在协程准备好继续执行时被调用（不管是因为他一开始暂停然后恢复了，还是根本就没暂停）。await_resume() 的返回值被传递给协程自己，作为 co_await 或者 co_yield 表达式的返回值。

例如，你决定写一个 awaiter，然后你可以使用 co_await 来等待一个网络任务完成，你就可以选择通过这个办法来将返回值传递给协程。你的代码会像是：

```cpp
UserFacing my_coroutine() {
    // ...

    std::optional<SomeData> result = co_await SomeNetworkTransaction();
    if (result) {
        // do something with the output
    } else {
        // error handling
    }

    // ...
}
```

就像你平时调用函数一样。

以上是 awaiter 可以做的所有事情，这里总结一下：

- 当协程被创建时，其让你可以选择它在开始时是否暂停，或者立刻执行直到遇到第一个 yield 或者 await 点。
- 在任何 co_await 或者 co_yield 调用时，awaiter 允许你配置那些操作是否暂停，或者将控制权交给不同的协程; 和程序交互，决定其他协程什么时候恢复；在 await 和 yield 结束后返回一个有用的值。
- 当协程结束时（不管是返回还是异常），这种情况下，不允许再运行协程，丹尼可以让他在终点暂停，或者将控制权转移给其他协程，作为其最后的行为。

Full source code for this section, demonstrating lots of simple custom awaiters: [`co_demo.awaiters.cpp`](https://www.chiark.greenend.org.uk/~sgtatham/quasiblog/coroutines-c++20/co_demo.awaiters.cpp).

---

### 使用 std::coroutine_traits 分辨 promise type

在前面的所有例子中，我们的 promise type 都定义在用户感知类型中，但是前文也提到，我们不止可以这么做。

当你写一个协程时发生了什么？C++实现会根据协程的函数签名实例化一个模板类 std::coroutine_traits，然后 *询问* promise type 是什么。

*默认* 的 STL 实现是寻找函数类型 T 的返回值，然后期望 T::promise_type 存在。但如果你不想在类中存储 promise type，你可以特化你自己的 std::coroutine_traits

这么做的目的是什么？

一个原因是有时候可能你无法将该类型放入你的类中，例如一些标准库类型，比如 std::unique_ptr。或者也可能是一些简单的东西，比如裸指针 甚至是 int。之后我们会展示一个例子，你可能会用到你无法控制的类型。

另外一个原因是 std::coroutine_raits 模板并不只是关注协程的 *返回* 类型，而且会关注参数的类型。所以如果你的 promise type 依赖那些的话，那么你就需要写一个模板特化。

以下是一个用来展示语法的简单示例：

```cpp
template <>
struct std::coroutine_traits<UserFacing> {
  	using promise_type = SomePromiseType;  
};
```

这会告诉编译器：如果一个协程的返回值类型是 UserFacing，*并且不接收任何参数*，那么它的 promise type 就应该是 SomePromiseType（假设你已经定义好了）

下一步，增加一些特殊参数：

```cpp
template <>
struct std::coroutine_traits<UserFacing, bool, char*> {
  	using promise_type = SomePromiseType;  
};
```

这个会精确匹配到返回值是 UserFacing 且接受那两个类型参数的协程。

但大部分情况下你应该不会关心参数，所以可能是这样：

```cpp
template <typename... Ts>
struct std::coroutine_traits<UserFacing, Ts...> {
	using promise_type = SomePromiseType;
};
```

这个特化会匹配 *任何* 返回值是 UserFacing 的协程，不管他几个参数。所以如果你的 promise type *只* 依赖协程的返回值，且你不想定义 UserFacing::promise_type，那么大概你就会这么做。

----

### 允许 promise type 访问协程的参数

目前的示例中，我还没有展示 promise 类中的构造函数。所以，C++会自动生成一个默认构造。

但这不是唯一的方式，如果有一个合适的构造函数，那么 promise 类将会按照接收的参数进行构造。例如，你可能会：

```cpp
class UserFacing {
  public:
    class promise_type {
      public:
        promise_type(int x, std::string_view sv) { /* ... */ }
        // ...
    };
};

UserFacing demo_coroutine(int x, std::string y) {
    // ...
    co_return;
}
```

之后当你调用协程的时候，`demo_coroutine(1, "foo")`，promise type 的构造函数就会被调用，接收到两个参数。

参数的类型也不一定需要完全相同，只要可以转换到合适的类型即可，就像普通函数一样。例如，我这里的构造函数接受一个 std::string_view ，而函数体则是 std::string，并且一样能用，编译器会自动进行转换。

这样做的话可以避免额外的拷贝：如果 promise_type 的构造函数单纯接收一个 string，那么就会额外调用一次拷贝构造来传递这个副本。除非你真的需要这么干，不然尽量避免节外生枝。

（当然，promise type 的构造函数接收的也可以是 const std::string& ，以前的 C++可以这样，但是现在已经有 string_view了）

*为什么* 要这么做？

这样的话协程的参数可以被当做控制 promise 类的办法，而不是让协程本身使用的。例如，假设你需要 promise 类包含一个指向 main-loop 的指针。最简单的办法就是通过构造 promise 对象时传入，所以你可能会做类似这样的事情：

```cpp
class UserFacing {
  public:
    class promise_type {
        MainLoopThingy *mlt;

      public:
        template<typename... ArgTypes>
        promise_type(MainLoopThingy *mlt, ArgTypes &&...) : mlt(mlt) {
            // maybe also tie this promise object into the main loop right here
        }

        // ...
    };
};

UserFacing demo_coroutine(MainLoopThingy *, int x, std::string y) {
    // this code ignores the MainLoopThingy and uses just the other parameters
    co_return;
}
```

这个例子中，我使用变参模板实现 promise type 的构造函数，所以它并不在意除了 MainLoopThingy 之外的任何参数。所以协程使用该 promise 类的协程不需要有相同的函数原型：他们 *只* 需要拥有 MainLoopThingy* 作为第一个参数即可。

如果你想这么干的话，你也许 *也* 会用到 std::coroutine_traits 来选择promise type，所以你可以使用不同的指针类型来定义协程，以此选择不同的 promise type。但这种情况的应用场景只在你需要让用户感知到的返回类型 *相同*。

另外一个特殊的情况是，你的协程是一个类成员函数。（这个情况完全支持，且之后我会更详细的描述。）在这种情况下，promise type 的构造函数（如果你只声明了一个参数），那么会接收类本身的引用作为第一个参数。如：

```cpp
class MyClass;

class UserFacing {
  public:
    class promise_type {
        MyClass *c;
      public:
        template<typename... ArgTypes>
        promise_type(MyClass &c, ArgTypes &&...) : c(&c) {}

        // ...
    };
};

class MyClass {
  public:
    UserFacing coroutine_method(int x, std::string y) {
        // ...
        co_return;
    }
};
```

这里，当 coroutine_method 被 MyClass 的实例调用时，promise type 的构造函数会接收该实例的引用（*this的引用），之后是 int 和 std::string 两个参数。

在 promise type 中，也存在一个跟之前示例类似的模板构造函数。但是构造函数希望接收 MyClass& 作为第一个参数，并且会存储指向该类的指针。这个保证 promise type 跟与他管理的类正常运作。

然而，注意！如果 MyClass 的实例是被拷贝的，那么 promise 类只能存储指向其中一个副本的指针。如果它被移动了，那么 promise 类也得更新指针。所以如果你这么做了，你就应该保证每个 MyClass 是不可移动的（delete copy ctor and move ctor *and* 运算符），或者让他是只能移动的类型并且在移动构造函数以及移动赋值运算符中更新与 promise type 管理的指针。

（这也是为什么用存储指向 MyClass 的指针而不是引用，指针才能被修改）

## 非平凡的协程系统的例子

现在我已经列出了许多 C++ 允许你使用 promise 和 awaiter 干的事情。所以原则上你已经有了足够的知识可以上路了。

但是可能再展示一些有趣的例子会更有帮助。