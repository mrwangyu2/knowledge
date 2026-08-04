# draft-ietf-ice-trickle-21 完整中英对照

> Trickle ICE: Incremental Provisioning of Candidates for the Interactive Connectivity Establishment (ICE) Protocol — ICE 候选的增量提供

---

Network Working Group                                            E. Ivov
Internet-Draft                                                 Atlassian
Intended status: Standards Track                             E. Rescorla
Expires: October 17, 2018                                     RTFM, Inc.
                                                               J. Uberti
                                                                  Google
                                                          P. Saint-Andre
                                                                 Mozilla
                                                          April 15, 2018

（网络工作组                                                E. Ivov
互联网草案                                                    Atlassian
预期状态：标准跟踪                                        E. Rescorla
过期时间：2018年10月17日                                   RTFM, Inc.
                                                               J. Uberti
                                                                  Google
                                                          P. Saint-Andre
                                                                 Mozilla
                                                          2018年4月15日）

---

Trickle ICE: Incremental Provisioning of Candidates for the Interactive
               Connectivity Establishment (ICE) Protocol
                       draft-ietf-ice-trickle-21

（Trickle ICE：为交互式连接建立（ICE）协议增量提供候选
                       draft-ietf-ice-trickle-21）

---

**Abstract（摘要）**

This document describes "Trickle ICE", an extension to the Interactive Connectivity Establishment (ICE) protocol that enables ICE agents to begin connectivity checks while they are still gathering candidates, by incrementally exchanging candidates over time instead of all at once. This method can considerably accelerate the process of establishing a communication session.

本文档描述了"Trickle ICE"，这是对交互式连接建立（ICE）协议的一个扩展，它使 ICE 代理能够在仍在收集候选的同时开始连通性检查，通过随时间增量式交换候选而不是一次性全部交换。该方法可以显著加速建立通信会话的过程。

---

**Status of This Memo（本文档状态）**

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

本互联网草案完全符合 BCP 78 和 BCP 79 的规定提交。

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at http://datatracker.ietf.org/drafts/current/.

互联网草案是互联网工程任务组（IETF）的工作文档。注意，其他组织也可能将工作文档作为互联网草案分发。当前互联网草案列表位于 http://datatracker.ietf.org/drafts/current/。

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

互联网草案是有效期最长为六个月的草案文档，可能随时被其他文档更新、替换或淘汰。将互联网草案用作参考资料或以"正在进行中的工作"以外的其他方式引用是不恰当的。

This Internet-Draft will expire on October 17, 2018.

本互联网草案将于2018年10月17日过期。

---

**Copyright Notice（版权声明）**

Copyright (c) 2018 IETF Trust and the persons identified as the document authors. All rights reserved.

版权所有 (c) 2018 IETF 信托基金以及被确定为文档作者的人员。保留所有权利。

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (http://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document. Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

本文档受 BCP 78 和 IETF 信托基金关于 IETF 文档的法律条款（http://trustee.ietf.org/license-info）（在本文档发布之日生效）约束。请仔细阅读这些文档，因为它们描述了您与本文档相关的权利和限制。从本文档中提取的代码组件必须包括信托法律条款第 4.e 节中描述的简化 BSD 许可证文本，并按简化 BSD 许可证中所述提供，无任何保证。

---

## Table of Contents（目录）

1.  Introduction（简介） . . . . . . . . . . . . . . . . . . . . . . . .   3
2.  Terminology（术语） . . . . . . . . . . . . . . . . . . . . . . . .   5
3.  Determining Support for Trickle ICE（确定对 Trickle ICE 的支持） . .   6
4.  Generating the Initial ICE Description（生成初始 ICE 描述） . . . .   7
5.  Handling the Initial ICE Description and Generating the Initial ICE Response（处理初始 ICE 描述并生成初始 ICE 响应） .   7
6.  Handling the Initial ICE Response（处理初始 ICE 响应） . . . . . .   8
7.  Forming Check Lists（形成检查列表） . . . . . . . . . . . . . . . .   8
8.  Performing Connectivity Checks（执行连通性检查） . . . . . . . . .   8
9.  Gathering and Conveying Newly Gathered Local Candidates（收集和传送新收集的本地候选） . .   9
10. Pairing Newly Gathered Local Candidates（配对最新收集的本地候选） .  10
11. Receiving Trickled Candidates（接收增量发送的候选） . . . . . . .  11
12. Inserting Trickled Candidate Pairs into a Check List（将增量发送的候选对插入检查列表） . . . .  12
13. Generating an End-of-Candidates Indication（生成候选结束指示）  . .  16
14. Receiving an End-of-Candidates Indication（接收候选结束指示） . . .  17
15. Subsequent Exchanges and ICE Restarts（后续交换和 ICE 重启） . . .  18
16. Half Trickle（半增量式）  . . . . . . . . . . . . . . . . . . . .  18
17. Preserving Candidate Order while Trickling（在增量发送中保持候选顺序） .  19
18. Requirements for Using Protocols（使用协议的要求）  . . . . . . . .  20
19. IANA Considerations（IANA 考虑） . . . . . . . . . . . . . . . . .  21
20. Security Considerations（安全考虑） . . . . . . . . . . . . . . . .  21
21. Acknowledgements（致谢）  . . . . . . . . . . . . . . . . . . . . .  21
22. References（参考文献）  . . . . . . . . . . . . . . . . . . . . . .  22
    22.1.  Normative References（规范性参考文献） . . . . . . . . . . .  22
    22.2.  Informative References（信息性参考文献） . . . . . . . . . .  22
Appendix A.  Interaction with Regular ICE（与常规 ICE 的交互） . . . .  23
Appendix B.  Interaction with ICE Lite（与 ICE Lite 的交互）  . . . .  25
Appendix C.  Changes from Earlier Versions（与早期版本的变化） . . . .  26

---

## 1.  Introduction（简介）

The Interactive Connectivity Establishment (ICE) protocol [rfc5245bis] describes how an ICE agent gathers candidates, exchanges candidates with a peer ICE agent, and creates candidate pairs. Once the pairs have been gathered, the ICE agent will perform connectivity checks, and eventually nominate and select pairs that will be used for sending and receiving data within a communication session.

交互式连接建立（ICE）协议 [rfc5245bis] 描述了 ICE 代理如何收集候选、与对等 ICE 代理交换候选以及创建候选对。一旦候选对收集完成，ICE 代理将执行连通性检查，并最终提名和选择将用于在通信会话中发送和接收数据的候选对。

Following the procedures in [rfc5245bis] can lead to somewhat lengthy establishment times for communication sessions, because candidate gathering often involves querying STUN servers [RFC5389] and allocating relayed candidates using TURN servers [RFC5766]. Although many ICE procedures can be completed in parallel, the pacing requirements from [rfc5245bis] still need to be followed.

遵循 [rfc5245bis] 中的程序可能导致通信会话的建立时间较长，因为候选收集通常涉及查询 STUN 服务器 [RFC5389] 和使用 TURN 服务器 [RFC5766] 分配中继候选。尽管许多 ICE 程序可以并行完成，但仍需要遵循 [rfc5245bis] 中的 pacing 要求。

This document defines "Trickle ICE", a supplementary mode of ICE operation in which candidates can be exchanged incrementally as soon as they become available (and simultaneously with the gathering of other candidates). Connectivity checks can also start as soon as candidate pairs have been created. Because Trickle ICE enables candidate gathering and connectivity checks to be done in parallel, the method can considerably accelerate the process of establishing a communication session.

本文档定义了"Trickle ICE"，这是 ICE 操作的一个补充模式，在此模式下候选可以在变得可用时立即增量交换（同时与其他候选的收集同步进行）。连通性检查也可以在候选对创建后立即开始。由于 Trickle ICE 使候选收集和连通性检查能够并行进行，该方法可以显著加速建立通信会话的过程。

This document also defines how to discover support for Trickle ICE, how the procedures in [rfc5245bis] are modified or supplemented when using Trickle ICE, and how a Trickle ICE agent can interoperate with an ICE agent compliant to [rfc5245bis].

本文档还定义了如何发现对 Trickle ICE 的支持、在使用 Trickle ICE 时如何修改或补充 [rfc5245bis] 中的程序，以及 Trickle ICE 代理如何与符合 [rfc5245bis] 的 ICE 代理互操作。

This document does not define any protocol-specific usage of Trickle ICE. Instead, protocol-specific details for Trickle ICE are defined in separate usage documents. Examples of such documents are [I-D.ietf-mmusic-trickle-ice-sip] (which defines usage with the Session Initiation Protocol (SIP) [RFC3261] and the Session Description Protocol [RFC3261]) and [XEP-0176] (which defines usage with XMPP [RFC6120]). However, some of the examples in this document use SDP and the offer/answer model [RFC3264] to explain the underlying concepts.

本文档不定义 Trickle ICE 的任何协议特定用法。相反，Trickle ICE 的协议特定细节在单独的使用文档中定义。此类文档的示例包括 [I-D.ietf-mmusic-trickle-ice-sip]（定义了与会话初始协议（SIP）[RFC3261] 和会话描述协议（SDP）[RFC4566] 的使用）和 [XEP-0176]（定义了与 XMPP [RFC6120] 的使用）。但是，本文档中的一些示例使用 SDP 和 offer/answer 模型 [RFC3264] 来解释基本概念。

The following diagram illustrates a successful Trickle ICE exchange with a using protocol that follows the offer/answer model:

下图说明了使用遵循 offer/answer 模型的协议进行成功的 Trickle ICE 交换：

           Alice                                            Bob
             |                     Offer                     |
             |---------------------------------------------->|
             |            Additional Candidates              |
             |---------------------------------------------->|
             |                     Answer                    |
             |<----------------------------------------------|
             |            Additional Candidates              |
             |<----------------------------------------------|
             | Additional Candidates and Connectivity Checks |
             |<--------------------------------------------->|
             |<========== CONNECTION ESTABLISHED ===========>|



                              Figure 1: Flow
                              （图 1：流程）

The main body of this document is structured to describe the behavior of Trickle ICE agents in roughly the order of operations and interactions during an ICE session:

本文档的主体结构大致按照 ICE 会话期间的操作和交互顺序来描述 Trickle ICE 代理的行为：

1.  Determining support for trickle ICE
    （确定对 Trickle ICE 的支持）

2.  Generating the initial ICE description
    （生成初始 ICE 描述）

3.  Handling the initial ICE description and generating the initial ICE response
    （处理初始 ICE 描述并生成初始 ICE 响应）

4.  Handling the initial ICE response
    （处理初始 ICE 响应）

5.  Forming check lists, pruning candidates, performing connectivity checks, etc.
    （形成检查列表、修剪候选、执行连通性检查等）

6.  Gathering and conveying candidates after the initial ICE description and response
    （在初始 ICE 描述和响应之后收集和传送候选）

7.  Handling inbound trickled candidates
    （处理入站增量发送的候选）

8.  Generating and handling the end-of-candidates indication
    （生成和处理候选结束指示）

9.  Handling ICE restarts
    （处理 ICE 重启）

There is quite a bit of operational experience with the technique behind Trickle ICE, going back as far as 2005 (when the XMPP Jingle extension defined a "dribble mode" as specified in [XEP-0176]); this document incorporates feedback from those who have implemented and deployed the technique over the years.

Trickle ICE 背后的技术有相当多的运营经验，最早可追溯到 2005 年（当时 XMPP Jingle 扩展在 [XEP-0176] 中定义了"dribble 模式"）；本文档吸收了过去多年来实现和部署该技术的人员的反馈。

---

## 2.  Terminology（术语）

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119].

本文档中的关键词 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"MAY" 和 "OPTIONAL" 将按照 [RFC2119] 中所述进行解释。

This specification makes use of all terminology defined for Interactive Connectivity Establishment in [rfc5245bis]. In addition, it defines the following terms:

本规范使用 [rfc5245bis] 中为交互式连接建立定义的所有术语。此外，它还定义了以下术语：

