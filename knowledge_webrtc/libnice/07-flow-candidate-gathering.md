# 07 -- 流程：候选收集

## 概述

ICE 候选收集（Candidate Gathering）是 ICE 协议的第一阶段，目的是发现本机所有可能的网络地址（候选）。libnice 的候选收集流程分为三个顺序阶段：

1. **Host 候选** -- 收集本机网卡上的私有/公网 IP 地址
2. **Server Reflexive (srflx) 候选** -- 通过 STUN Binding 请求从 STUN 服务器获取 NAT 映射后的公网地址
3. **Relay 候选** -- 通过 TURN Allocate 请求从 TURN 服务器获取中继地址

总体流程从 `nice_agent_gather_candidates()` 开始，到 `"candidate-gathering-done"` 信号发射结束。在三个主要阶段之间，还可能插入 UPnP IGD 端口映射进行 NAT 穿透。

核心数据结构是 `CandidateDiscovery`（定义在 `agent/discovery.h`），代理维护一个全局链表 `agent->discovery_list`。每个 `CandidateDiscovery` 项表示一个待发送或已发送的 STUN 请求（Binding 或 Allocate），其中包含：

- `type` -- `NICE_CANDIDATE_TYPE_SERVER_REFLEXIVE` 或 `NICE_CANDIDATE_TYPE_RELAYED`
- `nicesock` -- 发送 STUN 请求使用的底层 socket
- `server` -- STUN/TURN 服务器地址
- `stun_agent` + `stun_message` + `stun_buffer` -- STUN 事务上下文
- `timer` + `next_tick` -- 重传定时器及下次 tick 时间戳
- `pending` -- 是否已开始（已发送首包）/ `done` -- 是否已完成或放弃

## 触发点

### nice_agent_gather_candidates() (`agent/agent.c:3753`)

```c
nice_agent_gather_candidates (NiceAgent *agent, guint stream_id);
```

入口函数的执行流程：

1. **参数验证** -- 检查 `agent` 非空、`stream_id >= 1`，通过 `agent_find_stream()` 查找目标流
2. **重复调用检查** -- 若 `stream->gathering_started` 已为 TRUE，直接返回 TRUE（幂等）
3. **本地地址收集** -- 若用户未通过 `nice_agent_add_local_address()` 手动指定地址，则调用 `nice_interfaces_get_local_ips(FALSE)` 自动枚举所有网卡 IP
4. **STUN 服务器 DNS 解析** -- 若处于 full mode 且配置了 `stun_server_ip`（非强制 relay 模式），则异步解析 STUN 服务器域名（`resolve_stun_in_context`）
5. **Host 候选创建** -- 对每个本地 IP 地址、每个流中的 component，调用 `discovery_add_local_host_candidate()` 创建 host 候选（UDP、TCP-ACTIVE、TCP-PASSIVE）
6. **TURN 发现项注册** -- 对每个成功创建的 host 候选，遍历 component 的 TURN 服务器列表，调用 `priv_add_new_candidate_discovery_turn()` 向 `discovery_list` 中追加 relay 类型的发现项
7. **STUN 发现项注册** -- DNS 解析完成后，在 `stun_server_resolved_cb()` 中调用 `priv_add_new_candidate_discovery_stun()` 追加 srflx 类型的发现项
8. **信号发射** -- 收集完毕后发射 `"new-candidate"` 信号通知每个 host 候选
9. **调度/完成判定** -- 若无待调度的发现项、无 DNS 解析中、无 TURN 解析中、无 UPnP 映射，则直接调用 `agent_gathering_done()`；否则调用 `discovery_schedule()` 启动定时器

## 阶段一：Host 候选收集

### 1. 网络接口枚举 (`agent/interfaces.c`)

`nice_interfaces_get_local_ips(gboolean include_loopback)` 负责枚举本机所有有效的 IPv4/IPv6 地址。

