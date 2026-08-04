# 09 -- 流程：提名与数据收发

## 概述

ICE 提名（Nomination）是 ICE 流程的最后一步：在所有连接性检查完成后，选出一个最终用于数据传输的候选对（candidate pair）。提名完成后，对应 component 进入 `NICE_COMPONENT_STATE_READY` 状态，应用可以开始发送和接收数据。

libnice 支持两种提名模式（RFC 5245 第 8.1.1 节）：

- **Regular Nomination（常规提名）**：控制方在连接性检查成功后，单独发送一次带有 `USE-CANDIDATE` 属性的 Binding Request 来提名一个有效对。被控方收到该请求后标记该对为 nominated。
- **Aggressive Nomination（激进提名）**：控制方在每个 Binding Request 中都携带 `USE-CANDIDATE` 属性，第一个成功的检查结果立即当选为 nominated，无需额外一轮握手。

两种模式的切换由 `agent->nomination_mode` 控制（`NICE_NOMINATION_MODE_REGULAR` 或 `NICE_NOMINATION_MODE_AGGRESSIVE`），默认情况下只对 RFC 5245 或 OC2007R2 兼容模式生效。

---

## 第一部分：提名流程

### 1.1 conncheck 定时器的提名阶段

提名逻辑是 conncheck 定时器回调 `conn_check_tick()` 的第三步。在 `agent/conncheck.c:1179-1186`，定时器每轮调度的顺序为：

1. `priv_conn_check_tick_stream()` -- 重传/重试超时的 STUN 事务
2. `priv_conn_check_ordinary_check()` -- 发起新一轮普通检查（从 frozen 队列取出）
3. `priv_conn_check_tick_stream_nominate()` -- 提名候选对

```c
/* step: try to nominate a pair */
for (i = agent->streams; i; i = i->next) {
    NiceStream *stream = i->data;
    if (priv_conn_check_tick_stream_nominate (agent, stream))
        keep_timer_going = TRUE;
}
```

### 1.2 Regular Nomination -- 控制方（Controlling）提名

`priv_conn_check_tick_stream_nominate()` (conncheck.c:787) 是控制方提名决策的核心。该函数遍历所有 stream 和 component，检查"停止准则"（stopping criterion）是否满足。停止准则基于 ICE 8.1.1.1 节的规定：

1. 该 component 至少有一个有效对（valid pair），且该有效对是 **HOST-HOST 类型**（即至少有一个 HOST-HOST valid pair），或
2. 该 component 有 **至少 2 个有效对**（`NICE_MIN_NUMBER_OF_VALID_PAIRS = 2`），或
3. 该 component **没有任何 frozen / waiting / in-progress 对**了，或
4. 在另一个 component 或另一个 stream 中已经找到了 nominated 对（触发整体提名）

当停止准则满足后，函数找到该 component 当前最佳的有效对（`this_component_pair`），设置 `use_candidate_on_next_check = TRUE`，将其加入 triggered check queue。下一次该对发起 conncheck 时，`conn_check_send()` 会在 Binding Request 中附带 `USE-CANDIDATE` 属性。

```c
this_component_pair->use_candidate_on_next_check = TRUE;
priv_add_pair_to_triggered_check_queue (agent, this_component_pair);
keep_timer_going = TRUE;
```

### 1.3 Aggressive Nomination -- 控制方提名

在激进模式下（`agent/conncheck.c:1087-1106`），停止准则被忽略。对于每个 component，直接选取优先级最高且状态为 `SUCCEEDED` 或 `DISCOVERED` 的对：

```c
p->nominated = TRUE;
conn_check_update_selected_pair (agent, component, p);
priv_add_pair_to_triggered_check_queue (agent, p);
```

该对立即被标记为 nominated，并更新 selected pair。后续的 conncheck 都会携带 `USE-CANDIDATE`。

### 1.4 conn_check_send() 中的 USE-CANDIDATE 添加

`conn_check_send()` (conncheck.c:2841) 负责构造和发送 Binding Request。在构建 STUN 消息前，根据 nomination_mode 决定 `cand_use` 的值：

```c
if (NICE_AGENT_IS_COMPATIBLE_WITH_RFC5245_OR_OC2007R2 (agent)) {
    switch (agent->nomination_mode) {
      case NICE_NOMINATION_MODE_REGULAR:
        cand_use = pair->use_candidate_on_next_check;  // 由提名逻辑设置
        break;
      case NICE_NOMINATION_MODE_AGGRESSIVE:
        cand_use = controlling;  // 只要自己是控制方就携带
        break;
    }
}
```

