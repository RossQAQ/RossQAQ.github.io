---
title: io_uring P2 - 实现 cp
description: 一起学习使用 liburing 一次提交多个 requests 来实现 cp 命令
slug: io_uring-cp
date: 2024-02-16 00:00:00+0000
image: 
categories:
    - io-uring
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文是文章 [io_uring by example: Part 2 – Queuing multiple requests](https://unixism.net/2020/04/io-uring-by-example-part-2-queuing-multiple-requests/) 的翻译与总结。

同样只摘录重要的部分。

这一篇文章使用 liburing 来构造类似于 cp 命令的程序，但区别于 P1，由于 P1 的重点在于理解 io_uring 本身，所以并没有用上很多特性。

## 简介

在 P1 中，我们构建了一个 cat 程序。但我们的例子中，一次只提交了一个 request。而 `io_uring` 的使命就是减少 syscall 的次数，办法是让用户向队列中一次尽可能多地提交 request，然后内核可以一次性全部处理，就不用用户对每个 io request 都调用一次 syscall了。

这一部分，我们实现一个复制文件的程序。它一次尽可能多地提交请求（在队列长度限制内）。To give credit where it is due, this is heavily based on [a program from the fio package](https://github.com/axboe/fio/blob/master/t/io_uring.c).

## cp_liburing

```cpp
#include <asm-generic/errno-base.h>
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <assert.h>
#include <errno.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <liburing.h>
#define QD  2
#define BS (16 * 1024)

static int infd, outfd;

struct io_data {
    int read;
    off_t first_offset, offset;
    size_t first_len;
    struct iovec iov;
};

static int setup_context(unsigned entries, struct io_uring* ring) {
    int ret;

    ret = io_uring_queue_init(entries, ring, 0);
    if (ret < 0) {
    fprintf(stderr, "queue_init: %s\n", strerror(-ret));
        return -1;
    }
    return 0;
}

static int get_file_size(int fd, off_t* size) {
    struct stat st;
    if (fstat(fd, &st) < 0 )
        return -1;
    if(S_ISREG(st.st_mode)) {
        *size = st.st_size;
        return 0;
    } else if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;
        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0)
            return -1;
        *size = bytes;
        return 0;
    }
    return -1;
}

static void queue_prepped(struct io_uring* ring, struct io_data* data) {
    struct io_uring_sqe* sqe;

    sqe = io_uring_get_sqe(ring);
    assert(sqe);

    if (data->read) {
        io_uring_prep_readv(sqe, infd, &data->iov, 1, data->offset);
    } else {
        io_uring_prep_writev(sqe, outfd, &data->iov, 1, data->offset);
    }

    io_uring_sqe_set_data(sqe, data);
}

static int queue_read(struct io_uring* ring, off_t size, off_t offset) {
    struct io_uring_sqe* sqe;
    struct io_data* data;

    data = malloc(size + sizeof(*data));
    if (!data) {
        return -1;
    }

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        free(data);
        return 1;
    }

    data->read = 1;
    data->offset = data->first_offset = offset;

    data->iov.iov_base = data + 1;
    data->iov.iov_len = size;
    data->first_len = size;

    io_uring_prep_readv(sqe, infd, &data->iov, 1, offset);
    io_uring_sqe_set_data(sqe, data);
    return 0;
}

static void queue_write(struct io_uring* ring, struct io_data* data) {
    data->read = 0;
    data->offset = data->first_offset;

    data->iov.iov_base = data + 1;
    data->iov.iov_len = data->first_len;

    queue_prepped(ring, data);
    io_uring_submit(ring);
}

int copy_file(struct io_uring* ring, off_t insize) {
    unsigned long reads, writes;
    struct io_uring_cqe* cqe;
    off_t write_left, offset;
    int ret;

    write_left = insize;
    writes = reads = offset = 0;

    while (insize || write_left) {
        int had_reads, got_comp;

        /* 尽可能多地提交读操作 */
        while (insize) {
            off_t this_size = insize;

            if (reads + writes >= QD) {
                break;
            }
            if (this_size > BS) {
                this_size = BS;
            } else if (!this_size) {
                break;
            }

            if (queue_read(ring, this_size, offset)) {
                break;
            }

            insize -= this_size;
            offset += this_size;
            reads++;
        }

        if (had_reads != reads) {
            ret = io_uring_submit(ring);
            if (ret < 0) {
                fprintf(stderr, "io_uring_submit: %s\n", strerror(-ret));
                break;
            }
        }

        /* 此时队列已经满了，我们需要找到至少一个 completion */
        got_comp = 0;
        while (write_left) {
            struct io_data* data;
            if (!got_comp) {
                ret = io_uring_wait_cqe(ring, &cqe);
                got_comp = 1;
            } else {
                ret = io_uring_peek_cqe(ring, &cqe);
                if (ret == -EAGAIN) {
                    cqe = NULL;
                    ret = 0;
                }
            }
            if (ret < 0) {
                fprintf(stderr, "io_uring_peek_cqe: %s\n",
                        strerror(-ret));
                return 1;
            }
            if (!cqe) {
                break;
            }

            data = io_uring_cqe_get_data(cqe);
            if (cqe->res < 0) {
                if (cqe->res == -EAGAIN) {
                    queue_prepped(ring, data);
                    io_uring_cqe_seen(ring, cqe);
                    continue;
                }
                fprintf(stderr, "cqe failed: %s\n",
                        strerror(-cqe->res));
                return 1;
            } else if (cqe->res != data->iov.iov_len) {
                /* short read/write; adjust and requeue */ // ? 没看懂
                data->iov.iov_base += cqe->res;
                data->iov.iov_len -= cqe->res;
                queue_prepped(ring, data);
                io_uring_cqe_seen(ring, cqe);
                continue;
            }

            /**
             * 完成。如果写，不需要做什么了，如果读，就提交对应的写
             */
            if (data->read) {
                queue_write(ring, data);
                write_left -= data->first_len;
                reads--;
                writes++;
            } else {
                free(data);
                writes--;
            }
            io_uring_cqe_seen(ring, cqe);
        }
    }
    return 0;
}

int main(int argc, char *argv[]) {
    struct io_uring ring;
    off_t insize;
    int ret;

    if (argc < 3) {
        printf("Usage: %s <infile> <outfile>\n", argv[0]);
        return 1;
    }

    infd = open(argv[1], O_RDONLY);
    if (infd < 0) {
        perror("open infile");
        return 1;
    }

    outfd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (outfd < 0) {
        perror("open outfile");
        return 1;
    }

    if (setup_context(QD, &ring)) {
        return 1;
    }

    if (get_file_size(infd, &insize)) {
        return 1;
    }

    ret = copy_file(&ring, insize);

    close(infd);
    close(outfd);
    io_uring_queue_exit(&ring);
    return ret;
}
```

## 程序结构

该程序和普通的 cp 一样。

程序的核心是 copy_file() 函数。这里，我们外层的 while 循环内包含了两个 while 循环。

外层的 while 循环保证源文件中的字节都被拷贝了，第一个嵌套的 while 循环的任务是尽可能多的创建 readv 请求，实际上，它会创建队列长度允许的数量的请求。

一旦队列满了，我们就来到第二个 while 循环。这个循环获取 SQE 并且提交写文件的请求，现在 data 已经读完了。

有很多跟踪状态的变量，看起来有一些迷惑。但一个异步 copying 的程序能有多难呢？

> 意思是不想解释了让读者自己看。还行，多看几遍基本能理解整体逻辑。学习 io_uring 还是为了网络编程，所以这里我就大概只理清了逻辑。

## 下一步？

现在我们知道了如何一次性多的使用 io_uring 提交请求。下一次我们进行一些网络编程。我们会简单的建一个 web server，从头开始使用 io_uring 完成所有 IO。

## About me

My name is Shuveb Hussain and I’m the author of this Linux-focused blog. You can [follow me on Twitter](https://twitter.com/shuveb) where I post tech-related content mostly focusing on Linux, performance, scalability and cloud technologies.