**Unix 平台**（有 `getifaddrs`）：
- 调用 `getifaddrs()` 获取所有网络接口信息
- 过滤规则：接口必须为 `IFF_UP` 且 `IFF_RUNNING`；跳过 `AF_INET`/`AF_INET6` 以外的地址族
- macOS 特殊处理：跳过 `awdl`/`llw` 前缀的接口（Apple Wireless Direct Link），跳过以 `fe80::` 开头的 `utun` 接口
- 有 `net/if_media.h` 时：通过 `SIOCGIFMEDIA` ioctl 检查接口是否活跃
- 回环接口：仅当 `include_loopback=TRUE` 时加入结果列表末尾
- 排序：私有 IP 地址追加（`g_list_append`）在列表末尾，公网 IP 地址前插（`g_list_prepend`）在列表头部 -- 公网地址优先

**Unix 平台**（无 `getifaddrs`，fallback）：
- 通过 socket + `ioctl(SIOCGIFCONF)` 获取接口配置列表
- 相同的过滤/排序逻辑

**Windows 平台**：
- 调用 `GetAdaptersAddresses()` 获取适配器列表
- 过滤：`IP_ADAPTER_RECEIVE_ONLY`、`IfOperStatusDown`/`NotPresent`/`LowerLayerDown` 被跳过
- 通过 `GetBestInterfaceEx()` 获取默认路由接口，将其放在列表头部

### 2. discovery_add_local_host_candidate() (`agent/discovery.c:762`)

```c
HostCandidateResult discovery_add_local_host_candidate(
    NiceAgent *agent, guint stream_id, guint component_id,
    NiceAddress *address, NiceCandidateTransport transport,
    gboolean accept_duplicate, NiceCandidateImpl **outcandidate);
```

步骤：

1. **Socket 创建** -- 根据 transport 类型创建底层 socket：
   - UDP: `nice_udp_bsd_socket_new()` (`socket/udp-bsd.c`)
   - TCP-ACTIVE: `nice_tcp_active_socket_new()`
   - TCP-PASSIVE: `nice_tcp_passive_socket_new()`
2. **候选对象构造** -- `nice_candidate_new(NICE_CANDIDATE_TYPE_HOST)` 创建候选
   - 设置 `transport`、`stream_id`、`component_id`、`addr`、`base_addr`
3. **优先级计算** -- 根据兼容模式调用不同的优先级函数：
   - Google (Jingle): `nice_candidate_jingle_priority()`
   - MSN/OC2007: `nice_candidate_msn_priority()`
   - OC2007R2: `nice_candidate_ms_ice_priority()`
   - RFC 5245/8445: `nice_candidate_ice_priority()`（含 type pref、local pref、component ID）
4. **凭证生成** (`priv_generate_candidate_credentials`):
   - MSN/OC2007 兼容模式：随机生成 32 字节 username + 16 字节 password，Base64 编码
   - Google 兼容模式：随机生成 16 字符 username，password 为 NULL
   - 其他模式：username/password 保持 NULL（使用 stream 级别的 ufrag/password）
5. **Foundation 分配** (`priv_assign_foundation`):
   - 遍历已存在的所有 local candidates，寻找相同 type + transport + base address（非 relay 类型还要求 stun server 相同）的候选
   - 找到则继承其 foundation；否则分配新的递增 ID
6. **Sockaddr 更新** -- `candidate->addr = nicesock->addr`（取 bind 后实际分配的 socket 地址，因为端口可能由 OS 自动分配）
7. **端口去重** -- `priv_local_host_candidate_duplicate_port()` 检查是否与其他 stream/component 中的候选端口冲突
8. **冗余裁剪** -- `priv_add_local_candidate_pruned()` 按 ICE 4.1.3 节去重：
   - 相同的 `base_addr` + `addr` + `transport` 视为冗余
   - 相同 IP（忽略端口）的同类型 relay / srflx 候选视为冗余
9. **Socket 注册** -- `nice_component_attach_socket()` 将 socket 注册到 component，使其在 main loop 轮询中接收数据
10. **设置 TOS** -- `_priv_set_socket_tos()` 配置 socket 的 DSCP/TOS 字段

### 3. 端口分配策略 (`agent/agent.c:3881-3915`)

端口分配采用循环试凑策略：

- **指定范围** (`component->min_port != 0`)：在 `[min_port, max_port]` 内随机选择起始端口，循环递增直到成功或回到起始点
- **随机端口** (`component->min_port == 0`)：`current_port = 0`，由 OS 自动分配（仅尝试一次）
- **失败处理**：`CANT_CREATE_SOCKET` 或 `DUPLICATE_PORT` 时递增端口重试；回到起始端口后设置 `accept_duplicate=TRUE` 允许端口复用（含警告日志）

