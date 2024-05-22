---
title: Structured Concurrency
description: 结构化并发，你需要了解。
slug: coro-structed-con
date: 2024-05-12 00:00:00+0000
image: 
categories:
    - coroutine
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

原文地址：[Structured Concurrency – Eric Niebler](https://ericniebler.com/2020/11/08/structured-concurrency/)

全文会将 Structured 翻译为 结构化的/结构化

写 C++ 协程库的时候遇到了各种各样的 lifetime 问题，发现 CppCon2023 有 lecture 描述了如何处理这些问题，什么才是正确的编程模式。这是那篇 lecture 中引用的一篇宗旨文章。

注意，你需要对协程有比较深入的了解，或者至少你也得了解协程地语法以及其他某种语言的异步编程方式，你才能看懂这篇文章。

## 序

太长不看版：**“结构化并发” 指代的是一种异步操作结构，其子部分保证完成于父部分之前，就像函数调用保证在它的 caller 结束前完成一样**。听起来这个很简单很无聊，但对 C++ 来讲所有事情都没那么简单。结构化并发对 C++ 协程异步架构的正确性和简单性有深远影响。它通过把异步 lifetimes 与 C++ 语句作用域对应，将 [现代 C++ 风格](https://docs.microsoft.com/en-us/cpp/cpp/welcome-back-to-cpp-modern-cpp?view=msvc-160) 代入了我们的异步程序，以此消除了引用计数管理生命周期的需求。

## 结构化编程与 C++

在上世纪 50 年代，新生的计算工业提出了结构化编程：高级语言拥有词法范围（lexical scopes）、控制结构，而 subroutines 会使得程序相较于使用 test-and-jump 指令以及 `goto` 变得更容易读写和维护。这是一个巨大的进步，以至于没有人讨论结构化编程，因为它只是“编程”。

> 意思就是概念早已经被人习惯了，现在没人提。

C++，比其他语言更复杂，充分利用了结构化编程。对象生命周期的语义代表了它跟其作用域的严格嵌套和绑定。例如，你代码的 *structure*。函数的执行流嵌套，作用域嵌套，对象的生命周期嵌套。对象的生命周期在大括号 `}` 处结束，然后对象会按照构造顺序的逆序进行析构。

现代 C++ 风格建立在这种结构化的基础上。对象拥有**值语义**，它们的行为就像 int，资源会在析构器中自动释放，这保证了对象不会在生命周期结束后被使用。**这非常重要**。

当我们放弃将对象生命周期和其作用域绑定时，那么，我们会在堆上使用引用计数来管理一个对象、或者我们使用单例模式；此时，我们正在与语言优势对抗而不是跟语言协作。

## 多线程带来的麻烦

编写正确的并发程序远比单线程程序困难得多。有许多原因，一是因为线程和单例一样都是动态分配的对象，根本不会嵌套在作用域内。即使你使用现代 C++ **提供** 的线程，但当逻辑跟生命周期跨线程时，程序的层级结构会让人摸不着头脑。我们在单线程下用以管理代码复杂性的工具 —— 将嵌套的生命周期和嵌套的作用域绑定，无法替我们管理异步代码。

为了进一步解释，我们看看将同步函数变为异步函数会发生什么：

```cpp
void computeResult(State & s);

int doThing() {
    State s;
    computeResult(s);
    return s.result;
}
```

`doThing` 非常简单，它声明了一些内部状态，调用一个 helper，然后返回结果。现在我们要把两个函数都变成异步的，因为它们可能需要比较长的时间。我们直接使用 Boost future，它支持连续的链式调用：

```cpp
boost::future<void> computeResult(State& s);

boost::future<int> doThing() {
    State s;
    auto fut = computeResult(s);
    return fut.then([&](auto&&) {return s.result; });	// OOPS
}
```

如果你使用过 future 的话，你可能会大喊：“不~~！”，`.then` 指定了一些在 `computeResult()` 完成后的工作，`doThing()` 之后返回结果 future。问题在于，`doThing()` 返回时，`State` 的生命周期结束了，**并且 continuation 捕获了它的引用**，现在这是悬垂引用，很可能会导致崩溃。

问题在哪？futures 允许我们计算一个现在还不需要的结果，Boost 风格允许我们链式执行，但是 continuation 是一个单独的函数，具有单独的作用域。没有嵌套的作用域，没有嵌套的生命周期，我们需要手动管理 state 的生命周期。

```cpp
boost::future<void> computeResult(std::shared_ptr<State> s);

boost::future<int> doThing() {
    auto s = std::make_shared<State>();
    auto fut = computeResult(s);
    return fut.then([s](auto&&) {return s.result; });
}
```

因为两个异步操作都需要 state，所以它们都有责任确保其存活。

另一种思考方式是：**这个异步计算的生命周期是什么？**它从 `doTing()` 被调用开始，直到 continuation，传入 `then` 的 lambda 返回。**并没有作用域跟这个生命周期对应**，这就是问题的根源。

## 非结构化并发

当我们考虑 executor 时情况会变得更复杂。executors 用来解决执行时的上下文，你可以在它上面调度任务，通常可以是一个线程或者线程池。许多代码库都有 executor 的概念，其中有一些允许你使用 defer 或者其他策略安排调度。我们可以做一些有意思的事情，比如把计算从一个 IO 线程池挪到 CPU 线程池，或者延迟后重试一些异步操作。`goto` 可以派上用场，但是它非常低级，而且一点也不清晰。

举个例子，最近我遇到了一个算法，它通过执行器和回调（listener）来重试某些资源的异步分配。以下是一个删减后的版本：

```cpp
// 异步操作完成后会调用该 continuation
struct Manager::Listener : ListenerInterface {
  shared_ptr<Manager> manager_;
  executor executor_;
  size_t retriesCount_;
 
  void onSucceeded() override {
    /* 分配成功了…… */
  }
  void onFailed() override {
    // 分配失败，向 executor 请求过后再分配一次
    auto alloc = [manager = manager_]() {
      manager->allocate();
    };
    // 未来某个时刻重新分配
    executor_.execute_after(
      alloc, 10ms * (1 << retriesCount_));
  }
};

// 使用以上的类作为 continuation 尝试异步分配资源
void Manager::allocate() {
  // 我们已经尝试过很多次了吗？
  if (retriesCount_ > kMaxRetries) {
    /* 通知观察者，分配失败了 */
    return;
  }
 
  // 再试一次：
  ++retriesCount_;
  allocator_.doAllocate(
    make_shared<Listener>(
      shared_from_this(),
      executor_,
      retriesCount_));
}
```

`allocate()` 成员函数首先检查异步操作执行了多少寸，如果没执行过，就直接调用 `doAllocate()` ，传一个回调来通知它是成功还是失败。失败的话，handler 会向执行器再提交一个 deferred 任务，会重新调用 `alloate`，过后再重试分配。

这是一个有繁多状态并且非常迂回的异步算法。逻辑生成了很多函数、很多对象，控制流和数据流都不明显。还得注意保证对象生存期的引用计数。向执行器提交任务让它更加困难。这段代码中的执行器没有 continuation 的概念，所以在任务执行中的错误也无处可去。`allocate()` 函数不能通过异常传递错误以此从错误中恢复，错误处理必须手动完成。如果我们想支持取消，也是同理。

这是一种 **非结构化并发**：我们以一种 *临时* 的方式 queue up 异步操作；我们串联相关的工作，使用 continuations 或者 strand 执行器保证顺序一致；我们使用强弱引用计数保证数据在我们需要时存活。没有正式的描述任务 A 是任务 B的子任务的概念，无法强制子任务在父任务前完成，我们也不能指着代码中的某一处说：这是算法。

> *如果你不介意的话，通过执行器进行的跳跃有那么一点像是非 local 的 `goto` 语句，不管是时间上还是空间上。“X ms 后，在某个线程上，立刻 jump 到程序中的这一点。”*

这种 non local 的不连续性使得我们很难推导程序的正确性和效率。将这种非结构化的并发扩展到整个程序，然后处理许多并发的实时事件，手动处理代外异步数据的控制和数据量，控制并发访问共享状态，管理对象生命周期都很难受。

## 结构化并发

在很久之前，非结构化的编程就已经给结构化风格让路了。有 C++ 协程的支持，使得如今的很多异步代码也发生了类似的变化。如果我们使用协程重写以上算法（使用了 [cppcoro](https://github.com/lewissbaker/cppcoro)），看起来就会是：

```cpp
// 尝试分配资源，失败就会重试
cppcoro::task<> Manager::allocate() {
  // 尝试分配，直到次数达到 kMaxRetries;
  for (int retriesCount = 1;
       retriesCount <= kMaxRetries;
       ++retriesCount) {
    try {
      co_await allocator_.doAllocate();
      co_return; // success!
    } catch (...) {}
 
    // 失败，让出线程，稍后重试
    co_await scheduler_.schedule_after(
      10ms * (1 << retriesCount));
  }
 
  // 错误，失败太多次
  throw std::runtime_error(
    "Resource allocation retry count exceeded.");
}
```

*说明：cppcoro 的 scheduler 和 上文的 executor 概念类似。*

我们列出以上做法的优点：

1. 只有一个函数，非常有逻辑性。
2. 状态（例如 `retriesCount`）可以被轻松的维护，而不需要使用引用计数。
3. 我们可以使用普通的 C++ 错误处理技术。
4. 保证结构化，即异步调用 `alllocator_.doAllocate()` 在函数继续执行前完成。

第四点有深刻的意义。考虑文章开头的简单的例子，以下的使用协程的实现非常安全：

```cpp
cppcoro::task<> computeResult(State& s);
cppcoro::task<int> doThing() {
    State s;
    co_await computeResult(s);
    co_return s.result;
}
```

以上代码很安全，因为我们知道 `computeResult` 会在 `doThing` 恢复前完成，也即 `s` 析构之前。

> *有了结构化并发，将 local 变量作为引用传递给子任务来 await 非常安全。*

## 取消

如果使用结构化并发的方法，那么并发操作的生命周期就会严格内嵌于资源的生命周期中，绑定在其作用域上，允许我们避免像是 `shared_ptr` 之类的垃圾回收机制。这样代码会更加效率，只需要更少的堆分配以及很少的 atomic ref count 操作，当然代码也更易读而且 bug 更容易分析。

然而，这种方法有一个隐含的要求，即**我们必须在父操作完成前 join 并且 wait 子操作**。我们并不能 detach 这些子操作，然后让资源自动释放。为了避免在结果已经使用过的子操作上进行不必要的等待，我们需要一个机制来取消这些子操作，这样它们才能尽快结束。因此，结构化并发模型需要对取消操作的深度支持，以避免不必要的延迟。

注意，我们会在每次向子协程传递 local 变量的时候依赖结构化生命周期以及结构化并发。我们必须确保子协程完成并且不再需要那个变量，之后父协程中 local 变量作用域结束再被销毁。

## 结构化并发 > 协程

当我说起“结构化并发”时，我不仅仅是说协程，虽然协程是最明显的表现方式。为了解释我的意思，我们简单的讨论一下协程**是什么**、**不是什么**。注意，C++的协程根本没有固有的并发性质！它们只是编译器把函数改为回调的一种方式。

考虑以下简单的协程：

```cpp
cppcoro::task<> computeResult(State& s);

cppcoro::task<int> doThing() {
    State s;
    co_await computeResult(s);
    co_return s.result;
}
```

`co_await` 是什么意思？很老套的答案：这取决于作者想让 `cppcoro::task<>` 是什么。完整的答案是，`co_await` 暂停当前协程，将协程中剩下的内容打包（这里即 `co_return s.result;`），作为 continuation，然后把它传入 awaitable 对象（这里是 `task<>` 通过 `computeResult` 返回）。awaitable 会把它存到某个地方，在子任务完成之后就可以调用了。这是 `cppcoro::task<>` 做的。

换句话说，`task<>` 类型以及语言的协程一起将结构化并发置于无聊的回调之上。以上。这就是所有的 magic。只是回调而已，但是是另一种模式的回调，而这种模式使其变得结构化。这种模式保证了子操作在父操作前完成，就是这种属性带来了收益。

一旦我们认识到结构化并发只是回调的一种特殊模式后，我们就会发现我们可以**不使用协程**实现结构化并发。使用回调当然不是什么新鲜事，这些模式可以被编码到一个库中然后变得可复用。这就是 [libunifex](https://github.com/facebookexperimental/libunifex) 做的。如果你关注 C++ 标准委员会，就知道这是 [Executors 提案](http://wg21.link/P0443) 中的 sender/receiver 抽象。

使用 libunifex 作为结构化并发的基础，我们就可以写出以下的代码：

```cpp
unifex::any_sender_of<> computeResult(State& s);

auto doThing() {
    return unifex::let_with(
    // 声明 "local variable":
    [] {return State{};},
    // 使用 local 来构造异步任务
    [](State& s) {
        return unifex::transform(
            computeResult(s),
            [&] { return s.result; })；
    });
}
```

我们都有协程了为什么还要写这个？你需要一个更好的解释，我只能抛砖引玉。有了协程，你会在协程第一次被调用时进行分配，然后在它每次恢复后间接调用一个函数。编译器有时可以消除其开销，有时不行。而通过结构化并发的方式直接使用回调，我们可以直接获得同样的收益但是没有协程调用的开销。

这种风格的编程导致了不一样的开销：很难编写出跟协程一样可读性的代码。我认为未来 90% 的异步代码都会因可维护性而使用协程编写。对于 hot code，选择性地使用 lower-level 的方式替代协程。

## 并发

我在上面提到，协程本身并不是并发的；它们只是回调的一种编写方式。协程本质上是顺序的，而`task<>` 的惰性（协程开始时暂停，被 awaited 时才执行）代表我们不能使用它来向程序中引入并发。已有的 `future`-based 代码经常假设操作已经 eagerly 开始，引入 *临时* 的并发你就需要小心的处理。这会迫使你一遍又一遍地使用 *临时* 的风格重新实现并发模式。 

有了结构化并发，我们就可以在算法中贯彻并发模式并且以结构化的方式引入并发。例如，我们如果有一堆 `task`s 并且想要等待它们都完成后把它们的结果作为 tuple 返回，我们可以将其传入 `cppcoro::when_all` 并且 `co_await` 结果。（libunifex 也有 `when_all` 算法）

目前，不管是 cppcoro 还是 libunifex 都没有 `when_any`，所以你不能运行并发操作然后当 *第一个* 操作完成后就返回。虽然这是一个非常重要又有趣的基础算法。为了维护结构化并发的保证，当第一个子任务完成时，`when_any` 需要取消其他所有的任务，**并且等待它们完成**。此算法的效果取决于其他的异步操作对于你的取消请求的响应速度，这表明了对取消的深度支持在现代异步程序中的重要性。

## Migration

目前，我已经讨论了什么是结构化并发以及它为什么重要。我还没有讨论我们怎么达成。如果你已经使用协程来编写异步 C++ 了，恭喜。你可以继续享受到结构化并发带来的收益，也许还对协程为什么具有改革性有了更深的理解。

对于缺少结构化并发的代码，对取消的支持，或者是异步机制的抽象都极具难度。它可能从引入复杂性开始，开辟一座孤岛，周围的代码保证结构化并发需要的条件。例如，这包括创建一个任务的取消操作，即使底层的执行上下文不直接提供取消。增加的复杂性可以被隔离在一层中，结构化并发的孤岛构建于其之上。然后剩下的就是简单的操作了，对于采用 future- 或者 callback-style 的代码，把它们转化为协程方式，理清父子关系、所有权，以及生命周期。

## 总结

`co_await` 的加入在不干预计算结构的基础上把同步函数变成异步函数。被 awaited 的异步操作必须在调用它的函数完成前完成，就像是普通的函数调用。革新的地方是：**没有改变的地方**。作用域和生命周期仍然像往常一样嵌套，只是现在的作用域并不是连续的了。使用传统的回调和 future，这种结构就没有了。

协程，以及更广泛意义上的结构化并发，带来的是现代 C++ 的风格：值语义，算法驱动的设计，清晰的所有权语义 with deterministic finalization，以上这些都加入了一步变成。它这么做的方式是因为它把异步的生命周期绑定回普通的 C++ 作用域上。协程把我们的异步函数变成拥有挂起点的回调函数，回调函数使用非常特殊的模式被调用，以此来维护严格嵌套的作用域、生命周期，以及函数栈（function activations）

我们在代码中使用 `co_await` ，然后我们可以继续使用我们熟悉的：异常来进行错误处理、local 变量、析构函数释放资源、值/引用传递参数，以及其他任何的 good、safe、惯用的现代 C++。

感谢阅读。

---

了解更多的话，一定要看看 CppCon 2019 Lewis Baker 的 [Structured Concurrency: Writing Safer Concurrent Code with Coroutines...](https://www.youtube.com/watch?v=1Wy5sq3s2rg) 的演讲。
