---
title: RFC 8445 (ICE) 协议分析
type: protocol
tags: [webrtc, ice, rfc8445, protocol]
sources: [raw/rfcs/rfc8445.md]
created: 2026-04-21
updated: 2026-04-22
---

# RFC 8445 (ICE) 协议分析

## 一句话理解

**ICE = Interactive Connectivity Establishment**，一种 NAT 穿透协议，通过系统性测试多个候选地址对，找出能工作的直接通信路径。

---

## 语义层 (What - 说什么)

### 核心概念

| 术语 | 语义含义 |
|------|----------|
| **ICE Agent** | ICE 协议实现者（发起方或响应方） |
| **Candidate (候选)** | 可用于通信的 IP+端口地址 |
| **Candidate Pair (候选对)** | 本地候选 + 远程候选的配对 |
| **Checklist (检查清单)** | 按优先级排序的候选对列表 |
| **Valid Pair (有效对)** | 通过连接性检查的候选对 |
| **Selected Pair (选中对)** | 最终用于传输数据的候选对 |
| **Foundation** | 用于分组的字符串，相同类型/基础地址的候选共享相同 Foundation |

### 候选类型

| 类型 | 英文名 | 说明 | 获取方式 | 推荐优先级 |
|------|--------|------|----------|-----------|
| **Host** | Host Candidate | 主机物理/虚拟网卡地址 | 直接绑定 | 126 (最高) |
| **Srflx** | Server-Reflexive | NAT 分配的公网地址 | STUN Binding | 100 |
| **Prflx** | Peer-Reflexive | 对端 NAT 分配的地址 | 连接性检查发现 | 110 |
| **Relay** | Relayed | TURN 服务器中继地址 | TURN Allocate | 0 (最低) |

### 候选关系图

```
                    To Internet
                        |
                        | /------------  Relayed Address (Y:y)
                    Y:y |
                    +--------+
                    |  TURN  |
                    | Server |
                    +--------+
                        |
                        | /------------  Server-Reflexive (X1':x1')
                 X1':x1'|
                 +------------+
                 |    NAT     |
                 +------------+
                        |
                        | /------------  Local Address (X:x)
                    X:x |
                    +--------+
                    |  Agent |
                    +--------+
```

### 角色语义

| 角色 | 英文名 | 职责 |
|------|--------|------|
| **Controlling Agent** | 控制方 | 负责提名候选对 |
| **Controlled Agent** | 被控方 | 等待控制方提名 |

### 实现类型

| 类型 | 说明 |
|------|------|
| **Full Implementation** | 完整实现，执行全部 ICE 功能，包括连通性检查 |
| **Lite Implementation** | 简化实现，只用 Host 候选，不生成检查，适用于公网 IP 终端 |

---

## 语法层 (How - 怎么说)

### 候选信息交换格式

每个候选需携带的信息：

```
Candidate Information:
├── Address: IP + Port
├── Transport: 传输协议（UDP/TCP）
├── Foundation: 字符串（最多32字符）
├── Component ID: 组件ID（RTP=1, RTCP=2）
├── Priority: 32位优先级
├── Type: 候选类型
├── Related Address: 关联地址
└── Base: 基准地址

Session Information:
├── Username Fragment: 至少24位随机
├── Password: 至少128位随机
├── Lite/Full: 实现类型
└── ICE Options: 扩展选项（如 ice2）
```

#### Foundation 与 Component ID

**Candidate SDP 格式**:

```
a=candidate:<foundation> <component-id> <transport> <priority>
             <connection-address> <port> typ <candidate-type>
```

**Foundation** = 候选的"血缘标签"，标识物理路径。规则：

```
Foundation = hash(候选类型, Base IP, STUN Server, 传输协议)
```

- **同 Foundation 的候选走同一条物理路径** → 只需验证一次连通性
- 作用：减少冗余的连通性检查（4个本地 × 4个远程 = 16对 → 跳过同 Foundation 后缩减至 ~10对）

