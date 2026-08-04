# RFC/draft 完整中英对照翻译
# RFC/draft Complete Chinese-English Parallel Translation

## RTCP message for Receiver Estimated Maximum Bitrate | 接收端估计最大比特率的RTCP消息

---

## Header | 文档头部

#### 原文

Network Working Group                                 H. Alvestrand, Ed.
Internet-Draft                                                    Google
Intended status: Experimental                           October 21, 2013
Expires: April 24, 2014


          RTCP message for Receiver Estimated Maximum Bitrate
                     draft-alvestrand-rmcat-remb-03

#### 中文

网络工作组                                           H. Alvestrand, Ed.
互联网草案                                                      Google
预期状态: 实验性                                      2013年10月21日
过期时间: 2014年4月24日


          接收端估计最大比特率的RTCP消息
                     draft-alvestrand-rmcat-remb-03

---

## Abstract | 摘要

#### 原文

   This document proposes an RTCP message for use in experimentally-
   deployed congestion control algorithms for RTP-based media flows.

   It also describes an absolute-value timestamp option for use in
   bandwidth estimatoin.

#### 中文

   本文档提出了一种RTCP消息，用于基于RTP的媒体流中实验性部署的拥塞控制算法。

   本文档还描述了一种用于带宽估计的绝对值时间戳选项。

---

## Requirements Language | 需求语言

#### 原文

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in RFC 2119 [RFC2119].

#### 中文

   本文档中的关键词 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、
   "SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"MAY"和"OPTIONAL" 应按照
   RFC 2119 [RFC2119] 中的描述进行解释。

---

## Status of this Memo | 本文状态

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

   This Internet-Draft will expire on April 24, 2014.

#### 中文

   本互联网草案完全符合BCP 78和BCP 79的规定提交。

   互联网草案是互联网工程任务组(IETF)的工作文档。请注意，其他组织也可能
   将工作文档作为互联网草案分发。当前互联网草案列表位于
   http://datatracker.ietf.org/drafts/current/。

   互联网草案是有效期最长为六个月的草案文件，可能随时被其他文件更新、
   替换或废弃。将互联网草案用作参考资料或以"正在进行的工作"之外的
   方式引用是不恰当的。

   本互联网草案将于2014年4月24日过期。

---

## Copyright Notice | 版权声明

