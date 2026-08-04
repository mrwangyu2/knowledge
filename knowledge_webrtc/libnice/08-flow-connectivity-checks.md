# 08 -- 流程：连接检查

## 概述

连接检查（Connectivity Checks）是 ICE 协议（RFC 8445）的核心机制。它负责对所有可能的候选对进行可达性测试，选出最优路径，最终驱动 component 状态从 CONNECTING 演进到 CONNECTED，再到 READY。

整个过程由 `nice_agent_set_remote_candidates()` 触发，通过一个定时器驱动的主循环（`priv_conn_check_tick_agent_locked()`）在后台执行，无需用户干预。

### 整体流程概要

```
set_remote_candidates()
  -> conn_check_remote_candidates_set()
    -> 为新候选创建候选对 (FROZEN)
    -> priv_schedule_next(): 启动 20ms 定时器
      -> priv_conn_check_tick_agent_locked()  [每 20ms 触发]
        -> 1. 处理 triggered checks (优先级最高)
        -> 2. 处理 STUN 重传 (tick_stream)
        -> 3. 处理 ordinary checks (解冻 + 发送)
        -> 4. 处理 nomination (选择并标记 candidate pairs)
      -> ... 接收端 ...
      -> conn_check_handle_inbound_stun()
        -> 验证 STUN 消息
        -> 如果是 Request: 回复 + 调度 triggered check
        -> 如果是 Response: priv_map_reply_to_conn_check_request()
          -> 事务 ID 匹配 -> STUN ICE process
          -> 更新候选对状态 SUCCEEDED/FAILED
          -> priv_process_response_check_for_reflexive()
          -> conn_check_unfreeze_related()
          -> conn_check_update_check_list_state_for_ready()
```

---

## 触发点

### nice_agent_set_remote_candidates() -> 连接检查启动

**文件**: `agent/agent.c` -> `agent/conncheck.c`

当用户调用 `nice_agent_set_remote_candidates()` 时：

1. `_set_remote_candidates_locked()` 将远端候选添加到 component 的 `remote_candidates` 列表
2. 调用 `conn_check_remote_candidates_set()` 为新候选创建候选对
3. 每个新候选对通过 `priv_conn_check_add_for_candidate_pair_matched()` 创建，初始状态为 `NICE_CHECK_FROZEN`
4. `priv_schedule_next()` 启动连接检查定时器和 keepalive 定时器

```c
// agent/agent.c:4560
if (added > 0)
    conn_check_remote_candidates_set(agent, stream, component);
```

### 远程凭据设置后的延迟处理

如果远端在发送 SDP answer 之前就已发送 STUN Binding Request（即 incoming check 先于 remote credentials 到达），这些请求会暂存在 `component->incoming_checks` 队列中。当 `conn_check_remote_credentials_set()` 被调用时，会回放这些暂存的 incoming checks。

---

## 步骤 1: 候选对构造与排序

### 候选对的创建规则

**文件**: `agent/conncheck.c:2444` `conn_check_add_for_candidate_pair()`

创建候选对时必须满足以下匹配条件：
- **相同地址族**: `local->addr.s.addr.sa_family == remote->addr.s.addr.sa_family`
- **兼容传输类型**: `local->transport == conn_check_match_transport(remote->transport)`
  - UDP <-> UDP, TCP-SO <-> TCP-SO
  - TCP-ACTIVE <-> TCP-PASSIVE
- **不跨越链路范围**: link-local 不与 non-link-local 配对

排除规则：
- 不创建以 server-reflexive 或 peer-reflexive 为 local 的候选对（RFC 8445 sect 6.1.2.4）
- 不创建以 TCP-PASSIVE 为 local 的候选对（ice-tcp-13 6.2）

### `conn_check_add_for_candidate()`

**文件**: `agent/conncheck.c:2495`

当添加一个 remote candidate 时，遍历所有 local candidate，调用 `conn_check_add_for_candidate_pair()` 尝试配对。

对于 peer-reflexive 类型的 remote candidate，在 RFC 5245 兼容模式下不主动创建配对（它们只由 incoming check 触发）。

### 候选对优先级计算

**文件**: `agent/conncheck.c:2320` `agent_candidate_pair_priority()`

