---
title: ICE 异常处理完整分析
type: comparison
tags: [webrtc, ice, stun, turn, error-handling, error-codes, failure, rfc8445, rfc5389, rfc5766]
sources: [rfc8445, rfc5389, rfc5766]
created: 2026-05-28
updated: 2026-05-28
---

# ICE 异常处理完整分析

> ICE (RFC 8445) 的异常处理是一个多层级的、贯穿整个连接建立生命周期的体系。从候选收集、连接性检查、提名到会话终结，每个阶段都有对应的错误检测和恢复机制。

---

## 一、异常处理架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    ICE 异常处理层次                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   L1: 事务级  ── STUN 超时重传 (RTO)、认证失败重试        │
│       ↓                                                 │
│   L2: 对级    ── 候选对状态机 → Failed                  │
│       ↓                                                 │
│   L3: 检查表级 ── Checklist → Failed (全部对失败)        │
│       ↓                                                 │
│   L4: 会话级  ── ICE Session → Failed (全部表失败)       │
│       ↓                                                 │
│   L5: 恢复级  ── ICE Restart、TURN 降级                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

核心设计理念：**层层上报，逐级兜底**。下层能处理的不上报，处理不了的向上传递，直到触发 ICE Restart 或 TURN 降级。

---

## 二、候选收集阶段的异常

### 2.1 可恢复错误 → 重试

候选收集由定时器 `Ta`（默认 50ms）驱动。每次 `Ta` 到期，代理可以发起新的 STUN/TURN 事务。

```
每个 Ta 周期:
  ├── 可恢复错误 (如 401 认证失败) → 重试该事务
  └── 正常 → 为新候选发起事务
```

**典型场景**：TURN Allocate 请求返回 `401 Unauthorized`（需要认证），客户端添加 `MESSAGE-INTEGRITY` 后重试。

### 2.2 主机候选失效

主机候选不会超时，但底层网络接口可能发生变化（IP 变更、网卡禁用）：

> ICE 代理应该监控其使用的接口，使基础已消失的候选者失效。

**对策**：在连接性检查开始前重新验证候选有效性。

### 2.3 不可恢复错误 → 跳过

| 错误类型 | 处理 |
|----------|------|
| TURN 服务器不可达 | 跳过该服务器，不生成 relay 候选 |
| STUN 服务器无响应 | 跳过，不生成 srflx 候选 |
| 协议栈不支持 IPv6 | 跳过 IPv6 候选 |

---

## 三、连接性检查阶段的异常

这是 ICE 异常处理最复杂的部分。RFC 8445 Section 7.2.5 详细定义了四种失败模式。

### 3.1 候选对状态机

```
                    ┌──────────┐
                    │  Frozen  │  ← 初始状态
                    └────┬─────┘
                         │ unfreeze
                         ▼
                    ┌──────────┐
                    │ Waiting  │
                    └────┬─────┘
                         │ 触发检查
                         ▼
                    ┌──────────┐
         failure    │In-Progress│   success
         ┌──────────│          │──────────┐
         ▼          └──────────┘          ▼
    ┌──────────┐                    ┌──────────┐
    │  Failed  │                    │ Succeeded│
    └──────────┘                    └────┬─────┘
                                        │ 提名
                                        ▼
                                   ┌──────────┐
                                   │ Nominated│
                                   └──────────┘
```

`Failed` 状态可能因以下任一种情况触发。

### 3.2 失败模式一：事务超时 (Timeout)

**触发条件**：Binding 请求的 STUN 事务超时（RTO 到期后未收到响应）。

**处理**：候选对状态 MUST 设为 Failed。

```
Initiator                     Responder
    │                             │
    │──── Binding Request ───────▶│
    │       (TID=0xABC)          │
    │                             │
    │   RTO 到期, 无响应           │
    │   重传 Binding Request      │
    │       (TID=0xABC)          │
    │                             │
    │   Rc 次重传后仍无响应        │
    │                             │
    │   候选对 → Failed           │
```

RFC 8445 Section 7.2.5.2.3：

> 如果代理是控制代理，并且 ICE 代理尚未为该数据流生成有效列表，它可以在成对的 STUN 事务上增加超时重传次数，直到有效列表全部填充。

**关键细节**：控制代理在尚无有效对时可以"超限重试"，因为找不到可用路径的代价远大于多等几秒。

### 3.3 失败模式二：非对称传输地址 (Non-Symmetric)

