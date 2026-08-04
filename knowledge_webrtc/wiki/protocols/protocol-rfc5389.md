---
title: RFC 5389 (STUN) 协议分析
type: protocol
tags: [webrtc, stun, rfc5389, protocol]
sources: [raw/rfcs/rfc5389.md]
created: 2026-04-22
updated: 2026-04-22
---

# RFC 5389 (STUN) 协议分析

## 一句话理解

**STUN 是 NAT 穿透工具协议，不是完整解决方案。** 它用于==发现 NAT 分配的公网地址、检查连接性、维持 NAT 绑定。==

---

## 语义层 (What - 说什么)

### 核心功能

| 功能 | 说明 |
|------|------|
| **地址发现** | 客户端想知道自己在 NAT 公网侧的地址是什么 |
| **连接性检查** | 两个端点之间能否直接通信 |
| **保活** | 维持 NAT 绑定不过期 |

### 关键术语

| 术语                           | 语义含义               |
| ---------------------------- | ------------------ |
| **反射地址 (Reflexive Address)** | 客户端从外部视角看到的自己的地址   |
| **绑定 (Binding)**             | NAT 为内部地址分配的公网映射关系 |
| **请求/响应事务**                  | 客户端发请求，服务器回响应      |
| **指示 (Indication)**          | 无需响应的消息（如保活心跳）     |

#### 深入理解：什么是 Binding？

**Binding = NAT 映射表中的那条记录**

```
┌─────────────────────────────────────────────────────────────┐
│                    NAT 设备                                 │
│                                                             │
│   内部: 192.168.1.100:5000  ←────── 绑定 (Binding) ──────→  │
│                                    公网: 203.0.113.50:4000  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Binding Request = "查询绑定"**

当你发送 Binding Request 到 STUN 服务器时，你是在问：

> "请告诉我，我现在在公网上是什么地址？"

服务器从**源 IP:Port**（经过 NAT 转换后的）返回给你，就是你的当前绑定。

**Binding Response = "返回绑定地址"**

响应中的 `XOR-MAPPED-ADDRESS` 就是你当前的 NAT 绑定地址（公网可见的地址）。

> **一句话总结：Binding = NAT 映射表中的那条记录**
>
> - Binding Request = "查询我当前的 NAT 绑定"
> - Binding Response = "你的绑定是 X.X.X.X:Port"

### 认证语义

| 凭证类型 | 说明 |
|----------|------|
| **短期凭证** | 临时用户名/密码，用于会话期间（如 ICE） |
| **长期凭证** | 持久的用户名/密码，用于长期服务 |

---

## 语法层 (How - 怎么说)

### 消息格式

```
┌─────────────────────────────────────────────────────────┐
│  STUN 头 (20 字节)                                       │
├─────────────────────────────────────────────────────────┤
│  | 类型 (16bit) | 长度 (16bit) |   Magic Cookie (32bit)  │
│  |              Transaction ID (96bit)                  │
├─────────────────────────────────────────────────────────┤
│  属性 1 (TLV)                                            │
├─────────────────────────────────────────────────────────┤
│  属性 2 (TLV)                                            │
├─────────────────────────────────────────────────────────┤
│  ...                                                    │
└─────────────────────────────────────────────────────────┘
```

### 头部结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|0 0|     STUN Message Type     |         Message Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Magic Cookie (0x2112A442)            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                     Transaction ID (96 bits)                |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**关键字段：**
- **前 2 位必须为 0**: 用于区分 STUN 和其他协议
- **Magic Cookie (0x2112A442)**: 固定值，用于协议识别和 XOR 混淆
- **Transaction ID**: 96 位随机数，用于关联请求和响应

### 消息类型编码

```
STUN 消息类型字段 (16-bit)，M（Method）与 C（Class）位交织排列:

  bit→ 15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
       [0][0]M11M10 M9 M8 M7 C1 M6 M5 M4 C0 M3 M2 M1 M0
        前2bit固定为0
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       | 0 | 0 | 0 | 0 | 0 | 0 | 0 |C1 | 0 | 0 | 0 |C0 | 0 | 0 | 0 | 1 |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
        方法 Method (12 bit)                类别 Class (2 bit)
        0b000000000001 = Binding            0b00 = 请求 (Request)
                                            0b01 = 指示 (Indication)
                                            0b10 = 成功响应 (Success)
                                            0b11 = 错误响应 (Error)

  M/C 位交织后的完整 Message Type 值:

  Request     0x0001 = 0b 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1  C=0b00
  Indication  0x0011 = 0b 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 1  C=0b01
  Success     0x0101 = 0b 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0 1  C=0b10
  Error       0x0111 = 0b 0 0 0 0 0 0 0 1 0 0 0 1 0 0 0 1  C=0b11

  M=0b000000000001(Binding)不变，仅 C1(bit8) 和 C0(bit4) 在变化。
