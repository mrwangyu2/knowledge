# RFC/draft 完整中英对照翻译
# RFC/draft Complete Chinese-English Parallel Translation

---

## Title | 标题

#### 原文

Network Working Group                                          S. Holmer
Internet-Draft                                                M. Flodman
Intended status: Experimental                                  E. Sprang
Expires: April 21, 2016                                           Google
                                                        October 19, 2015


          RTP Extensions for Transport-wide Congestion Control
           draft-holmer-rmcat-transport-wide-cc-extensions-01

#### 中文

网络工作组                                                      S. Holmer
互联网草案                                                      M. Flodman
预期状态：实验性                                                E. Sprang
到期时间：2016年4月21日                                            Google
                                                        2015年10月19日


              用于传输级拥塞控制的 RTP 扩展
           draft-holmer-rmcat-transport-wide-cc-extensions-01

---

## Abstract | 摘要

#### 原文

This document proposes an RTP header extension and an RTCP message
for use in congestion control algorithms for RTP-based media flows.
It adds transport-wide packet sequence numbers and corresponding
feedback message so that congestion control can be performed on a
transport level at the send-side, while keeping the receiver dumb.

#### 中文

本文档提出了一种 RTP 头部扩展和一种 RTCP 消息，用于基于 RTP 的媒体流的拥塞控制算法。
它增加了传输级的数据包序列号和相应的反馈消息，使得拥塞控制可以在发送端的传输层面执行，
同时保持接收端的简单性。

---

## Requirements Language | 需求语言

#### 原文

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 [RFC2119].

#### 中文

本文档中的关键词 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、
"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"MAY" 和 "OPTIONAL" 的解释应遵循
RFC 2119 [RFC2119] 中的描述。

---

## Status of This Memo | 本备忘录状态

#### 原文

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering
Task Force (IETF).  Note that other groups may also distribute
working documents as Internet-Drafts.  The list of current Internet-
Drafts is at http://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months
and may be updated, replaced, or obsoleted by other documents at any
time.  It is inappropriate to use Internet-Drafts as reference
material or to cite them other than as "work in progress."

This Internet-Draft will expire on April 21, 2016.

#### 中文

本互联网草案完全遵循 BCP 78 和 BCP 79 的规定提交。

互联网草案是互联网工程任务组（IETF）的工作文档。
请注意，其他组织也可能将工作文档作为互联网草案发布。
当前互联网草案列表见 http://datatracker.ietf.org/drafts/current/。

互联网草案的有效期最长为六个月，并且随时可能被其他文档更新、替代或废弃。
将互联网草案用作参考材料或以"正在进行中的工作"之外的方式引用是不合适的。

本互联网草案将于2016年4月21日到期。

---

## Copyright Notice | 版权声明

#### 原文