## 阶段二：Server Reflexive 候选收集

### 1. discovery_schedule() 启动定时器 (`agent/discovery.c:1462`)

```c
void discovery_schedule (NiceAgent *agent);
```

前提条件：`agent->discovery_list != NULL` 且 `agent->discovery_unsched_items > 0`（有尚未调度的发现项）。

- 第一次调用时立即执行一次 `priv_discovery_tick_unlocked(agent)`
- 若还有未完成项，创建 GLib 定时器源，以 `agent->timer_ta`（默认 20ms，`NICE_AGENT_TIMER_TA_DEFAULT`）为间隔重复调用 `priv_discovery_tick_agent_locked()`

### 2. discovery_tick() 推进 (`agent/discovery.c:1262`)

`priv_discovery_tick_unlocked()` 是候选收集的核心驱动函数：

1. 遍历 `agent->discovery_list`，对每个 `pending == FALSE` 的项：
   - 标记 `pending = TRUE`，递减 `discovery_unsched_items`
   - 根据 `type` 构造 STUN 请求：
     - **SRFLX**: `stun_usage_bind_create()` (`stun/usages/bind.c`)
     - **RELAY**: `stun_usage_turn_create()` (`stun/usages/turn.c`)
   - 通过 `agent_socket_send()` 发送请求
   - 成功则启动 `StunTimer`（不可靠 socket 使用指数退避重传；可靠 socket 使用固定超时 `stun_reliable_timeout`）
   - 失败则标记 `done = TRUE`，跳过
   - 设置 `need_pacing` 标志，本 tick 只启动一个发现项（Ta 间隔 = 20ms 的 pacing）

2. 对已完成发送的项（`done != TRUE`），检查定时器状态：
   - **TIMEOUT**: 调用 `stun_agent_forget_transaction()` 释放事务，标记 `done = TRUE`
   - **RETRANSMIT**: 通过 `agent_socket_send()` 重传 STUN 请求，设置下一次 tick 时间
   - **SUCCESS**: 等待下一次 tick

3. 当 `not_done == 0`（所有发现项均已完成或超时）：
   - 调用 `discovery_free()` 停止定时器
   - 调用 `agent_gathering_done()` 进入完成流程

### 3. STUN Binding 请求构造 (`stun/usages/bind.c:88`)

```c
size_t stun_usage_bind_create(StunAgent *agent, StunMessage *msg,
    uint8_t *buffer, size_t buffer_len);
```

最简单的 STUN 用法函数：
1. `stun_agent_init_request()` -- 初始化为 STUN_BINDING 方法、Request 类型
2. `stun_agent_finish_message()` -- 无需额外属性，直接序列化为线格式（28 字节头：类型 + 长度 + Magic Cookie + Transaction ID）

### 4. STUN Binding 响应处理 (`stun/usages/bind.c:96`)

```c
StunUsageBindReturn stun_usage_bind_process(StunMessage *msg,
    struct sockaddr *addr, socklen_t *addrlen,
    struct sockaddr *alternate_server, socklen_t *alternate_server_len);
```

处理流程：
1. 验证消息方法为 `STUN_BINDING`，类型为 Response/Error
2. Error 响应：提取 `ERROR-CODE`，若为 3xx 则解析 `ALTERNATE-SERVER` 属性，返回 `STUN_USAGE_BIND_RETURN_ALTERNATE_SERVER`
3. Success 响应：优先提取 `XOR-MAPPED-ADDRESS`（RFC 5389），失败则 fallback 到 `MAPPED-ADDRESS`（RFC 3489）
4. 将解析的 sockaddr 写入 `addr` 参数，返回 `STUN_USAGE_BIND_RETURN_SUCCESS`

### 5. 响应匹配与候选创建 (`agent/conncheck.c:3770`)

响应从 `conn_check_handle_inbound_stun()` 进入，经由 `priv_map_reply_to_discovery_request()` 匹配：

```c
static gboolean priv_map_reply_to_discovery_request(NiceAgent *agent,
    StunMessage *resp, const NiceAddress *server_address);
```

