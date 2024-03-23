---
title: 网络编程迷思
description: 崩溃中
slug: io_uring-and-network
date: 2024-03-05 00:00:00+0000
image: 
categories:
    - io-uring
    - cppnotes
    - techs
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

## 前言

该文章中的一切，不出意外的话会在我个人的服务器测试，

系统是 fedora 39，linux 内核版本 6.5.6，g++ 13.2，liburing 是最新版。

## io_uring echo-server

> 写于 2024.3.5，此时本人对 io_uring 刚刚懂 API，也不完全懂，网络编程略懂略懂，也许以后自己给自己 code review 吧hhh。
>
> 想了半天，先试试阻塞的 socket 以及 单发的 accept 吧，最简单的开始，且 SQ 大小也足够大，顺便使用最简单的 switch 。
>
> 第一个问题就是，socket 到底用不用非阻塞，参考了一些文章，确实已经异步了，是内核帮你执行，那你阻塞不阻塞有什么区别。此外旧版本 io_uring 似乎对 NONBLOCK socket 还有 bug，因为非阻塞，所以会立刻返回，也会被当做完成事件，确实蛋疼。于是我决定都写阻塞 socket。
>
> io_uring 自 5.7 之后支持 provided buffer，5.19 之后支持 ring mapped buffer，我决定直接使用，就不用自己思考 buffer 的问题了。确实，性能又高，思维压力又低；buffer 自己写起来估计很折磨，设计思路可以参考 陈硕的圣经。
>
> 但我一想，如果以后写成服务器，那么全局使用 buffer 这种显然更麻烦，好像还不如自己设计 buffer

> 现在是 2024.3.20 学了一堆，感觉会写了，协程怎么设计也有点思路了。
>
> 首先好像似乎得用非阻塞 socket？但我决定用阻塞的看看会发生什么，其次我发现想写个没有内存泄漏的还真挺难。

## 关于协程 Task

1. Task 应该为惰性，这个很简单，promise 直接 initial_suspend return std::suspend_always

2. 本身可以被 co_await。需要 Task 本身是个 awaiter，可以在 Task 中重载 co_await 运算符，然后在 Task 内写一个结构体，通过 co_await 返回。

3. Task 本身被 co_await 后，利用 await_suspend 进行灵活调度，将控制权返回给 caller/最开始的函数，那么就需要保存 caller 的句柄。这个其实调用 co_await 的 caller 会自动传入 handle 作为参数。**区分好 caller 和 callee**
4. 返回的值可以用 std::optional，如果发生异常，需要存入 std::exception_ptr 然后在 unhandled_exception 重新抛出
5. 考虑 std::variant 替代 std::optional，存储正常的返回值，和 std::exception_ptr，可以结合 emplace
6. Task 和 Promise 都定义为泛型，这样就可以支持任意的返回类型了。对 void 进行特化。注意待决名
7. 支持 sleep 操作，传统 sleep 会阻塞，协程 sleep 显然可以干别的，比如支持个超时。

但显然不能真睡，需要个调度器，在一堆协程里面恢复其他的协程。

那么调度器，最简单的实现就是，队列，协程全部默认加入调度器。

Task 可以写成隐式转换为 coro handle，这样放入调度器比较方便

睡的协程要加入到调度器的存储结构，时间+句柄。

可以搞一个小顶堆

之后，Task 可以支持 `when_all` ，支持参数内的 Task 一起执行。如果有返回值，可以用 tuple+结构化绑定接返回值。配合模板，实现任意数量的参数

之后，可以实现 `when_any`

## io_uring 使用思路

1. 声明 io_uring 
2. 初始化：io_uring_queue_init
3. 声明 sqe，
4. 获取 sqe：io_uring_get_sqe
5. 准备操作：io_uring_prep_foo
6. 绑定 data：io_uring_sqe_set_data
7. submit
8. wait/peek CQE
9. 使用 io_uring_cqe_get_data 获取数据
10. 检查 cqe -> res 错误码
11. io_uring_cqe_seen 释放 cqe
12. io_uring_queue_exit()

## io_uring 常用函数 reference

### io_uring_queue_init

```cpp
int io_uring_queue_init(unsigned entries,
                        struct io_uring *ring,
                        unsigned flags);
```

该函数调用 syscall `io_uring_setup` 来初始化 SQ 以及 CQ，SQ 的 entry 数量至少为参数 `entries`，之后会 mmap 结果 fd 到应用和 kernel 的共享内存。

返回值：0成功，并且 参数 `ring` 会指向共享内存；-errno 失败。

CQ 默认大小是 SQ 的两倍，对于文件 io 肯定足够，但是对于网络不一定。SQ 的 entries 数目**仅仅是限制一次提交给内核的 entry 数目**，并不是限制 uring 支持的正在进行请求数。