Full Trickle: The typical mode of operation for Trickle ICE agents, in which the initial ICE description can include any number of candidates (even zero candidates) and does not need to include a full generation of candidates as in half trickle.

完全增量式（Full Trickle）：Trickle ICE 代理的典型操作模式，在此模式下，初始 ICE 描述可以包含任意数量的候选（甚至零个候选），不需要像半增量式那样包含完整的一代候选。

Generation: All of the candidates conveyed within an ICE session.

代（Generation）：在 ICE 会话中传送的所有候选。

Half Trickle: A Trickle ICE mode of operation in which the initiator gathers a full generation of candidates strictly before creating and conveying the initial ICE description. Once conveyed, this candidate information can be processed by regular ICE agents, which do not require support for Trickle ICE. It also allows Trickle ICE capable responders to still gather candidates and perform connectivity checks in a non-blocking way, thus providing roughly "half" the advantages of Trickle ICE. The half trickle mechanism is mostly meant for use when the responder's support for Trickle ICE cannot be confirmed prior to conveying the initial ICE description.

半增量式（Half Trickle）：一种 Trickle ICE 操作模式，在此模式下，发起方在创建和传送初始 ICE 描述之前严格收集完整的一代候选。一旦传送，此候选信息可由不需要 Trickle ICE 支持的常规 ICE 代理处理。它还允许支持 Trickle ICE 的应答方仍然以非阻塞方式收集候选并执行连通性检查，从而提供大约"一半"的 Trickle ICE 优势。半增量机制主要用于在传送初始 ICE 描述之前无法确认应答方对 Trickle ICE 的支持的情况。

ICE Description: Any attributes related to the ICE session (not candidates) required to configure an ICE agent. These include but are not limited to the username fragment, password, and other attributes.

ICE 描述（ICE Description）：与 ICE 会话相关的任何属性（不是候选），用于配置 ICE 代理。这些包括但不限于 username fragment、password 和其他属性。

Trickled Candidates: Candidates that a Trickle ICE agent conveys after conveying the initial ICE description or responding to the initial ICE description, but within the same ICE session. Trickled candidates can be conveyed in parallel with candidate gathering and connectivity checks.

增量候选（Trickled Candidates）：Trickle ICE 代理在传送初始 ICE 描述或响应初始 ICE 描述之后但在同一 ICE 会话内传送的候选。增量候选可以与候选收集和连通性检查并行传送。

Trickling: The act of incrementally conveying trickled candidates.

增量发送（Trickling）：增量传送增量候选的行为。

Empty Check List: A check list that initially does not contain any candidate pairs because they will be incrementally added as they are trickled. (This scenario does not arise with a regular ICE agent, because all candidate pairs are known when the agent creates the check list set).

空检查列表（Empty Check List）：最初不包含任何候选对的检查列表，因为它们将在增量发送时被增量添加。（在常规 ICE 代理中不会出现此场景，因为在代理创建检查列表集时所有候选对都是已知的）。

---

## 3.  Determining Support for Trickle ICE（确定对 Trickle ICE 的支持）

To fully support Trickle ICE, using protocols SHOULD incorporate one of the following mechanisms so that implementations can determine whether Trickle ICE is supported:

为了完全支持 Trickle ICE，使用协议应该（SHOULD）包含以下机制之一，以便实现可以确定是否支持 Trickle ICE：

