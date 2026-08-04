---
title: RFC 5766 (TURN) 协议分析
type: protocol
tags: [webrtc, turn, rfc5766, protocol]
sources: [raw/rfcs/rfc5766.md]
created: 2026-04-22
updated: 2026-04-23
---

# RFC 5766 (TURN) 协议分析

## 一句话理解

**TURN = Traversal Using Relays around NAT**，当直连失败时，通过中继服务器转发数据的 NAT 穿透协议。

---

## 通俗理解：TURN 与 STUN 的关系

### STUN vs TURN 对照

| 能力 | STUN | TURN |
|------|------|------|
| 发现公网地址 | ✅ | ✅ |
| 检查连通性 | ✅ | ✅ |
| 保活 | ✅ | ✅ |
| 转发数据 | ❌ | ✅ |
| 多对端通信 | ❌ | ✅ |

### Allocate 详解：什么是"分配中继地址"？

#### 一句话理解

**Allocate = 向 TURN 服务器"租用一个信箱"**

#### 快递柜类比

| 场景 | 现实类比 | TURN 术语 |
|------|----------|-----------|
| 你在小区里 | 不能直接给邻居递信 | NAT 阻止直连 |
| 快递柜地址 | 别人可以往这里投信 | **Relayed Address**（中继地址） |
| 租快递柜 | 付钱获得一个柜子编号 | **Allocate**（分配中继地址） |
| 柜子有期限 | 到期要续租或退租 | **Refresh**（刷新） |
| 授权邻居投信 | 告诉他柜子密码 | **CreatePermission**（创建权限） |

#### Allocate 流程图

```
┌─────────────────────────────────────────────────────────────┐
│  客户端                                                     │
│     │                                                      │
│     │  "我要租一个信箱"                                      │
│     ▼                                                      │
│  ┌─────────────────┐                                       │
│  │   TURN Server   │                                       │
│  │   (快递柜管理员)  │                                       │
│  └────────┬────────┘                                       │
│           │                                                │
│           │  分配一个空柜子:                                  │
│           │  198.51.100.5:30000  ← Relay Address           │
│           │                                                │
│           ▼                                                │
│     你的中继地址                                            │
└─────────────────────────────────────────────────────────────┘
```

#### Allocate 消息格式

```
Client                                          TURN Server
  │                                                 │
  │──── Allocate Request (0x0003) ───────────────▶  │  1. 请求分配
  │     REQUESTED-TRANSPORT = UDP                   │
  │                                                 │
  │◀──── 401 Unauthorized ────────────────────────  │  2. 挑战
  │     ERROR-CODE: 401                             │
  │     REALM: "example.com"                        │
  │     NONCE: "dcd98b71..."                        │
  │                                                 │
  │──── Allocate Request ────────────────────────▶  │  3. 带凭据重试
  │     REQUESTED-TRANSPORT = UDP                   │
  │     USERNAME: "user@example.com"                │
  │     REALM: "example.com"                        │
  │     NONCE: "dcd98b71..."                        │
  │     MESSAGE-INTEGRITY: HMAC(请求, password)      │
  │                                                 │
  │◀──── Success (0x0103) ────────────────────────  │  4. 分配成功
  │     XOR-RELAYED-ADDRESS: 198.51.100.5:30000     │
  │     XOR-MAPPED-ADDRESS: 203.0.113.50:4000       │
  │     LIFETIME: 600s                              │
  │                                                 │
```

#### 为什么需要 Allocate？

```
场景：两个都在 NAT 后面的用户要通信

         NAT A                    NAT B
    192.168.1.100              10.0.0.50
         │                         │
         │  × 无法直连              │  × 无法直连
         ▼                         ▼
    公网: 203.0.113.50       公网: 198.51.100.30

解决方案：都用 TURN 服务器分配中继地址

    Client A ────▶ TURN Server ◀──── Client B
         │                               │
         │  198.51.100.5:30000           │  198.51.100.5:30001
         │                               │
         ◀────────────────────────────────
              通过 TURN 中继转发数据
```

#### Allocate 返回的关键信息

| 属性 | 说明 | 示例 |
|------|------|------|
| `XOR-RELAYED-ADDRESS` | 服务器分配的中继地址 | `198.51.100.5:30000` |
| `XOR-MAPPED-ADDRESS` | 客户端的公网反射地址 | `203.0.113.50:4000` |
| `LIFETIME` | 分配有效期（秒） | `600` |

---

### TURN 扩展 STUN 的内容

#### 1. 新增方法 (Methods)

STUN 只有 **Binding** 方法，TURN 新增 6 个方法：

