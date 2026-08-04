---
title: libnice 各协议层错误/异常处理完整分析
type: analysis
tags: [libnice, webrtc, ice, stun, turn, rtp, rtcp, error-handling, glib]
sources: [rfc8445, rfc5389, rfc5766, rfc3550]
created: 2026-05-28
updated: 2026-05-28
---

# libnice 各协议层错误/异常处理完整分析

> libnice 是 GStreamer/Janus 等 WebRTC 项目底层使用的 ICE 库（C/GLib），实现了 RFC 8445/5245。它的异常处理秉持 GObject 风格：底层（Socket/STUN/TURN）静默重试 + 内部状态机驱动，上层（ICE Component）通过 GObject 信号通知用户代码，顶层（RTP/RTCP）完全不处理，交由用户自行管理。

---

## 架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                    libnice 错误处理分层                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   L4: RTP/RTCP 层  ── 不处理（用户自管丢包/乱序/jitter）           │
│        ▲ 信号: 无                                                 │
│   L3: ICE 层       ── Component 状态机 + "component-state-changed"│
│        ▲ 信号: FAILED / DISCONNECTED / READY                     │
│   L2: TURN 层      ── discovery.c 静默重试/跳过                    │
│        ▲ 信号: 无（通过候选收集结果间接体现）                       │
│   L1: STUN 层      ── conncheck.c 内部分支（StunError 21 个枚举）  │
│        ▲ 信号: 无（通过候选对 Failed 间接影响）                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

核心哲学：**libnice 把协议层的错误尽可能内部消化掉，只把"连接状态"这一终极结论暴露给用户**。

---

## 一、STUN 层：`StunError` 枚举（21 个错误码）

定义在 `stun/stunmessage.h`，libnice 定义了比 RFC 5389 + RFC 5766 + RFC 8445 更多的错误码，涵盖了一些过渡性草案（TURN-04, TURN-IPv6-05）。

### 1.1 核心错误码（当前标准）

| 枚举常量 | 数值 | 来源 | libnice 内部处理 |
|----------|------|------|-----------------|
| `STUN_ERROR_TRY_ALTERNATE` | **300** | RFC 5389 | 尝试备用服务器（conncheck 中有 try-alternate 修复） |
| `STUN_ERROR_BAD_REQUEST` | **400** | RFC 5389 | 候选对 → Failed |
| `STUN_ERROR_UNAUTHORIZED` | **401** | RFC 5389 | 候选收集阶段 → 添加凭据后重试 |
| `STUN_ERROR_UNKNOWN_ATTRIBUTE` | **420** | RFC 5389 | 候选对 → Failed |
| `STUN_ERROR_ALLOCATION_MISMATCH` | **437** | TURN-12 | 重新 Allocate |
| `STUN_ERROR_STALE_NONCE` | **438** | RFC 5389 | 更新 NONCE 后重试 |
| `STUN_ERROR_WRONG_CREDENTIALS` | **441** | TURN-12 | 终端错误，不重试 |
| `STUN_ERROR_UNSUPPORTED_TRANSPORT` | **442** | TURN-12 | 换用 UDP |
| `STUN_ERROR_ALLOCATION_QUOTA_REACHED` | **486** | TURN-12 | 释放旧分配后重试 |
| **`STUN_ERROR_ROLE_CONFLICT`** | **487** | ICE-19 | **自动角色切换 + 重新入队** |
| `STUN_ERROR_SERVER_ERROR` | **500** | RFC 5389 | 延迟重试 |
| `STUN_ERROR_INSUFFICIENT_CAPACITY` | **508** | TURN-12 | 延迟重试或换备用服务器 |

### 1.2 历史草案错误码（仍定义但少用）

| 枚举常量 | 数值 | 来源 |
|----------|------|------|
| `STUN_ERROR_ACT_DST_ALREADY` | **439** | TURN-04 |
| `STUN_ERROR_UNSUPPORTED_FAMILY` | **440** | TURN-IPv6-05 |
| `STUN_ERROR_INVALID_IP` | **443** | TURN-04 |
| `STUN_ERROR_INVALID_PORT` | **444** | TURN-04 |
| `STUN_ERROR_OP_TCP_ONLY` | **445** | TURN-04 |
| `STUN_ERROR_CONN_ALREADY` | **446** | TURN-04 |
| `STUN_ERROR_SERVER_CAPACITY` | **507** | TURN-04 |

### 1.3 核心 API