1. 遍历 `discovery_list`，找到 type 为 `SERVER_REFLEXIVE` 且 Transaction ID 匹配的 `CandidateDiscovery` 项
2. 调用 `stun_usage_bind_process()` 提取映射地址
3. **ALTERNATE_SERVER**: 更新 `d->server`，重新调度（`pending = FALSE`）
4. **SUCCESS**: 调用 `discovery_add_server_reflexive_candidate()` 创建 srflx 候选
   - 若非 `force_relay` 模式，也调用 `discovery_discover_tcp_server_reflexive_candidates()` 为已有 TCP host 候选创建对应的 TCP srflx 候选
   - 标记 `done = TRUE`

### 6. discovery_add_server_reflexive_candidate() (`agent/discovery.c:861`)

```c
void discovery_add_server_reflexive_candidate(NiceAgent *agent,
    guint stream_id, guint component_id, NiceAddress *address,
    NiceCandidateTransport transport, NiceSocket *base_socket,
    const NiceAddress *server_address, gboolean nat_assisted);
```

- 创建 `NICE_CANDIDATE_TYPE_SERVER_REFLEXIVE` 候选
- `base_addr` = base_socket 的地址（即 host 候选的地址）
- `sockptr` = base_socket（与 host 候选复用同一个 socket）
- 计算优先级（`nat_assisted` 参数用于 OC2007R2 兼容模式的本地偏好值调整）
- 去重后发射 `"new-candidate"` 信号

## 阶段三：Relay 候选收集

### 1. TURN Allocate 请求构造 (`stun/usages/turn.c:71`)

```c
size_t stun_usage_turn_create(StunAgent *agent, StunMessage *msg,
    uint8_t *buffer, size_t buffer_len,
    StunMessage *previous_response,
    StunUsageTurnRequestPorts request_props,
    int32_t bandwidth, int32_t lifetime,
    uint8_t *username, size_t username_len,
    uint8_t *password, size_t password_len,
    StunUsageTurnCompatibility compatibility);
```

构造步骤：

1. 初始化为 `STUN_ALLOCATE` 方法的 Request 消息
2. 根据兼容模式添加属性：
   - **DRAFT9/RFC5766**: 添加 `REQUESTED-TRANSPORT`（值 `0x11000000` = UDP），可选 `BANDWIDTH`
   - **Google**: 添加 `MAGIC-COOKIE`（`TURN_MAGIC_COOKIE`）
   - **OC2007**: 额外添加 `MS_VERSION` = 1
3. 添加 `LIFETIME` 属性（若 lifetime >= 0）
4. 添加 `REQUESTED-PORT-PROPS`（DRAFT9/RFC5766，仅 EVEN/EVEN_AND_RESERVE 模式）
5. 若为重新认证（`previous_response != NULL`），从之前响应中提取 `REALM`、`NONCE`、`RESERVATION-TOKEN` 并附加
6. 添加 `USERNAME`（若使用短期凭证或有 previous_response）
7. `stun_agent_finish_message()` -- 序列化，使用密码计算消息完整性（MESSAGE-INTEGRITY）

### 2. discovery_tick() 中的 TURN 发送

与 srflx 在同一 tick 循环中处理，区别在于 `type == NICE_CANDIDATE_TYPE_RELAYED` 时调用 `stun_usage_turn_create()` 构造 Allocate 请求。

注意：Google 兼容模式下，TURN 候选使用一个独立的新 UDP socket（不与 host 候选共享），以便在 socket 级别区分来自 TURN 服务器和直接来自 peer 的流量。

### 3. TURN Allocate 响应处理 (`stun/usages/turn.c:272`)

```c
StunUsageTurnReturn stun_usage_turn_process(StunMessage *msg,
    struct sockaddr_storage *relay_addr, socklen_t *relay_addrlen,
    struct sockaddr_storage *addr, socklen_t *addrlen,
    struct sockaddr_storage *alternate_server, socklen_t *alternate_server_len,
    uint32_t *bandwidth, uint32_t *lifetime,
    StunUsageTurnCompatibility compatibility);
```

处理流程：