然后调用 `stun_usage_ice_conncheck_create()` 创建消息，将 `cand_use` 传入：

```c
buffer_len = stun_usage_ice_conncheck_create (&component->stun_agent,
    &stun->message, stun->buffer, sizeof(stun->buffer),
    uname, uname_len, password, password_len,
    cand_use, controlling, pair->stun_priority,
    agent->tie_breaker,
    pair->local->foundation,
    agent_to_ice_compatibility (agent));
```

消息构造好后通过 `agent_socket_send()` 或 `nice_socket_send()` 发送。

关于重传：非可靠传输（UDP）使用 `stun_timer_start()` 设定超时，可靠传输（TCP）使用 `stun_timer_start_reliable()`。

### 1.5 受控方（Controlled）响应提名

#### priv_conn_check_process_response() 处理 USE-CANDIDATE

当受控方收到 Binding Response（在 `priv_conn_check_process_response()` 中处理）时，代码位于 conncheck.c:3660-3712。如果 check pair 先前因 peer-reflexive 逻辑被"合并"到另一个 `ok_pair`，则根据条件决定是否标记 `ok_pair` 为 nominated：

```c
/* 控制方模式：根据提名模式标记 nominated */
if (agent->controlling_mode) {
    switch (agent->nomination_mode) {
      case NICE_NOMINATION_MODE_REGULAR:
        if (p->use_candidate_on_next_check)
            ok_pair->nominated = TRUE;
        break;
      case NICE_NOMINATION_MODE_AGGRESSIVE:
        if (!p->nominated)
            ok_pair->nominated = TRUE;
        break;
    }
} else {
    /* 受控方模式：检查 mark_nominated_on_response_arrival */
    if (p->mark_nominated_on_response_arrival) {
        ok_pair->nominated = TRUE;
    }
}
```

如果 `ok_pair->nominated == TRUE`，则：
1. 调用 `conn_check_update_selected_pair()` 更新 component 的 selected pair
2. 如果 component 状态还未到 READY，推进到 CONNECTED
3. 调用 `conn_check_update_check_list_state_for_ready()` 检查是否可以推进到 READY

#### priv_reply_to_conn_check() 处理 Incoming Request 的 USE-CANDIDATE

当受控方收到带 `USE-CANDIDATE` 的 Binding Request（在 `priv_reply_to_conn_check()` 中处理，conncheck.c:3233），代码逻辑如下：

```c
if (rcand && stream->remote_ufrag[0]) {
    priv_schedule_triggered_check (agent, stream, component, sockptr, rcand);
    if (use_candidate)
        priv_mark_pair_nominated (agent, stream, component, lcand, rcand);
}
```

即：收到带 `USE-CANDIDATE` 的请求时，立刻触发一次对方向检查（triggered check），并调用 `priv_mark_pair_nominated()` 标记该候选对。

### 1.6 priv_mark_pair_nominated()

`priv_mark_pair_nominated()` (conncheck.c:2201) 是受控方标记候选对为 nominated 的核心函数。关键逻辑：

1. **控制方提前返回**：如果自己是 controlling 模式，不处理（这是受控方的专属逻辑）。

```c
if (NICE_AGENT_IS_COMPATIBLE_WITH_RFC5245_OR_OC2007R2 (agent) &&
    agent->controlling_mode)
    return res;
```

2. **查找匹配对**：遍历 `stream->conncheck_list`，找到 `local == localcand && remote == remotecand` 的对。

3. **TCP/peer-reflexive 替换**：如果该对状态为 `SUCCEEDED` 且存在 `discovered_pair`（peer-reflexive 发现的），则替换为 `discovered_pair`。

4. **延迟标记**（RFC 5245, 7.2.1.5）：如果该对正在 triggered check queue 中或在 `IN_PROGRESS`，则设置 `mark_nominated_on_response_arrival = TRUE`，等待响应到达后再标记。

5. **立即标记**：如果该对已经是 valid，则直接标记 `nominated = TRUE`。

6. **更新 selected pair**：调用 `conn_check_update_selected_pair()` 将组件状态推进到 CONNECTING/CONNECTED。

7. **检查 READY**：如果 `nominated` 已设置，调用 `conn_check_update_check_list_state_for_ready()`。

