---
title: C++ 协程，以及其调度
description: 对 Dian-Lun Lin CppNow 2023 的演讲的翻译与总结。
slug: coro-schedule
date: 2024-02-16 00:00:00+0000
image: 
categories:
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Introduction to C++ Coroutines Through a Thread Scheduling Demonstration](https://www.youtube.com/watch?v=kIPzED3VD3w)

这个视频简单介绍了协程，并且有 compiler view，可以方便的看出协程的行为，之后实现了一个简单的调度器，很有学习价值。

协程的基础使用方法已经在文章 [Writing custom C++20 coroutine systems](https://rossqaq.github.io/article/coroutine-system/) 中介绍的足够详细，所以视频中关于协程基础的用法我会选择性翻译或者直接略过。

也就是说，本文假设你已经了解了协程的基础知识，例如 Promise，Awaitable 等等

重点摘录后半部分对于调度的讲述内容。

## Why coroutine

视频中举了一个例子，实际上这一部分也可以参考隔壁[微软 C# 的异步文档](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)，道理是一样的。

协程非常有用，如果你需要异步计算（例如GPU，TPU）或者做异步I/O

没有协程：

```cpp
void cpu_work() {
    cpu_matmul(matA, matB...);
}

void gpu_work() {
    cudaStream_t stream;
    cudaStreamCreate(stream);
    gpu_matmul<<<8, 256, 0>>>(matA, matB, ...);
    cudaStreamSynchronize(stream);		// 这里同步，意味着需要等待 gpu 计算的结果
    cudaStreamDestroy(stream);
}

// 实际上 CPU(假设CPU只有一个线程) 和 GPU 是独立的，所以完全没必要顺序执行。
int main() {
    cpu_work();
    gpu_work();
    
    // 或者
    gpu_work();
    cpu_work();
}
```

使用协程：

```cpp
void cpu_work() {
    cpu_matmul(matA, matB...);
}

Coro gpu_work() {
    cudaStream_t stream;
    cudaStreamCreate(stream);
    gpu_matmul<<<8, 256, 0>>>(matA, matB, ...);
    while (cudaStreamQuery(stream) != cudaSuccess) {
        co_await std::suspend_always{};
    }
    cudaStreamDestroy(stream);
}

int main() {
    auto coro = gpu_work();
    cpu_work();
    
    while (!coro.done()) { coro.resume(); }
}
```

## Coroutine/Promise Compiler's View

```cpp
Coro gpu_work() {
    cudaStream_t stream;
    cudaStreamCreate(stream);
    gpu_matmul<<<8, 256, 0>>>(matA, matB, ...);
    while (cudaStreamQuery(stream) != cudaSuccess) {
        co_await std::suspend_always{};
    }
    cudaStreamDestroy(stream);
}
```

↓ Complier View

```cpp
Coro gpu_work() {
    Coro::promise_type p();
    Coro coro_obj = p.get_return_object();
    try {
        co_await p.initial_suspend();
        cudaStream_t stream;
        cudaStreamCreate(stream);
        gpu_matmul<<<8, 256, 0>>>(matA, matB, ...);
        while (cudaStreamQuery(stream) != cudaSuccess) {
            co_await std::suspend_always{};
        }
        cudaStreamDestroy(stream);
    } catch (...) {
        p.unhandled_exception();
    }
    co_await p.final_suspend();
}
```

这就是 promise 和 coroutine 的行为。

对于 `co_await std::suspend_always{};` in Compiler View:

```cpp
// compiler transform
auto&& awaiter = std::suspend_always{};
if (!awaiter.await_ready()) {
    awaiter.await_suspend(std::coroutine_handle<>...);
    // <suspend/resume> 协程在这里选择性的暂停
}
awaiter.await_resume();
```

总而言之一句话：

Promise 控制协程的行为：`initial_suspend()`, `final_suspend()`, exception handling...

Awaitable 控制暂停点的行为

## Coroutine Handle

coroutine handle 就像是指向协程的指针，你可以通过它**访问 promise**

而 `std::coroutine_handle<>` 则是类型擦除版本，**它可以代表所有协程，但也代表了你不能用它访问 promise**

## Coroutine Tasks 以及 Scheduler APIs

接下来举个例子，来编写一个调度器

### 单线程调度器

在编写调度器之前，我们先看看想要调度什么样的协程

```cpp
Task TaskA(Scheduler& sch) {
    std::cout << "Hello from TaskA\n";
    co_await sch.suspend();
    std::cout << "Executing the TaskA\n";
    co_await sch.suspend();
    std::cout << "TaskA finished.\n";
}

Task TaskB(Scheduler& sch) {
    std::cout << "Hello from TaskB\n";
    co_await sch.suspend();
    std::cout << "Executing the TaskB\n";
    co_await sch.suspend();
    std::cout << "TaskB finished.\n";
}

// emplace: emplace a coroutine handle(task)
// schedule: 调度所有 emplaced task
// get_handle: 获取 coroutine handle
int main() {
    Scheduler sch;
    
    sch.emplace(TaskA(sch).get_handle());
    sch.emplace(TaskB(sch).get_handle());
    
    std::cout << "Start scheduling...\n";
    
    sch.schedule();
}
```

```cpp
struct Task {
	struct promise_type {
    	std::suspend_always initial_suspend() {return {};}
        std::suspend_always final_suspend() noexcept {return {};}
        Task get_return_object() {
            return std::coroutine_handle<promise_type>::from_promise(*this);
        }
        void return_void() {}
        void unhandled_exception() {}
    };
    
    Task(std::coroutine_handle<promise_type> handle) : handle(handle) {}
    
    auto get_handle() {return handle;}
    
    std::coroutine_handle<promise_type> handle;
};
```

由于 initial_suspend 是 std::suspend_always ，所以在 emplace 后，只会创建协程，而并不执行。

final_suspend 这里返回什么要思考一下，例如返回 suspend_never 就会很容易 ub

```cpp
class Scheduler {
	std::queue<std::coroutine_handle<>> tasks_;
    
public:
    void emplace(std::coroutine_handle<> task) {
        tasks_.push(task);
    }
    
    void schedule() {
        while(!tasks_.empty()) {
            auto task = tasks_.front();
            tasks_.pop();
            task.resume();
            if (!task.done()) {
                tasks_.push(task);
            }
        }
    }
    
    auto suspend() {
        return std::syspend_always{};
    }
};
```

如果使用队列的话，结果会是：

```
Start scheduling...
Hello from TaskA
Hello from TaskB
Executing the TaskA
Executing the TaskB
TaskA finished
TaskB finished
```

如果把 scheduler 的队列换成栈呢？变成了类似普通的函数

```
Start scheduling...
Hello from TaskB
Executing the TaskB
TaskB finished
Hello from TaskA
Executing the TaskA
TaskA finished
```

### 多线程调度器

```cpp
Task TaskA(Scheduler& sch) {
    std::cout << "Hello from TaskA\n";
    co_await sch.suspend();
    std::cout << "Executing the TaskA\n";
    co_await sch.suspend();
    std::cout << "TaskA finished.\n";
}

Task TaskB(Scheduler& sch) {
    std::cout << "Hello from TaskB\n";
    co_await sch.suspend();
    std::cout << "Executing the TaskB\n";
    co_await sch.suspend();
    std::cout << "TaskB finished.\n";
}

// emplace: emplace a coroutine handle(task)
// schedule: 调度所有 emplaced task
// get_handle: 获取 coroutine handle
int main() {
    Scheduler sch;
    
    sch.emplace(TaskA(sch).get_handle());
    sch.emplace(TaskB(sch).get_handle());
    
    std::cout << "Start scheduling...\n";
    
    sch.schedule();
    sch.wait();
}
```

执行的代码大多数没有任何区别，区别在于 Scheduler 的实现

```cpp
class Scheduler {
public:
    Scheduler(size_t num_threads);
    void emplace(std::coroutine_handle<> task);
    auto suspend();
    void schedule();
    void wait();
    
private:
    void enqueue(std::coroutine_handle<> task);				// 插入任务准备执行
    void process(std::coroutine_handle<> task);				// 恢复任务
    
    std::vector <std::coroutine_handle<>> tasks_;			// 存储所有的任务
    std::queue<std::coroutine_handle<>> pending_tasks_;		// 存储可以被恢复的任务
    std::vector<std::thread> workers_;						// 存储所有线程
    
    std::mutex mtx_;
    std::condition_variable cv_;							// block/unblock 现成
    bool stop_{false};										// 让所有线程返回
    std::atomic<size_t> finished_{};						// 记录完成的任务
};

Scheduler::Scheduler(size_t num_threads) {
    workers_.reserve(num_threads);
    
    for (size_t t = 0; t < num_threads; ++t) {
        workers_.emplace_back([this]() {
            while(true) {
                // 这里跟普通的调度器唯一的区别就是，std::function<void()> 变成了 std::coroutine_handle<>
                std::coroutine_handle<> task;
                {
                    std::unique_lock lock(mtx_);
                    // 如果谓词为 true，那么就不阻塞。
                    cv_.wait(lock, [this]{
                    	return stop_ || (!pending_tasks_.empty());
                    });
                }
                // 先检查 stop_，如果其为真，那意味着所有的任务执行完毕。
                if (stop_) return;
                
                task = pending_tasks.front();
                pending_tasks.pop();
                if (task) {
                    process_(task);
                }
            }
        });
    }
}

// 恢复任务
// 如果任务没有完成，就把它添加到 等待队列
// 如果任务完成，就增加 finished_，并检查是否所有任务已经完成
void Scheduler::process_(std::coroutine_handle<> task) {
    task.resume();
    
    if (!task.done()) {
        enqueue(task);
    } else {
        if (finished.fetch_add(1) + 1 == tasks_.size()) {
            std::unique_lock lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
    }
}

void Scheduler::enqueue(std::coroutine_handle<> task) {
    {
        std::unique_lock lock(mtx_);
        pending_tasks_.push(task);
    }
    cv_.notify_one();
}

void Scheduler::emplace(std::coroutine_handle<> task) {
    tasks_.emplace_back(task);
}

void Scheduler::schedule() {
    for (auto task : tasks_) {
        enqueue(task);
    }
}

void Scheduler::wait() {
    for(auto& w : workers_) {
        w.join();
    }
}
```

代码实现还可以，不是特别难。

接下来的部分是 CPU-GPU 的调度器，这个比较难，对我个人来说也不涉及，所以暂时不翻译。