```
candidate-pair-priority = 2^32 * MIN(G, D) + 2 * MAX(G, D) + (G > D ? 1 : 0)
```

其中：
- **controlling agent**: G = local candidate priority, D = remote candidate priority
- **controlled agent**: G = remote candidate priority, D = local candidate priority

候选对存储在 `stream->conncheck_list` 中，按优先级降序排序（`conn_check_compare()`）。

### 初始状态分配

**文件**: `agent/conncheck.c:2309` `priv_add_new_check_pair()`

新创建的候选对初始状态为 `NICE_CHECK_FROZEN`。创建后调用 `priv_conn_check_unfreeze_maybe()` -- 如果已有同 foundation 的 SUCCEEDED 候选对，则直接解冻为新候选对。

同时检查 `component->selected_pair.priority`：如果新候选对的优先级低于已选定候选对，则不创建该候选对。

候选对数量有硬上限（`priv_limit_conn_check_list_size()`），按照 RFC 5245 sect 5.7.3 实现，过量的低优先级候选对会被移除。

---

## 步骤 2: 定时器驱动的检查循环

### 定时器启动

**文件**: `agent/conncheck.c:1722` `priv_schedule_next()`

```c
if (agent->conncheck_timer_source == NULL) {
    agent_timeout_add_with_context (agent, &agent->conncheck_timer_source,
        "Connectivity check schedule", agent->timer_ta,
        priv_conn_check_tick_agent_locked, NULL);
}
```

默认 `timer_ta` = 20ms（即 Ta = 20ms，符合 ICE 规范的 pacing 要求）。

### priv_conn_check_tick_agent_locked() -- 主循环入口

**文件**: `agent/conncheck.c:1142`

每次触发时按优先级顺序执行，且每次只发送一个 STUN 请求（pacing 约束）：

```
1. priv_conn_check_triggered_check()     -- 处理 triggered checks 队列 (优先级最高)
2. priv_conn_check_tick_stream()          -- 处理 STUN 重传/超时
3. priv_conn_check_ordinary_check()       -- 发送 ordinary checks (解冻 + 发送)
4. priv_conn_check_tick_stream_nominate() -- 候选对提名 (nomination)
```

只要任一步骤发送了 STUN 消息（`stun_sent == TRUE`），定时器就继续保持。

### 空闲检测与停止

如果 `keep_timer_going == FALSE`，累计 `conncheck_ongoing_idle_delay`。当空闲时间超过 `idle_timeout` 时，调用 `priv_update_check_list_failed_components()` 将没有 valid pair 的 component 标记为 FAILED，然后停止定时器。

---

### 2a. Triggered Checks（触发检查）

**文件**: `agent/conncheck.c:765` `priv_conn_check_triggered_check()`

从 `agent->triggered_check_queue` 取出一个候选对，立即调用 `priv_conn_check_initiate()` 发送检查。Triggered check 优先级最高，用于响应 incoming check 或 nomination 重发。

### 2b. STUN 事务管理与重传

**文件**: `agent/conncheck.c:633` `priv_conn_check_tick_stream()`

遍历所有候选对的 `stun_transactions` 列表，对每个 STUN 事务调用 `stun_timer_refresh()`：

| 返回值 | 行为 |
|--------|------|
| `RETRANSMIT` | 重新发送 STUN 请求，更新下次 tick 时间，`keep_timer_going=TRUE` |
| `TIMEOUT` | 移除该 STUN 事务（`priv_remove_stun_transaction()`） |
| `SUCCESS` | 更新下次 tick 时间 |

当候选对的所有 STUN 事务都结束后（`remaining == 0`），该候选对标记为 FAILED，调用 `candidate_check_pair_fail()`。

如果重传被 nomination 禁止（`p->retransmit == FALSE` 或 `index > 0`），直接超时处理。

### 2c. Ordinary Checks（普通检查）

**文件**: `agent/conncheck.c:736` `priv_conn_check_ordinary_check()`

按 RFC 8445 sect 6.1.4.2 执行：

1. 从 `conncheck_list` 中查找第一个 `NICE_CHECK_WAITING` 状态的候选对
2. 如果找不到，调用 `priv_conn_check_unfreeze_next()` 解冻候选对
3. 对找到的候选对调用 `priv_conn_check_initiate()` 发送 STUN Binding Request