```

### 属性 (Attributes)

**TLV 格式：**

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Type                  |            Length           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Value (variable)                ...|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**主要属性：**

| 属性 | 类型值 | 作用 |
|------|--------|------|
| `MAPPED-ADDRESS` | 0x0001 | 返回反射地址（RFC3489 兼容） |
| `XOR-MAPPED-ADDRESS` | 0x0020 | 返回反射地址（XOR 混淆，推荐） |
| `USERNAME` | 0x0006 | 用户名，用于认证 |
| `MESSAGE-INTEGRITY` | 0x0008 | HMAC-SHA1 消息完整性 |
| `FINGERPRINT` | 0x8028 | 用于协议区分（可选） |
| `ERROR-CODE` | 0x0009 | 错误码和原因短语 |
| `REALM` | 0x0014 | 认证领域 |
| `NONCE` | 0x0015 | 防重放随机数 |
| `UNKNOWN-ATTRIBUTES` | 0x000A | 未知属性列表 |
| `SOFTWARE` | 0x8022 | 服务器软件描述 |

**属性分类：**
- **Comprehension-Required (0x0000-0x7FFF)**: 必须理解，不认识则丢弃消息
- **Comprehension-Optional (0x8000-0xFFFF)**: 可忽略

### XOR 地址编码详解

#### 为什么需要 XOR 编码？

某些 NAT/防火墙有 **ALG（应用层网关）** 功能，会检查 UDP 包中的 IP 地址并尝试"帮忙"转换：

```
问题: NAT/防火墙误认为 STUN 包中的 IP 是"普通地址"
       │
       ▼
┌─────────────────────────────────────┐
│  ALG 尝试修改包内容                  │
│  结果：反而把 XOR-MAPPED-ADDRESS     │
│        或 MESSAGE-INTEGRITY 改坏了  │
│        导致 STUN 失败               │
└─────────────────────────────────────┘
```

#### XOR 编码原理

XOR = **异或（Exclusive OR）** 运算

```
XOR 运算规则:
0 XOR 0 = 0
0 XOR 1 = 1
1 XOR 0 = 1
1 XOR 1 = 0
```

**STUN 使用 Magic Cookie (0x2112A442) 作为 XOR 密钥：**

```
原始地址:     203.0.113.50  (十进制)
             = 0xCB007132   (十六进制)

Magic Cookie: 0x2112A442   (STUN 固定值)

XOR 编码:    0xCB007132 XOR 0x2112A442
             = 0xEB12C372   (看起来像随机数)

服务器还原:   0xEB12C372 XOR 0x2112A442
             = 0xCB007132 ✓
```

#### MAPPED-ADDRESS vs XOR-MAPPED-ADDRESS

| 属性 | 编码方式 | 二进制特征 | 被 NAT 误改风险 |
|------|----------|------------|----------------|
| `MAPPED-ADDRESS` | 明文 | 有明显 IP 模式 (192.168.x.x, 10.x.x.x) | 高 |
| `XOR-MAPPED-ADDRESS` | XOR 混淆 | 看起来像随机数 | 低 |

#### TURN 中的 XOR 地址

TURN 继承了 STUN 的 XOR 编码机制，所有地址属性都用 XOR：

| 属性 | 作用 |
|------|------|
| `XOR-MAPPED-ADDRESS` | 客户端的反射地址 |
| `XOR-RELAYED-ADDRESS` | TURN 分配的**中继地址** |
| `XOR-PEER-ADDRESS` | 对端的地址 |

#### 一句话总结

> **XOR = 混淆编码，把地址变成"乱码"**
>
> 防止 NAT/防火墙误认为这是"普通 IP 地址"而尝试修改它。

---

## 时序层 (When - 何时说)

> STUN Binding 事务在 WebRTC 中有两种截然不同的使用场景：发向 STUN 服务器的"候选收集"，和发向对端的"连通性测试"。同一套协议，完全不同的目的。

### 场景一：收集候选 (Candidate Gathering)

客户端向独立的 STUN/TURN 服务器发 Binding Request，获取自己的公网反射地址（server-reflexive candidate）：

```
Client (10.0.0.1:4000)                STUN Server (1.2.3.4:3478)
   │                                            │
   │  ──── Binding Request ──────────────────▶  │  ← 无认证 (或不强制)
   │       (属性: 空或仅 FINGERPRINT)            │
   │                                            │
   │  ◀──── Binding Success ──────────────────  │
   │       (属性: XOR-MAPPED-ADDRESS            │
   │        = 1.2.3.4:60000)                    │  ← "你的公网地址"
   │                                            │
