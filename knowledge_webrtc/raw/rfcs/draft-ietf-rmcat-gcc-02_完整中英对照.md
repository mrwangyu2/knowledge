# draft-ietf-rmcat-gcc-02 完整中英对照翻译
# draft-ietf-rmcat-gcc-02 Complete Chinese-English Parallel Translation

## A Google Congestion Control Algorithm for Real-Time Communication | 谷歌实时通信拥塞控制算法

---

## Header | 头部信息

#### 原文

Network Working Group                                          S. Holmer
Internet-Draft                                                 H. Lundin
Intended status: Informational                                    Google
Expires: January 9, 2017                                     G. Carlucci
                                                             L. De Cicco
                                                              S. Mascolo
                                                     Politecnico di Bari
                                                            July 8, 2016


   A Google Congestion Control Algorithm for Real-Time Communication
                        draft-ietf-rmcat-gcc-02

#### 中文

网络工作组                                                S. Holmer
互联网草案                                                   H. Lundin
预期状态：信息性                                                Google
过期日期：2017年1月9日                                    G. Carlucci
                                                            L. De Cicco
                                                             S. Mascolo
                                                     巴里理工大学
                                                          2016年7月8日


   一种用于实时通信的谷歌拥塞控制算法
                        draft-ietf-rmcat-gcc-02

---

## Abstract | 摘要

#### 原文

   This document describes two methods of congestion control when using
   real-time communications on the World Wide Web (RTCWEB); one delay-
   based and one loss-based.

   It is published as an input document to the RMCAT working group on
   congestion control for media streams.  The mailing list of that
   working group is rmcat@ietf.org.

#### 中文

   本文档描述了在使用万维网实时通信（RTCWEB）时的两种拥塞控制方法：一种基于延迟，一种基于丢包。

   本文作为RMCAT工作组关于媒体流拥塞控制的输入文档发布。该工作组的邮件列表为rmcat@ietf.org。

---

## Requirements Language | 需求语言

#### 原文

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in RFC 2119 [RFC2119].

#### 中文

   本文档中的关键词"MUST"（必须）、"MUST NOT"（不得）、"REQUIRED"（要求）、"SHALL"（应当）、"SHALL NOT"（不应）、"SHOULD"（应该）、"SHOULD NOT"（不应该）、"RECOMMENDED"（推荐）、"MAY"（可以）以及"OPTIONAL"（可选）应按照RFC 2119 [RFC2119]中的描述进行解释。

---

## Status of This Memo | 本文状态

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

   This Internet-Draft will expire on January 9, 2017.

#### 中文

   本互联网草案完全符合BCP 78和BCP 79的规定提交。

   互联网草案是互联网工程任务组（IETF）的工作文档。请注意，其他组织也可能以互联网草案的形式分发工作文档。当前互联网草案的列表见http://datatracker.ietf.org/drafts/current/。

   互联网草案是有效期最长为六个月的草案文档，可能随时被其他文档更新、替代或淘汰。将互联网草案用作参考资料或以"进行中的工作"以外的方式引用是不恰当的。

   本互联网草案将于2017年1月9日过期。

---

## Copyright Notice | 版权声明

#### 原文

   Copyright (c) 2016 IETF Trust and the persons identified as the
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

   版权所有 (c) 2016 IETF Trust及被标识为文档作者的人员。保留所有权利。

   本文档受BCP 78及在本文件出版之日生效的IETF Trust关于IETF文档的法律规定（http://trustee.ietf.org/license-info）的约束。请仔细阅读这些文档，因为它们描述了您对本文件的权利和限制。从本文档中提取的代码组件必须包含Trust法律条款第4.e节所述的简化BSD许可证文本，并按简化BSD许可证所述无担保提供。

---

## Table of Contents | 目录

#### 原文

   1.  Introduction  . . . . . . . . . . . . . . . . . . . . . . . .   3
     1.1.  Mathematical notation conventions . . . . . . . . . . . .   3
   2.  System model  . . . . . . . . . . . . . . . . . . . . . . . .   4
   3.  Feedback and extensions . . . . . . . . . . . . . . . . . . .   4
   4.  Sending Engine  . . . . . . . . . . . . . . . . . . . . . . .   5
   5.  Delay-based control . . . . . . . . . . . . . . . . . . . . .   5
     5.1.  Arrival-time model  . . . . . . . . . . . . . . . . . . .   6
     5.2.  Pre-filtering . . . . . . . . . . . . . . . . . . . . . .   7
     5.3.  Arrival-time filter . . . . . . . . . . . . . . . . . . .   7
     5.4.  Over-use detector . . . . . . . . . . . . . . . . . . . .   8
     5.5.  Rate control  . . . . . . . . . . . . . . . . . . . . . .  10
     5.6.  Parameters settings . . . . . . . . . . . . . . . . . . .  12
   6.  Loss-based control  . . . . . . . . . . . . . . . . . . . . .  13
   7.  Interoperability Considerations . . . . . . . . . . . . . . .  14
   8.  Implementation Experience . . . . . . . . . . . . . . . . . .  14
   9.  Further Work  . . . . . . . . . . . . . . . . . . . . . . . .  15
   10. IANA Considerations . . . . . . . . . . . . . . . . . . . . .  15
   11. Security Considerations . . . . . . . . . . . . . . . . . . .  15
   12. Acknowledgements  . . . . . . . . . . . . . . . . . . . . . .  15
   13. References  . . . . . . . . . . . . . . . . . . . . . . . . .  16
     13.1.  Normative References . . . . . . . . . . . . . . . . . .  16
     13.2.  Informative References . . . . . . . . . . . . . . . . .  16
   Appendix A.  Change log . . . . . . . . . . . . . . . . . . . . .  16
     A.1.  Version -00 to -01  . . . . . . . . . . . . . . . . . . .  16
     A.2.  Version -01 to -02  . . . . . . . . . . . . . . . . . . .  17
     A.3.  Version -02 to -03  . . . . . . . . . . . . . . . . . . .  17
     A.4.  rtcweb-03 to rmcat-00 . . . . . . . . . . . . . . . . . .  17
     A.5.  rmcat -00 to -01  . . . . . . . . . . . . . . . . . . . .  17
     A.6.  rmcat -01 to -02  . . . . . . . . . . . . . . . . . . . .  17
     A.7.  rmcat -02 to -03  . . . . . . . . . . . . . . . . . . . .  18
     A.8.  ietf-rmcat -00 to ietf-rmcat -01  . . . . . . . . . . . .  18
     A.9.  ietf-rmcat -01 to ietf-rmcat -02  . . . . . . . . . . . .  18
   Authors' Addresses  . . . . . . . . . . . . . . . . . . . . . . .  18