1. 验证消息方法为 `STUN_ALLOCATE`，类型为 Response/Error
2. Error 响应：提取 `ERROR-CODE`；3xx 时提取 `ALTERNATE-SERVER`
3. Success 响应 -- 根据兼容模式解析中继地址：
   - **DRAFT9/RFC5766**: 提取 `XOR-RELAYED-ADDRESS`（作为 relay_addr），可选提取 `XOR-MAPPED-ADDRESS`（作为 mapped addr）
   - **Google**: 提取 `MAPPED-ADDRESS`（作为 relay_addr，Google TURN 不规范地将 MAPPED-ADDRESS 用作 Relay 地址）
   - **MSN**: 提取 `MSN-MAPPED-ADDRESS`（自定义属性 `0x8000`）
   - **OC2007**: 提取 `MS-XOR-MAPPED-ADDRESS`（使用 Transaction ID 的第一个 u32 作为 XOR magic），提取 `MAPPED-ADDRESS`（作为 relay_addr）
4. 提取 `LIFETIME`、`BANDWIDTH` 属性
5. 返回 `STUN_USAGE_TURN_RETURN_RELAY_SUCCESS` 或 `STUN_USAGE_TURN_RETURN_MAPPED_SUCCESS`（同时获得了映射地址）或 `STUN_USAGE_TURN_RETURN_ALTERNATE_SERVER`

### 4. 响应匹配与候选创建 (`agent/conncheck.c:3991`)

`priv_map_reply_to_relay_request()` 匹配 relay 类型的发现项并处理响应：

1. **ALTERNATE_SERVER**: 调用 `priv_handle_turn_alternate_server()` 切换服务器
2. **RELAY_SUCCESS / MAPPED_SUCCESS**:
   - 若收到 MAPPED_ADDRESS（即 `MAPPED_SUCCESS`），且 TURN 类型为 UDP：
     - 调用 `discovery_add_server_reflexive_candidate()` 创建 srflx 候选（利用 TURN 响应中的映射地址，节省一次独立的 STUN Binding）
     - 若启用 ICE-TCP，也创建对应 TCP srflx 候选
   - 根据 socket 可靠性创建 relay 候选：
     - 可靠 socket（TCP TURN）：创建 `TCP-ACTIVE` 和 `TCP-PASSIVE` 两种类型的 relay 候选
     - 不可靠 socket（UDP TURN）：创建 `UDP` 类型的 relay 候选
   - 调用 `discovery_add_relay_candidate()` 创建 relay 候选
   - 调用 `priv_add_new_turn_refresh()` 创建 `CandidateRefresh` 项，用于后续定期刷新 TURN 分配

### 5. discovery_add_relay_candidate() (`agent/discovery.c:976`)

```c
NiceCandidateImpl *discovery_add_relay_candidate(NiceAgent *agent,
    guint stream_id, guint component_id, NiceAddress *address,
    NiceCandidateTransport transport, NiceSocket *base_socket,
    TurnServer *turn, uint32_t *lifetime);
```

- 创建 `NICE_CANDIDATE_TYPE_RELAYED` 候选
- 通过 `nice_udp_turn_socket_new()` 创建一个新的 TURN socket（封装了 TURN 服务器的地址和认证凭证），该 socket 会用 TURN Send/Data indication 进行实际数据收发
- `sockptr` = 新创建的 relay_socket（不是 base_socket）
- `base_addr` = base_socket 的地址
- `turn` 指针由 `turn_server_ref()` 增加引用计数存储
- Google 兼容模式：candidate username 直接使用 TURN 服务器的 username
- 调用 `priv_add_local_candidate_pruned()` 去重
- 调用 `nice_component_attach_socket()` 将 relay socket 注册到 component

### 6. TURN Refresh 定时器 (`agent/discovery.c:2970-2967`)

分配成功后调用 `priv_add_new_turn_refresh()` 创建 `CandidateRefresh` 结构：

- 复制 STUN agent、服务器地址、认证信息等
- 保存 Allocate success response 的 buffer（`stun_resp_msg`），用于 refresh 请求中的 REALM/NONCE 认证
- 启动 GLib 定时器（超时 = `lifetime - 60` 秒，最少 `lifetime / 2`），到期后发送 Refresh 请求
- Refresh 请求通过 `stun_usage_turn_create_refresh()` 构造（DRAFT9/RFC5766 使用 `STUN_REFRESH` 方法，旧版本仍使用 `STUN_ALLOCATE` 并携带 zero port 特征）