**Component-ID** = 标识候选用于哪个媒体子流：

| Component-ID | 用途 | 特征 |
|:---:|------|------|
| **1** | RTP（媒体数据） | 基础端口 |
| **2** | RTCP（控制报告） | 通常 RTP端口+1 |

- 每个 Component 有**独立的状态机**和独立的连通性检查列表
- 优先级公式中 `(256 - component_id)` 使 RTP 候选比 RTCP 候选高 1 点

**示例**：一台双网卡主机（WiFi 192.168.1.100 + 有线 10.0.0.50）的音频 RTP 候选：

```
Component 1 (RTP):
  Foundation "1": host 192.168.1.100:5000    ← WiFi 路径
  Foundation "2": host 10.0.0.50:5000        ← 有线路径
  Foundation "3": srflx 203.0.113.50:4000    ← NAT 公网路径

Component 2 (RTCP):
  Foundation "1": host 192.168.1.100:5001    ← 同上 WiFi，不同端口
  Foundation "2": host 10.0.0.50:5001
  Foundation "3": srflx 203.0.113.50:4001
```

> Foundation 回答"两个候选是否走同一条物理路径"（避免重复检查），Component-ID 回答"这个候选用于 RTP 还是 RTCP"（独立状态机），两者正交。

### 候选优先级公式

```
priority = (2^24) × type_preference +
           (2^8) × local_preference +
           (2^0) × (256 - component_id)

推荐 type_preference:
- Host: 126 (最高)
- Peer-Reflexive: 110
- Server-Reflexive: 100
- Relayed: 0 (最低)

local_preference: 0-65535，IPv4 单地址推荐 65535
component_id: 组件ID，RTP=1, RTCP=2
```

### 候选对优先级公式

```
pair_priority = 2^32 × MIN(G,D) + 2 × MAX(G,D) + (G>D?1:0)

其中:
- G = 控制方(controlling)候选的优先级
- D = 被控方(controlled)候选的优先级
```

### ICE 扩展的 STUN 属性

| 属性 | 类型值 | 作用 |
|------|--------|------|
| `PRIORITY` | 0x0024 | 声明候选优先级 |
| `USE-CANDIDATE` | 0x0025 | 提名候选对（控制方使用） |
| `ICE-CONTROLLING` | 0x8029 | 声明自己是控制方，含 tiebreaker |
| `ICE-CONTROLLED` | 0x802A | 声明自己是被控方 |

### ICE 错误码

| 错误码 | 含义 |
|--------|------|
| 487 | Role Conflict - 角色冲突，需要切换角色 |

---

## 时序层 (When - 何时说)

### ICE 完整流程

```
┌─────────────────────────────────────────────────────────────┐
│  阶段 1: 候选收集 (Candidate Gathering)                       │
├─────────────────────────────────────────────────────────────┤
│  Agent L                    Agent R                         │
│    │                           │                            │
│    ├─ 收集 Host Candidates ───▶│                            │
│    ├─ STUN Binding ──────────▶│ (获取 Server-Reflexive)     │
│    ├─ TURN Allocate ─────────▶│ (获取 Relayed)              │
│    │                           │◀─ 收集 Host Candidates     │
│    │                           │◀─ STUN Binding             │
│    │                           │◀─ TURN Allocate            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  阶段 2: 候选交换 (Candidate Exchange)                        │
├─────────────────────────────────────────────────────────────┤
│    │                           │                            │
│    ├─ Offer (候选列表) ───────▶│  ← 通过信令通道（SDP等）       │
│    │                           │                            │
│    │◀─ Answer (候选列表) ──────┤                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  阶段 3: 连接性检查 (Connectivity Checks)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    L                              R                         │
│    │                               │                        │
│    │◀════ STUN Request ══════════│  (对端发起的检查)         │
│    │════ STUN Response ══════════▶│                        │
│    │                              │                       │
│    │════ STUN Request ══════════▶│  (己方发起的检查)       │
│    │◀════ STUN Response ═════════│                        │
│                                                             │
│  这形成 4-way handshake，双方互相验证连通性                  │
│  发现 Peer-Reflexive 候选                                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  阶段 4: 提名 (Nomination)                                  │
├─────────────────────────────────────────────────────────────┤
│    │                               │                        │
│    │  (Controlling Agent 提名)      │                        │
│    │════ STUN Req + USE-CAND ────▶│                        │
│    │◀═══ STUN Response ═══════════│                        │
│    │                               │ (设置 nominated flag)   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  阶段 5: 完成 (ICE Completed)                               │
├─────────────────────────────────────────────────────────────┤
│    │                               │                        │
│    │  Selected Pair = 最终通信路径  │                        │
│    │  使用该路径传输媒体数据        │                        │
│    │  保活: Binding Indication     │                        │
└─────────────────────────────────────────────────────────────┘
```