| 新增方法 | 用途 | 类比 |
|----------|------|------|
| **Allocate** | 申请中继地址 | "给我开一个信箱" |
| **Refresh** | 续期/删除分配 | "续租/退租" |
| **CreatePermission** | 授权对端 IP | "允许这些人往信箱投信" |
| **Send** | 发送数据给对端 | "帮我把这封信寄给某人" |
| **Data** | 接收对端数据 | "收到一封信" |
| **ChannelBind** | 绑定通道号 | "给某人分配一个短编号" |

#### 2. 新增属性 (Attributes)

| 新增属性 | 作用 |
|----------|------|
| `XOR-RELAYED-ADDRESS` | 返回服务器分配的中继地址 |
| `XOR-PEER-ADDRESS` | 指定对端的地址 |
| `DATA` | 携带要发送的应用数据 |
| `LIFETIME` | 分配/权限的存活时间 |
| `CHANNEL-NUMBER` | 高效传输用的通道号 |
| `REQUESTED-TRANSPORT` | 请求的传输协议 |
| `EVEN-PORT` | 偶数端口请求 |
| `DONT-FRAGMENT` | 不分片标志 |

#### 3. 新增概念

| 新概念 | 说明 |
|--------|------|
| **Allocation** | 服务器上的"分配"数据结构 |
| **Permission** | 权限白名单，只有授权的 IP 才能往中继地址发数据 |
| **Channel** | 用短通道号替代长地址，节省 32 字节开销 |
| **5-tuple** | 唯一标识一个 Allocation |

### 具体例子

#### STUN 场景：发现地址

```
Client ──────────▶ STUN Server
                        │
                        │  Binding Request
                        │
                        │◀───────────────
                        │
                        │  XOR-MAPPED-ADDRESS: 203.0.113.50:5000
                        │  (你的公网地址是这个)
                        │◀───────────────

结果: 客户端知道自己被 NAT 映射后的公网地址
```

#### TURN 场景：中继数据

```
Client ──────────▶ TURN Server ──────────▶ Peer
                        │                      │
                        │  Allocate Request    │
                        │                      │
                        │◀─────────────────────
                        │
                        │  XOR-RELAYED-ADDRESS: 198.51.100.5:30000
                        │  (给你分配的中继地址是这个)
                        │◀─────────────────────

                        │                      │
                        │  CreatePermission    │
                        │  (Peer IP: 203.0.113.50)
                        │                      │
                        │◀─────────────────────
                        │
                        │  Send Indication    │
                        │  (DATA: 媒体数据)    │
                        │─────────────────────▶
                        │                      │
                        │                      │  UDP Datagram

结果: 通过 TURN 服务器中继传输媒体数据
```

### 一句话总结

```
STUN = 工具箱里的"指南针"
       用来发现自己在哪

TURN = 工具箱里的"快递柜"
       用来让别人能找到你、传递数据

ICE = 工具箱里的"指挥官"
      协调使用指南针和快递柜
```

**TURN 扩展 STUN，本质上是把"查询地址"升级为"中转数据"。**

---

## 语义层 (What - 说什么)

### 核心概念

| 术语                            | 语义含义                              |
| ----------------------------- | --------------------------------- |
| **TURN Client**               | 发起中继请求的客户端                        |
| **TURN Server**               | 充当数据转发中继的服务器                      |
| **Peer**                      | TURN 客户端想要通信的对端主机                 |
| **Allocation**                | 服务器上的数据结构，包含中继地址及相关状态             |
| **Relayed Transport Address** | TURN 服务器分配的供对端发送数据的目标地址           |
| **Permission**                | 权限列表，规定哪些 IP 可以向中继地址发送数据          |
| **Channel**                   | 通道号与对端地址的绑定，用于高效传输                |
| **5-tuple**                   | (客户端IP:端口, 服务器IP:端口, 协议) 用于唯一标识分配 |
|                               |                                   |

### TURN 与 STUN 的关系

```
STUN: 发现地址（我公网 IP 是什么）
TURN: 中继数据（直连不行，用服务器转发）
```

TURN 是 STUN 的扩展，大部分 TURN 消息使用 STUN 格式。

### TURN 拓扑图

```
                          Peer A
                          Server-Reflexive    +---------+
                          Transport Address   |         |
                          192.0.2.150:32102   |         |
                              |              /|         |
        TURN                  |            / ^|  Peer A |
   Client's                  |           /  ||         |
   Host Transport            |         //   ||         |
   Address                  |        //     |+---------+
  10.1.1.2:49721          +-+  //       Peer A
           |               |N| /         Host Transport
           |   +-+         ||/          Address
           |   | |         ||        192.168.100.2:49582
+----------+|   |N|         |T|     +---------+
|          ||   | |         |+|     |         |
| TURN     ||   |A|----------|-----|  Peer B |
| Client   ||   | |^         ||     |         |
|          ||   |T||         ||     |         |
+----------+    | ||         +------+---------+
                 | ||
                 +-||
                    ||
               Client's                   Peer B
               Server-Reflexive    Relayed             Transport
               Transport Address   Transport Address   Address
               192.0.2.1:7000      192.0.2.15:50000     192.0.2.210:49191
```

