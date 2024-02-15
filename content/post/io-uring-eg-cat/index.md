---
title: io_uring P1 - 实现 cat
description: 一起阅读文章，学习 readv, io_uring, liburing 实现 cat 的做法。
slug: io_uring-cat
date: 2024-02-15 00:00:00+0000
image: 
categories:
    - techs
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文是文章 [io_uring by example: Part 1 – Introduction](https://unixism.net/2020/04/io-uring-by-example-part-1-introduction/) 的翻译与总结。

原文比较长，故只摘录重要部分。

学习 Linux 新异步 I/O API io_uring 的使用，以及与传统同步 API 的异同，并接触更高级更方便的 liburing 库。

一切都从实现一个 cat 程序开始。

## 介绍

Linux 原生提供了同步I/O 和 异步 I/O（aio） 两种 API，同步 IO 就是熟悉的阻塞 IO，而异步 IO 的 aio 则只能支持直接 IO，buffered IO 并不能异步，这就是问题所在。

`io_uring` 的诞生就是为了解决 Linux 内核没有异步 IO 的问题。

`io_uring` 不仅提供了优雅的 kernel/user 接口，还提供了一些提高性能的方式（特殊的 polling mode）来避免数据跨越 kernel/user 空间时的系统调用。

`io_uring`  提供了更高级封装后的 `liburing`，隐藏了很多实现细节，但如果不理解底层 api 只用 liburing 那有什么意思呢？后面的例子都会使用 `liburing`，但我们先从底层的 API 开始实现。

## 普通的 cat

我们实现一个简单的 `cat` 指令， 通过使用 syscall `readv()`，它是阻塞同步 I/O 方式。你需要熟悉一下 readv 是怎么工作的。readv 称之为 vectored I/O。

read 和 write 的参数是 fd， buffer，长度；而 readv 和 writev 的参数是 fd，指向 `struct iovec` 的结构体指针。

iovec 结构体如下：

```cpp
struct iovec {
	void* iov_base;
    size_t iov_len;
};
```

对比常规的 read/write 有什么区别呢？readv/writev 的使用更加符合直觉，你可以填充结构体的多个数据成员然后一次 syscall 读完；此外 readv/writev 是原子的。

我们的 cat 例子中，我们会使用 readv 读取文件然后打印到控制台。我们会一个 chunk 一个 chunk 的读取，每个都会使用 iovec 指向。readv 会在完成时阻塞，假设没有错误，`struct iovec` 指向一系列的存储 file 内容的 buffer。之后再打印。很简单。

```c
#include <bits/types/struct_iovec.h>
#include <stdio.h>
#include <sys/uio.h>
#include <sys/stat.h>
#include <linux/fs.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <stdlib.h>
#define BLOCK_SZ    4096

/**
 * 返回传入的 fd 大小。可以处理常规文件和硬件驱动。
 */
off_t get_file_size(int fd) {
    struct stat st;

    if (fstat(fd, &st) < 0) {
        perror("fstat");
        return -1;
    }

    if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;
        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0) {
            perror("ioctl");
            return -1;
        }
        return bytes;
    } else if (S_ISREG(st.st_mode)) {
        return st.st_size;
    }
    return -1;
}

/**
 * 向 stdout 输出长度为 len 的字符串。
 * 我们使用 buffered 输出提高效率。
 * 因此我们需要一个一个的输出字符。
 */
void output_to_console(char* buf, int len) {
    while (len--) {
        fputc(*buf++, stdout);
    }
}

int read_and_print_file(char* file_name) {
    struct iovec* iovecs;
    int fd = open(file_name, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    } 

    off_t file_sz = get_file_size(fd);
    off_t bytes_remaining = file_sz;
    int blocks = (int) file_sz / BLOCK_SZ;
    if (file_sz % BLOCK_SZ) blocks++;
    iovecs = malloc(sizeof(struct iovec) * blocks);

    int cur_blk = 0;

    /**
     * 对于我们要读的文件，先分配足够的空间存放数据。
     * 每个块都被描述为一个 iovec 结构，
     * 被传递给 readv 作为 iovecs 数组的一部分 
     */
    
    while (bytes_remaining) {
        off_t bytes_to_read = bytes_remaining;
        if (bytes_to_read > BLOCK_SZ) {
            bytes_to_read = BLOCK_SZ;
        }

        void *buf;
        if (posix_memalign(&buf, BLOCK_SZ, BLOCK_SZ)) {
            perror("posix_memalign");
            return 1;
        }
        iovecs[cur_blk].iov_base = buf;
        iovecs[cur_blk].iov_len = bytes_to_read;
        cur_blk++;
        bytes_remaining -= bytes_to_read;
    }

    /**
     * readv() 调用会阻塞，直到 iovecs 读满。
     * 当他返回时，我们就可以访问读取的数据了
     */

     int ret = readv(fd, iovecs, blocks);

     if (ret < 0) {
        perror("readv");
        return 1;
     }

     for (int i = 0; i < blocks; ++i) {
        output_to_console(iovecs[i].iov_base, iovecs[i].iov_len);
     }

     return 0;
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <filename1> [<filename2>...]\n", argv[0]);
        return 1;
    }

    /**
     * 对于每个传入的文件都调用 read_and_print_file() 函数
     */

    for (int i = 1; i < argc; ++i) {
        if (read_and_print_file(argv[i])) {
            fprintf(stderr, "Error reading file\n");
            return 1;
        }
    }

    return 0;
}
```

以上代码很简单，之后我们会把他和 io_uring 的版本对比。

它的核心在于一个循环先计算我们要读的文件需要多少块 blocks 来存储数据。分配所有 iovec 的内存。之后迭代，分配 block-sized 内存来存储实际的数据，最后调用 readv。就像我们之前说的，readv 是同步的，**意味着在完成前会一直阻塞**。当它返回时，数据已经读取好了。我们就可以输出到控制台了。

## Cat uring

我们赶紧来实现 io_uring  的版本，在 io_uring 中使用的操作会是 readv。

### io_uring 接口

io_uring 接口很简单。有一个 submission queue 和一个 completion queue。

在 submission queue 中，你提交你想要执行的操作信息。

例如，对于这个程序，我们想要使用 readv() 读取文件，所以我们布置一个描述它的 submission queue request 作为 submission queue entry（SQE）的一部分。因为它是队列，所以你可以放置多个请求，只要队列的长度允许（你可以自己定义）即可。执行的操作可以是 reads, writes 等等。之后我们**调用 `io_uring_enter()`** syscall 来告诉内核，我们向 submission queue 添加了一个操作。

内核完成请求后，会将结果放置在 completion queue 作为 CQE，或者说 a completion queue entry one for each corresponding SQE. (? 实在没看懂这句怎么翻译)

CQEs 可以在用户态下访问。

精明的读者会发现，这个接口会先装满队列再使用*一次 syscall* 而不是对于每个 IO 请求都调用一次 syscall，已经提升了效率。为更高的效率，**io_uring 提供一种内核持续轮询（*polls*）的模式来检测是否有提交项，而不需要调用 `io_uring_enter()` 来通知内核。**

在做这些之前，你需要 setup 队列，也就是拥有固定长度的环形缓冲区。你可以使用 `io_uring_setup()` 来完成。我们要做的工作是通过向环形缓冲区中添加 submission queue entries 并且从从 completion queue 环形缓冲区中读取 completion queue entries。这就是 io_uring 的设计总览。

### Completion Queue Entry

现在我们脑子里已经知道他是怎么工作的了，我们来看看实现细节。跟 submission queue entry (SQE) 比起来，completion queue entry (CQE) 非常简单。SQE 是你用来提交请求的结构体，你要把他提交给环形缓冲区。CQE 是内核对于每个添加到 submission queue 的 SQE 结构体的响应结构体。他包括了你通过 SQE 实例请求的操作的结果。

```c
    struct io_uring_cqe {
  __u64  user_data;  /* sqe->user_data submission passed back */
  __s32  res;    /* result code for this event */
  __u32  flags;
    };
```

`user_data` field 是按原样从 SQE 传递到 CQE 实例的内容。假设你传递了一堆操作给 submission queue，它们的完成顺序与到达 completion queue 的顺序是不重要的。因为底层的 IO 速度可能不同。总之，CQEs 可以以任何顺序进入 completion queue ，只要它们完成了，那么就会立刻进入 completion queue。那么如何识别 SQE 对应的 CQE 呢？之后会有详细解释。

CQE 很简单，因为它只关心它的 syscall 的返回值，存储在 res 字段中。例如，如果你提交一个 读 操作，那么完成后，他就会包含读取的字节数。如果有错误，它就会包含 -errno。就像 read() 本身的行为一样。

### Ordering

虽然 CQEs 确实不是按顺序返回，但你也可以强制其按 SQE 的顺序返回，具体看 [canonical io_uring reference](https://kernel.dk/io_uring.pdf) 

### Submission Queue Entry

submission queue 更加复杂，因为他要保证兼容如今 linux 能做的所有 IO 操作。

```cpp
struct io_uring_sqe {
  __u8  opcode;    /* type of operation for this sqe */
  __u8  flags;    /* IOSQE_ flags */
  __u16  ioprio;    /* ioprio for the request */
  __s32  fd;    /* file descriptor to do IO on */
  __u64  off;    /* offset into file */
  __u64  addr;    /* pointer to buffer or iovecs */
  __u32  len;    /* buffer size or number of iovecs */
  union {
    __kernel_rwf_t  rw_flags;
    __u32    fsync_flags;
    __u16    poll_events;
    __u32    sync_range_flags;
    __u32    msg_flags;
  };
  __u64  user_data;  /* data to be passed back at completion time */
  union {
    __u16  buf_index;  /* index into fixed buffers, if used */
    __u64  __pad2[3];
  };
};
```

结构体看上去很复杂，但实际上常用的不多。我们通过 cat 和使用 readv() 来理解他。

- `opcode` 指定 I/O 操作，我们的情况中，readv() 使用 `IORING_OP_READV`
- `fd`，文件描述符
- `addr`，指向我们定义的 `iovecs` 结构，存储了我们为了 I/O 分配的 buffer 和长度

- 最后 len 是 iovecs 数组的大小

现在感觉不是很难了，我们可以一次入队多个 SQEs 然后一次 syscall 全部解决。

### io_uring 版本的 cat

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>
#include <sys/mman.h>
#include <sys/uio.h>
#include <linux/fs.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

/* If your compilation fails because the header file below is missing,
 * your kernel is probably too old to support io_uring.
 * */
#include <linux/io_uring.h>

#define QUEUE_DEPTH 1
#define BLOCK_SZ    1024

/* This is x86 specific */
#define read_barrier()  __asm__ __volatile__("":::"memory")
#define write_barrier() __asm__ __volatile__("":::"memory")

struct app_io_sq_ring {
    unsigned *head;
    unsigned *tail;
    unsigned *ring_mask;
    unsigned *ring_entries;
    unsigned *flags;
    unsigned *array;
};

struct app_io_cq_ring {
    unsigned *head;
    unsigned *tail;
    unsigned *ring_mask;
    unsigned *ring_entries;
    struct io_uring_cqe *cqes;
};

struct submitter {
    int ring_fd;
    struct app_io_sq_ring sq_ring;
    struct io_uring_sqe *sqes;
    struct app_io_cq_ring cq_ring;
};

struct file_info {
    off_t file_sz;
    struct iovec iovecs[];      /* Referred by readv/writev */
};

/*
 * This code is written in the days when io_uring-related system calls are not
 * part of standard C libraries. So, we roll our own system call wrapper
 * functions.
 * */

int io_uring_setup(unsigned entries, struct io_uring_params *p)
{
    return (int) syscall(__NR_io_uring_setup, entries, p);
}

int io_uring_enter(int ring_fd, unsigned int to_submit,
                          unsigned int min_complete, unsigned int flags)
{
    return (int) syscall(__NR_io_uring_enter, ring_fd, to_submit, min_complete,
                   flags, NULL, 0);
}

/*
 * Returns the size of the file whose open file descriptor is passed in.
 * Properly handles regular file and block devices as well. Pretty.
 * */

off_t get_file_size(int fd) {
    struct stat st;

    if(fstat(fd, &st) < 0) {
        perror("fstat");
        return -1;
    }
    if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;
        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0) {
            perror("ioctl");
            return -1;
        }
        return bytes;
    } else if (S_ISREG(st.st_mode))
        return st.st_size;

    return -1;
}