### 2d. Nomination（提名）

**文件**: `agent/conncheck.c:786` `priv_conn_check_tick_stream_nominate()`

见步骤 8 的详细说明。核心是：controlling agent 根据 stopping criterion 选择要 nominate 的候选对，将其加入 triggered check queue 并以 USE-CANDIDATE 属性重发。

---

## 步骤 3: 发送连接检查

### conn_check_send()

**文件**: `agent/conncheck.c:2841`

发送单个连接检查的核心函数：

1. **构造用户名**: `priv_create_username()` -- 格式为 `local_ufrag:remote_ufrag`（出站方向）
2. **获取密码**: `priv_get_password()` -- 用于 MESSAGE-INTEGRITY 属性
3. **创建 STUN 事务**: `priv_add_stun_transaction()` -- 在候选对上创建新的 StunTransaction
4. **构造 STUN 消息**: `stun_usage_ice_conncheck_create()` 设置属性
5. **启动重传定时器**:
   - 可靠传输（TCP）: `stun_timer_start_reliable(&stun->timer, timeout)` -- 无最大重传次数
   - 不可靠传输（UDP）: `stun_timer_start(&stun->timer, timeout, agent->stun_max_retransmissions)` -- 最大 7 次
6. **TCP-ACTIVE 特殊处理**: 如果 `sockptr->fileno == NULL`（TCP 尚未连接），调用 `nice_tcp_active_socket_connect()` 创建新连接
7. **发送**: `agent_socket_send()` 将 STUN 消息发送到远端地址

---

## 步骤 4: STUN 消息构造

### stun_usage_ice_conncheck_create()

**文件**: `stun/usages/ice.c:62`

构造 ICE 连接检查的 STUN Binding Request：

1. `stun_agent_init_request()` -- 设置 STUN Method = Binding
2. 根据兼容模式添加属性：
   - **USE-CANDIDATE** flag -- 如果 `cand_use == TRUE`（仅 controlling agent 在 nomination 时设置）
   - **PRIORITY** -- 候选对的 STUN 优先级（`pair->stun_priority`）
   - **ICE-CONTROLLING** (64-bit tie-breaker) -- 用于角色冲突检测
   - 或 **ICE-CONTROLLED** (64-bit tie-breaker) -- controlled agent 使用
3. **USERNAME** 属性 -- `local_ufrag:remote_ufrag`
4. 兼容模式扩展：
   - **MS-ICE2**: 添加 CANDIDATE-IDENTIFIER 和 MS-IMPLEMENTATION-VERSION
5. `stun_agent_finish_message()` -- 计算 MESSAGE-INTEGRITY（HMAC-SHA1），完成消息

---

## 步骤 5: 网络发送

### agent_socket_send() -> NiceSocket

**文件**: `agent/agent.c:7804`

```c
if (nice_socket_is_reliable(sock)) {
    // ICE-TCP: 使用 RFC 4571 帧封装
    ret = nice_socket_send_messages_reliable(sock, addr, &local_message, 1);
} else {
    // UDP: 先尝试可靠发送（若有 PseudoTCP），否则直接 send
    ret = nice_socket_send_reliable(sock, addr, len, buf);
    if (ret < 0)
        ret = nice_socket_send(sock, addr, len, buf);
}
```

### 虚表分派

- `nice_socket_send()` -> 通过 `NiceSocket` 虚表分派到 `udp-bsd.c`、`tcp-bsd.c`、`udp-turn.c` 等具体实现
- 对于可靠传输，使用 RFC 4571 帧封装（2 字节长度前缀 + payload）

---

## 步骤 6: 接收响应与消息分发

### Socket Recv -> Component -> Dispatching

**文件**: `agent/agent.c:4682` `agent_recv_message_unlocked()`

数据到达流程：