### 候选对状态机

```
                    ┌──────────┐
                    │  Frozen  │  ← 初始状态，等待解冻
                    └────┬─────┘
                         │ unfreeze
                         ▼
                    ┌──────────┐
                    │ Waiting  │  ← 等待发送检查
                    └────┬─────┘
                         │ perform check
                         ▼
              ┌──────────┴──────────┐
              │                     │
         In-Progress           In-Progress
         (检查中)               (检查中)
              │                     │
        failure│              success│
              ▼                     ▼
         ┌──────────┐          ┌──────────┐
         │  Failed  │          │ Succeeded │ ← 有效对
         └──────────┘          └──────────┘
```

### ICE 会话状态

| 状态 | 含义 |
|------|------|
| **Running** | ICE 正在进行中 |
| **Completed** | 所有检查表都完成，找到可用路径 |
| **Failed** | 所有检查表都失败，无法建立连接 |

### 定时器

| 定时器 | 说明 | 默认值 |
|--------|------|--------|
| **Ta** | STUN/TURN 事务生成间隔 | 50ms |
| **RTO** | 重传超时 | 动态计算 |

### ICE 重启

触发条件：
- 改变数据流目的地
- 切换 Lite/Full 实现类型
- 更换密码和用户名片段

---

## 与 RFC 5245 的主要变化

| 项目 | RFC 5245 (旧) | RFC 8445 (新) |
|------|---------------|---------------|
| 提名方式 | 支持 aggressive nomination | 移除，仅支持 regular nomination |
| 优先级算法 | 相同 | 已澄清，更加清晰 |
| 检查表初始化 | 仅第一个检查表 | 所有检查表并行初始化 |
| 协议识别 | 无特殊处理 | 更容易被 Wireshark 识别 |

---

## 一句话总结

> **ICE = ==收集候选 + 交换地址 + 两两测试 + 选择最优路径**==
>
> **语法 = 候选信息(Candidate) + 候选对(Pair) + STUN扩展属性**
>
> **时序 = 收集→交换→检查(4-way)→提名→完成**

---

## 相关链接

- [[wiki/tutorials/tutorial-udp-hole-punching]] - UDP 打洞流程详解（ICE = 打洞的标准化实现）
- [[wiki/concepts/concept-protocol]] - 协议通用理解
- [[wiki/concepts/concept-ice]] - ICE 通俗理解
- [[wiki/protocols/protocol-rfc5389]] - STUN 协议分析
- [[wiki/protocols/protocol-rfc3550]] - RTP/RTCP 协议分析
- [[wiki/protocols/protocol-rfc3551]] - RTP A/V Profile
- [[raw/rfcs/rfc8445]] - RFC 8445 英文原文
- [[raw/rfcs/rfc8445_完整中英对照]] - RFC 8445 完整中英对照
- [[raw/rfcs/rfc5389]] - STUN 协议 (RFC 5389)
- [[raw/rfcs/rfc5766]] - TURN 协议 (RFC 5766)