```c
// 构建错误响应消息（自动复制原请求的 method + TID，添加 ERROR-CODE 属性）
StunMessageReturnCode stun_message_append_error(StunMessage *msg, StunError code);

// 错误码 → 人类可读字符串
const char *stun_strerror(StunError code);

// 创建一个 STUN_ERROR 类别的响应消息
StunMessageReturnCode stun_agent_build_error(
    StunAgent *agent, StunMessage *msg,
    StunMessage *request, StunError error_code);
```

---

## 二、ICE 连接性检查层：`conncheck.c` 的异常处理

libnice 的连接性检查实现 (`agent/conncheck.c`) 是最核心的错误处理逻辑所在，对应 RFC 8445 Section 7。

### 2.1 超时与重传机制

```
STUN 事务超时 → Rc 次重传后放弃 → candidate_check_pair_fail() → 候选对 → Failed

RTO 计算（RFC 8445 Section 14）:
  rto = agent->timer_ta × waiting_and_in_progress
  rto = max(rto, 100ms)

可配置属性:
  timer_ta: 重传超时基数（默认 50ms）
  Rc:       最大重传次数（默认 7）
```

**commit `d516fca1`** 修复了一个关键问题：双方同时 recheck 同一个 pair 时，如果网络 RTT 超过初始 RTO，会产生 ping-pong 死循环：

```
问题: 同时 recheck → 双方都超时 → 都重新发请求 → 又超时 → 死循环

修复: recheck 时不重置定时器 → 指数退避使超时逐渐超过 RTT → 一方先收到响应
```

### 2.2 响应处理的核心三分支

`priv_map_reply_to_conn_check_request()` 的返回值决定处理路径：

| 返回值 | 含义 | libnice 处理 |
|--------|------|-------------|
| `0` (成功) | 找到匹配的请求 | 候选对 → CONNECTED，触发 peer-reflexive 发现 |
| `ECONNRESET` | **角色冲突 (487)** | 重启连接性检查，自动切换角色 |
| 其他 (非 `EAGAIN`) | **STUN 错误** | 候选对 → FAILED，释放 STUN 上下文 |

```c
if (res == 0) {
    // → CONNECTED: 候选对验证成功
    priv_update_check_list_state_for_component(agent, stream, component);
} else if (res == ECONNRESET) {
    // → ROLE CONFLICT: 切换角色，重新入队 Waiting
    g_debug("conncheck %p ROLE CONFLICT, restarting", p);
} else if (res != EAGAIN) {
    // → FAILED: 标记失败，释放资源
    g_debug("conncheck %p FAILED.", p);
    p->stun_ctx = NULL;
}
```

### 2.3 候选对 FAILED 的连锁反应

```
候选对 Failed
    │
    ├─→ priv_update_check_list_state_for_ready()
    │     │
    │     ├─ 所有 pair 处于终端状态 (Succeeded/Discovered/Failed)
    │     │   AND 没有 in-progress 的 pair
    │     │   AND 没有每个组件都有 valid pair
    │     │       │
    │     │       └─→ Component → FAILED (带 grace delay)
    │     │
    │     └─ 每个组件都有 valid pair
    │           └─→ Component → READY
    │
    └─→ 若被提名 pair 失败 (nominated pair fails recheck)
          └─→ 从 valid list 移除 → Checklist → Failed
```

**grace delay** 是 libnice 特有的优化：不立即声明 FAILED，而是等待一小段时间，防止因网络抖动导致的 transient 状态震荡（connecting → failed → connecting）。

### 2.4 孤儿响应静默丢弃

对于找不到对应请求的有效 STUN 响应（通常是重传包在原始请求已收到回复后才到达）：

> *"Drop valid STUN for which we can't find a request — most likely caused by a retransmission received after the initial request already had a reply."*

直接静默丢弃，不作任何错误处理。

### 2.5 候选对状态机

```
                    ┌──────────┐
                    │  FROZEN  │  ← 等待其他 pair 先成功
                    └────┬─────┘
                         │ unfreeze
                         ▼
                    ┌──────────┐
                    │ WAITING  │
                    └────┬─────┘
                         │ 触发检查
                         ▼
                    ┌──────────┐
         failure    │IN_PROGRESS│  success
         ┌──────────│          │──────────┐
         ▼          └──────────┘          ▼
    ┌──────────┐                    ┌──────────┐
    │  FAILED  │                    │SUCCEEDED │
    └──────────┘                    └────┬─────┘
                                        │ 提名
                                        ▼
                                   ┌──────────┐
                                   │ NOMINATED│
                                   └──────────┘
```

### 2.6 Trickle ICE 模式下的特殊行为

当 agent 设置了 `"ice-trickle": TRUE`：