#### 原文

   Copyright (c) 2013 IETF Trust and the persons identified as the
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

   版权所有 (c) 2013 IETF Trust及被确认为文档作者的人员。保留所有权利。

   本文档受BCP 78和IETF Trust关于IETF文档的法律规定
   (http://trustee.ietf.org/license-info) 的约束，该规定在本文档
   发布之日生效。请仔细阅读这些文件，因为它们描述了您对本文档的权利
   和限制。从本文档中提取的代码组件必须包含Trust法律条款第4.e节中
   描述的简化BSD许可证文本，并且按照简化BSD许可证中的描述提供，
   不附带任何担保。

---

## Table of Contents | 目录

#### 原文

   1.  Introduction  . . . . . . . . . . . . . . . . . . . . . . . . . 3
   2.  Receiver Estimated Max Bitrate (REMB) . . . . . . . . . . . . . 3
     2.1.  Semantics . . . . . . . . . . . . . . . . . . . . . . . . . 3
     2.2.  Message format  . . . . . . . . . . . . . . . . . . . . . . 3
     2.3.  Signaling of use of this extension  . . . . . . . . . . . . 5
   3.  Absolute Send Time  . . . . . . . . . . . . . . . . . . . . . . 5
   4.  IANA considerations . . . . . . . . . . . . . . . . . . . . . . 6
   5.  Security Considerations . . . . . . . . . . . . . . . . . . . . 6
   6.  Acknowledgements  . . . . . . . . . . . . . . . . . . . . . . . 6
   7.  References  . . . . . . . . . . . . . . . . . . . . . . . . . . 7
     7.1.  Normative References  . . . . . . . . . . . . . . . . . . . 7
     7.2.  Informative References  . . . . . . . . . . . . . . . . . . 7
   Appendix A.  Change log . . . . . . . . . . . . . . . . . . . . . . 7
     A.1.  From appendix of -congestion-01 to -00  . . . . . . . . . . 7
     A.2.  From -00 to -02 . . . . . . . . . . . . . . . . . . . . . . 7
     A.3.  From -02 to -03 . . . . . . . . . . . . . . . . . . . . . . 8
   Author's Address  . . . . . . . . . . . . . . . . . . . . . . . . . 8

#### 中文

   1.  引言 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 3
   2.  接收端估计最大比特率(REMB)  . . . . . . . . . . . . . . . . . 3
     2.1.  语义  . . . . . . . . . . . . . . . . . . . . . . . . . . 3
     2.2.  消息格式  . . . . . . . . . . . . . . . . . . . . . . . . 3
     2.3.  此扩展使用的信令  . . . . . . . . . . . . . . . . . . . . 5
   3.  绝对发送时间 . . . . . . . . . . . . . . . . . . . . . . . . . 5
   4.  IANA考虑 . . . . . . . . . . . . . . . . . . . . . . . . . . . 6
   5.  安全考虑 . . . . . . . . . . . . . . . . . . . . . . . . . . . 6
   6.  致谢  . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 6
   7.  参考文献 . . . . . . . . . . . . . . . . . . . . . . . . . . . 7
     7.1.  规范性参考文献 . . . . . . . . . . . . . . . . . . . . . . 7
     7.2.  信息性参考文献 . . . . . . . . . . . . . . . . . . . . . . 7
   附录A. 变更日志  . . . . . . . . . . . . . . . . . . . . . . . . . 7
     A.1.  从-congestion-01附录到-00  . . . . . . . . . . . . . . . . 7
     A.2.  从-00到-02  . . . . . . . . . . . . . . . . . . . . . . . . 7
     A.3.  从-02到-03  . . . . . . . . . . . . . . . . . . . . . . . . 8
   作者地址 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 8

---

## 1. Introduction | 1. 引言

#### 原文

   This document proposes an RTCP feedback message signalling the
   estimated total available bandwidth for a session.

   If this function is available, it is possible to implement the
   algorithm in [I-D.alvestrand-rtcweb-congestion], or other algorithms
   with the same kind of feedback messaging need, in a fashion that
   covers multiple RTP streams at once.

#### 中文

   本文档提出了一种RTCP反馈消息，用于通知会话的估计总可用带宽。

   如果此功能可用，则可以以覆盖多个RTP流的方式实现
   [I-D.alvestrand-rtcweb-congestion]中的算法，或其他具有相同类型
   反馈消息需求的算法。

---

## 2. Receiver Estimated Max Bitrate (REMB) | 2. 接收端估计最大比特率(REMB)

### 2.1. Semantics | 2.1. 语义

#### 原文

   This feedback message is used to notify a sender of multiple media
   streams over the same RTP session of the total estimated available
   bit rate on the path to the receiving side of this RTP session.

   Within the common packet header for feedback messages (as defined in
   section 6.1 of [RFC4585]), the "SSRC of packet sender" field
   indicates the source of the notification.  The "SSRC of media source"
   is not used and SHALL be set to 0.  This usage of the value zero is
   also done in other RFCs.

   The reception of a REMB message by a media sender conforming to this
   specification SHALL result in the total bit rate sent on the RTP
   session this message applies to being equal to or lower than the bit
   rate in this message.  The new bit rate constraint should be applied
   as fast as reasonable.  The sender is free to apply additional
   bandwidth restrictions based on its own restrictions and estimates.

#### 中文

   此反馈消息用于通知同一RTP会话上多个媒体流的发送端，告知该RTP会话
   接收方路径上的总估计可用比特率。

   在反馈消息的通用数据包头中（如[RFC4585]第6.1节所定义），"数据包
   发送者SSRC"字段指示通知的来源。 "媒体源SSRC"不使用，应设置为0。
   这种使用零值的做法在其他RFC中也有采用。

   符合本规范的媒体发送端收到REMB消息后，该消息所适用的RTP会话上
   发送的总比特率应等于或低于此消息中的比特率。新的比特率约束应
   尽快合理地应用。发送端可以根据自己的限制和估计自由地施加额外的
   带宽限制。

---

### 2.2. Message format | 2.2. 消息格式

#### 原文

   This document describes a message using the application specific
   payload type.  This is suitable for experimentation; upon
   standardization, a specific type can be assigned for the purpose.

   The message is an RTCP message with payload type 206.  RFC 3550
   [RFC3550] defines the range, RFC 4585 defines the specific PT value
   206 and the FMT value 15.

#### 中文

   本文档描述了一个使用应用特定载荷类型的消息。这适用于实验；
   在标准化后，可以为此目的分配一个特定的类型。

   该消息是载荷类型为206的RTCP消息。RFC 3550 [RFC3550]定义了范围，
   RFC 4585定义了具体的PT值206和FMT值15。

---

### Message Format Diagram | 消息格式图

#### 原文

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P| FMT=15  |   PT=206      |             length            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                  SSRC of packet sender                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                  SSRC of media source                         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Unique identifier 'R' 'E' 'M' 'B'                            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Num SSRC     | BR Exp    |  BR Mantissa                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   SSRC feedback                                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  ...                                                          |
```

#### 中文

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P| FMT=15  |   PT=206      |             length            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                  数据包发送者SSRC                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                  媒体源SSRC                                    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  唯一标识符 'R' 'E' 'M' 'B'                                    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Num SSRC     | BR Exp    |  BR Mantissa                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   SSRC反馈                                                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  ...                                                          |
```

---

### Field Descriptions | 字段描述

#### 原文

   The fields V, P, SSRC, and length are defined in the RTP
   specification [2], the respective meaning being summarized below:

   version (V): (2 bits):   This field identifies the RTP version.  The
               current version is 2.

   padding (P) (1 bit):   If set, the padding bit indicates that the
               packet contains additional padding octets at the end that
               are not part of the control information but are included
               in the length field.  Always 0.

   Feedback message type (FMT) (5 bits):  This field identifies the type
               of the FB message and is interpreted relative to the type
               (transport layer, payload- specific, or application layer
               feedback).  Always 15, application layer feedback
               message.  RFC 4585 section 6.4.

   Payload type (PT) (8 bits):   This is the RTCP packet type that
               identifies the packet as being an RTCP FB message.
               Always PSFB (206), Payload-specific FB message.  RFC 4585
               section 6.4.

   Length (16 bits):  The length of this packet in 32-bit words minus
               one, including the header and any padding.  This is in
               line with the definition of the length field used in RTCP
               sender and receiver reports [3].  RFC 4585 section 6.4.

   SSRC of packet sender (32 bits):  The synchronization source
               identifier for the originator of this packet.  RFC 4585
               section 6.4.

   SSRC of media source (32 bits):  Always 0; this is the same
               convention as in [RFC5104] section 4.2.2.2 (TMMBN).

   Unique identifier (32 bits):  Always 'R' 'E' 'M' 'B' (4 ASCII
               characters).

   Num SSRC (8 bits):  Number of SSRCs in this message.

   BR Exp (6 bits):   The exponential scaling of the mantissa for the
               maximum total media bit rate value, ignoring all packet
               overhead.  The value is an unsigned integer [0..63], as
               in RFC 5104 section 4.2.2.1.

   BR Mantissa (18 bits):   The mantissa of the maximum total media bit
               rate (ignoring all packet overhead) that the sender of
               the REMB estimates.  The BR is the estimate of the
               traveled path for the SSRCs reported in this message.
               The value is an unsigned integer in number of bits per
               second.

   SSRC feedback (32 bits)  Consists of one or more SSRC entries which
               this feedback message applies to.

#### 中文

   V、P、SSRC和length字段在RTP规范[2]中定义，其各自含义总结如下：

   version (V): (2比特):   此字段标识RTP版本。当前版本为2。

   padding (P) (1比特):   如果设置，填充位表示数据包末尾包含额外的
               填充字节，这些字节不属于控制信息，但计入长度字段。
               始终为0。

   Feedback message type (FMT) (5比特):  此字段标识FB消息的类型，
               并相对于类型（传输层、载荷特定或应用层反馈）进行解释。
               始终为15，即应用层反馈消息。参见RFC 4585第6.4节。

   Payload type (PT) (8比特):   这是RTCP数据包类型，用于将数据包
               标识为RTCP FB消息。始终为PSFB (206)，即载荷特定FB消息。
               参见RFC 4585第6.4节。

   Length (16比特):  此数据包的长度，以32位字为单位减一，包括头部
               和任何填充。这与RTCP发送端和接收端报告[3]中使用的长度
               字段的定义一致。参见RFC 4585第6.4节。

   SSRC of packet sender (32比特):  此数据包发起者的同步源标识符。
               参见RFC 4585第6.4节。

   SSRC of media source (32比特):  始终为0；这与[RFC5104]第4.2.2.2节
               (TMMBN)中的约定相同。

   Unique identifier (32比特):  始终为'R' 'E' 'M' 'B'（4个ASCII字符）。

   Num SSRC (8比特):  此消息中的SSRC数量。

   BR Exp (6比特):   最大总媒体比特率值尾数的指数缩放，忽略所有数据包
               开销。该值为无符号整数[0..63]，如RFC 5104第4.2.2.1节所述。

   BR Mantissa (18比特):   REMB发送端估计的最大总媒体比特率（忽略所有
               数据包开销）的尾数。BR是该消息中报告的SSRC所经路径的
               估计值。该值为无符号整数，单位为比特每秒。

   SSRC feedback (32比特)  由一个或多个此反馈消息适用的SSRC条目组成。

---

### 2.3. Signaling of use of this extension | 2.3. 此扩展使用的信令

#### 原文

   We negotiate use of the message in SDP using a header extension
   according to RFC 4585 section 4.2, with the value "goog-remb":

   a=rtcp-fb:<payload type> goog-remb

#### 中文

   我们根据RFC 4585第4.2节，使用头部扩展在SDP中协商此消息的使用，
   其值为"goog-remb"：

   a=rtcp-fb:<载荷类型> goog-remb

---

## 3. Absolute Send Time | 3. 绝对发送时间

#### 原文

   Google has found that there are issues with relative send time offset
   when packets are relayed at nodes that are not the source of the RTP
   clock; it is hard to generate accurate offsets when you have to
   regenerate the base clock from the incoming packets before you can
   figure out what time to match; also, using signals from multiple
   flows becomes impossible unless the timestamps come from a common
   clock.

   The Absolute Send Time extension is used to stamp RTP packets with a
   timestamp showing the departure time from the system that put this
   packet on the wire (or as close to this as we can manage).

   o  Name: "Absolute Sender Time" ; "RTP Header Extension for Absolute
      Sender Time".

   o  Formal name:
      "http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time".

   o  Wire format: 1-byte extension, 3 bytes of data. total 4 bytes
      extra per packet (plus shared 4 bytes for all extensions present:
      2 byte magic word 0xBEDE, 2 byte # of extensions).

   o  Encoding: Timestamp is in seconds, 24 bit 6.18 fixed point,
      yielding 64s wraparound and 3.8us resolution (one increment for
      each 477 bytes going out on a 1Gbps interface).

   o  Relation to NTP timestamps: abs_send_time_24 = (ntp_timestamp_64
      >> 14) & 0x00ffffff ; NTP timestamp is the number of seconds since
      the epoch, in 32.32 bit fixed point format.

   o  Notes: Packets are time stamped when going out, preferably close
      to metal.  Intermediate RTP relays (RTP entities possibly altering
      the relative timing of packets in the stream) should remove the
      extension or overwrite its value with its own timestamp.

   When signalled in SDP, the standard mechanism for RTCP extensions
   [RFC5285] is used:

   a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time

#### 中文

   Google发现，当数据包在不是RTP时钟源的节点上进行中继时，相对发送时间
   偏移存在问题；当您必须从传入数据包重新生成基准时钟才能确定要匹配的
   时间时，很难生成准确的偏移量；此外，除非时间戳来自公共时钟，否则
   使用来自多个流的信号变得不可能。

   绝对发送时间扩展用于在RTP数据包上打上时间戳，显示该数据包从将其
   放到网络上的系统离开的时间（或尽可能接近该时间）。

   o  名称: "Absolute Sender Time" ; "RTP Header Extension for Absolute
      Sender Time"（绝对发送者的RTP头部扩展）。

   o  正式名称:
      "http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time"。

   o  线格式: 1字节扩展，3字节数据。每个数据包总共额外4字节
      （加上所有存在的扩展共享的4字节：2字节魔数0xBEDE，2字节扩展数量）。

   o  编码: 时间戳以秒为单位，24位6.18定点格式，产生64秒回绕和3.8微秒
      分辨率（在1Gbps接口上每477字节递增一次）。

   o  与NTP时间戳的关系: abs_send_time_24 = (ntp_timestamp_64
      >> 14) & 0x00ffffff ; NTP时间戳是从纪元以来的秒数，采用32.32位
      定点格式。

   o  备注: 数据包在发出时打上时间戳，最好尽可能靠近硬件。
      中间RTP中继（可能改变流中数据包相对时序的RTP实体）应移除此扩展
      或用自己的时间戳覆盖其值。

   在SDP中发出信号时，使用RTCP扩展的标准机制[RFC5285]：

   a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time

---

## 4. IANA considerations | 4. IANA考虑

#### 原文

   Upon publication of this document as an RFC (if it is decided to
   publish it), IANA is requested to register the string "goog-remb" in
   its registry of "rtcp-fb" values in the SDP attribute registry group.

#### 中文

   在本文档作为RFC发布时（如果决定发布），请求IANA在其SDP属性注册表组
   的"rtcp-fb"值注册表中注册字符串"goog-remb"。

---

## 5. Security Considerations | 5. 安全考虑

#### 原文

   If the RTCP packet is not protected, it is possible to inject fake
   RTCP packets that can increase or decrease bandwidth.  This is not
   different from security considerations for any other RTCP message.

#### 中文

   如果RTCP数据包未受到保护，则可能注入伪造的RTCP数据包，从而增加或
   减少带宽。这与任何其他RTCP消息的安全考虑没有区别。

---

## 6. Acknowledgements | 6. 致谢

#### 原文

   This proposal has emerged from discussions between, among others,
   Justin Uberti, Magnus Flodman, Patrik Westin, Stefan Holmer and
   Henrik Lundin.

#### 中文

   本提案源于Justin Uberti、Magnus Flodman、Patrik Westin、Stefan Holmer
   和Henrik Lundin等人之间的讨论。

---

## 7. References | 7. 参考文献

### 7.1. Normative References | 7.1. 规范性参考文献

#### 原文

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, March 1997.

   [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
              Jacobson, "RTP: A Transport Protocol for Real-Time
              Applications", STD 64, RFC 3550, July 2003.

   [RFC4585]  Ott, J., Wenger, S., Sato, N., Burmeister, C., and J. Rey,
              "Extended RTP Profile for Real-time Transport Control
              Protocol (RTCP)-Based Feedback (RTP/AVPF)", RFC 4585,
              July 2006.

   [RFC5104]  Wenger, S., Chandra, U., Westerlund, M., and B. Burman,
              "Codec Control Messages in the RTP Audio-Visual Profile
              with Feedback (AVPF)", RFC 5104, February 2008.

   [RFC5285]  Singer, D. and H. Desineni, "A General Mechanism for RTP
              Header Extensions", RFC 5285, July 2008.

#### 中文

   [RFC2119]  Bradner, S., "用于在RFC中指示需求级别的关键词",
              BCP 14, RFC 2119, 1997年3月。

   [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., 和 V.
              Jacobson, "RTP: 实时应用的传输协议", STD 64, RFC 3550,
              2003年7月。

   [RFC4585]  Ott, J., Wenger, S., Sato, N., Burmeister, C., 和 J. Rey,
              "用于基于实时传输控制协议(RTCP)反馈的扩展RTP配置文件
              (RTP/AVPF)", RFC 4585, 2006年7月。

   [RFC5104]  Wenger, S., Chandra, U., Westerlund, M., 和 B. Burman,
              "具有反馈的RTP音视频配置文件(AVPF)中的编解码器控制消息",
              RFC 5104, 2008年2月。

   [RFC5285]  Singer, D. 和 H. Desineni, "RTP头部扩展的通用机制",
              RFC 5285, 2008年7月。

---

### 7.2. Informative References | 7.2. 信息性参考文献

#### 原文

   [I-D.alvestrand-rtcweb-congestion]
              Holmer, S. and H. Alvestrand, "A Google Congestion Control
              Algorithm for Real-Time Communication on the World Wide
              Web", draft-alvestrand-rtcweb-congestion-03 (work in
              progress), October 2012.

   [RFC5450]  Singer, D. and H. Desineni, "Transmission Time Offsets in
              RTP Streams", RFC 5450, March 2009.

#### 中文

   [I-D.alvestrand-rtcweb-congestion]
              Holmer, S. 和 H. Alvestrand, "用于万维网上实时通信的
              Google拥塞控制算法", draft-alvestrand-rtcweb-congestion-03
              （工作进行中）, 2012年10月。

   [RFC5450]  Singer, D. 和 H. Desineni, "RTP流中的传输时间偏移",
              RFC 5450, 2009年3月。

---

## Appendix A. Change log | 附录A. 变更日志

### A.1. From appendix of -congestion-01 to -00 | A.1. 从-congestion-01附录到-00

#### 原文

   The timestamp option was removed.  Discussion concluded that the RFC
   5450 [RFC5450] "transmission time offset" header likely gives
   accurate enough send-time information for our purposes.

#### 中文

   时间戳选项被移除。讨论得出结论，RFC 5450 [RFC5450]的"传输时间偏移"
   头部可能为我们的目的提供了足够准确的发送时间信息。

---

### A.2. From -00 to -02 | A.2. 从-00到-02

#### 原文

   No changes.  These are "keepalive" publications.

#### 中文

   无更改。这些是"保活"发布。

---

### A.3. From -02 to -03 | A.3. 从-02到-03

#### 原文

   Added information on the absolute-timestamp extension and on SDP
   negotiation of REMB.

#### 中文

   添加了关于绝对时间戳扩展和REMB的SDP协商的信息。

---

## Author's Address | 作者地址

#### 原文

   Harald Alvestrand (editor)
   Google
   Kungsbron 2
   Stockholm,   11122
   Sweden

   Email: harald@alvestrand.no

#### 中文

   Harald Alvestrand (编辑)
   Google
   Kungsbron 2
   Stockholm,   11122
   Sweden

   电子邮件: harald@alvestrand.no