/*
 * io_uring requires a lot of setup which looks pretty hairy, but isn't all
 * that difficult to understand. Because of all this boilerplate code,
 * io_uring's author has created liburing, which is relatively easy to use.
 * However, you should take your time and understand this code. It is always
 * good to know how it all works underneath. Apart from bragging rights,
 * it does offer you a certain strange geeky peace.
 * */

int app_setup_uring(struct submitter *s) {
    struct app_io_sq_ring *sring = &s->sq_ring;
    struct app_io_cq_ring *cring = &s->cq_ring;
    struct io_uring_params p;
    void *sq_ptr, *cq_ptr;

    /*
     * We need to pass in the io_uring_params structure to the io_uring_setup()
     * call zeroed out. We could set any flags if we need to, but for this
     * example, we don't.
     * */
    memset(&p, 0, sizeof(p));
    s->ring_fd = io_uring_setup(QUEUE_DEPTH, &p);
    if (s->ring_fd < 0) {
        perror("io_uring_setup");
        return 1;
    }

    /*
     * io_uring communication happens via 2 shared kernel-user space ring buffers,
     * which can be jointly mapped with a single mmap() call in recent kernels. 
     * While the completion queue is directly manipulated, the submission queue 
     * has an indirection array in between. We map that in as well.
     * */

    int sring_sz = p.sq_off.array + p.sq_entries * sizeof(unsigned);
    int cring_sz = p.cq_off.cqes + p.cq_entries * sizeof(struct io_uring_cqe);

    /* In kernel version 5.4 and above, it is possible to map the submission and 
     * completion buffers with a single mmap() call. Rather than check for kernel 
     * versions, the recommended way is to just check the features field of the 
     * io_uring_params structure, which is a bit mask. If the 
     * IORING_FEAT_SINGLE_MMAP is set, then we can do away with the second mmap()
     * call to map the completion ring.
     * */
    if (p.features & IORING_FEAT_SINGLE_MMAP) {
        if (cring_sz > sring_sz) {
            sring_sz = cring_sz;
        }
        cring_sz = sring_sz;
    }

    /* Map in the submission and completion queue ring buffers.
     * Older kernels only map in the submission queue, though.
     * */
    sq_ptr = mmap(0, sring_sz, PROT_READ | PROT_WRITE, 
            MAP_SHARED | MAP_POPULATE,
            s->ring_fd, IORING_OFF_SQ_RING);
    if (sq_ptr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    if (p.features & IORING_FEAT_SINGLE_MMAP) {
        cq_ptr = sq_ptr;
    } else {
        /* Map in the completion queue ring buffer in older kernels separately */
        cq_ptr = mmap(0, cring_sz, PROT_READ | PROT_WRITE, 
                MAP_SHARED | MAP_POPULATE,
                s->ring_fd, IORING_OFF_CQ_RING);
        if (cq_ptr == MAP_FAILED) {
            perror("mmap");
            return 1;
        }
    }
    /* Save useful fields in a global app_io_sq_ring struct for later
     * easy reference */
    sring->head = sq_ptr + p.sq_off.head;
    sring->tail = sq_ptr + p.sq_off.tail;
    sring->ring_mask = sq_ptr + p.sq_off.ring_mask;
    sring->ring_entries = sq_ptr + p.sq_off.ring_entries;
    sring->flags = sq_ptr + p.sq_off.flags;
    sring->array = sq_ptr + p.sq_off.array;

    /* Map in the submission queue entries array */
    s->sqes = mmap(0, p.sq_entries * sizeof(struct io_uring_sqe),
            PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE,
            s->ring_fd, IORING_OFF_SQES);
    if (s->sqes == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    /* Save useful fields in a global app_io_cq_ring struct for later
     * easy reference */
    cring->head = cq_ptr + p.cq_off.head;
    cring->tail = cq_ptr + p.cq_off.tail;
    cring->ring_mask = cq_ptr + p.cq_off.ring_mask;
    cring->ring_entries = cq_ptr + p.cq_off.ring_entries;
    cring->cqes = cq_ptr + p.cq_off.cqes;

    return 0;
}

/*
 * Output a string of characters of len length to stdout.
 * We use buffered output here to be efficient,
 * since we need to output character-by-character.
 * */
void output_to_console(char *buf, int len) {
    while (len--) {
        fputc(*buf++, stdout);
    }
}

/*
 * Read from completion queue.
 * In this function, we read completion events from the completion queue, get
 * the data buffer that will have the file data and print it to the console.
 * */

void read_from_cq(struct submitter *s) {
    struct file_info *fi;
    struct app_io_cq_ring *cring = &s->cq_ring;
    struct io_uring_cqe *cqe;
    unsigned head, reaped = 0;

    head = *cring->head;

    do {
        read_barrier();
        /*
         * Remember, this is a ring buffer. If head == tail, it means that the
         * buffer is empty.
         * */
        if (head == *cring->tail)
            break;

        /* Get the entry */
        cqe = &cring->cqes[head & *s->cq_ring.ring_mask];
        fi = (struct file_info*) cqe->user_data;
        if (cqe->res < 0)
            fprintf(stderr, "Error: %s\n", strerror(abs(cqe->res)));

        int blocks = (int) fi->file_sz / BLOCK_SZ;
        if (fi->file_sz % BLOCK_SZ) blocks++;

        for (int i = 0; i < blocks; i++)
            output_to_console(fi->iovecs[i].iov_base, fi->iovecs[i].iov_len);

        head++;
    } while (1);

    *cring->head = head;
    write_barrier();
}
/*
 * Submit to submission queue.
 * In this function, we submit requests to the submission queue. You can submit
 * many types of requests. Ours is going to be the readv() request, which we
 * specify via IORING_OP_READV.
 *
 * */
int submit_to_sq(char *file_path, struct submitter *s) {
    struct file_info *fi;

    int file_fd = open(file_path, O_RDONLY);
    if (file_fd < 0 ) {
        perror("open");
        return 1;
    }

    struct app_io_sq_ring *sring = &s->sq_ring;
    unsigned index = 0, current_block = 0, tail = 0, next_tail = 0;

    off_t file_sz = get_file_size(file_fd);
    if (file_sz < 0)
        return 1;
    off_t bytes_remaining = file_sz;
    int blocks = (int) file_sz / BLOCK_SZ;
    if (file_sz % BLOCK_SZ) blocks++;

    fi = malloc(sizeof(*fi) + sizeof(struct iovec) * blocks);
    if (!fi) {
        fprintf(stderr, "Unable to allocate memory\n");
        return 1;
    }
    fi->file_sz = file_sz;

    /*
     * For each block of the file we need to read, we allocate an iovec struct
     * which is indexed into the iovecs array. This array is passed in as part
     * of the submission. If you don't understand this, then you need to look
     * up how the readv() and writev() system calls work.
     * */
    while (bytes_remaining) {
        off_t bytes_to_read = bytes_remaining;
        if (bytes_to_read > BLOCK_SZ)
            bytes_to_read = BLOCK_SZ;

        fi->iovecs[current_block].iov_len = bytes_to_read;

        void *buf;
        if( posix_memalign(&buf, BLOCK_SZ, BLOCK_SZ)) {
            perror("posix_memalign");
            return 1;
        }
        fi->iovecs[current_block].iov_base = buf;

        current_block++;
        bytes_remaining -= bytes_to_read;
    }

    /* Add our submission queue entry to the tail of the SQE ring buffer */
    next_tail = tail = *sring->tail;
    next_tail++;
    read_barrier();
    index = tail & *s->sq_ring.ring_mask;
    struct io_uring_sqe *sqe = &s->sqes[index];
    sqe->fd = file_fd;
    sqe->flags = 0;
    sqe->opcode = IORING_OP_READV;
    sqe->addr = (unsigned long) fi->iovecs;
    sqe->len = blocks;
    sqe->off = 0;
    sqe->user_data = (unsigned long long) fi;
    sring->array[index] = index;
    tail = next_tail;

    /* Update the tail so the kernel can see it. */
    if(*sring->tail != tail) {
        *sring->tail = tail;
        write_barrier();
    }

    /*
     * Tell the kernel we have submitted events with the io_uring_enter() system
     * call. We also pass in the IOURING_ENTER_GETEVENTS flag which causes the
     * io_uring_enter() call to wait until min_complete events (the 3rd param)
     * complete.
     * */
    int ret =  io_uring_enter(s->ring_fd, 1,1,
            IORING_ENTER_GETEVENTS);
    if(ret < 0) {
        perror("io_uring_enter");
        return 1;
    }

    return 0;
}

int main(int argc, char *argv[]) {
    struct submitter *s;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);
        return 1;
    }

    s = malloc(sizeof(*s));
    if (!s) {
        perror("malloc");
        return 1;
    }
    memset(s, 0, sizeof(*s));

    if(app_setup_uring(s)) {
        fprintf(stderr, "Unable to setup uring!\n");
        return 1;
    }

    for (int i = 1; i < argc; i++) {
        if(submit_to_sq(argv[i], s)) {
            fprintf(stderr, "Error reading file\n");
            return 1;
        }
        read_from_cq(s);
    }

    return 0;
}
```

> 有点过于高深了，就不自己手敲了。简单翻译一下。

#### The initial setup

从 main() 开始，我们调用 `app_setup_uring()` ，为我们做一些使用 io_uring 的必要准备。首先调用 syscall `io_uring_setup()` 并提供我们需要的队列长度和 `io_uring_params` 的实例，全部设置为 0. 调用返回时，内核将会向这个结构体中填充值，`io_uring_params` 长得像

```c
struct io_uring_params {
  __u32 sq_entries;
  __u32 cq_entries;
  __u32 flags;
  __u32 sq_thread_cpu;
  __u32 sq_thread_idle;
  __u32 resv[5];
  struct io_sqring_offsets sq_off;
  struct io_cqring_offsets cq_off;
};
```

你唯一能指定的只有 flags 字段，但在这里，我们并不想传递什么。同时，这个例子里我们串行处理请求，不使用任何并行 I/O，因为这个例子的目的主要是理解 `io_uring`。我们设置队列长度为1.

`io_uring_setup()` 的返回值是 *文件描述符 fd*，其他的 io_uring_param 结构会被之后使用 *mmap()* 来映射到用户态的两个环形缓冲，以及一个 SQEs 数组。我们现在关注 mmap() 的部分

```cpp
    /* 映射到 SQ 和 CQ 的缓冲区中。
     * 旧版内核可能只能映射 SQ。
     * */
    sq_ptr = mmap(0, sring_sz, PROT_READ | PROT_WRITE, 
            MAP_SHARED | MAP_POPULATE,
            s->ring_fd, IORING_OFF_SQ_RING);
    if (sq_ptr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    if (p.features & IORING_FEAT_SINGLE_MMAP) {
        cq_ptr = sq_ptr;
    } else {
        /* 在旧版内核中再手动映射 CQ */
        cq_ptr = mmap(0, cring_sz, PROT_READ | PROT_WRITE, 
                MAP_SHARED | MAP_POPULATE,
                s->ring_fd, IORING_OFF_CQ_RING);
        if (cq_ptr == MAP_FAILED) {
            perror("mmap");
            return 1;
        }
    }

    /* 映射 SQEs 数组 */
    s->sqes = mmap(0, p.sq_entries * sizeof(struct io_uring_sqe),
            PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE,
            s->ring_fd, IORING_OFF_SQES);
```

我们保存 `app_io_sq_ring` 和 `app_io_cq_ring` 的重要信息方便以后进行引用。我们分别将两个环形缓冲区映射为 submission 和 completion，你可能会奇怪第二个 mmap 是干什么的？completion queue 环是直接索引 CQEs 数组，而 submission queue 环有一个间接的数组。submission 的环形缓冲保存的是进入按顺序保存了 SQEs 索引的数组的索引。

这对于一些将提交请求嵌入到内部数据结构的程序比较有用，这种设计允许他们一次提交多个项目，同时允许他们使用 io_uring 更简单。

注意：5.4 内核以及以上一次 mmap 就能映射 submission 和 completion 队列。

#### 了解 shared ring buffer

常规的编程中，我们习惯用很清晰的接口来处理用户态和内核态：system call。然而，syscall 具有比较大的开销，所以一些像 io_uring 的高性能接口就想要尽可能避免它们。io_uring 允许我们 batch 许多 IO 请求，然后通过一次调用 io_uring_enter() 解决问题，甚至可以使用 polling mode，都不需要调用 io_uring_enter()。

在用户空间中读取或者更新 shared ring buffer 时，有一点需要注意，当读取时，你看到的是最新的数据；在更新后，你正在 flushing 或者说 syncing 写入，这样内核才能看到你的更新。这是因为编译器和CPU 都可以重排序 读写指令。如果发生在同一个CPU上，这个一般不是问题，但对于 io_uring 这种需要在用户态和内核态切换上下文的情况，有可能在不同 CPU 上运行。你需要在读之前确保之前的写入可见。或者，当你在 SQE 中写入信息并更新到 submission ring buffer 尾部后，确保你对数据的写入发生在插入他被插入之前。

如果写入没有被排序，那么内核可能只看到尾部更新，读取 SQE 时里面的数据却不正确。**在 polling mode 下，这个就真的是个问题了**。这是因为 CPUs 和编译器对于读写操作的重排有利于优化。

#### 读取 CQE

先说 completion side，因为比较简单。这里是必须要讨论的，因为要考虑内存序的问题。对于 completion events，内核向缓冲区添加 CQEs 并且更新其尾部，我们在用户空间内读的是头部。就像任何的环形缓冲一样，如果 head 和 tail 相等，那就意味着缓冲区为空。我们看一下下面的代码：

```cpp
unsigned head;
head = cqring->head;
read_barrier(); /* ensure previous writes are visible */
if (head != cqring->tail) {
    /* There is data available in the ring buffer */
    struct io_uring_cqe *cqe;
    unsigned index;
    index = head & (cqring->mask);
    cqe = &cqring->cqes[index];
    /* process completed cqe here */
     ...
    /* we've now consumed this entry */
    head++;
}
cqring->head = head;
write_barrier();
```

为了获取头部的索引，应用程序需要 mask 头和缓冲区大小的mask。记住，上面的任何一行都可能在上下文切换后运行。所以，在比较之前，我们需要 read_barrier，这样如果内核更新了尾部，我们可以在 if 中读取到他。一旦我们获取了 CQE 并对他进行处理，我们就要更新头来让内核知道我们从缓冲区中消费了一个 entry。最终的 write_barrier 保证了我们的更新可见。

#### 提交

提交与读取 completion 相反。我们向缓冲区尾部添加 entry，内核从头部读取。

```cpp
struct io_uring_sqe *sqe;
unsigned tail, index;
tail = sqring->tail;
index = tail & (*sqring->ring_mask);
sqe = &sqring→sqes[index];
/* this function call fills in the SQE details for this IO request */
app_init_io(sqe);
/* fill the SQE index into the SQ ring array */
sqring->array[index] = index;
tail++;
write_barrier();
sqring->tail = tail;
write_barrier();
```

在上面的代码中，`app_init_io()` 会填充提交信息的细节。在 tail 更新前，我们需要 write_barrier 来保证之前的写排序在我们提交之前。之后我们更新尾部，还要调用 write_barrier 来保证更新可见。We’re lining up our ducks here.

> 这部分看不懂可以自行了解下 CPU 指令重排，以及内存序。在 C++ 中即 std::memory_order。

## Cat liburing

### 代码实现

可以看出，使用 io_uring 来构建一个读取文件的程序似乎不是很简单。甚至比普通的同步代码量还要多。但如果你分析了 `cat_uring` 代码，你可能会看出那些代码大部分都是模板。我们都需要了解底层 io_uring 的 API 来便于我们理解细节，但如果你要在你的程序中使用 io_uring，还是应该使用 `liburing` ，也就是其封装版。

我们现在来看看 `liburing` 的版本跟 `cat_uring` 有多相似

```cpp
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <liburing.h>
#include <stdlib.h>
#define QUEUE_DEPTH 1
#define BLOCK_SZ    1024

struct file_info {
    off_t file_sz;
    struct iovec iovecs[];
};

/**
 * 返回传入的 fd 大小。可以处理常规文件和硬件驱动。
 */
off_t get_file_size(int fd) {
    struct stat st;

    if (fstat(fd, &st) < 0) {
        perror("fstat");
        return -1;
    }

    if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;
        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0) {
            perror("ioctl");
            return -1;
        }
        return bytes;
    } else if (S_ISREG(st.st_mode)) {
        return st.st_size;
    }
    return -1;
}