### 1.7 Selected Pair 更新

#### conn_check_update_selected_pair()

`conn_check_update_selected_pair()` (conncheck.c:2038) 是连接检查层更新 selected pair 的入口：

```c
void conn_check_update_selected_pair (NiceAgent *agent, NiceComponent *component,
    CandidateCheckPair *pair)
{
    g_assert (pair->nominated);  // pair 必须是 nominated

    if (pair->priority > component->selected_pair.priority) {
        // 只有当新 pair 优先级更高时才替换
        cpair.local = (NiceCandidateImpl *) pair->local;
        cpair.remote = (NiceCandidateImpl *) pair->remote;
        cpair.priority = pair->priority;
        cpair.remote_consent.have = TRUE;

        nice_component_update_selected_pair (agent, component, &cpair);

        priv_conn_keepalive_tick_unlocked (agent);  // 重启 keepalive

        agent_signal_new_selected_pair (agent, pair->stream_id, component->id,
            pair->local, pair->remote);
    }
}
```

关键点：
- 只有当新 pair 的 priority 高于当前 selected pair 的 priority 时才替换。这保证了最高优先级对最终被选中。
- 替换后立即重启 keepalive 定时器，向新对发送 STUN keepalive。
- 发出 `new-selected-pair` 信号通知应用。

#### nice_component_update_selected_pair()

`nice_component_update_selected_pair()` (component.c:541) 执行底层的 selected pair 替换。除了复制 pair 信息外，还处理 TURN candidate 的生命周期管理：

```c
void nice_component_update_selected_pair (NiceAgent *agent,
    NiceComponent *component, const CandidatePair *pair)
{
    // 如果旧的 selected_pair.local 是 turn_candidate（上一个 ICE 会话的遗留）
    // 需要清理其 socket/prune 相关资源
    if (component->selected_pair.local &&
        component->selected_pair.local == component->turn_candidate) {
        discovery_prune_socket (agent, component->turn_candidate->sockptr);
        if (stream)
            conn_check_prune_socket (agent, stream, component,
                component->turn_candidate->sockptr);
        refresh_prune_candidate_async (agent, component->turn_candidate, ...);
        component->turn_candidate = NULL;
    }

    nice_component_clear_selected_pair (component);  // 销毁旧 consent tick source

    component->selected_pair.local = pair->local;
    component->selected_pair.remote = pair->remote;
    component->selected_pair.priority = pair->priority;
    component->selected_pair.stun_priority = pair->stun_priority;
    component->selected_pair.remote_consent.have = pair->remote_consent.have;

    nice_component_add_valid_candidate (agent, component,
        (NiceCandidate *) pair->remote);
}
```

### 1.8 READY 状态判定

`conn_check_update_check_list_state_for_ready()` (conncheck.c:2152) 判定 component 是否可以进入 READY 状态：

```c
void conn_check_update_check_list_state_for_ready (NiceAgent *agent,
    NiceStream *stream, NiceComponent *component)
{
    // 统计 nominated 对数量
    for (i = stream->conncheck_list; i; i = i->next) {
        CandidateCheckPair *p = i->data;
        if (p->component_id == component->id) {
            if (p->valid && p->nominated)
                ++nominated;
        }
    }

    if (nominated > 0) {
        // 需要等待所有 in-progress 检查完成
        if (priv_prune_pending_checks (agent, stream, component) == 0) {
            // 按顺序推进状态：CONNECTING -> CONNECTED -> READY
            if (component->state < NICE_COMPONENT_STATE_CONNECTING ||
                component->state == NICE_COMPONENT_STATE_FAILED)
                agent_signal_component_state_change (... CONNECTING);
            if (component->state < NICE_COMPONENT_STATE_CONNECTED)
                agent_signal_component_state_change (... CONNECTED);
            agent_signal_component_state_change (... READY);
        }
    }
}
```

进入 READY 的条件：
1. 至少有一个 nominated 有效对
2. 所有 in-progress 的检查都已结束（`priv_prune_pending_checks()` 返回 0）

---

## 第二部分：数据发送

### 2.1 nice_agent_send()

`nice_agent_send()` (agent.c:6030) 是基本的发送接口。它内部将参数封装为 `NiceOutputMessage`，然后调用内部实现：

