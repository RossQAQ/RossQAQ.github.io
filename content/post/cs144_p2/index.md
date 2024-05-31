---
title: CS144-2024 Winter - Lab 2
description: Computer Network, 2024 Winter, p2 记录
slug: cs144-p2
date: 2024-05-29 00:00:00+0000
image: 
categories:
    - cs144
    - finished
tags: 
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
comments: false
---

这几天一直在玩博德之门，终于想起来更新博客了。这次的 TCP Receiver 比上次的 Reassembler 简单多了…虽然有两个部分。

下一次就是 TCP Sender 了，我估计可能是整个课程中最难的部分，学过 TCP 的应该都懂吧… 想想又要 close 又要处理 shutdown 又要处理重复的、未收到的部分，光想我就很累了。

## 总览

p0 实现了 ByteStream，p1 实现了 Reassembler，这个模型在 TCP 中非常有用，但都不是 TCP 协议本身的内容。在 p2，会实现 `TCPReceiver`用来接收传入的字节流。

`TCPReceiver` 从对应的 sender 接收信息（通过 `receive()` 方法），然后把它传给 `Reassembler`，写入 `ByteStream`。应用程序从这个 `ByteStream` 中读取信息，就像 p0 中你从 `TCPSocket` 中读取信息一样。

同时，`TCPReceiver` 也要生成返回给 sender 的信息，通过 `send()` 方法。这些 receiver messages 是为了告诉 sender：

1. first unassembled 字节的索引，也就是 acknowledgment number, **ackno**，也就是 receiver 需要的下一个字节的索引。
2. `ByteStream` 的剩余空间，即 **window size**

二者共同描述了 receiver 的窗口：允许 sender 发送的字节**序号的范围**。ackno 也是窗口的 left edge，ackno + window size 是 right edge（左闭右开）。

在写 `Reassembler` 和 `ByteStream` 时已经完成了大部分的算法部分了；本 lab 把以上部分变成 TCP 的一部分。最难的地方是思考 TCP 如何表示流中的每个字节 —— 通常是 sequence number。

## 开始

1. commit checkpoint 1
2. `git fetch --all` 来获取最新的代码
3. 下载 checkpoint2 需要的代码 `git merge origin/check2-startercode`
4. `cmake -S . -B build`
5. 编译 `cmake --build build`
6. 写 `check2.md`

## checkpoint 2：TCP Receiver

TCP 是基于不可靠的 datagram 上的可靠的一对具有流量控制的字节流。TCP 连接中拥有两个参与者，每个参与者都要扮演 sender 和 receiver。

TCP Receiver 要做的是从 sender 接收字节流，重组，然后决定需要发回的 ackno 以及流量控制。

> acknowledgement 代表着 receiver 需要的 **下一个** 字节的索引。sender 根据这个序号来决定是 send 还是 resend。
>
> flow control 代表 receiver 需要的字节索引的范围。sender 根据这个来获取 **被允许** 发送多少字节。

### 64位 索引转换为 32位 seq number

我们根据 TCP 协议以 TCP 的方式来表示索引。上次实现的 Reassembler 使用的是 64位 的流索引，流总是从 0 开始。64位 的索引基本可以看成永远不会溢出。在 TCP 头部中，空间比较宝贵，使用的是 32位 `sqeno` 来表示索引，这样就要复杂一些。

1. **你的实现需要包装 32位 整数**。TCP 的流可以非常长，没有规定 TCP 字节流的发送大小限制。索引到达 2^32-1 后需要从 0 开始重新计数。
2. **TCP seq number 的值开始时是随机的。**为了提高鲁棒性以及防止被旧的来自同一个 endpoint 的连接混淆，TCP 尽量保证 seq number 不会被猜出，并且不太可能重复。**所以流开始时的 seq number 不为 0，而是随机的 32位 整数，叫做 ISN**。这个数代表的是一个流的开始，即 SYN。剩下的 seq number 都从这个数字开始计数。
3. **逻辑上的建立和关闭连接都要占用一个 seq number**。SYN（流的开始）和 FIN（流的结束）的 control flags 被赋为 seq number，这些都要占用 **一个** seq number（SYN 的是 ISN）。**记住：SYN 和 FIN 并不是流本身的一部分，不是字节，它们代表的是字节流自身的开始和节数。**