### 7. TURN Deallocate (`agent/discovery.c:221-292`)

当 agent 关闭或 stream 被移除时，调用 `refresh_remove_async()` 发送 lifetime=0 的 Refresh/Allocate 请求释放 TURN 服务器上的端口分配。若 `agent->close_forced`，则发送两次后立即释放资源；否则正常重传直到超时。

## 状态机

```
                        +-------------+
                        | NOT_STARTED |
                        +------+------+
                               |
                     nice_agent_gather_candidates()
                               |
                               v
                     +---------+---------+
                     | GATHERING_HOST    |  (immediate in gather_candidates)
                     | - 枚举本地 IP     |
                     | - 创建 host 候选   |
                     | - 注册 discovery   |
                     |   items            |
                     +---------+---------+
                               |
                     discovery_schedule()
                               |
                  +------------v------------+
                  | GATHERING_SRFLX/RELAY   |
                  | (ta=20ms pacing,         |
                  |  discovery_tick 驱动)     |
                  | - 发送 Bind/Allocate     |
                  | - 接收响应               |
                  | - 创建 srflx/relay 候选  |
                  +------------+------------+
                               |
                     not_done == 0
                    (全部完成或超时)
                               |
                               v
                     +---------+---------+
                     |       DONE        |
                     | (发射 gathering-  |
                     |  done 信号)        |
                     +-------------------+
```

注意：
- 若无 STUN 服务器配置（或设置了 `force_relay`），srflx 阶段被跳过
- 若无 TURN 服务器配置，relay 阶段被跳过
- UPnP IGD 映射（`priv_add_upnp_discovery`）时间线与 discovery tick 并行，完成后同样调用 `discovery_add_server_reflexive_candidate()` 添加候选
- DNS 解析（STUN 域名、TURN 域名）是异步的，在解析完成前 discovery tick 不会包含对应的发现项

## 关键定时器

| 定时器 | 默认值 | 用途 |
|--------|--------|------|
| `timer_ta` | 20ms (`NICE_AGENT_TIMER_TA_DEFAULT`) | discovery tick 间隔（pacing），每个 tick 最多启动一个发现项 |
| `stun_initial_timeout` | 500ms | STUN 请求首次重传超时 |
| `stun_max_retransmissions` | 7 | STUN 请求最大重传次数 |
| `stun_reliable_timeout` | 默认值（config.h） | 可靠传输（TCP）上的 STUN 固定超时 |
| TURN refresh interval | `lifetime - 60` 秒 | TURN allocation 续期定时器 |

STUN 重传采用指数退避（RFC 5389）：首次 RTO = 500ms，每次重传翻倍直至到达最大重传次数（7 次），总超时约 63.5 秒。

## 信号

### new-candidate

每次发现新候选时发射（包括 host、srflx、relay、peer-rflx）。

```c
/* "new-candidate" 信号 */
agent_queue_signal(agent, signals[SIGNAL_NEW_CANDIDATE],
    candidate->stream_id, candidate->component_id, candidate->foundation);
/* "new-candidate-full" 信号（传递完整 NiceCandidate 对象）*/
agent_queue_signal(agent, signals[SIGNAL_NEW_CANDIDATE_FULL], candidate);
```

- **发射时机**：`agent_signal_new_candidate()` 在候选被成功添加并去重通过后调用
- host 候选在 `nice_agent_gather_candidates()` 末尾批量发射
- srflx/relay 候选在 discovery tick 收到响应后逐个发射

### candidate-gathering-done

所有候选收集完成时发射。

```c
/* "candidate-gathering-done" 信号 */
agent_queue_signal(agent, signals[SIGNAL_CANDIDATE_GATHERING_DONE], stream->id);
```

- **触发条件** (`agent_gathering_done()` -> `agent_signal_gathering_done()`):
  - `agent->discovery_timer_source == NULL`（discovery tick 已停止，`not_done == 0`）
  - 无 UPnP 映射进行中
  - 无 DNS 解析进行中
- 发射前将 `stream->gathering` 设为 FALSE

## 跨模块调用链