/**
 * 向 stdout 输出长度为 len 的字符串。
 * 我们使用 buffered 输出提高效率。
 * 因此我们需要一个一个的输出字符。
 */
void output_to_console(char* buf, int len) {
    while (len--) {
        fputc(*buf++, stdout);
    }
}

/**
 * 等待 completion 可用，从 readv 中获取数据并且打印
 * 
 */
int get_completion_and_print(struct io_uring* ring) {
    struct io_uring_cqe* cqe;
    int ret = io_uring_wait_cqe(ring, &cqe);
    if (ret < 0) {
        perror("io_uring_wait_cqe");
        return 1;
    }

    if (cqe->res < 0) {
        fprintf(stderr, "Async readv failed.\n");
        return 1;
    }

    struct file_info* fi = io_uring_cqe_get_data(cqe);
    int blks = (int) fi->file_sz / BLOCK_SZ;
    if (fi->file_sz % BLOCK_SZ) blks++;
    for (int i = 0; i < blks; ++i) {
        output_to_console(fi->iovecs[i].iov_base, fi->iovecs[i].iov_len);
    }
    io_uring_cqe_seen(ring, cqe);
    return 0;
}

/**
 * 通过 liburing 来提交 readv 请求
 * 
 */
int submit_read_request(char* file_path, struct io_uring* ring) {
    int file_fd = open(file_path, O_RDONLY);
    if (file_fd < 0) {
        perror("open");
        return 1;
    }
    off_t file_sz = get_file_size(file_fd);
    off_t bytes_reamining = file_sz;
    off_t offset = 0;
    int current_block = 0;
    int blks = (int) file_sz / BLOCK_SZ;
    if (file_sz % BLOCK_SZ) blks++;
    struct file_info* fi = malloc(sizeof(*fi) + (sizeof(struct iovec) * blks));

    /**
     * 对于每个 block 我们都需要读，分配一个 iovec struct，
     * 代表 iovecs array 的索引。
     * 该 array 也会作为传入 submission 的一部分。
     * 如果你不理解这个的话，你需要了解一下 readv() writev() 是怎么工作的。
     */
     while (bytes_reamining) {
        off_t bytes_to_read = bytes_reamining;
        if (bytes_to_read > BLOCK_SZ) {
            bytes_to_read = BLOCK_SZ;
        }
        offset += bytes_to_read;
        fi->iovecs[current_block].iov_len = bytes_to_read;

        void* buf;
        if (posix_memalign(&buf, BLOCK_SZ, BLOCK_SZ)) {
            perror("posix_memalign");
            return 1;
        }
        fi->iovecs[current_block].iov_base = buf;

        current_block++;
        bytes_reamining -= bytes_to_read;
     }

     fi->file_sz = file_sz;

     /* 获取 SQE */
     struct io_uring_sqe* sqe = io_uring_get_sqe(ring);

     /* 设置 readv 操作 */
     io_uring_prep_readv(sqe, file_fd, fi->iovecs, blks, 0);

     /* 设置 user data */
     io_uring_sqe_set_data(sqe, fi);

     /* 最终，提交 */
     io_uring_submit(ring);

     return 0;
}

