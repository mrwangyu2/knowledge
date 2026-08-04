# 10 -- 流程：TURN 分配与中继

## 概述

TURN (RFC 5766) 是 ICE 的中继传输方式。当 P2P 直连失败时，端点通过 TURN 服务器中继数据。libnice 在 `socket/udp-turn.c` 中实现了完整的 TURN 客户端：Allocate 分配、CreatePermission 权限管理、ChannelBind 信道绑定、Send/Receive 数据中继，以及 Refresh 保活机制。

TURN 信令使用 STUN 协议（`stun/usages/turn.c` 负责构造/解析消息），重传由定时器模块（`stun/usages/timer.c`）驱动。对于 TCP 上的 TURN，libnice 支持通过 SOCKS5 代理（`socket/socks5.c`）和 HTTP CONNECT 代理（`socket/http.c`）建立到 TURN 服务器的隧道。

## 流程总览

```
创建 TURN socket -> Allocate -> CreatePermission -> ChannelBind -> 数据中继 -> 周期性 Refresh -> 释放
```

## 步骤 1：TURN Socket 创建

### nice_udp_turn_socket_new() (socket/udp-turn.c:193-277)

初始化 TURN 客户端 socket，将自身注册为 `NiceSocketType::NICE_SOCKET_TYPE_UDP_TURN`，挂在 `base_socket`（底层 UDP 或 TCP socket）之上。

**兼容模式选择（NiceTurnSocketCompatibility）：**

| 模式 | STUN 兼容性 | 认证方式 | 说明 |
|------|-------------|----------|------|
| `DRAFT9` / `RFC5766` | RFC 5389 | 长期凭证 (Long-Term) | 标准 TURN |
| `MSN` | RFC 3489 | 短期凭证 (Short-Term) | 旧版 MSN TURN |
| `GOOGLE` | RFC 3489 | 忽略凭证 (Ignore) | Google Talk 中继 |
| `OC2007` | OC2007 定制 | 长期凭证 (Long-Term) + 无对齐属性 | MS Office Communicator |

**关键初始化动作：**
- `stun_agent_init()` -- 初始化 `StunAgent`，配置已知属性、兼容性标志和认证模式
- 用户名/密码存储 -- MSN/OC2007 模式对凭据做 Base64 解码
- `send_data_queues` -- 创建按 peer 地址哈希的发送队列，用于权限未就绪时暂存待发数据
- `send_buffer` -- 预分配最大 STUN 消息大小的发送缓冲区（`STUN_MAX_MESSAGE_SIZE`）
- 设置函数指针表：`send_messages`、`recv_messages`、`is_reliable` 等，全部委托给 `base_socket`

### TURN over TCP 路径

当 `base_socket` 是可靠传输（TCP）时：
- `udp-turn-over-tcp.c` 提供 TCP 上的 RFC 4571 帧封装（2 字节长度前缀 `<length><data>`）
- 支持通过 HTTP CONNECT 代理或 SOCKS5 代理先建立隧道，再在隧道上运行 TURN 协议

**可靠性判定：** `socket_is_reliable()` 直接委托给 `base_socket->is_reliable()`。TCP TURN 场景下 reliable 为 TRUE，UDP TURN 为 FALSE。

## 步骤 2：TURN 分配 (Allocate)

### 2.1 构造 Allocate Request

**入口：** `stun_usage_turn_create()` (`stun/usages/turn.c:71-166`)

```c
stun_agent_init_request (agent, msg, buffer, buffer_len, STUN_ALLOCATE);
```

- **STUN Method：** `STUN_ALLOCATE` (0x003)
- **属性：**
  - `REQUESTED-TRANSPORT` = `0x11000000` (UDP, protocol 17) -- 仅 Draft9/RFC5766
  - `LIFETIME` -- 默认 3600s (1 小时)，可配置
  - `BANDWIDTH` -- 可选带宽请求 (Draft9/RFC5766)
  - `REQUESTED-PORT-PROPS` -- 可选：EVEN-PORT (E=0x80000000) 和/或 EVEN-AND-RESERVE (R=0x40000000) -- 仅 Draft9/RFC5766
  - `MAGIC_COOKIE` = `TURN_MAGIC_COOKIE` -- 旧版兼容模式 (Google/MSN/OC2007)
  - `MS_VERSION` = 1 -- OC2007 模式