```c
gint nice_agent_send (NiceAgent *agent, guint stream_id, guint component_id,
    guint len, const gchar *buf)
{
    GOutputVector local_buf = { buf, len };
    NiceOutputMessage local_message = { &local_buf, 1 };

    n_sent_bytes = nice_agent_send_messages_nonblocking_internal (agent,
        stream_id, component_id, &local_message, 1, TRUE, NULL);
    return n_sent_bytes;
}
```

### 2.2 nice_agent_send_messages_nonblocking_internal()

`nice_agent_send_messages_nonblocking_internal()` (agent.c:5794) 是发送的内部实现，支持批量发送（当 `allow_partial = FALSE` 时按消息计数，当 `allow_partial = TRUE` 时按字节计数）。

关键步骤：

1. **查找 component 和 stream**：
```c
if (!agent_find_component (agent, stream_id, component_id, &stream, &component)) {
    g_set_error (... "Invalid stream/component.");
    goto done;
}
```

2. **检查 Consent**：
```c
if (component->selected_pair.local != NULL &&
    !component->selected_pair.remote_consent.have) {
    g_set_error (... "Consent to send has been revoked by the peer");
    goto done;
}
```

3. **可靠模式（PseudoTCP）发送**：
```c
if (agent->reliable &&
    !nice_socket_is_reliable (component->selected_pair.local->sockptr)) {
    if (!pseudo_tcp_socket_is_closed (component->tcp)) {
        n_sent = pseudo_tcp_socket_send_messages (component->tcp, messages,
            n_messages, allow_partial, &child_error);
        adjust_tcp_clock (agent, stream, component);
        if (!pseudo_tcp_socket_can_send (component->tcp))
            g_cancellable_reset (component->tcp_writable_cancellable);
        if (n_sent < 0 && !g_error_matches (... WOULD_BLOCK))
            priv_pseudo_tcp_error (agent, component);
    }
}
```

数据先经过 `pseudo_tcp_socket_send_messages()` 进行分段和队列管理，然后由 PseudoTCP 内部的定时器逐个发出。`adjust_tcp_clock()` 调整伪 TCP 时钟间隔。当 PseudoTCP 发送窗口满时，重置 writable cancellable，以便后续通过 writable 事件恢复发送。

4. **非可靠模式直接发送**：
```c
else {
    NiceSocket *sock = component->selected_pair.local->sockptr;
    NiceAddress *addr = &component->selected_pair.remote->c.addr;

    if (nice_socket_is_reliable (sock)) {
        // ICE-TCP: 需要 RFC4571 帧格式封装
        for (i = 0; i < n_messages; i++) {
            // 构造带 2 字节长度前缀的 RFC4571 帧
            framed.header = htons (message_len);
            // ...
            n_sent_framed = nice_socket_send (sock, addr,
                message_len + sizeof (guint16), (const gchar *) framed_buf);
        }
    } else {
        // 直接 UDP 发送
        n_sent = nice_socket_send_messages (sock, addr, messages, n_messages,
            &child_error);
    }
}
```

关键：数据总是通过 `component->selected_pair.local->sockptr`（选定的本地 socket）发送到 `component->selected_pair.remote->c.addr`（选定的远程地址）。

### 2.3 nice_agent_send_messages_nonblocking()

公开的批量发送 API (agent.c:6004)，调用 `nice_agent_send_messages_nonblocking_internal()` 并设置 `allow_partial = FALSE`，表示必须完整发送每条消息。

```c
gint nice_agent_send_messages_nonblocking (NiceAgent *agent, ...)
{
    return nice_agent_send_messages_nonblocking_internal (agent,
        stream_id, component_id, messages, n_messages, FALSE, error);
}
```

### 2.4 可靠模式写入恢复

当 PseudoTCP 的发送窗口满（返回 `WOULD_BLOCK`）时，发送方注册对底层 socket 的可写回调。一旦底层 socket 变得可写，PseudoTCP 的刷新机制会将队列中的数据重新发出。这个过程由 `adjust_tcp_clock()` 和 `component->tcp_writable_cancellable` 协调完成。

---

## 第三部分：数据接收

libnice 提供两种接收模式：

1. **回调模式（通过 `nice_agent_attach_recv()`）**：由 GMainLoop 事件驱动，socket 可读时自动触发用户回调
2. **阻塞模式（通过 `nice_agent_recv_messages()`）**：线程主动拉取方式，适合非 GLib 应用

### 3.1 component_io_cb() -- I/O 事件的入口

