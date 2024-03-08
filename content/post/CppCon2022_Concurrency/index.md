---
title: C++ 的并发 API 总结
description: 对 CppCon 2022, Anthony Williams 演讲的翻译与总结。
slug: CppCon 2022 Concurrency
date: 2023-12-29 00:00:00+0000
image: cppcon2022-cover.png
categories:
    - cppcon
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[An Introduction to Multithreading in C++20 - Anthony Williams - ACCU 2022 ](https://www.youtube.com/watch?v=3uH-2CkBlPQ)

CppCon 2022 也讲了，以前记录过。这次选用 ACCU 的版本，因为内容更全一些。

Anthony 应该都认识，写《C++并发编程》第二版的那个作者。这是他在 ACCU/CppCon 的讲座的总结，内容是关于 C++20 的所有并发 API。

首先那个并发执行策略直接跳过，17进入标准，马上就要废弃了纯纯没用。

## 取消操作

假设你的 GUI 下载程序支持取消操作，你点一下下载，发现需要很长时间。你可能会说，啊，快取消这个操作，然后让其他线程帮他清理资源。

C++20 提供了两个工具，`std::stop_source`, `std::stop_token` 来解决协作任务的取消。

`std::stop_source` 用来检查是否有人让你取消操作。

**取消是一个协作任务，如果别人让你取消，但你不检查 `std::stop_source`，那么什么都不会发生。**

---

具体的用法如下：

1. 创建 `std::stop_source`
2. 从 `std::stop_source` 获取 `std::stop_token`
3. 将 `std::stop_token` 传递给一个新的线程或者 task（std::async）
4. 当你想取消操作的时候，就调用 `source.request_stop()`
5. 周期性检测：`token.stop_requested()` 来查看是否有人让你暂停
6. **如果你不检查**，那么什么都不发生

```cpp
void stoppable_func(std::stop_token st) {
    while (!st.stop_requested()) {
        do_stuff();
    }
}

void stopper(std::stop_source source) {
    while(!done()) {
        do_something();
    }
    source.request_stop();
}
```

他们背后并没有什么同步机制，总之你得自己检查。

----

你可以使用 `std::stop_callback` 来提供你自己的取消机制，如：

```cpp
Data read_file(std::stop_token st, std::filesystem::path filename) {
    auto handle = open_file(filename);
    std::stop_callback cb(st, [&]{cancel_io(handle);});
    return read_data(handle);  // 阻塞
}
```

这样可以在结束后执行回调。

## 管理线程

### std::jthread

C++20 后，`std::jthread` 才是你的首选；很少的情况下也许会使用 `std::async`。

`std::jthread` 会自动提供你 `stop_token`（当然，前提是你的函数接收这个参数，不接受的话就不会给你）

```cpp
std::jthread t{my_func, arg1, arg2};	// 调用 jthread
my_func(stop_token, arg1, arg2);		// 会得到 stop_token
```

我们来看看 `std::jthread` 的 API：

- `std::jthread` 默认构造
  - 创建一个空的对象
- `std::jthread x{Callable, Args...}`
  - 创建新的 `std::stop_source` - src
  - 创建新线程调用 `Callable(src.get_token(), Args...)` or `Callable(Args...)`
- 析构函数
  - 调用 `src.request_stop()`，然后等待线程结束后帮你 join
- `x.get_id()`
- `x.join()` 等待线程结束，不会析构对象

`std::jthread` 是一个值类型，已经是一个 handle 了，是可移动的，可以转移所有权，可以被存储在容器里。**不要 new**。

> Anthony 表示求你了，人家已经是一个 handle 了，别再 new 了。（说实话我也不懂为什么有人这么干。对 std::thread 就有人这么干。难道是写 `->` 运算符看起来高级吗？）

### 线程：Callable and Args

callabe 和 args 会被**拷贝**到新线程的 local。

这个主要是防止悬垂引用和竞争，如果想用引用请用 `std::ref()` 或者 lambda。

### 取消 API

```cpp
std::jthread x{some_callable};


x.get_stop_source();

x.get_stop_token();

x.request_stop();
// 等同于 x.get_stop_source().request_stop()
```

## 同步工具

多数情况下，线程之间有交互，例如数据交互。那么要小心**数据竞争**。在 C++ 中，数据竞争全是 ub。

C++20 提供了一大堆同步工具：

- latches
- barriers
- futures
- mutexes
- semaphores
- atomics

这个顺序是一会讲解的顺序，也是推荐你使用的顺序，按照这个顺序依次思考，这个组件是否能解决你的问题。

> Anthony 表示，例如 atomic 就很容易用错，不过你的团队里总有人知道怎么正确使用。所以普通人不懂的就不要乱用。
>
> （确实，不懂内存序那也用不明白 atomic。）

### Latches

`std::latch` 是**只能使用一次的计数器**，线程们等待它归零后通行。

1. 创建一个非 0 count 的 latch
2. 一个或者多个线程减少 count（原子的）
3. 其他线程可能等待 latch 被触发
4. 当 latch 归零的时候，会**保持触发**，所有的等待它的线程都会被唤醒

```cpp
std::latch x{cnt};

// 减少 count
x.count_down();

// 等待 latch 触发
x.wait();

// x.count_down() + x.wait()
x.arrive_and_wait();
```

> 就像一个结界，latch 归 0，线程就都能通过了。

这个也可以用于测试并发，让所有的线程等待一个 latch，之后执行你的可能会发生 data race 的代码。

### Barriers

`std::barrier<>` 是一个模板类：

1. 构造一个 barrier，需要一个 count，以及一个 **Completion Function**
2. 一个或者多个线程到达 barrier
3. 等待 barrier 被触发
4. count 归 0，触发 barrier，**会调用 Completion Function**，然后**重复以上过程**。

```cpp
std::barrier<task_type> x{cnt, task};

// 递减 count，如果触发了 barrier，就会执行 completion 函数
auto arrival_token = x.arrive();

// 等待 completion 函数完成
x.wait(arrival_token);

// x.wait(x.arrive());
x.arrive_and_wait();

// 使 cnt 永远减少（当然也可能触发 completion 函数）而不需要等待
x.arrive_and_drop();
```

在游戏渲染可能比较有用，因为游戏每一帧都要同步。而 nvidia 对这俩都有特殊的指令支持。

barrier 可以循环使用，提供了 completion function 也方便在触发后执行一些操作，比如写入文件。

### Futures

#### std::future

有时你只是单纯想把数据从一个线程传递到另外一个线程，future 就是干这个的。

futures 用于线程间的数据传递。

- `std::async` 发起任务，并返回一个值。**推荐的使用方法时，使用 `std::async` 来立马进行一个计算，之后会使用 future 获取值。否则你可能需要的是 `std::packaged_task`**
- `std::promise` 显式设定值
- `std::packaged_task` 将返回一个值的任务封装。它会保存一个任务，你可以在未来对其进行调用，之后获取 future。**可以使用这个配合线程池，来获取返回值**

以上所有这些，你都可以使用 `std::future<T>` 来获取结果。

```cpp
// 空 future
std::future<T> default_ctor;

// 检查 future 的 state
f.valid();

// 等待数据完成
f.wait();
f.wait_for(duration);
f.wait_until(time_point);

// 等待数据并且获取它。数据如果已经完成，那么也不会阻塞
x.get();
```

---

#### std::promise

promise 可能是你用的最多的：

```cpp
// 空 promise
std::promise<T> default_ctor;

// 检查 promise 的 state
p.valid();

// state 中设置值
p.set_value();

// state 中设置异常
p.set_exception(ex_ptr);

// 获取当前状态下的 future 
p.get_future();
```

使用 future/promise 传递数据：

```cpp
std::promise<MyData> prom;
std::future<MyData> f = prom.get_future();

std::jthread th1{ [f=std::move(f)] { do_stuff(f.get()); }};
std::jthread th2{ [&prom] { prom.set_value(make_data()); }};
```

注意，**future 是一次性的，所以你需要注意移动。此外，调用 get() 后，其就不拥有值了**。

对于异常：

```cpp
std::promise<MyData> prom;
std::future<MyData> f = prom.get_future();

// 该线程内部抛出异常
std::jthread th1{ [f=std::move(f)] { do_stuff(f.get()); }};
std::jthread th2{ [&prom] { prom.set_exception(
    std::make_exception_ptr(my_exception{})); 
}};
```

----

#### std::async

还可以使用 `std::async` 来发起一个任务：

`std::async` 可以用来创建一个线程，只要你第一个参数传递 `std::launch::async`，否则的话是否新建一个线程是实现定义。

```cpp
auto f = std::async(std::launch::async, func, arg1, arg2);
```

- `f.get()` 会返回 func 的结果

- f 拥有一个类似 `jthread` 的线程，析构函数会帮你处理现成的

#### std::shared_future

`std::shared_future` 允许你让**多个线程**收到执行结果。

```cpp
std::promise<Data> prom;
std::shared_future<Data> f = prom.get_future().share();

std::jthread th1 {[f]{do_stuff(f.get());}};
std::jthread th2 {[f]{do_stuff(f.get());}};
```

----

### std::mutex

future 比 mutex 好一点，但因为它是一次性的。所以你也许需要 mutex

C++ 提供了 6 种 mutex，实则有 5 个没啥用

```cpp
std::mutex;	// ← 用这个
std::timed_mutex;
std::recursive_mutex;
std::recursive_timed_mutex;
std::shared_mutex;  // ← 读写锁，而实践中用这个很蛋疼。用 mutex 就行。
std::shared_timed_mutex;
```

对于上锁和解锁，我们也有 RAII 封装：

```cpp
std::scoped_lock;	// ← 用这个
std::unique_lock;
std::lock_guard;
std::shared_lock;
```

你把所有的锁都给 `std::scoped_lock` ，他会一起给你上锁，而且保证不会死锁。

## 等待数据

如何等待数据完成？

忙等吗？这个是个 bad idea，浪费 cpu 性能。

```cpp
std::mutex m;
std::optional<Data> data;

void busy_wait() {
    while (true) {
        std::scoped_lock lock(m);
        if (data.has_value()) break;
    }
    process_data();
}
```

所以我们有 `std::condition_variable` 来通知，他并不能同步数据，但可以**避免忙等待**。

我们需要一个 **`std::unique_lock`，因为其跟 scoped_lock 不合**

```cpp
std::mutex m;
std::condition_variable cond;
std::optional<Data> data;

void cv_wait() {
    // 会锁定互斥量
    std::unique_lock lock(m);
    cond.wait(lock, []{ return data.has_value(); });
    process_data();
}
```

首先你需要给 `mutex` 上锁，然后你再调用条件变量来传递 `lock` 以及一个 lambda，表明：**你到底在等待什么？**当你的 lambda 返回 true 时，就会替你解锁。

对于 `std::condition_variable` ，使用忙等实现是合法的，但是操作系统不会这么做，因为操作系统知道这样做不好。如何通知呢？你需要调用一下 `notify()`

```cpp
void cv_notify() {
    {
    	std::scoped_lock lock(m);
    	data = make_data();
    }
    cond.notify_one();
}
```

如何使用条件变量配合取消操作呢？使用 `std::condition_variable_any`

```cpp
std::condition_variable_any cond;

void cv_wait(std::stop_token token) {
    std::unique_lock lock(m);
    if(!cond.wait(lock, token, []{return data.has_value(); })) {
        return ;
    }
    process_data;
}
```

现在 wait 会返回一个布尔变量，告诉你你的谓词是否返回 true，如果返回 false，就代表我需要暂停操作，因为某些线程告诉我 stop。

当然，如果同时发生 stop 和数据准备好两个事件，那么依然会返回 true，那么依然会处理数据。具体怎么做取决于你，如果你的操作很快，那可以继续，如果想取消，就显式判断 stop。

## 信号量

如果以上的工具都不能满足你（显然都不满足你有点难度），那么你可能需要信号量。

信号量代表一些可以被使用的 "slots"。如果你 **acquire** 一个 slot，那么信号量会减少，直到你 **release** 该 slot。

**acquire** 一个 slot，但是 count 为 0 时，会阻塞或者失败。

信号量可以实现所有的同步机制，包括 latches, barriers, mutexes，当然，大部分情况下你直接用更高级的结构即可。

**`binary_semaphore` 有 2 个状态：1 slot free，no slots free，就像 mutex 一样使用。**

C++20 提供了 `std::counting_semaphore<max_count>`, `std::binary_semaphore` 是其 `max_count = 1` 时的特化。

有阻塞的 `sem.acquire()` 也有 `sem.try_acquire()`，..for, ...until，替代阻塞的版本。

## 原子变量

原子变量是最最底层的同步设施了。在 C++ 里是 `std::atomic<T>`

T 必须 **可平凡拷贝**，**Bitwise comparable**

### 原子变量 lock_free 吗？

`std::atomic<T>` 可能不是 lock-free 的，可能会使用内部 mutex。

`std::atomic_flags`, `std::atomic_signed_lock_free`, `std::atomic_unsigned_lock_free` 是唯三保证 lock-free 的类型。

在大多数平台上，`std::atomic<integral-type>` 以及 `std::atomic<T*>` 是 lock-free 的。

你可以通过 `std::atomic<T>::is_always_lock_free` 来查看到底是不是 lock-free。