**触发条件**：响应的源/目标 IP+Port 与请求不匹配。

```
请求: src=10.0.0.1:4000, dst=1.2.3.4:6000
响应: src=1.2.3.4:6001, dst=10.0.0.1:4000
        ↑ 端口变了！→ 非对称 → Failed
```

**处理**：候选对 MUST 设为 Failed。这是防止 IP 欺骗的关键安全措施。

RFC 8445 Section 7.2.5.2.1：

> 如果 Binding 请求的 STUN 事务生成成功响应，但响应中传输地址与请求不匹配（非对称），代理 MUST 将候选对状态设为 Failed。

### 3.4 失败模式三：ICMP 错误

**触发条件**：收到 ICMP 错误消息（如 Destination Unreachable）。

**处理**：

| ICMP 类型 | 处理 |
|-----------|------|
| Hard error (Host/Network Unreachable) | SHOULD 设为 Failed |
| Soft error (TTL Exceeded, Port Unreachable) | 可忽略，继续重试 |
| 任何 ICMP | 警惕 DoS 攻击（伪造 ICMP） |

RFC 8445 对此持谨慎态度：

> 代理 MAY 处理 ICMP 错误。硬 ICMP 错误 SHOULD 将候选对设为 Failed。注意 ICMP 可能被伪造，用于 DoS 攻击。

### 3.5 失败模式四：不可恢复的 STUN 响应

**触发条件**：收到非 487 的 STUN 错误响应（如 400 Bad Request, 401 Unauthorized, 420 Unknown Attribute）。

**处理**：SHOULD 将候选对设为 Failed（非强制性，留给实现灵活处理）。

### 3.6 特殊异常：角色冲突 (Role Conflict) — 487

这是 ICE 唯一定义的错误响应码。

```
场景:
  Agent A (认为自己是 controlling) 发送: ICE-CONTROLLING
  Agent B (也认为自己是 controlling) 收到后生成 487

处理流程:
  1. B 返回 487 (Role Conflict) 错误响应
  2. A 收到 487
     - 如果 A 发了 ICE-CONTROLLING → 切换到 controlled
     - 如果 A 发了 ICE-CONTROLLED → 切换到 controlling
  3. A 重新计算候选对优先级
  4. A 将该候选对加入 triggered-check 队列，状态设为 Waiting
  5. 下次检查时以正确的角色重试
```

**关键**：角色冲突不是真正的"失败"——它是可自动恢复的，不将候选对设为 Failed。这与四种失败模式有本质区别。

---

## 四、STUN/TURN/ICE 错误码完整参考

ICE 体系中的错误码定义在三个 RFC 中：STUN (RFC 5389) 定义了基础错误码，TURN (RFC 5766) 扩展了 6 个，ICE (RFC 8445) 新增了 1 个。共计 **13 个错误码**。

### 4.1 ERROR-CODE 属性格式

所有错误码通过 STUN 消息的 `ERROR-CODE` 属性（`0x0009`）携带：

```
ERROR-CODE 属性结构:
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |           Reserved (MBZ)      |  Class  |   Number           |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |      Reason Phrase (variable, UTF-8, max 128 chars)         |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  Class:  3 bit, 值 3-6 (对应百位 300/400/500/600)
  Number: 8 bit, 值 0-99 (对应十位和个位)
  Code = Class × 100 + Number
```

### 4.2 客户端默认处理规则（RFC 5389 Section 7.3.4）

| 范围 | 默认行为 |
|------|---------|
| **300-399** | 事务失败，但应先检查 `ALTERNATE-SERVER` 属性尝试重定向 |
| **400-499** | 事务失败（例外：420 提供 `UNKNOWN-ATTRIBUTES` 回调；487 触发 ICE 角色切换） |
| **500-599** | 可延迟重试（需限制重试频率） |
| **其他** | 事务失败 |

### 4.3 STUN 错误码（RFC 5389）— 6 个

| 错误码 | 名称 | 触发条件 | 客户端处理 |
|--------|------|---------|-----------|
| **300** | Try Alternate | 服务器重定向 | 获取 `ALTERNATE-SERVER` 属性，用新地址重试。5 分钟内不重复跳转同一服务器 |
| **400** | Bad Request | 请求格式错误（如缺少必需属性） | **不重试**，需修改请求后重新发起 |
| **401** | Unauthorized | 缺少认证或认证失败。响应带 `REALM` + `NONCE` | 用 `USERNAME` + `REALM` + `NONCE` + `MESSAGE-INTEGRITY` 重试 |
| **420** | Unknown Attribute | 包含服务器不理解的 comprehension-required 属性 | 响应带 `UNKNOWN-ATTRIBUTES` 列表。客户端可排除这些属性后重试 |
| **438** | Stale Nonce | NONCE 已过期。响应带新 `NONCE` + `REALM` | 用新 NONCE 重试（类似 401 但 NONCE 只是过期而非缺失） |
| **500** | Server Error | 服务器临时故障 | 可延迟重试 |

