# WebRTC 知识库 - Index

> 本目录由 LLM 维护，最后更新: 2026-05-28

## Wiki 页面

### 概念 (Concepts)
| 页面                                 | 描述          | 标签               |
| ---------------------------------- | ----------- | ---------------- |
| [[wiki/concepts/concept-ice]]      | ICE 通俗理解    | webrtc, ice      |
| [[wiki/concepts/concept-protocol]] | 协议通俗理解      | webrtc, protocol |
| [[wiki/concepts/concept-glossary]] | WebRTC 缩写字典 | webrtc, glossary |

### 协议 (Protocols)
|| 页面 | 描述 | 来源 |
||------|------|------|
|| [[wiki/protocols/protocol-rfc8445]] | ICE NAT穿透协议 | RFC 8445 |
|| [[wiki/protocols/protocol-rfc5389]] | STUN 协议分析 | RFC 5389 |
|| [[wiki/protocols/protocol-rfc5766]] | TURN 中继协议分析 | RFC 5766 |
|| [[wiki/protocols/protocol-rfc5128]] | P2P 跨 NAT 通信状态 | RFC 5128 |
|| [[wiki/protocols/protocol-rfc3261]] | SIP 会话发起协议 | RFC 3261 |
|| [[wiki/protocols/protocol-rfc4566]] | SDP 会话描述协议 | RFC 4566 |
|| [[wiki/protocols/protocol-rfc3550]] | RTP/RTCP 实时传输协议 | RFC 3550 |
|| [[wiki/protocols/protocol-rfc3551]] | RTP A/V Profile 负载类型映射 | RFC 3551 |
|| [[wiki/protocols/protocol-rfc3711]] | SRTP 安全实时传输协议 | RFC 3711 |
|| [[wiki/protocols/protocol-rfc6716]] | Opus 音频编解码器 | RFC 6716 |
|| [[wiki/protocols/protocol-transport-wide-cc]] | 传输层拥塞控制扩展 | draft-holmer |
|| [[wiki/protocols/protocol-remb]] | 接收端最大码率估计 (REMB) | draft-alvestrand |
|| [[wiki/protocols/protocol-gcc]] | 谷歌拥塞控制算法 (GCC) | draft-ietf-rmcat |
|| [[wiki/protocols/protocols-relationship]] | 协议关系图谱 | Canvas |

### 对比分析 (Comparisons)
| 页面 | 描述 | 标签 |
|------|------|------|
| [[wiki/comparisons/comparison-turn-rtp-socket]] | TURN vs 原生 Socket 转发 RTP/RTCP 对比 | webrtc, turn, socket |
| [[wiki/comparisons/comparison-ice-error-handling]] | ICE 异常处理完整分析 + STUN/TURN/ICE 错误码参考 (13 个错误码) | webrtc, ice, error-handling |

### 教程 (Tutorials)
| 页面 | 描述 | 标签 |
|------|------|------|
| [[wiki/tutorials/tutorial-ice-candidate-gathering]] | ICE 候选收集详细流程 (含 STUN/TURN TLV) | webrtc, ice, tutorial |
| [[wiki/tutorials/tutorial-ice-connectivity-checks]] | ICE 连接性检查与提名详解 (4-way Handshake) | webrtc, ice, tutorial |
| [[wiki/tutorials/tutorial-udp-hole-punching]] | UDP 打洞流程详解 (含 NAT 类型适配、时序分析) | webrtc, nat, p2p, tutorial |
| [[wiki/tutorials/tutorial-webrtc-api-usage]] | WebRTC C++ API 使用方法总结 (基于 peerconnection 示例) | webrtc, api, peerconnection, tutorial, cpp |

### 源码分析 (Source Analysis)
| 页面 | 描述 | 标签 |
|------|------|------|
| [[wiki/libnice/libnice-error-handling]] | libnice 各协议层错误/异常处理完整分析 (含 StunError 21 枚举) | libnice, ice, stun, turn, error-handling |

### 任务管理 (Tasks)
| 页面 | 描述 | 标签 |
|------|------|------|
| [[wiki/daily-tasks]] | 日常任务库：工作目标、学习目标、定时任务汇总 | tasks, webrtc, life |

---

## Raw Sources (原始资料)