这些 seqnos 在每个 TCP 段的头部传输（有两个 stream，每一个都是单向的，每个流有单独的 seqnos 以及单独的随机 ISN）。可能会分开讨论 absolute sequence number 和 stream index。永远从 0 开始，并不被包装；后者是 Reassembler 中使用的，代表流需要的下一个字节的索引，从 0 开始。

以下是举例说明，考虑一个仅有 3个 字节的字符串，'cat'，假设它的 SYN 是 seqno 2^32-2，那么有：

| element        | `SYN`    | c        | a    | t    | `FIN` |
| -------------- | -------- | -------- | ---- | ---- | ----- |
| seqno          | 2^32 - 2 | 2^32 - 1 | 0    | 1    | 2     |
| absolute seqno | 0        | 1        | 2    | 3    | 4     |
| stream index   |          | 0        | 1    | 2    |       |

在 absolute seqno 和 stream index 中转换很简单，单纯的加或减 1 即可。在 seqno 和 absolute seqno 中转换有点困难，很容易混淆。为了防止这些问题，我们把 seqno 包装为 `Warp32` 类型，将他和 absolute seqno（`uint64_t`）之间进行转换。`Wrap32` 是一个 *wrapper type*。

已经定义好了这些 helper 函数（`wrapping_integers.hh`），你需要实现 `wrapping_integers.cc`：

1. `static Warp32 Warp32::wrap( uint64_t n, Wrap32 zero_point )`

   转换 absolute seqno -> seqno。给出 absolute seqno 和 ISN（zero_point），返回 `n` 的 seqno

2. `uint64_t unwrap( Wrap32 zero_point, uint64_t checkpoint ) const`

   转换 seqno -> absolute seqno。给出 `Wrap32` 类型的 seqno，ISN，以及一个 absolute checkpoint seqno，找到最接近 checkpoint 的对应的 absolute seqno

   注意：**checkpoint** 存在的意义是任何给出的 seqno 可能对应多个 absolute seqnos。例：ISN = 0，seqno 为 17 其对应的 absolute seqno 可能为 17，也可能是 2^32 + 17,2^33 + 17 等等。checkpoint 帮助你解决这种问题， **在你的实现中，使用 first unassembled index 作为 checkpoint。**

> [!NOTE]
>
> 最简单的实现会使用 `wrapping_integers.hh` 的 helper。*wrap/unwrap 操作应该保留偏移量，两个相差 17 的 seqnos 对应两个同样相差 17 的 absolute seqnos。*
>
> **希望 `wrap` 的代码量只有 1行，`unwrap` 的代码量少于 10 行**，如果你的实现代码量比较多，你可以思考一下。 

测试方法：`cmake --build build --target check2`

> wrap 没什么说的，注意范围是 0 ~ u32max，这里面共有 u32max + 1 个数，也就是 (1UL << 32)，n 对它取余再加 zero 即可。
>
> unwrap 略复杂一点，首先你需要画一画数字轮盘，理解一下溢出时的行为，我们需要让他溢出，这样比较好算。需要求出距 checkpoint 的距离，我决定把它先 wrap 一下，顺时针和逆时针分别算出来比较。
>
> 如果是顺时针方向比较小（在轮盘中，checkpoint 在左边，raw_value_ 在右边）那么自然是 checkpoint + 距离；
>
> 如果是逆时针方向比较小（在轮盘中，raw_value_ 在左边，checkpoint 在右边）那么就是 checkpoint - 距离；
>
> 因为包装了 checkpoint，所以最终求结果时已经是相对 zero_point 的结果了。
>
> 但对于 `raw_value_ >= checkpoint` 的情况单独搞。 
>
> 10 行代码完全可以搞定。

----

### 实现 TCP Receiver

这个部分实现的是 `TCPReceiver`，它需要：

1. 接收来自 sender 的消息，并且使用 `Reassembler` 重排 `ByteStream`
2. 将 `ackno` 以及 window size 发给 sender

**我们预期 15 行左右的代码就可以完成。**

首先，我们复习一下 TCP "sender messager" 的格式，它包含了 ByteStream 的信息。

以下是从 `TCPSender` 发送给 `TCPReceiver` 的内容：

`TCPSenderMessage` 结构包含五个字段 (`minnow/util/tcp_sender_message.hh` ) ，