```
OS socket readable event
  -> nice_socket_recv_messages()
    -> 数据到达 agent_recv_message_unlocked()
      -> STUN 快速检测: stun_message_validate_buffer_length_fast()
        -> 如果是 STUN:
           -> conn_check_handle_inbound_stun()  (agent/conncheck.c:4532)
             -> 分类处理: Request / Response / Error
        -> 如果不是 STUN:
           -> 检查是否是 PseudoTCP 数据
           -> 否则作为 data 传递给应用层 (attached recv callback 或 recv queue)
```

### conn_check_handle_inbound_stun()

**文件**: `agent/conncheck.c:4532`

收到 STUN 消息后的主分发函数：

1. **验证**: 依次尝试 component、discovery、relay refresh 的 `stun_agent_validate()`
2. **错误处理**: BAD_REQUEST / UNAUTHORIZED / FORBIDDEN（403）等
3. **根据 STUN Class 分流**:

#### A. STUN_REQUEST（入站连接检查）

1. 查找 local candidate（通过 socket 地址匹配）
2. 查找 remote candidate（通过 from 地址匹配，）-- 若未匹配，创建 peer-reflexive remote candidate
3. 检查 `component->have_local_consent`（RFC 7675），若无则回复 FORBIDDEN
4. 调用 `stun_usage_ice_conncheck_create_reply()` 构造响应
5. 检测角色冲突: `priv_check_for_role_conflict(agent, control)`
6. 调用 `priv_reply_to_conn_check()` 发送响应
7. 如果 remote credentials 尚未就绪，将请求暂存到 `component->incoming_checks`
8. 否则调度 triggered check: `priv_schedule_triggered_check()`

#### B. STUN_RESPONSE（出站检查的响应）

按优先级尝试匹配事务：

1. `priv_map_reply_to_conn_check_request()` -- 匹配连接检查事务
2. `priv_map_reply_to_discovery_request()` -- 匹配候选发现事务
3. `priv_map_reply_to_relay_request()` -- 匹配 TURN 分配事务
4. `priv_map_reply_to_relay_refresh()` -- 匹配 TURN 刷新事务
5. `priv_map_reply_to_relay_remove()` -- 匹配 TURN 移除事务
6. `priv_map_reply_to_keepalive_conncheck()` -- 匹配 keepalive 事务

---

## 步骤 7: 处理连接检查响应

### priv_map_reply_to_conn_check_request()

**文件**: `agent/conncheck.c:3542`

这是处理连接检查响应的核心函数。

1. **事务 ID 匹配**: 遍历所有候选对的 `stun_transactions` 列表，通过 Transaction ID 匹配
2. **STUN ICE Process**: 调用 `stun_usage_ice_conncheck_process()` 解析响应中的 XOR-MAPPED-ADDRESS
3. **源地址校验**: 响应必须来自与请求相同的目的地址（RFC 8445 7.1.2.1 Failure Cases）
4. **Peer Reflexive 发现**: `priv_process_response_check_for_reflexive()` -- 检查 mapped address 是否匹配已知 local candidate，如果不匹配则触发 peer-reflexive candidate 发现
5. **状态更新**: 标记候选对为 SUCCEEDED（及可能的 DISCOVERED 候选对）
6. **解冻**: `conn_check_unfreeze_related()` -- 解冻同 foundation 的 FROZEN 候选对
7. **Nomination 标记**: 根据 USE-CANDIDATE 属性和 nomination mode 设置 nominated flag
8. **状态机推进**: 检查是否达到 CONNECTED 或 READY 状态

**错误响应处理**:

| StunUsageIceReturn | 行为 |
|---|---|
| `ROLE_CONFLICT` | 角色冲突：切换 controlling/controlled，将候选对加入 triggered check queue |
| `ERROR` | 候选对标记为 FAILED |

### stun_usage_ice_conncheck_process()

**文件**: `stun/usages/ice.c:142`

解析连接检查响应的 STUN 消息：
1. 验证 STUN Method = Binding
2. 只处理 `STUN_RESPONSE` 类消息（Request/Indication 返回 INVALID）
3. 检查错误码：`STUN_ERROR_ROLE_CONFLICT` 返回 `STUN_USAGE_ICE_RETURN_ROLE_CONFLICT`
4. 提取 XOR-MAPPED-ADDRESS（及其 fallback MAPPED-ADDRESS）
5. 兼容 MSN 模式时使用特殊的 magic cookie 解 XOR 地址

