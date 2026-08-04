---
title: SDP: Session Description Protocol (RFC 4566)
type: protocol
tags: [webrtc, sdp, session-description, protocol, rfc4566]
sources: [raw/rfcs/rfc4566.md]
created: 2026-04-27
updated: 2026-04-27
---

# SDP: Session Description Protocol (RFC 4566)

## 一句话理解

**SDP = WebRTC 会话描述协议，用于描述多媒体会话的参数（媒体类型、地址、编码格式）**

---

## 协议概述

### 什么是 SDP？

SDP (Session Description Protocol) 用于**描述多媒体会话**，但不传输媒体本身。

### SDP 的用途

| 用途 | 说明 |
|------|------|
| **会话公告** | 公告多播会话信息 |
| **会话邀请** | 邀请参与者加入会话 |
| **其他会话启动** | 其他形式的多媒体会话初始化 |

### SDP 不能做什么

> **SDP 不支持会话内容或媒体编码的协商**

这由其他协议（如 SIP 的 Offer/Answer 模型）处理。

---

## SDP 格式

### 基本格式

SDP 由多行文本组成，格式为：

```
<type>=<value>
```

- **type**: 单个字符（区分大小写）
- **value**: 结构化文本
- **必须严格按顺序排列**

### SDP 结构

```
┌─────────────────────────────────────────────────────────────┐
│  SDP 消息结构                                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │  Session-level (会话级)                           │        │
│  │  v=, o=, s=, i=, u=, e=, p=, c=, b=, t=, r=,     │        │
│  │  z=, k=, a=                                       │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
│  ┌─────────────────────────────────────────────────┐        │
│  │  Media-level (媒体级) - 零个或多个               │        │
│  │  m=, i=, c=, b=, k=, a=                         │        │
│  └─────────────────────────────────────────────────┘        │
│                                                              │
│  * = 可选字段                                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## SDP 字段详解

### 会话级字段 (Session-level)

| 字段 | 含义 | 必需 | 示例 |
|------|------|------|------|
| **v=** | 协议版本 | ✅ | `v=0` |
| **o=** | 会话发起者 | ✅ | `o=alice 53655765 2353687637 IN IP4 10.0.0.1` |
| **s=** | 会话名称 | ✅ | `s=SDP Seminar` |
| **i=** | 会话信息 | * | `i=A Seminar on SDP` |
| **u=** | URI | * | `u=http://example.com/sdp` |
| **e=** | Email | * | `e=j.doe@example.com` |
| **p=** | 电话 | * | `p=+1 555-1234` |
| **c=** | 连接信息 | * | `c=IN IP4 224.2.17.12/127` |
| **b=** | 带宽信息 | * | `b=CT:1000` |
| **t=** | 会话时间 | ✅ | `t=2873397496 2873404696` |
| **r=** | 重复时间 | * | `r=604800 3600 0 0 0` |
| **z=** | 时区调整 | * | `z=2882844526 -1h 2898848076 0` |
| **k=** | 加密密钥 | * | `k=clear:1234` |
| **a=** | 会话属性 | * | `a=recvonly` |

### 媒体级字段 (Media-level)

| 字段 | 含义 | 必需 | 示例 |
|------|------|------|------|
| **m=** | 媒体名称和端口 | ✅ | `m=audio 49170 RTP/AVP 0` |
| **i=** | 媒体信息 | * | `i=Audio Stream` |
| **c=** | 连接信息 | * | `c=IN IP4 224.2.17.12/127` |
| **b=** | 带宽信息 | * | `b=AS:128` |
| **k=** | 加密密钥 | * | `k=uri:http://example.com/key` |
| **a=** | 媒体属性 | * | `a=rtpmap:0 PCMU/8000` |

---

## 字段详解

### v= (Protocol Version)

```sdp
v=0
```

协议版本，当前只有 **v=0**。

### o= (Origin)

```sdp
o=<username> <sess-id> <sess-version> <nettype> <addrtype> <unicast-address>
```

示例：
```sdp
o=alice 53655765 2353687637 IN IP4 10.0.0.1
```

| 字段 | 说明 |
|------|------|
| **username** | 用户登录名，`-` 表示无 |
| **sess-id** | 会话标识符，全局唯一 |
| **sess-version** | 会话版本号，修改时递增 |
| **nettype** | 网络类型，`IN` 表示 Internet |
| **addrtype** | 地址类型，`IP4` 或 `IP6` |
| **unicast-address** | 主机地址（域名或 IP） |