### 4.4 TURN 错误码（RFC 5766）— 新增 6 个

| 错误码 | 名称 | 触发条件 | 客户端处理 |
|--------|------|---------|-----------|
| **403** | Forbidden | 认证通过但策略拒绝（IP 黑名单、无权限、配额用尽不属此类） | **不重试**（终端错误），需换凭据或换服务器 |
| **437** | Allocation Mismatch | 客户端地址/端口与现有分配不匹配。常见于 NAT rebinding 后 | 重新发起 `Allocate` 请求，旧分配已丢失 |
| **441** | Wrong Credentials | 凭据本身错误（区别于 401 的"未认证"） | **不重试**，需获取并更换正确凭据 |
| **442** | Unsupported Transport | `REQUESTED-TRANSPORT` 不支持（TURN 标准仅要求 UDP） | 换用 UDP 重试，或切换到支持所需协议的服务器 |
| **486** | Allocation Quota Reached | 达到单用户 TURN 分配数上限 | 先释放旧分配（Refresh `LIFETIME=0`），再重新 `Allocate` |
| **508** | Insufficient Capacity | 服务器资源不足（地址池/带宽/CPU/内存耗尽） | 延迟重试，或通过 300 机制切换到备用服务器 |

### 4.5 ICE 错误码（RFC 8445）— 新增 1 个

| 错误码 | 名称 | 触发条件 | 客户端处理 |
|--------|------|---------|-----------|
| **487** | Role Conflict | 双方角色冲突：受控方收到 `ICE-CONTROLLING`，或控制方收到 `ICE-CONTROLLED` | **自动恢复**：切换角色 → 重新计算优先级 → 候选对重新入队 Waiting → 重试。候选对**不标记 Failed** |

**487 触发的三种场景**（RFC 8445 Section 7.3.1.1）：

```
场景 1: Agent 处于 controlled 角色，收到带 ICE-CONTROLLING 的 Binding 请求
场景 2: Agent 处于 controlling 角色，收到带 ICE-CONTROLLED 的 Binding 请求
场景 3: Agent 收到不含 ICE-CONTROLLING 和 ICE-CONTROLLED 的 Binding 请求
        → 响应中附上自己的角色属性，由对方调整
```

**487 处理流程**（RFC 8445 Section 7.2.5.1）：

```
1. 如果请求含 ICE-CONTROLLING → 切换到 controlled 角色
2. 如果请求含 ICE-CONTROLLED  → 切换到 controlling 角色
3. 重新计算候选对优先级（角色影响优先级计算）
4. 候选对加入 triggered-check 队列，状态设为 Waiting
5. 更改 tiebreaker 值
6. 下次连接性检查时使用新角色
```

### 4.6 按恢复策略分类总览