```

| 特征 | 值 |
|------|-----|
| **目标** | 独立的 STUN/TURN 服务器 |
| **目的** | 获取反射地址 (host→srflx)、中继地址 (host→relay) |
| **认证** | STUN 通常不强制，TURN 需要长期凭证 |
| **关键属性** | 请求: `FINGERPRINT`(可选)；响应: `XOR-MAPPED-ADDRESS` |
| **ICE 产物** | srflx candidate、relay candidate |

### 场景二：连通性测试 (Connectivity Checks)

对端之间互相发 Binding Request，验证网络路径是否可达：

```
Alice (controlling)                           Bob (controlled)
   │                                            │
   │  ──── Binding Request ──────────────────▶  │  ← 需要 MESSAGE-INTEGRITY
   │       (USERNAME + MESSAGE-INTEGRITY        │     (短期凭证, SDP协商的密码)
   │        + PRIORITY                          │
   │        + ICE-CONTROLLING)                  │     ← Alice 声明自己是控制方
   │                                            │
   │  ◀──── Binding Success ──────────────────  │
   │       (USERNAME + MESSAGE-INTEGRITY        │
   │        + XOR-MAPPED-ADDRESS)               │     ← Bob 的反射地址
   │                                            │
   │  ──── Binding Request ──────────────────▶  │  ← 再次检查 + 提名
   │       (USERNAME + MESSAGE-INTEGRITY        │
   │        + USE-CANDIDATE)                    │     ← "这个对我要了"
   │                                            │
   │  ◀──── Binding Success ──────────────────  │
   │                                            │
   │  ===== 提名完成, 媒体可以传输 ============   │
```

| 特征         | 值                                                               |
| ---------- | --------------------------------------------------------------- |
| **目标**     | 对端 (peer)                                                       |
| **目的**     | 验证对端可达性 + 提名最优路径                                                |
| **认证**     | 必须（短期凭证，SDP 协商的 `ice-ufrag` + `ice-pwd`）                        |
| **关键属性**   | `PRIORITY`, `USE-CANDIDATE`, `ICE-CONTROLLING`/`ICE-CONTROLLED` |
| **ICE 产物** | valid pair、nominated pair                                       |
|            |                                                                 |
**为什么必须是两次、不能一次？**

因为连通性检查的结果可能是**多个候选对都通过了**（Host↔Host、Host↔Srflx、Relay↔Relay 全通）。控制方需要先知道哪些路径有效，再从有效的中挑最优的提名。如果把验证和提名合并成一次（旧 RFC 5245 的 Aggressive Nomination），等于还没验证通不通就直接提名——可能提名的路径根本不可达。RFC 8445 因此移除了激进模式，只保留 Regular Nomination。

### 两种场景对比

```
                    收集候选                      连通性测试
                  ───────────                  ────────────
    方向:     Client → STUN/TURN            Alice ⟷ Bob
    目的:     发现"我是谁"                    证明"我能通"
    往返:     1 RTT                         4-way handshake
    认证:     STUN不需, TURN需长期            必须短期凭证
    产物:     srflx/relay candidate          valid pair → nominated pair
```

### UDP 重传机制

```
时间线:  0 ── 500ms ── 1500ms ── 3500ms ── 7500ms ── 15500ms ── 31500ms
         │    重传1   重传2    重传3    重传4     重传5     重传6

默认参数:
- RTO 初始值: 500ms (固定线路推荐)
- 重传次数 Rc: 7 次
- 超时倍数 Rm: 16
- 总超时: 约 39.5 秒
```

### 认证机制

RFC 5389 §10 定义两种认证机制，TURN (RFC 5766) 直接复用：

| | 短期凭证 (Short-Term) | 长期凭证 (Long-Term) |
|---|---|---|
| **类比** | HTTP Basic Auth | HTTP Digest Auth |
| **凭证来源** | STUN 之前通过其他协议获取（如 ICE 信令） | 预先为用户配置，类似"登录密码" |
| **有效期** | 时间限定（如 5 分钟，或会话级） | 持久有效，直至用户退订或修改 |
| **防重放** | 靠时效性（凭证过期即作废） | 靠 NONCE 质询-响应 |
| **流程图** | 请求直接带凭据 | 先发无认证请求 → 401 挑战 → 带凭据重试 |
| **保护 Indication** | ✓ 可以 | ✗ 不能（Indication 不能响应挑战） |
| **ICE 中使用** | ✓（两个端点用信令协商 ufrag/pwd） | — |
| **TURN 中使用** | — | ✓（几乎全部 TURN 部署） |

#### 短期凭证 (Short-Term Credential)

无挑战流程，请求直接附带 USERNAME + MESSAGE-INTEGRITY：

```
Client                                      Server
   │                                            │
   │  ──── Request ──────────────────────────▶  │
   │       (USERNAME + MESSAGE-INTEGRITY)       │  ← 直接认证,不先发试探包
   │                                            │
   │  ◀──── Success / Error ──────────────────  │
   │                                            │