**首次 vs 后续请求：**
- 首次请求：`previous_response = NULL`，不带 REALM/NONCE
- 重试（401 响应后）：从 `previous_response` 提取 REALM、NONCE 和 RESERVATION-TOKEN 回填到新请求
- 仅在短期凭证模式或已有 `previous_response` 时添加 USERNAME

**认证完成：** `stun_agent_finish_message()` 计算 MESSAGE-INTEGRITY 并最终化消息，返回完整的线格式缓冲区长度。

### 2.2 发送与重传

由 agent 层（`agent/discovery.c:discovery_add_relay_candidate()`）发起：
- 调用 `nice_udp_turn_socket_new()` 创建 TURN socket
- 调用 `stun_usage_turn_create()` 构造 Allocate Request
- 通过 `nice_socket_send()` 发送到 TURN server

**STUN 重传定时器（`stun/usages/timer.c`）：**

| 传输类型 | 初始超时 | 最大重传次数 | 超时策略 |
|----------|----------|--------------|----------|
| UDP (不可靠) | `STUN_TIMER_DEFAULT_TIMEOUT` = 500ms | `STUN_TIMER_DEFAULT_MAX_RETRANSMISSIONS` = 3 | 指数退避：500 -> 1000 -> 2000 -> 1000 (= 4500ms 总超时) |
| TCP (可靠) | `STUN_TIMER_DEFAULT_RELIABLE_TIMEOUT` = 2000ms | 0 | 2000ms 后判定超时 |

重传逻辑（`stun_timer_refresh()`）：
1. 调用 `stun_timer_remainder()` 查询剩余时间
2. 若到期：`retransmissions >= max_retransmissions` -> 返回 `TIMEOUT`；否则加倍 `delay`（最后一次减半），递增计数，返回 `RETRANSMIT`
3. 若未到期：返回 `SUCCESS`

### 2.3 处理 Allocate Response

**入口：** `stun_usage_turn_process()` (`stun/usages/turn.c:272-408`)

**成功路径：**

```
XOR-RELAYED-ADDRESS (Draft9/RFC5766) 或 MAPPED-ADDRESS (Google/MSN/OC2007)
    -> 提取为中继候选地址 (relay_addr)
```

**返回值枚举：**
- `STUN_USAGE_TURN_RETURN_RELAY_SUCCESS` -- 分配成功，有中继地址
- `STUN_USAGE_TURN_RETURN_MAPPED_SUCCESS` -- 分配成功，同时返回映射地址和中继地址
- `STUN_USAGE_TURN_RETURN_ERROR` -- 错误响应
- `STUN_USAGE_TURN_RETURN_INVALID` -- 非有效响应
- `STUN_USAGE_TURN_RETURN_ALTERNATE_SERVER` -- 3xx 错误，需切换到备用服务器

**错误处理：**
- `401 Unauthorized` -- 需重试，带上 REALM 和 NONCE
- `403 Forbidden` -- 认证失败
- `437 Mismatch` -- 分配不匹配（通常刷新时发生）
- `441 Wrong Connection` -- 错误的 TCP 连接
- `300 Try Alternate` -- 提取 ALTERNATE-SERVER 属性，通知上层切换服务器

## 步骤 3：权限管理 (CreatePermission)

### 3.1 权限机制

TURN 服务器默认不向任何 peer 转发数据。必须通过 CreatePermission 请求为每个 peer 地址单独授权。libnice 实现了基于 peer 地址的权限生命周期管理。

**相关数据结构（`UdpTurnPriv`）：**
- `permissions` -- 已安装的权限列表（`GList<NiceAddress>`）
- `sent_permissions` -- 已发送但未收到响应的权限列表（防止重复发送）
- `pending_permissions` -- 待处理的重传 CreatePermission 消息列表（`GList<TURNMessage*>`）
- `send_data_queues` -- 按 peer 哈希的发送数据队列（`GHashTable<NiceAddress, GQueue<SendData>>`）：权限未就绪时暂存待发数据

### 3.2 创建权限

**触发时机：**
1. 主动发送数据到未授权 peer 时（`socket_send_message()` 中检测 `!priv_has_permission_for_peer()`）
2. 收到来自未授权 peer 的 Data Indication 时（`nice_udp_turn_socket_parse_recv()` 中检测）

