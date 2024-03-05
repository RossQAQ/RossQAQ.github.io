---
title: select/poll/epoll 老三样
description: 老三样的使用方法
slug: io_multiplxer
date: 2023-12-27 00:00:00+0000
image: 
categories:
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

## 前言

记录一下老三样的使用方法

顺便，初始化 socket 就不重复写了。

## 事件

我们需要区分一下网络通讯（tcp）的事件，例如

读事件

1. 新客户端连接，accept 队列中有新的 socket
2. 客户端发送的数据已经到达缓存
3. 客户端断连

写事件

发送缓冲区有空间，可以向客户端发送数据。

## select

```cpp
#include <sys/select.h>
#include <unistd.h>

#include <array>
#include <iostream>

#include "util.hpp"

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cerr << "Usage: ./select port\n";
        return -1;
    }

    int sockfd = socket_init(atoi(argv[1]));
    std::cout << "Listen: " << sockfd << '\n';

    // select 可以监视上述读事件和写事件

    // 监视的 socket 集合，1024 bit 的 bitmap (int[32])
    // 操作位图的宏：
    // void FD_CLR(int fd, fd_set* set)
    // int FD_ISSET(int fd, fd_set* set)
    // void FD_SET(int fd, fd_set* set)
    // void FD_ZERO(fd_set* set)
    fd_set fds;
    FD_ZERO(&fds);
    FD_SET(sockfd, &fds);

    int max_fd = sockfd;

    for (;;) {
        timeval timeout{ .tv_sec = 10, .tv_usec = 0 };
        // 先拷贝一份 bitmap，因为会被修改
        fd_set tmpfds = fds;

        // select
        // p1: bitmap size (max + 1)
        // p2: 监视读事件的 bitmap
        // p3: 监视写事件的 bitmap，如果监视写事件，那么发送缓冲区没满就会一直通知
        // p4: 监视异常事件的 bitmap
        int nfds = select(max_fd + 1, &tmpfds, nullptr, nullptr, nullptr);

        if (nfds < 0) {
            std::cerr << "select() failed\n";
        }
        if (nfds == 0) {
            std::cerr << "select() timeout\n";
        }

        // if > 0
        for (int eventfd{}; eventfd <= max_fd; ++eventfd) {
            if (FD_ISSET(eventfd, &tmpfds) == 0)
                continue;

            //  发生事件的是 linsten socket，那么说明有新的客户端连接
            if (eventfd == sockfd) {
                sockaddr_in client;
                socklen_t len = sizeof(sockaddr_in);
                int client_sock = accept(sockfd, (sockaddr*)&client, &len);

                if (client_sock < 0) {
                    std::cerr << "accept() failed\n";
                    continue;
                }

                std::cout << "accept client. socket = " << client_sock << '\n';

                // 把新连接的 socket 也要进行监听
                FD_SET(client_sock, &fds);

                max_fd = std::max(max_fd, client_sock);
            } else {
                // 如果是客户端的 socket 发生事件，
                // 那么 1. 说明是接收到了数据; 2. 客户端断开连接
                std::array<char, 1024> buf{};

                // 客户端断开连接
                if (read(eventfd, buf.data(), buf.size()) <= 0) {
                    std::cout << "Disconnected. socket = " << eventfd << '\n';
                    close(eventfd);
                    FD_CLR(eventfd, &fds);

                    // 恰好是最后一个客户端，那么就要重新计算 max_fd
                    if (eventfd == max_fd) {
                        for (int i = max_fd; i > 0; --i) {
                            if (FD_ISSET(i, &fds)) {
                                max_fd = i;
                                break;
                            }
                        }
                    }
                } else {
                    // 如果客户端有数据发送
                    std::cout << "client: " << eventfd << "\nmsg: " << buf.data();
                    write(eventfd, buf.data(), buf.size());
                }
            }
        }
    }
    return 0;
}
```

select 遵循水平触发，fd 发生事件 select 就会返回通知；如果事件没有被处理，那么再次调用 select 也会返回。

问题：

- 轮询方式扫描 bitmap，socket一多性能就拉胯
- **每次都得拷贝 bitmap**
- bitmap 数量有限制

## poll

```cpp
#include <poll.h>
#include <unistd.h>

#include <array>
#include <iostream>

#include "util.hpp"

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cerr << "Usage: ./select port\n";
        return -1;
    }

    int sockfd = socket_init(atoi(argv[1]));
    std::cout << "Listen: " << sockfd << '\n';

    // fds 存放需要监视的 socket
    pollfd fds[1024];
    // poll 会忽略 -1 的 socket
    for (int i{}; i < 1024; ++i) { fds[i].fd = -1; }

    // 监视 listen socket
    fds[sockfd].fd = sockfd;
    fds[sockfd].events = POLLIN;
    // POLLIN | POLLOUT

    int max_fd = sockfd;
    for (;;) {
        int nfds = poll(fds, 1024, -1);
        if (nfds < 0) {
            std::cerr << "poll() failed.\n";
        }

        for (int eventfd{}; eventfd <= max_fd; ++eventfd) {
            if (fds[eventfd].revents & POLLIN == 0)
                continue;

            //  发生事件的是 linsten socket，那么说明有新的客户端连接
            if (eventfd == sockfd) {
                sockaddr_in client;
                socklen_t len = sizeof(sockaddr_in);
                int client_sock = accept(sockfd, (sockaddr*)&client, &len);

                if (client_sock < 0) {
                    std::cerr << "accept() failed\n";
                    continue;
                }

                std::cout << "accept client. socket = " << client_sock << '\n';

                fds[client_sock].fd = client_sock;
                fds[client_sock].events = POLLIN;

                max_fd = std::max(max_fd, client_sock);
            } else {
                // 如果是客户端的 socket 发生事件，
                // 那么 1. 说明是接收到了数据; 2. 客户端断开连接
                std::array<char, 1024> buf{};

                // 客户端断开连接
                if (read(eventfd, buf.data(), buf.size()) <= 0) {
                    std::cout << "Disconnected. socket = " << eventfd << '\n';
                    close(eventfd);
                    fds[eventfd].fd = -1;

                    // 恰好是最后一个客户端，那么就要重新计算 max_fd
                    if (eventfd == max_fd) {
                        for (int i = max_fd; i > 0; --i) {
                            if (fds[eventfd].fd != -1) {
                                max_fd = i;
                                break;
                            }
                        }
                    }
                } else {
                    // 如果客户端有数据发送
                    std::cout << "client: " << eventfd << "\nmsg: " << buf.data();
                    write(eventfd, buf.data(), buf.size());
                }
            }
        }
    }
}
```