### RFCs
| 文件 | 说明 |
|------|------|
| [[raw/rfcs/rfc8445_完整中英对照.md]] | RFC 8445 完整中英对照 |
| [[raw/rfcs/rfc8445_中英对照.md]] | RFC 8445 中英对照（旧版）|
| [[raw/rfcs/rfc8445_中文.md]] | RFC 8445 中文翻译 |
| [[raw/rfcs/rfc8445.md]] | RFC 8445 英文原文 |
| [[raw/rfcs/rfc5389_完整中英对照.md]] | RFC 5389 完整中英对照 |
| [[raw/rfcs/rfc5766_完整中英对照.md]] | RFC 5766 完整中英对照 |
| [[raw/rfcs/rfc6887_完整中英对照.md]] | RFC 6887 完整中英对照 |
| [[raw/rfcs/rfc7350_完整中英对照.md]] | RFC 7350 完整中英对照 |
| [[raw/rfcs/rfc5128.md]] | RFC 5128 P2P跨NAT通信状态 |
| [[raw/rfcs/rfc5128_完整中英对照.md]] | RFC 5128 完整中英对照 |
| [[raw/rfcs/rfc4566.md]] | RFC 4566 SDP 会话描述协议 |
| [[raw/rfcs/rfc4566_完整中英对照.md]] | RFC 4566 完整中英对照 |
| [[raw/rfcs/rfc3261.md]] | RFC 3261 SIP 会话发起协议 |
| [[raw/rfcs/rfc3261_完整中英对照.md]] | RFC 3261 完整中英对照 |
| [[raw/rfcs/rfc3261_完整中英对照_Part1.md]] | RFC 3261 完整中英对照（第一部分，旧版）|
| [[raw/rfcs/rfc4585.md]] | RFC 4585 RTP/AVPF 英文原文 |
| [[raw/rfcs/rfc4585_完整中英对照.md]] | RFC 4585 完整中英对照 |
| [[raw/rfcs/rfc3551.md]] | RFC 3551 RTP Profile 音视频会议配置 |
| [[raw/rfcs/rfc3551_完整中英对照.md]] | RFC 3551 完整中英对照 |
| [[raw/rfcs/rfc3711.md]] | RFC 3711 SRTP 英文原文 |
| [[raw/rfcs/rfc3711_完整中英对照.md]] | RFC 3711 完整中英对照 |
| [[raw/rfcs/rfc6716.md]] | RFC 6716 Opus 英文原文 |
| [[raw/rfcs/rfc6716_完整中英对照.md]] | RFC 6716 完整中英对照 |
| [[raw/rfcs/draft-holmer-rmcat-transport-wide-cc-extensions-01.md]] | Transport-wide CC 英文原文 |
| [[raw/rfcs/draft-holmer-rmcat-transport-wide-cc-extensions-01_完整中英对照.md]] | Transport-wide CC 完整中英对照 |
| [[raw/rfcs/draft-alvestrand-rmcat-remb-03.md]] | REMB 英文原文 |
| [[raw/rfcs/draft-alvestrand-rmcat-remb-03_完整中英对照.md]] | REMB 完整中英对照 |
| [[raw/rfcs/draft-ietf-rmcat-gcc-02.md]] | GCC 英文原文 |
| [[raw/rfcs/draft-ietf-rmcat-gcc-02_完整中英对照.md]] | GCC 完整中英对照 |
| [[raw/rfcs/draft-ietf-ice-trickle-21.md]] | Trickle ICE 英文原文 |
| [[raw/rfcs/draft-ietf-ice-trickle-21_完整中英对照.md]] | Trickle ICE 完整中英对照 |

### XEPs
| 文件 | 说明 |
|------|------|
| [[raw/xep/xep-0176.md]] | XEP-0176 Jingle ICE-UDP 英文原文 |
| [[raw/xep/xep-0176_完整中英对照.md]] | XEP-0176 完整中英对照 |

### Papers (论文)
_(暂无)_

### Articles (文章)
| 文件 | 说明 |
|------|------|
| [[karpathy的wiki方法论_完整中英对照.md]] | Karpathy LLM Wiki 方法论完整中英对照 |

---

## 统计

- Wiki 页面: 20
- Raw 文档: 35 (15 RFC + 6 drafts + 18 翻译 + 2 XEP)
- 最后更新: 2026-06-04
