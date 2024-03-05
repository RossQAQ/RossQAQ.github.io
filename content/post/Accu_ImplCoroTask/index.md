---
title: C++ 协程手把手实现 Task
description: 对 Dietmar Kühl ACCU 2023 的演讲的翻译与总结。
slug: coro-tasks
date: 2024-02-25 00:00:00+0000
image: 
categories:
    - coroutine
    - techs
    - cppnotes
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

[Implementing a C++ Coroutine Task from Scratch - Dietmar Kühl - ACCU 2023](https://www.youtube.com/watch?v=Npiw4cYElng)

众所周知，直到 C++23 标准库都没有什么可用的协程组件，所以我们来自己实现一个 Task 封装。

协程的基础知识不会再介绍，做简单总结。详细可以参考之前的博文。

## 什么可以被 co_await?

- 异步操作：I/O，等待其他线程
- 外部事件：用户交互，其他程序
- 不可用资源：内存，fd
- 重排序数据：批处理请求
- 延迟访问：未来才会使用的数据

## 关键概念

- Awaiter：指定异步操作该如何执行
- Promise type：指定协程如何操作

## value = co_await expr?

这一句代码发生了什么？简化版：

```cpp
auto awaiter = operator co_await(expr);

if (awaiter.await_ready()) {
    awaiter.await_suspend(handle-to-coro);
    <resume here when handle.resume() is called>
}

value = awaiter.await_resume();
```

Awaiter Type：

- await_ready() : 告诉系统是否需要暂停
- await_suspend(std::coroutine_handle<> h) :
  - 设定一些条件以便于恢复协程
  - 在调用 h.resume() 前安排一些工作
- await_resume(): 产生 awaited 表达式的值

## C f() {... co_foo ...}

```cpp
auto* f = new frame<C::promise_type>();
C rc = f->promise.get_return_object();
invoke([&]) {
    try {
        co_await f->promise.initial_suspend();
        {... co_foo ...}
        co_await f->promise.final_suspend();
    } catch (...) {
        f->promise.unhandled_exception();
    }
});
return rc;
```

Promise Type：

- initial_suspend(): 如何启动协程
- final_suspend(): 如何结束协程
- unhandled_exception() : 抛出异常怎么办
- get_return_object() : 工厂函数，如何创建协程的返回值

## 实现 Task

建议理清当前的执行流程~

实现了一个并不是 io context 的 io context，并没有做 io，但你可以看到它一起运行了这些。

```cpp
#include <iostream>
#include <coroutine>
#include <unordered_map>
#include <functional>

template<typename V>
struct value_awaiter {
	V value;
	constexpr bool await_ready() { return true; }
	void await_suspend(auto) {}
	V await_resume() { return value; }
};

struct task {
	struct promise_type {
		std::exception_ptr error{};
		std::suspend_never initial_suspend() { return {}; }
		std::suspend_never final_suspend() noexcept { return {}; }
		task get_return_object() { return {}; }
		void unhandled_exception() { error = std::current_exception();  }
		void return_void() {}

		// if you want co_await everything
		/*template <typename V>
		auto await_transfrom(V v) { return value_awaiter<V>{ v }; }*/
	};
};

// 假装是 context
struct io {
	std::unordered_map<int, std::function<void(std::string)>> outstanding;
	void submit(int fd, auto fun) {
		outstanding[fd] = fun;
	}
	void complete(int fd, std::string value) {
		auto it = outstanding.find(fd);
		if (it != outstanding.end()) {
			// 这里可能需要一些 Completion function
			auto fun = it->second;
			outstanding.erase(it);
			fun(value);
		}
	}
};

struct async_read {
	// 为了调度，大概需要记录一下 ctx
	io& ctx;
	int fd;
	std::string value;
	// async_read 多数情况下不会 ready
	constexpr bool await_ready() const { return false; }

	// 我们需要在这里决定如何 io，以及如何完成
	void await_suspend(std::coroutine_handle<> h) {
		ctx.submit(fd, [this, h](const std::string& line) {
			value = line;
			h.resume();
		});
	}

	// 为了简单就读 string 了，因为这不是关于 io 的讲解
	constexpr std::string await_resume() {
		return value;
	}
};

task f(io& ctx) {
	// async_read 可以一些 select/epoll/kqueue/io_uring etc.
	// 可能会有 context 来调度，然后还有 fd 用来执行 io
	std::cout << "first: " << co_await async_read{ ctx, 1 } << std::endl;
	std::cout << "second: " << co_await async_read{ ctx, 1 } << std::endl;
}

int main() {
	try {
		io ctx;
		auto t = f(ctx);
		std::cout << "before: ";
		ctx.complete(1, "first line");
		std::cout << "back to main\n";
		ctx.complete(1, "second line");
	}
	catch (const std::exception& ex) {
		std::cout << ex.what() << '\n';
	}
	return 0;
}
```

我们下一步实现一些更好玩的，让 await_transform 捕获 awaiter

顺便，傻批msvc从这里开始代码是过不了编译的

```cpp
#include <iostream>
#include <coroutine>
#include <unordered_map>
#include <functional>

template<typename V>
struct value_awaiter {
	V value;
	constexpr bool await_ready() { return true; }
	void await_suspend(auto) {}
	V await_resume() { return value; }
};

struct is_awaiter_test {
	struct promise_type {
		constexpr std::suspend_always initial_suspend() const { return {}; }
		constexpr std::suspend_always final_suspend() const noexcept { return {}; }
		void unhandled_exception() {}
		is_awaiter_test get_return_object() { return {}; }
	};
};

// co_await 不能用在不求值语境，所以用了个 lambda
template <typename T>
concept is_awaiter
	= std::is_class_v<T>
	&& requires() { [](T t)->is_awaiter_test { co_await t; }; }
;

struct task {
	struct promise_type {
		std::exception_ptr error{};
		std::suspend_never initial_suspend() { return {}; }
		std::suspend_never final_suspend() noexcept { return {}; }
		task get_return_object() { return {}; }
		void unhandled_exception() { error = std::current_exception(); }
		void return_void() {}

		template<typename A>
			requires is_awaiter<A>
		auto await_transform(A a) { return a; }

		// if you want co_await everything
		template <typename V>
			requires (!is_awaiter<V>)
		auto await_transfrom(V v) { return value_awaiter<V>{ v }; }
	};
};

// 假装是 context
struct io {
	std::unordered_map<int, std::function<void(std::string)>> outstanding;
	void submit(int fd, auto fun) {
		outstanding[fd] = fun;
	}
	void complete(int fd, std::string value) {
		auto it = outstanding.find(fd);
		if (it != outstanding.end()) {
			// 这里可能需要一些 Completion function
			auto fun = it->second;
			outstanding.erase(it);
			fun(value);
		}
	}
};

struct async_read {
	// 为了调度，大概需要记录一下 ctx
	io& ctx;
	int fd;
	std::string value;

	async_read(io& context, int fd) : ctx(context), fd(fd) {}

	// async_read 多数情况下不会 ready
	constexpr bool await_ready() const { return false; }

	// 我们需要在这里决定如何 io，以及如何完成
	void await_suspend(std::coroutine_handle<> h) {
		ctx.submit(fd, [this, h](const std::string& line) {
			value = line;
			h.resume();
        });
	}

	// 为了简单就读 string 了，因为这不是关于 io 的讲解
	constexpr std::string await_resume() {
		return value;
	}
};

int to_be_made_async() {
	return 17;
}

task g(io& ctx) {
	std::cout << "second: " << co_await async_read{ ctx, 1 } << std::endl;
}

task f(io& ctx) {
	// async_read 可以一些 select/epoll/kqueue/io_uring etc.
	// 可能会有 context 来调度，然后还有 fd 用来执行 io
	std::cout << "first: " << co_await async_read{ ctx, 1 } << std::endl;
	std::cout << "value: " << co_await to_be_made_async() << std::endl;
	co_await g(ctx);
}

int main() {
	try {
		io ctx;
		auto t = f(ctx);
		std::cout << "before: ";
		ctx.complete(1, "first line");
		std::cout << "back to main\n";
		ctx.complete(1, "second line");
	}
	catch (const std::exception& ex) {
		std::cout << ex.what() << '\n';
	}
	return 0;
}
```



