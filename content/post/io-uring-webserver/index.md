---
title: io_uring P3 - 实现 webserver
description: 一起学习使用 liburing 在网络编程中的使用
slug: io_uring-web
date: 2024-02-26 00:00:00+0000
image: 
categories:
    - io-uring
    - techs
    - unfinished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

本文是文章 [io_uring By Example: Part 3 - A Web Server with io_uring - Unixism](https://unixism.net/2020/04/io-uring-by-example-part-3-a-web-server-with-io-uring/) 的翻译与总结。

同样只摘录重要的部分。

这一篇文章使用 liburing 来实现了一个 webserver，我们可以看看 io_uring 在网络编程中的应用

## 简介

这一次，我们来看看 accept 是怎么作用于 io_uring 的，配合 readv/writev，你已经可以编写一个简单的 web server 了。这篇文章的代码类似于 [ZeroHTTPd](https://unixism.net/2019/04/linux-applications-performance-introduction/), 我会使用 io_uring 来实现。

## web server

```cpp
#include <stdio.h>
#include <netinet/in.h>
#include <string.h>
#include <ctype.h>
#include <unistd.h>
#include <stdlib.h>
#include <signal.h>
#include <liburing.h>
#include <sys/stat.h>
#include <fcntl.h>

#define SERVER_STRING           "Server: zerohttpd/0.1\r\n"
#define DEFAULT_SERVER_PORT     8000
#define QUEUE_DEPTH             256
#define READ_SZ                 8192

#define EVENT_TYPE_ACCEPT       0
#define EVENT_TYPE_READ         1
#define EVENT_TYPE_WRITE        2

struct request {
    int event_type;
    int iovec_count;
    int client_socket;
    struct iovec iov[];
};

struct io_uring ring;

const char *unimplemented_content = \
        "HTTP/1.0 400 Bad Request\r\n"
        "Content-type: text/html\r\n"
        "\r\n"
        "<html>"
        "<head>"
        "<title>ZeroHTTPd: Unimplemented</title>"
        "</head>"
        "<body>"
        "<h1>Bad Request (Unimplemented)</h1>"
        "<p>Your client sent a request ZeroHTTPd did not understand and it is probably not your fault.</p>"
        "</body>"
        "</html>";

const char *http_404_content = \
        "HTTP/1.0 404 Not Found\r\n"
        "Content-type: text/html\r\n"
        "\r\n"
        "<html>"
        "<head>"
        "<title>ZeroHTTPd: Not Found</title>"
        "</head>"
        "<body>"
        "<h1>Not Found (404)</h1>"
        "<p>Your client is asking for an object that was not found on this server.</p>"
        "</body>"
        "</html>";

/*
 * Utility function to convert a string to lower case.
 * */

void strtolower(char *str) {
    for (; *str; ++str)
        *str = (char)tolower(*str);
}
/*
 One function that prints the system call and the error details
 and then exits with error code 1. Non-zero meaning things didn't go well.
 */
void fatal_error(const char *syscall) {
    perror(syscall);
    exit(1);
}

/*
 * Helper function for cleaner looking code.
 * */

void *zh_malloc(size_t size) {
    void *buf = malloc(size);
    if (!buf) {
        fprintf(stderr, "Fatal error: unable to allocate memory.\n");
        exit(1);
    }
    return buf;
}

/*
 * This function is responsible for setting up the main listening socket used by the
 * web server.
 * */

int setup_listening_socket(int port) {
    int sock;
    struct sockaddr_in srv_addr;

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        fatal_error("socket()");

    int enable = 1;
    if (setsockopt(sock,
                   SOL_SOCKET, SO_REUSEADDR,
                   &enable, sizeof(int)) < 0)
        fatal_error("setsockopt(SO_REUSEADDR)");


    memset(&srv_addr, 0, sizeof(srv_addr));
    srv_addr.sin_family = AF_INET;
    srv_addr.sin_port = htons(port);
    srv_addr.sin_addr.s_addr = htonl(INADDR_ANY);

    /* We bind to a port and turn this socket into a listening
     * socket.
     * */
    if (bind(sock,
             (const struct sockaddr *)&srv_addr,
             sizeof(srv_addr)) < 0)
        fatal_error("bind()");

    if (listen(sock, 10) < 0)
        fatal_error("listen()");

    return (sock);
}

int add_accept_request(int server_socket, struct sockaddr_in *client_addr,
                       socklen_t *client_addr_len) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_accept(sqe, server_socket, (struct sockaddr *) client_addr,
                         client_addr_len, 0);
    struct request *req = malloc(sizeof(*req));
    req->event_type = EVENT_TYPE_ACCEPT;
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);

    return 0;
}

int add_read_request(int client_socket) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    struct request *req = malloc(sizeof(*req) + sizeof(struct iovec));
    req->iov[0].iov_base = malloc(READ_SZ);
    req->iov[0].iov_len = READ_SZ;
    req->event_type = EVENT_TYPE_READ;
    req->client_socket = client_socket;
    memset(req->iov[0].iov_base, 0, READ_SZ);
    /* Linux kernel 5.5 has support for readv, but not for recv() or read() */
    io_uring_prep_readv(sqe, client_socket, &req->iov[0], 1, 0);
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);
    return 0;
}

int add_write_request(struct request *req) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    req->event_type = EVENT_TYPE_WRITE;
    io_uring_prep_writev(sqe, req->client_socket, req->iov, req->iovec_count, 0);
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);
    return 0;
}

void _send_static_string_content(const char *str, int client_socket) {
    struct request *req = zh_malloc(sizeof(*req) + sizeof(struct iovec));
    unsigned long slen = strlen(str);
    req->iovec_count = 1;
    req->client_socket = client_socket;
    req->iov[0].iov_base = zh_malloc(slen);
    req->iov[0].iov_len = slen;
    memcpy(req->iov[0].iov_base, str, slen);
    add_write_request(req);
}

/*
 * When ZeroHTTPd encounters any other HTTP method other than GET or POST, this function
 * is used to inform the client.
 * */

void handle_unimplemented_method(int client_socket) {
    _send_static_string_content(unimplemented_content, client_socket);
}

/*
 * This function is used to send a "HTTP Not Found" code and message to the client in
 * case the file requested is not found.
 * */

void handle_http_404(int client_socket) {
    _send_static_string_content(http_404_content, client_socket);
}

/*
 * Once a static file is identified to be served, this function is used to read the file
 * and write it over the client socket using Linux's sendfile() system call. This saves us
 * the hassle of transferring file buffers from kernel to user space and back.
 * */

void copy_file_contents(char *file_path, off_t file_size, struct iovec *iov) {
    int fd;

    char *buf = zh_malloc(file_size);
    fd = open(file_path, O_RDONLY);
    if (fd < 0)
        fatal_error("read");

    /* We should really check for short reads here */
    int ret = read(fd, buf, file_size);
    if (ret < file_size) {
        fprintf(stderr, "Encountered a short read.\n");
    }
    close(fd);

    iov->iov_base = buf;
    iov->iov_len = file_size;
}

/*
 * Simple function to get the file extension of the file that we are about to serve.
 * */

const char *get_filename_ext(const char *filename) {
    const char *dot = strrchr(filename, '.');
    if (!dot || dot == filename)
        return "";
    return dot + 1;
}

/*
 * Sends the HTTP 200 OK header, the server string, for a few types of files, it can also
 * send the content type based on the file extension. It also sends the content length
 * header. Finally it send a '\r\n' in a line by itself signalling the end of headers
 * and the beginning of any content.
 * */

void send_headers(const char *path, off_t len, struct iovec *iov) {
    char small_case_path[1024];
    char send_buffer[1024];
    strcpy(small_case_path, path);
    strtolower(small_case_path);

    char *str = "HTTP/1.0 200 OK\r\n";
    unsigned long slen = strlen(str);
    iov[0].iov_base = zh_malloc(slen);
    iov[0].iov_len = slen;
    memcpy(iov[0].iov_base, str, slen);

    slen = strlen(SERVER_STRING);
    iov[1].iov_base = zh_malloc(slen);
    iov[1].iov_len = slen;
    memcpy(iov[1].iov_base, SERVER_STRING, slen);

    /*
     * Check the file extension for certain common types of files
     * on web pages and send the appropriate content-type header.
     * Since extensions can be mixed case like JPG, jpg or Jpg,
     * we turn the extension into lower case before checking.
     * */
    const char *file_ext = get_filename_ext(small_case_path);
    if (strcmp("jpg", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/jpeg\r\n");
    if (strcmp("jpeg", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/jpeg\r\n");
    if (strcmp("png", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/png\r\n");
    if (strcmp("gif", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: image/gif\r\n");
    if (strcmp("htm", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/html\r\n");
    if (strcmp("html", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/html\r\n");
    if (strcmp("js", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: application/javascript\r\n");
    if (strcmp("css", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/css\r\n");
    if (strcmp("txt", file_ext) == 0)
        strcpy(send_buffer, "Content-Type: text/plain\r\n");
    slen = strlen(send_buffer);
    iov[2].iov_base = zh_malloc(slen);
    iov[2].iov_len = slen;
    memcpy(iov[2].iov_base, send_buffer, slen);

    /* Send the content-length header, which is the file size in this case. */
    sprintf(send_buffer, "content-length: %ld\r\n", len);
    slen = strlen(send_buffer);
    iov[3].iov_base = zh_malloc(slen);
    iov[3].iov_len = slen;
    memcpy(iov[3].iov_base, send_buffer, slen);

    /*
     * When the browser sees a '\r\n' sequence in a line on its own,
     * it understands there are no more headers. Content may follow.
     * */
    strcpy(send_buffer, "\r\n");
    slen = strlen(send_buffer);
    iov[4].iov_base = zh_malloc(slen);
    iov[4].iov_len = slen;
    memcpy(iov[4].iov_base, send_buffer, slen);
}

void handle_get_method(char *path, int client_socket) {
    char final_path[1024];

    /*
     If a path ends in a trailing slash, the client probably wants the index
     file inside of that directory.
     */
    if (path[strlen(path) - 1] == '/') {
        strcpy(final_path, "public");
        strcat(final_path, path);
        strcat(final_path, "index.html");
    }
    else {
        strcpy(final_path, "public");
        strcat(final_path, path);
    }

    /* The stat() system call will give you information about the file
     * like type (regular file, directory, etc), size, etc. */
    struct stat path_stat;
    if (stat(final_path, &path_stat) == -1) {
        printf("404 Not Found: %s (%s)\n", final_path, path);
        handle_http_404(client_socket);
    }
    else {
        /* Check if this is a normal/regular file and not a directory or something else */
        if (S_ISREG(path_stat.st_mode)) {
            struct request *req = zh_malloc(sizeof(*req) + (sizeof(struct iovec) * 6));
            req->iovec_count = 6;
            req->client_socket = client_socket;
            send_headers(final_path, path_stat.st_size, req->iov);
            copy_file_contents(final_path, path_stat.st_size, &req->iov[5]);
            printf("200 %s %ld bytes\n", final_path, path_stat.st_size);
            add_write_request( req);
        }
        else {
            handle_http_404(client_socket);
            printf("404 Not Found: %s\n", final_path);
        }
    }
}

/*
 * This function looks at method used and calls the appropriate handler function.
 * Since we only implement GET and POST methods, it calls handle_unimplemented_method()
 * in case both these don't match. This sends an error to the client.
 * */

void handle_http_method(char *method_buffer, int client_socket) {
    char *method, *path, *saveptr;

    method = strtok_r(method_buffer, " ", &saveptr);
    strtolower(method);
    path = strtok_r(NULL, " ", &saveptr);

    if (strcmp(method, "get") == 0) {
        handle_get_method(path, client_socket);
    }
    else {
        handle_unimplemented_method(client_socket);
    }
}

int get_line(const char *src, char *dest, int dest_sz) {
    for (int i = 0; i < dest_sz; i++) {
        dest[i] = src[i];
        if (src[i] == '\r' && src[i+1] == '\n') {
            dest[i] = '\0';
            return 0;
        }
    }
    return 1;
}

int handle_client_request(struct request *req) {
    char http_request[1024];
    /* Get the first line, which will be the request */
    if(get_line(req->iov[0].iov_base, http_request, sizeof(http_request))) {
        fprintf(stderr, "Malformed request\n");
        exit(1);
    }
    handle_http_method(http_request, req->client_socket);
    return 0;
}

void server_loop(int server_socket) {
    struct io_uring_cqe *cqe;
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);

    add_accept_request(server_socket, &client_addr, &client_addr_len);

    while (1) {
        int ret = io_uring_wait_cqe(&ring, &cqe);
        struct request *req = (struct request *) cqe->user_data;
        if (ret < 0)
            fatal_error("io_uring_wait_cqe");
        if (cqe->res < 0) {
            fprintf(stderr, "Async request failed: %s for event: %d\n",
                    strerror(-cqe->res), req->event_type);
            exit(1);
        }

        switch (req->event_type) {
            case EVENT_TYPE_ACCEPT:
                add_accept_request(server_socket, &client_addr, &client_addr_len);
                add_read_request(cqe->res);
                free(req);
                break;
            case EVENT_TYPE_READ:
                if (!cqe->res) {
                    fprintf(stderr, "Empty request!\n");
                    break;
                }
                handle_client_request(req);
                free(req->iov[0].iov_base);
                free(req);
                break;
            case EVENT_TYPE_WRITE:
                for (int i = 0; i < req->iovec_count; i++) {
                    free(req->iov[i].iov_base);
                }
                close(req->client_socket);
                free(req);
                break;
        }
        /* Mark this request as processed */
        io_uring_cqe_seen(&ring, cqe);
    }
}

void sigint_handler(int signo) {
    printf("^C pressed. Shutting down.\n");
    io_uring_queue_exit(&ring);
    exit(0);
}

int main() {
    int server_socket = setup_listening_socket(DEFAULT_SERVER_PORT);

    signal(SIGINT, sigint_handler);
    io_uring_queue_init(QUEUE_DEPTH, &ring, 0);
    server_loop(server_socket);

    return 0;
}
```

## 程序结构

首先，main() 调用 setup_listening_socket() 来监听一个端口。但我们不用 accept() 来监听端口，我们使用的是 io_uring 提供的。

程序的核心自然是 server_loop() ，用来提交请求给 io_uring，并且等待 CQE 之后对他们进行处理。我们看看

```cpp
void server_loop(int server_socket) {
    struct io_uring_cqe *cqe;
    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);

    add_accept_request(server_socket, &client_addr, &client_addr_len);

    while (1) {
        int ret = io_uring_wait_cqe(&ring, &cqe);
        struct request *req = (struct request *) cqe->user_data;
        if (ret < 0)
            fatal_error("io_uring_wait_cqe");
        if (cqe->res < 0) {
            fprintf(stderr, "Async request failed: %s for event: %d\n",
                    strerror(-cqe->res), req->event_type);
            exit(1);
        }

        switch (req->event_type) {
            case EVENT_TYPE_ACCEPT:
                add_accept_request(server_socket, &client_addr, &client_addr_len);
                add_read_request(cqe->res);
                free(req);
                break;
            case EVENT_TYPE_READ:
                if (!cqe->res) {
                    fprintf(stderr, "Empty request!\n");
                    break;
                }
                handle_client_request(req);
                free(req->iov[0].iov_base);
                free(req);
                break;
            case EVENT_TYPE_WRITE:
                for (int i = 0; i < req->iovec_count; i++) {
                    free(req->iov[i].iov_base);
                }
                close(req->client_socket);
                free(req);
                break;
        }
        /* Mark this request as processed */
        io_uring_cqe_seen(&ring, cqe);
    }   
}
```

在 while 循环中，我们提交 accept 请求，通过调用 `add_accept_request()`。

```cpp
int add_accept_request(int server_socket, struct sockaddr_in *client_addr,
                       socklen_t *client_addr_len) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_accept(sqe, server_socket, (struct sockaddr *) client_addr,
                         client_addr_len, 0);
    struct request *req = malloc(sizeof(*req));
    req->event_type = EVENT_TYPE_ACCEPT;
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);
    return 0;
}
```

我们先获取一个 SQE，然后准备使用 `io_uring_prep_accept()` 来提交 accept 请求。使用结构体 `request` 来跟踪我们的每一个请求。这个 req 实例包含了每个请求的从一个状态到另一个状态时的上下文。

```cpp
struct request {
    int event_type;
    int iovec_count;
    int client_socket;
    struct iovec iov[];
};
```

客户端会经历三个请求，此外这个结构体可以保存足够的信息来 handle 这些请求并转移这个状态。三个请求的顺序：

Accepted -> Request read -> Response written

我们来看看一旦 accept 完成，在 switch 中的 completion side 会发生什么：

```cpp
           case EVENT_TYPE_ACCEPT:
                add_accept_request(server_socket, &client_addr, &client_addr_len);
                add_read_request(cqe->res);
                free(req);
                break;
```

我们重新提交一个 accept 请求，因为我们之前已经处理过一个了。否则我们的程序就不能再接受来自其他客户端的 accept。之后就可以调用 read 请求使用 readv 来读取客户端的 HTTP 请求。

这里有很多要说的：我们看使用 read，但直到 kernel 5.6 才支持该函数，在撰写本文时，他是最新的稳定版本。

使用 readv/writev 也能允许我们实现常规操作，尤其是 buffer 管理。现在我们来看 `add_read_request()`

```cpp
int add_read_request(int client_socket) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    struct request *req = malloc(sizeof(*req) + sizeof(struct iovec));
    req->iov[0].iov_base = malloc(READ_SZ);
    req->iov[0].iov_len = READ_SZ;
    req->event_type = EVENT_TYPE_READ;
    req->client_socket = client_socket;
    memset(req->iov[0].iov_base, 0, READ_SZ);
    /* Linux kernel 5.5 has support for readv, but not for recv() or read() */
    io_uring_prep_readv(sqe, client_socket, &req->iov[0], 1, 0);
    io_uring_sqe_set_data(sqe, req);
    io_uring_submit(&ring);
    return 0;
}
```

> 译者：在2024年看到这篇文章，那肯定是都支持了。官方手册里也说，使用 readv 不如 read 性能高，此外 read 还支持别的操作。这里也是把 readv 当 read 用。

很直接，我们分配一个足够大的 buffer 来处理客户端请求，之后 `io_uring_prep_readv()`，对应的 handling 在 switch 语句中是：

```cpp
            case EVENT_TYPE_READ:
                if (!cqe->res) {
                    fprintf(stderr, "Empty request!\n");
                    break;
                }
                handle_client_request(req);
                free(req->iov[0].iov_base);
                free(req);
                break;
```

这里，我们调用 handle_client_request 函数处理 HTTP 请求，如果一切正常的话，客户端正在请求一个本地的文件，这一段会处理请求

```cpp
            struct request *req = zh_malloc(sizeof(*req) + (sizeof(struct iovec) * 6));
            req->iovec_count = 6;
            req->client_socket = client_socket;
            set_headers(final_path, path_stat.st_size, req->iov);
            copy_file_contents(final_path, path_stat.st_size, &req->iov[5]);
            printf("200 %s %ld bytes\n", final_path, path_stat.st_size);
            add_write_request( req);
```

`set_headers()` 包括了 5 个小的 buffer，代表 5 个不同的 iovec，最后一个包含了读取的数据，最终 调用 `add_write_request()`

```cpp
  int add_write_request(struct request *req) {
      struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
      req->event_type = EVENT_TYPE_WRITE;
      io_uring_prep_writev(sqe, req->client_socket, req->iov, req->iovec_count, 0);
      io_uring_sqe_set_data(sqe, req);
      io_uring_submit(&ring);
      return 0;
  }
```

completion side:

```cpp
              case EVENT_TYPE_WRITE:
                  for (int i = 0; i < req->iovec_count; i++) {
                      free(req->iov[i].iov_base);
                  }
                  close(req->client_socket);
                  free(req);
                  break;
```

把所有的 iovec 释放，把所有的 req 释放，并且关闭 socket，这就是完整的 HTTP client 服务。

## Conclusion

I hope you had a good time reading this article series. I sure had a lot of fun trying `io_uring` out. It is still early days for `io_uring`. It continues to gain features and gets better at performance as the days go by. I’m confident it will very quickly be adopted by various software projects.

### Source code

The full source code for all the examples is available [here at Github](https://github.com/shuveb/io_uring-by-example).

## About me

My name is Shuveb Hussain and I’m the author of this Linux-focused blog. You can [follow me on Twitter](https://twitter.com/shuveb) where I post tech-related content mostly focusing on Linux, performance, scalability and cloud technologies.