### Peer Reflexive Candidate Discovery

**文件**: `agent/conncheck.c:3432` `priv_process_response_check_for_reflexive()`

当响应中的 mapped address 与任何已知 local candidate 不匹配时：
1. 调用 `discovery_add_peer_reflexive_candidate()` 创建新的 peer-reflexive local candidate
2. 调用 `priv_add_peer_reflexive_pair()` 创建 DISCOVERED 状态的候选对
3. DISCOVERED 候选对继承其 parent SUCCEEDED 候选对的 nominated 状态

---

## 步骤 8: 解冻机制

### priv_conn_check_unfreeze_next()

**文件**: `agent/conncheck.c:365`

按照 RFC 8445 sect 6.1.2.6 和 6.1.4.2 的简化算法：

1. 如果已有 WAITING 状态候选对，不做任何事情
2. 否则，遍历所有 FROZEN 候选对：
   - 每个 foundation 只解冻一个 FROZEN 候选对，将其设为 WAITING
   - 跳过已有 WAITING 候选对的 foundation

这是一个幂等操作。

### conn_check_unfreeze_related()

**文件**: `agent/conncheck.c:428`

当某个候选对检查成功后调用：
- 将所有同 foundation 的 FROZEN 候选对设为 WAITING
- 实现 RFC 8445 sect 7.2.5.3.3 "Updating Candidate Pair States"

### priv_conn_check_unfreeze_maybe()

**文件**: `agent/conncheck.c:472`

当新候选对被创建（FROZEN）时调用：
- 如果已有同 foundation 的 SUCCEEDED 候选对，直接解冻新候选对

---

## 步骤 9: 重传机制

### STUN 重传定时器

**文件**: `stun/usages/timer.c`

#### stun_timer_start() -- 启动重传定时器

```c
void stun_timer_start(StunTimer *timer,
    unsigned int initial_timeout,      // 初始超时 (通常 500ms)
    unsigned int max_retransmissions)  // 最大重传次数 (通常 7)
```

- `retransmissions = 1`, `delay = initial_timeout`
- 设置 deadline = now + delay

#### stun_timer_refresh() -- 检查重传时机

```c
StunUsageTimerReturn stun_timer_refresh(StunTimer *timer)
```

| 条件 | 返回值 |
|---|---|
| 未到 deadline | `RETURN_SUCCESS` |
| 已达 deadline 且已用尽重传次数 | `RETURN_TIMEOUT` |
| 已达 deadline 且还有重传次数 | `RETURN_RETRANSMIT` |

指数退避策略：
- 前 `max_retransmissions - 1` 次: `delay = delay * 2`（指数增长）
- 最后一次重传: `delay = delay / 2`（减半，用于最后的重试）

RTO 初始值计算（`priv_compute_conncheck_timer()`）：
```
RTO = timer_ta * (math.pow(2, waiting + in_progress) - 1) + timer_ta * (waiting + in_progress)
```
即 RTO 随正在进行的检查数量动态增长，用于 TCP-friendly pacing。

默认值:
- `initial_timeout = 500ms`（UDP RTO）
- `max_retransmissions = 7`
- 总重传时间约 39.5 秒

#### stun_timer_start_reliable() -- 可靠传输定时器

```c
void stun_timer_start_reliable(StunTimer *timer, unsigned int initial_timeout)
```

内部调用 `stun_timer_start(timer, initial_timeout, 0)`，即 `max_retransmissions = 0`。对于 TCP 等可靠传输，只需等待响应，不需要重传。

---

## 步骤 10: CONNECTED 与 READY 判定

### priv_conn_check_update_check_list_state_for_ready()

**文件**: `agent/conncheck.c:2152`

检查 component 是否可以从 CONNECTED 过渡到 READY：

1. 统计该 component 的 valid + nominated 候选对数量
2. 如果有 nominated pair（`nominated > 0`）：
   - 调用 `priv_prune_pending_checks()` 清理低优先级候选对
   - 如果没有剩余的 in-progress/triggered 候选对：
     - 渐进式状态推进: `CONNECTING -> CONNECTED -> READY`
     - 发射 `"component-state-changed"` 信号

### priv_update_check_list_failed_components()

