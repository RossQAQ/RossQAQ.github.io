---
title: io_uring 以及网络编程，我的想法
description: 崩溃中
slug: io_uring-and-network
date: 2024-03-05 00:00:00+0000
image: 
categories:
    - io-uring
    - cppnotes
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

## 前言

该文章中的一切，不出意外的话会在我个人的服务器测试，

系统是 fedora 39，linux 内核版本 6.5.6，g++ 13.2，liburing 是最新版。

## io_uring echo-server

> 写于 2024.3.5，此时本人对 io_uring 刚刚懂 API，也不完全懂，网络编程略懂略懂，也许以后自己给自己 code review 吧hhh。

研究个 echo_server，然后慢慢迭代？

### 基础款

> 2024.3.5，想了半天，先试试阻塞的 socket 以及 单发的 accept 吧，最简单的开始，且 SQ 大小也足够大，顺便使用最简单的 switch 来调度一下。



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

### io_uring_set_data

```cpp
void io_uring_sqe_set_data(struct io_uring_sqe *sqe,
                           void *user_data);
```

该函数存储一个含有 SQE 的指针 *user_data*

在用户使用过 `io_uring_get_sqe()` 后，需要使用该函数把 user_data 和 SQE 相关联。完成后，可以调用函数 `io_uring_cqe_get_data()` 来获取提交的数据指针中的值。

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

---

### io_uring_cqe_seen

```cpp
void io_uring_cqe_seen(struct io_uring *ring,
                       struct io_uring_cqe *cqe);
```

该函数用于标记 `ring` 的 `cqe` 已经被消耗掉了。

在 caller 使用 `io_uring_submit` 提交了请求后，可以通过 `io_uring_wait_cqe` 以及 `io_uring_peek_cqe` 来获取当前结果，**并且需要将其标记为已经消耗，这样这个 CQE slot 才能被复用。**

## 资料出处