int main(int argc, char* argv[]) {
    struct io_uring ring;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s [file name] <[file name] ...>\n",
                argv[0]);
        return 1;
    }

    /* 初始化 io_uring */
    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);

    for (int i = 1; i < argc; ++i) {
        int ret = submit_read_request(argv[i], &ring);
        if (ret) {
            fprintf(stderr, "Error reading file: %s\n", argv[i]);
            return 1;
        }
        get_completion_and_print(&ring);
    }

    /* 调用清理函数 */
    io_uring_queue_exit(&ring);
    return 0;
}
```

对比一下他们的行数：

- 常规 cat：120行
- io_uring 原生：360行
- liburing：160 行

我们来针对关键部分逻辑快速过一下代码：

首先我们初始化 `io_uring`

```c
io_uring_queue_init(QUEUE_DEPTH, &ring, 0);
```

在函数 `submit_read_request()` 中，我们获得 SQE，并且准备一个 readv 请求之后再提交

```cpp
    /* Get an SQE */
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    /* Setup a readv operation */
    io_uring_prep_readv(sqe, file_fd, fi->iovecs, blocks, 0);
    /* Set user data */
    io_uring_sqe_set_data(sqe, fi);
    /* Finally, submit the request */
    io_uring_submit(ring);
```

等待 completion event，并且获取我们之前提交的请求返回的用户数据：

```cpp
    struct io_uring_cqe *cqe;
    int ret = io_uring_wait_cqe(ring, &cqe);
    struct file_info *fi = io_uring_cqe_get_data(cqe);