Copyright (c) 2015 IETF Trust and the persons identified as the
document authors.  All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal
Provisions Relating to IETF Documents
(http://trustee.ietf.org/license-info) in effect on the date of
publication of this document.  Please review these documents
carefully, as they describe your rights and restrictions with respect
to this document.  Code Components extracted from this document must
include Simplified BSD License text as described in Section 4.e of
the Trust Legal Provisions and are provided without warranty as
described in the Simplified BSD License.

#### 中文

版权所有（c）2015 IETF Trust 及本文档作者。保留所有权利。

本文档受 BCP 78 和 IETF Trust 的《与 IETF 文档相关的法律规定》
（http://trustee.ietf.org/license-info）的约束，以本文档发布之日生效的版本为准。
请仔细阅读这些文档，因为它们描述了您对本文件的权利和限制。
从本文档中提取的代码组件必须包含 Trust 法律规定第4.e节所述的 Simplified BSD License 文本，
并按 Simplified BSD License 的规定提供，不附带任何担保。

---

## Table of Contents | 目录

#### 原文

   1.  Introduction  . . . . . . . . . . . . . . . . . . . . . . . .   2
   2.  Transport-wide Sequence Number  . . . . . . . . . . . . . . .   3
     2.1.  Semantics . . . . . . . . . . . . . . . . . . . . . . . .   3
     2.2.  RTP header extension format . . . . . . . . . . . . . . .   3
     2.3.  Signaling of use of this extension  . . . . . . . . . . .   3
   3.  Transport-wide RTCP Feedback Message  . . . . . . . . . . . .   4
     3.1.  Message format  . . . . . . . . . . . . . . . . . . . . .   4
       3.1.1.  Packet Status Symbols . . . . . . . . . . . . . . . .   6
       3.1.2.  Packet Status Chunks  . . . . . . . . . . . . . . . .   7
       3.1.3.  Run Length Chunk  . . . . . . . . . . . . . . . . . .   7
       3.1.4.  Status Vector Chunk . . . . . . . . . . . . . . . . .   8
       3.1.5.  Receive Delta . . . . . . . . . . . . . . . . . . . .   9
   4.  Overhead discussion . . . . . . . . . . . . . . . . . . . . .  10
   5.  IANA considerations . . . . . . . . . . . . . . . . . . . . .  10
   6.  Security Considerations . . . . . . . . . . . . . . . . . . .  10
   7.  Acknowledgements  . . . . . . . . . . . . . . . . . . . . . .  10
   8.  References  . . . . . . . . . . . . . . . . . . . . . . . . .  10
     8.1.  Normative References  . . . . . . . . . . . . . . . . . .  10
     8.2.  Informative References  . . . . . . . . . . . . . . . . .  10
   Appendix A.  Change log . . . . . . . . . . . . . . . . . . . . .  11
     A.1.  First version . . . . . . . . . . . . . . . . . . . . . .  11
   Authors' Addresses  . . . . . . . . . . . . . . . . . . . . . . .  11

#### 中文

   1.  引言 . . . . . . . . . . . . . . . . . . . . . . . . . . . .   2
   2.  传输级序列号 . . . . . . . . . . . . . . . . . . . . . . . .   3
     2.1.  语义 . . . . . . . . . . . . . . . . . . . . . . . . . .   3
     2.2.  RTP 头部扩展格式 . . . . . . . . . . . . . . . . . . . .   3
     2.3.  扩展使用的信令 . . . . . . . . . . . . . . . . . . . . .   3
   3.  传输级 RTCP 反馈消息 . . . . . . . . . . . . . . . . . . . .   4
     3.1.  消息格式 . . . . . . . . . . . . . . . . . . . . . . . .   4
       3.1.1.  数据包状态符号 . . . . . . . . . . . . . . . . . . .   6
       3.1.2.  数据包状态块 . . . . . . . . . . . . . . . . . . . .   7
       3.1.3.  游程长度块 . . . . . . . . . . . . . . . . . . . . .   7
       3.1.4.  状态向量块 . . . . . . . . . . . . . . . . . . . . .   8
       3.1.5.  接收增量 . . . . . . . . . . . . . . . . . . . . . .   9
   4.  开销讨论 . . . . . . . . . . . . . . . . . . . . . . . . . .  10
   5.  IANA 考量 . . . . . . . . . . . . . . . . . . . . . . . . . .  10
   6.  安全考量 . . . . . . . . . . . . . . . . . . . . . . . . . .  10
   7.  致谢 . . . . . . . . . . . . . . . . . . . . . . . . . . . .  10
   8.  参考文献 . . . . . . . . . . . . . . . . . . . . . . . . . .  10
     8.1.  规范性参考文献 . . . . . . . . . . . . . . . . . . . . .  10
     8.2.  信息性参考文献 . . . . . . . . . . . . . . . . . . . . .  10
   附录 A.  变更日志 . . . . . . . . . . . . . . . . . . . . . . . .  11
     A.1.  首个版本 . . . . . . . . . . . . . . . . . . . . . . . .  11
   作者地址 . . . . . . . . . . . . . . . . . . . . . . . . . . . .  11

---

## 1. Introduction | 1. 引言

#### 原文

This document proposes RTP header extension containing a transport-
wide packet sequence number and an RTCP feedback message feeding back
the arrival times and sequence numbers of the packets received on a
connection.

Some of the benefits that these extensions bring are:

o  The congestion control algorithms are easier to maintain and
   improve as there is less synchronization between sender and
   receiver versions needed.  It should be possible to implement
   [I-D.ietf-rmcat-gcc], [I-D.ietf-rmcat-nada] and
   [I-D.ietf-rmcat-scream-cc] with the proposed protocol.

o  More flexibility in what algorithms are used, as long as they are
   having most of their logic on the send-side.  For instance
   different behavior can be used depending on if the rate produced
   is application limited or not.

#### 中文

本文档提出了一个包含传输级数据包序列号的 RTP 头部扩展，以及一个反馈连接上接收到的数据包的到达时间和序列号的 RTCP 反馈消息。

这些扩展带来的好处包括：

o  拥塞控制算法更易于维护和改进，因为发送端和接收端之间需要的版本同步更少。
   使用本文提出的协议，应该可以实现 [I-D.ietf-rmcat-gcc]、
   [I-D.ietf-rmcat-nada] 和 [I-D.ietf-rmcat-scream-cc]。

o  算法选择更加灵活，只要其主要逻辑在发送端即可。
   例如，可以根据产生的速率是否受限于应用层而采用不同的行为。

---

## 2. Transport-wide Sequence Number | 2. 传输级序列号

---

## 2.1. Semantics | 2.1. 语义

#### 原文

This RTP header extension is added on the transport layer, and uses
the same counter for all packets which are sent over the same
connection (for instance as defined by bundle).

The benefit with a transport-wide sequence numbers is two-fold:

o  It is a better fit for congestion control as the congestion
   controller doesn't operate on media streams, but on packet flows.

o  It allows for earlier packet loss detection (and recovery) since a
   loss in stream A can be detected when a packet from stream B is
   received, thus we don't have to wait until the next packet of
   stream A is received.

#### 中文

此 RTP 头部扩展添加在传输层上，对于在同一连接上发送的所有数据包使用同一个计数器
（例如由 bundle 定义的连接）。

传输级序列号的好处有两个方面：

o  它更适合拥塞控制，因为拥塞控制器不基于媒体流操作，而是基于数据包流。

o  它允许更早地检测（和恢复）数据包丢失，因为流 A 中的丢失可以在接收到流 B 的数据包时被检测到，
   因此不必等到流 A 的下一个数据包到达。

---

## 2.2. RTP header extension format | 2.2. RTP 头部扩展格式

#### 原文

This document describes a message using the application specific
payload type.  This is suitable for experimentation; upon
standardization, a specific type can be assigned for the purpose.

    0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |       0xBE    |    0xDE       |           length=1            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  ID   | L=1   |transport-wide sequence number | zero padding  |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


An RTP header extension with a 16 bits sequence number attached to
all packets sent.  This sequence number is incremented by 1 for each
packet being sent over the same socket.

#### 中文

本文档描述了使用应用特定负载类型的消息。这适用于实验目的；
在标准化时，可以为此目的分配一个特定的类型。

    0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |       0xBE    |    0xDE       |           length=1            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  ID   | L=1   |transport-wide sequence number | zero padding  |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

一个附加在所有发送数据包上的 16 位序列号的 RTP 头部扩展。
该序列号在同一套接字上每发送一个数据包就递增 1。

---

## 2.3. Signaling of use of this extension | 2.3. 扩展使用的信令

#### 原文

When signalled in SDP, the standard mechanism for RTP header
extensions [RFC5285] is used:

a=extmap:5 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-
cc-extensions

#### 中文

在 SDP 中发出信令时，使用 RTP 头部扩展的标准机制 [RFC5285]：

a=extmap:5 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-
cc-extensions

---

## 3. Transport-wide RTCP Feedback Message | 3. 传输级 RTCP 反馈消息

#### 原文

To allow the most freedom possible to the sender, information about
each packet delivered is needed.  The simplest way of accomplishing
that is to have the receiver send back a message containing an
arrival timestamp and a packet identifier for each packet received.
This way, the receiver is dumb and simply records arrival timestamps
(A) of packets.  The sender keeps a map of in-flight packets, and
upon feedback arrival it looks up the on-wire timestamp (S) of the
corresponding packet.  From these two timestamps the sender can
compute metrics such as:

o  Inter-packet delay variation: d(i) = A(i) - S(i) - (A(i-1) -
   S(i-1))

o  Estimated queueing delay: q(i) = A(i) - S(i) -
   min{j=i-1..i-w}(A(j) - S(j))

Since the sender gets feedback about each packet sent, it will be set
to better assess the cost of sending bursts of packets compared to
aiming at sending at a constant rate decided by the receiver.

Two down-sides with this approach are:

o  It isn't possible to differentiate between lost feedback on the
   downlink and lost packets on the uplink.

o  Increased feedback rate on the reverse direction.

From a congestion control perspective, lost feedback messages are
handled by ignoring packets which would have been reported as lost or
received in the lost feedback messages.  This behavior is similar to
how a lost RTCP receiver report is handled.

It is recommended that a feedback message is sent for every frame
received, but in cases of low uplink bandwidth it is acceptable to
send them less frequently, e.g., for instance once per RTT, to reduce
the overhead.

#### 中文

为了给予发送端尽可能大的自由度，需要获知每个已传送数据包的信息。
实现这一目标最简单的方法是让接收端发送回一条消息，其中包含每个接收数据包的到达时间戳和数据包标识符。
这样，接收端保持简单，仅记录数据包的到达时间戳（A）。发送端维护一个在途数据包的映射表，
在收到反馈时查找相应数据包的线上时间戳（S）。发送端可以根据这两个时间戳计算以下指标：

o  包间延迟变化：d(i) = A(i) - S(i) - (A(i-1) - S(i-1))

o  估计排队延迟：q(i) = A(i) - S(i) - min{j=i-1..i-w}(A(j) - S(j))

由于发送端会收到关于每个已发送数据包的反馈，因此可以更好地评估发送突发数据包的成本，
与以接收端决定的恒定速率发送相比。

这种方法的两个缺点是：

o  无法区分下行链路上丢失的反馈与上行链路上丢失的数据包。

o  反向方向上的反馈速率增加。

从拥塞控制的角度来看，处理丢失的反馈消息的方法是忽略那些本应在丢失的反馈消息中报告为丢失或已接收的数据包。
这种行为类似于处理丢失的 RTCP 接收端报告的方式。

建议为每个接收到的帧发送一个反馈消息，但在上行带宽较低的情况下，
可以降低发送频率（例如每个 RTT 发送一次）以减少开销。

---

## 3.1. Message format | 3.1. 消息格式

#### 原文

The message is an RTCP message with payload type 206.  RFC 3550
[RFC3550] defines the range, RFC 4585 [RFC3550] defines the specific
PT value 206 and the FMT value 15.

       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |V=2|P|  FMT=15 |    PT=205     |           length              |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                     SSRC of packet sender                     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                      SSRC of media source                     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |      base sequence number     |      packet status count      |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                 reference time                | fb pkt. count |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |          packet chunk         |         packet chunk          |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      .                                                               .
      .                                                               .
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |         packet chunk          |  recv delta   |  recv delta   |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      .                                                               .
      .                                                               .
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |           recv delta          |  recv delta   | zero padding  |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


version (V):  2 bits This field identifies the RTP version.  The
              current version is 2.

padding (P):  1 bit If set, the padding bit indicates that the packet
              contains additional padding octets at the end that are
              not part of the control information but are included in
              the length field.

feedback message type (FMT):  5 bits This field identifies the type
              of the FB message.  It must have the value 15.

payload type (PT):  8 bits This is the RTCP packet type that
              identifies the packet as being an RTCP FB message.  The
              value must be RTPFB = 205.

SSRC of packet sender:  32 bits The synchronization source identifier
              for the originator of this packet.

SSRC of media source:  32 bits The synchronization source identifier
              of the media source that this piece of feedback
              information is related to.  TODO: This is transport wide,
              do we just pick any of the media source SSRCs?

base sequence number:  16 bits The transport-wide sequence number of
              the first packet in this feedback.  This number is not
              necessarily increased for every feedback; in the case of
              reordering it may be decreased.

packet status count:  16 bits The number of packets this feedback
              contains status for, starting with the packet identified
              by the base sequence number.

reference time:  24 bits Signed integer indicating an absolute
              reference time in some (unknown) time base chosen by the
              sender of the feedback packets.  The value is to be
              interpreted in multiples of 64ms.  The first recv delta
              in this packet is relative to the reference time.  The
              reference time makes it possible to calculate the delta
              between feedbacks even if some feedback packets are lost,
              since it always uses the same time base.

feedback packet count:  8 bits A counter incremented by one for each
              feedback packet sent.  Used to detect feedback packet
              losses.

packet chunk:  16 bits A list of packet status chunks.  These
              indicate the status of a number of packets starting with
              the one identified by base sequence number.  See below
              for details.

recv delta: 8 bits For each "packet received" status, in the packet
              status chunks, a receive delta block will follow.  See
              details below.

#### 中文

此消息是负载类型为 206 的 RTCP 消息。RFC 3550 [RFC3550] 定义了范围，
RFC 4585 [RFC3550] 定义了具体的 PT 值 206 和 FMT 值 15。

       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |V=2|P|  FMT=15 |    PT=205     |           length              |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                     SSRC of packet sender                     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                      SSRC of media source                     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |      base sequence number     |      packet status count      |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                 reference time                | fb pkt. count |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |          packet chunk         |         packet chunk          |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      .                                                               .
      .                                                               .
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |         packet chunk          |  recv delta   |  recv delta   |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      .                                                               .
      .                                                               .
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |           recv delta          |  recv delta   | zero padding  |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

版本（V）：2 位。此字段标识 RTP 版本。当前版本为 2。

填充（P）：1 位。如果设置，填充位指示数据包末尾包含额外的填充字节，
这些字节不属于控制信息，但包含在长度字段中。

反馈消息类型（FMT）：5 位。此字段标识 FB 消息的类型。其值必须为 15。

负载类型（PT）：8 位。这是 RTCP 数据包类型，用于将数据包标识为 RTCP FB 消息。
其值必须为 RTPFB = 205。

数据包发送者 SSRC：32 位。标识此数据包发起者的同步源标识符。

媒体源 SSRC：32 位。标识此反馈信息所关联的媒体源的同步源标识符。
TODO：这是传输级的，我们是否只需任意选择一个媒体源 SSRC？

基础序列号：16 位。此反馈中第一个数据包的传输级序列号。
该数值不一定在每次反馈中都递增；在有重排序的情况下可能会减小。

数据包状态计数：16 位。此反馈包含状态的数据包数量，
从基础序列号标识的数据包开始计算。

参考时间：24 位。有符号整数，表示反馈数据包发送方选择的某个（未知）时间基准中的绝对参考时间。
该值以 64ms 的倍数来解释。此数据包中的第一个接收增量是相对于参考时间的。
参考时间使得即使某些反馈数据包丢失，也能计算不同反馈之间的时间差，因为它始终使用相同的时间基准。

反馈数据包计数：8 位。每发送一个反馈数据包递增 1 的计数器。用于检测反馈数据包丢失。

数据包块：16 位。数据包状态块的列表。这些状态块指示从基础序列号标识的数据包开始的一系列数据包的状态。
详细信息见下文。

接收增量：8 位。对于数据包状态块中的每个"数据包已接收"状态，将跟随一个接收增量块。
详细信息见下文。

---

## 3.1.1. Packet Status Symbols | 3.1.1. 数据包状态符号

#### 原文

The status of a packet is described using a 2-bit symbol:

   00 Packet not received

   01 Packet received, small delta

   10 Packet received, large or negative delta

   11 [Reserved]

Packets with status "Packet not received" should not necessarily be
interpreted as lost.  They might just not have arrived yet.

For each packet received with a delta, to the previous received
packet, within +/-8191.75ms, a receive delta block is appended to the
feedback message.

Note: In the case the base sequence number is decreased, creating a
window overlapping the previous feedback messages, the status for any
packets previously reported as received must be marked as "Packet not
received" and thus no delta included for that symbol.

#### 中文

数据包的状态使用 2 位符号描述：

   00 数据包未接收

   01 数据包已接收，小增量

   10 数据包已接收，大增量或负增量

   11 [保留]

状态为"数据包未接收"的数据包不一定应被解释为已丢失。它们可能只是尚未到达。

对于每个已接收的数据包，如果其相对于前一个已接收数据包的增量在 +/-8191.75ms 范围内，
则会在反馈消息中附加一个接收增量块。

注意：当基础序列号减小，导致窗口与前一个反馈消息重叠时，
任何之前已报告为已接收的数据包的状态必须标记为"数据包未接收"，
因此不会为该符号包含增量。

---

## 3.1.2. Packet Status Chunks | 3.1.2. 数据包状态块

#### 原文

Packet status is described in chunks, similar to a Loss RLE Report
Block.  The are two different kinds of chunks:

o  Run length chunk

o  Status vector chunk

All chunk types are 16 bits in length.  The first bit of the chunk
identifies whether it is an RLE chunk or a vector chunk.

#### 中文

数据包状态以块的形式描述，类似于 Loss RLE Report Block。
块有两种不同的类型：

o  游程长度块（Run length chunk）

o  状态向量块（Status vector chunk）

所有块类型均为 16 位长度。块的第一位用于标识它是 RLE 块还是向量块。

---

## 3.1.3. Run Length Chunk | 3.1.3. 游程长度块

#### 原文

A run length chunk starts with 0 bit, followed by a packet status
symbol and the run length of that symbol.

      0                   1
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |T| S |       Run Length        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

chunk type (T):  1 bit A zero identifies this as a run length chunk.

packet status symbol (S):  2 bits The symbol repeated in this run.
              See above.

run length (L):  13 bits An unsigned integer denoting the run length.

Example 1:

      0                   1
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |0|0 0|0 0 0 0 0 1 1 0 1 1 1 0 1|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

This is a run of the "packet not received" status of length 221.

Example 2:

      0                   1
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |0|1 1|0 0 0 0 0 0 0 0 1 1 0 0 0|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

This is a run of the "packet received, w/o recv delta" status of
length 24.

#### 中文

游程长度块以 0 位开头，后跟一个数据包状态符号和该符号的游程长度。

      0                   1
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |T| S |       Run Length        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

块类型（T）：1 位。值为 0 表示这是一个游程长度块。

数据包状态符号（S）：2 位。此游程中重复的符号。参见上文。

游程长度（L）：13 位。表示游程长度的无符号整数。

示例 1：

      0                   1
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |0|0 0|0 0 0 0 0 1 1 0 1 1 1 0 1|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

这是一个长度为 221 的"数据包未接收"状态的游程。

示例 2：

      0                   1
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |0|1 1|0 0 0 0 0 0 0 0 1 1 0 0 0|
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

这是一个长度为 24 的"数据包已接收，无接收增量"状态的游程。

---

## 3.1.4. Status Vector Chunk | 3.1.4. 状态向量块

#### 原文

A status vector chunk starts with a 1 bit to identify it as a vector
chunk, followed by a symbol size bit and then 7 or 14 symbols,
depending on the size bit.

       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |T|S|       symbol list         |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

chunk type (T):  1 bit A one identifies this as a status vector
              chunk.

symbol size (S):  1 bit A zero means this vector contains only
              "packet received" (0) and "packet not received" (1)
              symbols.  This means we can compress each symbol to just
              one bit, 14 in total.  A one means this vector contains
              the normal 2-bit symbols, 7 in total.

symbol list:  14 bits A list of packet status symbols, 7 or 14 in
              total.

Example 1:

       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1|0|0 1 1 1 1 1 0 0 0 1 1 1 0 0|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

This chunk contains, in order:

   1x "packet not received"

   5x "packet received"

   3x "packet not received"

   3x "packet received"

   2x "packet not received"

Example 2:

       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1|1|0 0 1 1 0 1 0 1 0 1 0 0 0 0|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

This chunk contains, in order:

   1x "packet not received"

   1x "packet received, w/o timestamp"

   3x "packet received"

   2x "packet not received"

#### 中文

状态向量块以 1 位开头，以标识其为向量块，后跟一个符号大小位，
然后是 7 个或 14 个符号，具体取决于大小位。

       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |T|S|       symbol list         |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

块类型（T）：1 位。值为 1 表示这是一个状态向量块。

符号大小（S）：1 位。值为 0 表示此向量仅包含"数据包已接收"（0）和"数据包未接收"（1）符号。
这意味着可以将每个符号压缩为仅 1 位，共 14 个符号。
值为 1 表示此向量包含普通的 2 位符号，共 7 个符号。

符号列表：14 位。数据包状态符号的列表，共 7 个或 14 个。

示例 1：

       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1|0|0 1 1 1 1 1 0 0 0 1 1 1 0 0|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

此块按顺序包含：

   1 个 "数据包未接收"

   5 个 "数据包已接收"

   3 个 "数据包未接收"

   3 个 "数据包已接收"

   2 个 "数据包未接收"

示例 2：

       0                   1
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |1|1|0 0 1 1 0 1 0 1 0 1 0 0 0 0|
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

此块按顺序包含：

   1 个 "数据包未接收"

   1 个 "数据包已接收，无时间戳"

   3 个 "数据包已接收"

   2 个 "数据包未接收"

---

## 3.1.5. Receive Delta | 3.1.5. 接收增量

#### 原文

Deltas are represented as multiples of 250us:

o  If the "Packet received, small delta" symbol has been appended to
   the status list, an 8-bit unsigned receive delta will be appended
   to recv delta list, representing a delta in the range [0, 63.75]
   ms.

o  If the "Packet received, large or negative delta" symbol has been
   appended to the status list, a 16-bit signed receive delta will be
   appended to recv delta list, representing a delta in the range
   [-8192.0, 8191.75] ms.

o  If the delta exceeds even the larger limits, a new feedback
   message must be used, where the 24-bit base receive delta can
   cover very large gaps.

Note that the first receive delta is relative to the reference time
indicated by the base receive delta.

TODO: Add examples.

The smaller receive delta upper bound of 63.75 ms means that this is
only viable at about 1000/25.5 ~= 16 packets per second and above.
With a packet size of 1200 bytes/packet that amounts to a bitrate of
about 150 kbit/s.

The 0.25 ms resolution means that up to 4000 packets per second can
be represented.  With a 1200 bytes/packet payload, that amounts to
38.4 Mbit/s payload bandwidth.

#### 中文

增量以 250us 的倍数表示：

o  如果"数据包已接收，小增量"符号已被添加到状态列表中，
   一个 8 位无符号接收增量将被附加到接收增量列表中，
   表示范围为 [0, 63.75] ms 的增量。

o  如果"数据包已接收，大增量或负增量"符号已被添加到状态列表中，
   一个 16 位有符号接收增量将被附加到接收增量列表中，
   表示范围为 [-8192.0, 8191.75] ms 的增量。

o  如果增量甚至超出了较大的限制，则必须使用新的反馈消息，
   其中 24 位的基础接收增量可以覆盖非常大的间隔。

注意，第一个接收增量是相对于基础接收增量所指示的参考时间的。

TODO：添加示例。

较小的接收增量上限 63.75 ms 意味着这仅在大约 1000/25.5 ≈ 16 包/秒及以上时才可行。
在数据包大小为 1200 字节/包的情况下，这大约相当于 150 kbit/s 的比特率。

0.25 ms 的分辨率意味着最多可以表示每秒 4000 个数据包。
在 1200 字节/包的负载情况下，这相当于 38.4 Mbit/s 的负载带宽。

---

## 4. Overhead discussion | 4. 开销讨论

#### 原文

TODO: Examples of overhead in various scenarios.

#### 中文

TODO：各种场景下的开销示例。

---

## 5. IANA considerations | 5. IANA 考量

#### 原文

Upon publication of this document as an RFC (if it is decided to
publish it), IANA is requested to register the string "goog-remb" in
its registry of "rtcp-fb" values in the SDP attribute registry group.

#### 中文

如果将本文档作为 RFC 发布（如果决定发布），请 IANA 在 SDP 属性注册组的
"rtcp-fb" 值注册表中注册字符串 "goog-remb"。

---

## 6. Security Considerations | 6. 安全考量

#### 原文

If the RTCP packet is not protected, it is possible to inject fake
RTCP packets that can increase or decrease bandwidth.  This is not
different from security considerations for any other RTCP message.

#### 中文

如果 RTCP 数据包未受保护，可能会注入伪造的 RTCP 数据包，从而增加或减少带宽。
这与任何其他 RTCP 消息的安全考量没有区别。

---

## 7. Acknowledgements | 7. 致谢

#### 原文

#### 中文

（本草案无致谢内容。）

---

## 8. References | 8. 参考文献

---

## 8.1. Normative References | 8.1. 规范性参考文献

#### 原文

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119, March 1997.

[RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
           Jacobson, "RTP: A Transport Protocol for Real-Time
           Applications", STD 64, RFC 3550, July 2003.

[RFC5285]  Singer, D. and H. Desineni, "A General Mechanism for RTP
           Header Extensions", RFC 5285, DOI 10.17487/RFC5285, July
           2008, <http://www.rfc-editor.org/info/rfc5285>.

#### 中文

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119, 1997年3月.

[RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
           Jacobson, "RTP: A Transport Protocol for Real-Time
           Applications", STD 64, RFC 3550, 2003年7月.

[RFC5285]  Singer, D. and H. Desineni, "A General Mechanism for RTP
           Header Extensions", RFC 5285, DOI 10.17487/RFC5285, 2008年7月,
           <http://www.rfc-editor.org/info/rfc5285>.

---

## 8.2. Informative References | 8.2. 信息性参考文献

#### 原文

[I-D.ietf-rmcat-gcc]
           Holmer, S., Marcon, J., Carlucci, G., Cicco, L., and S.
           Mascolo, "A Google Congestion Control Algorithm for Real-
           Time Communication", draft-ietf-rmcat-gcc-00 (work in
           progress), September 2015.

[I-D.ietf-rmcat-nada]
           Zhu, X., Pan, R., Ramalho, M., Cruz, S., Jones, P., Fu,
           J., D'Aronco, S., and C. Ganzhorn, "NADA: A Unified
           Congestion Control Scheme for Real-Time Media", draft-
           ietf-rmcat-nada-01 (work in progress), October 2015.

[I-D.ietf-rmcat-scream-cc]
           Johansson, I. and Z. Sarker, "Self-Clocked Rate Adaptation
           for Multimedia", draft-ietf-rmcat-scream-cc-01 (work in
           progress), July 2015.

#### 中文

[I-D.ietf-rmcat-gcc]
           Holmer, S., Marcon, J., Carlucci, G., Cicco, L., and S.
           Mascolo, "A Google Congestion Control Algorithm for Real-
           Time Communication", draft-ietf-rmcat-gcc-00（进行中的工作），2015年9月.

[I-D.ietf-rmcat-nada]
           Zhu, X., Pan, R., Ramalho, M., Cruz, S., Jones, P., Fu,
           J., D'Aronco, S., and C. Ganzhorn, "NADA: A Unified
           Congestion Control Scheme for Real-Time Media", draft-
           ietf-rmcat-nada-01（进行中的工作），2015年10月.

[I-D.ietf-rmcat-scream-cc]
           Johansson, I. and Z. Sarker, "Self-Clocked Rate Adaptation
           for Multimedia", draft-ietf-rmcat-scream-cc-01（进行中的工作），2015年7月.

---

## Appendix A. Change log | 附录 A. 变更日志

---

## A.1. First version | A.1. 首个版本

#### 原文

#### 中文

（首个版本，无变更记录。）

---

## Authors' Addresses | 作者地址

#### 原文

Stefan Holmer
Google
Kungsbron 2
Stockholm  11122
Sweden

Email: holmer@google.com


Magnus Flodman
Google
Kungsbron 2
Stockholm  11122
Sweden

Email: mflodman@google.com


Erik Sprang
Google
Kungsbron 2
Stockholm  11122
Sweden

Email: sprang@google.com

#### 中文

Stefan Holmer
Google
Kungsbron 2
Stockholm  11122
Sweden

Email: holmer@google.com


Magnus Flodman
Google
Kungsbron 2
Stockholm  11122
Sweden

Email: mflodman@google.com


Erik Sprang
Google
Kungsbron 2
Stockholm  11122
Sweden

Email: sprang@google.com