**函数：** `priv_send_create_permission()` (`udp-turn.c:2013-2078`)

```c
stun_usage_turn_create_permission(&priv->agent, &msg->message, ...,
    &addr.storage,
    STUN_USAGE_TURN_COMPATIBILITY_RFC5766);
```

构造 `STUN_CREATEPERMISSION` 请求，包含：
- `XOR-PEER-ADDRESS` -- 要授权的 peer 地址（XOR 编码）
- `REALM` / `NONCE` / `USERNAME` -- 认证信息（从缓存的 Allocate 响应中获取）

**重传处理（`priv_retransmissions_create_permission_tick_unlocked()`）**：与 ChannelBind 使用相同的定时器框架。超时后假设服务器不支持 CreatePermission，直接标记权限为已安装（`priv_add_permission_for_peer()`），并释放队列中的待发数据。

### 3.3 权限刷新与超时

- **刷新间隔：** `STUN_PERMISSION_TIMEOUT` = 300 - 60 = **240 秒**
  - 在 CreatePermission 成功响应的 240 秒后触发 `priv_permission_timeout()`
  - 清除所有权限 (`priv_clear_permissions()`)
  - 下次发送数据到该 peer 时会自动重新创建权限
- **定时器安装：** 首次成功创建权限后，通过 `priv_timeout_add_seconds_with_context()` 安装 `priv_permission_timeout()` 回调

## 步骤 4：信道绑定 (ChannelBind)

### 4.1 Channel 号分配

**RFC 5766 规定 channel number 范围：** 0x4000 -- 0x7FFF

**libnice 使用范围：** 0x4000 -- 0xFFFF（从 0x4000 开始，在已有绑定列表中递增查找最小未使用的 channel 号）

分配逻辑（`priv_add_channel_binding()` 中）：
```c
uint16_t channel = 0x4000;
for (i = priv->channels; i; i = i->next) {
    if (channel == b->channel) {
        i = priv->channels;  // 重新扫描
        channel++;
    }
}
```

### 4.2 ChannelData 帧格式

ChannelData 消息（4 字节头）：
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Channel Number        |            Length             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                       Application Data                        |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**发送方（`socket_send_message()`）：** 已绑定 channel 时，使用 4 字节 ChannelData 头（`channel16 + len16`）替代 STUN 封装。代码：
```c
memcpy(buffer, &channel16, sizeof(uint16_t));
memcpy(buffer + sizeof(uint16_t), &len16, sizeof(uint16_t));
// 后面紧跟 application data
```

**接收方（`nice_udp_turn_socket_parse_recv()`）：** 首先匹配收到的 channel number（`ntohs(recv_buf.u16[0])`）与绑定列表（`priv->channels`）对比，找到对应 peer 地址。提取 `len = ntohs(recv_buf.u16[1])`，跳过 4 字节头，将剩余数据传上应用层。

### 4.3 ChannelBind 生命周期

**绑定流程：**
1. `nice_udp_turn_socket_set_peer()` -> `priv_add_channel_binding()` -> `priv_send_channel_bind()`
2. 构造 `STUN_CHANNELBIND` 请求，包含 CHANNEL-NUMBER 和 XOR-PEER-ADDRESS
3. 发送并设置重传定时器
4. 收到成功响应：从 `priv->current_binding` 移入 `priv->channels` 列表；安装刷新定时器（`STUN_BINDING_TIMEOUT` = 540s）
5. 收到未授权响应（stale nonce 或 401）：缓存新的 realm/nonce，重发 ChannelBind
6. 其他错误：放弃该绑定，处理下一个待处理绑定

**刷新机制：**
- **刷新触发：** `STUN_BINDING_TIMEOUT` = 600 - 60 = **540 秒**
  - `priv_binding_timeout()` 标记 `binding->renew = TRUE`，安装 60 秒的过期定时器 (`priv_binding_expired_timeout`)
  - 如果没有正在进行的绑定操作，发送刷新 ChannelBind 请求
- **过期处理：** 60 秒内刷新未完成 -> `priv_binding_expired_timeout()` 移除过期绑定，将 peer 地址重新加入待处理列表 (`priv_add_channel_binding`)