**文件**: `agent/conncheck.c:2080`

遍历所有 component，检查是否有 valid pair：
- 如果没有 valid pair，且所有候选对都处于终态（无 FROZEN/WAITING/IN_PROGRESS），标记 component 为 FAILED

### conn_check_update_selected_pair()

**文件**: `agent/conncheck.c:2037`

当候选对被提名后，更新 `component->selected_pair`，记录选定候选对信息：
- local/remote candidate
- priority
- foundation

---

## 步骤 11: 角色冲突处理

### priv_check_for_role_conflict()

**文件**: `agent/conncheck.c:3402`

当收到角色冲突错误或 incoming request 中角色不匹配时：
1. 如果本地角色与 needed 角色不同：
   - 切换 `agent->controlling_mode`
   - 调用 `recalculate_pair_priorities()` 重新计算所有候选对优先级
2. 如果本地角色已经匹配：不做改变（peer 需要切换）

### 角色冲突的两种场景

1. **收到 STUN Error Response (487 Role Conflict)**:
   - 在 `priv_map_reply_to_conn_check_request()` 中处理
   - 将候选对加入 triggered check queue，等待重试

2. **处理入站 STUN Request 时**:
   - 在 `stun_usage_ice_conncheck_create_reply()` 中检测
   - 根据 tie-breaker 值比较决定哪个 agent 切换角色
   - 如果本地需要切换：`*control = !*control`，返回 `STUN_USAGE_ICE_RETURN_ROLE_CONFLICT`
   - 如果本地不需要切换：返回错误响应 487

### 优先级重算

```c
void recalculate_pair_priorities(NiceAgent *agent) {
    // 重新计算所有候选对的优先级并重新排序
    p->priority = agent_candidate_pair_priority(agent, p->local, p->remote);
    stream->conncheck_list = g_slist_sort(stream->conncheck_list, ...);
}
```

---

## 步骤 12: Triggered Check 调度

### priv_schedule_triggered_check()

**文件**: `agent/conncheck.c:3107`

在收到入站连接检查后调度触发检查：

1. 在 `conncheck_list` 中查找匹配的候选对（同 component、同 remote candidate、同 socket）
2. 根据候选对状态决定行为：

| 当前状态 | 行为 |
|---|---|
| WAITING / FROZEN | 加入 triggered check queue |
| IN_PROGRESS | 如果优先级高于 `selected_pair`，加入 triggered check queue |
| FAILED | 如果优先级高于 `selected_pair`，重新激活并加入 queue |
| SUCCEEDED | 无需操作 |

3. 如果不存在匹配候选对，创建新的候选对并加入 triggered check queue

特殊情况处理：
- FAILED 候选对重新激活时：如果 component 状态为 FAILED -> 切回 CONNECTING；如果 READY -> 切回 CONNECTED
- DISCOVERED 候选对：跟随其 `succeeded_pair` parent pair

---

## 步骤 13: Nomination 提名机制

### Regular Nomination vs Aggressive Nomination

**文件**: `agent/conncheck.c:786` `priv_conn_check_tick_stream_nominate()`

两种提名模式（RFC 5245 sect 8.1.1）：

| 模式 | 行为 |
|---|---|
| `NICE_NOMINATION_MODE_REGULAR` | 收到有效候选对后，controlling agent 根据 stopping criterion 选择最佳候选对，重发一个带 USE-CANDIDATE 的检查 |
| `NICE_NOMINATION_MODE_AGGRESSIVE` | controlling agent 在每个发出的检查中都带上 USE-CANDIDATE |

### Stopping Criterion（停止准则）

Regular nomination 模式下，controlling agent 根据以下准则决定何时 nominate：

1. **首次提名场景**:
   - 有 valid host-host pair -> 立即停止
   - 或 valid pairs >= 2 -> 停止
   - 或无更多候选对可查 (frozen=0, waiting=0, in-progress=0) -> 停止

2. **同 component 已有提名候选对**: 选择相同传输类型的 valid pair

3. **同 stream 已有提名候选对**: 选择相同传输类型的 valid pair

选定后，设置 `pair->use_candidate_on_next_check = TRUE`，加入 triggered check queue。

### Controlled Agent 的提名