---

## 语法层 (How - 怎么说)

### TURN 新增的 STUN 方法

| 方法 | 十六进制 | 名称 | 用途 |
|------|----------|------|------|
| 3 | 0x003 | **Allocate** | 创建分配，获取中继地址 |
| 9 | 0x009 | **Refresh** | 刷新/删除分配 |
| 13 | 0x00D | **CreatePermission** | 创建权限 |
| 14 | 0x00E | **Send** | 发送数据到对端 |
| 16 | 0x010 | **Data** | 从对端接收数据 |
| 17 | 0x011 | **ChannelBind** | 绑定通道 |

### TURN 新增的 STUN 属性

| 属性 | 类型值 | 作用 |
|------|--------|------|
| `CHANNEL-NUMBER` | 0x000C | 通道号 (范围 0x4000-0x4FFF) |
| `LIFETIME` | 0x000D | 分配/权限的存活时间 |
| `XOR-PEER-ADDRESS` | 0x0012 | 对端地址（用于 Send/Data） |
| `DATA` | 0x0013 | 应用数据载荷 |
| `XOR-RELAYED-ADDRESS` | 0x0016 | 中继地址（分配成功后返回） |
| `EVEN-PORT` | 0x0018 | 请求偶数端口或保留下一端口 |
| `REQUESTED-TRANSPORT` | 0x0019 | 请求的传输协议（固定 UDP） |
| `DONT-FRAGMENT` | 0x001A | 设置 IP DF 位 |
| `RESERVATION-TOKEN` | 0x0022 | 使用预保留的端口 |

### ChannelData 消息格式

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Channel Number         |            Length           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                       Data (variable)                         |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

比 Send/Data indication 节省 32 字节开销，更适合大数据传输。

### Allocation 状态数据结构

```
Allocation State:
├── Relayed Transport Address: 中继地址（服务器分配的端口）
├── 5-tuple: (客户端IP:端口, 服务器IP:端口, 协议)
├── Authentication Info: 认证信息（用户名、密码、realm、nonce）
├── Time-to-Expiry: 剩余存活时间
├── Permissions List: 权限列表
└── Channels List: 通道绑定列表
```

---

## 时序层 (When - 何时说)

### 完整通信流程

```
┌─────────────────────────────────────────────────────────────┐
│  阶段 1: 创建 Allocation                                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client                          Server                     │
│    │                                │                       │
│    │─── Allocate Request ──────────▶│                       │
│    │    (REQUESTED-TRANSPORT=UDP)   │                       │
│    │                                │                       │
│    │◀── 401 Unauthorized ──────────│                       │
│    │                                │                       │
│    │─── Allocate Request ──────────▶│                       │
│    │    (with credentials)           │                       │
│    │                                │                       │
│    │◀── Allocate Success ──────────│                       │
│    │    (XOR-RELAYED-ADDRESS)       │                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  阶段 2: 创建权限 (CreatePermission)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client                          Server                     │
│    │                                │                       │
│    │─── CreatePermission ──────────▶│                       │
│    │    (XOR-PEER-ADDRESS: Peer IP) │                       │
│    │                                │                       │
│    │◀── Success ───────────────────│                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  阶段 3: 数据传输 (Send/Data)                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Client  ──── Server  ────▶  Peer                          │
│    │                                │                        │
│    │─── Send Indication ──────────▶│                        │
│    │    (XOR-PEER-ADDRESS)          │                        │
│    │    (DATA: 应用数据)             │                        │
│    │                                │                        │
│    │                                │◀═══ UDP Datagram ═════│
│    │◀── Data Indication ───────────│                        │
│    │    (XOR-PEER-ADDRESS)          │                        │
│    │    (DATA: 应用数据)             │                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 使用 Channel 的数据传输

```
Client  ──── Server  ────▶  Peer
  │                                │
  │─── ChannelBind ──────────────▶│
  │    (Channel=0x4001)            │
  │    (Peer Address)              │
  │                                │
  │◀── Success ───────────────────│
  │                                │
  │─── ChannelData[0x4001] ──────▶│  ← 4字节头部，比 Send indication 省开销
  │    (Data)                      │