**并发控制：**
- `current_binding` -- 正在进行的绑定操作（新绑定或刷新），同一时间只有一个进行中
- `pending_bindings` -- 等待处理的绑定请求队列
- `priv_process_pending_bindings()` -- 当前绑定完成后处理队列中的下一个

### 4.4 旧版兼容模式绑定

**Google 模式：** 不需要发送 ChannelBind 请求，`priv_add_channel_binding()` 直接创建 binding 并返回 TRUE。

**MSN/OC2007 模式：** 发送 `STUN_OLD_SET_ACTIVE_DST` 请求（而非 ChannelBind）。成功后转到 `msn_google_lock` 标签：清除旧绑定列表，将 `current_binding` 作为唯一绑定加入列表。OC2007 还需附加 `MS_SEQUENCE_NUMBER` 和 `REALM`。

## 步骤 5：数据中继

### 5.1 发送路径

**函数：** `socket_send_message()` (`udp-turn.c:791-983`)

```
应用数据 (NiceOutputMessage)
    |
    +--> 查找该 peer 的 ChannelBinding
    |
    +--> [已绑定] --> 构造 4 字节 ChannelData 头 --> 发送到 TURN server
    |
    +--> [未绑定，Draft9/RFC5766]
    |       |
    |       +--> 构造 STUN_IND_SEND (Send Indication)
    |       |    属性：XOR-PEER-ADDRESS + DATA
    |       |
    |       +--> [无权限]
    |       |     +--> priv_send_create_permission()
    |       |     +--> 数据入队 (send_data_queues)
    |       |     +--> 权限就绪后 socket_dequeue_all_data() 发送
    |       |
    |       +--> [有权限] --> 通过 base_socket 发送
    |
    +--> [未绑定，旧版模式] --> 构造 STUN_SEND Request
           属性：MAGIC_COOKIE + DESTINATION-ADDRESS + DATA + USERNAME
```

**关键决策逻辑：** 对于 RFC5766 模式，发送前检测 `priv_has_permission_for_peer()`。若未安装权限且未在进行中，自动触发 `priv_send_create_permission()` 并暂存数据。这保证了 TURN 发送对上层是透明的 -- 即使权限未就绪，数据不会丢失。

**可靠发送限制：** `socket_send_messages_reliable()` 仅当 `base_socket->type != NICE_SOCKET_TYPE_UDP_BSD` 时才工作。因为 UDP 上无法保证队列中数据的可靠投递。

### 5.2 接收路径

**函数：** `nice_udp_turn_socket_parse_recv()` (`udp-turn.c:1319-1717`)

```
base_socket 收到数据
    |
    +--> [来源 == TURN server] --> StunAgent 验证 STUN 消息
    |       |
    |       +--> STUN_IND_DATA (Data Indication)
    |       |     提取：XOR-REMOTE-ADDRESS + DATA
    |       |     -> 验证权限 (RFC5766 模式下)
    |       |     -> memmove() 将纯数据写入用户缓冲区
    |       |     -> 返回数据长度
    |       |
    |       +--> STUN_CHANNELBIND (响应)
    |       |     -> 匹配 transaction ID
    |       |     -> 成功：加入 channels 列表，安装刷新定时器
    |       |     -> 未授权：缓存 realm/nonce，重发
    |       |
    |       +--> STUN_CREATEPERMISSION (响应)
    |       |     -> 匹配 transaction ID
    |       |     -> 成功/错误：标记权限已安装
    |       |     -> 释放该 peer 的发送队列
    |       |
    |       +--> STUN_SEND (响应) -- 旧版模式
    |       |     -> 清除对应的 SendRequest 记录
    |       |
    |       +--> STUN_OLD_SET_ACTIVE_DST (响应) -- MSN/OC2007
    |             -> 成功：锁定当前绑定
    |             -> 失败：清除 current_binding
    |
    +--> [来源 != TURN server] --> 遍历 channels 列表
            |
            +--> [Draft9/RFC5766] 匹配 channel number (recv_buf.u16[0])
            +--> [旧版] 第一个 channel binding
            |
            +--> 提取 peer 地址 -> memmove() 数据到用户缓冲区
```