#### 中文

   1.  引言  . . . . . . . . . . . . . . . . . . . . . . . . . . .   3
     1.1.  数学符号约定 . . . . . . . . . . . . . . . . . . . . .   3
   2.  系统模型  . . . . . . . . . . . . . . . . . . . . . . . . .   4
   3.  反馈与扩展 . . . . . . . . . . . . . . . . . . . . . . . .   4
   4.  发送引擎  . . . . . . . . . . . . . . . . . . . . . . . . .   5
   5.  基于延迟的控制 . . . . . . . . . . . . . . . . . . . . . .   5
     5.1.  到达时间模型  . . . . . . . . . . . . . . . . . . . . .   6
     5.2.  预过滤 . . . . . . . . . . . . . . . . . . . . . . . . .   7
     5.3.  到达时间滤波器 . . . . . . . . . . . . . . . . . . . . .   7
     5.4.  过度使用检测器 . . . . . . . . . . . . . . . . . . . . .   8
     5.5.  速率控制  . . . . . . . . . . . . . . . . . . . . . . .  10
     5.6.  参数设置 . . . . . . . . . . . . . . . . . . . . . . .   12
   6.  基于丢包的控制  . . . . . . . . . . . . . . . . . . . . . .  13
   7.  互操作性考量 . . . . . . . . . . . . . . . . . . . . . . .  14
   8.  实现经验 . . . . . . . . . . . . . . . . . . . . . . . . .  14
   9.  后续工作  . . . . . . . . . . . . . . . . . . . . . . . . .  15
   10. IANA考量 . . . . . . . . . . . . . . . . . . . . . . . . .  15
   11. 安全考量 . . . . . . . . . . . . . . . . . . . . . . . . .  15
   12. 致谢  . . . . . . . . . . . . . . . . . . . . . . . . . . .  15
   13. 参考文献  . . . . . . . . . . . . . . . . . . . . . . . . .  16
     13.1.  规范性参考文献 . . . . . . . . . . . . . . . . . . . .  16
     13.2.  信息性参考文献 . . . . . . . . . . . . . . . . . . . .  16
   附录 A.  变更日志 . . . . . . . . . . . . . . . . . . . . . . .  16
     A.1.  版本 -00 至 -01  . . . . . . . . . . . . . . . . . . . .  16
     A.2.  版本 -01 至 -02  . . . . . . . . . . . . . . . . . . . .  17
     A.3.  版本 -02 至 -03  . . . . . . . . . . . . . . . . . . . .  17
     A.4.  rtcweb-03 至 rmcat-00 . . . . . . . . . . . . . . . . .  17
     A.5.  rmcat -00 至 -01  . . . . . . . . . . . . . . . . . . .  17
     A.6.  rmcat -01 至 -02  . . . . . . . . . . . . . . . . . . .  17
     A.7.  rmcat -02 至 -03  . . . . . . . . . . . . . . . . . . .  18
     A.8.  ietf-rmcat -00 至 ietf-rmcat -01  . . . . . . . . . . .  18
     A.9.  ietf-rmcat -01 至 ietf-rmcat -02  . . . . . . . . . . .  18
   作者地址  . . . . . . . . . . . . . . . . . . . . . . . . . . .  18

---

## 1.  Introduction | 1. 引言

#### 原文

   Congestion control is a requirement for all applications sharing the
   Internet resources [RFC2914].

   Congestion control for real-time media is challenging for a number of
   reasons:

   o  The media is usually encoded in forms that cannot be quickly
      changed to accommodate varying bandwidth, and bandwidth
      requirements can often be changed only in discrete, rather large
      steps

   o  The participants may have certain specific wishes on how to
      respond - which may not be reducing the bandwidth required by the
      flow on which congestion is discovered

   o  The encodings are usually sensitive to packet loss, while the
      real-time requirement precludes the repair of packet loss by
      retransmission

   This memo describes two congestion control algorithms that together
   are able to provide good performance and reasonable bandwidth sharing
   with other video flows using the same congestion control and with TCP
   flows that share the same links.

   The signaling used consists of experimental RTP header extensions and
   RTCP messages RFC 3550 [RFC3550] as defined in [abs-send-time],
   [I-D.alvestrand-rmcat-remb] and
   [I-D.holmer-rmcat-transport-wide-cc-extensions].

#### 中文

   拥塞控制是所有共享互联网资源的应用的需求 [RFC2914]。

   实时媒体的拥塞控制由于多种原因而充满挑战：

   o  媒体通常以无法快速改变以适应变化带宽的形式编码，且带宽需求通常只能以离散的、较大的步长进行调整

   o  参与者可能对如何响应有特定的意愿——这可能不是减少发现拥塞的流的带宽需求

   o  编码通常对数据包丢失敏感，而实时性需求排除了通过重传来修复丢包的可能性

   本备忘录描述了两种拥塞控制算法，它们共同能够提供良好的性能，并与使用相同拥塞控制的其他视频流以及共享相同链路的TCP流合理地共享带宽。

   所使用的信令由实验性RTP头扩展和RTCP消息RFC 3550 [RFC3550]组成，如[abs-send-time]、[I-D.alvestrand-rmcat-remb]和[I-D.holmer-rmcat-transport-wide-cc-extensions]中定义。

---

## 1.1.  Mathematical notation conventions | 1.1. 数学符号约定

#### 原文

   The mathematics of this document have been transcribed from a more
   formula-friendly format.

   The following notational conventions are used:

   X_hat  An estimate of the true value of variable X - conventionally
      marked by a circumflex accent on top of the variable name.

   X(i)  The "i"th value of vector X - conventionally marked by a
      subscript i.

   E{X}  The expected value of the stochastic variable X

#### 中文

   本文档中的数学内容是从更适合公式表达的格式转录而来的。

   本文使用以下符号约定：

   X_hat  变量X真实值的估计值——通常在变量名上方加帽状标记（^）表示。

   X(i)  向量X的第i个值——通常用下标i表示。

   E{X}  随机变量X的期望值

---

## 2.  System model | 2. 系统模型

#### 原文

   The following elements are in the system:

   o  RTP packet - an RTP packet containing media data.

   o  Group of packets - a set of RTP packets transmitted from the
      sender uniquely identified by the group departure and group
      arrival time (absolute send time) [abs-send-time].  These could be
      video packets, audio packets, or a mix of audio and video packets.

   o  Incoming media stream - a stream of frames consisting of RTP
      packets.

   o  RTP sender - sends the RTP stream over the network to the RTP
      receiver.  It generates the RTP timestamp and the abs-send-time
      header extension

   o  RTP receiver - receives the RTP stream, marks the time of arrival.

   o  RTCP sender at RTP receiver - sends receiver reports, REMB
      messages and transport-wide RTCP feedback messages.

   o  RTCP receiver at RTP sender - receives receiver reports and REMB
      messages and transport-wide RTCP feedback messages, reports these
      to the sender side controller.

   o  RTCP receiver at RTP receiver, receives sender reports from the
      sender.

   o  Loss-based controller - takes loss rate measurement, round trip
      time measurement and REMB messages, and computes a target sending
      bitrate.

   o  Delay-based controller - takes the packet arrival info, either at
      the RTP receiver, or from the feedback received by the RTP sender,
      and computes a maximum bitrate which it passes to the loss-based
      controller.

   Together, loss-based controller and delay-based controller implement
   the congestion control algorithm.

#### 中文

   系统中包含以下元素：

   o  RTP数据包——包含媒体数据的RTP数据包。

   o  数据包组——一组从发送端传输的RTP数据包，由组的出发时间和组的到达时间（绝对发送时间）[abs-send-time]唯一标识。这些数据包可以是视频数据包、音频数据包，或音频和视频数据包的混合。

   o  传入媒体流——由RTP数据包组成的帧序列。

   o  RTP发送端——通过网络将RTP流发送到RTP接收端。它生成RTP时间戳和abs-send-time头扩展。

   o  RTP接收端——接收RTP流，记录到达时间。

   o  RTP接收端的RTCP发送者——发送接收端报告、REMB消息和传输范围RTCP反馈消息。

   o  RTP发送端的RTCP接收者——接收接收端报告、REMB消息和传输范围RTCP反馈消息，并将这些报告给发送端控制器。

   o  RTP接收端的RTCP接收者——从发送端接收发送端报告。

   o  基于丢包的控制器——获取丢包率测量、往返时间测量和REMB消息，计算目标发送比特率。

   o  基于延迟的控制器——在RTP接收端或通过RTP发送端收到的反馈中获取数据包到达信息，计算最大比特率并将其传递给基于丢包的控制器。

   基于丢包的控制器和基于延迟的控制器共同实现拥塞控制算法。

---

## 3.  Feedback and extensions | 3. 反馈与扩展

#### 原文

   There are two ways to implement the proposed algorithm.  One where
   both the controllers are running at the send-side, and one where the
   delay-based controller runs on the receive-side and the loss-based
   controller runs on the send-side.

   The first version can be realized by using a per-packet feedback
   protocol as described in
   [I-D.holmer-rmcat-transport-wide-cc-extensions].  Here, the RTP
   receiver will record the arrival time and the transport-wide sequence
   number of each received packet, which will be sent back to the sender
   periodically using the transport-wide feedback message.  The
   RECOMMENDED feedback interval is once per received video frame or at
   least once every 30 ms if audio-only or multi-stream.  If the
   feedback overhead needs to be limited this interval can be increased
   to 100 ms.

   The sender will map the received {sequence number, arrival time}
   pairs to the send-time of each packet covered by the feedback report,
   and feed those timestamps to the delay-based controller.  It will
   also compute a loss ratio based on the sequence numbers in the
   feedback message.

   The second version can be realized by having a delay-based controller
   at the receive-side, monitoring and processing the arrival time and
   size of incoming packets.  The sender SHOULD use the abs-send-time
   RTP header extension [abs-send-time] to enable the receiver to
   compute the inter-group delay variation.  The output from the delay-
   based controller will be a bitrate, which will be sent back to the
   sender using the REMB feedback message [I-D.alvestrand-rmcat-remb].
   The packet loss ratio is sent back via RTCP receiver reports.  At the
   sender the bitrate in the REMB message and the fraction of packets
   lost are fed into the loss-based controller, which outputs a final
   target bitrate.  It is RECOMMENDED to send the REMB message as soon
   as congestion is detected, and otherwise at least once every second.

