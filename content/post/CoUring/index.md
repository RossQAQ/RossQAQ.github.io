---
title: io_uring + coroutine
description: 一起阅读文章，学习 io_uring 以及 协程 以及 多线程如何组合成为强力的武器。
slug: io_uring-coro
date: 2024-03-09 00:00:00+0000
image: 
categories: 
    - io-uring
    - coroutine
    - techs
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

终于让我发现了 io_uring 与协程结合的教程，后面还包括与多线程结合。让我来验证一下我的猜想究竟对不对…

本文是系列文章：

[C++20 Coroutines and io_uring - Part 1/3](https://pabloariasal.github.io/2022/11/12/couring-1/)

[C++20 Coroutines and io_uring - Part 2/3](https://pabloariasal.github.io/2022/11/13/couring-2/)

[C++20 Coroutines and io_uring - Part 3/3](https://pabloariasal.github.io/2022/11/13/couring-3/)

的**翻译与总结**。文章间会使用分割线分开。

关于作者

I’m Pablo, a software engineer living in Munich. Please write me an email if you have feedback or have discovered an error in a post.

-------

# Part 1/3

## P1 前言

在这个系列中，我们会写使用 `io_uring` 和 C++20 协程的读取很多硬盘上的文件的程序。主要的目的是因为，我发现有很多关于 `io_uring` 和 C++20 协程的单独资料，也有讲的比较深入的，但是基本没有展示如何结合二者的。我们会揭开异步 IO 和 协程结合的谜底，就像黄油和面包一样。

这个系列会分成三部分，P1 我们首先仅用 `io_uring` 来解决问题。P2 我们会重构实现，并且加入协程。P3 我们会使用多线程优化，这才能展现出协程的真正力量。

## Async I/O ❤ Coroutines

异步 IO，跟同步 IO 相反，描述一种**不阻塞当前 calling 线程**的 IO 操作。跟等待操作完成不同，当前线程会立刻 release，然后执行其他操作，而此时后台正在执行你请求的 IO 操作。待一段时间后，calling 线程（或者其他的线程）可以回来并且收集请求操作的结果。这就好像你对一个 pizza guy 说：“将 margherita 放到烤箱，我去药房取药，一会回来。”

异步 IO 可以比同步 IO 更有效率，线程不需要等待资源可用，但同时程序也会变得更加复杂。程序需要异步工作，必须记得回来取走他们点的 margherita。

**协程允许我们以同步的形式编写异步代码**，如果你使用协程来从硬盘读取一个文件，它可以挂起自己然后将控制流返回给 caller，此时文件正在后台读取，等数据读取完后恢复协程，然后获取数据。所有的代码写起来就和 good-old 同步调用类似。

结合异步 IO 和协程，允许我们编写异步程序，但并不用写繁琐的异步代码。取了二者的长处。

## 目标

这个系列中，我们会写一个程序，从硬盘中读取并且 parse 几百个 [wavefront OBJ](https://en.wikipedia.org/wiki/Wavefront_.obj_file) 文件。

了解读取和 parsing 的区别很重要。读取代表**把文件从内存中装载到内存中**。parsing 代表**从内存中取数据，然后把它翻译为应用程序可以理解的结构**。

OBJ 文件使用 ASCII 编码，描述的是 3D 三角形组成的网格图形。该文件对构成三角形的网络、顶点、颜色等信息编码。parsing 一个 OBJ 意味着将 ASCII 表现形式转换为有更方便访问网格属性 API 的  C++ 对象形式。

> 玩过 3dmax 这种建模软件的应该都知道 .obj 吧。不是你程序编译中间生成的目标文件。是记录 3d 图形的。

为了 parsing obj 文件，我们使用一个三方库 [tinyobjloader](https://github.com/tinyobjloader/tinyobjloader)。它接受一个 string，然后把它 parse 成 `ObjReader` 对象。

```cpp
std::string obj_data = ...;	// read from obj files
tinyobj::ObjReader reader;
reader.ParseFromString(obj_data);
```

我们可以使用 `reader` API 来访问图形的属性，例如，列出有多少个多边形：

```cpp
std::cout << reader.GetShapes().size() << '\n';
```

## 地基

首先定义一些抽象，来让我们的实现更简单。

我们要读取文件，所以写一些 RAII 类来管理只读文件：

```cpp
class ReadOnlyFile {
public:
  ReadOnlyFile(const std::string &file_path) : path_{file_path} {
    fd_ = open(file_path.c_str(), O_RDONLY);
    if (fd_ < 0) {
      throw std::runtime_error("Fail to open file");
    }
    size_ = get_file_size(fd_);
    if (size_ < 0) {
      throw std::runtime_error("Fail to get size of file");
    }
  }

  ReadOnlyFile(ReadOnlyFile &&other)
      : path_{std::exchange(other.path_, {})},
        fd_{std::exchange(other.fd_, -1)},
        size_{other.size()} {}

  ~ReadOnlyFile() {
    if (fd_) {
      close(fd_);
    }
  }

  int fd() const { return fd_; }
  off_t size() const { return get_file_size(fd_); }
  const std::string &path() const { return path_; }

private:
  std::string path_;
  int fd_;
  off_t size_;
};
```

非常简单，只读模式打开文件并且析构时关闭文件。实现了一些比较笨的错误处理，以及一个移动构造，这样该类型就可以存在于 `std::vector` 之类的容器了。

> 这里作者似乎也忘了 noexcept 了。

另一个类型是 `Result` ，我们会经常使用：

```cpp
struct Result {
  tinyobj::ObjReader result; // 存储解析后的对象
  int status_code{0};        // 读操作的状态
  std::string file;          // OBJ 的路径
};
```

我们的程序最终要分析一系列的 OBJ 文件，然后返回 `std::vector<Result>`

## 初次尝试：平凡的实现

该到正餐部分了。像往常一样，我们来实现一个最简单的版本：**单线程阻塞读取**

```cpp
Result readSynchronous(const ReadOnlyFile &file) {
  Result result{.file = file.path()};
  std::vector<char> buff(file.size());
  read(file.fd(), buff.data(), buff.size()); // 完成前会阻塞
  readObjFromBuffer(buff, result.result);
  return result;
}
```

`readSynchronous` 接收文件，把它的内容读取到 buffer 中，之后将 buffer 中的内容解析为 obj 对象。`readObjFromBuffer` 包装了一个简单的实现，并且初始化 `Result` 的 `result` 成员：

```cpp
void readObjFromBuffer(const std::vector<char> &buff, tinyobj::ObjReader &reader) {
  auto s = std::string(buff.data(), buff.size());
  reader.ParseFromString(s, std::string{});
}
```

可惜，`tinyobjloader` 不支持 `std::string_view`，所以我们只能拷贝一次 buffer 了。可能之后我会提 PR。

现在我们需要做的是对每个文件调用 `readSynchronous`：

```cpp
std::vector<Result> trivialApproach(const std::vector<ReadOnlyFile> &files) {
  std::vector<Result> results;
  results.reserve(files.size());
  for (const auto &file : files) {
    results.push_back(readSynchronous(file));
  }
  return results;
}
```

太简单了，但是非常慢慢慢慢。`read` 系统调用会阻塞 calling 线程，直到读取完所有的数据。我意思是 *thread*，只有一个线程做 IO 然后解析所有文件。**我们不能在下一个文件读取完毕之前做上一个读取好的文件的解析！**

用户态和内核态的上下文切换也很费时间，每次调用 `read` 都要切换到内核态。如果我们读取几百个文件，就得切换几百次。

我们可以做的更好一些。

## 下一步：线程池

我知道你想说什么，并行！

```cpp
std::vector<Result> threadPool(const std::vector<ReadOnlyFile> &files) {
  std::vector<Result> result(files.size());
  BS::thread_pool pool;
  pool.parallelize_loop(files.size(),
                        [&files, &result](int a, int b) {
                          for (int i = a; i < b; ++i) {
                            result[i] = readSynchronous(files[i]);
                          }
                        })
      .wait();
  return result;
}
```

这里使用了 [bshoshany’s thread-pool library](https://github.com/bshoshany/thread-pool) 来并行执行独立的循环。每个线程都执行固定次数的迭代，记为范围 [a, b)。你也可以用 openMP 之类的库，或者 `std::async`，思想都一样。

>  补充：对于不了解这个库的人，这个函数大概就是给 size 分块，线程池里的线程执行固定数量的 IO。

这就好多了，即使线程仍然会在 `read` 上阻塞，但这次文件会并行处理。代码的更改也很少，然后他还有了更多的优化机会：我们可以给每个线程分配一个 buffer，然后可以被多个文件复用。

对于大多数程序来说这个已经很效率了，但是想象如果你是 web server 的开发者，一次监听上千个 socket。你会给每个 socket 创建一个线程吗？估计不会。你要做的是告诉操作系统：“听好，我对这些 sockets 感兴趣，当他们有可以被读取的数据时告诉我，我会继续我的工作。” 你需要的是异步 IO。

## 使用 io_uring

linux kernel 5.1 后推出了新的异步 IO API，io_uring。以前通常会使用 `epoll`, `poll`, `select`, `aio` 等等，它们各有各的限制和问题。`io_uring` 的目标是使用标准 API 给内核中的所有异步 IO 操作打开一个新的篇章。

API 叫做 `io_uring` 因为它基于两个 buffer：submission queue（SQ） 以及 completion queue（CQ）。buffer 在内核和用户之间共享，从它们中读取/写入数据不需要任何 syscall 或者拷贝。

中心思想很简单：用户代码编写提交给 SQ 的请求，之后把它们提交给内核。内核消耗队列中的请求，执行请求的操作，然后将结果写入 CQ。用户代码可以异步地在之后的某个时间点获取 CQ 中完成后的请求。

`io_uring` 原生 API 非常复杂，所以应用程序通常会使用库 `liburing`（`io_uring` 的作者帮你封装的），它提取了很多重复的操作，为 `io_uring` 的使用提供了方便的工具。

## 使用 `liburing` 解析 OBJs

我们现在使用 `liburing` 来完成实现

首先先封装一个 RAII 类，初始化 io_uring 对象并且释放：

```cpp
class IOUring {
public:
    explicit IOUring(size_t queue_size) {
        if (auto s = io_uring_queue_init(queue_size, &ring_, 0); s < 0) {
            throw std::runtime_error("error initializing io_uring: " + std::to_string(s));
        }
    }
    IOUring(const IOUring &) = delete;
    IOUring &operator=(const IOUring &) = delete;
    IOUring(IOUring &&) = delete;
    IOUring &operator=(IOUring &&) = delete;
    ~IOUring() { io_uring_queue_exit(&ring_); }
    
    struct io_uring* get() {
        return &ring_;
    }
private:
    struct io_uring ring_;
};
```

`io_uring_queue_init()` 初始化 io_uring 实例，使用 `queue_size` 的长度（这是 SQ 和 CQ 的环形缓冲区长度）。`io_uring_queue_exit()` 销毁 io_uring 实例。

> 这里作者应该说错了，CQ 长度默认是 SQ 的二倍。想要具体指定你可以使用 `io_uring_params` 传给 `io_uring_init_params()` 来初始化 `io_uring`

现在我们尝试使用 `liburing` 来实现 OBJ Loader

实现必须包含两个部分：首先我们提交读请求给 SQ，之后我们等待完成请求进入到 CQ 中再解析 buffer 中的内容。

```cpp
std::vector<Result> iouringOBJLoader(const std::vector<ReadOnlyFile>& files) {
    IOUring ring{files.size()};
    auto buf = initializeBuffers(files);
    pushEntriesToSubmissionQueue(files, buf, uring);
    return readEntriesFromCompletionQueue(files, buf, uring);
}
```

我们创建足够大的 io_uring 实例来容纳下所有的文件的请求。之后分配 buffer，每个文件一个。

`pushEntriesToSubmissionQueue()` 内编写 submission entries 提交给 SQ：

```cpp
void pushEntriesToSubmissionQueue(const std::vector<ReadOnlyFile> &files,
                                  const std::vector<std::vector<char>> &buffs,
                                  IOUring &uring) {
  for (size_t i = 0; i < files.size(); ++i) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(uring.get());
    io_uring_prep_read(sqe, files[i].fd(), buffs[i].data(), buffs[i].size(), 0);
    io_uring_sqe_set_data64(sqe, i);
  }
}
```

`io_uring_get_sqe()` 创建一个 SQ 的 entry，`sqe`。我们现在可以使用 `io_uring_prep_read()` 来设置 entry 的内容，指定内核读取 `files[i].fd()` 的文件给 buffer `buffs[i]`。

可以通过使用 `io_uring_sqe_set_data()` 来给 entry 追加**用户数据**。**kernel 不会使用这个部分的数据，只是单纯的拷贝给当前提交请求对应的 completion entry**。这个很重要，可以让我们区分哪个 completion entry 对应的是哪个 submission entry。在这个情况下，我们只是单纯写文件的 index，也可以用来区分。

在外面把所有的 SQ 都写入队列后，需要将他们提交给内核，并且等待他们出现在 CQ 中。一旦出现了 completion entry，我们就读取对应 OBJ 文件的 buffer。

```cpp
std::vector<Result> readEntriesFromCompletionQueue(const std::vector<ReadOnlyFile> &files,
                                const std::vector<std::vector<char>> &buffs,
                                IOUring &uring) {
  std::vector<Result> results;
  results.reserve(files.size());

  while (results.size() < files.size()) {
    io_uring_submit_and_wait(uring.get(), 1);
    io_uring_cqe *cqe;
    unsigned head;
    int processed{0};
    io_uring_for_each_cqe(uring.get(), head, cqe) {
      auto id = io_uring_cqe_get_data64(cqe);
      results.push_back({.status_code = cqe->res, .file = files[id].path()});
      if (results.back().status_code) {
        readObjFromBuffer(buffs[id], results.back().result);
      }
      ++processed;
    }

    io_uring_cq_advance(uring.get(), processed);
  }
  return results;
}
```

首先我们调用 `io_uring_submit_and_wait()`，来提交所有的 entry 给内核，并且阻塞，直到等待第一个 completion entry 出现。

我们获得 completion entry 后就可以处理他们了。`io_uring_for_each_cqe()` 是一个定义在 `liburing` 内的宏，其意义是对 CQ 中的所有 completion entry 执行操作。

以下是我们需要在 completion entry 到达时执行的操作：

1. 获取 completion entry 对应的文件的 id。这和我们写给 submission entry 的 id 是同一个。
2. 将 status code 写入 `Result` 对象，当前状态是内核执行的 `read` 操作。
3. 如果读取成功，从 buffer 中解析 OBJ 文件，放入 `Result` 对象。

最后，我们可以释放一些空间，因为我们已经完成了一些 completion entry 的处理。我们使用 `io_uring_cq_advance()` ，这个函数唯一做的事情就是把 ring buffer 的头向后移动 n 个位置，使得有足够的空间来存放 entries。

## p1 结束语

使用 io_uring 最大的优点就是实现的代码可以减少很多 syscall。实际上，使用 `strace` 可以看到 `io_uring` 的实现比同步代码的实现少了 512 次 syscall。这主要是因为 `read` 的 syscall 的减少：

```
> strace -c -e read -- ./build_release/couring --trivial
Running trivial
Processed 512 files.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000591           1       517           read

> strace -c -e read -- ./build_release/couring --iouring
Running iouring
Processed 512 files.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000053          10         5           read
```

不同于对每个文件都调用 `read`，`io_uring` 的做法不会有 syscall，因为向 `io_uring` 的队列读写并不是 syscall，也不会有上下文切换。

我们的 `io_uring` 实现仍然有问题。首先，代码比同步版本明显复杂很多，我们并不能在一个函数内完成读取和解析，而是在一个程序内将请求写入队列然后轮询。这很难扩展。

其次，我们的程序是 CPU-bound，如下所示 (gprof)，大部分时间在解析 obj：

```cpp
  %   cumulative   self
 time   seconds   seconds
 31.25      0.05     0.05    tinyobj::tryParseDouble(char const*, char const*, double*)
 18.75      0.08     0.03    tinyobj::LoadObj(tinyobj::attrib_t*, std::vector<tinyobj::shape_t, std::allocator<tinyobj::shape_t> >*, std::vector<tinyobj::material_t, std::allocator<tinyobj::material_t> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*, std::istream*, tinyobj::MaterialReader*, bool, bool)
 12.50      0.10     0.02    allDone(std::vector<Task, std::allocator<Task> > const&)
 12.50      0.12     0.02    tinyobj::parseReal(char const**, double)
  6.25      0.13     0.01    tinyobj::parseReal3(float*, float*, float*, char const**, double, double, double)
  ...
```

解析 obj 仍然单线程。处理 SQE 是内核的线程池完成，而消费 CQE 则是用户空间的单线程完成。显然我们必须并行解析。

---

# Part 2/3

## P2 前言

这一部分我们会使用 C++20 的协程来重写第一部分的程序，读取 OBJ 之后解析。本文的目标是结合 `io_uring` 与协程，不懂 C++ 协程的可以自己看  [Lewis Baker’s blog series](https://lewissbaker.github.io/)。

> 本博客内还有其他优秀文章的翻译。Lewiss Baker 的也很不错。

## 中心思想

我们要做的很简单：实现一个协程，读取并且解析硬盘中的 OBJ 文件，底层 IO 使用 `io_uring`。当协程被调用时，它把请求提交给 SQ，然后暂停执行，返回控制权给 caller。一旦相应的 completion entry 完成，那么协程就恢复，OBJ 可以被解析了。

> woc，我感觉我是天才，自己悟出来的和文章里想的是一样的。之前悟了几个月。

```cpp
Task parseOBJFile(IOUring& uring, const ReadOnlyFile& file) {
    std::vector<char> buff(file.size());
    auto status = co_await ReadFileAwaitable{uring, file, buff};
    // 此时 completion entry 已经准备好,
    // buff 内也填充了数据，可以解析了
    Result result{ .status_code = 0, .file = file.path() };
    readObjFromBuffer(buff, result.result);
    co_return result;
}
```

这是我们的协程。看起来和普通函数很像，实际上，他确实就是按照传统的同步代码模式编写：分配缓冲，读取内容，解析 buffer，返回结果。

协程天然就不是同步执行的：它在特定位置暂停执行并且让出控制权给 caller。例如，await 表达式：`co_await ReadFileAwaitable{uring, file, buff}`。这里，我们提交 SQE，并且将控制流返回给 caller。`co_await` 是一个运算符，需要一个 *awaitable* 类型，然后暂停执行。`ReadFileAwaitable` 就是那么一个 awaitable，它的任务是暂停写成然后注册它，准备以后唤醒。一旦协程恢复了执行，那么就从 `co_await` 的下一行开始执行。

## 暂停执行

`ReadFileAwaitable` 是一个用来暂停协程的 awaitable。我们会解释发生了什么：

```cpp
class ReadFileAwaitable {
public:
    ReadFileAwaitable(IOUring& ring, cont ReadOnlyFile& file, const std::vector<char>& buf) {
        sqe_ = io_uring_get_sqe(uring.get());
        io_uring_prep_read(sqe_, file.fd(), buf.data(), buf.size(), 0);
    }
    
    auto operator co_await() {
        struct Awaiter {
          	io_uring_sqe* entry;
            RequestData requestData;
            
            Awaitable(io_uring_sqe* sqe) : entry{sqe} {}
            bool await_ready() {return false;}
            void await_suspend(std::coroutine_handle<> h) noexcept {
                requestData.handle = h;
                io_uring_sqe_set_data(entry, &requestData);
            }
            int await_resume() {return requestData.statusCode;}
        };
        return Awaiter{sqe_};
    }
private:
    io_uring_sqe* sqe_;
};
```

`ReadFileAwaitable` 创建一个 SQE 然后把它提交。我们必须在协程暂停之前提交我们的请求。

> 我从来没重载过 operator co_await() 只在文档里看过，感觉还是写 await_transform() 好点。

`Awaiter` 有两个成员变量：`entry` and `requestData`，`entry` 是一个指针，指向 SQE，我们需要他来被传递给 `io_uring_sqe_set_data()` 以此绑定用户数据。用户数据是任意的（`void*`）所以我们可以绑定我们的提交请求。用户数据会原样传递给内核，并由内核传递给 CQE，主要用来分辨不同的 CQE，将其链接到正确的 SQE。

`ReadFileAwaitable` 重载 `operator co_await()` 返回一个 *awaiter*：`Awaiter`. `Awaiter` 实现了标准要求的三个函数：`await_ready()`, `await_suspend()`, `await_resume()`。

`await_ready()` 一直返回 `false`。代表我们的协程永远会在 `co_await ReadFileAwaitable{}` 处暂停。我们也可以继承自 `std::suspend_always`

现在我们实现 `await_suspend()`，这个函数会在协程暂停后调用，并且**参数是协程句柄**。这是一个很好的函数，我们可以用它将用户数据写入 SQE。但用户数据应该写啥呢？

我们仔细想想：我们提交 SQE。会有一个对应的 CQE 在完成时进入 CQ。一旦 CQE 到达，那么我们就必须恢复我们暂停的协程。我们怎么恢复协程呢？协程句柄已经传递给我们了！我们把写成句柄写入用户数据作为提交请求！

> 我真是天才，几天前就自己悟了。

协程句柄作为成员变量，存储在 `RequestData`:

```cpp
struct RequestData {
  std::coroutine_handle<> handle;
  int statusCode{-1};
};
```

`RequestData` 存储协程句柄和状态码。状态码之后会被写入 `Awaiter` 对象，当完成请求到达时，用户数据则是指向 awaiter 对象的 `requestData` 数据成员的指针。

最终，我们实现 `await_resume()` 。`await_resume()` 会在协程恢复时立刻调用，并且返回写操作的状态码。换句话说，我们可以假设 `requestData.statuCode` 在 `await_resume()` 调用时初始化。

**`await_resume()` 的返回值就是整个 `co_await` 表达式的结果**，我们可以写：

```cpp
int status = co_await ReadFileAwaitable{};
if (!status) {
    //...
}
```

就像我们调用了一个普通的 `read`。

## 恢复执行

调用 `co_await ReadFileAwaitable` 会让协程挂起，但我们怎么唤醒它呢？简单，**我们等待 CQE，然后从中取出协程句柄即可。**

```cpp
int consumeCQEntries(IOUring &uring) {
  int processed{0};
  io_uring_cqe *cqe;
  unsigned head; // 环形缓冲区头位置，未被使用
  io_uring_for_each_cqe(uring.get(), head, cqe) {
    auto *request_data = static_cast<RequestData *>(io_uring_cqe_get_data(cqe));
    // 记得在恢复协程前设定 statusCode
    request_data->statusCode = cqe->res;
    request_data->handle.resume(); // 这里恢复协程然后自动调用 await_resume()
    ++processed;
  }
  io_uring_cq_advance(uring.get(), processed);
  return processed;
}
```

在恢复协程前，我们必须在 `request_data` 中写入 status code。`request_data` 是一个指向 `Awaiter` 对象中 `reqeustData` 数据成员的指针。

我们现在写一个 `consumeCQEntriesBlocking()` 帮助函数，会向内核提交 SQE，然后阻塞直到至少有一个 CQE 完成。

```cpp
int consumeCQEntriesBlocking(IOUring &uring) {
  io_uring_submit_and_wait(uring.get(), 1); // CQ 为空会阻塞
  return consumeCQEntries(uring);
}
```

我们已经学会了暂停和恢复协程的机制，现在我们可以写客户端代码，来读取 OBJ 文件了。

直观上，我们必须使用入 `std::vector` 来包含 `parseOBJFile` 返回的结果，但是 `parseOBJFile` 的返回值是什么？协程的返回类型是什么？它是一个 *coroutine type* ，这里命名为 `Task`。

## Coroutine Type：Task

`Task` 是我们协程的返回类型。我们必须自己实现它，实现标准规定的 API。

```cpp
class Task {
public:
  struct promise_type {
    Result result;

    Task get_return_object() { return Task(this); }

    void unhandled_exception() noexcept {}

    void return_value(Result result) noexcept { result = std::move(result); }
    std::suspend_never initial_suspend() noexcept { return {}; }
    std::suspend_always final_suspend() noexcept { return {}; }
  };

  explicit Task(promise_type *promise)
      : handle_{HandleT::from_promise(*promise)} {}
  Task(Task &&other) : handle_{std::exchange(other.handle_, nullptr)} {}

  ~Task() {
    if (handle_) {
      handle_.destroy();
    }
  }

  Result getResult() const & {
    assert(handle_.done());
    return handle_.promise().result;
  }

  Result&& getResult() && {
    assert(handle_.done());
    return std::move(handle_.promise().result);
  }

  bool done() const { return handle_.done(); }

  using HandleT = std::coroutine_handle<promise_type>;
  HandleT handle_;
};
```

`Task` 定义了 promise type，每个协程都拥有 promise 对象，其位于协程帧中。promise 对象用于传输协程的数据（或者重新抛出协程抛出的异常）。此外，promise type 有一个成员 `Result result`，包括了最终的解析 OBJ 文件的结果。

```cpp 
struct Result {
	tinyobj::ObjReader result; // stores the actual parsed obj
  	int status_code{0};        // the status code of the read operation
  	std::string file;          // the file the OBJ was loaded from
};
```

`result` 通过内部的成员函数 `return_value` 初始化，当 `co_return` 执行时会被调用。

`Task` 定义了一个成员函数 `getResult()` 来方便的从 promise 对象中得到返回的结果。

`promise_type` 必须定义成员函数 `get_return_object()` 返回实际的协程对象。在我们的示例中，是 `Task` 实例。

`unhandled_expection()` 在协程体抛异常时会被调用，我们暂未实现因为我们是 exception free （或者目标是）。`initial_suspend` 和 `final_suspend` 决定了协程的初始和最终行为，协程是否在开始和结束时暂停。

`Task` 包含了协程句柄 `handle_` 并且管理其生命周期：它会通过析构时调用 `handle_.destroy()` 来销毁协程帧。同时还定义了 `done()` 成员函数表明协程是否执行完毕。

C++20 协程是 raw，代表他并不是一个完整的 cake，而是一堆 flour，eggs 和 butter。为了实现一个协程你必须写一些支持代码以及模版，自己来烤蛋糕。因为这些原因，可以使用一些库，比如 [cppcoro by Lewis Baker](https://github.com/lewissbaker/cppcoro)，实现了泛型版本的 `Task` 类型，使用了很多有用的抽象，大量减少了模板代码。

## 结合

现在我们实现顶层函数，使用协程解析 OBJ：

```cpp
std::vector<Result> parseOBJFiles(const std::vector<ReadOnlyFile> &files) {
  IOUring uring{files.size()};
  std::vector<Task> tasks;
  tasks.reserve(files.size());
  for (const auto &file : files) {
    tasks.push_back(parseOBJFile(uring, file));
  }
  while (!allDone(tasks)) {
    // consume all entries in the submission queue
    // if the queue is empty block until the next completion arrives
    consumeCQEntriesBlocking(uring);
  }
  return gatherResults(tasks);
}
```

通过执行 `parseOBJFile` 分配了一个 vector 的 `Task` 协程。注意 `initial_suspend()` 返回的是 `std::suspend_never` ，代表 `parseOJBFile` 协程在协程开始时永远不会暂停，直到执行到了 `co_await ReadFileAwaitable`，协程才会暂停。

一旦协程暂停，内核就会做它的工作，我们什么都不用做，只需要等到 CQE 完成，`consumeCQEntriesBlocking` 会唤醒协程，因为它们对应的 CQE 已经完成了。

`allDone` 是一个简单的帮助函数，检查是否所有的协程都已经执行完毕。

```cpp
bool allDone(const std::vector<Task> &tasks) {
  return std::all_of(tasks.cbegin(), tasks.cend(),
                     [](const auto &t) { return t.done(); });
}
```

然后协程恢复，解析 OBJ 文件，并且通过 `co_return result` 返回结果。

最后我们可以从完成的协程中获取最终的结果：

```cpp
std::vector<Result> gatherResults(const std::vector<Task> &tasks) {
  std::vector<Result> results;
  results.reserve(tasks.size());
  for (auto &&t : tasks) {
    results.push_back(std::move(t).getResult());
  }
  return results;
}
```

在函数块最后，所有的 `Task` 都会销毁，解分配所有的协程帧。

## p2 结束语

你可能有问题：这篇文章的实现和不使用协程实现关键区别在哪？这是一个争议性的问题，有的人可能觉得我们只是像程序中加入了一些模板式的代码来实现已经实现过的功能。也没有更加效率：**我们仍然是单线程串行解析。**

好在我们还没有完成所有的操作。协程的魅力之处在于其可以组合其他功能，在实现了一个基础的协程设施后，添加更多的 awaitable 和 其他协程很简单。

在 P3 我们会扩展实现，例如使用线程池并行解析文件。这才是协程最终的魔力。

# Part 3/3

## P3 前言

已经到了系列的最后一篇文章。在 P2 我们写了基于协程的程序，读取并且解析 OBJ 文件，使用协程和 `io_uring`。程序仍然有最后一个缺点：他是 CPU-bound。解析文件，最耗费时间的部分是算法，是在单线程上串行执行的。

问题的根源是我们的协程在 main 线程上恢复。理想中我们希望他在其他线程上恢复，这样才可以并行解析文件。

这正是我们这篇文章要做的。我们添加第二个 `await` 表达式 `co_await pool.schedule()`，这会让我们的协程暂停并且被线程池调度和恢复。

```cpp
Task parseOBJFile(IOUring &uring, const ReadOnlyFile &file, ThreadPool &pool) {
  std::vector<char> buff(file.size());
  int status = co_await ReadFileAwaitable(uring, file, buff);
  co_await pool.schedule();
  // This is now running on a worker thread
  Result result{.status_code = 0, .file = file.path()};
  readObjFromBuffer(buff, result.result);
  co_return result;
}
```

## 线程池

`ThreadPool` 实现了我们的核心思想。

`ThreadPool` 封装了  [bshoshany Thread Pool’s object](https://github.com/bshoshany/thread-pool) ，它的 API 很简单，你可以使用 `push_task` 来调度一个在线程池上跑的任务。

```cpp
class ThreadPool {
public:
  auto schedule() {
    struct Awaiter : std::suspend_always {
      BS::thread_pool &tpool;
      Awaitable(BS::thread_pool &pool) : tpool{pool} {}
      void await_suspend(std::coroutine_handle<> handle) {
        tpool.push_task([handle, this]() { handle.resume(); });
      }
    };
    return Awaiter{pool_};
  }

  size_t numUnfinishedTasks() const { return pool_.get_tasks_total(); }

private:
  BS::thread_pool pool_;
};
```

`ThreadPool` 定义了成员函数 `schedule()` ，返回一个 `Awaiter` 的实例。当协程 `co_await Awaiter` 对象时，协程会暂停（注意 `Awaiter` 继承自 `std::suspend_always`）并且在 `await_suspend()` 中调度在 worker 线程上恢复。

很棒！通过简单的写 `co_await pool.schedule()` 我们就可以在线程池中唤醒当前协程，大大提高了我们的执行效率。并行解析 OBJ。

## 多线程实现

以上。现在我们来实现顶层的函数，使用新的协程来加载和解析 OBJ 文件。

```cpp
std::vector<Result> coroutinesThreadPool(const std::vector<ReadOnlyFile> &files) {
  IOUring uring{files.size()};
  ThreadPool pool;
  std::vector<Task> tasks;
  tasks.reserve(files.size());
  for (const auto &file : files) {
    tasks.push_back(parseOBJFile(uring, file, pool));
  }
  io_uring_submit(uring.get());
  while (pool.numUnfinishedTasks() > 0 || !allDone(tasks)) {
    // consume entries in the completion queue
    // return immediately if the queue is empty
    consumeCQEntriesNonBlocking(uring);
  }

  return gatherResults(tasks);
}
```

我们初始化一个线程池和 `io_uring` 实例，然后对每个文件调用 `parseOBJFile` ，这会填满 SQ，之后我们使用 `io_uring_sumbit()` 向内核提交请求。

一旦内核在后台读取文件，协程会在相应的 CQE 到达时被唤醒。这在函数 `consumeCQEntriesNonBlocking()` 中执行：

```cpp
int consumeCQEntriesNonBlocking(IOUring &uring) {
  io_uring_cqe *temp;
  if (io_uring_peek_cqe(uring.get(), &temp) == 0) {
    return consumeCQEntries(uring);
  }
  return 0;
}
```

使用 `io_uring_peek_cqe()` 来查看 CQ 中是否已经有 CQE，如果为空的话则会退出。

我们继续等待所有协程完成。因为 `allDone` 是线性检查，所以我们给一个短路的选项避免一直调用它、如果在线程池中有未完成的任务，那么显然我们还没有完成。

## P3 结束语

大功告成！希望你能感受到协程的 cool 了，一旦你已经有了一个协程，那么添加其他的 `co_awaits` 易如反掌。

你可以在 [这里](https://github.com/pabloariasal/couring) 找到所有的代码。