```
nice_agent_gather_candidates()                          [agent/agent.c:3753]
  |
  +-- nice_interfaces_get_local_ips()                   [agent/interfaces.c:424]
  |     |
  |     +-- getifaddrs() / ioctl(SIOCGIFCONF)           [OS]
  |
  +-- resolve_stun_in_context()                         [agent/agent.c] (异步 DNS)
  |     |
  |     +-- stun_server_resolved_cb()                   [agent/agent.c:2791]
  |           |
  |           +-- priv_add_new_candidate_discovery_stun()  [agent/agent.c:2755]
  |                 |
  |                 +-- agent->discovery_list 追加 srflx CandidateDiscovery
  |
  +-- discovery_add_local_host_candidate()              [agent/discovery.c:762]
  |     |
  |     +-- nice_udp_bsd_socket_new()                   [socket/udp-bsd.c:94]
  |     |     |
  |     |     +-- g_socket_new() -> socket()            [GLib / OS]
  |     |     +-- g_socket_bind() -> bind()             [GLib / OS]
  |     |
  |     +-- nice_candidate_new(NICE_CANDIDATE_TYPE_HOST)
  |     +-- priv_assign_foundation()
  |     +-- priv_add_local_candidate_pruned()
  |     +-- nice_component_attach_socket()              [agent/component.c]
  |
  +-- priv_add_new_candidate_discovery_turn()           [agent/agent.c:2965]
  |     |
  |     +-- agent->discovery_list 追加 relay CandidateDiscovery
  |
  +-- agent_signal_new_candidate() (host)               [agent/agent.c:2639]
  |
  +-- discovery_schedule()                              [agent/discovery.c:1462]
  |     |
  |     +-- priv_discovery_tick_unlocked()              [agent/discovery.c:1262]
  |           |
  |           +-- stun_usage_bind_create()              [stun/usages/bind.c:88]
  |           |     |
  |           |     +-- stun_agent_init_request()       [stun/stunagent.c]
  |           |     +-- stun_agent_finish_message()     [stun/stunagent.c]
  |           |
  |           +-- stun_usage_turn_create()              [stun/usages/turn.c:71]
  |           |     |
  |           |     +-- stun_agent_init_request()       [stun/stunagent.c]
  |           |     +-- stun_message_append32()         [stun/stunmessage.c]
  |           |     +-- stun_agent_finish_message()     [stun/stunagent.c]
  |           |
  |           +-- agent_socket_send()                   [agent/agent.c]
  |                 |
  |                 +-- nice_socket_send()              [socket/socket.c]
  |                       |
  |                       +-- socket_send_messages()    [socket/udp-bsd.c:398]
  |                             |
  |                             +-- g_socket_send_message() [GLib -> sendmsg()]
  |
  | [网络往返...]
  |
  +-> component I/O 回调:                                [agent/component.c]
       |
       +-- socket_recv_messages()                       [socket/udp-bsd.c:273]
       |     |
       |     +-- g_socket_receive_message()             [GLib -> recvmsg()]
       |
       +-- conn_check_handle_inbound_stun()             [agent/conncheck.c:4532]
             |
             +-- stun_agent_validate()                  [stun/stunagent.c]
             |
             +-- priv_map_reply_to_discovery_request()  [agent/conncheck.c:3770]
             |     |
             |     +-- stun_usage_bind_process()        [stun/usages/bind.c:96]
             |     +-- discovery_add_server_reflexive_candidate() [agent/discovery.c:861]
             |     +-- agent_signal_new_candidate()     [agent/agent.c:2639]
             |
             +-- priv_map_reply_to_relay_request()      [agent/conncheck.c:3991]
                   |
                   +-- stun_usage_turn_process()        [stun/usages/turn.c:272]
                   +-- discovery_add_relay_candidate()  [agent/discovery.c:976]
                   +-- priv_add_new_turn_refresh()      [agent/conncheck.c:3866]
                   +-- agent_signal_new_candidate()     [agent/agent.c:2639]

[全部完成后]

agent_gathering_done()                                  [agent/agent.c:2374]
  |
  +-- agent_signal_gathering_done()                     [agent/agent.c:2483]
        |
        +-- 发射 "candidate-gathering-done" 信号
```