**接收缓冲区管理（TCP 场景）：**
- TCP 使用 RFC 4571 帧封装，每个 TURN 消息前有 2 字节长度前缀
- `fragment_buffer` (`GByteArray`) -- 存储不完整的消息片段
- `priv->from` -- 保存片段化消息的源地址
- 在 `socket_recv_messages()` 中，首先消费 `fragment_buffer` 中已完成的帧，再从 `base_socket` 读取新数据并解析

## 步骤 6：分配刷新与保活

### 6.1 Refresh Request

**函数：** `stun_usage_turn_create_refresh()` (`stun/usages/turn.c:168-222`)

对于 Draft9/RFC5766：
- 使用 `STUN_REFRESH` method（而非 Allocate）
- 可选 LIFETIME 属性（刷新续期时间）
- 包含之前响应中的 REALM 和 NONCE
- 仅在短期凭证或有 previous_response 时添加 USERNAME

对于旧版兼容模式（MSN/Google/OC2007）：
- 回退到 `stun_usage_turn_create()`（使用 `STUN_ALLOCATE` method），因为旧协议没有独立的 Refresh method

刷新在 agent 层（`agent/` 目录）以周期性定时器驱动，而非在 socket 层。

### 6.2 Allocation 超时处理

- 刷新失败（STUN 超时或错误响应） -> socket 层关闭 -> 通知上层 agent
- agent 将对应的 relay candidate 标记为失效
- 在 ICE 状态机中触发相应的失败/回退处理

## 步骤 7：代理路径（TCP 上的 TURN）

当 TURN 服务器需要通过代理访问时，libnice 提供两种代理协议实现：

### 7.1 SOCKS5 代理 (socket/socks5.c)

**状态机：**

| 状态 | 动作 |
|------|------|
| `SOCKS_STATE_INIT` | 发送 SOCKS5 握手（版本 + 支持的认证方法列表） -> 接收服务器选择的方法 |
| `SOCKS_STATE_AUTH` | 用户名/密码认证（如服务器要求） -> 接收认证结果 |
| `SOCKS_STATE_CONNECT` | 发送 CONNECT 命令（目标地址 + 端口） -> 接收连接结果和绑定地址 |
| `SOCKS_STATE_CONNECTED` | 隧道建立完成，透传数据到 `base_socket` |
| `SOCKS_STATE_ERROR` | 错误状态，所有操作返回 -1 |

**握手消息：**
1. 客户端 -> 服务器：`{ 0x05, 0x01/0x02, [0x00], [0x02] }` -- SOCKS5 版本 + 认证方法
2. 服务器 -> 客户端：`{ version, method }` -- 选中的认证方法
3. (可选) 用户名/密码认证：版本(0x01) + 用户名长度 + 用户名 + 密码长度 + 密码
4. 客户端 -> 服务器：CONNECT 命令（0x01）+ 保留(0x00) + 地址类型 + 地址 + 端口
5. 服务器 -> 客户端：版本 + 回复码 + 保留 + 地址类型 + 绑定地址 + 端口

**在非 CONNECTED 状态下：**
- `send_messages()` 返回 0（EWOULDBLOCK）
- `send_messages_reliable()` 将消息入队 (`send_queue`)，连接完成后 `nice_socket_flush_send_queue()` 发送

### 7.2 HTTP CONNECT 代理 (socket/http.c)

**状态机：**

| 状态 | 动作 |
|------|------|
| `HTTP_STATE_INIT` | 解析 HTTP 响应行（`HTTP/1.x 2xx`） |
| `HTTP_STATE_HEADERS` | 解析响应头，提取 `Content-Length` |
| `HTTP_STATE_BODY` | 消费 `Content-Length` 长度的 body |
| `HTTP_STATE_CONNECTED` | 隧道建立完成，透传数据 |
| `HTTP_STATE_ERROR` | 错误状态 |

**CONNECT 请求格式：**
```
CONNECT <host>:<port> HTTP/1.0
Host: <host>
User-Agent: libnice
Content-Length: 0
Proxy-Connection: Keep-Alive
Connection: Keep-Alive
Cache-Control: no-cache
Pragma: no-cache
Proxy-Authorization: Basic <base64(user:pass)>  (可选)
\r\n
```