### s= (Session Name)

```sdp
s=WebRTC Video Call
```

会话名称。

### c= (Connection Information)

```sdp
c=IN IP4 224.2.17.12/127
```

连接信息格式：
```sdp
c=<nettype> <addrtype> <connection-address>
```

| 类型 | 示例 |
|------|------|
| **多播** | `c=IN IP4 224.2.17.12/127` |
| **单播** | `c=IN IP4 192.168.1.100` |

### t= (Timing)

```sdp
t=2873397496 2873404696
```

时间格式：
```sdp
t=<start-time> <stop-time>
```

| 字段 | 说明 |
|------|------|
| **start-time** | 开始时间（NTP 时间戳） |
| **stop-time** | 结束时间（NTP 时间戳） |

### m= (Media Description)

```sdp
m=audio 49170 RTP/AVP 0
```

格式：
```sdp
m=<media> <port> <proto> <fmt> [fmt2] [fmt3]...
```

| 字段 | 说明 | 示例 |
|------|------|------|
| **media** | 媒体类型 | `audio`, `video`, `application` |
| **port** | 端口号 | `49170` |
| **proto** | 传输协议 | `RTP/AVP`, `RTP/SAVP` |
| **fmt** | 负载类型 | `0`, `8`, `96` (动态) |

### a= (Attribute)

属性字段是 SDP 的**主要扩展机制**。

#### 会话级属性

```sdp
a=recvonly
a=sendrecv
a=sendonly
a=inactive
```

#### 媒体级属性

```sdp
a=rtpmap:96 H264/90000
a=fmtp:96 profile-level-id=42E016;packetization-mode=1
a=candidate:1 1 UDP 2130706431 192.168.1.100 5000 typ host
```

| 类型 | 格式 | 说明 |
|------|------|------|
| **媒体属性** | `a=rtpmap:<payload> <codec>/<clock>` | 映射负载类型到编码 |
| **格式参数** | `a=fmtp:<payload> <params>` | 编码参数 |
| **ICE 候选** | `a=candidate:<foundation> ...` | ICE 候选地址 |
| **连接方向** | `a=sendrecv` | 发送/接收方向 |

---

## 完整 SDP 示例

### 简单音频会话

```sdp
v=0
o=alice 53655765 2353687637 IN IP4 10.0.0.1
s=Audio Session
c=IN IP4 192.168.1.100
t=0 0
m=audio 5004 RTP/AVP 0 8 96
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:96 G722/8000
a=sendrecv
```

### WebRTC 视频会话

```sdp
v=0
o=- 53655765 2353687637 IN IP4 10.0.0.1
s=WebRTC Video Call
t=0 0
a=group:BUNDLE 0 1

m=audio 5004 UDP/TLS/RTP/SAVPF 111 103 104
c=IN IP4 0.0.0.0
a=rtcp:5005 IN IP4 0.0.0.0
a=ice-ufrag:abcd
a=ice-pwd:abcdefghijklmnopqrstuvwxyz
a=ice-options:trickle
a=fingerprint:sha-256 00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33
a=setup:actpass
a=mid:0
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:104 telephone-event/8000
a=ssrc:1000 cname:abcd1234

m=video 5006 UDP/TLS/RTP/SAVPF 107 108
c=IN IP4 0.0.0.0
a=rtcp:5007 IN IP4 0.0.0.0
a=ice-ufrag:efgh
a=ice-pwd:zyxwvutsrqponmlkjihgfedcba
a=ice-options:trickle
a=fingerprint:sha-256 00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33
a=setup:actpass
a=mid:1
a=rtpmap:107 H264/90000
a=fmtp:107 profile-level-id=42E016;packetization-mode=1
a=rtpmap:108 VP8/90000
a=ssrc:2000 cname:abcd1234
a=ssrc:2001 cname:abcd1234
```

---

## WebRTC 中的 SDP

### SDP 在 WebRTC 信令中的位置