1. 片开始的 `seqno`。如果置 `SYN` 位，那么这就是 `SYN` 位的序号，否则就是携带子串的序号。
2. `SYN`位，如果被置位，那么该片是字节流的开始，seq 包含的是 ISN —— zero point。
3. payload：字节流的子串（可能空）。
4. `FIN`位，如果被置位，那么 payload 代表字节流的结束。
5. `RST`位，如果被置位，那么 stream 出现错误，应该关闭连接。

```cpp
 struct TCPSenderMessage
 {
	 Wrap32 seqno { 0 };
     
	 bool SYN {};
	 std::string payload {};
	 bool FIN {};
     
	 bool RST {};
     
	 // 使用了多少个 seqno
	 size_t sequence_length() const { return SYN + payload.size() + FIN; }
 };
```

`TCPReceiver` 生成自己的消息发送回 `TCPSender`:

`TCPReceiverMessage` 结构包含三个字段（`minnow/util/tcp_receiver_message.hh` ），

1. `ackno`，receiver 所需的 ***下一个*** seqno。这个字段可选，如果 `TCPReceiver` 还没有收到 ISN 的话就为空。
2. window size，从当前的 `ackno` 开始，代表 receiver 一次希望接收的 seq 个数的大小，最大 65535 （u16 max）
3. `RST` 位，如果置位，同 `TCPSender::RST`

```cpp
 struct TCPReceiverMessage
 {
 	std::optional<Wrap32> ackno {};
 	uint16_t window_size {};
 	bool RST {};
 };
```

`TCPReceiver`：

```cpp
 class TCPReceiver
 {
 public:
 	// Construct with given Reassembler
 	explicit TCPReceiver( Reassembler&& reassembler ) : reassembler_( std::move( reassembler ) )
 
   	// The TCPReceiver receives TCPSenderMessages from the peer's TCPSender.
 	void receive( TCPSenderMessage message );
 
 	// The TCPReceiver sends TCPReceiverMessages to the peer's TCPSender.
	TCPReceiverMessage send() const;

    // Access the output (only Reader is accessible non-const)
 	const Reassembler& reassembler() const { return reassembler_; }
 	Reader& reader() { return reassembler_.reader(); }
 	const Reader& reader() const { return reassembler_.reader(); }
 	const Writer& writer() const { return reassembler_.writer(); }
 
 private:
 	Reassembler reassembler_;
 };
```

#### receive()

这个方法会在每次有新的 segment 被接收时调用。该方法需要：

1. **如果有必要的话，设置 ISN。** 第一个到达的设置了 SYN 的片的 seqno 需要设置为 ISN。你可能需要这个值，因为你要一直在 32位的 seqno/ackno 和 absolute seqno 之间转换。

   > [!note]
   >
   > 注意 SYN 只是一个位而已，它可以包含数据，也可以包含 FIN。

2. **把数据传递给 `Reassembler`。**如果设置了 `FIN` 位，代表这一片的最后一个字节是整个字节流的结尾。**记住，`Reassembler` 希望流的索引从 0 开始，所以你得 unwrap seqnos。**

> 首先要思考一下这几个 seqno 的关系。
>
> `Reassembler` 所需要的 `first_index` 对应的是 `stream_index`，`TCPReceiver` 不管序号是多少，把它传给 `Reassembler` 就行。
>
> `TCPSenderMessage` 给出的 `seqno` 对应的是 `seqno`
>
> `TCPReceiverMessage` 要发送的 `ackno` 是 `seqno + 1`，类型是 `Wrap32`
>
> 收到消息，判断是否 `RST`，然后判断是否 `SYN`，是 `SYN` 的话记录 `zero_point`，要把 `message` 插入到 `Reassembler`，需要 `first_index` 以及数据。
>
> `ackno` 如何计算？ `ackno` 需要提醒 sender 重发未接收的数据。正常情况下则是 当前已经写入的字符 + 1 + 是否关闭。写入的字符 + 1 代表正常的 `ackno`，是否关闭代表是否计算 FIN 位。因为如果没有接受完，那么自然不应该计入 FIN，接收之后字节流会关闭，那自然需要计算 FIN。
>
> 
>
> `first_index` 应该如何计算？考虑以下情况：
>
> 1. 片带 `SYN`，且携带数据。此时 `first_index` 为 `unwrap(seqno) + 1`（需要记录 zero point）
> 2. 片不带 `SYN` 与 `FIN`。此时 `first_index`，即为 `unwrap(seqno)`
> 3. 片仅带 `FIN`，此时 `first_index` 也为 `unwrap(seqno)`
> 4. 片同时携带 `SYN`，`FIN，`此时 `first_index` 为 `unwrap(seqno) + 1`
>
> 记录 `zero_point`，unwrap 后 +1 就代表数据正式开始，对应 stream index(`first_index`) 为0。之后的就只需要算出unwrap 后的 `seqno` 跟其的差值即可。
>
> 
>
> 此外，需要处理索引不合法的错误数据。我目测就是 stream index 与 SYN 的序号相同时不合法。
>
> 在 SYN 之前的数据一律丢弃，也就是说需要知道当前连接的状态。
>
> RST 在错误时发生，错误指 ByteStream 中出现错误（`has_error()`）或者报文中收到 RST 位，这时候要手动设置 ByteStream 的 `set_error()`。
>
> 但最终我也没在 15 行内完成任务……用了25行。

