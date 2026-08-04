---
title: WebRTC 缩写字典
type: concept
tags: [webrtc, glossary, nat-traversal]
sources: []
created: 2026-04-22
updated: 2026-04-23
---

# WebRTC 缩写字典

## NAT 穿透协议

| 缩写 | 全称 | 中文 | 说明 |
|------|------|------|------|
| **NAT** | Network Address Translation | 网络地址转换 | 网络设备，将私有 IP 映射为公网 IP |
| **STUN** | Session Traversal Utilities for NAT | NAT 会话穿越工具 | 发现 NAT 分配的公网地址 |
| **TURN** | Traversal Using Relays around NAT | 通过中继绕过 NAT | 当直连失败时，用服务器转发数据 |
| **ICE** | Interactive Connectivity Establishment | 交互式连接建立 | 完整 NAT 穿透方案，系统性找出最优路径 |
| **EIM** | Endpoint-Independent Mapping | 端点无关映射 | 同一内部地址发给任何目标，映射到同一公网端口 |
| **EDM** | Endpoint-Dependent Mapping | 端点相关映射 | 同一内部地址发给不同目标，映射到不同公网端口 |
| **SDP** | Session Description Protocol | 会话描述协议 | 描述媒体会话的格式 |
| **SDES** | Session Description Protocol Security Descriptions | SDP 安全描述 | SDP 的加密描述 |
| **DTLS** | Datagram Transport Layer Security | 数据报传输层安全 | UDP 的 TLS |
| **SRTP** | Secure Real-time Transport Protocol | 安全实时传输协议 | RTP 的加密版本 |
| **RTP** | Real-time Transport Protocol | 实时传输协议 | 实时媒体传输 |
| **RTCP** | RTP Control Protocol | RTP 控制协议 | RTP 的控制协议 |

---

## NAT 类型分类

| 缩写 | 全称 | 中文 | 说明 |
|------|------|------|------|
| **Full Cone NAT** | Full Cone NAT | 完全圆锥型 NAT | 端点无关映射 + 无限制过滤 |
| **Restricted Cone NAT** | Restricted Cone NAT | 受限圆锥型 NAT | 端点无关映射 + 仅已访问 IP 可入站 |
| **Port-Restricted Cone NAT** | Port-Restricted Cone NAT | 端口受限圆锥型 NAT | 端点无关映射 + 仅已访问 IP:端口可入站 |
| **Symmetric NAT** | Symmetric NAT | 对称型 NAT | 端点相关映射，P2P 穿透最困难 |

### NAT 类型速查

```
EIM-NAT (P2P 友好) ✅
├── Full Cone NAT       → 打洞最容易
├── Restricted Cone NAT  → 打洞容易
└── Port-Restricted NAT → 打洞中等

EDM-NAT (P2P 不友好) ❌
└── Symmetric NAT       → 无法打洞，必须 TURN 中继
```

---

## 快速区分

```
STUN:  发现地址（我公网 IP 是什么）
TURN:  中继数据（直连不行，用服务器转发）
ICE:   完整方案（STUN + TURN + 系统性测试）

EIM:   一视同仁（发给谁，端口都一样）
EDM:   区别对待（发给谁，端口都可能不同）
```

---

## 快速区分

```
STUN:  发现地址（我公网 IP 是什么）
TURN:  中继数据（直连不行，用服务器转发）
ICE:   完整方案（STUN + TURN + 系统性测试）
```

---

## 其他常见缩写

| 缩写 | 全称 | 中文 |
|------|------|------|
| **IP** | Internet Protocol | 网际协议 |
| **UDP** | User Datagram Protocol | 用户数据报协议 |
| **TCP** | Transmission Control Protocol | 传输控制协议 |
| **TLS** | Transport Layer Security | 传输层安全 |
| **DNS** | Domain Name System | 域名系统 |
| **SRV** | Service | 服务记录 |
| **TTL** | Time To Live | 生存时间 |
| **MTU** | Maximum Transmission Unit | 最大传输单元 |
| **DoS** | Denial of Service | 拒绝服务攻击 |
| **DDoS** | Distributed Denial of Service | 分布式拒绝服务 |

---

## 相关链接

- [[wiki/tutorials/tutorial-udp-hole-punching]] - UDP 打洞流程详解（NAT 术语实战应用）
- [[wiki/concepts/concept-ice]] - ICE 通俗理解
- [[wiki/protocols/protocol-rfc5128]] - NAT 类型分类详解
- [[wiki/protocols/protocol-rfc5389]] - STUN 协议分析
- [[wiki/protocols/protocol-rfc5766]] - TURN 协议分析
- [[wiki/protocols/protocol-rfc8445]] - ICE 协议分析
- [[wiki/concepts/concept-protocol]] - 协议通俗理解