**接收缓冲区管理：**
- 使用环形缓冲区 (`recv_buf` + `recv_buf_pos` + `recv_buf_fill`) 实现无拷贝解析
- 初始缓冲区 1KB，满时翻倍增长
- `memcpy_ring_buffer_to_input_messages()` -- 将环形缓冲区中的数据直接分发到多个 `NiceInputMessage`

**与 SOCKS5 相同的语义：** 在非 CONNECTED 状态下，send 被阻塞/入队。

## 跨模块调用链

```
discovery_add_relay_candidate() (agent/discovery.c)
  |
  +--> nice_udp_turn_socket_new() (socket/udp-turn.c)
  |      |
  |      +--> stun_agent_init() (stun/stunagent.c)
  |      +--> [TCP 代理] nice_socks5_socket_new() 或 nice_http_socket_new()
  |
  +--> stun_usage_turn_create() (stun/usages/turn.c)  [Allocate Request]
  +--> nice_socket_send() (socket.c)
  |
  +--> [接收 Allocate Response]
  |      +--> stun_agent_validate() (stun/stunagent.c)
  |      +--> stun_usage_turn_process() (stun/usages/turn.c)
  |             -> 提取 XOR-RELAYED-ADDRESS / XOR-MAPPED-ADDRESS
  |
  +--> [Permission 管理] (socket/udp-turn.c)
  |      +--> stun_usage_turn_create_permission() (stun/usages/turn.c)
  |      +--> stun_timer_start() / stun_timer_refresh() (stun/usages/timer.c)
  |
  +--> [ChannelBind] (socket/udp-turn.c)
  |      +--> priv_add_channel_binding() -> priv_send_channel_bind()
  |             -> STUN_CHANNELBIND 请求 + CHANNEL-NUMBER + XOR-PEER-ADDRESS
  |      +--> stun_timer_start() -> 重传管理
  |      +--> priv_schedule_tick() -> GLib timer source
  |
  +--> [数据传输]
  |      +--> socket_send_message()
  |      |      有 channel -> ChannelData (4 字节头)
  |      |      无 channel -> Send Indication (STUN 封装)
  |      +--> nice_udp_turn_socket_parse_recv()
  |              ChannelData (匹配 channel number -> peer -> memmove)
  |              或 Data Indication (提取 XOR-REMOTE-ADDRESS + DATA -> memmove)
  |
  +--> [刷新]
         +--> stun_usage_turn_create_refresh() (stun/usages/turn.c)
         +--> stun_usage_turn_refresh_process() (stun/usages/turn.c)
```

## 定时器配置汇总

| 常量 | 值 | 含义 |
|------|-----|------|
| `STUN_TIMER_DEFAULT_TIMEOUT` | 500ms | UDP 上 STUN 请求的初始 RTO |
| `STUN_TIMER_DEFAULT_MAX_RETRANSMISSIONS` | 3 | UDP 上 STUN 请求的最大重传次数 |
| `STUN_TIMER_DEFAULT_RELIABLE_TIMEOUT` | 2000ms | TCP 上 STUN 请求的初始超时 |
| `STUN_END_TIMEOUT` | 8000ms | Send Request (Google 模式) 的超时 |
| `STUN_PERMISSION_TIMEOUT` | 240s | 权限刷新间隔（300s 许可 - 60s 预刷新） |
| `STUN_BINDING_TIMEOUT` | 540s | ChannelBinding 刷新间隔（600s 许可 - 60s 预刷新） |
| `STUN_EXPIRE_TIMEOUT` | 60s | 刷新失败后的宽限期 |

UDP 重传总超时：500 * (1 + 2 + 1) = 2000ms（初始 500，第一次加倍到 1000，第二次加倍到 2000，最后一次减半到 1000）

## ChannelData vs Send/Data Indication 开销对比

| 方式 | 头部大小 | 适用场景 | 限制 |
|------|---------|----------|------|
| ChannelData | 4 bytes (channel + length) | 高频数据传输 | 需先完成 ChannelBind |
| Send Indication | ~36 bytes (STUN 头 + XOR-PEER-ADDRESS + DATA 属性) | 低频数据、未绑定 peer | 每个包都有完整 STUN 头开销 |
| Data Indication | ~36 bytes (同 Send Indication) | 接收方向，TURN server 转发 | 无 |