如果 CQ ring 溢出，也就是说应用程序的处理能力跟不上 CQ ring 了，那么如果内核支持 `IORING_FEAT_NODROP` 那么 ring 会进入 CQ 溢出状态。否则会扔掉 CQEs 然后增加 `io_uring` 中 `cq.koverflow` 的大小（按照扔掉 CQEs 的数量增加）。溢出状态通过 SQ ring 的 flag `IORING_SQ_CQ_OVERFLOW` 指定，除非内核没有内存用了，不然不会丢弃 entry。**但你应该避免其溢出，可以使用 `io_uring_params` 中的 `cq_entries` 指定 CQ entry 数量，会自动近似为 2 的整幂次数。**

flags 和 syscall `io_uring_setup` 相同，详见 [io_uring_setup(2)](https://man.archlinux.org/man/io_uring_setup.2.en)

---

### io_uring_get_sqe

```cpp
struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);
```

该函数会从 `ring` 中获取 SQ 的下一个可用的 SQE

返回值：成功返回指向 SQE 的指针；失败返回 NULL。

如果该函数执行成功，那么下一步就是使用 io_uring_prep_foo() 函数对 SQE 进行操作，之后提交。

注意，不管是 `io_uring_get_sqe()` 还是 `io_uring_prep_foo()` 都不会设置（或清空） SQE 的 **user_data** 字段。所以 caller **必须在提交前调用 `io_uring_sqe_set_data()` 函数**。

如果该函数返回 NULL，则 SQ ring 已满，必须提交进行处理，之后才能分配新的 entry。

---

### io_uring_prep_read

```cpp
void io_uring_prep_read(struct io_uring_sqe *sqe,
                        int fd,
                        void *buf,
                        unsigned nbytes,
                        __u64 offset);
```

io_uring_prep_foo 系列都是准备 IO 请求。

CQE 的 *res* 字段会包括操作的结果，io_uring 永远不会使用 errno，都是使用 *-errno*，存于 CQE 的 *res* 字段。

----

### io_uring_sqe_set_data/io_uring_cqe_get_data

```cpp
void io_uring_sqe_set_data(struct io_uring_sqe *sqe,
                           void *user_data);
void io_uring_sqe_set_data64(struct io_uring_sqe *sqe,
                             __u64 data);

void *io_uring_cqe_get_data(struct io_uring_cqe *cqe);
__u64 io_uring_cqe_get_data64(struct io_uring_cqe *cqe);
```

该函数存储一个含有 SQE 的指针 *user_data*

在用户使用过 `io_uring_get_sqe()` 后，需要使用该函数把 user_data 和 SQE 相关联。完成后，可以调用函数 `io_uring_cqe_get_data()` 来获取提交的数据指针中的值。

**内核只是单纯拷贝 user_data，什么都不会动**，你要在 submit 前先提交绑定请求；这个函数是用来给你识别你的请求的，因为 io_uring 并不是按提交顺序执行后进入 CQ，而是谁执行完谁就先进 CQ。

---

### io_uring_wait_cqe

```cpp
int io_uring_wait_cqe(struct io_uring *ring,
                      struct io_uring_cqe **cqe_ptr);
```

该函数等待 参数 `ring` 中的 IO completion。如果调用时已经完成，那么不会等待，`cqe_ptr` 会被填写。

在调用 `io_uring_submit()` 后，应用程序可以通过 `io_uring_wait_cqe` 获取 completion。

返回值：成功返回0，填写 cqe_ptr；失败返回 -errno 表明正在等待 CQE。

---

### io_uring_peek_cqe

```cpp
int io_uring_peek_cqe(struct io_uring *ring,
                      struct io_uring_cqe **cqe_ptr);
```

该函数检查 `ring` 并在可以消费时返回一个 IO Completion。如果成功返回，那么 `cqe_ptr` 会填充有效的 CQE。

这个函数不会进入内核等待 event，只会在 CQ 完成 CQE 时返回

返回值：成功返回0，并填充 `cqe_ptr`；失败返回 -EAGAIN

---

### io_uring_prep_accept/multishot accept

```cpp
void io_uring_prep_accept(struct io_uring_sqe *sqe,
                          int sockfd,
                          struct sockaddr *addr,
                          socklen_t *addrlen,
                          int flags);

void io_uring_prep_multishot_accept(struct io_uring_sqe *sqe,
                                    int sockfd,
                                    struct sockaddr *addr,
                                    socklen_t *addrlen,
                                    int flags);
```

`io_uring_prep_accept(...)` 用于准备一个请求，向 SQ 提交一个 accept 类似的请求，在 `sock_fd` 上准备 accept 连接，地址如参数所示。

此外，kernel 5.19 后支持连发 (`multishot`) accept，普通的 accept 一次只能接受一个连接，产生一个 CQE，之后就必须重新提交请求了，但是 multishot 版本中，内核会自动帮你保持监听状态，下一次收到连接时依然会产生 CQE。

要注意的是，在连发版本中写 `addr`,  `addrlen` 可能意义不大，因为每个接受的连接都使用相同的值，这意味着写入 `addr` 的数据可能会在被处理前就被新链接覆盖掉。

连发请求一般会一直有效，除非

- 被取消，例如使用 `io_uring_prep_cancel()` 这样的函数
- 本身遇到错误

如果连发请求还有效，那么 CQE 的 flags 字段会包含 `IORING_CQE_F_MORE`。如果没有该标志，那么连发请求失效，程序会重新提交新请求。

对于

- `io_uring_prep_accept`, CQE 中的 *res* 保存了连接的 socket。
- 对于 `io_uring_prep_multishot_accept`，可能需要配合 linux `getpeername()` 来获取客户端地址。

---

### io_uring_cqe_seen

```cpp
void io_uring_cqe_seen(struct io_uring *ring,
                       struct io_uring_cqe *cqe);
```

**该函数用于标记 `ring` 的 `cqe` 已经被消耗掉了。**

在 caller 使用 `io_uring_submit` 提交了请求后，可以通过 `io_uring_wait_cqe` 以及 `io_uring_peek_cqe` 来获取当前结果，**并且需要将其标记为已经消耗，这样这个 CQE slot 才能被复用。**

---

### io_uring_prep_provide_buffers

```cpp
void io_uring_prep_provide_buffers(struct io_uring_sqe *sqe,
                                   void *addr,
                                   int len,
                                   int nr,
                                   int bgid,
                                   int bid);
```

该函数准备一个给 kernel 提供 buffer 的请求。`sqe` 设置为消费 `nr` 数量的 `len` 长度的 buffer，起始地址为 `addr`，被唯一标识 buffer group ID `bgid`，并且从 `bid` 开始顺序排列。

通过设置 SQE 中的 `buf_group` 字段，然后在 `SQE` `flags` 中设置 `IOSQE_BUFFER_SELECT` 实现。

如果使用该方法来提供 buffer，那么**不应该在地址字段中提供缓冲区**，直接设置为 NULL 即可。应该在 gourp ID 中进行设置来匹配一个之前提交给内核的 buffer。之后 kernel 会在该 group 中进行 IO。完成后，`CQE` `flags` 将会填充 `IORING_CQE_F_BUFFER`，且所选取的**缓冲区 ID 会在 `flags` 的上16位**

不同的 buffer group ID 可以让应用程序用来识别不同的大小和类型的 buffer。一旦 buffer 被消费，io_uring 就不再使用它。必要的话需要**重新提供一次**，**否则你可以手动销毁**。

buffer IDs 内部通过 `bid` 进行升序排列。**如果提供了 16 个 buffer，且初始 `bid` 是 0，那么 buffer 的 id 范围是 0~15.**

buffer IDs 的取值范围是 0~65535，因为只有 `CQE` 的上 16 位用作记录。如果添加的 buffer ID 太大会 -E2BIG，在转换为 16bit 值时会 -EINVAL。

并不是所有的请求都支持该操作，**只对从内核接受数据的操作有效，**而向内核写数据时不能这么干。目前，支持任何的 file 读或者 socket 接收请求。否则 CQE 中会返回 -EINVAL。

此外，该操作支持 iovec，但前提是你只提供了一个 iovec。

如果 IO 触发太快，那 buffer 可能会耗尽。这种情况下操作会失败，然后 -ENOBUFS。可以选择为同一类型的 buffer 设置多个 group，然后循环使用。一旦溢出，就切换到下一个 group，然后重新提交 IO 请求。

kernel 5.7 后支持。

---

### io_uring_submit_and_wait

```cpp
int io_uring_submit_and_wait(struct io_uring *ring,
                             unsigned wait_nr);
```

向 SQ 提交请求，并且等待 n 个完成事件。

只有一次 syscall。

返回值：成功返回提交成功的 SQE 数量，失败返回 -errno

---

### io_uring_cq_advance

```cpp
void io_uring_cq_advance(struct io_uring *ring,
                          unsigned nr);
```

将 `nr` 个 `io_uring` CQE 标记为被消耗。

`io_uring_cqe_seen()` 会调用这个函数。

----

## 资料出处

[Arch manual pages (archlinux.org)](https://man.archlinux.org/listing/liburing)

[axboe/liburing (github.com)](https://github.com/axboe/liburing)

《Linux 多线程服务端编程》陈硕。