Controlled agent 不主动提名，而是在收到带 USE-CANDIDATE 的 incoming request 后：
- `priv_mark_pair_nominated()` 标记对应的候选对为 nominated
- 标记 `pair->mark_nominated_on_response_arrival = TRUE`

### priv_prune_pending_checks()

**文件**: `agent/conncheck.c:3015`

提名完成后，清理低优先级候选对：
- 移除所有优先级低于 selected_pair 的 FROZEN 和 WAITING 候选对
- 对优先级低于 selected_pair 的 IN_PROGRESS 候选对，禁用重传

---

## 完整的候选对状态机

```
                    ┌──────────┐
                    │  FROZEN  │──── priv_conn_check_unfreeze_next() ────┐
                    └──────────┘        (每 foundation 解冻一个)           │
                         │                                               │
         unfreeze_maybe │ (已有同 foundation SUCCEEDED 对)                │
                         ▼                                               ▼
                    ┌──────────┐                                  ┌──────────┐
    ────────────→  │  WAITING  │── priv_conn_check_initiate() →  │IN_PROGRESS│ ←── triggered check
                    └──────────┘     conn_check_send()            └─────┬────┘
                                                                       │
                                     ┌─────────────────────────────────┼─────────────────────────┐
                                     ▼                                 ▼                          ▼
                               ┌──────────┐                    ┌────────────┐              ┌──────────┐
                               │SUCCEEDED │                    │ DISCOVERED  │              │  FAILED  │
                               │ (valid)  │                    │ (peer-rflx) │              └──────────┘
                               └────┬─────┘                    └──────┬──────┘
                                    │                                │
                                    └──── nominated ─────────────────┘
```

状态说明：
- **FROZEN**: 等待同 foundation 的检查完成才被解冻
- **WAITING**: 已解冻，等待被选取发送
- **IN_PROGRESS**: 已发送 STUN Binding Request，等待响应
- **SUCCEEDED**: 收到有效响应，候选对有效（valid=TRUE）
- **DISCOVERED**: 通过 peer-reflexive candidate 发现的新候选对
- **FAILED**: 超时、地址不匹配、或收到错误响应

---

## 跨模块调用链

```
nice_agent_set_remote_candidates()                           (agent/agent.c)
  -> _set_remote_candidates_locked()                         (agent/agent.c)
    -> conn_check_remote_candidates_set()                    (agent/conncheck.c:1861)
      -> conn_check_add_for_candidate()                      (agent/conncheck.c:2495)
        -> conn_check_add_for_candidate_pair()               (agent/conncheck.c:2444)
          -> priv_conn_check_add_for_candidate_pair_matched()(agent/conncheck.c:2412)
            -> priv_add_new_check_pair()                     (agent/conncheck.c:2309)
              -> [FROZEN] -> priv_conn_check_unfreeze_maybe()(agent/conncheck.c:472)
            -> priv_schedule_next()                          (agent/conncheck.c:1722)
              -> timer_ta (20ms) -> priv_conn_check_tick_agent_locked()

  ─── 定时器循环 ───

priv_conn_check_tick_agent_locked()                          (agent/conncheck.c:1142)
  -> 1. priv_conn_check_triggered_check()                    (agent/conncheck.c:765)
       -> priv_conn_check_initiate()                         (agent/conncheck.c:332)
         -> conn_check_send()                                (agent/conncheck.c:2841)
           -> stun_usage_ice_conncheck_create()              (stun/usages/ice.c:62)
           -> stun_timer_start() / stun_timer_start_reliable()(stun/usages/timer.c)
           -> agent_socket_send()                            (agent/agent.c:7804)
             -> nice_socket_send()                           (socket/socket.c -> udp-bsd/tcp-bsd/udp-turn)

  -> 2. priv_conn_check_tick_stream()                        (agent/conncheck.c:633)
       -> stun_timer_refresh()                              (stun/usages/timer.c:141)
       -> TIMEOUT: priv_remove_stun_transaction()
       -> RETRANSMIT: agent_socket_send() retransmit

  -> 3. priv_conn_check_ordinary_check()                     (agent/conncheck.c:736)
       -> priv_conn_check_find_next_waiting()                (agent/conncheck.c:312)
       -> priv_conn_check_unfreeze_next()                    (agent/conncheck.c:365)
       -> priv_conn_check_initiate() + conn_check_send()

  -> 4. priv_conn_check_tick_stream_nominate()               (agent/conncheck.c:786)
       -> Regular / Aggressive nomination

  ─── 接收侧 ───

socket readable event
  -> nice_socket_recv_messages()                             (socket/socket.c)
  -> agent_recv_message_unlocked()                           (agent/agent.c:4682)
    -> STUN fast validate -> conn_check_handle_inbound_stun()(agent/conncheck.c:4532)
      -> stun_agent_validate()                               (stun/stunagent.c)
      -> [REQUEST]: stun_usage_ice_conncheck_create_reply()  (stun/usages/ice.c:236)
           -> priv_reply_to_conn_check()                     (agent/conncheck.c:3233)
           -> priv_schedule_triggered_check()                (agent/conncheck.c:3107)
           -> priv_mark_pair_nominated()                     (agent/conncheck.c:2201)

      -> [RESPONSE]: priv_map_reply_to_conn_check_request()  (agent/conncheck.c:3542)
           -> stun_usage_ice_conncheck_process()             (stun/usages/ice.c:142)
           -> priv_process_response_check_for_reflexive()    (agent/conncheck.c:3432)
             -> discovery_add_peer_reflexive_candidate()     (agent/discovery.c)
             -> priv_add_peer_reflexive_pair()               (agent/conncheck.c:3325)
           -> conn_check_unfreeze_related()                  (agent/conncheck.c:428)
           -> conn_check_update_check_list_state_for_ready()  (agent/conncheck.c:2152)
             -> priv_prune_pending_checks()                  (agent/conncheck.c:3015)
             -> agent_signal_component_state_change()         (agent/agent.c)
               -> READY / CONNECTED / CONNECTING / FAILED
```

