---
title: CS144-2024 Winter - Lab 0
description: Computer Network, 2024 Winter, p0 记录
slug: cs144-p0
date: 2024-05-08 00:00:00+0000
image: 
categories:
    - cs144
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

准备做一下 CS144 系列，恰好锻炼一下我的网络编程。

由于发现网上已经有很多代码了，所以我干脆也放出来了，其实这些公开课是不应该公开代码的。

## 概述

讲了一大堆说明以及政策，大意就是实验和未来工作的难度类似，你可以从中学到东西；后面的实验通常会基于前面的实验，所以不要跳过前面的实验。

## 配置环境

这里我的是 WSL Ubuntu-24.04，其他内容照着 pdf 装就可以。这个实验要求**使用 C++20**，还挺好的，非常新。

## 手动上网

1. 手动获取网页

2. 发送 email

### 获取网页

1. 在浏览器中访问 [cs144.keithw.org/hello](http://cs144.keithw.org/hello) 然后观察结果
2. 现在将要手动获取网页信息，就和浏览器一样。
   - 在虚拟机里，`telnet cs144.keithw.org http`, 这样 telnet 会建立 TCP 连接以及一个 http 服务。
   - [按照说明书继续即可]。

```bash
❯ telnet cs144.keithw.org http
Trying 104.196.238.229...
Connected to cs144.keithw.org.
Escape character is '^]'.
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close

HTTP/1.1 200 OK
Date: Thu, 09 May 2024 12:57:56 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
Connection closed by foreign host.
```

3. **作业**：使用上面的技巧来从 url  [cs144.keithw.org/lab0/sunetid](http://cs144.keithw.org/lab0/sunetid)  中获得你自己的 id，把 `sunetid` 替换成你自己的 id，然后查看自己的私有 SUNet ID, X-Your-Code-Is: blabla. 记得把它存一下。

> 呃，发个 GET 请求，没什么好说。

---

### 发邮件

还是使用 TCP，这次换 SMTP 协议发邮件。

这一步前面似乎也没法连斯坦福的服务器，跳过好了。

**作业**：给 `cs144grader@gmail.com` 发邮件。

### 监听和连接

这一节主要使用的是 `netcat` 来让自己作为服务器。照着做即可。

`nc` 默认的实现基本可以看作为一个聊天服务器。

## 使用 OS stream socket 编写网络程序

这一节是编写一个 Socket Stream 程序，就是熟悉熟悉 Linux 的底层 socket API 而已。

提到现实生活中经常使用的 UDP datagrams（每个包上限 1500 字节）作为数据传输方式，介绍了一些 UDP 的不稳定性，以及 TCP 的来历。

lab 0 会使用 OS 内置的 TCP 编写一个叫做 `webget` 的程序，获取一个网页的内容。之后会从 0 开始实现一个 TCP。

### Getting Started - 获取初始代码

clone 个代码然后按说明来在 github 上 `private` 备份并且完成其他的构建步骤。

### 现代 C++ 介绍

这里介绍了一大堆 C++ 的内容，多用 RAII。以下需要遵守：

1. 多看 cppref
2. 不要用 `malloc()` 和 `free()`
3. 不要用 `new`/`delete`
4. 使用 smart pointers 代替裸指针
5. CS144中，不要用模板、线程、锁、虚函数
6. 避免 char* 以及 `strlen` `strcpy` 这种 C 字符串函数
7. 不要用 C 风格转换。
8. 多传 const ref
9. 除非变量需要被修改，否则多让它 const
10. 除非对象需要被修改，否则方法也 const
11. 避免全局变量，每个变量尽可能让其作用域更小
12. 在提交作业前，使用 `cmake --build build --target tidy` 来改进代码，然后使用 `cmake --build build --target format` 来格式化代码

在 git 的使用上：尽可能频繁的提交小的更改，并且用 commit message 描述清楚你的更改。

### 读 Minnow support code

为了支持这种格式的编程，Minnow 的类包装了 OS 接口，尤其是 socket fd。

**请阅读** public API，(`util/socket.hh`, `util/file_descriptor.hh`) 注意，`Socket` 是 `FileDescriptor` 的一种， 而 `TCPSocet` 是 `Socket` 的一种。

### 编写 webget

该来实现 `webget` 了~

1. 在 build 里，打开 `../apps/webget.cc`
2. 使用 `TCPScoket` `Address` 两个类在 `get_URL` 函数里完成这个程序

3. 提示

   - 注意 HTTP 每一个完整请求以 CRLF 结尾（`\r\n`）

   - 不要忘记在你的请求里写上 `Conncetion: close`，这样就能告诉服务器，在这条报文后面没有其他的请求了，然后服务器才会立刻对你进行响应。**你会发现你收到的 stream 会以 EOF 结尾。**

   - 所以要 read 多次，直到 EOF

   - 预计写个十几行代码

4. 使用 `make` 编译你的程序
5. 通过 `./apps/webget cs144.keithw.org /hello` 来测试你的程序；你当然也可以跟其他 http 服务器做实验。
6. 当它们看起来正常之后，使用 `cmake --build build --target check_webget` 来运行测试。
7. `graders` 会使用**不同的 host name** 进行测试，所以你得保证它在任何服务器上正常工作。

> 想通过测试记得把 buf 的内容输出…

## in-memory 版本的可靠字节流

功能描述：

1. 写者在输入端写入字节流，读者在输出端读取字节流；
2. 写入和读取的顺序相同；
3. 字节流是有限的：写者在结束输入后，无法再写入其他字节；读者读到 EOF 代表没有数据可读；
4. 字节流在构造时确定容量；容量指的是构造时某一方预期的最大存储量，这么做是为了限制写者写入的数量；
5. 读者读取字节并且将其弹出后，写者被允许继续写入；
6. 注意：字节流长度有限，但写者在输入结束前可以写入任意多数据，必须处理字节流长度比容量大的情况。容量为 1 的字节流也可以持有 TB 级别的字节数。
7. 单线程下运行；

> 被第六点迷惑了，我还以为是字节流保存所有输入的数据……实则只保存 available len 的数据。
>
> 那其实一点也不难，buffer 直接用 `std::vector<char>` 即可。
>
> 当然，`std::vector<char>` 无可避免的会有复制，这样实现吞吐量肯定比较低，不过我也懒得优化了。以后有空再说吧。
>
> 想优化的话你可以换成 `std::vector<std::string>` 这样的容器，`erase` 多余的内容然后直接移动就是了。`erase` 擦除末尾的元素是 `O(1)` 移动开销也很低，估计会快不少。

## 源代码

```cpp
#pragma once

#include <cstdint>
#include <deque>
#include <string>
#include <string_view>
#include <vector>

class Reader;
class Writer;

class ByteStream
{
public:
  explicit ByteStream( uint64_t capacity );

  // Helper functions (provided) to access the ByteStream's Reader and Writer interfaces
  Reader& reader();
  const Reader& reader() const;
  Writer& writer();
  const Writer& writer() const;

  void set_error() { error_ = true; };       // Signal that the stream suffered an error.
  bool has_error() const { return error_; }; // Has the stream had an error?

protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  uint64_t capacity_;
  bool error_ {};

  uint64_t bytes_buffered_ {};
  uint64_t bytes_pushed_ {};
  uint64_t bytes_popped_ {};
  bool closed_ { false };
  std::vector<char> buf_;
};

class Writer : public ByteStream
{
public:
  void push( std::string data ); // Push data to stream, but only as much as available capacity allows.
  void close();                  // Signal that the stream has reached its ending. Nothing more will be written.

  bool is_closed() const;              // Has the stream been closed?
  uint64_t available_capacity() const; // How many bytes can be pushed to the stream right now?
  uint64_t bytes_pushed() const;       // Total number of bytes cumulatively pushed to the stream
};

class Reader : public ByteStream
{
public:
  std::string_view peek() const; // Peek at the next bytes in the buffer
  void pop( uint64_t len );      // Remove `len` bytes from the buffer

  bool is_finished() const;        // Is the stream finished (closed and fully popped)?
  uint64_t bytes_buffered() const; // Number of bytes currently buffered (pushed and not popped)
  uint64_t bytes_popped() const;   // Total number of bytes cumulatively popped from stream
};

/*
 * read: A (provided) helper function thats peeks and pops up to `len` bytes
 * from a ByteStream Reader into a string;
 */
void read( Reader& reader, uint64_t len, std::string& out );
```

```cpp
#include "byte_stream.hh"

#include <algorithm>

using namespace std;

ByteStream::ByteStream( uint64_t capacity ) : capacity_( capacity ), buf_()
{
  buf_.reserve( capacity_ );
}

bool Writer::is_closed() const
{
  return closed_;
}

void Writer::push( string data )
{
  auto len = std::min( data.length(), available_capacity() );

  if ( len > 0 && !closed_ ) [[likely]] {
    std::copy( data.begin(), data.begin() + len, std::back_inserter( buf_ ) );
    bytes_buffered_ += len;
    bytes_pushed_ += len;
  }
}

void Writer::close()
{
  closed_ = true;
}

uint64_t Writer::available_capacity() const
{
  return capacity_ - bytes_buffered_;
}

uint64_t Writer::bytes_pushed() const
{
  return bytes_pushed_;
}

bool Reader::is_finished() const
{
  return !static_cast<bool>( bytes_buffered_ ) && closed_;
}

uint64_t Reader::bytes_popped() const
{
  return bytes_popped_;
}

string_view Reader::peek() const
{
  return std::string_view { buf_.data(), bytes_buffered_ };
}

void Reader::pop( uint64_t len )
{
  if ( len > bytes_buffered_ ) [[unlikely]] {
    len = bytes_buffered_;
  }
  buf_.erase( buf_.begin(), buf_.begin() + len );
  bytes_buffered_ -= len;
  bytes_popped_ += len;
}

uint64_t Reader::bytes_buffered() const
{
  return bytes_buffered_;
}
```