#### 中文

   实现所提出的算法有两种方式。一种是两个控制器都在发送端运行，另一种是基于延迟的控制器在接收端运行，基于丢包的控制器在发送端运行。

   第一种方式可以通过使用如[I-D.holmer-rmcat-transport-wide-cc-extensions]中所述的逐包反馈协议来实现。在此方式中，RTP接收端记录每个接收数据包的到达时间和传输范围序列号，并使用传输范围反馈消息周期性地发送回发送端。推荐的反馈间隔为每接收一个视频帧一次，或者如果是纯音频或多流，则至少每30毫秒一次。如果需要限制反馈开销，此间隔可以增加到100毫秒。

   发送端将接收到的{序列号, 到达时间}对映射到反馈报告覆盖的每个数据包的发送时间，并将这些时间戳输入基于延迟的控制器。它还将根据反馈消息中的序列号计算丢包率。

   第二种方式可以通过在接收端设置基于延迟的控制器来实现，监控和处理传入数据包的到达时间和大小。发送端应使用abs-send-time RTP头扩展[abs-send-time]，以使接收端能够计算组间延迟变化。基于延迟的控制器的输出将是一个比特率，该比特率将使用REMB反馈消息[I-D.alvestrand-rmcat-remb]发送回发送端。丢包率通过RTCP接收端报告发送回。在发送端，REMB消息中的比特率和丢包比例被输入到基于丢包的控制器中，该控制器输出最终的目标比特率。建议一旦检测到拥塞就发送REMB消息，否则至少每秒发送一次。

---

## 4.  Sending Engine | 4. 发送引擎

#### 原文

   Pacing is used to actuate the target bitrate computed by the
   controllers.

   When media encoder produces data, this is fed into a Pacer queue.
   The Pacer sends a group of packets to the network every burst_time
   interval.  RECOMMENDED value for burst_time is 5 ms.  The size of a
   group of packets is computed as the product between the target
   bitrate and the burst_time.

#### 中文

   使用 pacing（速率平滑）来执行控制器计算的目标比特率。

   当媒体编码器产生数据时，数据被送入Pacer队列。Pacer每隔burst_time间隔向网络发送一组数据包。推荐的burst_time值为5毫秒。一组数据包的大小计算为目标比特率与burst_time的乘积。

---

## 5.  Delay-based control | 5. 基于延迟的控制

#### 原文

   The delay-based control algorithm can be further decomposed into four
   parts: a pre-filtering, an arrival-time filter, an over-use detector,
   and a rate controller.

#### 中文

   基于延迟的控制算法可以进一步分解为四个部分：预过滤、到达时间滤波器、过度使用检测器和速率控制器。

---

## 5.1.  Arrival-time model | 5.1. 到达时间模型

#### 原文

   This section describes an adaptive filter that continuously updates
   estimates of network parameters based on the timing of the received
   groups of packets.

   We define the inter-arrival time, t(i) - t(i-1), as the difference in
   arrival time of two groups of packets.  Correspondingly, the inter-
   departure time, T(i) - T(i-1), is defined as the difference in
   departure-time of two groups of packets.  Finally, the inter-group
   delay variation, d(i), is defined as the difference between the
   inter-arrival time and the inter-departure time.  Or interpreted
   differently, as the difference between the delay of group i and group
   i-1.

     d(i) = t(i) - t(i-1) - (T(i) - T(i-1))

   An inter-departure time is computed between consecutive groups as
   T(i) - T(i-1), where T(i) is the departure timestamp of the last
   packet in the current packet group being processed.  Any packets
   received out of order are ignored by the arrival-time model.

   Each group is assigned a receive time t(i), which corresponds to the
   time at which the last packet of the group was received.  A group is
   delayed relative to its predecessor if t(i) - t(i-1) > T(i) - T(i-1),
   i.e., if the inter-arrival time is larger than the inter-departure
   time.

   We can model the inter-group delay variation as:

     d(i) = w(i)

   Here, w(i) is a sample from a stochastic process W, which is a
   function of the link capacity, the current cross traffic, and the
   current sent bitrate.  We model W as a white Gaussian process.  If we
   are over-using the channel we expect the mean of w(i) to increase,
   and if a queue on the network path is being emptied, the mean of w(i)
   will decrease; otherwise the mean of w(i) will be zero.

   Breaking out the mean, m(i), from w(i) to make the process zero mean,
   we get

   Equation 1

     d(i) = m(i) + v(i)

   The noise term v(i) represents network jitter and other delay effects
   not captured by the model.

#### 中文

   本节描述一个自适应滤波器，该滤波器基于接收到的数据包组的时间信息持续更新网络参数的估计。

   我们将组间到达时间 t(i) - t(i-1) 定义为两组数据包到达时间的差值。相应地，组间出发时间 T(i) - T(i-1) 定义为两组数据包出发时间的差值。最后，组间延迟变化 d(i) 定义为组间到达时间与组间出发时间之间的差值。或者从不同角度解释，即组i与组i-1之间延迟的差值。

     d(i) = t(i) - t(i-1) - (T(i) - T(i-1))

   组间出发时间在连续组之间计算为 T(i) - T(i-1)，其中 T(i) 是当前正在处理的数据包组中最后一个数据包的出发时间戳。任何乱序接收的数据包都会被到达时间模型忽略。

   每个组被分配一个接收时间 t(i)，对应于该组最后一个数据包被接收的时间。如果 t(i) - t(i-1) > T(i) - T(i-1)，即组间到达时间大于组间出发时间，则该组相对于其前一组延迟。

   我们可以将组间延迟变化建模为：

     d(i) = w(i)

   其中，w(i) 是来自随机过程 W 的样本，W 是链路容量、当前交叉流量和当前发送比特率的函数。我们将 W 建模为白高斯过程。如果信道被过度使用，我们预期 w(i) 的均值会增加；如果网络路径上的队列正在被清空，w(i) 的均值将减小；否则 w(i) 的均值将为零。

   从 w(i) 中分离出均值 m(i) 以使过程为零均值，我们得到

   公式1

     d(i) = m(i) + v(i)

   噪声项 v(i) 表示网络抖动和其他模型未捕获的延迟效应。

---

## 5.2.  Pre-filtering | 5.2. 预过滤

#### 原文

   The pre-filtering aims at handling delay transients caused by channel
   outages.  During an outage, packets being queued in network buffers,
   for reasons unrelated to congestion, are delivered in a burst when
   the outage ends.

   The pre-filtering merges together groups of packets that arrive in a
   burst.  Packets are merged in the same group if one of these two
   conditions holds:

   o  A sequence of packets which are sent within a burst_time interval
      constitute a group.

   o  A Packet which has an inter-arrival time less than burst_time and
      an inter-group delay variation d(i) less than 0 is considered
      being part of the current group of packets.

#### 中文

   预过滤旨在处理由信道中断引起的延迟瞬变。在中断期间，由于与拥塞无关的原因在网络缓冲区中排队的数据包在中断结束时以突发方式传递。

   预过滤将突发到达的数据包组合并在一起。如果满足以下两个条件之一，数据包将被合并到同一组中：

   o  在 burst_time 间隔内发送的一系列数据包构成一个组。

   o  组间到达时间小于 burst_time 且组间延迟变化 d(i) 小于 0 的数据包被视为当前数据包组的一部分。

---

## 5.3.  Arrival-time filter | 5.3. 到达时间滤波器