```
┌──────────────────────────────────────────────────────────────┐
│            STUN/TURN/ICE 错误码 — 恢复策略分类                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ★ 自动恢复（无需客户端干预）:                                  │
│     487 Role Conflict  → ICE 角色自动切换 + 重新入队            │
│                                                              │
│  ◆ 客户端可重试（修改后重试或换目标）:                           │
│     300 Try Alternate        → 换备用服务器                    │
│     401 Unauthorized         → 添加认证信息                    │
│     438 Stale Nonce          → 更新 NONCE                     │
│     437 Allocation Mismatch  → 重新 Allocate                  │
│     442 Unsupported Transport → 换用 UDP                      │
│     486 Allocation Quota     → 先释放旧分配再重试               │
│     500 Server Error         → 延迟重试                        │
│     508 Insufficient Capacity → 延迟重试或换备用服务器           │
│                                                              │
│  ▲ 终端错误（不可重试，需根本性修改）:                           │
│     400 Bad Request          → 请求格式错误                    │
│     403 Forbidden            → 策略拒绝                        │
│     420 Unknown Attribute    → 属性不兼容（可排除后重试）        │
│     441 Wrong Credentials    → 凭据错误                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.7 易混淆错误码辨析

| 对比 | 区别 |
|------|------|
| **401 vs 438** | 401 = 未认证（无 NONCE 或 NONCE 无效，首次挑战），438 = NONCE 过期（之前认证通过但 NONCE 超时）。两者都返回 `REALM` + `NONCE` 并期望客户端重试 |
| **401 vs 441** | 401 = "你没认证，请先认证"（挑战-响应流程），441 = "你的凭据就是错的"（终端错误，换了也白换） |
| **401 vs 403** | 401 = 身份未知/未认证（可重试），403 = 身份已知但无权限（不可重试） |
| **400 vs 420** | 400 = 请求整体格式错误，420 = 具体某个 comprehension-required 属性不识别（更有信息量，服务端告知是哪些属性） |
| **500 vs 508** | 500 = 通用服务器错误（STUN 层面），508 = TURN 资源不足（更具体的语义，指向容量问题） |
| **437 vs 401** | 437 = 地址不匹配（NAT rebinding 后地址变了），401 = 认证问题。437 不需要重新认证，只需要重新分配 |

---

## 五、提名阶段的异常

提名失败比普通连接性检查失败**更严重**——因为它意味着已经被标记为"可用"的路径突然失效了。

RFC 8445 Section 7.2.5.3.4：

> 如果控制代理发送带有 USE-CANDIDATE 属性的 Binding 请求并收到失败响应，代理 MUST:
> 1. 从有效列表中移除该候选对
> 2. 将候选对状态设为 Failed
> 3. 将检查列表状态设为 Failed

**连锁反应**：一次提名失败直接导致整个 Checklist Failed，触发回退到其他候选对或 ICE Restart。

---

## 六、聚合失败：从候选对到 ICE 会话

### 6.1 Checklist 级别

```
Checklist 判定为 Failed 的条件:
  所有候选对处于 Failed 或 Succeeded 状态
  AND 有效列表中没有有效对 (valid pair)
```

换言之：所有可能的路径都试过了，但没有一条能用的。

### 6.2 ICE 会话级别

```
ICE 状态机:
  Running   → 检查进行中
  Completed → 所有 Checklist 都有 nominated pair
  Failed    → 所有 Checklist 都是 Failed
```

**注意区分**：Succeeded ≠ Completed。Succeeded 意味着检查列表有 valid pair；Completed 要求 checklists 有 nominated pair。

---

## 七、恢复机制

### 7.1 检查调度中的提前中止

在每轮检查调度中，如果没有可执行的检查，ICE 代理会"中止后续步骤"(abort subsequent steps)。这是一种防御性设计——避免在不可能有进展的情况下浪费计算资源。

### 7.2 终止检查（本地策略）

> 代理可以选择随时终止对检查列表集中一个或多个检查列表执行连通性检查。

这是一个逃生阀：如果应用层判断继续尝试无意义（如用户挂断电话），可以主动终止。

### 7.3 ICE Restart

当以下任一条件满足时触发 ICE Restart：

| 触发条件 | 典型场景 |
|----------|---------|
| 数据流目标地址变化 | WiFi 切 4G，IP 地址变更 |
| Lite/Full 实现类型切换 | 与不支持 Full ICE 的旧设备互通 |
| 密码和 ufrag 变更 | 安全策略要求刷新凭证 |
| ICE 会话 Failed | 所有路径都断开 |

Restart 时重新生成 `pwd` 和 `ufrag`，generation 值递增（+1），整个 ICE 流程重来。

### 7.4 TURN 降级

当所有直连/打洞路径（host、srflx）都失败时，TURN 中继候选作为**最终兜底**：

```
优先级: host > srflx > relay(TURN)
失败时: host Failed → 尝试 srflx → srflx Failed → 使用 TURN
```

TURN 几乎总是能连通（只要中继服务器可达），代价是带宽和延迟开销。

---

## 八、NAT 层面的异常

UDP 打洞本身不是 ICE 的一部分，但 ICE 的连接性检查本质上在做打洞——因此 NAT 行为异常直接影响 ICE。

### 8.1 Symmetric NAT → 打洞必然失败

```
NAT-A (Cone NAT):  10.0.0.1:4000 → 1.2.3.4:60000 (发给谁都一样)
NAT-B (Symmetric): 10.0.0.2:5000 → 1.2.3.5:60001 (发给 A)
                                  → 1.2.3.5:60002 (发给 STUN 服务器)
                                  ↑ 端口不同！