```

### 多对端通信场景

TURN 的核心优势：**一个中继地址，服务多个对端**

#### 场景：群聊（Client 与 Peer A、B、C 通信）

```
                    Internet
                       │
               ┌───────┴───────┐
               │   TURN Server │
               │  (中继服务器)  │
               └───────┬───────┘
                       │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
     Peer A          Peer B          Peer C
   (电信网络)      (联通网络)      (移动网络)
```

#### 完整通信时序

```
Client          TURN Server       Peer A        Peer B        Peer C
  │                  │               │             │             │
  │── Allocate ────▶│               │             │             │
  │◀── 198.51.100.5:30000 ────────│             │             │
  │                  │               │             │             │
  │── CreatePerm(A)─▶│               │             │             │
  │── CreatePerm(B)─▶│               │             │             │
  │── CreatePerm(C)─▶│               │             │             │
  │                  │               │             │             │
  │── Send(A) ──────▶│── UDP ──────▶│             │             │
  │◀── Data(A) ◀────│◀─ UDP ◀──────│             │             │
  │                  │               │             │             │
  │── Send(B) ──────▶│──────────────▶│             │             │
  │◀── Data(B) ◀────│◀─────────────│             │             │
  │                  │               │             │             │
  │── Send(C) ──────▶│───────────────┼────────────▶│             │
  │◀── Data(C) ◀────│◀─────────────┼─────────────│             │
```

#### 为什么 TURN 支持多对端？

| 对比 | SOCKS 代理 | TURN |
|------|-----------|------|
| 一对端 | 1 个连接 | 1 个 Allocation |
| 多对端 | 需要多个连接 | **共用 1 个 Allocation** |
| 效率 | 低 | 高 |

**核心原因**: TURN 消息中携带了对端地址 (`XOR-PEER-ADDRESS`)，服务器根据这个地址区分数据发给谁/来自谁。

#### 一句话总结

> **TURN 用一个中继地址，服务多个对端**
>
> 消息格式里的 `XOR-PEER-ADDRESS` 就是"收件人姓名"，告诉服务器这封信该转给谁。

### Allocation 生命周期

```
                    分配创建
                       │
                       ▼
    ┌──────────────────────────────────────────┐
    │          Active Allocation               │
    │                                          │
    │  定期 Refresh (建议提前 1 分钟)            │
    │       或                                 │
    │  Refresh with LIFETIME=0 (删除)          │
    │                                          │
    └──────────────────────────────────────────┘
                       │
                       ▼
                   过期删除

默认 Lifetime: 600 秒 (10 分钟)
```

### Permission 生命周期

```
默认 Permission Lifetime: 300 秒 (5 分钟)
注意: 很多 NAT 的 UDP 绑定过期更快，需要更频繁的保活
```

---

## 传输协议支持

| 客户端 → 服务器 | 服务器 → 对端 |
|-----------------|---------------|
| UDP | UDP |
| TCP | UDP |
| TLS over TCP | UDP |

---

## 定时参数总结

| 参数 | 默认值 | 说明 |
|------|--------|------|
| **Allocation Lifetime** | 600 秒 | 分配默认存活时间 |
| **Permission Lifetime** | 300 秒 | 权限默认存活时间 |
| **Channel Binding Lifetime** | 600 秒 | 通道绑定存活时间 |

---

## 与 ICE 的配合

```
ICE 流程:
1. 收集候选 → 包括 Relayed Candidate (通过 TURN 获取)
2. 连接性检查 → 优先直连，失败则用 TURN 中继
3. 提名 → 控制方选择最优路径
```

TURN 专为 ICE 设计，但也可以单独使用。

---

## 一句话总结

> **TURN = Allocate（获取中继地址）+ Permission（授权对端）+ Send/Data（传输数据）**
>
> **语法 = STUN 扩展方法 + TURN 属性 + ChannelData 高效格式**
>
> **时序 = 分配→授权→传输→刷新维护**

---

## 相关链接

- [[wiki/tutorials/tutorial-udp-hole-punching]] - UDP 打洞流程详解（TURN = 打洞失败时的中继备选）
- [[wiki/concepts/concept-protocol]] - 协议通用理解
- [[wiki/protocols/protocol-rfc5389]] - STUN 协议分析
- [[wiki/protocols/protocol-rfc8445]] - ICE 协议分析
- [[raw/rfcs/rfc5766]] - RFC 5766 英文原文
- [[raw/rfcs/rfc5766_完整中英对照]] - RFC 5766 完整中英对照
- [[raw/rfcs/rfc5389]] - STUN 协议 (RFC 5389)
- [[raw/rfcs/rfc8445]] - ICE 协议 (RFC 8445)