#### 原文

   The parameter d(i) is readily available for each group of packets, i
   > 1.  We want to estimate m(i) and use this estimate to detect
   whether or not the bottleneck link is over-used.  The parameter can
   be estimated by any adaptive filter - we are using the Kalman filter.

   Let m(i) be the estimate at time i

   We model the state evolution from time i to time i+1 as

     m(i+1) = m(i) + u(i)

   where u(i) is the state noise that we model as a stationary process
   with Gaussian statistic with zero mean and variance

     q(i) = E{u(i)^2}

   q(i) is RECOMMENDED equal to 10^-3

   Given equation 1 we get

       d(i) = m(i) + v(i)

   where v(i) is zero mean white Gaussian measurement noise with
   variance var_v = E{v(i)^2}

   The Kalman filter recursively updates our estimate m_hat(i) as

     z(i) = d(i) - m_hat(i-1)

     m_hat(i) = m_hat(i-1) + z(i) * k(i)

                        e(i-1) + q(i)
     k(i) = ----------------------------------------
                var_v_hat(i) + (e(i-1) + q(i))

     e(i) = (1 - k(i)) * (e(i-1) + q(i))

   The variance var_v(i) = E{v(i)^2} is estimated using an exponential
   averaging filter, modified for variable sampling rate

     var_v_hat(i) = max(alpha * var_v_hat(i-1) + (1-alpha) * z(i)^2, 1)

     alpha = (1-chi)^(30/(1000 * f_max))

   where f_max = max {1/(T(j) - T(j-1))} for j in i-K+1,...,i is the
   highest rate at which the last K packet groups have been received and
   chi is a filter coefficient typically chosen as a number in the
   interval [0.1, 0.001].  Since our assumption that v(i) should be zero
   mean WGN is less accurate in some cases, we have introduced an
   additional outlier filter around the updates of var_v_hat.  If z(i) >
   3*sqrt(var_v_hat) the filter is updated with 3*sqrt(var_v_hat) rather
   than z(i).  For instance v(i) will not be white in situations where
   packets are sent at a higher rate than the channel capacity, in which
   case they will be queued behind each other.

#### 中文

   对于每组数据包（i > 1），参数 d(i) 可以直接获得。我们希望估计 m(i) 并使用该估计来检测瓶颈链路是否被过度使用。该参数可以通过任何自适应滤波器来估计——我们使用的是卡尔曼滤波器。

   令 m(i) 为时间 i 的估计值

   我们将从时间 i 到时间 i+1 的状态演化建模为

     m(i+1) = m(i) + u(i)

   其中 u(i) 是状态噪声，我们将其建模为一个具有零均值的平稳高斯统计过程，方差为

     q(i) = E{u(i)^2}

   推荐 q(i) 等于 10^-3

   根据公式1，我们得到

       d(i) = m(i) + v(i)

   其中 v(i) 是零均值白高斯测量噪声，方差为 var_v = E{v(i)^2}

   卡尔曼滤波器递归地更新我们的估计 m_hat(i)，如下所示：

     z(i) = d(i) - m_hat(i-1)

     m_hat(i) = m_hat(i-1) + z(i) * k(i)

                        e(i-1) + q(i)
     k(i) = ----------------------------------------
                var_v_hat(i) + (e(i-1) + q(i))

     e(i) = (1 - k(i)) * (e(i-1) + q(i))

   方差 var_v(i) = E{v(i)^2} 使用指数平均滤波器进行估计，并针对可变采样率进行了修改：

     var_v_hat(i) = max(alpha * var_v_hat(i-1) + (1-alpha) * z(i)^2, 1)

     alpha = (1-chi)^(30/(1000 * f_max))

   其中 f_max = max {1/(T(j) - T(j-1))}，对于 j 在 i-K+1,...,i 范围内，是最近 K 个数据包组接收的最高速率，chi 是一个滤波器系数，通常选择在区间 [0.1, 0.001] 内的值。由于我们关于 v(i) 应为零均值白高斯噪声的假设在某些情况下不够精确，我们在 var_v_hat 的更新中引入了额外的离群值滤波器。如果 z(i) > 3*sqrt(var_v_hat)，则使用 3*sqrt(var_v_hat) 而不是 z(i) 来更新滤波器。例如，当数据包以高于信道容量的速率发送时，v(i) 将不会是白噪声，在这种情况下数据包会相互排队。

---

## 5.4.  Over-use detector | 5.4. 过度使用检测器

#### 原文

   The inter-group delay variation estimate m(i), obtained as the output
   of the arrival-time filter, is compared with a threshold
   del_var_th(i).  An estimate above the threshold is considered as an
   indication of over-use.  Such an indication is not enough for the
   detector to signal over-use to the rate control subsystem.  A
   definitive over-use will be signaled only if over-use has been
   detected for at least overuse_time_th milliseconds.  However, if m(i)
   < m(i-1), over-use will not be signaled even if all the above
   conditions are met.  Similarly, the opposite state, under-use, is
   detected when m(i) < -del_var_th(i).  If neither over-use nor under-
   use is detected, the detector will be in the normal state.

   The threshold del_var_th has a remarkable impact on the overall
   dynamics and performance of the algorithm.  In particular, it has
   been shown that using a static threshold del_var_th, a flow
   controlled by the proposed algorithm can be starved by a concurrent
   TCP flow [Pv13].  This starvation can be avoided by increasing the
   threshold del_var_th to a sufficiently large value.

   The reason is that, by using a larger value of del_var_th, a larger
   queuing delay can be tolerated, whereas with a small del_var_th, the
   over-use detector quickly reacts to a small increase in the offset
   estimate m(i) by generating an over-use signal that reduces the
   delay-based estimate of the available bandwidth A_hat (see
   Section 4.4).  Thus, it is necessary to dynamically tune the
   threshold del_var_th to get good performance in the most common
   scenarios, such as when competing with loss-based flows.

   For this reason, we propose to vary the threshold del_var_th(i)
   according to the following dynamic equation:

del_var_th(i) =
         del_var_th(i-1) + (t(i)-t(i-1)) * K(i) * (|m(i)|-del_var_th(i-1))

   with K(i)=K_d if |m(i)| < del_var_th(i-1) or K(i)=K_u otherwise.  The
   rationale is to increase del_var_th(i) when m(i) is outside of the
   range [-del_var_th(i-1),del_var_th(i-1)], whereas, when the offset
   estimate m(i) falls back into the range, del_var_th is decreased.  In
   this way when m(i) increases, for instance due to a TCP flow entering
   the same bottleneck, del_var_th(i) increases and avoids the
   uncontrolled generation of over-use signals which may lead to
   starvation of the flow controlled by the proposed algorithm [Pv13].
   Moreover, del_var_th(i) SHOULD NOT be updated if this condition
   holds:

     |m(i)| - del_var_th(i) > 15

   It is also RECOMMENDED to clamp del_var_th(i) to the range [6, 600],
   since a too small del_var_th(i) can cause the detector to become
   overly sensitive.

   On the other hand, when m(i) falls back into the range
   [-del_var_th(i-1),del_var_th(i-1)] the threshold del_var_th(i) is
   decreased so that a lower queuing delay can be achieved.

   It is RECOMMENDED to choose K_u > K_d so that the rate at which
   del_var_th is increased is higher than the rate at which it is
   decreased.  With this setting it is possible to increase the
   threshold in the case of a concurrent TCP flow and prevent starvation
   as well as enforcing intra-protocol fairness.  RECOMMENDED values for
   del_var_th(0), overuse_time_th, K_u and K_d are respectively 12.5 ms,
   10 ms, 0.01 and 0.00018.