poll 基本就是 select 的 plus 版，没啥太大区别

- poll 的数据结构是数组，在内核中为链表
- 调用一次 select 要拷贝两次 bitmap (自己拷贝一次，内核拷贝一次)，poll 要拷贝一次结构体数组
- poll 没有 1024 限制，但也很烦

## epoll

### LT

水平触发：不管是读事件还是写事件，只要有数据，那么调用 epoll_wait 就会通知，触发事件。

也就是说，epoll 会一直通知你，直到你把数据发送/接收完。

```cpp
#include <sys/epoll.h>
#include <unistd.h>

#include <array>
#include <iostream>

#include "util.hpp"

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cerr << "Usage: ./select port\n";
        return -1;
    }

    int sockfd = socket_init(atoi(argv[1]));
    std::cout << "Listen: " << sockfd << '\n';

    // 创建 epoll fd
    int epollfd = epoll_create(1);

    // 为服务端准备读事件
    epoll_event ev;
    // 指定事件的自定义数据，会随 epoll_wait() 返回的事件一起返回
    ev.data.fd = sockfd;
    ev.events = EPOLLIN;

    // 需要监听的 socket 加入 epoll
    epoll_ctl(epollfd, EPOLL_CTL_ADD, sockfd, &ev);

    // 事件数组，可以随意设置大小
    epoll_event evs[10];

    for (;;) {
        int nfds = epoll_wait(epollfd, evs, 10, -1);
        if (nfds < 0) {
            std::cerr << "epoll_wait() failed.\n";
            break;
        }
        if (nfds == 0) {
            std::cerr << "epoll_wait() timeout.\n";
            continue;
        }

        for (int i{}; i < nfds; ++i) {
            // 如果发生事件的是 sockfd 表示有新连接
            if (evs[i].data.fd == sockfd) {
                sockaddr_in client;
                socklen_t len = sizeof(sockaddr_in);
                int client_sock = accept(sockfd, (sockaddr*)&client, &len);

                if (client_sock < 0) {
                    std::cerr << "accept() failed\n";
                    continue;
                }

                std::cout << "accept client. socket = " << client_sock << '\n';

                // 为新的连接准备事件
                ev.data.fd = client_sock;
                ev.events = EPOLLIN;
                epoll_ctl(epollfd, EPOLL_CTL_ADD, client_sock, &ev);
            } else {
                // 如果是客户端的 socket 发生事件，
                // 那么 1. 说明是接收到了数据; 2. 客户端断开连接
                std::array<char, 1024> buf{};

                // 客户端断开连接
                if (recv(evs[i].data.fd, buf.data(), buf.size(), 0) <= 0) {
                    std::cout << "Disconnected. socket = " << evs[i].data.fd << '\n';
                    close(evs[i].data.fd);
                    // socket 关闭后会自动被删除
                    // epoll_ctl(epollfd, EPOLL_CTL_DEL, evs[i].data.fd, 0);
                } else {
                    // 如果客户端有数据发送
                    std::cout << "client: " << evs[i].data.fd << "\nmsg: " << buf.data();
                    send(evs[i].data.fd, buf.data(), buf.size(), 0);
                }
            }
        }
    }
    return 0;
}
```

### ET

边缘触发：

读事件：**只有新的数据到达**，才会触发读事件，不管有没有处理读事件，epoll_wait 对于每个事件只提醒一次。也就是说，如果数据没读完，那也不会再通知你；每次只会读 buffer 数量的数据（下一次有新数据时继续读）

写事件：epoll_wait 触发写后，如果发送缓冲区仍然没有写满，那么就不会再次触发。发送缓冲区从满 -> 不满，才会触发写事件。

监听 socket 采用边缘触发时，要使用非阻塞模式，否则会有 socket 在队列中没有被处理；然后使用循环。

如果客户端连接进来的 socket 使用边缘触发，也要设置为非阻塞模式，之后循环调用 recv

```cpp

```

### 阻塞与非阻塞 IO

阻塞：发起一个调用，在调用的结果完成之前，线程被阻塞，期间让出 CPU 的使用权。

非阻塞：发起一个调用，不管结果，立刻返回。

在网络编程中的阻塞函数：connect(), accept(), send(), recv()

非阻塞 connect：立刻返回， errno = EINPROGRESS。可以用 poll 监视 socket 的写事件，可写就连接成功。但实则没啥作用，一般服务端才会使用非阻塞 IO。

非阻塞 accept（设定监听 socket 为非阻塞）：如果连接队列没有 socket，errno = EAGAIN。还是判断 EAGAIN，如果不是就失败。

非阻塞 send/recv：发送/接收缓冲区为空，立刻返回，errno = EAGAIN