1.  Provide a capabilities discovery method so that agents can verify support of Trickle ICE prior to initiating a session (XMPP's Service Discovery [XEP-0030] is one such mechanism).
    （提供能力发现方法，以便代理可以在启动会话之前验证对 Trickle ICE 的支持（XMPP 的服务发现 [XEP-0030] 就是这样的机制）。）

2.  Make support for Trickle ICE mandatory so that user agents can assume support.
    （将对 Trickle ICE 的支持设为强制性的，以便用户代理可以假定支持。）

If a using protocol does not provide a method of determining ahead of time whether Trickle ICE is supported, agents can make use of the half trickle procedure described in Section 16.

如果使用协议不提供提前确定 Trickle ICE 是否得到支持的方法，代理可以使用第 16 节中描述的半增量式过程。

Prior to conveying the initial ICE description, agents that implement using protocols that support capabilities discovery can attempt to verify whether or not the remote party supports Trickle ICE. If an agent determines that the remote party does not support Trickle ICE, it MUST fall back to using regular ICE or abandon the entire session.

在传送初始 ICE 描述之前，实现支持能力发现的使用协议的代理可以尝试验证远程方是否支持 Trickle ICE。如果代理确定远程方不支持 Trickle ICE，它必须（MUST）回退到使用常规 ICE 或放弃整个会话。

Even if a using protocol does not include a capabilities discovery method, a user agent can provide an indication within the ICE description that it supports Trickle ICE by communicating an ICE option of 'trickle'. This token MUST be provided either at the session level or, if at the data stream level, for every data stream (an agent MUST NOT specify Trickle ICE support for some data streams but not others). Note: The encoding of the 'trickle' ICE option, and the message(s) used to carry it to the peer, are protocol specific; for instance, the encoding for the Session Description Protocol (SDP) [RFC4566] is defined in [I-D.ietf-mmusic-trickle-ice-sip].

即使使用协议不包含能力发现方法，用户代理可以通过传达 'trickle' 的 ICE 选项在 ICE 描述中提供对 Trickle ICE 支持的指示。此令牌必须（MUST）在会话级别提供，或者如果在数据流级别，则为每个数据流提供（代理不得（MUST NOT）为某些数据流指定 Trickle ICE 支持而不为其他数据流指定）。注意：'trickle' ICE 选项的编码以及用于将其传递给对端的消息是协议特定的；例如，会话描述协议（SDP）[RFC4566] 的编码在 [I-D.ietf-mmusic-trickle-ice-sip] 中定义。

Dedicated discovery semantics and half trickle are needed only prior to initiation of an ICE session. After an ICE session is established and Trickle ICE support is confirmed for both parties, either agent can use full trickle for subsequent exchanges (see also Section 15).

专用的发现语义和半增量式仅在 ICE 会话启动之前需要。在 ICE 会话建立并且双方确认 Trickle ICE 支持后，任何一方代理都可以在后续交换中使用完全增量式（另见第 15 节）。

---

## 4.  Generating the Initial ICE Description（生成初始 ICE 描述）

An ICE agent can start gathering candidates as soon as it has an indication that communication is imminent (e.g., a user interface cue or an explicit request to initiate a communication session). Unlike in regular ICE, in Trickle ICE implementations do not need to gather candidates in a blocking manner. Therefore, unless half trickle is being used, the user experience is improved if the initiating agent generates and transmits its initial ICE description as early as possible (thus enabling the remote party to start gathering and trickling candidates).

ICE 代理可以在有迹象表明通信即将发生（例如，用户界面提示或启动通信会话的显式请求）时立即开始收集候选。与常规 ICE 不同，在 Trickle ICE 中，实现不需要以阻塞方式收集候选。因此，除非使用半增量式，否则如果发起代理尽快生成并传送其初始 ICE 描述（从而使远程方能开始收集和增量发送候选），用户体验将会得到改善。

An initiator MAY include any mix of candidates when conveying the initial ICE description. This includes the possibility of conveying all the candidates the initiator plans to use (as in half trickle), conveying only a publicly-reachable IP address (e.g., a candidate at a data relay that is known to not be behind a firewall), or conveying no candidates at all (in which case the initiator can obtain the responder's initial candidate list sooner and the responder can begin candidate gathering more quickly).

发起方在传送初始 ICE 描述时可以（MAY）包含任意组合的候选。这包括以下可能性：传送发起方计划使用的所有候选（如在半增量式中）、仅传送一个可公开访问的 IP 地址（例如，已知不在防火墙后面的数据中继处的候选），或者根本不传送任何候选（在这种情况下，发起方可以更快地获取应答方的初始候选列表，应答方可以更快地开始候选收集）。

For candidates included in the initial ICE description, the methods for calculating priorities and foundations, determining redundancy of candidates, and the like work just as in regular ICE [rfc5245bis].

对于初始 ICE 描述中包含的候选，计算优先级和 foundation、确定候选冗余等的方法与常规 ICE [rfc5245bis] 中的工作方式相同。

---

## 5.  Handling the Initial ICE Description and Generating the Initial ICE Response（处理初始 ICE 描述并生成初始 ICE 响应）

When a responder receives the initial ICE description, it will first check if the ICE description or initiator indicates support for Trickle ICE as explained in Section 3. If not, the responder MUST process the initial ICE description according to regular ICE procedures [rfc5245bis] (or, if no ICE support is detected at all, according to relevant processing rules for the using protocol, such as offer/answer processing rules [RFC3264]). However, if support for Trickle ICE is confirmed, a responder will automatically assume support for regular ICE as well.

当应答方收到初始 ICE 描述时，它将首先按照第 3 节中的说明检查 ICE 描述或发起方是否指示支持 Trickle ICE。如果不支持，应答方必须（MUST）按照常规 ICE 程序 [rfc5245bis] 处理初始 ICE 描述（或者，如果根本没有检测到 ICE 支持，则按照使用协议的相关处理规则，如 offer/answer 处理规则 [RFC3264]）。但是，如果确认支持 Trickle ICE，应答方将自动假定也支持常规 ICE。

If the initial ICE description indicates support for Trickle ICE, the responder will determine its role and start gathering and prioritizing candidates; while doing so, it will also respond by conveying an initial ICE response, so that both the initiator and the responder can form check lists and begin connectivity checks.

如果初始 ICE 描述指示支持 Trickle ICE，应答方将确定其角色并开始收集和优先排序候选；同时，它还将通过传送初始 ICE 响应来做出回应，以便发起方和应答方都可以形成检查列表并开始连通性检查。

A responder can respond to the initial ICE description at any point while gathering candidates. The initial ICE response MAY contain any set of candidates, including all candidates or no candidates. (The benefit of including no candidates is to convey the initial ICE response as quickly as possible, so that both parties can consider the ICE session to be under active negotiation as soon as possible.)

应答方可以在收集候选期间的任何时刻响应初始 ICE 描述。初始 ICE 响应可以（MAY）包含任意候选集，包括所有候选或不包含任何候选。（不包含任何候选的好处是尽可能快地传送初始 ICE 响应，以便双方可以尽快认为 ICE 会话处于主动协商中。）

As noted in Section 3, in using protocols that use SDP the initial ICE response can indicate support for Trickle ICE by including a token of "trickle" in the ice-options attribute.

如第 3 节所述，在使用 SDP 的使用协议中，初始 ICE 响应可以通过在 ice-options 属性中包含 "trickle" 令牌来指示支持 Trickle ICE。

---

## 6.  Handling the Initial ICE Response（处理初始 ICE 响应）

When processing the initial ICE response, the initiator follows regular ICE procedures to determine its role, after which it forms check lists (Section 7) and performs connectivity checks (Section 8).

在处理初始 ICE 响应时，发起方遵循常规 ICE 程序来确定其角色，然后形成检查列表（第 7 节）并执行连通性检查（第 8 节）。

---

## 7.  Forming Check Lists（形成检查列表）

According to regular ICE procedures [rfc5245bis], in order for candidate pairing to be possible and for redundant candidates to be pruned, the candidates would need to be provided in the initial ICE description and initial ICE response. By contrast, under Trickle ICE check lists can be empty until candidates are conveyed or received. Therefore a Trickle ICE agent handles check list formation and candidate pairing in a slightly different way than a regular ICE agent: the agent still forms the check lists, but it populates a given check list only after it actually has candidate pairs for that check list. Every check list is initially placed in the Running state, even if the check list is empty (this is consistent with Section 6.1.2.1 of [rfc5245bis]).

根据常规 ICE 程序 [rfc5245bis]，为了使候选配对成为可能并修剪冗余候选，候选需要在初始 ICE 描述和初始 ICE 响应中提供。相比之下，在 Trickle ICE 下，检查列表可以在候选被传送或接收之前为空。因此，Trickle ICE 代理以与常规 ICE 代理略有不同的方式处理检查列表形成和候选配对：代理仍然形成检查列表，但仅在它实际上拥有该检查列表的候选对之后才填充给定的检查列表。每个检查列表最初都放置在 Running 状态，即使检查列表为空（这与 [rfc5245bis] 第 6.1.2.1 节一致）。

---

## 8.  Performing Connectivity Checks（执行连通性检查）

As specified in [rfc5245bis], whenever timer Ta fires, only check lists in the Running state will be picked when scheduling connectivity checks for candidate pairs. Therefore, a Trickle ICE agent MUST keep each check list in the Running state as long as it expects candidate pairs to be incrementally added to the check list. After that, the check list state is set according to the procedures in [rfc5245bis].

如 [rfc5245bis] 中规定的，每当定时器 Ta 触发时，在为候选对调度连通性检查时只会选择 Running 状态的检查列表。因此，只要 Trickle ICE 代理期望候选对被增量添加到检查列表中，它必须（MUST）将每个检查列表保持在 Running 状态。之后，检查列表状态根据 [rfc5245bis] 中的程序设置。

Whenever timer Ta fires and an empty check list is picked, no action is performed for the list. Without waiting for timer Ta to expire again, the agent selects the next check list in the Running state, in accordance with Section 6.1.4.2 of [rfc5245bis].

每当定时器 Ta 触发并选择了一个空检查列表时，不对该列表执行任何操作。无需等待定时器 Ta 再次到期，代理根据 [rfc5245bis] 第 6.1.4.2 节选择下一个 Running 状态的检查列表。

Section 7.2.5.3.3 of [rfc5245bis] requires that agents update check lists and timer states upon completing a connectivity check transaction. During such an update, regular ICE agents would set the state of a check list to Failed if both of the following two conditions are satisfied:

[rfc5245bis] 第 7.2.5.3.3 节要求代理在完成连通性检查事务后更新检查列表和定时器状态。在此更新过程中，如果满足以下两个条件，常规 ICE 代理会将检查列表的状态设置为 Failed：

o  all of the pairs in the check list are either in the Failed state or Succeeded state; and
  （检查列表中的所有候选对都处于 Failed 状态或 Succeeded 状态；以及）

o  there is not a pair in the valid list for each component of the data stream.
  （数据流的每个组件在有效列表中都没有对应的候选对。）

With Trickle ICE, the above situation would often occur when candidate gathering and trickling are still in progress, even though it is quite possible that future checks will succeed. For this reason, Trickle ICE agents add the following conditions to the above list:

在 Trickle ICE 中，上述情况通常发生在候选收集和增量发送仍在进行中时，即使未来的检查很有可能成功。出于这个原因，Trickle ICE 代理在上述列表中添加了以下条件：

o  all candidate gathering has completed and the agent is not expecting to discover any new local candidates; and
  （所有候选收集已完成，并且代理不期望发现任何新的本地候选；以及）

o  the remote agent has conveyed an end-of-candidates indication for that check list as described in Section 13.
  （远程代理已按照第 13 节所述为该检查列表传送了候选结束指示。）

---

## 9.  Gathering and Conveying Newly Gathered Local Candidates（收集和传送新收集的本地候选）

After Trickle ICE agents have conveyed initial ICE descriptions and initial ICE responses, they will most likely continue gathering new local candidates as STUN, TURN, and other non-host candidate gathering mechanisms begin to yield results. Whenever an agent discovers such a new candidate it will compute its priority, type, foundation, and component ID according to regular ICE procedures.

在 Trickle ICE 代理传送了初始 ICE 描述和初始 ICE 响应之后，它们很可能在 STUN、TURN 和其他非主机候选收集机制开始产生结果时继续收集新的本地候选。每当代理发现这样的新候选时，它将根据常规 ICE 程序计算其优先级、类型、foundation 和组件 ID。

The new candidate is then checked for redundancy against the existing list of local candidates. If its transport address and base match those of an existing candidate, it will be considered redundant and will be ignored. This would often happen for server reflexive candidates that match the host addresses they were obtained from (e.g., when the latter are public IPv4 addresses). Contrary to regular ICE, Trickle ICE agents will consider the new candidate redundant regardless of its priority.

然后将新候选与现有本地候选列表进行冗余检查。如果其传输地址和 base 与现有候选的地址和 base 匹配，则将被视为冗余并被忽略。这通常发生在服务器反射候选与其获取自的主机地址匹配的情况下（例如，当后者是公共 IPv4 地址时）。与常规 ICE 相反，Trickle ICE 代理将不考虑优先级而将新候选视为冗余。

Next the agent "trickles" the newly discovered candidate(s) to the remote agent. The actual delivery of the new candidates is handled by a using protocol such as SIP or XMPP. Trickle ICE imposes no restrictions on the way this is done (e.g., some using protocols might choose not to trickle updates for server reflexive candidates and instead rely on the discovery of peer reflexive ones).

接下来，代理将新发现的候选"增量发送"到远程代理。新候选的实际传递由使用协议（如 SIP 或 XMPP）处理。Trickle ICE 对此方式不施加任何限制（例如，一些使用协议可能选择不增量发送服务器反射候选的更新，而是依赖于对端反射候选的发现）。

When candidates are trickled, the using protocol MUST deliver each candidate (and any end-of-candidates indication as described in Section 13) to the receiving Trickle ICE implementation exactly once and in the same order it was conveyed. If the using protocol provides any candidate retransmissions, they need to be hidden from the ICE implementation.

当候选被增量发送时，使用协议必须（MUST）将每个候选（以及第 13 节中描述的任何候选结束指示）恰好一次地、以相同的传送顺序交付给接收 Trickle ICE 实现。如果使用协议提供任何候选重传，它们需要对 ICE 实现隐藏。

Also, candidate trickling needs to be correlated to a specific ICE session, so that if there is an ICE restart, any delayed updates for a previous session can be recognized as such and ignored by the receiving party. For example, using protocols that signal candidates via SDP might include a Username Fragment value in the corresponding a=candidate line, such as:

此外，候选增量发送需要与特定的 ICE 会话相关联，以便如果有 ICE 重启，之前会话的任何延迟更新可以被识别并由此被接收方忽略。例如，通过 SDP 发送候选信号的使用协议可能在相应的 a=candidate 行中包含 Username Fragment 值，例如：

     a=candidate:1 1 UDP 2130706431 2001:db8::1 5000 typ host ufrag 8hhY

Or, as another example, WebRTC implementations might include a Username Fragment in the JavaScript objects that represent candidates.

或者，作为另一个示例，WebRTC 实现可能在表示候选的 JavaScript 对象中包含 Username Fragment。

Note: The using protocol needs to provide a mechanism for both parties to indicate and agree on the ICE session in force (as identified by the Username Fragment and Password combination) so that they have a consistent view of which candidates are to be paired. This is especially important in the case of ICE restarts (see Section 15).

注意：使用协议需要提供一种机制，使双方能够指示和商定正在生效的 ICE 会话（由 Username Fragment 和 Password 组合标识），以便它们对要配对的候选有一致的视图。这在 ICE 重启的情况下尤其重要（见第 15 节）。

Note: A using protocol might prefer not to trickle server reflexive candidates to entities that are known to be publicly accessible and where sending a direct STUN binding request is likely to reach the destination faster than the trickle update that travels through the signaling path.

注意：使用协议可能更倾向于不将服务器反射候选增量发送到已知可公开访问的实体，因为在这些实体中发送直接的 STUN 绑定请求可能比通过信令路径传递的增量更新更快到达目的地。

---

## 10.  Pairing Newly Gathered Local Candidates（配对最新收集的本地候选）

As a Trickle ICE agent gathers local candidates, it needs to form candidate pairs; this works as described in the ICE specification [rfc5245bis], with the following provisos:

当 Trickle ICE 代理收集本地候选时，它需要形成候选对；这按照 ICE 规范 [rfc5245bis] 中所述工作，但有以下附带条件：

1.  A Trickle ICE agent MUST NOT pair a local candidate until it has been trickled to the remote party.
    （Trickle ICE 代理不得（MUST NOT）在本地候选被增量发送到远程方之前将其配对。）

2.  Once the agent has conveyed the local candidate to the remote party, the agent checks if any remote candidates are currently known for this same stream and component. If not, the agent merely adds the new candidate to the list of local candidates (without pairing it).
    （一旦代理将本地候选传送给了远程方，代理将检查当前是否已知此相同流和组件的任何远程候选。如果没有，代理仅将新候选添加到本地候选列表中（不进行配对）。）

3.  Otherwise, if the agent has already learned of one or more remote candidates for this stream and component, it attempts to pair the new local candidate as described in the ICE specification [rfc5245bis].
    （否则，如果代理已经获悉此流和组件的一个或多个远程候选，它将尝试按照 ICE 规范 [rfc5245bis] 中所述配对新的本地候选。）

4.  If a newly formed pair has a local candidate whose type is server reflexive, the agent MUST replace the local candidate with its base before completing the relevant redundancy tests.
    （如果新形成的候选对具有类型为服务器反射的本地候选，代理必须（MUST）在完成相关冗余测试之前用其 base 替换本地候选。）

5.  The agent prunes redundant pairs by following the rules in Section 6.1.2.4 of [rfc5245bis], but checks existing pairs only if they have a state of Waiting or Frozen; this avoids removal of pairs for which connectivity checks are in flight (a state of In-Progress) or for which connectivity checks have already yielded a definitive result (a state of Succeeded or Failed).
    （代理通过遵循 [rfc5245bis] 第 6.1.2.4 节中的规则来修剪冗余候选对，但仅检查状态为 Waiting 或 Frozen 的现有候选对；这避免了移除正在进行连通性检查（In-Progress 状态）或连通性检查已经产生明确结果（Succeeded 或 Failed 状态）的候选对。）

6.  If after the relevant redundancy tests the check list where the pair is to be added already contains the maximum number of candidate pairs (100 by default as per [rfc5245bis]), the agent SHOULD discard any pairs in the Failed state to make room for the new pair. If there are no such pairs, the agent SHOULD discard a pair with a lower priority than the new pair in order to make room for the new pair, until the number of pairs is equal to the maximum number of pairs. This processing is consistent with Section 6.1.2.5 of [rfc5245bis].
    （如果在相关冗余测试之后，要添加候选对的检查列表已经包含最大数量的候选对（根据 [rfc5245bis] 默认为 100），代理应该（SHOULD）丢弃任何处于 Failed 状态的候选对以为新候选对腾出空间。如果没有这样的候选对，代理应该（SHOULD）丢弃优先级低于新候选对的候选对以为新候选对腾出空间，直到候选对数量等于最大数量。此处理与 [rfc5245bis] 第 6.1.2.5 节一致。）

---

## 11.  Receiving Trickled Candidates（接收增量发送的候选）

At any time during an ICE session, a Trickle ICE agent might receive new candidates from the remote agent, from which it will attempt to form a candidate pair; this works as described in the ICE specification [rfc5245bis], with the following provisos:

在 ICE 会话期间的任何时候，Trickle ICE 代理可能会从远程代理接收新候选，并将尝试从中形成候选对；这按照 ICE 规范 [rfc5245bis] 中所述工作，但有以下附带条件：

1.  The agent checks if any local candidates are currently known for this same stream and component. If not, the agent merely adds the new candidate to the list of remote candidates (without pairing it).
    （代理检查当前是否已知此相同流和组件的任何本地候选。如果没有，代理仅将新候选添加到远程候选列表中（不进行配对）。）

2.  Otherwise, if the agent has already gathered one or more local candidates for this stream and component, it attempts to pair the new remote candidate as described in the ICE specification [rfc5245bis].
    （否则，如果代理已经为此流和组件收集了一个或多个本地候选，它将尝试按照 ICE 规范 [rfc5245bis] 中所述配对新的远程候选。）

3.  If a newly formed pair has a local candidate whose type is server reflexive, the agent MUST replace the local candidate with its base before completing the redundancy check in the next step.
    （如果新形成的候选对具有类型为服务器反射的本地候选，代理必须（MUST）在执行下一步的冗余检查之前用其 base 替换本地候选。）

4.  The agent prunes redundant pairs as described below, but checks existing pairs only if they have a state of Waiting or Frozen; this avoids removal of pairs for which connectivity checks are in flight (a state of In-Progress) or for which connectivity checks have already yielded a definitive result (a state of Succeeded or Failed).
    （代理按照以下描述修剪冗余候选对，但仅检查状态为 Waiting 或 Frozen 的现有候选对；这避免了移除正在进行连通性检查（In-Progress 状态）或连通性检查已经产生明确结果（Succeeded 或 Failed 状态）的候选对。）

    A.  If the agent finds a redundancy between two pairs and one of those pairs contains a newly received remote candidate whose type is peer reflexive, the agent SHOULD discard the pair containing that candidate, set the priority of the existing pair to the priority of the discarded pair, and re-sort the check list. (This policy helps to eliminate problems with remote peer reflexive candidates for which a STUN binding request is received before signaling of the candidate is trickled to the receiving agent, such as a different view of pair priorities between the local agent and the remote agent, since the same candidate could be perceived as peer reflexive by one agent and as server reflexive by the other agent.)
    （A. 如果代理发现两个候选对之间存在冗余，并且其中一个候选对包含一个新接收的远程候选，其类型为对端反射，代理应该（SHOULD）丢弃包含该候选的候选对，将现有候选对的优先级设置为被丢弃候选对的优先级，并重新排序检查列表。此策略有助于消除远程对端反射候选的问题，对于这些候选，STUN 绑定请求在候选信令被增量发送到接收代理之前就被接收到，例如本地代理和远程代理之间对候选对优先级的视图不同，因为同一个候选可能被一个代理视为对端反射，被另一个代理视为服务器反射。）

    B.  The agent then applies the rules defined in Section 6.1.2.4 of [rfc5245bis].
    （B. 然后代理应用 [rfc5245bis] 第 6.1.2.4 节中定义的规则。）

5.  If after the relevant redundancy tests the check list where the pair is to be added already contains the maximum number of candidate pairs (100 by default as per [rfc5245bis]), the agent SHOULD discard any pairs in the Failed state to make room for the new pair. If there are no such pairs, the agent SHOULD discard a pair with a lower priority than the new pair in order to make room for the new pair, until the number of pairs is equal to the maximum number of pairs. This processing is consistent with Section 6.1.2.5 of [rfc5245bis].
    （如果在相关冗余测试之后，要添加候选对的检查列表已经包含最大数量的候选对（根据 [rfc5245bis] 默认为 100），代理应该（SHOULD）丢弃任何处于 Failed 状态的候选对以为新候选对腾出空间。如果没有这样的候选对，代理应该（SHOULD）丢弃优先级低于新候选对的候选对以为新候选对腾出空间，直到候选对数量等于最大数量。此处理与 [rfc5245bis] 第 6.1.2.5 节一致。）

---

## 12.  Inserting Trickled Candidate Pairs into a Check List（将增量发送的候选对插入检查列表）

After a local agent has trickled a candidate and formed a candidate pair from that local candidate (Section 9), or after a remote agent has received a trickled candidate and formed a candidate pair from that remote candidate (Section 11), a Trickle ICE agent adds the new candidate pair to a check list as defined in this section.

在本地代理增量发送了一个候选并从该本地候选形成了候选对（第 9 节）之后，或者在远程代理接收了一个增量候选并从该远程候选形成了候选对（第 11 节）之后，Trickle ICE 代理将根据本节定义将新候选对添加到检查列表中。

As an aid to understanding the procedures defined in this section, consider the following tabular representation of all check lists in an agent (note that initially for one of the foundations, i.e., f5, there are no candidate pairs):

为了帮助理解本节中定义的过程，请考虑以下代理中所有检查列表的表格表示（注意，最初对于其中一个 foundation，即 f5，没有候选对）：


   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  F   |  F   |  F   |      |      |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  F   |  F   |  F   |  F   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+


                   Figure 2: Example of Check List State
                   （图 2：检查列表状态示例）

Each row in the table represents a component for a given data stream (e.g., s1 and s2 might be the RTP and RTCP components for audio) and thus a single check list in the check list set. Each column represents one foundation. Each cell represents one candidate pair. In the tables shown in this section, "F" stands for "frozen", "W" stands for "waiting", and "S" stands for "succeeded"; in addition, "^^" is used to notate newly-added candidate pairs.

表格中的每一行表示给定数据流的一个组件（例如，s1 和 s2 可能是音频的 RTP 和 RTCP 组件），因此是检查列表集中的一个单独检查列表。每一列表示一个 foundation。每个单元格表示一个候选对。在本节所示的表格中，"F" 代表 "frozen"，"W" 代表 "waiting"，"S" 代表 "succeeded"；此外，"^^" 用于标记新添加的候选对。

When an agent commences ICE processing, in accordance with Section 6.1.2.6 of [rfc5245bis], for each foundation it will unfreeze the pair with the lowest component ID and, if the component IDs are equal, with the highest priority (this is the topmost candidate pair in every column). This initial state is shown in the following table.

当代理开始 ICE 处理时，根据 [rfc5245bis] 第 6.1.2.6 节，对于每个 foundation，它将解冻具有最低组件 ID 的候选对，如果组件 ID 相等则解冻具有最高优先级的候选对（这是每列中最顶部的候选对）。此初始状态显示在下表中。


   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  W   |  W   |  W   |      |      |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  F   |  F   |  F   |  W   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  F   |      |      |      |      |
   +-----------------+------+------+------+------+------+


                    Figure 3: Initial Check List State
                    （图 3：初始检查列表状态）

Then, as the checks proceed (see Section 7.2.5.4 of [rfc5245bis]), for each pair that enters the Succeeded state (denoted here by "S"), the agent will unfreeze all pairs for all data streams with the same foundation (e.g., if the pair in column 1, row 1 succeeds then the agent will unfreeze the pair in column 1, rows 2, 3, and 4).

然后，随着检查的进行（参见 [rfc5245bis] 第 7.2.5.4 节），对于每个进入 Succeeded 状态（此处用 "S" 表示）的候选对，代理将解冻具有相同 foundation 的所有数据流的所有候选对（例如，如果第 1 列第 1 行的候选对成功，则代理将解冻第 1 列第 2、3 和 4 行的候选对）。


   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      |      |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+


         Figure 4: Check List State with Succeeded Candidate Pair
         （图 4：具有 Succeeded 候选对的检查列表状态）

Trickle ICE preserves all of these rules as they apply to "static" check list sets. This implies that if a Trickle ICE agent were to begin connectivity checks with all of its pairs already present, the way that pair states change is indistinguishable from that of a regular ICE agent.

Trickle ICE 保留了所有这些规则，因为它们适用于"静态"检查列表集。这意味着如果 Trickle ICE 代理在开始连通性检查时其所有候选对已经存在，则候选对状态变化的方式与常规 ICE 代理无法区分。

Of course, the major difference with Trickle ICE is that check list sets can be dynamically updated because candidates can arrive after connectivity checks have started. When this happens, an agent sets the state of the newly formed pair as described below.

当然，Trickle ICE 的主要区别在于检查列表集可以动态更新，因为候选可能在连通性检查开始后到达。当这种情况发生时，代理如下所述设置新形成的候选对的状态。

Rule 1: If the newly formed pair has the lowest component ID and, if the component IDs are equal, the highest priority of any candidate pair for this foundation (i.e., if it is the topmost pair in the column), set the state to Waiting. For example, this would be the case if the newly formed pair were placed in column 5, row 1. This rule is consistent with Section 6.1.2.6 of [rfc5245bis].

规则 1：如果新形成的候选对具有此 foundation 的最低组件 ID，并且若组件 ID 相等则具有最高优先级（即，如果它是该列中最顶部的候选对），则将状态设置为 Waiting。例如，如果将新形成的候选对放在第 5 列第 1 行，就是这种情况。此规则与 [rfc5245bis] 第 6.1.2.6 节一致。


   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      | ^W^  |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   |      |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+


         Figure 5: Check List State with Newly Formed Pair, Rule 1
         （图 5：新形成候选对的检查列表状态，规则 1）

Rule 2: If there is at least one pair in the Succeeded state for this foundation, set the state to Waiting. For example, this would be the case if the pair in column 5, row 1 succeeded and the newly formed pair were placed in column 5, row 2. This rule is consistent with Section 7.2.5.3.3 of [rfc5245bis].

规则 2：如果此 foundation 至少有一个处于 Succeeded 状态的候选对，则将状态设置为 Waiting。例如，如果第 5 列第 1 行的候选对成功，并且新形成的候选对放在第 5 列第 2 行，就是这种情况。此规则与 [rfc5245bis] 第 7.2.5.3.3 节一致。


   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      |  S   |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   | ^W^  |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+


         Figure 6: Check List State with Newly Formed Pair, Rule 2
         （图 6：新形成候选对的检查列表状态，规则 2）

Rule 3: In all other cases, set the state to Frozen. For example, this would be the case if the newly formed pair were placed in column 3, row 3.

规则 3：在所有其他情况下，将状态设置为 Frozen。例如，如果将新形成的候选对放在第 3 列第 3 行，就是这种情况。


   +-----------------+------+------+------+------+------+
   |                 |  f1  |  f2  |  f3  |  f4  |  f5  |
   +-----------------+------+------+------+------+------+
   | s1 (Audio.RTP)  |  S   |  W   |  W   |      |  S   |
   +-----------------+------+------+------+------+------+
   | s2 (Audio.RTCP) |  W   |  F   |  F   |  W   |  W   |
   +-----------------+------+------+------+------+------+
   | s3 (Video.RTP)  |  W   |      | ^F^  |      |      |
   +-----------------+------+------+------+------+------+
   | s4 (Video.RTCP) |  W   |      |      |      |      |
   +-----------------+------+------+------+------+------+


         Figure 7: Check List State with Newly Formed Pair, Rule 3
         （图 7：新形成候选对的检查列表状态，规则 3）

---

## 13.  Generating an End-of-Candidates Indication（生成候选结束指示）

Once all candidate gathering is completed or expires for an ICE session associated with a specific data stream, the agent will generate an "end-of-candidates" indication for that session and convey it to the remote agent via the signaling channel. Although the exact form of the indication depends on the using protocol, the indication MUST specify the generation (Username Fragment and Password combination) so that an agent can correlate the end-of-candidates indication with a particular ICE session. The indication can be conveyed in the following ways:

一旦与特定数据流关联的 ICE 会话的所有候选收集完成或过期，代理将为该会话生成一个"候选结束"指示，并通过信令通道将其传送给远程代理。尽管指示的确切形式取决于使用协议，但指示必须（MUST）指定 generation（Username Fragment 和 Password 组合），以便代理可以将候选结束指示与特定的 ICE 会话关联起来。该指示可以通过以下方式传送：

o  As part of an initiation request (which would typically be the case with the initial ICE description for half trickle)
  （作为发起请求的一部分（通常是半增量式中初始 ICE 描述的情况））

o  Along with the last candidate an agent can send for a stream
  （与代理可以为流发送的最后一个候选一起）

o  As a standalone notification (e.g., after STUN Binding requests or TURN Allocate requests to a server time out and the agent is no longer actively gathering candidates)
  （作为独立通知（例如，在发送到服务器的 STUN 绑定请求或 TURN 分配请求超时且代理不再主动收集候选后））

Conveying an end-of-candidates indication in a timely manner is important in order to avoid ambiguities and speed up the conclusion of ICE processing. In particular:

及时传送候选结束指示对于避免歧义和加速 ICE 处理的结论非常重要。特别是：

o  A controlled Trickle ICE agent SHOULD convey an end-of-candidates indication after it has completed gathering for a data stream, unless ICE processing terminates before the agent has had a chance to complete gathering.
  （受控 Trickle ICE 代理应该（SHOULD）在完成数据流的收集后传送候选结束指示，除非 ICE 处理在代理有机会完成收集之前终止。）

o  A controlling agent MAY conclude ICE processing prior to conveying end-of-candidates indications for all streams. However, it is RECOMMENDED for a controlling agent to convey end-of-candidates indications whenever possible for the sake of consistency and to keep middleboxes and controlled agents up-to-date on the state of ICE processing.
  （控制代理可以（MAY）在传送所有流的候选结束指示之前完成 ICE 处理。但是，为了保持一致性并使中间盒和受控代理了解 ICE 处理的最新状态，建议（RECOMMENDED）控制代理在可能的情况下传送候选结束指示。）

When conveying an end-of-candidates indication during trickling (rather than as a part of the initial ICE description or a response thereto), it is the responsibility of the using protocol to define methods for associating the indication with one or more specific data streams.

在增量发送期间（而不是作为初始 ICE 描述或其响应的一部分）传送候选结束指示时，使用协议有责任定义将该指示与一个或多个特定数据流关联的方法。

An agent MAY also choose to generate an end-of-candidates indication before candidate gathering has actually completed, if the agent determines that gathering has continued for more than an acceptable period of time. However, an agent MUST NOT convey any more candidates after it has conveyed an end-of-candidates indication.

如果代理确定候选收集已经持续超过了可接受的时间长度，代理也可以（MAY）选择在候选收集实际完成之前生成候选结束指示。但是，代理不得（MUST NOT）在传送候选结束指示之后再传送任何候选。

When performing half trickle, an agent SHOULD convey an end-of-candidates indication together with its initial ICE description unless it is planning to potentially trickle additional candidates (e.g., in case the remote party turns out to support Trickle ICE).

在执行半增量式时，代理应该（SHOULD）将候选结束指示与其初始 ICE 描述一起传送，除非它计划可能增量发送额外的候选（例如，在远程方结果证明支持 Trickle ICE 的情况下）。

After an agent conveys the end-of-candidates indication, it will update the state of the corresponding check list as explained in Section 8. Past that point, an agent MUST NOT trickle any new candidates within this ICE session. Therefore, adding new candidates to the negotiation is possible only through an ICE restart (see Section 15).

在代理传送候选结束指示后，它将按照第 8 节中的说明更新相应检查列表的状态。此后，代理不得（MUST NOT）在此 ICE 会话中增量发送任何新候选。因此，只能通过 ICE 重启（见第 15 节）才能向协商中添加新候选。

This specification does not override regular ICE semantics for concluding ICE processing. Therefore, even if end-of-candidates indications are conveyed, an agent will still need to go through pair nomination. Also, if pairs have been nominated for components and data streams, ICE processing MAY still conclude even if end-of-candidates indications have not been received for all streams. In all cases, an agent MUST NOT trickle any new candidates within an ICE session after nomination of a candidate pair as described in Section 8.1.1 of [rfc5245bis].

本规范不会覆盖用于完成 ICE 处理的常规 ICE 语义。因此，即使传送了候选结束指示，代理仍然需要通过候选对提名过程。此外，如果已经为组件和数据流提名了候选对，即使尚未收到所有流的候选结束指示，ICE 处理仍然可以（MAY）完成。在所有情况下，代理不得（MUST NOT）在按照 [rfc5245bis] 第 8.1.1 节所述提名候选对之后在 ICE 会话中增量发送任何新候选。

---

## 14.  Receiving an End-of-Candidates Indication（接收候选结束指示）

Receiving an end-of-candidates indication enables an agent to update check list states and, in case valid pairs do not exist for every component in every data stream, determine that ICE processing has failed. It also enables an agent to speed up the conclusion of ICE processing when a candidate pair has been validated but it involves the use of lower-preference transports such as TURN. In such situations, an implementation MAY choose to wait and see if higher-priority candidates are received; in this case the end-of-candidates indication provides a notification that such candidates are not forthcoming.

接收候选结束指示使代理能够更新检查列表状态，并且在每个数据流中的每个组件都不存在有效候选对的情况下，确定 ICE 处理已失败。它也使代理能够在候选对已验证但涉及使用低优先级传输（如 TURN）时加速 ICE 处理的完成。在此类情况下，实现可以（MAY）选择等待，看是否会收到更高优先级的候选；在这种情况下，候选结束指示提供了不会再有此类候选的通知。

When an agent receives an end-of-candidates indication for a specific data stream, it will update the state of the relevant check list as per Section 8 (which might lead to some check lists being marked as Failed). If the check list is still in the Running state after the update, the agent will persist the fact that an end-of-candidates indication has been received and take it into account in future updates to the check list.

当代理收到特定数据流的候选结束指示时，它将根据第 8 节更新相关检查列表的状态（这可能导致一些检查列表被标记为 Failed）。如果检查列表在更新后仍处于 Running 状态，代理将持久保存已收到候选结束指示的事实，并在未来对检查列表的更新中将其考虑在内。

After an agent has received an end-of-candidates indication, it MUST ignore any newly received candidates for that data stream or data session.

在代理收到候选结束指示后，它必须（MUST）忽略任何新收到的该数据流或数据会话的候选。

---

## 15.  Subsequent Exchanges and ICE Restarts（后续交换和 ICE 重启）

Before conveying an end-of-candidates indication, either agent MAY convey subsequent candidate information at any time allowed by the using protocol. When this happens, agents will use [rfc5245bis] semantics (e.g., checking of the Username Fragment and Password combination) to determine whether or not the new candidate information requires an ICE restart.

在传送候选结束指示之前，任何一方代理都可以（MAY）在使用协议允许的任何时间传送后续候选信息。当发生这种情况时，代理将使用 [rfc5245bis] 语义（例如，检查 Username Fragment 和 Password 组合）来确定新的候选信息是否需要 ICE 重启。

If an ICE restart occurs, the agents can assume that Trickle ICE is still supported if support was determined previously, and thus can engage in Trickle ICE behavior as they would in an initial exchange of ICE descriptions where support was determined through a capabilities discovery method.

如果发生 ICE 重启，如果之前已确定支持，代理可以假定 Trickle ICE 仍然受支持，因此可以像在通过能力发现方法确定支持的初始 ICE 描述交换中那样，采用 Trickle ICE 行为。

---

## 16.  Half Trickle（半增量式）

In half trickle, the initiator conveys the initial ICE description with a usable but not necessarily full generation of candidates. This ensures that the ICE description can be processed by a regular ICE responder and is mostly meant for use in cases where support for Trickle ICE cannot be confirmed prior to conveying the initial ICE description. The initial ICE description indicates support for Trickle ICE, so that the responder can respond with something less than a full generation of candidates and then trickle the rest. The initial ICE description for half trickle can contain an end-of-candidates indication, although this is not mandatory because if trickle support is confirmed then the initiator can choose to trickle additional candidates before it conveys an end-of-candidates indication.

在半增量式中，发起方传送带有可用但不一定完整的一代候选的初始 ICE 描述。这确保了 ICE 描述可以被常规 ICE 应答方处理，主要用于在传送初始 ICE 描述之前无法确认 Trickle ICE 支持的情况。初始 ICE 描述指示支持 Trickle ICE，以便应答方可以使用少于完整一代的候选进行响应，然后增量发送其余部分。半增量式的初始 ICE 描述可以包含候选结束指示，尽管这不是强制性的，因为如果增量支持得到确认，发起方可以选择在传送候选结束指示之前增量发送额外的候选。

The half trickle mechanism can be used in cases where there is no way for an agent to verify in advance whether a remote party supports Trickle ICE. Because the initial ICE description contains a full generation of candidates, it can thus be handled by a regular ICE agent, while still allowing a Trickle ICE agent to use the optimization defined in this specification. This prevents negotiation from failing in the former case while still giving roughly half the Trickle ICE benefits in the latter.

半增量式机制可用于代理无法提前验证远程方是否支持 Trickle ICE 的情况。因为初始 ICE 描述包含完整的一代候选，因此它可以由常规 ICE 代理处理，同时仍然允许 Trickle ICE 代理使用本规范中定义的优化。这可以防止前一种情况下的协商失败，同时在后一种情况下仍然带来约一半的 Trickle ICE 优势。

Use of half trickle is only necessary during an initial exchange of ICE descriptions. After both parties have received an ICE description from their peer, they can each reliably determine Trickle ICE support and use it for all subsequent exchanges (see Section 15).

半增量式的使用仅在 ICE 描述的初始交换期间是必要的。在双方都从其对端收到 ICE 描述后，它们都可以可靠地确定 Trickle ICE 支持并在所有后续交换中使用它（见第 15 节）。

In some instances, using half trickle might bring more than just half the improvement in terms of user experience. This can happen when an agent starts gathering candidates upon user interface cues that the user will soon be initiating an interaction, such as activity on a keypad or the phone going off hook. This would mean that some or all of the candidate gathering could be completed before the agent actually needs to convey the candidate information. Because the responder will be able to trickle candidates, both agents will be able to start connectivity checks and complete ICE processing earlier than with regular ICE and potentially even as early as with full trickle.

在某些情况下，使用半增量式在用户体验方面带来的改善可能超过一半。当代理在用户即将发起交互的用户界面提示（例如键盘上的活动或电话摘机）时开始收集候选，这可能发生。这意味着在代理实际需要传送候选信息之前，部分或全部候选收集可能已经完成。由于应答方能够增量发送候选，两个代理都将能够比常规 ICE 更早地开始连通性检查并完成 ICE 处理，甚至可能与完全增量式一样早。

However, such anticipation is not always possible. For example, a multipurpose user agent or a WebRTC web page where communication is a non-central feature (e.g., calling a support line in case of a problem with the main features) would not necessarily have a way of distinguishing between call intentions and other user activity. In such cases, using full trickle is most likely to result in an ideal user experience. Even so, using half trickle would be an improvement over regular ICE because it would result in a better experience for responders.

然而，这种预判并不总是可能的。例如，一个多用途用户代理或一个 WebRTC 网页，其中通信并非核心功能（例如，在主功能出现问题时呼叫支持热线），不一定能够区分呼叫意图和其他用户活动。在这种情况下，使用完全增量式最有可能带来理想的用户体验。即便如此，使用半增量式也将是对常规 ICE 的改善，因为它将为应答方带来更好的体验。

---

## 17.  Preserving Candidate Order while Trickling（在增量发送中保持候选顺序）

One important aspect of regular ICE is that connectivity checks for a specific foundation and component are attempted simultaneously by both agents, so that any firewalls or NATs fronting the agents would whitelist both endpoints and allow all except for the first ("suicide") packets to go through. This is also important to unfreezing candidates at the right time. While not crucial, preserving this behavior in Trickle ICE is likely to improve ICE performance.

常规 ICE 的一个重要方面是，特定 foundation 和组件的连通性检查由两个代理同时尝试，以便位于代理前面的任何防火墙或 NAT 将两个端点都列入白名单，并允许除第一个（"自杀"）数据包之外的所有数据包通过。这对于在正确时间解冻候选也很重要。虽然不是至关重要，但在 Trickle ICE 中保持此行为可能提高 ICE 性能。

To achieve this, when trickling candidates, agents SHOULD respect the order of components as reflected by their component IDs; that is, candidates for a given component SHOULD NOT be conveyed prior to candidates for a component with a lower ID number within the same foundation. In addition, candidates SHOULD be paired, following the procedures in Section 12, in the same order they are conveyed.

为了实现这一点，在增量发送候选时，代理应该（SHOULD）尊重由组件 ID 反映的组件顺序；也就是说，给定组件的候选不应该（SHOULD NOT）在同一 foundation 中具有较低 ID 号的组件的候选之前被传送。此外，候选应该（SHOULD）按照它们被传送的相同顺序进行配对，遵循第 12 节中的程序。

For example, the following SDP description contains two components (RTP and RTCP) and two foundations (host and server reflexive):

例如，以下 SDP 描述包含两个组件（RTP 和 RTCP）和两个 foundation（host 和 server reflexive）：


     v=0
     o=jdoe 2890844526 2890842807 IN IP4 10.0.1.1
     s=
     c=IN IP4 10.0.1.1
     t=0 0
     a=ice-pwd:asd88fgpdd777uzjYhagZg
     a=ice-ufrag:8hhY
     m=audio 5000 RTP/AVP 0
     a=rtpmap:0 PCMU/8000
     a=candidate:1 1 UDP 2130706431 10.0.1.1 5000 typ host
     a=candidate:1 2 UDP 2130706431 10.0.1.1 5001 typ host
     a=candidate:2 1 UDP 1694498815 192.0.2.3 5000 typ srflx
         raddr 10.0.1.1 rport 8998
     a=candidate:2 2 UDP 1694498815 192.0.2.3 5001 typ srflx
         raddr 10.0.1.1 rport 8998


For this candidate information the RTCP host candidate would not be conveyed prior to the RTP host candidate. Similarly the RTP server reflexive candidate would be conveyed together with or prior to the RTCP server reflexive candidate.

对于此候选信息，RTCP 主机候选不会在 RTP 主机候选之前传送。同样，RTP 服务器反射候选将与 RTCP 服务器反射候选一起或在其之前传送。

---

## 18.  Requirements for Using Protocols（使用协议的要求）

In order to fully enable the use of Trickle ICE, this specification defines the following requirements for using protocols.

为了完全启用 Trickle ICE 的使用，本规范为使用协议定义了以下要求。

o  A using protocol SHOULD provide a way for parties to advertise and discover support for Trickle ICE before an ICE session begins (see Section 3).
  （使用协议应该（SHOULD）提供一种方式，使各方在 ICE 会话开始之前能够通告和发现对 Trickle ICE 的支持（参见第 3 节）。）

o  A using protocol MUST provide methods for incrementally conveying (i.e., "trickling") additional candidates after conveying the initial ICE description (see Section 9).
  （使用协议必须（MUST）提供在传送初始 ICE 描述后增量传送（即"增量发送"）额外候选的方法（参见第 9 节）。）

o  A using protocol MUST deliver each trickled candidate or end-of-candidates indication exactly once and in the same order it was conveyed (see Section 9).
  （使用协议必须（MUST）将每个增量候选或候选结束指示恰好一次地、以传送的相同顺序交付（参见第 9 节）。）

o  A using protocol MUST provide a mechanism for both parties to indicate and agree on the ICE session in force (see Section 9).
  （使用协议必须（MUST）提供一种机制，使双方能够指示和商定正在生效的 ICE 会话（参见第 9 节）。）

o  A using protocol MUST provide a way for parties to communicate the end-of-candidates indication, which MUST specify the particular ICE session to which the indication applies (see Section 13).
  （使用协议必须（MUST）提供一种方式，使各方能够传达候选结束指示，该指示必须（MUST）指定该指示适用的特定 ICE 会话（参见第 13 节）。）

---

## 19.  IANA Considerations（IANA 考虑）

IANA is requested to register the following ICE option in the "ICE Options" sub-registry of the "Interactive Connectivity Establishment (ICE) registry", following the procedures defined in [RFC6336].

请求 IANA 按照 [RFC6336] 中定义的程序，在"交互式连接建立（ICE）注册表"的"ICE 选项"子注册表中注册以下 ICE 选项。

ICE Option:  trickle
（ICE 选项：trickle）

Contact:  IESG, iesg@ietf.org
（联系方式：IESG, iesg@ietf.org）

Change control:  IESG
（变更控制：IESG）

Description:  An ICE option of "trickle" indicates support for incremental communication of ICE candidates.
（描述：ICE 选项 "trickle" 指示支持 ICE 候选的增量通信。）

Reference:  RFC XXXX
（参考资料：RFC XXXX）

---

## 20.  Security Considerations（安全考虑）

This specification inherits most of its semantics from [rfc5245bis] and as a result all security considerations described there apply to Trickle ICE.

本规范继承了 [rfc5245bis] 的大部分语义，因此其中描述的所有安全考虑都适用于 Trickle ICE。

If the privacy implications of revealing host addresses on an endpoint device are a concern (see for example the discussion in [I-D.ietf-rtcweb-ip-handling] and in Section 19 of [rfc5245bis]), agents can generate ICE descriptions that contain no candidates and then only trickle candidates that do not reveal host addresses (e.g., relayed candidates).

如果担心在端点上暴露主机地址的隐私影响（例如，参见 [I-D.ietf-rtcweb-ip-handling] 和 [rfc5245bis] 第 19 节中的讨论），代理可以生成不包含候选的 ICE 描述，然后仅增量发送不暴露主机地址的候选（例如，中继候选）。

---

## 21.  Acknowledgements（致谢）

The authors would like to thank Bernard Aboba, Flemming Andreasen, Rajmohan Banavi, Taylor Brandstetter, Philipp Hancke, Christer Holmberg, Ari Keranen, Paul Kyzivat, Jonathan Lennox, Enrico Marocco, Pal Martinsen, Nils Ohlmeier, Thomas Stach, Peter Thatcher, Martin Thomson, Brandon Williams, and Dale Worley for their reviews and suggestions on improving this document. Sarah Banks, Roni Even, and David Mandelberg completed opsdir, genart, and security reviews, respectively. Thanks also to Ari Keranen and Peter Thatcher in their role as chairs, and Ben Campbell in his role as responsible Area Director.

作者感谢 Bernard Aboba、Flemming Andreasen、Rajmohan Banavi、Taylor Brandstetter、Philipp Hancke、Christer Holmberg、Ari Keranen、Paul Kyzivat、Jonathan Lennox、Enrico Marocco、Pal Martinsen、Nils Ohlmeier、Thomas Stach、Peter Thatcher、Martin Thomson、Brandon Williams 和 Dale Worley 对改进本文档的审阅和建议。Sarah Banks、Roni Even 和 David Mandelberg 分别完成了 opsdir、genart 和安全审阅。也感谢 Ari Keranen 和 Peter Thatcher 作为主席的角色，以及 Ben Campbell 作为负责领域总监的角色。

---

## 22.  References（参考文献）

### 22.1.  Normative References（规范性参考文献）

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119, March 1997, <https://www.rfc-editor.org/info/rfc2119>.

[rfc5245bis]  Keranen, A., Holmberg, C., and J. Rosenberg, "Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal", draft-ietf-ice-rfc5245bis-20 (work in progress), March 2018.

（[rfc5245bis]  Keranen, A.、Holmberg, C. 和 J. Rosenberg，"交互式连接建立（ICE）：网络地址转换器（NAT）穿越协议"，draft-ietf-ice-rfc5245bis-20（进行中的工作），2018年3月。）

### 22.2.  Informative References（信息性参考文献）

[I-D.ietf-mmusic-trickle-ice-sip]  Ivov, E., Stach, T., Marocco, E., and C. Holmberg, "A Session Initiation Protocol (SIP) usage for Trickle ICE", draft-ietf-mmusic-trickle-ice-sip-14 (work in progress), February 2018.

（[I-D.ietf-mmusic-trickle-ice-sip]  Ivov, E.、Stach, T.、Marocco, E. 和 C. Holmberg，"Trickle ICE 的会话初始协议（SIP）用法"，draft-ietf-mmusic-trickle-ice-sip-14（进行中的工作），2018年2月。）

[I-D.ietf-rtcweb-ip-handling]  Uberti, J. and G. Shieh, "WebRTC IP Address Handling Requirements", draft-ietf-rtcweb-ip-handling-06 (work in progress), March 2018.

（[I-D.ietf-rtcweb-ip-handling]  Uberti, J. 和 G. Shieh，"WebRTC IP 地址处理要求"，draft-ietf-rtcweb-ip-handling-06（进行中的工作），2018年3月。）

[RFC1918]  Rekhter, Y., Moskowitz, B., Karrenberg, D., de Groot, G., and E. Lear, "Address Allocation for Private Internets", BCP 5, RFC 1918, DOI 10.17487/RFC1918, February 1996, <https://www.rfc-editor.org/info/rfc1918>.

[RFC3261]  Rosenberg, J., Schulzrinne, H., Camarillo, G., Johnston, A., Peterson, J., Sparks, R., Handley, M., and E. Schooler, "SIP: Session Initiation Protocol", RFC 3261, DOI 10.17487/RFC3261, June 2002, <https://www.rfc-editor.org/info/rfc3261>.

[RFC3264]  Rosenberg, J. and H. Schulzrinne, "An Offer/Answer Model with Session Description Protocol (SDP)", RFC 3264, DOI 10.17487/RFC3264, June 2002, <https://www.rfc-editor.org/info/rfc3264>.

[RFC4566]  Handley, M., Jacobson, V., and C. Perkins, "SDP: Session Description Protocol", RFC 4566, DOI 10.17487/RFC4566, July 2006, <https://www.rfc-editor.org/info/rfc4566>.

[RFC4787]  Audet, F., Ed. and C. Jennings, "Network Address Translation (NAT) Behavioral Requirements for Unicast UDP", BCP 127, RFC 4787, DOI 10.17487/RFC4787, January 2007, <https://www.rfc-editor.org/info/rfc4787>.

[RFC5389]  Rosenberg, J., Mahy, R., Matthews, P., and D. Wing, "Session Traversal Utilities for NAT (STUN)", RFC 5389, DOI 10.17487/RFC5389, October 2008, <https://www.rfc-editor.org/info/rfc5389>.

[RFC5766]  Mahy, R., Matthews, P., and J. Rosenberg, "Traversal Using Relays around NAT (TURN): Relay Extensions to Session Traversal Utilities for NAT (STUN)", RFC 5766, DOI 10.17487/RFC5766, April 2010, <https://www.rfc-editor.org/info/rfc5766>.

[RFC6120]  Saint-Andre, P., "Extensible Messaging and Presence Protocol (XMPP): Core", RFC 6120, DOI 10.17487/RFC6120, March 2011, <https://www.rfc-editor.org/info/rfc6120>.

[RFC6336]  Westerlund, M. and C. Perkins, "IANA Registry for Interactive Connectivity Establishment (ICE) Options", RFC 6336, DOI 10.17487/RFC6336, July 2011, <https://www.rfc-editor.org/info/rfc6336>.

[XEP-0030]  Hildebrand, J., Millard, P., Eatmon, R., and P. Saint-Andre, "XEP-0030: Service Discovery", XEP XEP-0030, June 2008.

[XEP-0176]  Beda, J., Ludwig, S., Saint-Andre, P., Hildebrand, J., Egan, S., and R. McQueen, "XEP-0176: Jingle ICE-UDP Transport Method", XEP XEP-0176, June 2009.

---

## Appendix A.  Interaction with Regular ICE（附录 A：与常规 ICE 的交互）

The ICE protocol was designed to be flexible enough to work in and adapt to as many network environments as possible. Despite that flexibility, ICE as specified in [rfc5245bis] does not by itself support trickle ICE. This section describes how trickling of candidates interacts with ICE.

ICE 协议被设计得足够灵活，可以在尽可能多的网络环境中工作并适应。尽管具有这种灵活性，但 [rfc5245bis] 中规定的 ICE 本身并不支持 Trickle ICE。本节描述候选的增量发送如何与 ICE 交互。

[rfc5245bis] describes the conditions required to update check lists and timer states while an ICE agent is in the Running state. These conditions are verified upon transaction completion and one of them stipulates that:

[rfc5245bis] 描述了在 ICE 代理处于 Running 状态时更新检查列表和定时器状态所需的条件。这些条件在事务完成时验证，其中之一规定：

   If there is not a pair in the valid list for each component of the data stream, the state of the check list is set to Failed.
   （如果数据流的每个组件的有效列表中没有候选对，则检查列表的状态设置为 Failed。）

This could be a problem and cause ICE processing to fail prematurely in a number of scenarios. Consider the following case:

这可能是一个问题，并在多种场景中导致 ICE 处理过早失败。考虑以下情况：

1.  Alice and Bob are both located in different networks with Network Address Translation (NAT). Alice and Bob themselves have different address but both networks use the same private internet block (e.g., the "20-bit block" 172.16/12 specified in [RFC1918]).
    （Alice 和 Bob 都位于具有网络地址转换（NAT）的不同网络中。Alice 和 Bob 本身具有不同的地址，但两个网络使用相同的专用互联网块（例如，[RFC1918] 中规定的"20 位块"172.16/12）。）

2.  Alice conveys to Bob the candidate 172.16.0.1 which also happens to correspond to an existing host on Bob's network.
    （Alice 向 Bob 传送候选 172.16.0.1，该候选恰好也对应 Bob 网络上的现有主机。）

3.  Bob creates a check list consisting solely of 172.16.0.1 and starts checks.
    （Bob 创建一个仅由 172.16.0.1 组成的检查列表并开始检查。）

4.  These checks reach the host at 172.16.0.1 in Bob's network, which responds with an ICMP "port unreachable" error; per [rfc5245bis] Bob marks the transaction as Failed.
    （这些检查到达 Bob 网络中位于 172.16.0.1 的主机，该主机以 ICMP "端口不可达"错误响应；根据 [rfc5245bis]，Bob 将事务标记为 Failed。）

At this point the check list only contains Failed candidates and the valid list is empty. This causes the data stream and potentially all ICE processing to fail, even though Trickle ICE agents could subsequently convey candidates that would cause previously empty check lists to become non-empty.

此时检查列表仅包含 Failed 候选，有效列表为空。这会导致数据流以及可能所有 ICE 处理失败，即使 Trickle ICE 代理随后可以传送会使原先为空的检查列表变为非空的候选。

A similar race condition would occur if the initial ICE description from Alice contains only candidates that can be determined as unreachable from any of the candidates that Bob has gathered (e.g., this would be the case if Bob's candidates only contain IPv4 addresses and the first candidate that he receives from Alice is an IPv6 one).

如果 Alice 的初始 ICE 描述仅包含可确定为从 Bob 已收集的任何候选不可达的候选，则会发生类似的竞态条件（例如，如果 Bob 的候选仅包含 IPv4 地址，而他收到的来自 Alice 的第一个候选是 IPv6 地址，就是这种情况）。

Another potential problem could arise when a non-trickle ICE implementation initiates an interaction with a Trickle ICE implementation. Consider the following case:

当非 Trickle ICE 实现启动与 Trickle ICE 实现的交互时，可能会出现另一个潜在问题。考虑以下情况：

1.  Alice's client has a non-Trickle ICE implementation.
    （Alice 的客户端具有非 Trickle ICE 实现。）

2.  Bob's client has support for Trickle ICE.
    （Bob 的客户端支持 Trickle ICE。）

3.  Alice and Bob are behind NATs with address-dependent filtering [RFC4787].
    （Alice 和 Bob 在具有地址相关过滤 [RFC4787] 的 NAT 后面。）

4.  Bob has two STUN servers but one of them is currently unreachable.
    （Bob 有两个 STUN 服务器，但其中一个当前不可达。）

After Bob's agent receives Alice's initial ICE description it would immediately start connectivity checks. It would also start gathering candidates, which would take a long time because of the unreachable STUN server. By the time Bob's answer is ready and conveyed to Alice, Bob's connectivity checks might have failed: until Alice gets Bob's answer, she won't be able to start connectivity checks and punch holes in her NAT. The NAT would hence be filtering Bob's checks as originating from an unknown endpoint.

在 Bob 的代理收到 Alice 的初始 ICE 描述后，它将立即开始连通性检查。它还将开始收集候选，由于 STUN 服务器不可达，这将花费很长时间。等到 Bob 的应答准备好并传送给 Alice 时，Bob 的连通性检查可能已经失败：在 Alice 收到 Bob 的应答之前，她无法开始连通性检查并在其 NAT 中打洞。因此 NAT 会将 Bob 的检查过滤为来自未知端点的流量。

---

## Appendix B.  Interaction with ICE Lite（附录 B：与 ICE Lite 的交互）

The behavior of ICE lite agents that are capable of Trickle ICE does not require any particular rules other than those already defined in this specification and [rfc5245bis]. This section is hence provided only for informational purposes.

能够使用 Trickle ICE 的 ICE Lite 代理的行为不需要除本规范和 [rfc5245bis] 中已定义的规则之外的任何特定规则。因此本节仅供信息性目的提供。

An ICE lite agent would generate candidate information as per [rfc5245bis] and would indicate support for Trickle ICE. Given that the candidate information will contain a full generation of candidates, it would also be accompanied by an end-of-candidates indication.

ICE Lite 代理将根据 [rfc5245bis] 生成候选信息并指示支持 Trickle ICE。鉴于候选信息将包含完整的一代候选，它还将附有候选结束指示。

When performing full trickle, a full ICE implementation could convey the initial ICE description or response thereto with no candidates. After receiving a response that identifies the remote agent as an ICE lite implementation, the initiator can choose to not trickle any additional candidates. The same is also true in the case when the ICE lite agent initiates the interaction and the full ICE agent is the responder. In these cases the connectivity checks would be enough for the ICE lite implementation to discover all potentially useful candidates as peer reflexive. The following example illustrates one such ICE session using SDP syntax:

在执行完全增量式时，完全 ICE 实现可以传送不包含候选的初始 ICE 描述或其响应。在收到将远程代理标识为 ICE Lite 实现的响应后，发起方可以选择不增量发送任何额外候选。当 ICE Lite 代理发起交互而完全 ICE 代理是应答方时，情况也是如此。在这些情况下，连通性检查将足以让 ICE Lite 实现发现所有可能作为对端反射的有用候选。以下示例使用 SDP 语法说明了这样一个 ICE 会话：


           ICE Lite                                          Bob
            Agent
              |   Offer (a=ice-lite a=ice-options:trickle)    |
              |---------------------------------------------->|
              |                                               |no cand
              |         Answer (a=ice-options:trickle)        |trickling
              |<----------------------------------------------|
              |              Connectivity Checks              |
              |<--------------------------------------------->|
     peer rflx|                                               |
    cand disco|                                               |
              |<========== CONNECTION ESTABLISHED ===========>|



                             Figure 8: Example
                             （图 8：示例）

In addition to reducing signaling traffic this approach also removes the need to discover STUN bindings or make TURN allocations, which can considerably lighten ICE processing.

除了减少信令流量外，这种方法还消除了发现 STUN 绑定或进行 TURN 分配的需要，这可以显著减轻 ICE 处理负担。

---

## Appendix C.  Changes from Earlier Versions（附录 C：与早期版本的变化）

Note to the RFC Editor: please remove this section prior to publication as an RFC.

（给 RFC 编辑的说明：请在作为 RFC 发布之前删除本节。）

### C.1.  Changes from draft-ietf-ice-trickle-20（从 draft-ietf-ice-trickle-20 的变化）

o  Slight corrections to handling of peer reflexive candidates.
  （对对端反射候选处理的轻微修正。）

o  Wordsmithing in a few sections.
  （少数几节的文字润色。）

### C.2.  Changes from draft-ietf-ice-trickle-19（从 draft-ietf-ice-trickle-19 的变化）

o  Further clarified handling of remote peer reflexive candidates.
  （进一步澄清了远程对端反射候选的处理。）

o  To improve readibility, renamed and restructured some sections and subsections, and modified some wording.
  （为了提高可读性，重命名和重组了一些节和小节，并修改了一些措辞。）

### C.3.  Changes from draft-ietf-ice-trickle-18（从 draft-ietf-ice-trickle-18 的变化）

o  Cleaned up pairing and redundancy checking rules for newly discovered candidates per IESG feedback and WG discussion.
  （根据 IESG 反馈和 WG 讨论，清理了新发现候选的配对和冗余检查规则。）

o  Improved wording in half trickle section.
  （改进了半增量式节的措辞。）

o  Changed "not more than once" to "exactly once".
  （将"不超过一次"改为"恰好一次"。）

o  Changed NAT examples back to IPv4.
  （将 NAT 示例改回 IPv4。）

### C.4.  Changes from draft-ietf-ice-trickle-17（从 draft-ietf-ice-trickle-17 的变化）

o  Simplified the rules for inserting a new pair in a check list.
  （简化了将新候选对插入检查列表的规则。）

o  Clarified it is not allowed to nominate a candidate pair after a pair has already been nominated (a.k.a. renomination or continuous nomination).
  （澄清了在已经提名候选对之后不允许再提名另一个候选对（即重新提名或持续提名）。）

o  Removed some text that referenced older versions of rfc5245bis.
  （删除了引用早期版本 rfc5245bis 的一些文本。）

o  Removed some text that duplicated concepts and procedures specified in rfc5245bis.
  （删除了与 rfc5245bis 中规定的概念和程序重复的一些文本。）

o  Removed the ill-defined concept of stream order.
  （删除了定义不明确的流顺序概念。）

o  Shortened the introduction.
  （缩短了引言。）

### C.5.  Changes from draft-ietf-ice-trickle-16（从 draft-ietf-ice-trickle-16 的变化）

o  Made "ufrag" terminology consistent with 5245bis.
  （使 "ufrag" 术语与 5245bis 一致。）

o  Applied in-order delivery rule to end-of-candidates indication.
  （将按序交付规则应用于候选结束指示。）

### C.6.  Changes from draft-ietf-ice-trickle-15（从 draft-ietf-ice-trickle-15 的变化）

o  Adjustments to address AD review feedback.
  （调整以处理 AD 审阅反馈。）

### C.7.  Changes from draft-ietf-ice-trickle-14（从 draft-ietf-ice-trickle-14 的变化）

o  Minor modifications to track changes to ICE core.
  （微调以跟踪 ICE 核心的变化。）

### C.8.  Changes from draft-ietf-ice-trickle-13（从 draft-ietf-ice-trickle-13 的变化）

o  Removed independent monitoring of check list "states" of frozen or active, since this is handled by placing a check list in the Running state defined in ICE core.
  （删除了对 frozen 或 active 的检查列表"状态"的独立监控，因为这是通过将检查列表置于 ICE 核心中定义的 Running 状态来处理的。）

### C.9.  Changes from draft-ietf-ice-trickle-12（从 draft-ietf-ice-trickle-12 的变化）

o  Specified that the end-of-candidates indication must include the generation (ufrag/pwd) to enable association with a particular ICE session.
  （规定候选结束指示必须包含 generation（ufrag/pwd），以便能够与特定 ICE 会话关联。）

o  Further editorial fixes to address WGLC feedback.
  （进一步的编辑修正以处理 WGLC 反馈。）

### C.10.  Changes from draft-ietf-ice-trickle-11（从 draft-ietf-ice-trickle-11 的变化）

o  Editorial and terminological fixes to address WGLC feedback.
  （编辑和术语修正以处理 WGLC 反馈。）

### C.11.  Changes from draft-ietf-ice-trickle-10（从 draft-ietf-ice-trickle-10 的变化）

o  Minor editorial fixes.
  （微小的编辑修正。）

### C.12.  Changes from draft-ietf-ice-trickle-09（从 draft-ietf-ice-trickle-09 的变化）

o  Removed immediate unfreeze upon Fail.
  （删除了 Fail 时的立即解冻。）

o  Specified MUST NOT regarding ice-options.
  （规定了关于 ice-options 的 MUST NOT。）

o  Changed terminology regarding initial ICE parameters to avoid implementer confusion.
  （更改了关于初始 ICE 参数的术语以避免实现者混淆。）

### C.13.  Changes from draft-ietf-ice-trickle-08（从 draft-ietf-ice-trickle-08 的变化）

o  Reinstated text about in-order processing of messages as a requirement for signaling protocols.
  （恢复了关于按序处理消息作为信令协议要求的文本。）

o  Added IANA registration template for ICE option.
  （添加了 ICE 选项的 IANA 注册模板。）

o  Corrected Case 3 rule in Section 8.1.1 to ensure consistency with regular ICE rules.
  （修正了第 8.1.1 节中的情况 3 规则，以确保与常规 ICE 规则一致。）

o  Added tabular representations to Section 8.1.1 in order to illustrate the new pair rules.
  （在第 8.1.1 节中添加了表格表示，以说明新的候选对规则。）

### C.14.  Changes from draft-ietf-ice-trickle-07（从 draft-ietf-ice-trickle-07 的变化）

o  Changed "ICE description" to "candidate information" for consistency with 5245bis.
  （将 "ICE description" 改为 "candidate information"，以与 5245bis 保持一致。）

### C.15.  Changes from draft-ietf-ice-trickle-06（从 draft-ietf-ice-trickle-06 的变化）

o  Addressed editorial feedback from chairs' review.
  （处理了主席审阅的编辑反馈。）

o  Clarified terminology regarding generations.
  （澄清了关于 generation 的术语。）

### C.16.  Changes from draft-ietf-ice-trickle-05（从 draft-ietf-ice-trickle-05 的变化）

o  Rewrote the text on inserting a new pair into a check list.
  （重写了关于将新候选对插入检查列表的文本。）

### C.17.  Changes from draft-ietf-ice-trickle-04（从 draft-ietf-ice-trickle-04 的变化）

o  Removed dependency on SDP and offer/answer model.
  （删除了对 SDP 和 offer/answer 模型的依赖。）

o  Removed mentions of aggressive nomination, since it is deprecated in 5245bis.
  （删除了对激进提名的提及，因为它在 5245bis 中已被弃用。）

o  Added section on requirements for signaling protocols.
  （添加了关于信令协议要求的章节。）

o  Clarified terminology.
  （澄清了术语。）

o  Addressed various WG feedback.
  （处理了各种 WG 反馈。）

### C.18.  Changes from draft-ietf-ice-trickle-03（从 draft-ietf-ice-trickle-03 的变化）

o  Provided more detailed description of unfreezing behavior, specifically how to replace pre-existing peer-reflexive candidates with higher-priority ones received via trickling.
  （提供了更详细的解冻行为描述，特别是如何通过增量发送收到的更高优先级候选替换预先存在的对端反射候选。）

### C.19.  Changes from draft-ietf-ice-trickle-02（从 draft-ietf-ice-trickle-02 的变化）

o  Adjusted unfreezing behavior when there are disparate foundations.
  （调整了存在不同 foundation 时的解冻行为。）

### C.20.  Changes from draft-ietf-ice-trickle-01（从 draft-ietf-ice-trickle-01 的变化）

o  Changed examples to use IPv6.
  （将示例更改为使用 IPv6。）

### C.21.  Changes from draft-ietf-ice-trickle-00（从 draft-ietf-ice-trickle-00 的变化）

o  Removed dependency on SDP (which is to be provided in a separate specification).
  （删除了对 SDP 的依赖（将在单独的规范中提供）。）

o  Clarified text about the fact that a check list can be empty if no candidates have been sent or received yet.
  （澄清了关于如果尚未发送或接收候选，检查列表可以为空的文本。）

o  Clarified wording about check list states so as not to define new states for "Active" and "Frozen" because those states are not defined for check lists (only for candidate pairs) in ICE core.
  （澄清了关于检查列表状态的措辞，以免为 "Active" 和 "Frozen" 定义新状态，因为这些状态在 ICE 核心中不是为检查列表（仅针对候选对）定义的。）

o  Removed open issues list because it was out of date.
  （删除了开放问题列表，因为它已过时。）

o  Completed a thorough copy edit.
  （完成了彻底的文本编辑。）

### C.22.  Changes from draft-mmusic-trickle-ice-02（从 draft-mmusic-trickle-ice-02 的变化）

o  Addressed feedback from Rajmohan Banavi and Brandon Williams.
  （处理了 Rajmohan Banavi 和 Brandon Williams 的反馈。）

o  Clarified text about determining support and about how to proceed if it can be determined that the answering agent does not support Trickle ICE.
  （澄清了关于确定支持以及如果能够确定应答代理不支持 Trickle ICE 时如何处理的文本。）

o  Clarified text about check list and timer updates.
  （澄清了关于检查列表和定时器更新的文本。）

o  Clarified when it is appropriate to use half trickle or to send no candidates in an offer or answer.
  （澄清了何时适合使用半增量式或在 offer 或 answer 中不发送候选。）

o  Updated the list of open issues.
  （更新了开放问题列表。）

### C.23.  Changes from draft-ivov-01 and draft-mmusic-00（从 draft-ivov-01 和 draft-mmusic-00 的变化）

o  Added a requirement to trickle candidates by order of components to avoid deadlocks in the unfreezing algorithm.
  （添加了按组件顺序增量发送候选的要求，以避免解冻算法中的死锁。）

o  Added an informative note on peer-reflexive candidates explaining that nothing changes for them semantically but they do become a more likely occurrence for Trickle ICE.
  （添加了关于对端反射候选的信息性说明，解释它们在语义上没有变化，但在 Trickle ICE 中确实变得更可能出现。）

o  Limit the number of pairs to 100 to comply with 5245.
  （将候选对数量限制为 100 以符合 5245。）

o  Added clarifications on the non-importance of how newly discovered candidates are trickled/sent to the remote party or if this is done at all.
  （添加了关于新发现候选如何被增量发送/发送到远程方或是否执行此操作不太重要的澄清。）

o  Added transport expectations for trickled candidates as per Dale Worley's recommendation.
  （根据 Dale Worley 的建议，添加了对增量候选的传输期望。）

### C.24.  Changes from draft-ivov-00（从 draft-ivov-00 的变化）

o  Specified that end-of-candidates is a media level attribute which can of course appear as session level, which is equivalent to having it appear in all m-lines. Also made end-of-candidates optional for cases such as aggressive nomination for controlled agents.
  （规定候选结束是一个媒体级别属性，当然可以作为会话级别出现，这等效于使其出现在所有 m-line 中。同时对于受控代理的激进提名等情况，使候选结束成为可选的。）

o  Added an example for ICE lite and Trickle ICE to illustrate how, when talking to an ICE lite agent, one doesn't need to send or even discover any candidates.
  （添加了一个 ICE Lite 和 Trickle ICE 的示例，说明在与 ICE Lite 代理通信时，不需要发送甚至不需要发现任何候选。）

o  Added wording that explicitly states ICE lite agents have to be prepared to receive no candidates over signaling and that they should not freak out if this happens. (Closed the corresponding open issue).
  （添加了明确说明 ICE Lite 代理必须准备好接收信令中没有任何候选的措辞，以及如果发生这种情况它们不应惊慌。（关闭了相应的开放问题）。）

o  It is now mandatory to use MID when trickling candidates and using m-line indexes is no longer allowed.
  （现在在增量发送候选时强制要求使用 MID，不再允许使用 m-line 索引。）

o  Replaced use of 0.0.0.0 to IP6 :: in order to avoid potential issues with RFC2543 SDP libraries that interpret 0.0.0.0 as an on-hold operation. Also changed the port number here from 1 to 9 since it already has a more appropriate meaning. (Port change suggested by Jonathan Lennox).
  （将 0.0.0.0 的使用替换为 IP6 ::，以避免 RFC2543 SDP 库将 0.0.0.0 解释为保持操作的潜在问题。还将此处的端口号从 1 更改为 9，因为它已经具有更合适的含义。（端口更改由 Jonathan Lennox 建议）。）

o  Closed the Open Issue about what to do with candidates received after end-of-cands. Solution: ignore, do an ICE restart if you want to add something.
  （关闭了关于收到候选结束后如何处理候选的开放问题。解决方案：忽略，如果要添加内容则执行 ICE 重启。）

o  Added more terminology, including trickling, trickled candidates, half trickle, full trickle.
  （添加了更多术语，包括 trickling、trickled candidates、half trickle、full trickle。）

o  Added a reference to the SIP usage for Trickle ICE as requested at the Boston interim.
  （根据波士顿临时会议的要求，添加了对 Trickle ICE 的 SIP 用法文档的引用。）

### C.25.  Changes from draft-rescorla-01（从 draft-rescorla-01 的变化）

o  Brought back explicit use of Offer/Answer. There are no more attempts to try to do this in an O/A independent way. Also removed the use of ICE Descriptions.
  （恢复了 Offer/Answer 的显式使用。不再尝试以独立于 O/A 的方式执行此操作。同时删除了 ICE Descriptions 的使用。）

o  Added SDP specification for trickled candidates, the trickle option and 0.0.0.0 addresses in m-lines, and end-of-candidates.
  （为增量候选、trickle 选项、m-line 中的 0.0.0.0 地址以及候选结束添加了 SDP 规范。）

o  Support and Discovery. Changed that section to be less abstract. As discussed in IETF85, the draft now says implementations and usages need to either determine support in advance and directly use trickle, or do half trickle. Removed suggestion about use of discovery in SIP or about letting implementing protocols do what they want.
  （支持和发现。将该节更改为不那么抽象。如在 IETF85 中讨论的，该草案现在指出实现和使用需要提前确定支持并直接使用增量式，或执行半增量式。删除了关于在 SIP 中使用发现或让实现协议做它们想做的事情的建议。）

o  Defined Half Trickle. Added a section that says how it works. Mentioned that it only needs to happen in the first o/a (not necessary in updates), and added Jonathan's comment about how it could, in some cases, offer more than half the improvement if you can pre-gather part or all of your candidates before the user actually presses the call button.
  （定义了半增量式。添加了一节说明其工作原理。提到它只需要在第一次 o/a 中发生（在更新中不需要），并添加了 Jonathan 的评论，指出在某些情况下，如果你可以在用户实际按下呼叫按钮之前预先收集部分或全部候选，它可以提供超过一半的改善。）

o  Added a short section about subsequent offer/answer exchanges.
  （添加了关于后续 offer/answer 交换的简短章节。）

o  Added a short section about interactions with ICE Lite implementations.
  （添加了关于与 ICE Lite 实现交互的简短章节。）

o  Added two new entries to the open issues section.
  （在开放问题节中添加了两个新条目。）

### C.26.  Changes from draft-rescorla-00（从 draft-rescorla-00 的变化）

o  Relaxed requirements about verifying support following a discussion on MMUSIC.
  （根据 MMUSIC 上的讨论，放宽了关于验证支持的要求。）

o  Introduced ICE descriptions in order to remove ambiguous use of 3264 language and inappropriate references to offers and answers.
  （引入了 ICE 描述，以消除 3264 语言的含糊使用和对 offer 和 answer 的不当引用。）

o  Removed inappropriate assumption of adoption by RTCWEB pointed out by Martin Thomson.
  （删除了 Martin Thomson 指出的关于 RTCWEB 采用的不当假设。）

---

## Authors' Addresses（作者地址）

Emil Ivov
Atlassian
303 Colorado Street, #1600
Austin, TX  78701
USA

Phone: +1-512-640-3000
Email: eivov@atlassian.com

（Emil Ivov
Atlassian
303 Colorado Street, #1600
Austin, TX  78701
美国

电话：+1-512-640-3000
电子邮箱：eivov@atlassian.com）

Eric Rescorla
RTFM, Inc.
2064 Edgewood Drive
Palo Alto, CA  94303
USA

Phone: +1 650 678 2350
Email: ekr@rtfm.com

（Eric Rescorla
RTFM, Inc.
2064 Edgewood Drive
Palo Alto, CA  94303
美国

电话：+1 650 678 2350
电子邮箱：ekr@rtfm.com）

Justin Uberti
Google
747 6th St S
Kirkland, WA  98033
USA

Phone: +1 857 288 8888
Email: justin@uberti.name

（Justin Uberti
Google
747 6th St S
Kirkland, WA  98033
美国

电话：+1 857 288 8888
电子邮箱：justin@uberti.name）

Peter Saint-Andre
Mozilla
P.O. Box 787
Parker, CO  80134
USA

Phone: +1 720 256 6756
Email: stpeter@mozilla.com
URI:   https://www.mozilla.com/

（Peter Saint-Andre
Mozilla
P.O. Box 787
Parker, CO  80134
美国

电话：+1 720 256 6756
电子邮箱：stpeter@mozilla.com
URI：https://www.mozilla.com/）