`component_io_cb()` (agent.c:6275) 是附着在每个 socket 上的 `GSocket` 回调。当底层 UDP/TCP socket 可读时，GLib 主循环调度此回调。

```c
static gboolean component_io_cb (GSocket *gsocket, GIOCondition condition,
    gpointer user_data)
{
    SocketSource *socket_source = user_data;
    NiceComponent *component = socket_source->component;
    // ...
    agent = g_weak_ref_get (&component->agent_ref);
    agent_lock (agent);
    stream = agent_find_stream (agent, component->stream_id);
    // ...
}
```

**HUP 处理**：如果 socket 收到 `G_IO_HUP` 且没有可读数据，且该 socket 正是当前 selected pair 的 socket，则将 component 标记为 FAILED：

```c
if (condition & G_IO_HUP && !(condition & G_IO_IN)) {
    if (component->selected_pair.local &&
        component->selected_pair.local->sockptr == socket_source->socket &&
        component->state == NICE_COMPONENT_STATE_READY) {
        agent_signal_component_state_change (... FAILED);
    }
    nice_component_remove_socket (agent, component, socket_source->socket);
    return G_SOURCE_REMOVE;
}
```

**选择接收缓冲区**：根据接收模式选择不同的缓冲区策略：

```c
has_io_callback = nice_component_has_io_callback (component);

// 如果使用 attach_recv()，使用组件的 recv_buffer
// 如果使用 recv_messages()，使用用户提供的 buffer
```

然后根据传输类型分三种路径读取：

1. **可靠模式 + 不可靠 socket（伪 TCP）**：通过 `pseudo_tcp_socket_notify_message()` 处理
2. **可靠模式 + 可靠 socket（原生 TCP）**：通过 `agent_recv_message_unlocked()` 循环读取，按 RFC4571 帧解析
3. **非可靠模式（UDP）**：通过 `agent_recv_message_unlocked()` 循环读取

```c
while (has_io_callback || ...) {
    retval = agent_recv_message_unlocked (agent, stream, component,
        socket_source->socket, &local_message, NULL);
    if (retval == RECV_WOULD_BLOCK) break;
    if (retval == RECV_ERROR) { remove_source = TRUE; break; }
    has_io_callback = nice_component_has_io_callback (component);
}
```

### 3.2 agent_recv_message_unlocked() -- STUN 与数据的分流

`agent_recv_message_unlocked()` (agent.c:4682) 是接收路径的核心甄别函数。它读取一条消息后，首先判断是 STUN 控制消息还是用户数据：

1. **RFC4571 解帧**（ICE-TCP）：对可靠传输先按 RFC4571 帧头解析长度

2. **TURN 消息处理**：调用 `_agent_recv_turn_message_unlocked()` 检查是否来自 TURN 服务器，如果是则通过 `nice_udp_turn_socket_parse_recv_message()` 解包

3. **STUN 快速验证**：
```c
if (stun_message_validate_buffer_length_fast (...) == message->length) {
    // 可能是 STUN，紧凑化 buffer 后精确验证长度
    validated_len = stun_message_validate_buffer_length (big_buf, big_buf_len, ...);
    if (validated_len == big_buf_len) {
        handled = conn_check_handle_inbound_stun (agent, stream, component,
            nicesock, message->from, big_buf, big_buf_len);
        if (handled) {
            retval = RECV_OOB;  // STUN 消息在连接检查层处理
            goto done;
        }
    }
}
```

4. **未知来源过滤**：
```c
if (!nice_component_verify_remote_candidate (component, message->from, nicesock)) {
    retval = RECV_OOB;  // 丢弃来自非已验证对端的包
    goto done;
}
```

5. **用户数据**：通过了以上所有检查的消息被视为用户数据。在可靠模式下，会先经过 PseudoTCP 处理；否则直接返回给上层。

返回值语义：
- `RECV_SUCCESS` (1)：收到有效用户数据
- `RECV_OOB` (0)：数据被带外处理（STUN 消息、被过滤的包等）
- `RECV_WOULD_BLOCK` (-1)：暂无数据可读
- `RECV_ERROR` (-2)：错误

### 3.3 回调模式：nice_agent_attach_recv()

`nice_agent_attach_recv()` (agent.c:6639) 是旧版 API，内部封装为 `CompatRecvData`，然后调用 `nice_agent_attach_recv_ex()`。

`nice_agent_attach_recv_ex()` (agent.c:6655) 核心逻辑：