#### 中文

   从到达时间滤波器输出获得的组间延迟变化估计 m(i) 与阈值 del_var_th(i) 进行比较。高于阈值的估计被认为是过度使用的指示。这种指示不足以让检测器向速率控制子系统发出过度使用信号。只有当过度使用被持续检测到至少 overuse_time_th 毫秒时，才会发出确定的过度使用信号。但是，如果 m(i) < m(i-1)，即使满足所有上述条件，也不会发出过度使用信号。类似地，相反状态——未充分利用（under-use）——在 m(i) < -del_var_th(i) 时检测到。如果既未检测到过度使用也未检测到未充分利用，检测器将处于正常状态。

   阈值 del_var_th 对算法的整体动态和性能有显著影响。特别是，已有研究表明，使用静态阈值 del_var_th 时，由所提出算法控制的流可能被并发的TCP流饿死 [Pv13]。通过将阈值 del_var_th 增加到足够大的值可以避免这种饿死。

   原因是，通过使用较大的 del_var_th 值，可以容忍更大的排队延迟；而使用较小的 del_var_th 时，过度使用检测器会对偏移估计 m(i) 的微小增加快速反应，生成过度使用信号来降低基于延迟的可用带宽估计 A_hat（参见第4.4节）。因此，有必要动态调整阈值 del_var_th，以在最常见的场景中获得良好性能，例如与基于丢包的流竞争时。

   为此，我们建议根据以下动态方程来变化阈值 del_var_th(i)：

del_var_th(i) =
         del_var_th(i-1) + (t(i)-t(i-1)) * K(i) * (|m(i)|-del_var_th(i-1))

   其中，如果 |m(i)| < del_var_th(i-1)，则 K(i)=K_d，否则 K(i)=K_u。其原理是当 m(i) 超出范围 [-del_var_th(i-1), del_var_th(i-1)] 时增加 del_var_th(i)，而当偏移估计 m(i) 回落到该范围内时减小 del_var_th。这样，当 m(i) 增加时——例如由于TCP流进入同一瓶颈——del_var_th(i) 增加，避免过度使用信号的不受控生成，否则可能导致由所提出算法控制的流被饿死 [Pv13]。此外，如果满足以下条件，则不应更新 del_var_th(i)：

     |m(i)| - del_var_th(i) > 15

   同时建议将 del_var_th(i) 限制在范围 [6, 600] 内，因为太小的 del_var_th(i) 会导致检测器变得过于敏感。

   另一方面，当 m(i) 回落到范围 [-del_var_th(i-1), del_var_th(i-1)] 内时，阈值 del_var_th(i) 被减小，从而实现更低的排队延迟。

   建议选择 K_u > K_d，使得 del_var_th 增加的速率高于其减少的速率。通过这种设置，可以在有并发TCP流的情况下增加阈值，防止饿死并强制实现协议内公平性。推荐的 del_var_th(0)、overuse_time_th、K_u 和 K_d 值分别为 12.5 毫秒、10 毫秒、0.01 和 0.00018。

---

## 5.5.  Rate control | 5.5. 速率控制

#### 原文

   The rate control is split in two parts, one controlling the bandwidth
   estimate based on delay, and one controlling the bandwidth estimate
   based on loss.  Both are designed to increase the estimate of the
   available bandwidth A_hat as long as there is no detected congestion
   and to ensure that we will eventually match the available bandwidth
   of the channel and detect an over-use.

   As soon as over-use has been detected, the available bandwidth
   estimated by the delay-based controller is decreased.  In this way we
   get a recursive and adaptive estimate of the available bandwidth.

   In this document we make the assumption that the rate control
   subsystem is executed periodically and that this period is constant.

   The rate control subsystem has 3 states: Increase, Decrease and Hold.
   "Increase" is the state when no congestion is detected; "Decrease" is
   the state where congestion is detected, and "Hold" is a state that
   waits until built-up queues have drained before going to "increase"
   state.

   The state transitions (with blank fields meaning "remain in state")
   are:

   +----+--------+-----------+------------+--------+
   |     \ State |   Hold    |  Increase  |Decrease|
   |      \      |           |            |        |
   | Signal\     |           |            |        |
   +--------+----+-----------+------------+--------+
   |  Over-use   | Decrease  |  Decrease  |        |
   +-------------+-----------+------------+--------+
   |  Normal     | Increase  |            |  Hold  |
   +-------------+-----------+------------+--------+
   |  Under-use  |           |   Hold     |  Hold  |
   +-------------+-----------+------------+--------+

   The subsystem starts in the increase state, where it will stay until
   over-use or under-use has been detected by the detector subsystem.
   On every update the delay-based estimate of the available bandwidth
   is increased, either multiplicatively or additively, depending on its
   current state.

   The system does a multiplicative increase if the current bandwidth
   estimate appears to be far from convergence, while it does an
   additive increase if it appears to be closer to convergence.  We
   assume that we are close to convergence if the currently incoming
   bitrate, R_hat(i), is close to an average of the incoming bitrates at
   the time when we previously have been in the Decrease state.  "Close"
   is defined as three standard deviations around this average.  It is
   RECOMMENDED to measure this average and standard deviation with an
   exponential moving average with the smoothing factor 0.95, as it is
   expected that this average covers multiple occasions at which we are
   in the Decrease state.  Whenever valid estimates of these statistics
   are not available, we assume that we have not yet come close to
   convergence and therefore remain in the multiplicative increase
   state.

   If R_hat(i) increases above three standard deviations of the average
   max bitrate, we assume that the current congestion level has changed,
   at which point we reset the average max bitrate and go back to the
   multiplicative increase state.

   R_hat(i) is the incoming bitrate measured by the delay-based
   controller over a T seconds window:

     R_hat(i) = 1/T * sum(L(j)) for j from 1 to N(i)

   N(i) is the number of packets received the past T seconds and L(j) is
   the payload size of packet j.  A window between 0.5 and 1 second is
   RECOMMENDED.

   During multiplicative increase, the estimate is increased by at most
   8% per second.

     eta = 1.08^min(time_since_last_update_ms / 1000, 1.0)
     A_hat(i) = eta * A_hat(i-1)

   During the additive increase the estimate is increased with at most
   half a packet per response_time interval.  The response_time interval
   is estimated as the round-trip time plus 100 ms as an estimate of
   over-use estimator and detector reaction time.

    response_time_ms = 100 + rtt_ms
    alpha = 0.5 * min(time_since_last_update_ms / response_time_ms, 1.0)
    A_hat(i) = A_hat(i-1) + max(1000, alpha * expected_packet_size_bits)

   expected_packet_size_bits is used to get a slightly slower slope for
   the additive increase at lower bitrates.  It can for instance be
   computed from the current bitrate by assuming a frame rate of 30
   frames per second:

     bits_per_frame = A_hat(i-1) / 30
     packets_per_frame = ceil(bits_per_frame / (1200 * 8))
     avg_packet_size_bits = bits_per_frame / packets_per_frame

   Since the system depends on over-using the channel to verify the
   current available bandwidth estimate, we must make sure that our
   estimate does not diverge from the rate at which the sender is
   actually sending.  Thus, if the sender is unable to produce a bit
   stream with the bitrate the congestion controller is asking for, the
   available bandwidth estimate should stay within a given bound.
   Therefore we introduce a threshold

     A_hat(i) < 1.5 * R_hat(i)

   When an over-use is detected the system transitions to the decrease
   state, where the delay-based available bandwidth estimate is
   decreased to a factor times the currently incoming bitrate.

     A_hat(i) = beta * R_hat(i)

   beta is typically chosen to be in the interval [0.8, 0.95], 0.85 is
   the RECOMMENDED value.

   When the detector signals under-use to the rate control subsystem, we
   know that queues in the network path are being emptied, indicating
   that our available bandwidth estimate A_hat is lower than the actual
   available bandwidth.  Upon that signal the rate control subsystem
   will enter the hold state, where the receive-side available bandwidth
   estimate will be held constant while waiting for the queues to
   stabilize at a lower level - a way of keeping the delay as low as
   possible.  This decrease of delay is wanted, and expected,
   immediately after the estimate has been reduced due to over-use, but
   can also happen if the cross traffic over some links is reduced.

   It is RECOMMENDED that the routine to update A_hat(i) is run at least
   once every response_time interval.