```

Alice 从 STUN 服务器得知 Bob 的公网地址是 `1.2.3.5:60002`，但 Bob 发给 Alice 时实际映射为 `60001`。Alice 发包到 `60002` → 被 Bob 的 NAT 丢弃 → 候选对 Failed。

**ICE 的应对**：不预测端口，而是通过连接性检查实际"试"出来。如果所有尝试都失败 → 降级 TURN。

### 8.2 NAT 映射超时

NAT 映射有超时计时器（通常 30-120 秒）。在打洞期间如果两次重试间隔太长，映射可能过期。

**ICE 的应对**：快速重传（RTO 初始 500ms，指数退避），确保在映射超时前完成。

### 8.3 过时的中继地址

候选收集完成后到连接性检查开始前，如果 Bob 的 NAT 映射已过期 → 端口变化 → 连接失败。

**ICE 的应对**：候选收集紧接着连接性检查，时间窗口极小。

---

## 九、STUN 消息多路分解与格式校验

ICE 依赖 STUN 消息的格式校验来判断是否收到了有效的 STUN 响应：

```
STUN 消息检测:
  1. 前 2 位是否为 0?         → 否 → 不是 STUN
  2. Magic Cookie = 0x2112A442? → 否 → 不是 STUN
  3. FINGERPRINT CRC32 校验?   → 否 → 不是 STUN

  全部通过 → 是 STUN 消息
  任一失败 → 交给其他协议处理 (RTP/RTCP/etc.)
```

这是一个**多路分解（demux）**机制：不是 STUN 的包不会被错误解释为 STUN 响应，而是路由给正确的协议处理器。

---

## 十、定时器与重传策略

| 参数 | 默认值 | 作用 |
|------|--------|------|
| **Ta** | 50ms | 候选收集和检查的事务间隔 |
| **RTO** | 动态 (基于 RTT) | STUN 事务重传超时 |
| **Rc** | 7 | 最大重传次数 |
| **Allocation Lifetime** | 600s | TURN 分配生命周期 |
| **Permission Lifetime** | 300s | TURN 权限生命周期 |

RTO 的计算基于 RTT 估计（类比 TCP），不是固定值：

```
RTO = max(500ms, SRTT + 4 × RTTVAR)
其中:
  SRTT = 平滑后的往返时间
  RTTVAR = RTT 变化方差
```

---

## 十一、异常处理设计原则总结

| 原则 | 体现 |
|------|------|
| **分层兜底** | 事务重试 → 候选对 Failed → Checklist Failed → Session Failed → ICE Restart |
| **可恢复优先** | 认证失败重试、487 角色冲突自动切换，均不标记为 Failed |
| **安全防御** | 非对称地址检测防 IP 欺骗；ICMP 伪造警示防 DoS |
| **渐进降级** | host → srflx → relay，优先直连，失败后走中继 |
| **快速失败** | RTO 500ms 起步，避免长时间阻塞 |
| **逃生阀** | 本地策略可随时终止检查；ICE Restart 提供全局重置 |
| **无单点故障** | 每个组件、每个候选对独立检查，一个失败不影响其他 |

---

## 十二、与其他协议的对比

| 维度 | ICE (RFC 8445) | 原始 UDP 打洞 | SIP |
|------|----------------|--------------|-----|
| 超时处理 | RTO 动态计算，最多 Rc 次重传 | 无标准，靠运气 | SIP 协议栈处理重传/故障转移 |
| 失败恢复 | 状态机驱动，自动降级 TURN | 无机制 | 依赖于底层传输 |
| 错误码 | 487 (Role Conflict) | 无 | 完整 SIP 响应码体系 |
| NAT 失败 | 检测到 Symmetric NAT → 降级 TURN | 打洞失败，无法恢复 | 不涉及 |

---

## 参考

- [[wiki/protocols/protocol-rfc8445]] — ICE 协议详细分析
- [[wiki/concepts/concept-ice]] — ICE 通俗理解
- [[wiki/tutorials/tutorial-ice-candidate-gathering]] — 候选收集流程（含 STUN/TURN TLV）
- [[wiki/tutorials/tutorial-ice-connectivity-checks]] — 连接性检查与提名
- [[wiki/tutorials/tutorial-udp-hole-punching]] — UDP 打洞流程与 NAT 类型适配
- [[raw/rfcs/rfc8445_完整中英对照.md]] — RFC 8445 完整中英对照