---

## 关键数据结构

### CandidateCheckPair

```c
typedef struct {
    guint stream_id;
    guint component_id;
    NiceCandidate *local;
    NiceCandidate *remote;
    NiceSocket *sockptr;
    gchar foundation[NICE_CANDIDATE_PAIR_MAX_FOUNDATION];
    guint64 priority;
    NiceCheckState state;          // FROZEN/WAITING/IN_PROGRESS/SUCCEEDED/FAILED/DISCOVERED
    gboolean nominated;            // 是否已提名
    gboolean valid;                // 是否已验证通过
    GSList *stun_transactions;     // 正在进行的 STUN 事务列表
    CandidateCheckPair *discovered_pair;   // peer-reflexive discovered pair
    CandidateCheckPair *succeeded_pair;    // parent succeeded pair (for discovered)
    guint32 stun_priority;         // STUN PRIORITY 属性值
    gboolean use_candidate_on_next_check;
    gboolean mark_nominated_on_response_arrival;
    gboolean retransmit;
} CandidateCheckPair;
```

### StunTransaction

```c
typedef struct {
    StunTimer timer;
    StunMessage message;
    uint8_t buffer[STUN_MAX_MESSAGE_SIZE];
    gint64 next_tick;              // 下次检查时间 (microseconds)
} StunTransaction;
```

### IncomingCheck

暂存的入站检查，在 remote credentials 未就绪时使用：

```c
typedef struct {
    NiceAddress from;
    NiceSocket *local_socket;
    uint32_t priority;
    gboolean use_candidate;
    uint8_t *username;
    uint16_t username_len;
} IncomingCheck;
```

---

## 安全考虑

1. **源地址校验**: `priv_map_reply_to_conncheck_request()` 验证响应来自初始请求的目的地址，防止 off-path 攻击
2. **MESSAGE-INTEGRITY**: 所有 STUN 消息包含 HMAC-SHA1 完整性校验，防止篡改
3. **403 Forbidden**: RFC 7675 consent freshness -- 当 local consent 丢失时拒绝入站检查
4. **STUN Validation**: `conncheck_stun_validater()` 验证 USERNAME 和密码，防止未授权访问
5. **Remote Consent**: 通过 `priv_conn_remote_consent_tick_agent_locked()` 定期刷新远端同意