#### 中文

   速率控制分为两部分：一部分基于延迟控制带宽估计，另一部分基于丢包控制带宽估计。两者的设计目标都是在未检测到拥塞时增加可用带宽估计 A_hat，并确保最终匹配信道的可用带宽并检测到过度使用。

   一旦检测到过度使用，基于延迟的控制器估计的可用带宽将被降低。通过这种方式，我们获得了可用带宽的递归和自适应估计。

   在本文档中，我们假设速率控制子系统是周期性执行的，且该周期是恒定的。

   速率控制子系统有3种状态：增加（Increase）、降低（Decrease）和保持（Hold）。"增加"是未检测到拥塞时的状态；"降低"是检测到拥塞时的状态；"保持"是在进入"增加"状态之前等待累积的队列排空的状态。

   状态转换（空白字段表示"保持原状态"）如下：

   +----+--------+-----------+------------+--------+
   |     \ State |   Hold    |  Increase  |Decrease|
   |      \      |           |            |        |
   | Signal\     |           |            |        |
   +--------+----+-----------+------------+--------+
   |  Over-use   | Decrease  |  Decrease  |        |
   +-------------+-----------+------------+--------+
   |  Normal     | Increase  |            |  Hold  |
   +-------------+-----------+------------+--------+
   |  Under-use  |           |   Hold     |  Hold  |
   +-------------+-----------+------------+--------+

   子系统从增加状态开始，在此状态下保持，直到检测器子系统检测到过度使用或未充分利用。每次更新时，基于延迟的可用带宽估计根据其当前状态进行乘法或加法增加。

   如果当前带宽估计似乎离收敛较远，系统执行乘法增加；如果看起来更接近收敛，则执行加法增加。如果当前输入比特率 R_hat(i) 接近我们先前处于降低状态时的输入比特率的平均值，我们假设已经接近收敛。"接近"定义为该平均值的三个标准差范围内。建议使用平滑因子为 0.95 的指数移动平均来测量该平均值和标准差，因为预期该平均值覆盖了多个处于降低状态的时刻。当没有这些统计量的有效估计时，我们假设尚未接近收敛，因此保持在乘法增加状态。

   如果 R_hat(i) 增加到超过平均最大比特率的三倍标准差，我们假设当前拥塞水平已经改变，此时我们重置平均最大比特率并回到乘法增加状态。

   R_hat(i) 是基于延迟的控制器在 T 秒窗口内测量的输入比特率：

     R_hat(i) = 1/T * sum(L(j)) for j from 1 to N(i)

   N(i) 是过去 T 秒内接收的数据包数量，L(j) 是数据包 j 的有效载荷大小。推荐窗口在 0.5 到 1 秒之间。

   在乘法增加期间，估计值每秒增加最多 8%。

     eta = 1.08^min(time_since_last_update_ms / 1000, 1.0)
     A_hat(i) = eta * A_hat(i-1)

   在加法增加期间，估计值每个 response_time 间隔最多增加半个数据包。response_time 间隔估计为往返时间加上 100 毫秒，作为过度使用估计器和检测器反应时间的估计。

    response_time_ms = 100 + rtt_ms
    alpha = 0.5 * min(time_since_last_update_ms / response_time_ms, 1.0)
    A_hat(i) = A_hat(i-1) + max(1000, alpha * expected_packet_size_bits)

   expected_packet_size_bits 用于在较低比特率下使加法增加的斜率略慢。例如，可以通过假设每秒30帧的帧率从当前比特率计算：

     bits_per_frame = A_hat(i-1) / 30
     packets_per_frame = ceil(bits_per_frame / (1200 * 8))
     avg_packet_size_bits = bits_per_frame / packets_per_frame

   由于系统依赖对信道的过度使用来验证当前的可用带宽估计，我们必须确保我们的估计不会偏离发送端实际发送的速率。因此，如果发送端无法以拥塞控制器要求的比特率产生比特流，可用带宽估计应保持在给定的范围内。为此我们引入一个阈值：

     A_hat(i) < 1.5 * R_hat(i)

   当检测到过度使用时，系统转换到降低状态，在此状态下基于延迟的可用带宽估计被降低为当前输入比特率的一个因子倍。

     A_hat(i) = beta * R_hat(i)

   beta 通常选择在区间 [0.8, 0.95] 内，推荐值为 0.85。

   当检测器向速率控制子系统发出未充分利用信号时，我们知道网络路径上的队列正在被清空，这表明我们的可用带宽估计 A_hat 低于实际可用带宽。收到该信号后，速率控制子系统将进入保持状态，在此状态下接收端可用带宽估计将保持不变，等待队列稳定在较低水平——这是一种尽量保持低延迟的方式。这种延迟的降低是期望的，也是预期的，通常发生在估计因过度使用而降低后不久，但也可能在经过某些链路的交叉流量减少时发生。

   建议更新 A_hat(i) 的例程至少每个 response_time 间隔运行一次。

---

## 5.6.  Parameters settings | 5.6. 参数设置

#### 原文

   +-----------------+-----------------------------------+-------------+
   | Parameter       | Description                       | RECOMMENDED |
   |                 |                                   | Value       |
   +-----------------+-----------------------------------+-------------+
   | burst_time      | Time limit in milliseconds        | 5 ms        |
   |                 | between packet bursts which       |             |
   |                 | identifies a group                |             |
   | q               | State noise covariance matrix     | q = 10^-3   |
   | e(0)            | Initial value of the  system      | e(0) = 0.1  |
   |                 | error covariance                  |             |
   | chi             | Coefficient used  for the         | [0.1,       |
   |                 | measured noise variance           | 0.001]      |
   | del_var_th(0)   | Initial value for the adaptive    | 12.5 ms     |
   |                 | threshold                         |             |
   | overuse_time_th | Time required to trigger an       | 10 ms       |
   |                 | overuse signal                    |             |
   | K_u             | Coefficient for the adaptive      | 0.01        |
   |                 | threshold                         |             |
   | K_d             | Coefficient for the adaptive      | 0.00018     |
   |                 | threshold                         |             |
   | T               | Time window for measuring the     | [0.5, 1] s  |
   |                 | received bitrate                  |             |
   | beta            | Decrease rate factor              | 0.85        |
   +-----------------+-----------------------------------+-------------+

          Table 1: RECOMMENDED values for delay based controller

                                  Table 1

#### 中文

   +-----------------+-----------------------------------+-------------+
   | 参数            | 描述                              | 推荐值       |
   +-----------------+-----------------------------------+-------------+
   | burst_time      | 数据包突发之间的时间限制          | 5 ms        |
   |                 | （毫秒），用于标识一个组          |             |
   | q               | 状态噪声协方差矩阵                | q = 10^-3   |
   | e(0)            | 系统误差协方差的初始值            | e(0) = 0.1  |
   | chi             | 用于测量噪声方差的系数            | [0.1,       |
   |                 |                                   | 0.001]      |
   | del_var_th(0)   | 自适应阈值的初始值                | 12.5 ms     |
   | overuse_time_th | 触发过度使用信号所需的时间        | 10 ms       |
   | K_u             | 自适应阈值的系数（增加方向）      | 0.01        |
   | K_d             | 自适应阈值的系数（减小方向）      | 0.00018     |
   | T               | 测量接收比特率的时间窗口          | [0.5, 1] s  |
   | beta            | 降低速率因子                      | 0.85        |
   +-----------------+-----------------------------------+-------------+

          表1：基于延迟的控制器的推荐值

                                  表1

---

## 6.  Loss-based control | 6. 基于丢包的控制

#### 原文

   A second part of the congestion controller bases its decisions on the
   round-trip time, packet loss and available bandwidth estimates A_hat
   received from the delay-based controller.  The available bandwidth
   estimates computed by the loss-based controller are denoted with
   As_hat.

   The available bandwidth estimates A_hat produced by the delay-based
   controller are only reliable when the size of the queues along the
   path sufficiently large.  If the queues are very short, over-use will
   only be visible through packet losses, which are not used by the
   delay-based controller.

   The loss-based controller SHOULD run every time feedback from the
   receiver is received.

   o  If 2-10% of the packets have been lost since the previous report
      from the receiver, the sender available bandwidth estimate
      As_hat(i) will be kept unchanged.

   o  If more than 10% of the packets have been lost a new estimate is
      calculated as As_hat(i) = As_hat(i-1)(1-0.5p), where p is the loss
      ratio.

   o  As long as less than 2% of the packets have been lost As_hat(i)
      will be increased as As_hat(i) = 1.05(As_hat(i-1))

   The loss-based estimate As_hat is compared with the delay-based
   estimate A_hat.  The actual sending rate is set as the minimum
   between As_hat and A_hat.

   We motivate the packet loss thresholds by noting that if the
   transmission channel has a small amount of packet loss due to over-
   use, that amount will soon increase if the sender does not adjust his
   bitrate.  Therefore we will soon enough reach above the 10% threshold
   and adjust As_hat(i).  However, if the packet loss ratio does not
   increase, the losses are probably not related to self-inflicted
   congestion and therefore we should not react on them.