- 组件**无限等待**更多 remote candidate（不会自动 FAILED）
- 必须用户显式调用 `nice_agent_peer_candidate_gathering_done()` 后，才可能进入 FAILED
- 设计理由：防止在有更多 candidate 还在路上时就过早放弃

---

## 三、TURN 候选收集层：`discovery.c` 的异常处理

### 3.1 可恢复 vs 不可恢复

候选收集由 `Ta` 定时器驱动，与 RFC 一致：

```
每个 Ta 周期:
  ├── 可恢复错误 (401, 438) → 修正认证信息后重试
  └── 不可恢复错误        → 跳过该服务器，不生成候选
```

### 3.2 TURN 认证流程

```
Client                                      TURN Server
  │── Allocate (无认证) ───────────────────▶│
  │◀── 401 Unauthorized ───────────────────│  ← REALM + NONCE + ERROR-CODE
  │── Allocate (带 MESSAGE-INTEGRITY) ────▶│
  │◀── Success ───────────────────────────│  ← 分配成功
```

### 3.3 特定 TURN 错误处理

| 错误码 | 触发场景 | libnice 处理 |
|--------|---------|-------------|
| **300** Try Alternate | 服务器重定向 | 尝试备用服务器地址 |
| **401** Unauthorized | 缺少认证 | 添加 MESSAGE-INTEGRITY 后重试 |
| **403** Forbidden | 策略拒绝 | 跳过该服务器，不生成 relay 候选 |
| **437** Allocation Mismatch | NAT rebinding | 重新发起 Allocate |
| **441** Wrong Credentials | 凭据错误 | **终端错误**，不重试 |
| **442** Unsupported Transport | 不支持请求的传输协议 | 换用 UDP |
| **486** Allocation Quota | 分配数达上限 | 释放旧分配后重试 |
| **508** Insufficient Capacity | 服务器资源不足 | 延迟重试或切换备用服务器 |

---

## 四、RTP/RTCP 媒体层：libnice 的独特处理

### 4.1 多路分解：STUN vs RTP/RTCP

libnice 通过检查 UDP 报文的**前 2 个 bit** 来区分包类型：

```c
if ((buf[0] & 0xc0) == 0x80) {
    /* 前 2 bit = 10 → RTP v2 → 交给用户回调 */
} else {
    /* 前 2 bit = 00 → STUN → 内部处理，永远不暴露给用户 */
}
```

| 包类型 | 前 2 bit | 处理路径 |
|--------|----------|---------|
| STUN | `00` | libnice 内部消费，**永不到达用户代码** |
| RTP v2 | `10` | 送达用户回调 (`nice_agent_recv()`) |
| RTCP | `10` | 同上（RTCP 也是 RTP v2 系列格式） |

### 4.2 RTP/RTCP 不内置任何错误处理

libnice **不处理** RTP/RTCP 协议层的错误：

| 场景 | 谁处理 |
|------|--------|
| 丢包 | 用户代码（或上层如 GStreamer webrtcbin） |
| 乱序 | 用户代码 |
| Jitter | 用户代码（通常 RTCP SR/RR 报告） |
| SRTP 解密失败 | 用户代码 |
| RTCP 反馈 (NACK, PLI, FIR) | 应用层 WebRTC 引擎 |

libnice 只负责**建立 P2P 连接**（打通 NAT），连接建立后是透明管道。

### 4.3 关键要求：接收 I/O 必须持续轮询

> *"Any STUN packets received will not be added to messages; instead, they'll be passed for processing to NiceAgent itself. Since NiceAgent does not poll for messages on its own, it's therefore essential to keep calling this function for ICE connection establishment to work."*

如果不持续调用 `nice_agent_recv_messages()` 或注册 I/O callback（`nice_agent_attach_recv()`），ICE 连接性检查将**永远不会完成**。

### 4.4 可靠模式 vs 非可靠模式

| 模式 | API | `recv()` 行为 |
|------|-----|-------------|
| 非可靠 | `nice_agent_new()` | 返回可用的任意长度数据 |
| 可靠 | `nice_agent_new_reliable()` | 阻塞直到填满整个请求的缓冲区 |

可靠模式下，缓冲区未满就一直阻塞 —— 曾导致数据看似"丢失"但实际在缓冲区中未返回。

---

## 五、组件状态机：用户可见的错误通知

### 5.1 `NiceComponentState` 枚举