```c
nice_component_set_io_context (component, ctx);
nice_component_set_io_callback (component, func, data, notify, NULL, 0, NULL);
```

- `nice_component_set_io_context()`：分离旧 context 上的 socket source，附加到新 context
- `nice_component_set_io_callback()`：设置回调函数，同时清理 `recv_messages` 数组

当 socket 收到数据（经过 `component_io_cb()` -> `agent_recv_message_unlocked()` 识别为非 STUN 数据后），`component_io_cb()` 调用 `nice_component_emit_io_callback()` 将数据传递给用户回调。

#### nice_component_emit_io_callback()

(component.c:1004) 根据当前线程是否拥有 component 的 GMainContext 决定同步或异步递送：

- **快路径**（线程拥有 context）：直接调用用户回调
```c
if (g_main_context_is_owner (component->ctx)) {
    agent_unlock_and_emit (agent);
    io_callback (agent, stream_id, component_id, buf_len,
        component->recv_buffer, &component->exdata, io_user_data);
    agent_lock (agent);
}
```

- **慢路径**（线程不拥有 context）：通过 idle source 异步递送
```c
else {
    data = io_callback_data_new (component->recv_buffer, buf_len, &component->exdata);
    g_queue_push_tail (&component->pending_io_messages, data);
    nice_component_schedule_io_callback (component);
}
```

Idle 回调 `emit_io_callback_cb()` 在 component 的 context 中执行，从 `pending_io_messages` 队列取出数据逐个传给用户回调。

### 3.4 阻塞模式：nice_agent_recv_messages()

`nice_agent_recv_messages()` (agent.c:5708) 和 `nice_agent_recv_messages_blocking_or_nonblocking()` (agent.c:5466) 实现阻塞接收。

关键初始化代码：

```c
// 设置 recv_messages 缓冲区，清除 io_callback
nice_component_set_io_callback (component, NULL, NULL, NULL,
    messages, n_messages, &child_error);
```

此调用将组件的 `io_callback` 设为 NULL（停止回调递送），同时将用户提供的 `messages` 数组注册为接收缓冲区。

然后进入阻塞循环：
- 阻塞模式：通过 `g_main_context_iteration()` / `g_main_loop_run()` 在主循环中等待数据
- 若提供了 `GCancellable`，则附加 cancellable source 以支持取消

数据到达时经过 `component_io_cb()` -> `agent_recv_message_unlocked()`，STUN 消息被过滤，用户数据写入 `component->recv_messages` 数组。

```c
while (!received_enough && ...) {
    // 检查 pending_io_messages（从回调模式切换过来的遗留数据）
    pending_io_messages_recv_messages (component, ...);

    // 检查 recv_messages_iter 是否有新数据
    received_enough = (nice_input_message_iter_get_n_valid_messages (...) > 0);
}
```

### 3.5 nice_agent_recv_messages_nonblocking()

`nice_agent_recv_messages_nonblocking()` (agent.c:5748) 是非阻塞版本，设置 `blocking = FALSE`：

```c
return nice_agent_recv_messages_blocking_or_nonblocking (agent, stream_id,
    component_id, FALSE, messages, n_messages, cancellable, error);
```

非阻塞模式下，函数读取一次后立即返回。如果当时没有数据，返回 `RECV_WOULD_BLOCK` 对应的错误 `G_IO_ERROR_WOULD_BLOCK`。

### 3.6 兼容 API

`nice_agent_recv()` 和 `nice_agent_recv_nonblocking()` 是单缓冲区包装，内部构造 `NiceInputMessage` 后调用对应的 `nice_agent_recv_messages*()`：

```c
gssize nice_agent_recv (NiceAgent *agent, guint stream_id, guint component_id,
    guint8 *buf, gsize buf_len, GCancellable *cancellable, GError **error)
{
    GInputVector local_bufs = { buf, buf_len };
    NiceInputMessage local_messages = { &local_bufs, 1, NULL, 0 };

    n_valid_messages = nice_agent_recv_messages (agent, stream_id, component_id,
        &local_messages, 1, cancellable, error);
    // ...
    return local_messages.length;
}
```

---

## 数据路径对比