#### 中文

   拥塞控制器的第二部分基于往返时间、数据包丢失以及从基于延迟的控制器接收的可用带宽估计 A_hat 来做出决策。由基于丢包的控制器计算的可用带宽估计用 As_hat 表示。

   基于延迟的控制器产生的可用带宽估计 A_hat 仅在路径上的队列大小足够大时才可靠。如果队列非常短，过度使用只能通过数据包丢失可见，而基于延迟的控制器不使用这些丢包信息。

   基于丢包的控制器应在每次收到接收端反馈时运行。

   o  如果自上次接收端报告以来有 2-10% 的数据包丢失，发送端可用带宽估计 As_hat(i) 将保持不变。

   o  如果有超过 10% 的数据包丢失，新的估计值计算为 As_hat(i) = As_hat(i-1)(1-0.5p)，其中 p 是丢包率。

   o  只要少于 2% 的数据包丢失，As_hat(i) 将按 As_hat(i) = 1.05(As_hat(i-1)) 增加。

   基于丢包的估计值 As_hat 与基于延迟的估计值 A_hat 进行比较。实际发送速率设置为 As_hat 和 A_hat 中的最小值。

   我们解释丢包阈值的原因：如果传输信道由于过度使用而有少量数据包丢失，如果发送端不调整其比特率，丢失量将很快增加。因此，我们很快会达到超过10%的阈值并调整 As_hat(i)。然而，如果丢包率没有增加，这些丢失可能与自身造成的拥塞无关，因此我们不应对其做出反应。

---

## 7.  Interoperability Considerations | 7. 互操作性考量

#### 原文

   In case a sender implementing these algorithms talks to a receiver
   which do not implement any of the proposed RTCP messages and RTP
   header extensions, it is suggested that the sender monitors RTCP
   receiver reports and uses the fraction of lost packets and the round-
   trip time as input to the loss-based controller.  The delay-based
   controller should be left disabled.

#### 中文

   如果实现这些算法的发送端与不实现任何所提议的RTCP消息和RTP头扩展的接收端通信，建议发送端监控RTCP接收端报告，并使用丢包比例和往返时间作为基于丢包的控制器的输入。基于延迟的控制器应保持禁用状态。

---

## 8.  Implementation Experience | 8. 实现经验

#### 原文

   This algorithm has been implemented in the open-source WebRTC
   project, has been in use in Chrome since M23, and is being used by
   Google Hangouts.

   Deployment of the algorithm have revealed problems related to, e.g,
   congested or otherwise problematic WiFi networks, which have led to
   algorithm improvements.  The algorithm has also been tested in a
   multi-party conference scenario with a conference server which
   terminates the congestion control between endpoints.  This ensures
   that no assumptions are being made by the congestion control about
   maximum send and receive bitrates, etc., which typically is out of
   control for a conference server.

#### 中文

   该算法已在开源WebRTC项目中实现，自Chrome M23版本以来一直在使用，并被Google Hangouts使用。

   算法的部署揭示了与例如拥塞或有其他问题的WiFi网络相关的问题，这促使了算法的改进。该算法还已在多方会议场景中进行了测试，该场景使用会议服务器终止端点之间的拥塞控制。这确保了拥塞控制不会对最大发送和接收比特率等做出假设，而这些通常对会议服务器来说是不可控的。

---

## 9.  Further Work | 9. 后续工作

#### 原文

   This draft is offered as input to the congestion control discussion.

   Work that can be done on this basis includes:

   o  Considerations of integrated loss control: How loss and delay
      control can be better integrated, and the loss control improved.

   o  Considerations of locus of control: evaluate the performance of
      having all congestion control logic at the sender, compared to
      splitting logic between sender and receiver.

   o  Considerations of utilizing ECN as a signal for congestion
      estimation and link over-use detection.

#### 中文

   本草案作为对拥塞控制讨论的输入而提供。

   在此基础上的后续工作包括：

   o  集成丢包控制的考量：如何更好地集成丢包和延迟控制，以及改进丢包控制。

   o  控制位置的考量：评估将所有拥塞控制逻辑放在发送端的性能，与在发送端和接收端之间分割逻辑的性能进行比较。

   o  利用ECN作为拥塞估计和链路过度使用检测信号的考量。

---

## 10.  IANA Considerations | 10. IANA考量

#### 原文

   This document makes no request of IANA.

   Note to RFC Editor: this section may be removed on publication as an
   RFC.

#### 中文

   本文档不向IANA提出任何请求。

   给RFC编辑的说明：本节在作为RFC发布时可能会被删除。

---

## 11.  Security Considerations | 11. 安全考量

#### 原文

   An attacker with the ability to insert or remove messages on the
   connection would have the ability to disrupt rate control.  This
   could make the algorithm to produce either a sending rate under-
   utilizing the bottleneck link capacity, or a too high sending rate
   causing network congestion.

   In this case, the control information is carried inside RTP, and can
   be protected against modification or message insertion using SRTP,
   just as for the media.  Given that timestamps are carried in the RTP
   header, which is not encrypted, this is not protected against
   disclosure, but it seems hard to mount an attack based on timing
   information only.

#### 中文

   能够在连接上插入或删除消息的攻击者将有能力破坏速率控制。这可能导致算法产生低于瓶颈链路容量利用率的发送速率，或产生过高的发送速率导致网络拥塞。

   在这种情况下，控制信息在RTP内部承载，可以像媒体一样使用SRTP保护以防止修改或消息插入。鉴于时间戳承载在RTP头部且未加密，这不能防止信息泄露，但似乎很难仅基于时序信息发起攻击。

---

## 12.  Acknowledgements | 12. 致谢

#### 原文

   Thanks to Randell Jesup, Magnus Westerlund, Varun Singh, Tim Panton,
   Soo-Hyun Choo, Jim Gettys, Ingemar Johansson, Michael Welzl and
   others for providing valuable feedback on earlier versions of this
   draft.

#### 中文

   感谢Randell Jesup、Magnus Westerlund、Varun Singh、Tim Panton、Soo-Hyun Choo、Jim Gettys、Ingemar Johansson、Michael Welzl以及其他人对本草案早期版本提供的有价值的反馈。

---

## 13.  References | 13. 参考文献

### 13.1.  Normative References | 13.1. 规范性参考文献