```

> ICE 是最典型的短期凭证用法：SDP 中 `ice-ufrag:xxx` 和 `ice-pwd:yyy` 就是短期凭证的 username/password。

#### 长期凭证 (Long-Term Credential)

经典质询-响应（challenge-response），客户端先"裸发"请求，服务器拒绝并下发挑战：

```
Client                                      Server
   │                                            │
   │  ──── Request (无认证) ─────────────────▶  │  ← 1. 初始请求
   │                                            │
   │  ◀──── 401 Unauthorized ────────────────  │  ← 2. 挑战
   │       (REALM + NONCE)                      │     (没有这步=短期凭证)
   │                                            │
   │  ──── Request ─────────────────────────▶  │  ← 3. 带凭据重试
   │       (USERNAME + REALM + NONCE            │
   │        + MESSAGE-INTEGRITY)               │
   │                                            │
   │  ◀──── Success ──────────────────────────  │  ← 4. 成功
   │                                            │
```

> TURN 的 Allocate 请求就是典型场景：客户端第一次发 Allocate 无认证 → 服务器回 401 → 客户端带凭据重试。

后续请求可复用之前的 NONCE，直到 NONCE 过期 → 服务器回 438 (Stale Nonce) → 客户端用新 NONCE 重试。

#### 为什么需要两种？

STUN 必须服务两类截然不同的场景，对认证有相反的要求：

| | P2P (ICE) → 短期 | C/S (TURN) → 长期 |
|---|---|---|
| **谁用** | Alice ⟷ Bob 对等端 | 客户端 → TURN 服务器 |
| **凭据怎么来** | 信令通道分发（SDP） | 提前注册配置 |
| **会话多久** | 分钟级 | 月/年级 |
| **核心约束** | 初始化越快越好 | 密码需长期安全共享 |

**P2P 若用长期** → 多 1 RTT（401 挑战），且对等端没有"服务器"来管理 REALM/NONCE。

**C/S 若用短期** → 无法提前给用户配置密码；长期有效的密码不质询则防重放太弱。

> **一句话**：短期凭证用时效性换效率（适合 ICE），长期凭证用 1 RTT 换长期安全（适合 TURN）。STUN 让**用法决定认证方式**，而非强制统一。

### 错误处理时序

| 错误码范围 | 行为 |
|-----------|------|
| 300-399 | 尝试备用服务器 (ALTERNATE-SERVER) |
| 400-499 | 事务失败（420 需返回 UNKNOWN-ATTRIBUTES）|
| 500-599 | 可重试（需限制次数）|

---

## 协议状态机

```
                    ┌──────────────┐
                    │    IDLE      │
                    └──────┬───────┘
                           │ 发送请求
                           ▼
                    ┌──────────────┐
              ┌─────│   SENT       │─────┐
              │     └──────────────┘     │
              │         │                │ 收到响应
              │  超时重传                  ▼
              │         │         ┌──────────────┐
              │         └────────▶│  COMPLETED   │
              │                   └──────────────┘
              │ 超限失败
              ▼
        ┌──────────────┐
        │   FAILED     │
        └──────────────┘
```

---

## 端口配置

| 用途 | 端口 |
|------|------|
| STUN (UDP/TCP) | 3478 |
| STUN over TLS | 5349 |

---

## 与 RFC 3489 的主要区别

| 项目 | RFC 3489 (旧) | RFC 5389 (新) |
|------|---------------|---------------|
| 定位 | 完整 NAT 穿透解决方案 | NAT 穿透工具 |
| 传输层 | 仅 UDP | UDP + TCP + TLS |
| Transaction ID | 128 位 | 96 位 (32 位给 Magic Cookie) |
| 地址编码 | MAPPED-ADDRESS | XOR-MAPPED-ADDRESS (推荐) |
| 协议区分 | 无 | Magic Cookie + 前两位为 0 |

---

## 一句话总结

> STUN = **发现公网地址** + **检查连接** + **维持绑定**
>
> 语法 = **20字节头** + **TLV属性列表**
>
> 时序 = **请求/响应** + **重传机制** + **认证握手**

---

## 相关链接

- [[wiki/tutorials/tutorial-udp-hole-punching]] - UDP 打洞流程详解（STUN = 打洞前的地址发现）
- [[wiki/concepts/concept-protocol]] - 协议通用理解
- [[wiki/concepts/concept-ice]] - ICE 协议
- [[wiki/protocols/protocol-rfc3550]] - RTP/RTCP 协议分析
- [[wiki/protocols/protocol-rfc3551]] - RTP A/V Profile
- [[raw/rfcs/rfc5389_完整中英对照]] - RFC 5389 完整中英对照
- [[raw/rfcs/rfc8445]] - ICE 协议 (RFC 8445)
- [[raw/rfcs/rfc7350]] - DTLS as Transport for STUN (RFC 7350)