## 源代码

```cpp
// wrapping_integers.cc
#include "wrapping_integers.hh"

using namespace std;

Wrap32 Wrap32::wrap( uint64_t n, Wrap32 zero_point )
{
  return Wrap32 { zero_point + static_cast<uint32_t>( n % ( 1UL << 32 ) ) };
}

uint64_t Wrap32::unwrap( Wrap32 zero_point, uint64_t checkpoint ) const
{
  if ( raw_value_ >= checkpoint ) {
    return raw_value_ - zero_point.raw_value_;
  }
  auto wrapped_checkpoint = Wrap32::wrap( checkpoint, zero_point ).raw_value_;
  uint32_t counter_clockwise = wrapped_checkpoint - raw_value_;
  uint32_t clockwise = raw_value_ - wrapped_checkpoint;
  if ( counter_clockwise < clockwise ) {
    return checkpoint - counter_clockwise;
  }
  return checkpoint + clockwise;
}
```

```cpp
// tcp_receiver.hh
#pragma once

#include "reassembler.hh"
#include "tcp_receiver_message.hh"
#include "tcp_sender_message.hh"
#include <optional>

class TCPReceiver
{
public:
  // Construct with given Reassembler
  explicit TCPReceiver( Reassembler&& reassembler ) : reassembler_( std::move( reassembler ) ) {}

  /*
   * The TCPReceiver receives TCPSenderMessages, inserting their payload into the Reassembler
   * at the correct stream index.
   */
  void receive( TCPSenderMessage message );

  // The TCPReceiver sends TCPReceiverMessages to the peer's TCPSender.
  TCPReceiverMessage send() const;

  // Access the output (only Reader is accessible non-const)
  const Reassembler& reassembler() const { return reassembler_; }
  Reader& reader() { return reassembler_.reader(); }
  const Reader& reader() const { return reassembler_.reader(); }
  const Writer& writer() const { return reassembler_.writer(); }

private:
  Reassembler reassembler_;

  std::optional<Wrap32> zero_point_ {};

  uint64_t ack_ {};

  uint64_t stream_index_ {};

  bool connected_ {};
};
```

```cpp
// tcp_receiver.cc
#include "tcp_receiver.hh"
#include <numeric>

using namespace std;

void TCPReceiver::receive( TCPSenderMessage message )
{
  if ( message.RST ) {
    reassembler_.reader().set_error();
    return;
  }
  if ( !message.SYN && !connected_ ) {
    return;
  }
  if ( message.SYN ) {
    connected_ = true;
    zero_point_ = message.seqno;
  } else {
    if ( zero_point_ == message.seqno ) {
      return;
    }
    stream_index_ = message.seqno.unwrap( *zero_point_, reassembler_.writer().bytes_pushed() ) - 1;
  }
  auto sequence_len = message.sequence_length();
  reassembler_.insert( stream_index_, std::move( message.payload ), message.FIN );
  ack_ = reassembler_.writer().bytes_pushed() + 1 + reassembler_.writer().is_closed();
  stream_index_ += sequence_len - message.FIN - message.SYN;
  this->send();
}

TCPReceiverMessage TCPReceiver::send() const
{
  return TCPReceiverMessage {
    zero_point_.has_value() ? Wrap32::wrap( ack_, *zero_point_ ) : std::optional<Wrap32> {},
    static_cast<uint16_t>( std::min( static_cast<uint64_t>( std::numeric_limits<uint16_t>::max() ),
                                     reassembler_.writer().available_capacity() ) ),
    reassembler_.writer().has_error() };
}
```