```
┌─────────────────────────────────────────────────────────────┐
│  WebRTC 信令流程                                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Peer A                              Peer B                 │
│    │                                   │                   │
│    │ 1. createOffer() → SDP            │                   │
│    │───setLocalDescription(SDP)───▶   │                   │
│    │                                   │                   │
│    │ 2. 通过信令通道发送 SDP            │                   │
│    │───INVITE + SDP ──────────────────▶│                   │
│    │                                   │                   │
│    │ 3. 对方设置 SDP                    │                   │
│    │      setRemoteDescription(SDP)    │                   │
│    │                                   │                   │
│    │ 4. 通过 ICE 交换 candidates         │                   │
│    │◀══════════════════════════════════▶│                   │
│    │                                   │                   │
│    │ 5. createAnswer() → SDP            │                   │
│    │◀──setRemoteDescription(SDP)───────│                   │
│    │                                   │                   │
└─────────────────────────────────────────────────────────────┘
```

### SDP Offer/Answer 模型

SIP 使用 **Offer/Answer 模型** 协商媒体参数：

```
┌─────────────────────────────────────────────────────────────┐
│  SDP Offer/Answer 流程                                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Caller:                    Callee:                         │
│    │                          │                             │
│    │ 1. createOffer()         │                             │
│    │    SDP Offer ──────────▶│  setRemoteDescription()    │
│    │                          │                             │
│    │ 2. ICE 候选交换          │                             │
│    │◀══════════════════════════▶│                             │
│    │                          │                             │
│    │ 3. createAnswer()        │                             │
│    │◀── SDP Answer ◀────────│  setRemoteDescription()    │
│    │                          │                             │
│    │ 4. setLocalDescription() │  setLocalDescription()     │
│    │    (确认 Answer)         │  (确认 Offer)               │
│    │                          │                             │
└─────────────────────────────────────────────────────────────┘
```

### SDP 中的 WebRTC 特定属性

| 属性 | 用途 |
|------|------|
| `a=group:BUNDLE` | 多路复用（捆绑媒体流） |
| `a=ice-ufrag` | ICE 用户名片段 |
| `a=ice-pwd` | ICE 密码 |
| `a=fingerprint` | DTLS 证书指纹 |
| `a=setup` | DTLS 角色协商 |
| `a=candidate` | ICE 候选地址 |

---

## ICE 与 SDP

### ICE 候选在 SDP 中的表示

```
a=candidate:1 1 UDP 2130706431 192.168.1.100 5000 typ host
```

格式：
```sdp
a=candidate:<foundation> <component-id> <transport> <priority>
              <connection-address> <port> typ <candidate-type>
```

| 字段 | 说明 | 示例 |
|------|------|------|
| **foundation** | 候选标识符 | `1` |
| **component-id** | 组件 ID (1=RTP, 2=RTCP) | `1` |
| **transport** | 传输协议 | `UDP` |
| **priority** | 优先级 | `2130706431` |
| **connection-address** | IP 地址 | `192.168.1.100` |
| **port** | 端口 | `5000` |
| **candidate-type** | 候选类型 | `host`, `srflx`, `relay` |

### 候选类型

| 类型 | candidate-type | 说明 |
|------|-----------------|------|
| Host | `host` | 本地网络接口 |
| Server-Reflexive | `srflx` | STUN 服务器发现的公网地址 |
| Peer-Reflexive | `prflx` | 连接时 NAT 分配的地址 |
| Relay | `relay` | TURN 服务器中继地址 |

---

## 一句话总结

> **SDP = 描述 WebRTC 会话参数的协议**
>
> **m= 行定义媒体类型和端口，a= 属性定义编码和能力**
>
> **通过 SIP Offer/Answer 交换 SDP，协商媒体参数**
>
> **ICE candidates 通过 a=candidate 属性携带**

---

## 相关链接

- [[wiki/tutorials/tutorial-udp-hole-punching]] - UDP 打洞流程详解（SDP = 候选信息的载体格式）
- [[wiki/protocols/protocol-rfc3261]] - SIP 协议分析
- [[wiki/protocols/protocol-rfc8445]] - ICE RFC 8445 协议分析
- [[wiki/protocols/protocol-rfc5389]] - STUN RFC 5389 协议分析
- [[wiki/tutorials/tutorial-ice-candidate-gathering]] - ICE 候选收集流程
- [[wiki/tutorials/tutorial-ice-connectivity-checks]] - ICE 连接性检查
- [[wiki/protocols/protocol-rfc3550]] - RTP/RTCP 协议分析
- [[wiki/protocols/protocol-rfc3551]] - RTP A/V Profile
- [[wiki/tutorials/tutorial-ice-nomination]] - ICE 提名流程
- [[raw/rfcs/rfc4566]] - RFC 4566 英文原文