#### 原文

   [I-D.alvestrand-rmcat-remb]
              Alvestrand, H., "RTCP message for Receiver Estimated
              Maximum Bitrate", draft-alvestrand-rmcat-remb-03 (work in
              progress), October 2013.

   [I-D.holmer-rmcat-transport-wide-cc-extensions]
              Holmer, S., Flodman, M., and E. Sprang, "RTP Extensions
              for Transport-wide Congestion Control", draft-holmer-
              rmcat-transport-wide-cc-extensions-00 (work in progress),
              March 2015.

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, March 1997.

   [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
              Jacobson, "RTP: A Transport Protocol for Real-Time
              Applications", STD 64, RFC 3550, July 2003.

   [abs-send-time]
              "RTP Header Extension for Absolute Sender Time",
              <http://www.webrtc.org/experiments/rtp-hdrext/
              abs-send-time>.

#### 中文

   [I-D.alvestrand-rmcat-remb]
              Alvestrand, H., "接收端估计最大比特率的RTCP消息",
              draft-alvestrand-rmcat-remb-03（进行中的工作），2013年10月。

   [I-D.holmer-rmcat-transport-wide-cc-extensions]
              Holmer, S., Flodman, M., and E. Sprang, "用于传输范围拥塞控制
              的RTP扩展", draft-holmer-rmcat-transport-wide-cc-extensions-00
              （进行中的工作），2015年3月。

   [RFC2119]  Bradner, S., "用于RFC中指示需求级别的关键词",
              BCP 14, RFC 2119, 1997年3月。

   [RFC3550]  Schulzrinne, H., Casner, S., Frederick, R., and V.
              Jacobson, "RTP：实时应用的传输协议",
              STD 64, RFC 3550, 2003年7月。

   [abs-send-time]
              "绝对发送者时间的RTP头扩展",
              <http://www.webrtc.org/experiments/rtp-hdrext/
              abs-send-time>.

---

### 13.2.  Informative References | 13.2. 信息性参考文献

#### 原文

   [Pv13]     De Cicco, L., Carlucci, G., and S. Mascolo, "Understanding
              the Dynamic Behaviour of the Google Congestion Control",
              Packet Video Workshop , December 2013.

   [RFC2914]  Floyd, S., "Congestion Control Principles", BCP 41, RFC
              2914, September 2000.

#### 中文

   [Pv13]     De Cicco, L., Carlucci, G., and S. Mascolo, "理解Google
              拥塞控制的动态行为", Packet Video Workshop,
              2013年12月。

   [RFC2914]  Floyd, S., "拥塞控制原则", BCP 41, RFC
              2914, 2000年9月。

---

## Appendix A.  Change log | 附录 A. 变更日志

### A.1.  Version -00 to -01 | A.1. 版本 -00 至 -01

#### 原文

   o  Added change log

   o  Added appendix outlining new extensions

   o  Added a section on when to send feedback to the end of section 3.3
      "Rate control", and defined min/max FB intervals.

   o  Added size of over-bandwidth estimate usage to "further work"
      section.

   o  Added startup considerations to "further work" section.

   o  Added sender-delay considerations to "further work" section.

   o  Filled in acknowledgments section from mailing list discussion.

#### 中文

   o  添加了变更日志

   o  添加了概述新扩展的附录

   o  在3.3节"速率控制"末尾添加了关于何时发送反馈的部分，并定义了最小/最大反馈间隔

   o  在"后续工作"部分添加了超出带宽估计使用的规模

   o  在"后续工作"部分添加了启动考量

   o  在"后续工作"部分添加了发送端延迟考量

   o  根据邮件列表讨论填写了致谢部分

---

### A.2.  Version -01 to -02 | A.2. 版本 -01 至 -02

#### 原文

   o  Defined the term "frame", incorporating the transmission time
      offset into its definition, and removed references to "video
      frame".

   o  Referred to "m(i)" from the text to make the derivation clearer.

   o  Made it clearer that we modify our estimates of available
      bandwidth, and not the true available bandwidth.

   o  Removed the appendixes outlining new extensions, added pointers to
      REMB draft and RFC 5450.

#### 中文

   o  定义了"帧"这个术语，将传输时间偏移纳入其定义，并删除了对"视频帧"的引用

   o  从正文中引用了"m(i)"以使推导更清晰

   o  更清楚地说明我们修改的是对可用带宽的估计，而不是真实的可用带宽

   o  删除了概述新扩展的附录，添加了指向REMB草案和RFC 5450的指针

---

### A.3.  Version -02 to -03 | A.3. 版本 -02 至 -03

#### 原文

   o  Added a section on how to process multiple streams in a single
      estimator using RTP timestamps to NTP time conversion.

   o  Stated in introduction that the draft is aimed at the RMCAT
      working group.

#### 中文

   o  添加了关于如何使用RTP时间戳到NTP时间转换在单个估计器中处理多个流的章节

   o  在引言中说明本草案针对RMCAT工作组

---

### A.4.  rtcweb-03 to rmcat-00 | A.4. rtcweb-03 至 rmcat-00

#### 原文

   Renamed draft to link the draft name to the RMCAT WG.

#### 中文

   重命名草案，将草案名称与RMCAT工作组关联。

---

### A.5.  rmcat -00 to -01 | A.5. rmcat -00 至 -01

#### 原文

   Spellcheck.  Otherwise no changes, this is a "keepalive" release.

#### 中文

   拼写检查。此外没有其他更改，这是一个"保持活跃"的发布。

---

### A.6.  rmcat -01 to -02 | A.6. rmcat -01 至 -02

#### 原文

   o  Added Luca De Cicco and Saverio Mascolo as authors.

   o  Extended the "Over-use detector" section with new technical
      details on how to dynamically tune the offset del_var_th for
      improved fairness properties.

   o  Added reference to a paper analyzing the behavior of the proposed
      algorithm.

#### 中文

   o  添加了Luca De Cicco和Saverio Mascolo作为作者

   o  扩展了"过度使用检测器"部分，新增了关于如何动态调整偏移量 del_var_th 以改善公平性的技术细节

   o  添加了对一篇分析所提出算法行为的论文的引用

---

### A.7.  rmcat -02 to -03 | A.7. rmcat -02 至 -03

#### 原文

   o  Swapped receiver-side/sender-side controller with delay-based/
      loss-based controller as there is no longer a requirement to run
      the delay-based controller on the receiver-side.

   o  Removed the discussion about multiple streams and transmission
      time offsets.

   o  Introduced a new section about "Feedback and extensions".

   o  Improvements to the threshold adaptation in the "Over-use
      detector" section.

   o  Swapped the previous MIMD rate control algorithm for a new AIMD
      rate control algorithm.

#### 中文

   o  将接收端/发送端控制器替换为基于延迟/基于丢包的控制器，因为不再要求基于延迟的控制器在接收端运行

   o  删除了关于多个流和传输时间偏移的讨论

   o  新增了关于"反馈与扩展"的章节

   o  改进了"过度使用检测器"部分中的阈值自适应

   o  将先前的MIMD速率控制算法替换为新的AIMD速率控制算法

---

### A.8.  ietf-rmcat -00 to ietf-rmcat -01 | A.8. ietf-rmcat -00 至 ietf-rmcat -01

#### 原文

   o  Arrival-time filter converted from a two dimensional Kalman filter
      to a scalar Kalman filter.

   o  The use of the TFRC equation was removed from the loss-based
      controller, as it turned out to have little to no effect in
      practice.

#### 中文

   o  将到达时间滤波器从二维卡尔曼滤波器转换为标量卡尔曼滤波器

   o  从基于丢包的控制器中移除了TFRC方程的使用，因为实践发现它几乎没有效果

---

### A.9.  ietf-rmcat -01 to ietf-rmcat -02 | A.9. ietf-rmcat -01 至 ietf-rmcat -02

#### 原文

   o  Added a section which better describes the pre-filtering
      algorithm.

#### 中文

   o  添加了一个更好地描述预过滤算法的章节

---

## Authors' Addresses | 作者地址

#### 原文

   Stefan Holmer
   Google
   Kungsbron 2
   Stockholm  11122
   Sweden

   Email: holmer@google.com


   Henrik Lundin
   Google
   Kungsbron 2
   Stockholm  11122
   Sweden

   Email: hlundin@google.com


   Gaetano Carlucci
   Politecnico di Bari
   Via Orabona, 4
   Bari  70125
   Italy

   Email: gaetano.carlucci@poliba.it


   Luca De Cicco
   Politecnico di Bari
   Via Orabona, 4
   Bari  70125
   Italy

   Email: l.decicco@poliba.it


   Saverio Mascolo
   Politecnico di Bari
   Via Orabona, 4
   Bari  70125
   Italy

   Email: mascolo@poliba.it

#### 中文

   Stefan Holmer
   Google
   Kungsbron 2
   斯德哥尔摩  11122
   瑞典

   电子邮件: holmer@google.com


   Henrik Lundin
   Google
   Kungsbron 2
   斯德哥尔摩  11122
   瑞典

   电子邮件: hlundin@google.com


   Gaetano Carlucci
   巴里理工大学
   Via Orabona, 4
   巴里  70125
   意大利

   电子邮件: gaetano.carlucci@poliba.it


   Luca De Cicco
   巴里理工大学
   Via Orabona, 4
   巴里  70125
   意大利

   电子邮件: l.decicco@poliba.it


   Saverio Mascolo
   巴里理工大学
   Via Orabona, 4
   巴里  70125
   意大利

   电子邮件: mascolo@poliba.it

---