| 状态 | 含义 | 触发条件 |
|------|------|---------|
| `NICE_COMPONENT_STATE_DISCONNECTED` | 初始状态 | 尚未开始或连接已断开 |
| `NICE_COMPONENT_STATE_GATHERING` | 正在收集候选 | STUN/TURN 获得 srflx/relay 候选（**host 候选不触发此状态**） |
| `NICE_COMPONENT_STATE_CONNECTING` | 正在检查连通性 | 连接性检查进行中 |
| `NICE_COMPONENT_STATE_CONNECTED` | 已连接（有 valid pair） | 至少一个候选对验证成功 |
| `NICE_COMPONENT_STATE_READY` | 就绪（已提名） | ICE 检查完成，nominated pair 已确定，可开始媒体传输 |
| `NICE_COMPONENT_STATE_FAILED` | 失败 | 所有候选对都失败，无法找到可用路径 |

### 5.2 `"component-state-changed"` 信号

这是用户代码最主要的错误/状态接收点：

```c
static void on_component_state_changed(NiceAgent *agent,
    guint stream_id, guint component_id,
    guint state, gpointer user_data)
{
    switch (state) {
    case NICE_COMPONENT_STATE_FAILED:
        // ICE 完全失败 → 可选：ICE Restart、通知用户、fallback
        g_warning("Stream %u component %u FAILED", stream_id, component_id);
        break;
    case NICE_COMPONENT_STATE_DISCONNECTED:
        // 连接断开 → 等待重连或 ICE Restart
        break;
    case NICE_COMPONENT_STATE_READY:
        // 连接就绪 → 开始发送媒体
        nice_agent_send(agent, stream_id, component_id, ...);
        break;
    }
}
```

### 5.3 提名成功信号

```c
// 推荐使用 -full 变体（自 libnice 0.1.8 起）
g_signal_connect(agent, "new-selected-pair-full",
    G_CALLBACK(on_new_selected_pair_full), NULL);

static void on_new_selected_pair_full(NiceAgent *agent,
    guint stream_id, guint component_id,
    NiceCandidate *lcandidate, NiceCandidate *rcandidate,
    gpointer user_data)
{
    // 最佳路径已选定，可获知具体使用的候选对
}
```

### 5.4 FAILED 状态的常见陷阱

1. **GATHERING 仅在获得 STUN/TURN 候选时触发**——只有 host 候选不会触发 GATHERING，组件可能一直停留在 DISCONNECTED
2. **FAILED 不会瞬间触发**——有内置 grace delay，防止网络抖动导致频繁状态切换
3. **FAILED → CONNECTING 可恢复**——如果意外 STUN 请求到达已失败的组件，libnice 会尝试恢复
4. **Trickle ICE 下需手动触底**——`nice_agent_peer_candidate_gathering_done()` 才能使组件进入 FAILED

---

## 六、异常处理总结对比

| 协议层 | 错误定义来源 | libnice 谁处理 | 用户如何感知 |
|--------|-------------|---------------|-------------|
| **STUN** | RFC 5389 6 个核心 | `conncheck.c` `priv_map_reply_to_conn_check_request()` | 透明（通过候选对 Failed 间接影响） |
| **TURN** | RFC 5766 6 个 + 历史草案 | `discovery.c` 候选收集 | 候选收集结果（有没有 relay 候选） |
| **ICE** | RFC 8445 (487) | `conncheck.c` 状态机 | `"component-state-changed"` 信号 → FAILED |
| **RTP/RTCP** | RFC 3550/3551 | **不处理** | 用户代码自行处理丢包/乱序/jitter |
| **Socket** | OS 级 (ICMP, errno) | 内部重试/跳过 | 透明 |

### 设计原则

| 原则 | 体现 |
|------|------|
| **静默内部消化** | STUN 超时重传、认证重试、孤儿响应丢弃 —— 用户完全不知道 |
| **信号暴露结论** | `"component-state-changed"` 只告诉用户最终状态，不暴露中间过程 |
| **分层隔离** | RTP/RTCP 与 ICE 完全解耦，libnice 只负责"打通"，不负责"传输质量" |
| **防御性设计** | grace delay 防震荡、Trickle ICE 无限等待防过早放弃 |
| **可配置性** | timer_ta、Rc 等参数开放调节，适应不同网络环境 |

---

## 参考

- [[wiki/comparisons/comparison-ice-error-handling]] — ICE 异常处理完整分析 + STUN/TURN/ICE 错误码参考
- [[wiki/protocols/protocol-rfc8445]] — ICE 协议详细分析
- [[wiki/tutorials/tutorial-ice-connectivity-checks]] — ICE 连接性检查与提名详解
- [libnice Reference Manual](https://libnice.freedesktop.org/libnice/)
- [libnice GitLab](https://gitlab.freedesktop.org/libnice/libnice)