```

对比原生 API 实在是太简单了。

### 译者的总结

流程：

初始化 io_uring -> 

提交请求 ->

1. 初始化 SQE
2. 指明 io_uring 对该 SQE 的操作
3. 指明 io_uring 存储结果的位置（user data）
4. 提交该 ring 的请求

获取 completion

1. 声明 CQE, 等待该 CQE 被操作完成
2. 从 CQE 中获取信息
3. 将该 CQE 标记为已被消费

| 步骤                            | 函数API                                                      |
| ------------------------------- | ------------------------------------------------------------ |
| 初始化 io_uring                 | `int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);` |
| 初始化 SQE                      | struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring); |
| 指明 io_uring 对该 SQE 的操作   | io_uring_prep_action_name                                    |
| 指明 io_uring 存储结果的位置    | void io_uring_sqe_set_data(struct io_uring_sqe *sqe, void *user_data); |
| 提交该 ring 的请求              | int io_uring_submit(struct io_uring *ring);                  |
| 声明 CQE, 等待该 CQE 被操作完成 | int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr); |
| 从 CQE 中获取信息               | void *io_uring_cqe_get_data(struct io_uring_cqe *cqe);       |
| 将该 CQE 标记为已被消费         | void io_uring_cqe_seen(struct io_uring *ring, struct io_uring_cqe *cqe); |

参考资料：

[Arch 手册](https://man.archlinux.org/listing/extra/liburing/)

## 异步编程的麻烦

如果你只需要编写每小时处理几千甚至几百请求的程序，那你完全无需考虑异步 IO。使用线程池为基础的架构已经足够了

>  [thread-pool based architectures will serve you just fine](https://unixism.net/2019/04/28/linux-applications-performance-introduction/) 

但如果你需要每小时处理百万请求，你可能需要关注一下异步编程。异步编程通过将 I/O 在一个线程上执行而避免了操作系统的线程/进程上下文切换开销。[读这里了解更多](https://unixism.net/2019/04/28/linux-applications-performance-introduction/)，了解不同的程序是如何构建 web server的。

## 常规文件的麻烦

linux 上的异步编程，特别是 sockets 使用的是 select(), poll(), epoll()。这些方法对于 socket 比较有效，对于常规文件没什么效果。如果你构建一个 web server 或者是 caching server，那么处理许多跟并发、存储速度有关的常规文件请求，访问文件会阻塞并且降低你服务器的速度。为了解决这个问题，libuv 使用了分开了处理文件 I/O 和其他事情的线程。

正如其[文档](http://docs.libuv.org/en/v1.x/design.html)中所说：

> Unlike network I/O, there are no platform-specific file I/O primitives libuv could rely on, so the current approach is to run blocking file I/O operations in a thread pool.
>
> libuv currently uses a global thread pool on which all loops can queue work. 3 types of operations are currently run on this pool:
>
> – File system operations
>
> – DNS functions (getaddrinfo and getnameinfo)
>
> – User specified code via uv_queue_work()

有了 io_uring，所有的操作，不管是发生在 socket 还是常规文件，都有了统一的解决方案。不需要用户再想其他的技巧来解决这些问题了。读 [this](https://blog.libtorrent.org/2012/10/asynchronous-disk-io/) 了解更多异步 IO 和文件 IO 的关系。

## 下一步？

第一部分文章，我们简单看了如何构建一个和 Unix 系 cat 命令相同的程序，使用了三种方法：同步，io_uring 原生 API，liburing。然而，我们在这里限制了一次只处理一个请求。我们的实现同时可以读取许多文件，但是提交到 io_uring 后，我们等待其就绪，最后再将下一个文件移入处理。我们故意这么设计，好让我们抓住 io_uring 工作的重点。**但 io_uring 真正的一次处理多个请求的威力，我们会在下一篇文章中再写**。我们会编写一个复制文件的程序，让 io_uring 一次性接受多个请求，一个文件一个 block。

## 原作者信息

My name is Shuveb Hussain and I’m the author of this Linux-focused blog. You can [follow me on Twitter](https://twitter.com/shuveb) where I post tech-related content mostly focusing on Linux, performance, scalability and cloud technologies.