| 特性 | 回调模式 (attach_recv) | 阻塞模式 (recv_messages) |
|------|----------------------|------------------------|
| 驱动方式 | GMainLoop 事件驱动 | 线程主动拉取 |
| STUN 过滤 | 自动过滤（`RECV_OOB`） | 自动过滤（`RECV_OOB`） |
| 线程安全 | GMainContext 保护 | agent_lock() + io_mutex 保护 |
| 缓冲区 | `component->recv_buffer`（内置 65535 字节） | 调用者提供的 `NiceInputMessage[]` |
| 数据递送 | `io_callback` 直接或通过 idle source | 写入 `recv_messages` 数组 |
| 适用场景 | GLib / GStreamer 应用 | 非 GLib 应用、同步 I/O 场景 |
| 连接管理 | socket source 自动处理 EOS | `G_IO_ERROR_BROKEN_PIPE` 错误码 |

---

## 跨模块调用链

### 提名链

```
conn_check_tick() (conncheck.c:1145)
  └── priv_conn_check_tick_stream_nominate() (conncheck.c:787)
        │    [Regular: 停止准则 → use_candidate_on_next_check]
        │    [Aggressive: 立即 nominated = TRUE]
        │
        ├── priv_add_pair_to_triggered_check_queue() (conncheck.c)
        │     └── conn_check_send() (conncheck.c:2841)
        │           └── stun_usage_ice_conncheck_create() (stun/usages/ice.c)
        │                 └── agent_socket_send() (conncheck.c)
        │                       └── nice_socket_send() (socket/)
        │
        └── [对端收到 STUN 响应]
              └── priv_conn_check_process_response() (conncheck.c:3550)
                    └── ok_pair->nominated = TRUE
                          └── conn_check_update_selected_pair() (conncheck.c:2038)
                                └── nice_component_update_selected_pair() (component.c:541)
                                      └── agent_signal_new_selected_pair() (agent.c)
                                └── conn_check_update_check_list_state_for_ready() (conncheck.c:2152)
                                      └── agent_signal_component_state_change() → READY

  [受控方收到带 USE-CANDIDATE 的 Request]
  └── priv_reply_to_conn_check() (conncheck.c:3233)
        └── priv_mark_pair_nominated() (conncheck.c:2201)
              ├── pair->nominated = TRUE (或 mark_nominated_on_response_arrival)
              ├── conn_check_update_selected_pair() → CONNECTED
              └── conn_check_update_check_list_state_for_ready() → READY
```

### 发送链

```
nice_agent_send() (agent.c:6030)
nice_agent_send_messages_nonblocking() (agent.c:6004)
  └── nice_agent_send_messages_nonblocking_internal() (agent.c:5794)
        ├── [检查 selected_pair 和 consent]
        ├── [可靠模式]
        │     └── pseudo_tcp_socket_send_messages() (pseudotcp.c)
        │           └── [分段/队列管理]
        │                 └── [定时刷新] → nice_socket_send()
        │
        └── [非可靠模式]
              ├── [ICE-TCP: RFC4571 封帧]
              │     └── nice_socket_send() socket/ → tcp-bsd.c
              │
              └── [UDP: 直接发送]
                    └── nice_socket_send_messages() socket/ → udp-bsd.c 或 udp-turn.c
```

### 接收链

```
socket 可读事件 (GMainLoop)
  └── component_io_cb() (agent.c:6275)
        ├── [HUP 检测 → FAILED]
        ├── [选择缓冲区: recv_buffer 或 recv_messages]
        │
        └── agent_recv_message_unlocked() (agent.c:4682)
              ├── [RFC4571 解帧] (ICE-TCP)
              ├── [TURN 解包] (_agent_recv_turn_message_unlocked)
              ├── [STUN 检测] → stun_message_validate_buffer_length_*()
              │     └── conn_check_handle_inbound_stun() → 内部处理
              │           ├── conncheck / discovery handler
              │           └── RECV_OOB (返回给调用者表示已处理)
              │
              ├── [来源验证] → nice_component_verify_remote_candidate()
              │     └── RECV_OOB (拒绝未知来源)
              │
              └── [用户数据]
                    ├── [可靠模式: pseudo_tcp_socket_notify_message()]
                    │
                    ├── [回调模式]
                    │     └── nice_component_emit_io_callback() (component.c:1004)
                    │           ├── [快路径] → 直接调用 io_callback
                    │           └── [慢路径] → idle source → emit_io_callback_cb()
                    │
                    └── [阻塞模式]
                          └── 写入 component->recv_messages[]
                                └── nice_agent_recv_messages_blocking_or_nonblocking()
                                      返回给调用者
```
