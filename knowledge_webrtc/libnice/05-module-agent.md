# 05 — 模块分析：agent/

## 模块概述

`agent/` 是 libnice 的核心模块，实现了 ICE 代理的全部逻辑。模块包含 **28 个源文件**（14 个 .c + 14 个 .h），总代码量约 **27500 行**，其中 `agent.c` 单文件达到 7957 行。整个模块基于 **GLib/GObject** 的事件驱动架构设计：一个运行中的 `GMainLoop` / `GMainContext` 驱动所有 ICE 定时器（连接检查、候选发现、保活），代理状态机的推进完全依赖 GLib 主循环的迭代。

模块职责覆盖：
- ICE 代理的 GObject 类型定义及完整的公共 API
- 媒体流（Stream）、组件（Component）、候选地址（Candidate）的内部结构与管理
- 连接检查状态机（conncheck）与候选发现流程（discovery）
- 可靠传输模式（PseudoTCP / bytestream TCP）
- GIO I/O 流封装（NiceInputStream / NiceOutputStream / NiceIOStream）
- 候选地址类型及优先级计算
- SDP 生成/解析（含 Trickle ICE 支持）

---

## 第一部分：GObject 类型系统与公共 API

### 1.1 NiceAgent 类型定义（agent.h）

#### 1.1.1 GObject 集成

`NiceAgent` 通过标准 GObject 宏系统注册为 GType：

```c
G_DEFINE_TYPE (NiceAgent, nice_agent, G_TYPE_OBJECT);  // agent.c:99
```

配套的 GObject 类型宏定义在 `agent.h` 中：

```c
#define NICE_TYPE_AGENT nice_agent_get_type()          // 返回 GType
#define NICE_AGENT(obj)             G_TYPE_CHECK_INSTANCE_CAST(...)
#define NICE_AGENT_CLASS(klass)     G_TYPE_CHECK_CLASS_CAST(...)
#define NICE_IS_AGENT(obj)          G_TYPE_CHECK_INSTANCE_TYPE(...)
#define NICE_IS_AGENT_CLASS(klass)  G_TYPE_CHECK_CLASS_TYPE(...)
#define NICE_AGENT_GET_CLASS(obj)   G_TYPE_INSTANCE_GET_CLASS(...)
```

`NiceAgentClass` 结构体极其简单，仅继承 `GObjectClass`，没有定义额外的虚函数槽位。所有扩展行为通过 GObject 属性和信号机制实现。

```c
struct _NiceAgentClass {
  GObjectClass parent_class;
};
```

#### 1.1.2 信号列表（10 个信号）

| 信号名 | 参数 | 说明 | 引入版本 |
|--------|------|------|----------|
| `"component-state-changed"` | `guint stream_id, guint component_id, guint state` | 组件状态发生变更 | 1.0 |
| `"candidate-gathering-done"` | `guint stream_id` | 候选收集完成（整个 stream 的所有 component） | 1.0 |
| `"new-selected-pair"` | `guint stream_id, guint component_id, gchar *lfoundation, gchar *rfoundation` | 选中新的候选对（仅传 foundation，已弃用） | 1.0, **deprecated 0.1.8** |
| `"new-selected-pair-full"` | `guint stream_id, guint component_id, NiceCandidate *lcandidate, NiceCandidate *rcandidate` | 选中新的候选对（传完整候选对象） | 0.1.8 |
| `"new-candidate"` | `guint stream_id, guint component_id, gchar *foundation` | 发现新本地候选（仅传 foundation，已弃用） | 1.0, **deprecated 0.1.8** |
| `"new-candidate-full"` | `NiceCandidate *candidate` | 发现新本地候选（传完整候选对象） | 0.1.8 |
| `"new-remote-candidate"` | `guint stream_id, guint component_id, gchar *foundation` | 发现新远端候选（peer reflexive，已弃用） | 1.0, **deprecated 0.1.8** |
| `"new-remote-candidate-full"` | `NiceCandidate *candidate` | 发现新远端候选（传完整候选对象） | 0.1.8 |
| `"initial-binding-request-received"` | `guint stream_id` | 收到首个来自对端的 binding request | 1.0 |
| `"reliable-transport-writable"` | `guint stream_id, guint component_id` | 底层传输变为可写（可靠模式下 send buffer 从满到可写，或连接刚建立时） | 0.0.11，0.1.23 起也支持非可靠传输 |
| `"streams-removed"` | `guint[] stream_ids` (zero-terminated array) | 一个或多个 stream 被移除 | 0.1.5 |

**信号设计模式**：旧版信号（`new-candidate`、`new-selected-pair`、`new-remote-candidate`）仅传递 foundation 字符串，导致应用层难以直接使用候选对象。新版 `-full` 后缀信号传递完整的 `NiceCandidate` 对象，推荐所有新代码使用 `-full` 版本。

#### 1.1.3 GObject 属性列表（28 个属性）

属性分为构造时属性和可读写属性两类。`CONSTRUCT_ONLY` 属性必须在 `g_object_new()` 时传入，之后不可修改。

| 属性 ID | 属性名 | 类型 | 默认值 | 说明 |
|----------|--------|------|--------|------|
| PROP_COMPATIBILITY | `compatibility` | `uint` | `RFC5245` | ICE 兼容模式，CONSTRUCT_ONLY |
| PROP_MAIN_CONTEXT | `main-context` | `pointer` | (构造时必须) | GMainContext，CONSTRUCT_ONLY |
| PROP_STUN_SERVER | `stun-server` | `string` | NULL | STUN 服务器 IP/主机名 |
| PROP_STUN_SERVER_PORT | `stun-server-port` | `uint` | 3478 | STUN 服务器端口 |
| PROP_CONTROLLING_MODE | `controlling-mode` | `boolean` | TRUE | 是否为 controlling 角色 |
| PROP_FULL_MODE | `full-mode` | `boolean` | TRUE | ICE-FULL 或 ICE-LITE 模式 |
| PROP_STUN_PACING_TIMER | `stun-pacing-timer` | `uint` | 20ms | STUN 事务重传定时器 Ta |
| PROP_MAX_CONNECTIVITY_CHECKS | `max-connectivity-checks` | `uint` | 100 | 最大连接检查对数量 (RFC 8445 6.1.2.5) |
| PROP_PROXY_TYPE | `proxy-type` | `enum` | `NONE` | 代理类型 (NONE/SOCKS5/HTTP) |
| PROP_PROXY_IP | `proxy-ip` | `string` | NULL | 代理服务器 IP |
| PROP_PROXY_PORT | `proxy-port` | `uint` | 0 | 代理服务器端口 |
| PROP_PROXY_USERNAME | `proxy-username` | `string` | NULL | 代理认证用户名 |
| PROP_PROXY_PASSWORD | `proxy-password` | `string` | NULL | 代理认证密码 |
| PROP_PROXY_EXTRA_HEADERS | `proxy-extra-headers` | `boxed` | NULL | 代理额外 HTTP 头 |
| PROP_UPNP | `upnp` | `boolean` | TRUE（如编译有 GUPnP） | 启用 UPnP 端口映射 |
| PROP_UPNP_TIMEOUT | `upnp-timeout` | `uint` | 200ms | UPnP 发现超时 |
| PROP_RELIABLE | `reliable` | `boolean` | FALSE | 可靠传输模式，CONSTRUCT_ONLY |
| PROP_ICE_UDP | `ice-udp` | `boolean` | TRUE | 启用 ICE-UDP |
| PROP_ICE_TCP | `ice-tcp` | `boolean` | TRUE | 启用 ICE-TCP |
| PROP_BYTESTREAM_TCP | `bytestream-tcp` | `boolean` | FALSE | 可靠 TCP 的 bytestream 模式，CONSTRUCT_ONLY |
| PROP_KEEPALIVE_CONNCHECK | `keepalive-conncheck` | `boolean` | FALSE | 使用 conncheck 作为 keepalive 机制 |
| PROP_FORCE_RELAY | `force-relay` | `boolean` | FALSE | 强制仅使用 relay 候选 |
| PROP_STUN_MAX_RETRANSMISSIONS | `stun-max-retransmissions` | `uint` | 0 | STUN 最大重传次数 (Rc) |
| PROP_STUN_INITIAL_TIMEOUT | `stun-initial-timeout` | `uint` | 0 | STUN 初始超时 (RTO) |
| PROP_STUN_RELIABLE_TIMEOUT | `stun-reliable-timeout` | `uint` | 0 | STUN 可靠超时 |
| PROP_NOMINATION_MODE | `nomination-mode` | `enum` | `AGGRESSIVE` | 提名模式 (REGULAR/AGGRESSIVE)，CONSTRUCT_ONLY |
| PROP_IDLE_TIMEOUT | `idle-timeout` | `uint` | 5000ms | 连接检查空闲超时（停止前等待延迟） |
| PROP_CONSENT_FRESHNESS | `consent-freshness` | `boolean` | FALSE | 启用 RFC 7675 consent freshness，CONSTRUCT_ONLY |
| PROP_CLOSE_FORCED | `close-forced` | `boolean` | FALSE | 强制关闭（不等待 TURN 响应），CONSTRUCT_ONLY |
| PROP_RECV_TOS | `recv-tos` | `boolean` | FALSE | 启用 IP_TOS/IPV6_TCLASS 接收，CONSTRUCT_ONLY |

---

### 1.2 重要枚举

#### 1.2.1 NiceCompatibility -- ICE 兼容模式

```c
typedef enum {
  NICE_COMPATIBILITY_RFC5245 = 0,
  NICE_COMPATIBILITY_DRAFT19   = NICE_COMPATIBILITY_RFC5245,  // 已弃用，等同 RFC5245
  NICE_COMPATIBILITY_GOOGLE,
  NICE_COMPATIBILITY_MSN,
  NICE_COMPATIBILITY_WLM2009,
  NICE_COMPATIBILITY_OC2007,
  NICE_COMPATIBILITY_OC2007R2,
  NICE_COMPATIBILITY_LAST      = NICE_COMPATIBILITY_OC2007R2,
} NiceCompatibility;
```

各种兼容模式的根本差异在于：
- **凭据粒度**：RFC5245 / WLM2009 / OC2007R2 / DRAFT19 使用**流级别**凭据（per-stream credentials），在 STUN 消息中使用 `STUN_ATTRIBUTE_USERNAME`。GOOGLE / MSN / OC2007 使用**候选级别**凭据（per-candidate credentials），username 直接携带在候选上。
- **优先级计算**：RFC5245 使用标准 ICE 优先级公式 `(2^24 * type_pref) + (2^8 * local_pref) + (2^0 * 256 - component_id)`。MSN 使用 MSN 专用优先级。GOOGLE 使用 Jingle 优先级（XEP-0176）。
- **候选对优先级**：RFC5245 计算 `2^32 * MIN(G,D) + 2 * MAX(G,D) + (G>D?1:0)` 其中 G=controlling agent, D=controlled agent。MSN 使用不同的公式。
- **连接检查行为**：GOOGLE 模式下不发送 STUN 的 USE-CANDIDATE 属性（不进行提名）。MSN / WLM2009 / OC2007 各有特殊的连接检查消息格式和流程。

一个便捷宏判断代理是否兼容 RFC5245 或 OC2007R2（这两种共享 regular/aggressive 提名模式支持）：

```c
#define NICE_AGENT_IS_COMPATIBLE_WITH_RFC5245_OR_OC2007R2(obj) \
  ((obj)->compatibility == NICE_COMPATIBILITY_RFC5245 || \
   (obj)->compatibility == NICE_COMPATIBILITY_OC2007R2)
```

#### 1.2.2 NiceAgentOption -- 代理选项（位标志）

定义在 `nice_agent_new_full()` 中组合使用的位标志位：

```c
typedef enum {
  NICE_AGENT_OPTION_NONE                 = 0,
  NICE_AGENT_OPTION_REGULAR_NOMINATION   = 1 << 0,  // 使用 regular 提名（默认 aggressive）
  NICE_AGENT_OPTION_RELIABLE             = 1 << 1,  // 启用可靠模式（PseudoTCP）
  NICE_AGENT_OPTION_LITE_MODE            = 1 << 2,  // ICE-LITE 模式
  NICE_AGENT_OPTION_ICE_TRICKLE          = 1 << 3,  // 启用 Trickle ICE
  NICE_AGENT_OPTION_SUPPORT_RENOMINATION = 1 << 4,  // 支持重提名
  NICE_AGENT_OPTION_CONSENT_FRESHNESS    = 1 << 5,  // RFC 7675 consent freshness
  NICE_AGENT_OPTION_BYTESTREAM_TCP       = 1 << 6,  // 可靠 TCP 的 bytestream 模式
  NICE_AGENT_OPTION_CLOSE_FORCED         = 1 << 7,  // 强制关闭（不等待 TURN 响应）
  NICE_AGENT_OPTION_RECV_TOS             = 1 << 8,  // 接收 IP_TOS/IPV6_TCLASS
} NiceAgentOption;
```

#### 1.2.3 NiceComponentState -- 组件状态

```c
typedef enum {
  NICE_COMPONENT_STATE_DISCONNECTED,  // 无活动，初始状态
  NICE_COMPONENT_STATE_GATHERING,     // 正在收集本地候选
  NICE_COMPONENT_STATE_CONNECTING,    // 正在执行连通性检查
  NICE_COMPONENT_STATE_CONNECTED,     // 至少一对候选可达（但尚未最终选定）
  NICE_COMPONENT_STATE_READY,         // ICE 完成，候选对选定最终化（可发送数据）
  NICE_COMPONENT_STATE_FAILED,        // 连通性检查完成但全部失败
  NICE_COMPONENT_STATE_LAST           // 哑值，表示枚举边界
} NiceComponentState;
```

状态机转换路径：`DISCONNECTED -> GATHERING -> CONNECTING -> CONNECTED -> READY`（或从任意状态进入 `FAILED`）。

#### 1.2.4 NiceComponentType -- 组件类型

```c
typedef enum {
  NICE_COMPONENT_TYPE_RTP  = 1,   // RTP 组件（component_id = 1）
  NICE_COMPONENT_TYPE_RTCP = 2,   // RTCP 组件（component_id = 2）
} NiceComponentType;
```

libnice 标准用法中，每个 stream 有 1 或 2 个 component：component 1 用于 RTP，component 2 用于 RTCP。用户也可以创建任意数量 component 的 stream。

#### 1.2.5 NiceCandidateTransport -- 候选传输类型

```c
typedef enum {
  NICE_CANDIDATE_TRANSPORT_UDP,
  NICE_CANDIDATE_TRANSPORT_TCP_ACTIVE,
  NICE_CANDIDATE_TRANSPORT_TCP_PASSIVE,
  NICE_CANDIDATE_TRANSPORT_TCP_SO,
} NiceCandidateTransport;
```

TCP 候选区分为三种：`TCP_ACTIVE`（主动发起 TCP 连接）、`TCP_PASSIVE`（被动监听 TCP 连接）、`TCP_SO`（TCP Simultaneous-Open，即双方同时主动连接）。

#### 1.2.6 其他辅助枚举

**NiceNominationMode**：提名模式
```c
typedef enum {
  NICE_NOMINATION_MODE_REGULAR = 0,
  NICE_NOMINATION_MODE_AGGRESSIVE,
} NiceNominationMode;
```
Aggressive 模式更快选择候选对但可能频繁切换（每次新的成功检查都提名新对）。Regular 模式按 RFC 5245 规定等待连接检查完成后才提名。

**NiceProxyType**：代理类型（用于 TCP TURN 中继）
```c
typedef enum {
  NICE_PROXY_TYPE_NONE   = 0,
  NICE_PROXY_TYPE_SOCKS5,
  NICE_PROXY_TYPE_HTTP,
} NiceProxyType;
```

#### 1.2.7 回调类型

**NiceAgentRecvFunc**（基础接收回调）：
```c
typedef void (*NiceAgentRecvFunc) (
  NiceAgent *agent, guint stream_id, guint component_id,
  guint len, gchar *buf, gpointer user_data);
```

**NiceAgentRecvFuncEx**（扩展接收回调，含 NiceMessageExtraData）：
```c
typedef void (*NiceAgentRecvFuncEx) (
  NiceAgent *agent, guint stream_id, guint component_id,
  guint len, gchar *buf, NiceMessageExtraData *exdata, gpointer user_data);
```
扩展版本通过 `NiceMessageExtraData` 可获取 IP TOS/ECN 信息（需启用 `NICE_AGENT_OPTION_RECV_TOS`）。

#### 1.2.8 数据消息结构体

**NiceInputMessage**（接收消息描述）：
```c
typedef struct {
  GInputVector *buffers;   // 接收缓冲区数组
  gint n_buffers;          // 缓冲区数量，-1 表示 NULL 终止
  NiceAddress *from;       // [out] 发送方地址
  gsize length;            // [out] 实际接收的字节总数
} NiceInputMessage;
```

**NiceOutputMessage**（发送消息描述）：
```c
typedef struct {
  GOutputVector *buffers;  // 发送缓冲区数组
  gint n_buffers;          // 缓冲区数量，-1 表示 NULL 终止
} NiceOutputMessage;
```

**NiceMessageExtraData**（接收消息的额外元数据）：
```c
struct _NiceMessageExtraData {
  GSocketControlMessage *tos;   // IP_TOS / IPV6_TCLASS 控制消息
};
```

---

### 1.3 文件：agent-priv.h（359 行）-- NiceAgent 私有结构

`struct _NiceAgent` 是理解 agent 内部架构的核心入口。以下按功能域分析每个字段的含义和角色：

```c
struct _NiceAgent {
  /* ===== GObject 基础 ===== */
  GObject parent;                     // GObject 父类指针
  GMutex agent_mutex;                 // 全代理互斥锁，保证线程安全

  /* ===== 全局配置属性 ===== */
  gboolean full_mode;                 // TRUE=ICE-FULL, FALSE=ICE-LITE
  gchar *stun_server_ip;              // STUN 服务器 IP 或主机名（用于 srflx 候选）
  guint stun_server_port;             // STUN 服务器端口，默认 3478
  gchar *proxy_ip;                    // 代理服务器 IP
  guint proxy_port;                   // 代理服务器端口
  NiceProxyType proxy_type;           // 代理类型（NONE/SOCKS5/HTTP）
  gchar *proxy_username;              // 代理认证用户名
  gchar *proxy_password;              // 代理认证密码
  GHashTable *proxy_extra_headers;    // HTTP 代理的额外请求头

  gboolean saved_controlling_mode;    // 用户设置的角色（对端可能在 ICE 交互中变化）
  guint timer_ta;                     // STUN 重传间隔，默认 20ms
  guint max_conn_checks;              // 最大连接检查对数量，默认 100
  gboolean force_relay;               // 强制仅使用 relay 候选
  guint stun_max_retransmissions;     // STUN 最大重传次数 (Rc)
  guint stun_initial_timeout;         // STUN 初始超时 (RTO)，毫秒
  guint stun_reliable_timeout;        // STUN 可靠超时，毫秒
  NiceNominationMode nomination_mode; // 提名模式（REGULAR/AGGRESSIVE）
  gboolean support_renomination;      // 支持 RENOMINATION STUN 属性
  guint idle_timeout;                 // 空闲超时，默认 5000ms

  /* ===== 网络和候选基础设施 ===== */
  GSList *local_addresses;            // 本地 NiceAddress 列表（host 候选的基础接口）
  GSList *streams;                    // Stream 对象链表（按添加顺序）
  GSList *pruning_streams;            // 正在异步清理的 Stream 对象链表
  GMainContext *main_context;         // GLib 主上下文（所有定时器/回调在此运行）
  guint next_candidate_id;            // 下一个候选 ID 的计数器
  guint next_stream_id;               // 下一个 stream ID 的计数器
  NiceRNG *rng;                       // 随机数生成器（凭据/tie-breaker 生成）

  /* ===== 候选发现 ===== */
  GSList *discovery_list;             // CandidateDiscovery 项链表（待处理的 STUN / TURN 发现任务）
  guint discovery_unsched_items;      // 未调度的发现项数量
  GSource *discovery_timer_source;    // 发现定时器 GSource
  GSList *refresh_list;               // CandidateRefresh 项链表（TURN 分配续期）
  GSList *pruning_refreshes;          // 正在关闭的 Refresh 项

  /* ===== 连接检查 ===== */
  GSList *triggered_check_queue;      // 待执行的触发检查（triggered check）队列
  GSource *conncheck_timer_source;    // 连接检查定时器 GSource
  GSource *keepalive_timer_source;    // 保活定时器 GSource
  guint64 tie_breaker;                // ICE 决胜值（8 字节随机数，决定 controlling/controlled 角色）
  gboolean controlling_mode;          // 当前实际使用的 controlling_mode（由 conncheck 算法动态确定）
  gboolean media_after_tick;          // keepalive tick 后是否收到过数据（用于检测连接故障）

  /* ===== 兼容性与传输选项 ===== */
  NiceCompatibility compatibility;    // 当前兼容模式
  gboolean reliable;                  // 是否可靠模式（启用 PseudoTCP）
  gboolean bytestream_tcp;            // 可靠 TCP 是否使用 bytestream 模式（而非 PseudoTCP）
  gboolean keepalive_conncheck;       // 是否使用 conncheck 替代普通 keepalive

  /* ===== UPnP ===== */
  gboolean upnp_enabled;              // 是否启用 UPnP 端口映射发现
  #ifdef HAVE_GUPNP
  GUPnPSimpleIgdThread* upnp;         // GUPnP IGD 线程代理
  guint upnp_timeout;                 // UPnP 发现超时，默认 200ms
  #endif

  /* ===== 软件标识 ===== */
  gchar *software_attribute;          // SOFTWARE STUN 属性值（RFC5245/WLM2009 模式下附带）

  /* ===== DNS 解析 ===== */
  GCancellable *stun_resolving_cancellable;  // 取消所有 STUN 主机名解析的 Cancellable
  GSList *stun_resolving_list;               // 正在进行的 STUN 主机名解析任务列表
  guint turn_resolving_count;                // 正在进行的 TURN 服务器解析任务计数

  /* ===== ICE 特性开关 ===== */
  gboolean use_ice_udp;               // 是否使用 ICE-UDP 候选
  gboolean use_ice_tcp;               // 是否使用 ICE-TCP 候选
  gboolean use_ice_trickle;           // 是否启用 Trickle ICE 模式

  /* ===== 运行时状态 ===== */
  guint conncheck_ongoing_idle_delay; // 连接检查空闲后的延迟计时器
  gboolean consent_freshness;         // 是否启用 RFC 7675 consent freshness
  gboolean close_forced;              // 是否启用强制关闭（不等待 TURN 响应）
  GTask *close_task;                  // nice_agent_close_async() 关联的异步任务
  gboolean recv_tos;                  // 是否接收 IP_TOS/IPV6_TCLASS
  GQueue pending_signals;             // 待发射的排队信号（延迟到 unlock 时发射）
};
```

**关键设计要点**：

1. **GMutex agent_mutex**：全代理级别的互斥锁。几乎每个公共 API 都以 `agent_lock()` / `agent_unlock_and_emit()` 包裹，保证线程安全。信号并非在锁内直接发射，而是先排入 `pending_signals` 队列，在 `agent_unlock_and_emit()` 中统一发射。这样避免了在持有锁时回调用户代码导致的死锁风险。

2. **GMainContext main_context**：所有基于时间的操作（定时器、IO source）都附加到此上下文。代理必须在调用 `nice_agent_new()` 时传入一个上下文，且该上下文的循环必须被持续迭代。

3. **tie_breaker**：ICE 规范 Section 5.2 "Determining Role" 中定义的无符号 64 位整数，由 `nice_rng_generate_bytes()` 生成 8 字节随机数。在连接检查过程中，双方通过比较 tie-breaker 值决定谁是 controlling agent、谁是 controlled agent。

4. **controlling_mode vs saved_controlling_mode**：`saved_controlling_mode` 存储用户通过属性设定的初始角色意向，而 `controlling_mode` 存储连接检查算法实际确定的当前角色。一旦任何 component 进入 DISCONNECTED 以上的状态，角色就由 conncheck 算法锁定，不可外部修改。

5. **pruning_streams**：Stream 的释放是异步的。当调用 `nice_agent_remove_stream()` 时，stream 先从 `streams` 列表移除并移到 `pruning_streams`，然后启动异步的 TURN refresh 清理，清理完成后才真正释放 Stream 对象。这种设计避免了在关闭过程中阻塞用户线程等待网络响应。

**私有内部 API（agent-priv.h 声明的内部函数）**：

| 函数 | 作用 |
|------|------|
| `agent_find_component()` | 根据 stream_id + component_id 查找 component（返回 bool） |
| `agent_find_stream()` | 根据 stream_id 查找 stream |
| `agent_gathering_done()` / `agent_signal_gathering_done()` | 内部触发候选收集完成 |
| `agent_signal_new_selected_pair()` | 内部触发选中新候选对信号 |
| `agent_signal_component_state_change()` | 内部触发状态变更信号 |
| `agent_signal_new_candidate()` / `agent_signal_new_remote_candidate()` | 内部触发新候选信号 |
| `agent_candidate_pair_priority()` | 计算候选对优先级 |
| `agent_timeout_add_with_context()` | 在 agent 的 GMainContext 中添加定时器 |
| `agent_to_ice_compatibility()` / `agent_to_turn_compatibility()` | 将代理兼容模式转换为 STUN 使用层兼容枚举 |
| `component_io_cb()` | 组件 socket 的 I/O 回调（接收数据的主入口） |
| `nice_debug_init()` | 通过 `NICE_DEBUG` 环境变量初始化调试 |

---

### 1.4 文件：agent.c -- GObject 生命周期

#### 1.4.1 nice_agent_class_init()

```c
static void nice_agent_class_init (NiceAgentClass *klass);
```

**作用**：GObject 类初始化，在首次实例化前由 GType 系统自动调用一次。

**关键逻辑**：
- 设置 `gobject_class->constructed`、`get_property`、`set_property`、`dispose` 虚函数
- 安装 **28 个 GObject 属性**（参见 1.1.3 节的属性表），所有属性都通过 `g_object_class_install_property()` 注册
- 注册 **10 个信号**（参见 1.1.2 节的信号表），使用 `g_signal_new()` 注册
- 最后调用 `nice_debug_init()` 通过 `NICE_DEBUG` 环境变量初始化调试级别

**属性注册要点**：
- `main-context` 和 `compatibility` 是 `CONSTRUCT_ONLY` 属性，必须在构造时提供且之后不可修改
- `reliable`、`nomination-mode`、`bytestream-tcp`、`ice-trickle`、`support-renomination`、`consent-freshness`、`close-forced`、`recv-tos` 也是 `CONSTRUCT_ONLY`
- 其他属性如 `stun-server`、`controlling-mode` 等可以在运行时通过 `g_object_set()` 修改

#### 1.4.2 nice_agent_init()

```c
static void nice_agent_init (NiceAgent *agent);
```

**作用**：实例初始化，每个 `NiceAgent` 对象创建时调用一次。

**关键逻辑**：
1. 初始化 ID 计数器：`next_candidate_id = 1`、`next_stream_id = 1`
2. 设置默认值（非构造参数，在 init 中设置）：
   - `stun_server_port = 3478`（DEFAULT_STUN_PORT）
   - `controlling_mode = TRUE`、`saved_controlling_mode = TRUE`（默认为 controlling agent）
   - `max_conn_checks = 100`（NICE_AGENT_MAX_CONNECTIVITY_CHECKS_DEFAULT）
   - `nomination_mode = AGGRESSIVE`、`support_renomination = FALSE`
   - `idle_timeout = 5000`（DEFAULT_IDLE_TIMEOUT）
   - `compatibility = RFC5245`
   - `reliable = FALSE`、`bytestream_tcp = FALSE`
   - `use_ice_udp = TRUE`、`use_ice_tcp = TRUE`
3. 创建随机数生成器：`agent->rng = nice_rng_new()`
4. 生成 tie-breaker：`priv_generate_tie_breaker()`（使用 `nice_rng_generate_bytes()` 生成 8 字节随机数）
5. 初始化信号队列：`g_queue_init (&agent->pending_signals)`
6. 初始化互斥锁：`g_mutex_init (&agent->agent_mutex)`
7. 创建 DNS 解析取消令牌：`agent->stun_resolving_cancellable = g_cancellable_new()`

#### 1.4.3 nice_agent_constructed()

```c
static void nice_agent_constructed (GObject *object);
```

**作用**：GObject 构造链中的 constructed 阶段，在所有构造属性设置完成后调用。

**关键逻辑**：
- Google 兼容模式下的可靠传输自动切换到 bytestream TCP：`if (agent->reliable && agent->compatibility == NICE_COMPATIBILITY_GOOGLE) agent->bytestream_tcp = TRUE;`
- 调用父类的 `constructed` 虚函数

#### 1.4.4 nice_agent_new_full()

```c
NiceAgent *nice_agent_new_full (
    GMainContext *ctx,
    NiceCompatibility compat,
    NiceAgentOption flags);
```

**原型**：创建 ICE 代理（完整版），自 0.1.15 版本引入。

**作用**：将所有 `NiceAgentOption` 位标志映射到对应的 GObject 构造属性，通过 `g_object_new()` 创建对象。

**关键逻辑（flags 映射）**：

| 标志位 | GObject 属性 | 值 |
|--------|------------|-----|
| `RELIABLE` | `reliable` | TRUE |
| `BYTESTREAM_TCP` | `bytestream-tcp` | TRUE |
| `REGULAR_NOMINATION` | `nomination-mode` | `REGULAR`（否则 `AGGRESSIVE`） |
| `LITE_MODE` | `full-mode` | FALSE（否则 TRUE） |
| `ICE_TRICKLE` | `ice-trickle` | TRUE |
| `SUPPORT_RENOMINATION` | `support-renomination` | TRUE |
| `CONSENT_FRESHNESS` | `consent-freshness` | TRUE |
| `CLOSE_FORCED` | `close-forced` | TRUE |
| `RECV_TOS` | `recv-tos` | TRUE |

**调用**：`g_object_new (NICE_TYPE_AGENT, "compatibility", compat, "main-context", ctx, ...)` 传入 9 个构造属性后返回 `NiceAgent *`。

#### 1.4.5 nice_agent_new()

```c
NiceAgent *nice_agent_new (GMainContext *ctx, NiceCompatibility compat);
```

**作用**：创建 ICE 代理（简化版），使用默认选项。

**关键逻辑**：直接转发到 `nice_agent_new_full(ctx, compat, NICE_AGENT_OPTION_NONE)`。

#### 1.4.6 nice_agent_new_reliable()

```c
NiceAgent *nice_agent_new_reliable (GMainContext *ctx, NiceCompatibility compat);
```

**作用**：创建可靠模式 ICE 代理。如果通过 ICE-UDP 建立连接，将透明使用 `PseudoTcpSocket` 保证消息可靠性。

**关键逻辑**：转发到 `nice_agent_new_full(ctx, compat, NICE_AGENT_OPTION_RELIABLE)`。

#### 1.4.7 nice_agent_dispose()

```c
static void nice_agent_dispose (GObject *object);
```

**作用**：GObject 销毁前清理，在对象最后一次引用被释放时调用。

**关键逻辑**（按顺序）：
1. **获取 agent 锁**
2. **清理发现定时器**：`discovery_free(agent)` -- 释放所有 `CandidateDiscovery` 项及其关联资源
3. **清理连接检查**：`conn_check_free(agent)` -- 释放连接检查状态和定时器
4. **取消 DNS 解析**：取消 `stun_resolving_cancellable`，释放 `stun_resolving_list`
5. **移除保活定时器**：`priv_remove_keepalive_timer(agent)`
6. **释放本地地址列表**：遍历 `local_addresses`，逐一 free，然后释放链表
7. **释放 TURN 刷新列表**：警告如果还有未清理的 refresh 项（建议先调用 `nice_agent_close_async()`），然后逐一 `refresh_free()`
8. **清理所有 Stream**：
   - 遍历 `streams` 列表：停止 UPnP、关闭 socket、`g_object_unref()` stream
   - 遍历 `pruning_streams` 列表：同上（异步清理未完成的 stream）
9. **清空待发射信号队列**：逐一 `free_queued_signal()`
10. **释放属性字符串**：`stun_server_ip`、`proxy_ip`、`proxy_username`、`proxy_password`、`proxy_extra_headers`
11. **释放随机数生成器**：`nice_rng_free(agent->rng)`
12. **释放 UPnP 代理**（如果编译时有 GUPnP）
13. **释放 SOFTWARE 属性字符串**
14. **释放 main_context 引用**
15. **释放 agent 锁**
16. **清理互斥锁**：`g_mutex_clear(&agent->agent_mutex)`
17. **调用父类 dispose**

---

### 1.5 文件：agent.c -- Stream 管理 API

#### 1.5.1 nice_agent_add_stream()

```c
guint nice_agent_add_stream (NiceAgent *agent, guint n_components);
```

**作用**：向代理添加一个包含 `n_components` 个 component 的数据流。

**关键逻辑**：
1. 参数校验：`n_components >= 1`
2. 获取 agent 锁
3. 调用 `nice_stream_new(agent->next_stream_id++, n_components, agent)` 创建 Stream 对象
4. 将 stream 追加到 `agent->streams` 链表
5. **可靠模式**：为每个 component 调用 `pseudo_tcp_socket_create()` 创建 PseudoTCP 包装
6. 调用 `nice_stream_initialize_credentials(stream, agent->rng)` 生成本地 ICE 凭据（ufrag 和 password）
7. 返回 stream ID（正整数），失败返回 0

**返回值**：新 stream 的 ID（保证为正整数）。

#### 1.5.2 nice_agent_remove_stream()

```c
void nice_agent_remove_stream (NiceAgent *agent, guint stream_id);
```

**作用**：移除并释放指定 stream。

**关键逻辑**：
1. 查找 stream
2. 调用内部函数 `nice_agent_remove_stream_internal()`：
   - 取消 stream 的 TURN 服务器解析任务
   - 停止 UPnP 映射
   - 从连接检查和发现列表中删除该 stream 的相关项
   - 从 `agent->streams` 移除，移入 `agent->pruning_streams`
   - 启动异步 TURN refresh 清理，清理完成后才调用 `nice_stream_close()` 和 `g_object_unref()`
   - 如果所有 stream 都被移除，移除保活定时器
   - 发射 `"streams-removed"` 信号

#### 1.5.3 nice_agent_set_stream_name() / nice_agent_get_stream_name()

```c
gboolean nice_agent_set_stream_name (NiceAgent *agent, guint stream_id, const gchar *name);
const gchar *nice_agent_get_stream_name (NiceAgent *agent, guint stream_id);
```

**作用**：设置/获取流的名称（用于 SDP m= 行）。

**set 关键逻辑**：
- 校验 name 是否为有效媒体类型：`"audio"`、`"video"`、`"text"`、`"application"`、`"message"`、`"image"`
- 不允许重复名称：遍历所有 stream，确保没有相同名称的其他 stream
- 将名称存储到 `stream->name`

**get 关键逻辑**：
- 直接返回 `stream->name`（字符串所有权仍归 stream）

---

### 1.6 文件：agent.c -- 候选管理 API

#### 1.6.1 nice_agent_gather_candidates()

```c
gboolean nice_agent_gather_candidates (NiceAgent *agent, guint stream_id);
```

**作用**：启动指定 stream 的候选收集过程，收集本地 host、server-reflexive (srflx)、relay (TURN) 候选。

**关键逻辑**（复杂的多步骤过程）：
1. 检查 stream 是否已开始收集（`stream->gathering_started`），若已开始则忽略
2. **自动发现本地地址**：如果用户未通过 `nice_agent_add_local_address()` 添加地址，则调用 `nice_interfaces_get_local_ips()` 自动收集所有本地 IP 地址
3. 对每个 component 进行三种类型候选的创建：
   - **Host 候选**：对每个本地地址分别创建 UDP / TCP_ACTIVE / TCP_PASSIVE 候选。如设置了端口范围（`min_port` / `max_port`），则在范围内随机选择端口。调用 `discovery_add_local_host_candidate()` 创建 socket 并加入候选列表
   - **Srflx 候选**：如果设置了 STUN 服务器（`agent->stun_server_ip`），则异步解析 STUN 服务器主机名，解析完成后启动 STUN binding discovery
   - **Relay 候选**：遍历 `component->turn_servers`，对每个已解析的 TURN 服务器（非链路本地地址的目标），调用 `priv_add_new_candidate_discovery_turn()` 启动 TURN allocate discovery
4. 对非 link-local 地址且非 TCP_PASSIVE 类型，启动 UPnP 端口映射发现
5. 设置 `stream->gathering = TRUE` 和 `stream->gathering_started = TRUE`
6. 发射 `"new-candidate"` / `"new-candidate-full"` 信号（force_relay 模式下仅发射 relay 候选）
7. 若所有发现项已在调用期间完成（`discovery_unsched_items == 0`），直接调用 `agent_gathering_done()` 发射 `"candidate-gathering-done"` 信号。否则调用 `discovery_schedule()` 调度异步发现

**返回值**：`TRUE` 成功，`FALSE` 表示 stream ID 无效或无法在任何接口/端口上分配 host 候选。

#### 1.6.2 nice_agent_set_remote_candidates()

```c
int nice_agent_set_remote_candidates (
    NiceAgent *agent, guint stream_id, guint component_id,
    const GSList *candidates);
```

**作用**：设置、追加或更新远端候选。每个候选触发内部 pairing 并可能启动 connection check。

**关键逻辑**：
1. 查找 stream 和 component
2. 遍历 candidates 链表，对每个有效地址的候选调用 `priv_add_remote_candidate()` 添加到 component 的 `remote_candidates` 列表
3. `priv_add_remote_candidate()` 会：
   - 检查 `max_conn_checks` 限制（超出限制的候选不会被配成检查对）
   - 与现有的全部本地候选进行配对，加入 conncheck 待检查列表
   - 触发 `conn_check_remote_candidates_set()` 调度连接检查
4. 返回成功添加的候选数量，负数表示错误

**注意**：自 0.1.3 起，不需要等待 `candidate-gathering-done` 信号即可设置远端候选。新发现的本地候选会自动与已有远端候选配对。

#### 1.6.3 nice_agent_get_local_candidates() / nice_agent_get_remote_candidates()

```c
GSList *nice_agent_get_local_candidates  (NiceAgent *agent, guint stream_id, guint component_id);
GSList *nice_agent_get_remote_candidates (NiceAgent *agent, guint stream_id, guint component_id);
```

**作用**：获取本地/远端候选列表。

**关键逻辑**：
- 遍历 `component->local_candidates`（或 `remote_candidates`）链表
- 调用 `nice_candidate_copy()` 浅拷贝每个候选
- 返回的 `GSList` 和其中的 `NiceCandidate` 对象所有权归调用者（需用 `g_slist_free_full()` + `nice_candidate_free()` 释放）
- `force_relay` 模式下，`get_local_candidates` 仅返回 `RELAYED` 类型的候选

#### 1.6.4 nice_agent_set_local_credentials() / nice_agent_get_local_credentials()

```c
gboolean nice_agent_set_local_credentials (NiceAgent *agent, guint stream_id,
    const gchar *ufrag, const gchar *pwd);
gboolean nice_agent_get_local_credentials (NiceAgent *agent, guint stream_id,
    gchar **ufrag, gchar **pwd);
```

**作用**：设置/获取本地 ICE 凭据（ufrag 和 password）。

**关键逻辑**：
- set：仅在 ICE 协商开始前有效。将 ufrag/pwd 拷贝到 `stream->local_ufrag` / `stream->local_password`（最大长度 `NICE_STREAM_MAX_UFRAG` / `NICE_STREAM_MAX_PWD`）。如果不显式设置，libnice 会在 `add_stream` 时自动生成随机凭据。
- get：`g_strdup()` 复制 `stream->local_ufrag` / `stream->local_password`，调用者需 `g_free()`。

#### 1.6.5 nice_agent_set_remote_credentials() / nice_agent_get_remote_credentials()

```c
gboolean nice_agent_set_remote_credentials (NiceAgent *agent, guint stream_id,
    const gchar *ufrag, const gchar *pwd);
gboolean nice_agent_get_remote_credentials (NiceAgent *agent, guint stream_id,
    gchar **ufrag, gchar **pwd);
```

**作用**：设置/获取远端 ICE 凭据。

**set 关键逻辑**：
- 将 ufrag/pwd 拷贝到 `stream->remote_ufrag` / `stream->remote_password`
- **重要**：调用 `conn_check_remote_credentials_set(agent, stream)` 将凭据应用到 stream 中所有已存在的远端候选上，检查是否已具备启动连接检查的全部条件（本地候选 + 远端候选 + 远端凭据 = 全部就绪）

#### 1.6.6 nice_agent_set_relay_info()

```c
gboolean nice_agent_set_relay_info (NiceAgent *agent, guint stream_id, guint component_id,
    const gchar *server_ip, guint server_port, const gchar *username,
    const gchar *password, NiceRelayType type);
```

**作用**：为候选发现过程设置 TURN 中继服务器。可多次调用以添加多个 relay 服务器（例如同时添加 TCP 和 UDP TURN 服务器各一个）。

**关键逻辑**：
- 将 relay 配置存储到 component 的 `turn_servers` 列表
- 支持主机名解析：TURN 服务器地址如果是主机名（而非 IP），则通过 `GResolver` 异步解析。解析回调中为每个解析出的 IP 地址创建 `TurnServer` 对象
- 如果 stream 的 gathering 已经开始，立即使用已有 host 候选启动 TURN discovery
- 通过 `resolve_turn_in_context()` 确保 DNS 解析在 agent 的 `main_context` 中执行

---

### 1.7 文件：agent.c -- 数据收发 API

#### 1.7.1 nice_agent_send()

```c
gint nice_agent_send (NiceAgent *agent, guint stream_id, guint component_id,
    guint len, const gchar *buf);
```

**作用**：通过指定的 stream/component 发送数据。

**关键逻辑**：
1. 将单缓冲区包装为 `NiceOutputMessage`
2. 转发到内部函数 `nice_agent_send_messages_nonblocking_internal()` 带 `allow_partial=TRUE`
3. 内部实现：
   - 查找 component 的 selected pair
   - **可靠传输**（socket 支持可靠发送）：加上 RFC4571 帧头（2 字节长度前缀），调用 `nice_socket_send_messages_reliable()`
   - **非可靠传输**：先尝试 `nice_socket_send_reliable()`，失败后回退到 `nice_socket_send()`
   - 如果发送缓冲区满，在可靠模式下触发 `"reliable-transport-writable"` 信号

**返回值**：发送的字节数，或 -1（可靠模式下缓冲区满 / 尚未连接 / 无效 ID）。

#### 1.7.2 nice_agent_attach_recv() / nice_agent_attach_recv_ex()

```c
gboolean nice_agent_attach_recv (NiceAgent *agent, guint stream_id, guint component_id,
    GMainContext *ctx, NiceAgentRecvFunc func, gpointer data);

gboolean nice_agent_attach_recv_ex (NiceAgent *agent, guint stream_id, guint component_id,
    GMainContext *ctx, NiceAgentRecvFuncEx func, gpointer data, GDestroyNotify notify);
```

**作用**：将 component 的 socket 附加到 GLib 主循环上下文，当数据到达时回调用户函数。这是**回调模式**的数据接收方式。

**关键逻辑**：
1. 查找 stream 和 component
2. 如果 `ctx` 为 NULL，使用 `g_main_context_default()`
3. 调用 `nice_component_set_io_context()` 和 `nice_component_set_io_callback()` 设置 component 的 I/O 上下文和回调
4. 底层通过 `component_io_cb()` 回调接收数据，内部处理 STUN 数据包（不传递给用户），仅将非 STUN 数据传递给用户的回调函数
5. `attach_recv` 是对 `attach_recv_ex` 的封装，通过兼容适配器 `compat_recv_func` 桥接 `NiceAgentRecvFunc` 到 `NiceAgentRecvFuncEx`
6. 回调模式可通过传入 `func=NULL` 来 detach 已有回调

#### 1.7.3 nice_agent_recv_messages() / nice_agent_recv()

```c
gint nice_agent_recv_messages (NiceAgent *agent, guint stream_id, guint component_id,
    NiceInputMessage *messages, guint n_messages, GCancellable *cancellable, GError **error);

gssize nice_agent_recv (NiceAgent *agent, guint stream_id, guint component_id,
    guint8 *buf, gsize buf_len, GCancellable *cancellable, GError **error);
```

**作用**：阻塞接收数据。这是**阻塞模式**的数据接收方式，可在独立线程中使用。

**关键逻辑**：
- `nice_agent_recv()` 是 `nice_agent_recv_messages()` 的单消息封装版（包装为 `n_messages=1` 的调用）
- 阻塞模式内部通过 `nice_socket_recv_messages()` 直接读取 socket
- STUN 数据包在内部被拦截处理，不会返回给调用者
- `cancellable` 允许从另一线程取消等待

#### 1.7.4 非阻塞发送/接收变体

```c
gint nice_agent_send_messages_nonblocking (NiceAgent *agent, guint stream_id,
    guint component_id, const NiceOutputMessage *messages, guint n_messages,
    GCancellable *cancellable, GError **error);

gint nice_agent_recv_messages_nonblocking (NiceAgent *agent, guint stream_id,
    guint component_id, NiceInputMessage *messages, guint n_messages,
    GCancellable *cancellable, GError **error);

gssize nice_agent_recv_nonblocking (NiceAgent *agent, guint stream_id,
    guint component_id, guint8 *buf, gsize buf_len,
    GCancellable *cancellable, GError **error);
```

**作用**：非阻塞版本的批量发送/接收。如果不立即可操作则返回 `G_IO_ERROR_WOULD_BLOCK` 而非阻塞等待。

**关键逻辑**：
- `send_messages_nonblocking_internal(..., allow_partial=FALSE)` -- 返回成功发送的消息数量（非字节数）
- `nice_agent_recv_messages_blocking_or_nonblocking(..., blocking=FALSE)` -- 尽可能接收但不阻塞
- `nice_agent_send()` 内部调用 `nice_agent_send_messages_nonblocking_internal(..., allow_partial=TRUE)` 允许部分发送

**两种接收模式的设计**：

| 特性 | `attach_recv()` （回调模式） | `recv_messages()` （阻塞模式） |
|------|---------------------------|----------------------------|
| 线程模型 | 单线程，由 GMainLoop 驱动 | 独立线程中阻塞循环 |
| I/O 触发 | GLib GSource I/O 回调 | 直接调用 socket recv |
| 回调开销 | 每包一次回调 | 批量返回，减少回调次数 |
| 互斥 | 同一 component 不能同时使用两种模式 | 同一 component 不能同时使用两种模式 |

---

### 1.8 文件：agent.c -- 查询与配置 API

#### 1.8.1 nice_agent_get_component_state()

```c
NiceComponentState nice_agent_get_component_state (
    NiceAgent *agent, guint stream_id, guint component_id);
```

**作用**：查询指定 component 的当前状态。

**关键逻辑**：读取 `component->state`，如果 component 不存在则返回 `NICE_COMPONENT_STATE_FAILED`。

#### 1.8.2 nice_agent_get_selected_pair()

```c
gboolean nice_agent_get_selected_pair (NiceAgent *agent, guint stream_id,
    guint component_id, NiceCandidate **local, NiceCandidate **remote);
```

**作用**：获取当前选中的候选对。

**关键逻辑**：返回 `component->selected_pair.local` 和 `component->selected_pair.remote`。如果没有选中候选对则返回 FALSE。

#### 1.8.3 nice_agent_get_selected_socket()

```c
GSocket *nice_agent_get_selected_socket (NiceAgent *agent, guint stream_id,
    guint component_id);
```

**作用**：获取选中候选对的底层 GSocket。用于传统应用集成或接管 socket 控制权。

**注意**：如果选中候选是 TURN 或代理类型的候选，则返回 NULL（因为数据被封装，应用无法直接使用底层 socket）。

#### 1.8.4 nice_agent_set_selected_pair() / nice_agent_set_selected_remote_candidate()

```c
gboolean nice_agent_set_selected_pair (NiceAgent *agent, guint stream_id,
    guint component_id, const gchar *lfoundation, const gchar *rfoundation);

gboolean nice_agent_set_selected_remote_candidate (NiceAgent *agent, guint stream_id,
    guint component_id, NiceCandidate *candidate);
```

**作用**：强制设置选中的候选对或远端候选，**跳过所有后续 ICE 处理**。

**关键逻辑**：
- 调用后禁用 connection check 和状态机更新，但 keepalive 继续发送
- 用于非 ICE 兼容的候选（如传统的非 ICE 端点）或连接检查失败时的强制选择

#### 1.8.5 nice_agent_set_port_range()

```c
void nice_agent_set_port_range (NiceAgent *agent, guint stream_id,
    guint component_id, guint min_port, guint max_port);
```

**作用**：限制本地候选的端口范围。必须在 `nice_agent_gather_candidates()` 之前调用。

#### 1.8.6 nice_agent_add_local_address()

```c
gboolean nice_agent_add_local_address (NiceAgent *agent, NiceAddress *addr);
```

**作用**：添加一个本地地址，供 host 候选收集时使用。如果从未调用此函数，libnice 会在 `gather_candidates()` 时自动发现本地地址。

#### 1.8.7 nice_agent_restart() / nice_agent_restart_stream()

```c
gboolean nice_agent_restart (NiceAgent *agent);
gboolean nice_agent_restart_stream (NiceAgent *agent, guint stream_id);
```

**作用**：执行 ICE 重启（RFC 5245 第 9.1.1.1 节和第 9.2.1.1 节）。

**关键逻辑**：
- `restart()` 对所有 stream 执行重启（生成新的 tie-breaker）
- `restart_stream()` 对单个 stream 执行重启（保留 tie-breaker）
- 两者都调用 `nice_stream_restart()`：重置本地凭据、清除远端候选、重置 conncheck 列表
- 如果启用了 consent freshness（RFC 7675），重启会恢复本地 consent 状态

#### 1.8.8 nice_agent_set_stream_tos()

```c
void nice_agent_set_stream_tos (NiceAgent *agent, guint stream_id, gint tos);
```

**作用**：设置 stream 的所有 socket 的 IP_TOS / IPV6_TCLASS 字段值。

#### 1.8.9 nice_agent_set_software()

```c
void nice_agent_set_software (NiceAgent *agent, const gchar *software);
```

**作用**：设置 STUN SOFTWARE 属性值（添加到连接检查消息中）。

**关键逻辑**：
- 仅在 RFC5245 和 WLM2009 兼容模式下发送
- 自动追加 libnice 版本号：`g_strdup_printf("%s/%s", software, PACKAGE_STRING)`
- 最大 128 字符，UTF-8 编码
- 调用后重置所有已创建的 StunAgent 以应用新 SOFTWARE 属性

#### 1.8.10 nice_agent_close_async()

```c
void nice_agent_close_async (NiceAgent *agent, GAsyncReadyCallback callback,
    gpointer callback_data);
```

**作用**：异步关闭代理，确保 TURN 中继端口被正确移除（而非遗留）。

**关键逻辑**：
- 创建 GTask
- 取消所有未完成的 STUN 解析
- 调用 `nice_agent_remove_streams()` 移除所有 stream
- 移除 stream 的过程触发 TURN refresh 清理，完成后回调用户

---

### 1.9 文件：agent.c -- SDP API

#### 1.9.1 nice_agent_generate_local_sdp() / nice_agent_generate_local_stream_sdp()

```c
gchar *nice_agent_generate_local_sdp (NiceAgent *agent);
gchar *nice_agent_generate_local_stream_sdp (NiceAgent *agent, guint stream_id,
    gboolean include_non_ice);
```

**作用**：生成本地 SDP 字符串，包含所有 stream 的 ICE 候选和凭据。

**关键逻辑**：
- `generate_local_sdp()` 遍历所有 stream，对每个调用内部 `_generate_stream_sdp()` 函数
- `_generate_stream_sdp()` 生成：
  - `m=<media> <port> <proto> <fmt>` 行（media 来自 stream name，port 使用默认候选的端口）
  - 如果 `include_non_ice`：`c=IN IP4/IP6 <addr>` 和 `a=rtcp:<port>` 行
  - `a=ice-ufrag:<ufrag>` 和 `a=ice-pwd:<pwd>` 行
  - 对每个候选生成 `a=candidate:...` 行
- 默认候选选择最低优先级的 IPv4 candidate（因为它是"最长路径但最可能成功"的策略）

#### 1.9.2 nice_agent_generate_local_candidate_sdp()

```c
gchar *nice_agent_generate_local_candidate_sdp (NiceAgent *agent,
    NiceCandidate *candidate);
```

**作用**：生成单个本地候选的 SDP 行（用于 Trickle ICE）。

**SDP 格式**：`a=candidate:<foundation> <component> <transport> <priority> <addr> <port> typ <type> [raddr <raddr> rport <rport>] [tcptype <tcptype>]`

#### 1.9.3 nice_agent_parse_remote_sdp() / nice_agent_parse_remote_stream_sdp()

```c
int nice_agent_parse_remote_sdp (NiceAgent *agent, const gchar *sdp);
GSList *nice_agent_parse_remote_stream_sdp (NiceAgent *agent, guint stream_id,
    const gchar *sdp, gchar **ufrag, gchar **pwd);
```

**作用**：解析远端 SDP，提取候选和凭据。

**`parse_remote_sdp()` 关键逻辑**：
1. 按 `\n` 分割 SDP
2. 逐行解析：
   - `m=` 行：切换到下一个 stream（按 stream 创建顺序匹配）
   - `a=ice-ufrag:`：提取远端 ufrag 写入 `stream->remote_ufrag`
   - `a=ice-pwd:`：提取远端 password 写入 `stream->remote_password`
   - `a=candidate:`：调用 `nice_agent_parse_remote_candidate_sdp()` 解析候选，然后调用 `_set_remote_candidates_locked()` 设置到对应的 component
3. 返回成功添加的候选数量，负数表示错误

**`parse_remote_stream_sdp()` 关键逻辑**：
- 仅解析单个 stream 的 SDP（不处理 `m=` 行关联）
- 返回解析出的候选链表（调用者通过 `ufrag`/`pwd` 输出参数获取凭据）

#### 1.9.4 nice_agent_parse_remote_candidate_sdp()

```c
NiceCandidate *nice_agent_parse_remote_candidate_sdp (NiceAgent *agent,
    guint stream_id, const gchar *sdp);
```

**作用**：解析单行 SDP 候选字符串为 `NiceCandidate` 对象。

**解析格式**：`a=candidate:1 1 UDP 2013266431 10.0.1.1 59221 typ host`

---

### 1.10 GIO I/O 流 API

#### 1.10.1 nice_agent_get_io_stream()

```c
GIOStream *nice_agent_get_io_stream (NiceAgent *agent, guint stream_id,
    guint component_id);
```

**作用**：获取 GIOStream 包装器（仅在可靠模式下可用）。

**关键逻辑**：
- 查找 component，如果 `component->iostream` 尚未创建，则调用 `nice_io_stream_new()` 创建
- `NiceIOStream` 内部包含 `NiceInputStream` 和 `NiceOutputStream`，它们分别实现 `GPollableInputStream` 和 `GPollableOutputStream` 接口
- 返回的 GIOStream 引用计数已加 1，调用者需 `g_object_unref()`

---

### 1.11 其他 API

#### nice_agent_forget_relays()

```c
gboolean nice_agent_forget_relays (NiceAgent *agent, guint stream_id, guint component_id);
```
清除通过 `nice_agent_set_relay_info()` 添加的所有 relay 服务器。当前连接不会立即断连（直到重启或协商到其他路径）。

#### nice_agent_peer_candidate_gathering_done()

```c
gboolean nice_agent_peer_candidate_gathering_done (NiceAgent *agent, guint stream_id);
```
通知 agent 远端已完成候选收集（Trickle ICE 中使用）。触发未完成 conncheck 的 component 最终转为 `FAILED` 状态。

#### nice_agent_consent_lost()

```c
gboolean nice_agent_consent_lost (NiceAgent *agent, guint stream_id, guint component_id);
```
通知代理对端已撤销接收许可（RFC 7675）。需要 `NICE_AGENT_OPTION_CONSENT_FRESHNESS` 已启用。

#### nice_agent_get_sockets()

```c
GPtrArray *nice_agent_get_sockets (NiceAgent *agent, guint stream_id, guint component_id);
```
获取 component 的所有底层 GSocket 数组。用于在 socket 创建后设置额外的 socket 选项。

#### nice_agent_get_default_local_candidate()

```c
NiceCandidate *nice_agent_get_default_local_candidate (NiceAgent *agent,
    guint stream_id, guint component_id);
```
返回推荐的默认候选（用于非 ICE 兼容客户端），通常是最低优先级的候选（最长路径但最可能成功）。对于 RTCP component，选择与 RTP 候选相同 foundation 的候选。

---

## 调用关系（第一部分）

```
public API                      内部分发
─────────────────────────────────────────────────────
nice_agent_new() ───────────→ nice_agent_new_full()
nice_agent_new_reliable() ──→ nice_agent_new_full()
nice_agent_new_full() ──────→ g_object_new() → nice_agent_init() → nice_agent_constructed()

nice_agent_add_stream() ────→ nice_stream_new() + pseudo_tcp_socket_create() (if reliable)
nice_agent_remove_stream() ─→ nice_agent_remove_stream_internal() → refresh_prune_stream_async()
nice_agent_gather_candidates() → discovery_add_local_host_candidate()
                              → priv_add_new_candidate_discovery_turn()
                              → priv_add_upnp_discovery()
                              → discovery_schedule()
nice_agent_set_remote_candidates() → priv_add_remote_candidate()
                                  → conn_check_remote_candidates_set()
nice_agent_set_remote_credentials() → conn_check_remote_credentials_set()

nice_agent_send() ──────────→ nice_agent_send_messages_nonblocking_internal()
                              → nice_socket_send() / nice_socket_send_reliable()

nice_agent_attach_recv() ───→ nice_agent_attach_recv_ex()
                              → nice_component_set_io_context()
                              → nice_component_set_io_callback()

nice_agent_recv_messages() ─→ nice_agent_recv_messages_blocking_or_nonblocking()
                              → component_io_cb() (indirect, via socket read)

nice_agent_restart() ───────→ nice_stream_restart() (for each stream)

nice_agent_dispose() ───────→ discovery_free() + conn_check_free()
                              → refresh_free() (for each refresh)
                              → nice_stream_close() (for each stream)
```

**调用模式总结**：
- 公共 API 函数统一遵循 `agent_lock() → 核心逻辑 → agent_unlock_and_emit()` 模式
- 信号不直接在锁内发射，而是排入 `pending_signals` 队列，在 `agent_unlock_and_emit()` 中发射，避免在持有锁时回调用户代码导致的死锁
- 所有 "业务逻辑"（连接检查、候选发现、TURN 刷新等）都在 agent 锁的保护下运行，但通过 GLib 定时器和 I/O source 异步驱动
- agent.c 的公共 API 函数主要作为**分发器**，将参数传递给 `stream.c`、`component.c`、`conncheck.c`、`discovery.c` 等内部模块

---

## 第二部分：信号系统与定时器驱动

### 2.1 信号发射机制

libnice 的信号发射遵循一个核心设计原则：**信号不在 agent 锁内直接发射，而是先排入队列，在解锁后统一发射**。这避免了在持有互斥锁时回调用户代码可能导致的死锁风险。

#### 2.1.1 QueuedSignal 结构与信号排队

```c
// agent.c:188-193
typedef struct {
  guint signal_id;        // GObject 信号 ID
  GSignalQuery query;      // 信号元信息（参数类型等）
  GValue *params;          // 参数数组（params[0]=agent, params[1..n]=信号参数）
} QueuedSignal;
```

**核心函数调用链**：

**`agent_queue_signal()`**（agent.c:243） -- 将信号排入 pending_signals 队列：
1. 通过 `g_signal_query()` 获取信号的参数个数和类型
2. 分配 `QueuedSignal` 和 `GValue` 参数数组
3. `params[0]` 固定为 agent 对象自身
4. 使用 `G_VALUE_COLLECT_INIT` 从可变参数中收集信号参数
5. 将信号推入 `agent->pending_signals` 队列尾部

**`agent_unlock_and_emit()`**（agent.c:213） -- 解锁并发射所有排队信号：
1. 将 `agent->pending_signals` 队列取出，并初始化一个新的空队列（原子交换）
2. 检查是否有待完成的 `close_task`（如果所有异步清理已完成则取出）
3. **释放 agent 锁**（`agent_unlock()`）
4. 遍历队列，通过 `g_signal_emitv()` 逐一发射信号（此时已无锁，用户回调可安全执行）
5. 每发射完一个信号立即 `free_queued_signal()` 释放
6. 如果有待完成的 `close_task`，调用 `g_task_return_boolean()` 完成异步关闭

**`free_queued_signal()`**（agent.c:196） -- 释放排队信号资源，释放所有 GValue 及参数内存。

#### 2.1.2 "component-state-changed" 信号

**发射函数**：`agent_signal_component_state_change()`（agent.c:2678）

**发射时机**：任何内部逻辑需要变更 component 状态时调用。

**关键逻辑**：
1. 通过 `agent_find_component()` 查找 stream 和 component
2. **状态转换校验**：通过 `TRANSITION(OLD, NEW)` 宏 + `g_assert()` 验证状态转换合法性。合法转换包括：
   - DISCONNECTED -> GATHERING / CONNECTING
   - GATHERING -> CONNECTING / FAILED
   - CONNECTING -> CONNECTED / FAILED / GATHERING (ICE restart)
   - CONNECTED -> READY / CONNECTING (TCP socket 断开)
   - READY -> CONNECTED (re-nomination)
   - FAILED -> CONNECTING / GATHERING (新候选到达)
   - 任意状态 -> FAILED（组件失败的最终状态）
   - 任意状态 -> GATHERING（ICE restart）
3. 将新状态写入 `component->state`
4. 可靠模式下调用 `process_queued_tcp_packets()` 处理积压的 TCP 数据包
5. 排队发射信号，参数：`(stream_id, component_id, new_state)`

**信号参数**：`guint stream_id, guint component_id, guint state`

#### 2.1.3 "candidate-gathering-done" 信号

**发射函数**：`agent_signal_gathering_done()`（agent.c:2483）

**触发路径**（共 6 个调用点）：
- `agent_gathering_done()` 中：当 `discovery_timer_source == NULL && !upnp_running && !dns_resolution_ongoing` 时
- 多个 UPnP 和 discovery 完成回调点

**关键逻辑**：
- `agent_gathering_done()` 检查所有条件满足后，才调用 `agent_signal_gathering_done()`
- `agent_signal_gathering_done()` 遍历所有 `agent->streams`，对每个 `gathering == TRUE` 的 stream：
  - 设置 `stream->gathering = FALSE`
  - 排队发射 `"candidate-gathering-done"` 信号，参数：`stream->id`
- 信号按 stream 逐个发射（每个 stream 一个信号）

**信号参数**：`guint stream_id`

#### 2.1.4 "new-candidate" 信号

**发射函数**：`agent_signal_new_candidate()`（agent.c:2639）

**发射时机**：每发现一个新本地候选时发射。调用位置包括：
- `discovery.c` 中 host 候选创建成功时
- `discovery.c` 中 STUN binding 发现 server-reflexive 候选成功时
- `discovery.c` 中 TURN allocate 发现 relay 候选成功时
- `conncheck.c` 中 peer-reflexive 候选发现时

**关键逻辑**：同时发射旧版和新版两个信号：
```c
agent_queue_signal (agent, signals[SIGNAL_NEW_CANDIDATE_FULL], candidate);
agent_queue_signal (agent, signals[SIGNAL_NEW_CANDIDATE],
    candidate->stream_id, candidate->component_id, candidate->foundation);
```

**信号参数**：
- 旧版 `"new-candidate"`：`guint stream_id, guint component_id, gchar *foundation`（已弃用）
- 新版 `"new-candidate-full"`：`NiceCandidate *candidate`（推荐使用）

#### 2.1.5 "new-remote-candidate" 信号

**发射函数**：`agent_signal_new_remote_candidate()`（agent.c:2647）

**发射时机**：收到远端候选时（peer-reflexive 候选发现），同样同时发射新旧两个版本。

**信号参数**：
- 旧版 `"new-remote-candidate"`：`guint stream_id, guint component_id, gchar *foundation`（已弃用）
- 新版 `"new-remote-candidate-full"`：`NiceCandidate *candidate`

#### 2.1.6 "new-selected-pair" 信号

**发射函数**：`agent_signal_new_selected_pair()`（agent.c:2565）

**发射时机**：conncheck 算法选中新的候选对时调用。调用位置包括：
- `conncheck.c` 中提名成功选中新候选对
- `agent.c` 中 `nice_agent_set_selected_pair()` 和 `nice_agent_set_selected_remote_candidate()` 强制设置时

**关键逻辑**：
1. 查找 stream 和 component
2. 如果 local candidate 是 UDP TURN 类型，调用 `nice_udp_turn_socket_set_peer()` 设置 TURN 对端
3. **可靠模式**：
   - 如果 socket 不支持可靠传输，则创建 PseudoTCP socket
   - 处理积压的 TCP 数据包（SYN/SYNACK/ACK）
   - 调用 `pseudo_tcp_socket_connect()` 连接
   - 启动 TCP 时钟（`adjust_tcp_clock()`）
4. 调试输出本地/远端选中候选对详情（地址、传输类型、候选类型）
5. 同时排队发射新旧两个 `new-selected-pair` 信号
6. 发射 **"reliable-transport-writable"** 信号（通过 `agent_signal_socket_writable()`）

**信号参数**：
- 旧版 `"new-selected-pair"`：`guint stream_id, guint component_id, gchar *lfoundation, gchar *rfoundation`（已弃用）
- 新版 `"new-selected-pair-full"`：`guint stream_id, guint component_id, NiceCandidate *lcandidate, NiceCandidate *rcandidate`

#### 2.1.7 "initial-binding-request-received" 信号

**发射函数**：`agent_signal_initial_binding_request_received()`（agent.c:2498）

**发射时机**：收到首个来自对端的 STUN Binding Request 时。在 `conncheck.c` 中处理入站 STUN 消息时调用。

**关键逻辑**：使用 `stream->initial_binding_request_received` 布尔标志保证仅发射一次。首次调用时：
- 设置 `stream->initial_binding_request_received = TRUE`
- 排队发射信号，参数：`stream->id`

**信号参数**：`guint stream_id`

#### 2.1.8 其他信号

**"reliable-transport-writable"**：在 `agent_signal_new_selected_pair()` 末尾发射，表示可靠模式下 socket 变为可写（send buffer 从满到可写，或连接刚建立）。

**"streams-removed"**：在 `nice_agent_remove_stream_internal()` 中排队发射（agent.c:4088）。
- 参数：`guint[]` 零终止数组（`stream_ids`）
- 通过 `g_memdup()` 复制 stream_id 数组传递

---

### 2.2 GLib 定时器系统

libnice 的所有定时操作都基于 GLib 的 `GSource` 机制，通过统一的辅助函数 `agent_timeout_add_with_context()` 创建并附加到 agent 的 `GMainContext` 上。所有定时器共享同一个 GMainContext，由 GLib 主循环统一调度。

#### 2.2.1 agent_timeout_add_with_context() 定时器辅助函数

**原型**：`agent.c:6958`
```c
void agent_timeout_add_with_context (NiceAgent *agent,
    GSource **out, const gchar *name, guint interval,
    NiceTimeoutLockedCallback function, gpointer user_data);
```

以及秒级版本（agent.c:6966）：
```c
void agent_timeout_add_with_context_seconds (NiceAgent *agent,
    GSource **out, const gchar *name, guint interval,
    NiceTimeoutLockedCallback function, gpointer user_data);
```

**内部实现**（`agent_timeout_add_with_context_internal`，agent.c:6925）：
1. 如果 `*out` 已有旧的 source，先 `g_source_destroy()` + `g_source_unref()` 销毁
2. 创建新 source：`g_timeout_source_new(interval)`（毫秒级）或 `g_timeout_source_new_seconds(interval)`（秒级）
3. 设置 source 名称（方便 gdb 调试）
4. 包装回调数据（agent + 回调函数 + user_data）到 `TimeoutData` 结构体
5. 设置 source 的回调函数为 `timeout_cb`（该回调在 agent 锁的保护下执行用户提供的 `function`）
6. 将 source 附加到 `agent->main_context`：`g_source_attach(source, agent->main_context)`
7. 将 source 指针返回给 `*out`（供后续销毁用）

**设计要点**：每个定时器都通过一个 `GSource **out` 参数追踪，确保同一时刻只有一个同类型定时器在运行 -- 创建新定时器时会自动销毁旧的。

#### 2.2.2 Conncheck 定时器

**创建位置**：`conncheck.c:1728-1731` -- `priv_schedule_next()` 函数

```c
agent_timeout_add_with_context (agent, &agent->conncheck_timer_source,
    "Connectivity check schedule", agent->timer_ta,
    priv_conn_check_tick_agent_locked, NULL);
```

**超时间隔**：`agent->timer_ta`，默认值 `NICE_AGENT_TIMER_TA_DEFAULT = 20ms`（可通过 `stun-pacing-timer` 属性配置）。

**回调函数**：`priv_conn_check_tick_agent_locked()`（conncheck.c:1142）

**每次 tick 的执行流程**（按优先级顺序，每次 tick 仅发送一个 STUN 请求以保证 pacing）：
1. **处理 triggered check**：`priv_conn_check_triggered_check()` -- 最高优先级，处理由入站检查触发的立即检查
2. **处理进行中的 STUN 事务**：`priv_conn_check_tick_stream()` -- 检查进行中的事务是否超时、是否需要重传
3. **处理普通检查**：`priv_conn_check_ordinary_check()` -- 按优先级顺序启动新的连接检查
4. **尝试提名**：`priv_conn_check_tick_stream_nominate()` -- 在 aggressive 模式下，对成功的检查立即提名

**定时器停止条件**：`conncheck.c:1201-1216`
- 当没有 STUN 请求被发送且 `conncheck_ongoing_idle_delay >= agent->idle_timeout`（默认 5000ms）时停止
- 停止前检查是否有失败组件：`priv_update_check_list_failed_components()`
- 调用 `conn_check_stop()` 销毁 timer source

**STUN 事务超时计算**：`priv_compute_conncheck_timer()`（conncheck.c:2805）
```c
rto = agent->timer_ta * waiting_and_in_progress;
return MAX (rto, STUN_TIMER_DEFAULT_TIMEOUT);
// STUN_TIMER_DEFAULT_TIMEOUT = 500ms（最小超时）
```
超时与正在等待和进行中的检查对数量成正比，但最低为 500ms。

#### 2.2.3 Discovery 定时器

**创建位置**：`discovery.c:1472` -- `discovery_schedule()` 函数

```c
agent_timeout_add_with_context (agent, &agent->discovery_timer_source,
    "Candidate discovery tick", agent->timer_ta,
    priv_discovery_tick_agent_locked, NULL);
```

**超时间隔**：`agent->timer_ta`，默认 20ms。

**回调函数**：`priv_discovery_tick_agent_locked()`（discovery.c:1439），内部调用 `priv_discovery_tick_unlocked()`

**作用**：驱动候选收集的异步步骤：
- STUN Binding Request/Response 的发送和接收
- TURN Allocate Request 的发送和响应处理
- TURN CreatePermission / ChannelBind 的发送

**定时器停止条件**：`discovery.c:1425-1437`
- 当所有 discovery 项完成（`not_done == 0`）时停止
- 调用 `discovery_free(agent)` 清理所有 discovery 项
- 调用 `agent_gathering_done(agent)` 检查是否可以发射 gathering-done 信号

**特殊设计**：`discovery_schedule()` 在设置定时器之前，先**立即执行一次** `priv_discovery_tick_unlocked(agent)`。如果该次调用已经全部完成（`res == FALSE`），则不创建定时器。

#### 2.2.4 Keepalive 定时器

**创建位置**：`conncheck.c:1735-1739` -- `priv_schedule_next()` 函数

```c
agent_timeout_add_with_context (agent, &agent->keepalive_timer_source,
    "Connectivity keepalive timeout", agent->timer_ta,
    priv_conn_keepalive_tick_agent_locked, NULL);
```

**初始超时间隔**：`agent->timer_ta`，默认 20ms（与 conncheck 定时器同时启动）。

**回调函数**：`priv_conn_keepalive_tick_agent_locked()`（conncheck.c:1585），内部调用 `priv_conn_keepalive_tick_unlocked()`（conncheck.c:1360）

**Keepalive 间隔计算**（conncheck.c:1370-1374）：
- 启用 consent freshness（RFC 7675）：`min_next_tick = now + 1000 * NICE_AGENT_TIMER_MIN_CONSENT_INTERVAL`（4000ms）
- 未启用 consent freshness：`min_next_tick = now + 1000 * NICE_AGENT_TIMER_TR_DEFAULT`（25000ms = 25 秒）

每次 tick 后，根据所有候选对的 keepalive 调度重新计算下一次 tick 时间，动态调整定时器间隔。

**Keepalive 内容**：
- **普通 keepalive**：对每个 `selected_pair.local` 非 null 的 component 发送 STUN Binding Indication（非 TCP 候选或明确启用 conncheck keepalive 时）
- **Conncheck keepalive**（`NICE_AGENT_DO_KEEPALIVE_CONNCHECKS`）：使用完整的 STUN Binding Request/Response 事务，帮助检测对端可达性并刷新 NAT 绑定
- 发送时机受 ICE 规范 Section 11 约束：仅当上一次发送或接收数据后超过 Tr（25 秒）且无其他数据包在此对上发送时

**定时器停止条件**（conncheck.c:1569-1582）：
- 所有 stream 的 component 都没有 selected_pair 且没有 consent freshness 时，`errors > 0` 导致返回 FALSE，定时器停止
- 定时器在 `agent_unlock_and_emit()` 信号发射前被销毁的条件：`agent->streams` 为空时调用 `priv_remove_keepalive_timer()`

#### 2.2.5 定时器调度策略总结

| 定时器 | 默认间隔 | 回调函数 | 启动时机 | 停止时机 |
|--------|---------|---------|---------|---------|
| **Conncheck** | 20ms (Ta) | `priv_conn_check_tick_agent_locked` | 远端候选设置后（`priv_schedule_next`） | 所有检查完成 + idle_timeout 空闲后 |
| **Discovery** | 20ms (Ta) | `priv_discovery_tick_agent_locked` | 候选收集开始时（`discovery_schedule`） | 所有发现任务完成时 |
| **Keepalive** | 动态（20ms初始，后按25s/4s调整）| `priv_conn_keepalive_tick_agent_locked` | 远端候选设置后（`priv_schedule_next`）| 所有 stream 无选中对时 |

**调度特点**：
- 所有定时器共享同一个 `GMainContext`，由 GLib 主循环统一调度
- 每个定时器在创建新 source 前先销毁旧 source，保证同类定时器只有一个实例
- 定时器回调在 **agent 锁保护下** 执行，确保与公共 API 的线程安全
- Conncheck 和 Keepalive 同时启动（都在 `priv_schedule_next()` 中），Discovery 独立启动
- Conncheck 每次 tick 最多发送一个 STUN 请求（pacing），StunTimer 为每个事务独立管理超时和重传
- 定时器使用 `g_timeout_source_new()` 创建，精度为毫秒级

---

### 2.3 debug.c / debug.h（共 302 行）

#### 2.3.1 nice_debug_init()

**原型**：`agent/agent-priv.h:336`，实现在 `agent/debug.c:84`

```c
void nice_debug_init (void);
```

**作用**：首次调用时初始化 libnice 的调试系统。使用幂等性保护（`static gboolean debug_initialized = FALSE`），确保只执行一次。

**关键逻辑**：
1. 读取 `NICE_DEBUG` 环境变量，解析为标志位（使用 `g_parse_debug_string()` + `keys[]` 表）
2. 读取 `G_MESSAGES_DEBUG` 环境变量，解析为附加标志位（使用 `gkeys[]` 表）。同时检测 `libnice-pseudotcp-verbose` 和 `libnice-verbose` 子字符串
3. 设置 STUN 层的调试处理器：`stun_set_debug_handler(stun_handler)`，将 STUN 层调试输出通过 `g_logv("libnice-stun", ...)` 输出
4. 设置全局 `debug_enabled` 变量（`libnice` 域）
5. 根据标志位启用/禁用 STUN 调试（`stun_debug_enable()` / `stun_debug_disable()`）
6. 设置 PseudoTCP 调试级别：
   - `pseudotcp-verbose` -> `PSEUDO_TCP_DEBUG_VERBOSE`
   - `pseudotcp` -> `PSEUDO_TCP_DEBUG_NORMAL`
   - 注意：`pseudotcp` 和 `pseudotcp-verbose` 互斥，后者覆盖前者

**调试标志位定义**（`keys[]` 和 `gkeys[]`）：

| NICE_DEBUG 标志 | G_MESSAGES_DEBUG 域 | 内部标志 |
|----------------|-------------------|---------|
| `stun` | `libnice-stun` | `NICE_DEBUG_STUN` (1) |
| `nice` | `libnice` | `NICE_DEBUG_NICE` (2) |
| `pseudotcp` | `libnice-pseudotcp` | `NICE_DEBUG_PSEUDOTCP` (4) |
| `pseudotcp-verbose` | `libnice-pseudotcp-verbose` | `NICE_DEBUG_PSEUDOTCP_VERBOSE` (8) |
| `nice-verbose` | `libnice-verbose` | `NICE_DEBUG_NICE_VERBOSE` (16) |

可用 `all` 启用所有域。

#### 2.3.2 nice_debug() / nice_debug_verbose()

**Release 构建**（`NDEBUG` 定义时，agent-priv.h:340-343）：
```c
static inline gboolean nice_debug_is_enabled (void) { return FALSE; }
static inline gboolean nice_debug_is_verbose (void) { return FALSE; }
static inline void nice_debug (const char *fmt, ...) { }
static inline void nice_debug_verbose (const char *fmt, ...) { }
```
所有调试调用被编译器优化为**零开销**（inline 空函数或常量 FALSE）。

**Debug 构建**（debug.c:156-173）：
```c
void nice_debug (const char *fmt, ...) {
  va_list ap;
  if (debug_enabled) {
    va_start (ap, fmt);
    g_logv (G_LOG_DOMAIN, G_LOG_LEVEL_DEBUG, fmt, ap);
    va_end (ap);
  }
}
```
- `nice_debug()`：检查 `debug_enabled` 后通过 GLib 日志系统输出
- `nice_debug_verbose()`：额外检查 `debug_verbose_enabled`
- 日志域 `G_LOG_DOMAIN` 由各模块定义（`"libnice"`, `"libnice-socket"`, `"libnice-stun"` 等）

#### 2.3.3 nice_debug_enable() / nice_debug_disable()

```c
void nice_debug_enable (gboolean with_stun);
void nice_debug_disable (gboolean with_stun);
```

**作用**：程序运行时动态启用/禁用调试输出。

**关键逻辑**：
- 内部先调用 `nice_debug_init()`（确保已初始化）
- 设置 `debug_enabled = 1/0`
- 可选同时启用/禁用 STUN 层调试

#### 2.3.4 nice_debug_is_enabled() / nice_debug_is_verbose()

```c
gboolean nice_debug_is_enabled (void);
gboolean nice_debug_is_verbose (void);
```

**作用**：查询调试是否启用。在 agent.c 的 `agent_signal_new_selected_pair()` 等地用于条件性地输出详细日志。

#### 2.3.5 NICE_DEBUG 调试域总结

| 域 | 覆盖范围 |
|---|---------|
| `nice` / `libnice` | agent 层核心日志（状态变更、候选操作、API 调用） |
| `stun` / `libnice-stun` | STUN 消息的编解码、事务处理 |
| `pseudotcp` / `libnice-pseudotcp` | PseudoTCP 层正常调试 |
| `pseudotcp-verbose` / `libnice-pseudotcp-verbose` | PseudoTCP 层详细调试（包头、重传等） |
| `nice-verbose` / `libnice-verbose` | agent 层详细调试（额外的 verbose 信息） |

---

## 第三部分：Component 组件状态管理

### 3.1 Component 结构体（component.h）

`NiceComponent` 是 GObject 类型，代表 ICE 流中的单个媒体组件（如 RTP 或 RTCP）。它在内部协调候选管理、socket I/O、状态转换等职责。每个 component 属于一个 `NiceStream`，通过 `GWeakRef` 反向引用其所属的 `NiceAgent`。

#### 3.1.1 核心标识与状态字段

```c
struct _NiceComponent {
  GObject parent;                  // GObject 父类

  NiceComponentType type;         // RTP(1) 或 RTCP(2)
  guint id;                       // component id（正整数）
  NiceComponentState state;       // 当前状态（DISCONNECTED/GATHERING/CONNECTING/CONNECTED/READY/FAILED）
  guint stream_id;                // 所属 stream 的 ID
  GWeakRef agent_ref;             // 所属 agent 的弱引用（避免循环引用）

  gboolean fallback_mode;         // 回退模式：接受任意来源数据包，跳过候选验证
  gboolean have_local_consent;    // 本地是否同意接收（RFC 7675 consent freshness）
};
```

**设计要点**：
- `agent_ref` 使用 `GWeakRef` 而非直接引用，避免 `NiceAgent` 与 `NiceComponent`/`NiceStream` 之间形成引用环。获取 agent 指针时使用 `g_weak_ref_get()`。
- `fallback_mode` 在 `nice_component_set_selected_remote_candidate()` 被调用时设为 TRUE，用于非 ICE 兼容的对端连接。

#### 3.1.2 候选管理字段

```c
  GSList *local_candidates;       // 本地候选列表（NiceCandidateImpl 链表）
  GSList *remote_candidates;      // 远端候选列表（NiceCandidateImpl 链表）
  GList *valid_candidates;        // 经过验证的远端候选列表（仅包含有效对中的远端候选）
  CandidatePair selected_pair;    // 当前选中的候选对（独立于 conncheck 候选对列表）
  NiceCandidate *restart_candidate; // ICE 重启时保留的活动远端候选
  NiceCandidateImpl *turn_candidate; // TURN 服务器清除后保留的当前选中 TURN 候选
  GList *turn_servers;            // TURN 服务器配置列表（TurnServer 对象）
```

**设计要点**：
- `local_candidates` 和 `remote_candidates` 使用 `GSList`（单向链表），`valid_candidates` 使用 `GList`（双向链表，用于更高效的查找和重排）。
- `selected_pair` 与 conncheck 的检查列表**相互独立**，遵循 ICE 规范 11.1 节"Sending Media"。一旦选定候选对，无论 conncheck 状态如何，数据都通过该对发送。
- `valid_candidates` 最大长度为 `NICE_COMPONENT_MAX_VALID_CANDIDATES`，超出时删除最旧条目。此列表用于 `nice_component_verify_remote_candidate()` 快速验证入站数据包的来源是否合法。

#### 3.1.3 Socket 管理字段

```c
  GSList *socket_sources;         // SocketSource 对象链表（只增不减）
  guint socket_sources_age;       // socket_sources 变化计数器（用于 ComponentSource 检测变化）
```

**SocketSource 结构体**（component.h:118-123）：
```c
typedef struct {
  NiceSocket *socket;       // 底层 NiceSocket（非空）
  GSource *source;          // GLib GSource（可能为 NULL，表示未附加）
  NiceComponent *component; // 反向引用所属 Component
} SocketSource;
```

**设计要点**：
- 每个 socket 创建一个 `GSource`（通过 `g_socket_create_source()`），附加到 `component->ctx`。GSource 的回调统一为 `component_io_cb()`。
- `socket_sources` 链表**只增不减**（monotonically growing）：socket 只被添加，从不从列表中移除（即使 source 被 detach，SocketSource 结构仍保留）。
- `socket_sources_age` 作为版本号：每当 socket 被添加或移除时递增。`ComponentSource`（GIO 集成用）通过比对 age 判断是否需要更新子 source 列表。
- **UDP TURN socket 特殊处理**：`NICE_SOCKET_TYPE_UDP_TURN` 类型的 socket 不创建 GSource，因为数据包已经通过基础 UDP socket 接收，创建额外 source 会导致重复接收。

#### 3.1.4 I/O 回调与消息队列

```c
  GMutex io_mutex;                      // I/O 互斥锁（保护 io_callback 等字段）
  NiceAgentRecvFuncEx io_callback;      // 用户 I/O 回调函数（回调模式）
  gpointer io_user_data;                // 回调用户数据
  GDestroyNotify io_user_data_notify;   // 用户数据释放通知
  GQueue pending_io_messages;           // 已接收但尚未传递给客户端的数据队列
  guint io_callback_id;                 // I/O 回调的 GSource ID（0 表示未调度）

  GMainContext *own_ctx;                // component 自己的私有 GMainContext
  GMainContext *ctx;                    // 当前使用的 GMainContext（可能被应用覆盖）
  NiceInputMessage *recv_messages;      // 阻塞接收模式的接收缓冲区（不拥有）
  guint n_recv_messages;                // recv_messages 长度
  NiceInputMessageIter recv_messages_iter; // recv_messages 的当前写入位置
  GError **recv_buf_error;              // 接收错误信息
```

**设计要点**：
- `io_callback`（回调模式）与 `recv_messages`（阻塞模式）**互斥**：同一时刻只能启用一种模式。`nice_component_set_io_callback()` 负责切换。
- `io_mutex` 保护 I/O 回调相关字段。调用顺序规则：如果要同时获取 agent 锁和 io_mutex，**必须先获取 agent 锁**。
- `own_ctx` 是每个 component 构造时创建的私有上下文；`ctx` 可能被 `nice_agent_attach_recv()` 传入的应用程序上下文覆盖。如果 `ctx` 被设为 NULL，则自动回退到 `own_ctx`。
- `pending_io_messages` 队列用于**慢路径**：当 `component_io_cb()` 所在线程不拥有 `component->ctx` 时，数据先拷贝到队列，然后通过 idle GSource 在上下文中分发。

#### 3.1.5 PseudoTCP 与可靠传输字段

```c
  PseudoTcpSocket *tcp;                // PseudoTCP socket（可靠传输模式）
  GSource* tcp_clock;                  // PseudoTCP 时钟定时器 GSource
  guint64 last_clock_timeout;          // 上次时钟超时时间（微秒）
  gboolean tcp_readable;               // PseudoTCP socket 是否有可读数据
  GCancellable *tcp_writable_cancellable; // 取消等待 PseudoTCP 可写
  GIOStream *iostream;                 // GIO I/O 流包装（GInputStream/GOutputStream）
  GQueue queued_tcp_packets;           // 等待 selected socket 就绪的 TCP 数据包队列
```

**设计要点**：
- `queued_tcp_packets`：可靠模式下，如果接收到的 TCP 数据包到达时 selected socket 尚未就绪（无法发送 ACK），则先排队。一旦 `selected_pair` 被设置，`process_queued_tcp_packets()` 将队列中的数据送入 PseudoTCP socket 处理。

#### 3.1.6 辅助字段

```c
  StunAgent stun_agent;               // STUN 代理（用于验证入站 STUN 请求）
  GQueue incoming_checks;             // 入站连接检查队列（IncomingCheck 对象）

  guint8 *recv_buffer;                // 接收缓冲区（构造函数中预分配，避免热路径分配）
  guint recv_buffer_size;             // 接收缓冲区大小（65535 字节）

  guint min_port;                     // 本地候选端口范围下限
  guint max_port;                     // 本地候选端口范围上限

  guint8 *rfc4571_buffer;             // RFC 4571 帧重组缓冲区
  guint rfc4571_buffer_offset;        // 帧缓冲区的当前写位置
  guint rfc4571_buffer_size;          // 帧缓冲区大小
  guint rfc4571_frame_offset;         // 当前帧在缓冲区内的起始偏移
  guint rfc4571_frame_size;           // 当前帧的预期大小
  guint rfc4571_consumed_size;        // 当前帧已消耗的大小（用于 GIO 流读取）
  NiceAddress rfc4571_remote_addr;    // 当前帧的远端地址
  gboolean rfc4571_wakeup_needed;     // 是否需要唤醒 ComponentSource

  GCancellable *stop_cancellable;             // 取消令牌（用于唤醒主循环）
  GSource *stop_cancellable_source;           // 取消令牌关联的 GSource
  GCancellable *turn_resolving_cancellable;   // 取消 TURN 服务器 DNS 解析
};
```

**设计要点**：
- `recv_buffer` 在 `nice_component_init()` 中一次性分配 65535 字节（UDP 最大载荷大小），避免在 `component_io_cb()` 热路径中每次分配内存。
- `rfc4571_buffer` 用于 TCP 候选的 RFC 4571 成帧协议（2 字节长度前缀 + 载荷）。TCP 流式数据可能跨多个 `recv()` 调用分段到达，需要缓冲区重组。
- `incoming_checks` 存储对端发来的 STUN Binding Request 触发检查的信息，用于 triggered check 处理。
- `stun_agent` 在 `nice_component_constructed()` 中通过 `nice_agent_init_stun_agent()` 初始化，用于验证入站 STUN 消息的完整性（基于 stream 凭据）。

#### 3.1.7 内部数据结构

**CandidatePair**（component.h:87-95）：
```c
struct _CandidatePair {
  NiceCandidateImpl *local;
  NiceCandidateImpl *remote;
  guint64 priority;                    // 候选对优先级
  guint32 stun_priority;
  CandidatePairKeepalive keepalive;    // keepalive 定时信息
  CandidatePairConsentCheck remote_consent; // RFC 7675 consent 状态
};
```

**CandidatePairKeepalive**（component.h:71-77）：
```c
struct _CandidatePairKeepalive {
  guint64 next_tick;    // 下一次 keepalive tick 的时间戳
  guint stream_id;
  guint component_id;
  StunTimer timer;      // STUN 重传定时器
};
```

**CandidatePairConsentCheck**（component.h:79-85）：
```c
struct _CandidatePairConsentCheck {
  GSource *tick_source;                // consent 刷新定时器 GSource
  gboolean have;                       // 是否持有有效的远端 consent
  guint64 last_received;               // 最近一次收到远端 consent 的时间
};
```

**IncomingCheck**（component.h:97-105）：
```c
struct _IncomingCheck {
  NiceAddress from;            // 来源地址
  NiceSocket *local_socket;    // 接收该检查的本地 socket
  guint32 priority;            // 检查优先级
  gboolean use_candidate;      // 是否包含 USE-CANDIDATE 属性
  uint8_t *username;           // STUN username
  uint16_t username_len;
};
```

---

### 3.2 Component 生命周期

#### 3.2.1 nice_component_new()

**原型**：component.c:157
```c
NiceComponent *nice_component_new (guint id, NiceAgent *agent, NiceStream *stream);
```

**作用**：创建新的 Component 实例（GObject 构造封装）。

**关键逻辑**：
- 通过 `g_object_new(NICE_TYPE_COMPONENT, "id", id, "agent", agent, "stream", stream, NULL)` 创建对象
- `id`、`agent`、`stream` 均为 `G_PARAM_CONSTRUCT_ONLY` 属性，仅在构造时设置

#### 3.2.2 nice_component_init()

**原型**：component.c:1165

**作用**：GObject 实例初始化函数，在 `g_object_new()` 内部自动调用。

**关键逻辑**：
1. **原子计数器**：递增全局的 `n_components_created`（用于调试/泄漏检测）
2. **初始化状态**：`state = NICE_COMPONENT_STATE_DISCONNECTED`
3. **初始化互斥锁**：`g_mutex_init(&component->io_mutex)` 和 `g_queue_init(&component->pending_io_messages)`
4. **创建私有 GMainContext**：`component->own_ctx = g_main_context_new()`
5. **创建取消令牌及 source**：`stop_cancellable` + `stop_cancellable_source`，附加到 `own_ctx`。这使得即使 I/O 回调已 detach，也可以通过取消令牌唤醒主循环来关闭 component
6. **初始化 GMainContext**：`component->ctx = g_main_context_ref(component->own_ctx)`
7. **初始化 I/O 状态**：调用 `nice_component_set_io_context()` 和 `nice_component_set_io_callback()` 将 I/O 置于暂停状态
8. **初始化 PseudoTCP 包队列**：`g_queue_init(&component->queued_tcp_packets)`
9. **初始化入站检查队列**：`g_queue_init(&component->incoming_checks)`
10. **设置本地 consent**：`component->have_local_consent = TRUE`
11. **预分配接收缓冲区**：`recv_buffer` 分配 65535 字节（UDP 最大载荷），`rfc4571_buffer` 分配 `sizeof(guint16) + G_MAXUINT16` 字节
12. **创建 TURN 解析取消令牌**：`turn_resolving_cancellable`

#### 3.2.3 nice_component_constructed()

**原型**：component.c:1214

**作用**：GObject 构造链的 constructed 阶段，在所有构造属性设置完成后调用。

**关键逻辑**：
- 通过 `g_weak_ref_get(&component->agent_ref)` 获取 agent 指针
- 调用 `nice_agent_init_stun_agent(agent, &component->stun_agent)` 初始化 STUN 代理
- 释放 agent 引用，调用父类 constructed

**StunAgent 初始化要点**：`component->stun_agent` 的初始化依赖 agent 的兼容模式和 stream 的凭据（ufrag/password），用于后续验证对端发送的 STUN 消息。

#### 3.2.4 nice_component_close()

**原型**：component.c:343
```c
void nice_component_close (NiceAgent *agent, NiceStream *stream, NiceComponent *cmp);
```

**作用**：关闭组件，释放所有候选、socket 和 I/O 资源。**必须在持有 agent 锁的情况下调用**。

**关键逻辑**（按顺序）：
1. **关闭 PseudoTCP socket**：如果 `cmp->tcp` 存在，调用 `pseudo_tcp_socket_close(cmp->tcp, TRUE)`。注意：存在已知的竞争条件 -- PseudoTCP 关闭仅发送第一个 FIN 包，底层 socket 立即被关闭截断握手过程
2. **释放 restart_candidate**：`nice_candidate_free(cmp->restart_candidate)`
3. **释放 turn_candidate**：`nice_candidate_free((NiceCandidate *) cmp->turn_candidate)`
4. **移除所有本地候选**：遍历 `local_candidates`，对每个调用 `agent_remove_local_candidate()` 后释放
5. **释放所有远端候选**：`g_slist_free_full(cmp->remote_candidates, nice_candidate_free)`
6. **释放所有 socket sources**：`nice_component_free_socket_sources(cmp)` -- 销毁所有 GSource 并释放 NiceSocket
7. **清空入站检查队列**：逐一 `incoming_check_free()`
8. **清理 TURN 服务器**：`nice_component_clean_turn_servers(agent, cmp)`
9. **销毁 PseudoTCP 时钟和可写取消令牌**
10. **清空待发送 I/O 消息队列**：逐一 `io_callback_data_free()`
11. **重置 I/O 回调**：`nice_component_set_io_callback(cmp, NULL, ...)` -- 将 I/O 置于暂停状态
12. **取消 stop_cancellable**
13. **清空排队 TCP 包队列**：释放 `GOutputVector` 和缓冲区
14. **释放接收缓冲区**：`recv_buffer` 和 `rfc4571_buffer`
15. **复制清理 exdata**

#### 3.2.5 nice_component_finalize()

**原型**：component.c:1304

**作用**：GObject 析构函数，在对象引用计数归零时调用。

**关键逻辑**：
1. 断言 `socket_sources == NULL`、`local_candidates == NULL`、`remote_candidates == NULL`（close 应已清理）
2. 释放 `valid_candidates` 列表
3. 取消并释放 `turn_resolving_cancellable`
4. 释放 `tcp`、`stop_cancellable`、`iostream`
5. 清理 `io_mutex`
6. 销毁并释放 `stop_cancellable_source`
7. 释放 `ctx` 和 `own_ctx`
8. 清理 `agent_ref` 弱引用
9. 原子递增 `n_components_destroyed`

---

### 3.3 状态管理

#### 3.3.1 NiceComponentState 枚举

已在第一部分（1.2.3 节）定义。状态机转换路径为：

```
DISCONNECTED --> GATHERING --> CONNECTING --> CONNECTED --> READY
    |               |              |              |            |
    |               v              v              |            |
    +-----------> FAILED <------------------------+            |
    ^                                                          |
    |       (ICE restart 可回到 GATHERING)                       |
    +----------------------------------------------------------+
```

#### 3.3.2 agent_signal_component_state_change()

**原型**：agent.c:2678
```c
void agent_signal_component_state_change (NiceAgent *agent, guint stream_id,
    guint component_id, NiceComponentState new_state);
```

**作用**：执行组件状态转换、验证合法性和发射状态变更信号。

**关键逻辑**：

**1. 状态相同则直接返回**：`if (new_state == old_state) return;`

**2. 状态转换合法性验证**（通过 `TRANSITION(OLD, NEW)` 宏 + `g_assert()`）：

合法转换列表完整覆盖以下场景：

| 旧状态 | 新状态 | 触发场景 |
|--------|--------|---------|
| DISCONNECTED | GATHERING | `gather_candidates()` 开始 |
| DISCONNECTED | CONNECTING | 直接 `set_remote_candidates()` 而不 gather（提前知道候选） |
| GATHERING | CONNECTING | `set_remote_candidates()` 在 gather 后触发 conncheck |
| CONNECTING | CONNECTED | 至少一对候选通过连接检查 |
| CONNECTED | READY | 提名完成，最终确定候选对 |
| READY | CONNECTED | `priv_conn_check_add_for_candidate_pair_matched()` -- 新候选对匹配 |
| CONNECTED | CONNECTING | TCP socket 断开（`conn_check_prune_socket()`） |
| FAILED | CONNECTING | 收到新远端候选（`set_remote_candidates()`） |
| FAILED | GATHERING | 添加新的 relay 服务器 |
| **任意状态** | **FAILED** | 连接检查全部失败（包括 DISCONNECTED->FAILED 的跨组件失败） |
| **任意状态** | **GATHERING** | ICE restart（`nice_stream_restart()`） |

**3. 写入新状态**：`component->state = new_state;`

**4. 可靠模式 TCP 数据包处理**：`process_queued_tcp_packets(agent, stream, component)` -- 如果状态升级（连接就绪），将排队等待的 TCP 数据包送入 PseudoTCP socket 处理

**5. 排队发射信号**：`agent_queue_signal(agent, signals[SIGNAL_COMPONENT_STATE_CHANGED], stream_id, component_id, new_state)` -- 信号不在 agent 锁内直接发射

**状态变更的调用方汇总**：

| 调用方 | 文件 | 典型场景 |
|--------|------|---------|
| `nice_agent_gather_candidates()` | agent.c:3972 | DISCONNECTED/FAILED -> GATHERING |
| `priv_conn_check_tick_agent_locked()` | conncheck.c | CONNECTING -> CONNECTED, CONNECTED -> READY, CONNECTING/READY -> FAILED |
| `conn_check_prune_socket()` | conncheck.c | CONNECTED -> CONNECTING（TCP socket 断开） |
| `priv_conn_check_add_for_candidate_pair_matched()` | conncheck.c | READY -> CONNECTED（新候选对匹配） |
| `priv_add_remote_candidate()` | agent.c | FAILED -> CONNECTING |
| `priv_map_upnp()` | discovery.c | FAILED -> GATHERING |
| `nice_stream_restart()` | stream.c | 任意状态 -> GATHERING |
| `component_io_cb()` | agent.c:6327 | READY -> FAILED（selected pair socket HUP） |

---

### 3.4 Selected Pair 管理

#### 3.4.1 nice_component_update_selected_pair()

**原型**：component.c:542
```c
void nice_component_update_selected_pair (NiceAgent *agent,
    NiceComponent *component, const CandidatePair *pair);
```

**作用**：将 component 的 selected pair 更新为指定候选对，**不发射信号**（信号由调用方负责）。

**关键逻辑**：
1. 将传入 pair 的优先级转换为字符串用于调试输出
2. **清理旧的 turn_candidate**：如果当前 `selected_pair.local` 是 `turn_candidate`（通过 `forget_relays()` 保留的 TURN 候选），则执行与 relay 候选裁剪相同的清理流程：`discovery_prune_socket()` -> `conn_check_prune_socket()` -> `refresh_prune_candidate_async()` -> 释放。这是为了确保旧 relay 分配被正确拆除
3. **清空当前 selected_pair**：`nice_component_clear_selected_pair(component)` -- 释放 consent tick source，清零结构体
4. **设置新值**：复制 `pair->local`、`pair->remote`、`pair->priority`、`pair->stun_priority`、`pair->remote_consent.have`
5. **将远端候选加入 valid_candidates 列表**：`nice_component_add_valid_candidate()` -- 用于后续入站数据包的来源验证

#### 3.4.2 nice_component_clear_selected_pair()

**原型**：component.c:329（static）

**作用**：清除当前选中的候选对，释放关联的 consent 定时器。

**关键逻辑**：
- 如果 `selected_pair.remote_consent.tick_source` 存在，先 `g_source_destroy()` 再 `g_source_unref()`
- 调用 `memset` 将整个 `CandidatePair` 结构体清零

#### 3.4.3 nice_component_find_pair()

**原型**：component.c:460
```c
gboolean nice_component_find_pair (NiceComponent *cmp, NiceAgent *agent,
    const gchar *lfoundation, const gchar *rfoundation, CandidatePair *pair);
```

**作用**：根据 foundation 字符串查找候选对。

**关键逻辑**：
1. 遍历 `local_candidates`，匹配 `foundation == lfoundation`，找到 local 候选
2. 遍历 `remote_candidates`，匹配 `foundation == rfoundation`，找到 remote 候选
3. 若两者都找到，计算候选对优先级（`agent_candidate_pair_priority()`），写入 `pair` 输出参数
4. 返回值：找到返回 TRUE，否则 FALSE

**注意**：该函数使用 `strncmp` 比较 foundation，最大比较 `NICE_CANDIDATE_MAX_FOUNDATION` 字符。该函数主要用于 API 兼容（旧版 `new-selected-pair` 信号仅传 foundation 字符串）。

#### 3.4.4 nice_component_set_selected_remote_candidate()

**原型**：component.c:612
```c
NiceCandidateImpl *nice_component_set_selected_remote_candidate (
    NiceComponent *component, NiceAgent *agent, NiceCandidate *candidate);
```

**作用**：强制设置指定的远端候选为选中对象，并自动选择最高优先级匹配的本地 host 候选。调用后进入 **fallback 模式**，跳过候选验证。

**关键逻辑**：
1. 遍历 `local_candidates`，筛选**同协议族**、**同传输类型**、**类型为 HOST** 的本地候选
2. 对符合条件的本地候选，计算与传入远端候选的优先级（`agent_candidate_pair_priority()`），选择优先级最高的
3. 在 `remote_candidates` 中查找匹配地址和传输的远端候选；如果不存在，则拷贝一份加入列表并发射 `"new-remote-candidate"` 信号
4. 清空当前 selected pair，设置新的 local/remote/priority
5. 设置 `fallback_mode = TRUE` -- 后续入站数据包不再验证来源，模拟 pre-ICE SIP 的预期行为（接受任何来源的数据包）

**返回值**：选中的本地候选（`NiceCandidateImpl *`），如果没找到合适的本地候选则返回 NULL。

#### 3.4.5 nice_component_find_remote_candidate()

**原型**：component.c:588
```c
NiceCandidate *nice_component_find_remote_candidate (NiceComponent *component,
    const NiceAddress *addr, NiceCandidateTransport transport);
```

**作用**：按地址和传输类型查找远端候选。

**关键逻辑**：遍历 `remote_candidates`，同时匹配 `nice_address_equal(&candidate->addr, addr)` 和 `candidate->transport == transport`。

---

### 3.5 Socket 管理

#### 3.5.1 nice_component_attach_socket()

**原型**：component.c:679
```c
void nice_component_attach_socket (NiceComponent *component, NiceSocket *nicesock);
```

**作用**：将 socket 接入 component，创建 GSource 并附加到 component 的 GMainContext。**接管 socket 的所有权**。

**前置条件**：`component->ctx != NULL`

**关键逻辑**：
1. 在 `component->socket_sources` 中查找是否已存在该 socket 的 SocketSource（通过 `_find_socket_source()` 比较 socket 指针）
2. 如果**已存在**：直接使用已有的 SocketSource
3. 如果**不存在**：分配新的 `SocketSource`，设置 `socket`、`component` 字段，**前置插入**到 `socket_sources` 链表头部。如有 fileno（非虚拟 socket），递增 `socket_sources_age`
4. 调用 `socket_source_attach()` 创建并附加 GSource

**socket_source_attach() 内部逻辑**（component.c:100）：
- 如果 socket 的 `fileno == NULL`（虚拟 socket），不创建 source
- 如果 socket 类型为 `NICE_SOCKET_TYPE_UDP_TURN`，**不创建 source**（数据已通过基础 UDP socket 接收，防止重复）
- 否则创建 `g_socket_create_source(socket->fileno, G_IO_IN, NULL)`
- 设置回调为 `component_io_cb()`，用户数据为 `SocketSource *`
- 将 source 附加到传入的 context

#### 3.5.2 nice_component_detach_socket()

**原型**：component.c:743（static）
```c
static void nice_component_detach_socket (NiceComponent *component,
    NiceSocket *nicesock, gboolean *socket_source_freed);
```

**作用**：从 component 中移除指定 socket 的 GSource，关闭并释放 socket 和 SocketSource。

**关键逻辑**：
1. **清理入站检查**：遍历 `incoming_checks` 队列，移除所有 `local_socket == nicesock` 的 IncomingCheck 项
2. **查找 SocketSource**：在 `socket_sources` 链表中找匹配项
3. **删除并释放**：从链表中删除，调用 `socket_source_free()` 销毁 GSource 并释放 socket，递增 `socket_sources_age`
4. 通过 `socket_source_freed` 输出参数通知调用方 socket 已被释放（避免重复释放）

**socket_source_free() 内部逻辑**（component.c:148）：
- `socket_source_detach()`：销毁 GSource 并释放引用
- `nice_socket_free(socket)`：释放底层 NiceSocket
- `g_slice_free(SocketSource, source)`：释放结构体

#### 3.5.3 nice_component_remove_socket()

**原型**：component.c:167
```c
void nice_component_remove_socket (NiceAgent *agent, NiceComponent *cmp,
    NiceSocket *nsocket);
```

**作用**：完整地从 component 中移除一个 socket，同时清理所有基于该 socket 的候选和发现/检查状态。

**关键逻辑**：
1. 清理发现层：`discovery_prune_socket(agent, nsocket)`、`refresh_prune_socket(agent, nsocket)`
2. 清理连接检查层：`conn_check_prune_socket(agent, stream, cmp, nsocket)`
3. **遍历本地候选**：对每个 `sockptr` 基于 `nsocket` 的本地候选：
   - 如果是 `selected_pair.local`，清除 selected pair
   - 清理 refresh 和子 socket 的 discovery/conncheck
   - 从 agent 移除候选、释放候选
4. **遍历远端候选**：对每个 `sockptr == nsocket` 的远端候选：
   - 如果是 `selected_pair.remote`，清除 selected pair
   - 清理 conncheck、释放候选
5. 调用 `nice_component_detach_socket(cmp, nsocket, NULL)` 移除 GSource

#### 3.5.4 nice_component_free_socket_sources()

**原型**：component.c:801
```c
void nice_component_free_socket_sources (NiceComponent *component);
```

**作用**：释放 component 的所有 socket sources（在 `nice_component_close()` 中调用）。

**关键逻辑**：
- `g_slist_free_full(component->socket_sources, socket_source_free)`：逐个调用 `socket_source_free()` 销毁 GSource 并释放 socket
- 清空 `socket_sources` 链表
- 递增 `socket_sources_age`
- 调用 `nice_component_clear_selected_pair()` 清除 selected pair

#### 3.5.5 nice_component_detach_all_sockets()

**原型**：component.c:788
```c
void nice_component_detach_all_sockets (NiceComponent *component);
```

**作用**：将 component 的所有 socket GSource 从当前 GMainContext 中移除，但**不释放** socket。用于 I/O 上下文切换时暂时 detach。

**关键逻辑**：遍历所有 `socket_sources`，对每个调用 `socket_source_detach()`，仅销毁 GSource 而不释放 socket。

---

### 3.6 I/O 回调系统

Component 的 I/O 系统设计为**两种模式互斥**：回调模式（`attach_recv`）和阻塞模式（`recv_messages`）。`component_io_cb()` 是所有 socket 数据的统一入口，它在内部区分 STUN 消息和用户数据。

#### 3.6.1 component_io_cb()

**原型**：agent.c:6275
```c
static gboolean component_io_cb (GSocket *gsocket, GIOCondition condition,
    gpointer user_data);
```

**作用**：GLib GSource 回调，处理来自 socket 的入站数据。这是所有 component I/O 的**统一入口**。

**关键逻辑**：
1. 从 `user_data`（`SocketSource *`）中提取 `component` 和 `agent`
2. **防御性检查**：验证 GSource 未被销毁、agent 引用有效、stream 存在
3. **HUP 处理**：如果 socket 返回 `G_IO_HUP` 且无 `G_IO_IN`：
   - 如果该 socket 是 selected pair 的 local socket 且状态为 READY，则**声明组件失败**：`agent_signal_component_state_change(agent, ..., NICE_COMPONENT_STATE_FAILED)`
   - 调用 `nice_component_remove_socket()` 移除该 socket
4. **数据读取**（后续逻辑在 `agent.c:6335+`）：
   - 调用底层 socket 的 `recv_messages()` 读取数据
   - **STUN 消息**：内部处理（conncheck / discovery），不传递给用户
   - **非 STUN 消息**：通过 `nice_component_emit_io_callback()` 或 recv_messages 缓冲区传递给用户

#### 3.6.2 nice_component_emit_io_callback()

**原型**：component.c:1004
```c
void nice_component_emit_io_callback (NiceAgent *agent,
    NiceComponent *component, gsize buf_len);
```

**作用**：将接收到的用户数据通过 I/O 回调传递给应用层。**必须在持有 agent 锁时调用**。

**关键逻辑（快路径 vs 慢路径）**：

**快路径**（`g_main_context_is_owner(component->ctx)` 为 TRUE）：
- 当前线程拥有 component 的 GMainContext
- **直接调用回调**：先 `agent_unlock_and_emit()` 释放锁并发射排队信号，然后直接调用 `io_callback(agent, stream_id, component_id, buf_len, component->recv_buffer, ...)`
- 回调完成后重新获取 agent 锁
- **无需数据拷贝**：直接传递 `component->recv_buffer` 指针

**慢路径**（当前线程不拥有 GMainContext）：
- 先将数据**拷贝**到 `IOCallbackData` 结构体（`io_callback_data_new()`）
- 推入 `pending_io_messages` 队列尾部
- 调用 `nice_component_schedule_io_callback()` 通过 idle GSource 在下一次主循环迭代中分发
- 输出警告：`**WARNING: SLOW PATH**`

**设计原因**：快慢路径分离确保了线程安全 -- 如果 `component_io_cb()` 在非主循环线程中被触发（例如由 `GSocket` 的条件变量唤醒），数据通过慢路径安全地传递到主循环线程。

#### 3.6.3 nice_component_schedule_io_callback()

**原型**：component.c:1063（static）
```c
static void nice_component_schedule_io_callback (NiceComponent *component);
```

**作用**：在 component 的 GMainContext 中调度一个 idle GSource，用于异步分发 I/O 回调。**必须在持有 io_mutex 时调用**。

**关键逻辑**：
1. 如果已经调度（`io_callback_id != 0`）或 `pending_io_messages` 为空，直接返回
2. 创建 `g_idle_source_new()` 的 idle source（优先级 `G_PRIORITY_DEFAULT`）
3. 设置回调为 `emit_io_callback_cb`
4. 附加到 `component->ctx`，保存 source ID 到 `io_callback_id`

#### 3.6.4 emit_io_callback_cb()

**原型**：component.c:931（static）
```c
static gboolean emit_io_callback_cb (gpointer user_data);
```

**作用**：idle GSource 回调，在 GMainContext 的迭代中消费 `pending_io_messages` 队列。

**关键逻辑**：
1. 获取 agent 引用（`g_weak_ref_get`）
2. 加锁 `io_mutex`
3. **循环消费队列**：
   - 检查 `io_callback` 和队列头部是否有数据
   - 释放 `io_mutex` 后调用用户回调 `io_callback(agent, stream_id, component_id, data->buf_len - data->offset, ...)`
   - 回调后检查 agent/component 是否在回调中被销毁（`agent_find_component()`）
   - 从队列弹出已处理的数据，释放
   - 重新获取 `io_mutex`
4. 队列为空或无回调时，清除 `io_callback_id = 0`
5. 返回 `G_SOURCE_REMOVE`（单次执行）

**安全性保护**：每次调用用户回调前释放 `io_mutex`，回调后检查 agent/component 是否仍存在，防止用户在回调中销毁对象导致的 use-after-free。

#### 3.6.5 nice_component_deschedule_io_callback()

**原型**：component.c:1086（static）
```c
static void nice_component_deschedule_io_callback (NiceComponent *component);
```

**作用**：取消已调度的 I/O 回调 idle source。**必须在持有 io_mutex 时调用**。

**关键逻辑**：
- 如果 `io_callback_id != 0`，调用 `g_source_remove()` 移除并清零
- 未发送的数据保留在 `pending_io_messages` 队列中，等待下次 attach 时重新调度

#### 3.6.6 nice_component_set_io_callback()

**原型**：component.c:854
```c
void nice_component_set_io_callback (NiceComponent *component,
    NiceAgentRecvFuncEx func, gpointer user_data, GDestroyNotify notify,
    NiceInputMessage *recv_messages, guint n_recv_messages, GError **error);
```

**作用**：设置/切换 component 的 I/O 模式（回调模式 vs 阻塞接收模式 vs 暂停接收）。

**关键逻辑**：
1. 断言 `func` 和 `recv_messages` 互斥（不能同时设置回调模式和阻塞模式）
2. 加锁 `io_mutex`
3. **释放旧回调的用户数据**：如果之前有 `io_user_data_notify`，调用它释放旧数据
4. **回调模式**（`func != NULL`）：
   - 设置 `io_callback`、`io_user_data`、`io_user_data_notify`
   - 清空 `recv_messages`（确保互斥）
   - 调用 `nice_component_schedule_io_callback()` -- 如果队列中已有等待数据，立即调度分发
5. **阻塞模式 / 暂停**（`func == NULL`）：
   - 设置 `recv_messages`、`n_recv_messages`（如果为 NULL 则暂停接收）
   - 清空 `io_callback`
   - 调用 `nice_component_deschedule_io_callback()` -- 取消 idle source
6. 重置 `recv_messages_iter`，设置 `recv_buf_error`
7. 释放 `io_mutex`

---

### 3.7 ICE Restart

#### 3.7.1 nice_component_restart()

**原型**：component.c:497
```c
void nice_component_restart (NiceComponent *cmp, NiceAgent *agent);
```

**作用**：将 component 重置为 ICE 重启状态，保留选中的远端候选。

**关键逻辑**：
1. **保留 selected pair 的远端候选**（符合 ICE 规范 9.1.1.1 "ICE Restarts" -- 不能移除活动远端候选）：
   - 遍历 `remote_candidates`，如果是 `selected_pair.remote`，则保存到 `restart_candidate`（先释放旧值）
   - 非 selected pair 的远端候选直接释放
2. **释放 remote_candidates 链表**（内存结构释放，但 selected pair 的远端候选指针保留在 `restart_candidate` 中）
3. **清空入站检查队列**：逐一 `incoming_check_free()`
4. **重置 selected_pair 优先级为 0**：确保 ICE 重新协商时会选择新的候选对
5. **重置本地 consent**：`have_local_consent = TRUE`
6. **重置 StunAgent**：`nice_agent_init_stun_agent(agent, &cmp->stun_agent)` -- 因为新的 stream 凭据会生效，旧的 `stun_agent` 可能包含对已释放远端候选密码的引用

**注意**：注释指出"component state managed by agent"，即重启后的状态变更（如转入 GATHERING）由调用方（`nice_stream_restart()`）通过 `agent_signal_component_state_change()` 负责。

---

### 3.8 GMainContext 管理

#### 3.8.1 nice_component_set_io_context()

**原型**：component.c:823
```c
void nice_component_set_io_context (NiceComponent *component, GMainContext *context);
```

**作用**：设置 component 的 I/O GMainContext。如果传入 NULL，则使用 component 自有的 `own_ctx`。

**关键逻辑**：
1. 加锁 `io_mutex`
2. 如果新 context 与当前 `ctx` 不同：
   - **解析默认值**：如果 `context == NULL`，则使用 `component->own_ctx`（增加引用）
   - **Detach 所有 socket**：`nice_component_detach_all_sockets()` -- 从旧 context 移除所有 GSource
   - **释放旧 context**：`g_main_context_unref(component->ctx)`
   - **设置新 context**：`component->ctx = context`
   - **Reattach 所有 socket**：`nice_component_reattach_all_sockets()` -- 将 GSource 重新附加到新 context
3. 释放 `io_mutex`

#### 3.8.2 nice_component_dup_io_context()

**原型**：component.c:814
```c
GMainContext *nice_component_dup_io_context (NiceComponent *component);
```

**作用**：获取 component 自有 `own_ctx` 的引用（用于 GIO 流构造）。

---

### 3.9 其他辅助函数

#### 3.9.1 nice_component_add_valid_candidate()

**原型**：component.c:1674
```c
void nice_component_add_valid_candidate (NiceAgent *agent,
    NiceComponent *component, const NiceCandidate *candidate);
```

**作用**：将一个远端候选加入已验证列表（`valid_candidates`），用于入站数据包的来源验证。

**关键逻辑**：
1. 遍历现有 `valid_candidates`，如果已存在相同地址的候选（`nice_candidate_equal_target()`），直接返回
2. 将新候选**前置插入**（`g_list_prepend`）到列表头部
3. 如果列表长度超过 `NICE_COMPONENT_MAX_VALID_CANDIDATES`，删除最后一个（最旧的）条目

#### 3.9.2 nice_component_verify_remote_candidate()

**原型**：component.c:1717
```c
gboolean nice_component_verify_remote_candidate (NiceComponent *component,
    const NiceAddress *address, NiceSocket *nicesock);
```

**作用**：验证入站数据包的来源地址是否属于已知有效的远端候选。用于防止从非协商地址接收数据。

**关键逻辑**：
1. 如果 `fallback_mode == TRUE`，跳过验证直接返回 TRUE
2. 遍历 `valid_candidates`，匹配地址和传输类型
3. **LRU 优化**：如果匹配项不是第一个，则将匹配项移到列表头部（`g_list_remove_link` + `g_list_concat`），使后续查找命中第一项时为 O(1)

#### 3.9.3 nice_component_prune_relay_candidate()

**原型**：component.c:259
```c
void nice_component_prune_relay_candidate (NiceAgent *agent,
    NiceComponent *cmp, NiceCandidateImpl *relay_cand);
```

**作用**：异步裁剪（释放）一个 relay 候选，包括其 socket 和关联的 TURN refresh。

**关键逻辑**：
1. 从发现和连接检查列表中删除该候选的 socket：`discovery_prune_socket()` + `conn_check_prune_socket()`
2. 异步释放 TURN 刷新：`refresh_prune_candidate_async()`
3. 回调 `on_candidate_refreshes_pruned()` 中释放 socket 和候选对象

#### 3.9.4 nice_component_clean_turn_servers()

**原型**：component.c:276
```c
void nice_component_clean_turn_servers (NiceAgent *agent, NiceComponent *cmp);
```

**作用**：清除所有 TURN 服务器配置，保留 selected pair 中的 TURN 候选。

**关键逻辑**：
1. 释放 `turn_servers` 列表中的所有 `TurnServer` 对象
2. 遍历 `local_candidates`，对 `RELAYED` 类型：
   - 如果候选是 `selected_pair.local`：保存到 `turn_candidate`（用于继续 active 连接的 TURN refresh），设置 `selected_pair.priority = 0` 以便 ICE 重启时替换
   - 否则：从 agent 移除候选，加入待清理列表
3. 对清理列表中的候选调用 `nice_component_prune_relay_candidate()`

#### 3.9.5 nice_component_shutdown()

**原型**：component.c:426
```c
void nice_component_shutdown (NiceComponent *component,
    gboolean shutdown_read, gboolean shutdown_write);
```

**作用**：优雅关闭（shutdown）component 的 PseudoTCP socket 和 TCP socket，支持半关闭。

**关键逻辑**：
- PseudoTCP socket：调用 `pseudo_tcp_socket_shutdown()` 按指定方向关闭
- TCP_BSD socket：调用 `g_socket_shutdown()` 按指定方向关闭底层的 GSocket

#### 3.9.6 nice_component_get_sockets()

**原型**：component.c:1757
```c
GPtrArray *nice_component_get_sockets (NiceComponent *component);
```

**作用**：获取 component 所有底层 GSocket（用于设置额外 socket 选项）。**必须在持有 agent 锁时调用**。

**关键逻辑**：遍历 `socket_sources`，提取 `nicesock->fileno`，去重后加入 `GPtrArray` 返回。

---

### 3.10 调用关系（第三部分）

```
component 生命周期
─────────────────────────────────────────────────────
nice_component_new() ──→ g_object_new() → nice_component_init()
                       → nice_component_constructed() → nice_agent_init_stun_agent()

nice_component_close() ──→ pseudo_tcp_socket_close()
                        → agent_remove_local_candidate() (for each)
                        → nice_component_free_socket_sources()
                        → nice_component_clean_turn_servers()
                        → nice_component_set_io_callback(NULL, ...)

socket 管理
─────────────────────────────────────────────────────
nice_component_attach_socket() ──→ socket_source_attach()
                                  → g_socket_create_source() + g_source_attach()

nice_component_detach_socket() ──→ socket_source_free()
                                  → socket_source_detach() + nice_socket_free()

nice_component_remove_socket() ──→ discovery_prune_socket()
                                → conn_check_prune_socket()
                                → agent_remove_local_candidate()
                                → nice_component_detach_socket()

I/O 回调系统
─────────────────────────────────────────────────────
component_io_cb() (GSource callback, agent.c)
  ├── HUP → agent_signal_component_state_change(FAILED)
  ├── STUN → conncheck / discovery 内部处理
  └── user data → nice_component_emit_io_callback()
                    ├── 快路径: 直接调用 io_callback()
                    └── 慢路径: io_callback_data_new() → pending_io_messages
                              → nice_component_schedule_io_callback()
                                → emit_io_callback_cb() [idle source]

状态管理
─────────────────────────────────────────────────────
agent_signal_component_state_change() (agent.c)
  ├── 验证状态转换合法性 (g_assert + TRANSITION 宏)
  ├── component->state = new_state
  ├── process_queued_tcp_packets() (可靠模式)
  └── agent_queue_signal(SIGNAL_COMPONENT_STATE_CHANGED)

selected pair 管理
─────────────────────────────────────────────────────
nice_component_update_selected_pair() ──→ nice_component_clear_selected_pair()
                                        → nice_component_add_valid_candidate()

nice_component_restart()
  ├── 保留 selected_pair.remote -> restart_candidate
  ├── 释放其他 remote_candidates
  ├── selected_pair.priority = 0
  └── nice_agent_init_stun_agent() (重置 STUN 代理)
```

---

### 3.11 Component 设计总结

1. **GObject 封装**：`NiceComponent` 是完整的 GObject 类型，支持构造属性（id/agent/stream，均为 CONSTRUCT_ONLY）和标准的 init/constructed/finalize 生命周期。

2. **线程安全分层**：
   - **agent 锁**保护候选列表、状态、selected pair 等核心数据
   - **io_mutex** 保护 I/O 回调注册和消息队列
   - 锁获取顺序：agent 锁在前，io_mutex 在后
   - 信号不在任何锁内发射（通过 `agent_unlock_and_emit()` 机制）

3. **两种 I/O 模式互斥**：回调模式（GLib GSource 驱动）和阻塞模式（直接 socket recv）不能在同一 component 上同时使用。切换通过 `nice_component_set_io_callback()` 完成。

4. **Socket 生命周期**：socket 通过 `attach` 接管所有权，通过 `remove` 级联清理候选、发现状态、连接检查状态。`socket_sources` 链表只增不减（monotonically growing）的设计简化了并发访问。

5. **快慢路径分离**：I/O 回调的分发区分当前线程是否拥有 GMainContext。快路径直接回调（零拷贝），慢路径通过数据拷贝 + idle GSource 保证线程安全。

6. **ICE 重启保留活动候选**：`restart_candidate` 和 `turn_candidate` 机制确保 ICE 重启时保留活动的远端/TURN 候选，符合 RFC 5245 第 9.1.1.1 节。

---

## 第四部分：Stream、Candidate、Address 与 Interfaces

### 4.1 Stream (stream.c/h, 340 lines)

#### 4.1.1 数据结构

`NiceStream` 是一个 GObject 类型，代表一个 ICE 媒体流（对应 SDP 中的 m-line）。其结构定义在 `stream.h:74-99`：

```c
struct _NiceStream {
  GObject parent;
  gchar *name;                          // 流名称（可由用户设置）
  guint id;                             // 流 ID（从 1 开始）
  guint n_components;                   // 组件数量（通常为 1，RTP+RTCP 为 2）
  gboolean initial_binding_request_received;  // 是否已收到初始 Binding Request
  GSList *components;                   // NiceComponent 对象链表
  GSList *conncheck_list;               // CandidateCheckPair 项链表
  gchar local_ufrag[NICE_STREAM_MAX_UFRAG];   // 本地 ICE ufrag（最大 256+1 字节）
  gchar local_password[NICE_STREAM_MAX_PWD];  // 本地 ICE 密码（最大 256+1 字节）
  gchar remote_ufrag[NICE_STREAM_MAX_UFRAG];  // 远端 ICE ufrag
  gchar remote_password[NICE_STREAM_MAX_PWD]; // 远端 ICE 密码
  gboolean gathering;                   // 是否正在收集候选
  gboolean gathering_started;           // 收集是否已启动
  gboolean peer_gathering_done;         // 远端是否已完成收集（trickle ICE 相关）
  gint tos;                             // IP TOS 值
  guint tick_counter;                   // 滴答计数器
  GSList *upnp_mapping;                 // UPnP 映射中候选
  GSList *upnp_mapped;                  // UPnP 已映射候选
  GSource *upnp_timer_source;           // UPnP 超时定时器
};
```

全局原子计数器 `n_streams_created` / `n_streams_destroyed` 用于追踪存活的 Stream 数量。

#### 4.1.2 凭证常量

凭证长度定义 (`stream.h:56-60`)：
- `NICE_STREAM_MAX_UFRAG` = 257（256 + NULL）
- `NICE_STREAM_MAX_PWD` = 257
- `NICE_STREAM_DEF_UFRAG` = 5（4 + NULL）-- 默认 ufrag 长度
- `NICE_STREAM_DEF_PWD` = 23（22 + NULL）-- 默认密码长度
- `NICE_STREAM_MAX_UNAME` = 514（2*256 + colon + NULL）-- 组合用户名最大长度

#### 4.1.3 关键函数

**`nice_stream_new(stream_id, n_components, agent)`**
- 原型：`NiceStream *nice_stream_new(guint stream_id, guint n_components, NiceAgent *agent)`
- 目的：创建新 Stream 并初始化其所有 Component
- 关键逻辑：
  1. 通过 `g_object_new(NICE_TYPE_STREAM, NULL)` 分配 Stream 对象
  2. 设置 `stream->id = stream_id`
  3. 循环创建 `n_components` 个 `NiceComponent` 对象（component ID 从 1 开始），追加到 `stream->components` 链表
  4. 如果未启用 trickle ICE（`!agent->use_ice_trickle`），设置 `peer_gathering_done = TRUE`
  5. 返回 Stream 指针

**`nice_stream_close(agent, stream)`**
- 原型：`void nice_stream_close(NiceAgent *agent, NiceStream *stream)`
- 目的：关闭 Stream 的所有 Component
- 关键逻辑：遍历 `stream->components`，对每个 Component 调用 `nice_component_close()`

**`nice_stream_find_component_by_id(stream, id)`**
- 原型：`NiceComponent *nice_stream_find_component_by_id(NiceStream *stream, guint id)`
- 目的：按 ID 查找 Stream 下的某个 Component
- 关键逻辑：线性遍历 `stream->components`，返回匹配 `component->id == id` 的第一个 Component

**`nice_stream_initialize_credentials(stream, rng)`**
- 原型：`void nice_stream_initialize_credentials(NiceStream *stream, NiceRNG *rng)`
- 目的：为 ICE 流生成本地 ufrag 和密码（RFC 5245 第 15.4 节）
- 关键逻辑：
  1. 调用 `nice_rng_generate_bytes_print()` 生成随机 ufrag 和密码（使用 RNG 加密随机源）
  2. 重置远端凭证（`remote_ufrag[0] = 0`，`remote_password[0] = 0`），因为无法假设在 conncheck 重新开始之前会从 SDP 收到新的远端凭证

**`nice_stream_restart(stream, agent)`**
- 原型：`void nice_stream_restart(NiceStream *stream, NiceAgent *agent)`
- 目的：将 Stream 重置为 ICE 重启状态（RFC 5245 第 9.1.1.1 节）
- 关键逻辑：
  1. 调用 `conn_check_prune_stream()` 清理所有连接检查
  2. 重置 `initial_binding_request_received = FALSE`
  3. 调用 `nice_stream_initialize_credentials()` 重新生成凭证
  4. 遍历每个 Component，调用 `nice_component_restart()` 并触发状态变更至 `NICE_COMPONENT_STATE_GATHERING`

**生命周期管理**：
- `nice_stream_init()`：GObject 实例初始化，自增 `n_streams_created`
- `nice_stream_finalize()`：释放 `name` 和 `components` 链表（同时 unref 每个 Component 对象），自增 `n_streams_destroyed`

### 4.2 Candidate (candidate.c/h/priv.h, 920 lines)

#### 4.2.1 NiceCandidate 结构

`NiceCandidate` 是一个 GBoxed 类型（通过 `G_DEFINE_BOXED_TYPE` 注册），代表 ICE 候选。结构定义在 `candidate.h:167-179`：

```c
struct _NiceCandidate {
  NiceCandidateType type;               // 候选类型
  NiceCandidateTransport transport;     // 传输协议
  NiceAddress addr;                     // 候选传输地址
  NiceAddress base_addr;                // 基础地址（srflx/relay 的原始地址）
  guint32 priority;                     // 优先级
  guint stream_id;                      // 所属 Stream ID
  guint component_id;                   // 所属 Component ID
  gchar foundation[NICE_CANDIDATE_MAX_FOUNDATION];  // 基础标识（最多 32+1 字符）
  gchar *username;                      // 候选特定用户名（可覆盖 Stream 级别）
  gchar *password;                      // 候选特定密码
};
```

#### 4.2.2 NiceCandidateImpl（内部结构）

定义在 `candidate-priv.h:118-125`，将 `NiceCandidate c` 作为第一个成员（继承模式）：

```c
struct _NiceCandidateImpl {
  NiceCandidate c;
  TurnServer *turn;                     // TURN 服务器配置（仅 relay 候选有效）
  NiceSocket *sockptr;                  // 底层 socket
  guint64 keepalive_next_tick;          // 下次 keepalive 时间戳
  NiceAddress *stun_server;            // STUN 服务器地址（仅 srflx 候选有效）
};
```

#### 4.2.3 TurnServer 结构

定义在 `candidate-priv.h:86-103`，存储 TURN 服务器配置：

```c
struct _TurnServer {
  gint ref_count;                       // 引用计数
  NiceAddress server;                   // TURN 服务器地址
  gchar *server_address;                // 未解析的服务器地址
  guint server_port;                    // 服务器端口
  gchar *username;                      // TURN 用户名
  gchar *password;                      // TURN 密码
  guint8 *decoded_username;             // Base64 解码后的用户名
  guint8 *decoded_password;             // Base64 解码后的密码
  gsize decoded_username_len;           // 解码用户名长度
  gsize decoded_password_len;           // 解码密码长度
  NiceRelayType type;                   // 中继类型（UDP/TCP/TLS）
  guint preference;                     // 唯一标识符，用于计算优先级
  gboolean resolution_failed;           // DNS 解析是否失败
};
```

TurnServer 使用引用计数（`turn_server_ref` / `turn_server_unref`）管理生命周期。

#### 4.2.4 枚举类型

**NiceCandidateType** — 候选类型：
- `NICE_CANDIDATE_TYPE_HOST` = 0 — 主机候选（本机 IP）
- `NICE_CANDIDATE_TYPE_SERVER_REFLEXIVE` = 1 — 服务器反射候选（通过 STUN 获取）
- `NICE_CANDIDATE_TYPE_PEER_REFLEXIVE` = 2 — 对端反射候选（连接检查中发现）
- `NICE_CANDIDATE_TYPE_RELAYED` = 3 — 中继候选（通过 TURN 获取）

**NiceCandidateTransport** — 传输协议：
- `NICE_CANDIDATE_TRANSPORT_UDP` = 0 — UDP
- `NICE_CANDIDATE_TRANSPORT_TCP_ACTIVE` = 1 — TCP 主动
- `NICE_CANDIDATE_TRANSPORT_TCP_PASSIVE` = 2 — TCP 被动
- `NICE_CANDIDATE_TRANSPORT_TCP_SO` = 3 — TCP 同时打开

**NiceRelayType** — 中继类型：
- `NICE_RELAY_TYPE_TURN_UDP` — TURN over UDP
- `NICE_RELAY_TYPE_TURN_TCP` — TURN over TCP
- `NICE_RELAY_TYPE_TURN_TLS` — TURN over TLS/TCP

#### 4.2.5 优先级计算

libnice 实现了多种优先级计算公式：

**类型优先级常量** (`candidate-priv.h:54-59`)：
```
NICE_CANDIDATE_TYPE_PREF_HOST             = 120 (最高)
NICE_CANDIDATE_TYPE_PREF_PEER_REFLEXIVE   = 110
NICE_CANDIDATE_TYPE_PREF_NAT_ASSISTED     = 105
NICE_CANDIDATE_TYPE_PREF_SERVER_REFLEXIVE = 100
NICE_CANDIDATE_TYPE_PREF_RELAYED_UDP      = 30
NICE_CANDIDATE_TYPE_PREF_RELAYED          = 20  (最低)
```

**ICE 优先级公式** (`nice_candidate_ice_priority_full()`) -- RFC 5245 推荐公式：
```
priority = (2^24 * type_preference) + (2^8 * local_preference) + (256 - component_id)
```
其中 `type_preference` 范围 [0, 126]，`local_preference` 范围 [0, 65535]，`component_id` 范围 [0, 255]。

**Local preference 编码** (`nice_candidate_ice_local_preference_full()`)：
```
bits 0-5:   other_preference（IP 在本地接口列表中的位置）
bits 6-8:   turn_preference（TURN 服务器序号，0-7）
bits 9-12:  未使用
bits 13-15: direction_preference（方向偏好，0-7）
```

**方向偏好值**（基于传输类型，由 `nice_candidate_ice_local_preference()` 计算）：
- UDP: `direction_preference = 1`
- TCP-ACTIVE (host/srflx): `direction_preference = 4`
- TCP-PASSIVE (host/srflx): `direction_preference = 2`
- TCP-SO (host/srflx): `direction_preference = 6`
- Relay 候选的 `turn_preference` 指向 TurnServer 的唯一 preference 值

**IP 本地偏好** (`nice_candidate_ip_local_preference()`)：通过候选地址在 `nice_interfaces_get_local_ips()` 返回列表中的位置计算，确保多宿主主机的不同 host 候选获得不同的优先级（RFC 5245 第 4.1.2.1 节要求）。

**MS-ICE 变体** (`nice_candidate_ms_ice_priority()`, `nice_candidate_ms_ice_local_preference()`)：为 MS-ICE 兼容模式实现不同的 preference 编码（transport_preference 占用 bits 12-15，direction_preference 占用 bits 9-11）。

**可靠性惩罚** (`nice_candidate_ice_type_preference()`)：如果代理启用可靠模式但候选是 UDP（或反之），类型偏好减半。

**Jingle 优先级** (`nice_candidate_jingle_priority()`)：Google Jingle 兼容的简单固定优先级（host=1000, srflx=900, relay=500）。

**MSN 优先级** (`nice_candidate_msn_priority()`)：MSN 兼容的固定优先级（host=830, srflx=550, relay=450）。

**候选对优先级** (`nice_candidate_pair_priority()`) -- RFC 5245 第 5.7.2 节：
```
pair_priority = (2^32 * MIN(G_D, G_D)) + 2 * MAX(G_D, G_D) + (G_D > G_D ? 1 : 0)
```
其中 G_D 为发起者候选优先级，G_D 为响应者候选优先级。

#### 4.2.6 关键函数

**`nice_candidate_new(type)`**
- 原型：`NiceCandidate *nice_candidate_new(NiceCandidateType type)`
- 目的：分配并初始化一个新的 NiceCandidateImpl（内部用 `g_slice_new0` 零分配），设置候选类型

**`nice_candidate_free(candidate)`**
- 原型：`void nice_candidate_free(NiceCandidate *candidate)`
- 目的：释放候选及相关资源
- 关键逻辑：释放 username/password 字符串，unref TurnServer，释放 stun_server，最终 `g_slice_free` 释放自身

**`nice_candidate_copy(candidate)`**
- 原型：`NiceCandidate *nice_candidate_copy(const NiceCandidate *candidate)`
- 目的：深拷贝候选
- 关键逻辑：分配新的 NiceCandidateImpl，`memcpy` 整体结构，然后深拷贝字符串成员（username、password、stun_server），注意不拷贝 turn 指针和 sockptr（重置为 NULL）

**`nice_candidate_equal_target(candidate1, candidate2)`**
- 原型：`gboolean nice_candidate_equal_target(const NiceCandidate *candidate1, const NiceCandidate *candidate2)`
- 目的：判断两个候选是否指向同一传输地址（比较 transport 和 addr，忽略其他字段）

**`nice_candidate_type_to_string(type)`**
- 返回静态字符串："host" / "srflx" / "prflx" / "relay"

**`nice_candidate_transport_to_string(transport)`**
- 返回静态字符串："udp" / "tcp-act" / "tcp-pass" / "tcp-so"

**`nice_candidate_relay_address(candidate, addr)`**
- 目的：获取 relay 候选的 TURN 服务器地址（`c->turn->server`）

**`nice_candidate_stun_server_address(candidate, addr)`**
- 目的：获取 srflx 候选的 STUN 服务器地址

### 4.3 Address (address.c/h, 786 lines)

#### 4.3.1 数据结构

`NiceAddress` 是一个 GBoxed 类型，通过 union 统一封装 IPv4 和 IPv6 地址。定义在 `address.h:77-85`：

```c
struct _NiceAddress {
  union {
    struct sockaddr     addr;   // 通用 socket 地址
    struct sockaddr_in  ip4;    // IPv4 地址 (AF_INET)
    struct sockaddr_in6 ip6;    // IPv6 地址 (AF_INET6)
  } s;
};
```

#### 4.3.2 地址族判断

NiceAddress 始终通过 `s.addr.sa_family` 字段判断当前存储的是 IPv4（`AF_INET`）还是 IPv6（`AF_INET6`）地址。初始化时设为 `AF_UNSPEC`（未指定）。

#### 4.3.3 关键函数

**`nice_address_init(addr)`**
- 原型：`void nice_address_init(NiceAddress *addr)`
- 目的：将地址初始化为未指定状态（`AF_UNSPEC`），清零整个结构

**`nice_address_new()`**
- 原型：`NiceAddress *nice_address_new(void)`
- 目的：分配并初始化一个新的 NiceAddress（`g_slice_new0`）

**`nice_address_free(addr)`**
- 原型：`void nice_address_free(NiceAddress *addr)`
- 目的：释放 NiceAddress（`g_slice_free`）

**`nice_address_dup(addr)`**
- 原型：`NiceAddress *nice_address_dup(const NiceAddress *addr)`
- 目的：复制地址（简单结构体赋值 `*dup = *a`）

**`nice_address_set_ipv4(addr, addr_ipv4)`**
- 原型：`void nice_address_set_ipv4(NiceAddress *addr, guint32 addr_ipv4)`
- 目的：设置为 IPv4 地址
- 关键逻辑：设置 `sin_family = AF_INET`，通过 `htonl` 转换主机字节序，端口重置为 0

**`nice_address_set_ipv6(addr, addr_ipv6)`**
- 原型：`void nice_address_set_ipv6(NiceAddress *addr, const guchar *addr_ipv6)`
- 目的：设置为 IPv6 地址
- 关键逻辑：设置 `sin6_family = AF_INET6`，复制 16 字节地址，端口和 scope_id 重置为 0

**`nice_address_set_port(addr, port)` / `nice_address_get_port(addr)`**
- 原型：`void nice_address_set_port(NiceAddress *addr, guint port)` / `guint nice_address_get_port(const NiceAddress *addr)`
- 目的：设置/获取端口号
- 关键逻辑：根据 `sa_family` 选择操作 `sin_port`（IPv4）或 `sin6_port`（IPv6），使用 `htons`/`ntohs` 转换网络字节序

**`nice_address_set_from_string(addr, str)`**
- 原型：`gboolean nice_address_set_from_string(NiceAddress *addr, const gchar *str)`
- 目的：从字符串（如 "192.168.1.1" 或 "::1"）设置地址
- 关键逻辑：
  1. 使用 `getaddrinfo()` 配合 `AI_NUMERICHOST` 标志（仅解析数字地址，不做 DNS 查询）
  2. 解析成功后调用 `nice_address_set_from_sockaddr()` 复制结果

**`nice_address_to_string(addr, dst)`**
- 原型：`void nice_address_to_string(const NiceAddress *addr, gchar *dst)`
- 目的：将地址转换为可读字符串
- 关键逻辑：根据地址族调用 `inet_ntop(AF_INET, ...)` 或 `inet_ntop(AF_INET6, ...)`

**`nice_address_dup_string(addr)`**
- 原型：`gchar *nice_address_dup_string(const NiceAddress *addr)`
- 目的：返回新分配的地址字符串

**`nice_address_equal(a, b)`**
- 原型：`gboolean nice_address_equal(const NiceAddress *a, const NiceAddress *b)`
- 目的：比较两个地址是否完全相同（包括端口）
- 关键逻辑：
  - 地址族不同则返回 FALSE
  - IPv4：比较 `sin_addr.s_addr` 和 `sin_port`
  - IPv6：使用 `IN6_ARE_ADDR_EQUAL` 宏比较 16 字节地址，同时比较 `sin6_port` 和 `sin6_scope_id`（仅在双方 scope_id 均非零时比较 scope_id）

**`nice_address_equal_no_port(a, b)`**
- 原型：`gboolean nice_address_equal_no_port(const NiceAddress *a, const NiceAddress *b)`
- 目的：比较地址（忽略端口），仅比较 IP 地址部分

**`nice_address_is_private(addr)`**
- 原型：`gboolean nice_address_is_private(const NiceAddress *addr)`
- 目的：判断是否为私有/不可路由地址
- IPv4 私有地址判断：检查 10.0.0.0/8、172.16.0.0/12、192.168.0.0/16、169.254.0.0/16（APIPA）、127.0.0.0/8（loopback）
- IPv6 私有地址判断：检查 fe80::/10（link-local）、fd00::/8（ULA）、fc00::/7（ULA）、::1（loopback）

**`nice_address_is_linklocal(addr)`**
- 原型：`gboolean nice_address_is_linklocal(const NiceAddress *addr)`
- 目的：判断是否为本地链路地址
- IPv4：检查 169.254.0.0/16
- IPv6：检查 fe80::/10

**`nice_address_is_valid(addr)`**
- 原型：`gboolean nice_address_is_valid(const NiceAddress *addr)`
- 目的：检查地址族是否为有效的 AF_INET 或 AF_INET6

**`nice_address_ip_version(addr)`**
- 原型：`int nice_address_ip_version(const NiceAddress *addr)`
- 目的：返回 IP 版本（4 或 6），无效地址返回 0

**`nice_address_set_from_sockaddr(addr, sa)` / `nice_address_copy_to_sockaddr(addr, sa)`**
- 原型：
  - `void nice_address_set_from_sockaddr(NiceAddress *addr, const struct sockaddr *sa)`
  - `void nice_address_copy_to_sockaddr(const NiceAddress *addr, struct sockaddr *sa)`
- 目的：在 NiceAddress 和 BSD socket API 的 `sockaddr` 结构之间互相转换
- 关键逻辑：使用 union 进行类型安全的指针转换（`sockaddr` → `sockaddr_in` / `sockaddr_in6`），通过 `memcpy` 复制

#### 4.3.4 Windows 兼容性

在 Windows 上，`address.c` 提供了本地的 `inet_ntop_win32()` 实现，基于 `getnameinfo()` 配合 `NI_NUMERICHOST` 标志。同时定义了 `IN6_ARE_ADDR_EQUAL` 宏（如果系统未定义）。

### 4.4 Interfaces (interfaces.c/h, 1027 lines)

#### 4.4.1 模块概述

`interfaces` 模块提供跨平台的本地网络接口枚举和 IP 地址发现功能，需要处理三种平台：
- **Unix (HAVE_GETIFADDRS)**：使用 POSIX `getifaddrs()` API
- **Unix (fallback)**：当 `getifaddrs()` 不可用时，使用 `ioctl(SIOCGIFCONF)` 作为回退
- **Windows**：使用 Win32 `GetAdaptersAddresses()` API

#### 4.4.2 公共函数

**`nice_interfaces_get_local_ips(include_loopback)`**
- 原型：`GList *nice_interfaces_get_local_ips(gboolean include_loopback)`
- 目的：获取本机所有 IPv4/IPv6 地址列表（去重）
- 关键逻辑：
  1. 根据平台选择 `getifaddrs()`、`ioctl` 回退或 Windows `GetAdaptersAddresses()`
  2. 遍历所有网络接口，跳过以下情况：
     - 接口 Down 状态（`IFF_UP` 未设置）
     - 接口 Not Running（`IFF_RUNNING` 未设置）
     - 地址族不是 `AF_INET` 或 `AF_INET6`
     - macOS 上排除 `awdl`、`llw` 和 `utun` (link-local) 伪接口
     - BSD 上排除 `IFM_ACTIVE` 为 0 的接口
  3. Loopback 地址默认排除，除非 `include_loopback=TRUE`（追加到列表末尾）
  4. 私有 IP 追加到列表尾部，公网 IP 插入到列表头部（`add_ip_to_list()`）
  5. Windows 上额外使用 `GetBestInterfaceEx()` 获取最佳路由接口，将其 IP 放在列表开头
  6. 支持 `IGNORED_IFACE_PREFIX` 编译时配置，可排除特定前缀的接口

**`nice_interfaces_get_local_interfaces()`**
- 原型：`GList *nice_interfaces_get_local_interfaces(void)`
- 目的：获取本机所有网络接口名称列表
- 关键逻辑：类似 `get_local_ips`，但只返回接口名称而非 IP 地址

**`nice_interfaces_get_ip_for_interface(interface_name)`**
- 原型：`gchar *nice_interfaces_get_ip_for_interface(gchar *interface_name)`
- 目的：获取指定接口名称的 IPv4 地址
- 关键逻辑：
  - Unix：通过 `ioctl(SIOCGIFADDR)` 获取接口地址
  - Windows：遍历适配器匹配 FriendlyName，返回首个 IPv4 unicast 地址

**`nice_interfaces_get_if_index_by_addr(addr)`**
- 原型：`guint nice_interfaces_get_if_index_by_addr(NiceAddress *addr)`
- 目的：根据 NiceAddress 查找对应的网络接口索引（用于需要接口索引的系统 API）
- 关键逻辑：
  - Unix：遍历接口列表，通过 `nice_address_equal_no_port()` 匹配后使用 `if_nametoindex()` 或 `ioctl(SIOCGIFINDEX)` 获取索引
  - Windows：遍历适配器 unicast 地址，匹配后返回 `IfIndex`（IPv4）或 `Ipv6IfIndex`（IPv6）

#### 4.4.3 辅助函数

**`sockaddr_to_string(addr)`**
- 原型：`static gchar *sockaddr_to_string(const struct sockaddr *addr)`
- 目的：将 BSD sockaddr 转换为字符串
- 关键逻辑：使用 `getnameinfo()` 配合 `NI_NUMERICHOST` 标志（不做 DNS 反向查询）

**`add_ip_to_list(list, ip, append)`**
- 原型：`static GList *add_ip_to_list(GList *list, gchar *ip, gboolean append)`
- 目的：向链表添加 IP，自动去重
- 关键逻辑：遍历现有链表检查重复，无重复则根据 `append` 参数决定追加或前置

#### 4.4.4 排序语义

`nice_interfaces_get_local_ips()` 返回的列表具有特定的排序规则：
1. **公网 IPv4/IPv6** 地址排在最前（prepend）
2. **私有 IPv4/IPv6** 地址排在中间（append）
3. **Loopback** 地址排在最后（如果 include_loopback=TRUE）
4. Windows 额外将**最佳路由接口**的地址放在列表头部

此排序被 `nice_candidate_ip_local_preference()` 用于计算候选优先级中 `other_preference` 字段的值，保证候选优先级在相同类型候选之间具有唯一的排序。

---

## 第五部分：连接检查 (Conncheck) -- 第一部分

### 概述

`conncheck.c` 是 `agent/` 模块第二大文件（5035 行），实现 ICE 连接检查状态机的全部逻辑。模块职责覆盖：
- 候选对（CandidateCheckPair）的创建、排序和生命周期管理
- 按优先级调度连接检查，支持普通检查和触发检查两种发起路径
- STUN Binding Request/Response 的构造、发送和重传
- 候选对状态机（WAITING / FROZEN / IN_PROGRESS / SUCCEEDED / FAILED / DISCOVERED）
- 提名（Nomination）逻辑：Regular 和 Aggressive 两种模式
- 角色冲突检测和候选对优先级重算
- Peer Reflexive 候选的发现与 Valid Pair 的构造
- 入选对（Selected Pair）的更新和组件状态推进

整个连接检查流程由 **Ta 定时器**（`priv_conn_check_tick_agent_locked`）驱动，通过 `GMainLoop` / `GMainContext` 周期性回调推进。

### conncheck.h -- 核心类型

#### CandidateCheckPair 结构体

候选对是连接检查的核心数据结构，关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `stream_id` / `component_id` | `guint` | 所属流和组件 ID |
| `local` / `remote` | `NiceCandidate *` | 指向本地和远程候选 |
| `sockptr` | `NiceSocket *` | 发送检查使用的套接字（通常是 `local->sockptr`，TCP 被动模式除外） |
| `foundation` | `gchar[NICE_CANDIDATE_PAIR_MAX_FOUNDATION]` | 候选对 foundation，格式为 `"local_foundation:remote_foundation"` |
| `state` | `NiceCheckState` | 当前状态（见下方枚举） |
| `nominated` | `gboolean` | 是否已被提名（控制方或被控方设置） |
| `valid` | `gboolean` | 是否为有效对（Valid Pair），触发时对应 Peer Reflexive 发现 |
| `use_candidate_on_next_check` | `gboolean` | 下次检查是否携带 USE-CANDIDATE 属性（Regular nomination） |
| `mark_nominated_on_response_arrival` | `gboolean` | 收到响应时标记为 nominated（被控方处理 incoming check 用） |
| `retransmit` | `gboolean` | 首个 STUN 请求是否允许重传（提名后低优先级对会被停止重传） |
| `discovered_pair` | `CandidateCheckPair *` | 指向由此对发现的 DISCOVERED 对（peer-reflexive） |
| `succeeded_pair` | `CandidateCheckPair *` | 指向生成此 DISCOVERED 对的 SUCCEEDED 父对 |
| `priority` | `guint64` | 候选对优先级（`agent_candidate_pair_priority()` 计算） |
| `stun_priority` | `guint32` | STUN 请求中携带的 PRIORITY 值（`stun_request_priority()`） |
| `stun_transactions` | `GSList *` | 进行中的 STUN 事务链表（重传支持多事务并存） |

#### StunTransaction 结构体

```c
struct _StunTransaction {
  gint64 next_tick;        /* 下次 tick 时间戳（微秒） */
  StunTimer timer;         /* STUN 重传定时器 */
  uint8_t buffer[STUN_MAX_MESSAGE_SIZE_IPV6];  /* STUN 消息缓冲区 */
  StunMessage message;     /* 解析后的 STUN 消息结构 */
};
```

每个候选对可以维护多个 STUN 事务（`stun_transactions` 链表）。事务由 `priv_add_stun_transaction()` 创建并添加到链表头部，旧事务在新事务发送后继续存活至超时或成功为止。每个事务有独立的重传定时器 `StunTimer`，支持不可靠传输（UDP，指数退避）和可靠传输（TCP，固定超时）两种模式。

#### NiceCheckState 枚举

```c
typedef enum {
  NICE_CHECK_WAITING     = 1,  // 等待调度
  NICE_CHECK_IN_PROGRESS,      // 检查已发起，等待响应
  NICE_CHECK_SUCCEEDED,        // 检查成功（收到有效响应）
  NICE_CHECK_FAILED,           // 检查失败（重传耗尽或出错）
  NICE_CHECK_FROZEN,           // 冻结（等待同类 foundation 的检查完成）
  NICE_CHECK_DISCOVERED,       // 已发现（peer-reflexive 产生的有效对）
} NiceCheckState;
```

状态可视化输出（`priv_state_to_gchar`）：`W`=WAITING, `I`=IN_PROGRESS, `S`=SUCCEEDED, `F`=FAILED, `Z`=FROZEN, `D`=DISCOVERED。

`SET_PAIR_STATE` 宏在状态变更时自动输出调试日志（Agent 指针、pair 指针、新状态名、调用函数名）。

#### 公开函数（conncheck.h）

| 函数 | 说明 |
|------|------|
| `conn_check_add_for_candidate()` | 新增远程候选时创建候选对 |
| `conn_check_add_for_local_candidate()` | 新增本地候选时创建候选对 |
| `conn_check_add_for_candidate_pair()` | 检测兼容性后创建单个候选对 |
| `conn_check_free()` | 释放所有连接检查资源 |
| `conn_check_send()` | 发送单个候选对的连接检查 |
| `conn_check_prune_stream()` | 清除指定流的所有检查 |
| `conn_check_handle_inbound_stun()` | 处理入站 STUN 检查请求/响应 |
| `conn_check_compare()` | 候选对排序比较函数（按优先级降序） |
| `conn_check_remote_candidates_set()` | 远程候选设置后处理缓存的 incoming checks |
| `conn_check_remote_credentials_set()` | 远程凭据设置后的延迟处理入口 |
| `conn_check_match_transport()` | TCP active/passive 传输类型匹配转换 |
| `conn_check_prune_socket()` | 清除指定套接字关联的检查 |
| `recalculate_pair_priorities()` | 角色冲突后重算所有候选对优先级 |
| `conn_check_update_selected_pair()` | 更新组件的入选对 |
| `conn_check_update_check_list_state_for_ready()` | 推进组件状态到 READY |
| `conn_check_unfreeze_related()` | 成功后解冻同 foundation 的 FROZEN 对 |
| `conn_check_stun_transactions_count()` | 统计所有进行中的 STUN 事务数 |

---

### 候选对管理

#### conn_check_add_for_candidate_pair() -- 候选对创建（入口）

```c
gboolean conn_check_add_for_candidate_pair (NiceAgent *agent, guint stream_id,
    NiceComponent *component, NiceCandidate *local, NiceCandidate *remote);
```

**配对规则（ICE 6.1.2.4 "Pruning the pairs" RFC 8445 + ice-tcp-13 6.2）**：

1. **排除本地 srv-reflexive / peer-reflexive 候选**：`local->type == SERVER_REFLEXIVE || PEER_REFLEXIVE` 时直接返回 FALSE（RFC 兼容模式）
2. **排除本地 TCP-PASSIVE 候选**：避免创建由被动方发起的 TCP 检查对
3. **传输类型匹配**：`local->transport == conn_check_match_transport(remote->transport)` -- TCP active 与 TCP passive 互相匹配，UDP 与 UDP 匹配
4. **地址族一致**：`local->addr.s.addr.sa_family == remote->addr.s.addr.sa_family`
5. **禁止 link-local 与非 link-local 配对**：`!_is_linklocal_to_non_linklocal()`

`conn_check_match_transport()` 实现传输匹配逻辑：
- `TCP_ACTIVE` 返回 `TCP_PASSIVE`
- `TCP_PASSIVE` 返回 `TCP_ACTIVE`
- `UDP` / `TCP_SO` 原样返回

配对成功后调用 `priv_conn_check_add_for_candidate_pair_matched()`，以 `NICE_CHECK_FROZEN` 初始状态创建候选对。

#### conn_check_add_for_candidate() -- 批量创建候选对

新增远程候选时遍历 `component->local_candidates`，对每个本地候选调用 `conn_check_add_for_candidate_pair()`。特殊规则：
- **远程 peer-reflexive 候选不参与配对**（RFC 5245 7.2.1.3 "Learning Peer Reflexive Candidates"）
- 当 `agent->force_relay` 为 TRUE 时，仅配对本地的 relayed 类型候选

#### conn_check_add_for_local_candidate() -- 本地候选的批量配对

新增本地候选时遍历 `component->remote_candidates`，对每个远程候选调用 `conn_check_add_for_candidate_pair()`。本地 peer-reflexive 候选同样不参与配对（RFC 5245 7.1.3.2.1）。

#### priv_add_new_check_pair() -- 候选对创建（核心实现）

候选对创建的核心流程：

1. **优先级检查**：计算候选对优先级，若低于当前组件 `selected_pair.priority`，直接返回 NULL（跳过低优先级对）
2. **内存分配**：`g_slice_new0(CandidateCheckPair)` 零初始化
3. **套接字选择**：TCP-PASSIVE 本地候选 + peer-reflexive 远程候选时，使用远程候选的 `sockptr`（被动 TCP 连接对应的已连接套接字）；否则使用本地候选的 `sockptr`
4. **Foundation 构造**：`"local_foundation:remote_foundation"` 格式，长度上限 `NICE_CANDIDATE_PAIR_MAX_FOUNDATION`（`NICE_CANDIDATE_MAX_FOUNDATION * 2`）
5. **优先级计算**：`agent_candidate_pair_priority()` 根据控制方/被控方角色计算
6. **STUN 优先级**：`stun_request_priority()` -- 本地 HOST 类型候选使用 peer-reflexive 候选的优先级值，其余类型使用原优先级
7. **排序插入**：`g_slist_insert_sorted()` 按优先级降序插入 `stream->conncheck_list`
8. **调度定时器**：`priv_schedule_next()` 启动连接检查和保活定时器
9. **解冻尝试**：若初始状态为 FROZEN，调用 `priv_conn_check_unfreeze_maybe()` 检查同 foundation 是否已有 SUCCEEDED 对
10. **数量限制**：RFC 5245 兼容模式下，`priv_limit_conn_check_list_size()` 裁剪超出 `agent->max_conn_checks` 的低优先级 FROZEN 对，同时移除所有 FAILED 对

#### priv_compute_pair_priority() 和 recalculate_pair_priorities()

候选对优先级由 `agent_candidate_pair_priority()` 计算（定义在 `agent/agent.c`），公式遵循 RFC 8445 Section 6.1.2.3：

```
pair priority = 2^32 * MIN(G, D) + 2 * MAX(G, D) + (G > D ? 1 : 0)
```
其中 G = 控制方候选优先级，D = 被控方候选优先级。

`recalculate_pair_priorities()` 在角色冲突后重算 `conncheck_list` 中所有候选对的优先级并重新排序。

#### priv_limit_conn_check_list_size() -- 候选对数量硬限制

遵循 RFC 8445 Section 6.1.2.5：当候选对数量超过 `agent->max_conn_checks` 时：
- 移除状态为 FROZEN 的低优先级对（安全移除，不影响正在进行中的检查）
- 无条件移除所有 FAILED 对
- WAITING / IN_PROGRESS / SUCCEEDED / DISCOVERED 状态的候选对**保留**，因为移除可能破坏进行中的连接检查

---

### 检查发起

#### conn_check_send() -- 发送连接检查请求

```c
int conn_check_send (NiceAgent *agent, CandidateCheckPair *pair);
```

**发送流程**：

1. **获取凭据**：`priv_create_username()` 构建 USERNAME（格式：`remote_ufrag:local_ufrag`），`priv_get_password()` 获取 remote password
2. **兼容性处理**：MSN/OC2007 模式下对 password 进行 base64 解码
3. **USE-CANDIDATE 决策**：
   - Regular Nomination：使用 `pair->use_candidate_on_next_check` 标记
   - Aggressive Nomination：控制方发出的每个检查均携带 USE-CANDIDATE
   - 非 RFC 兼容模式下直接设置 `nominated = controlling`
4. **创建 STUN 事务**：`priv_add_stun_transaction()` 分配并加入 `pair->stun_transactions` 链表
5. **构造 STUN Binding Request**：`stun_usage_ice_conncheck_create()` 生成完整的 STUN 消息（USERNAME、MESSAGE-INTEGRITY、PRIORITY、ICE-CONTROLLED/ICE-CONTROLLING、USE-CANDIDATE、FINGERPRINT 等属性）
6. **启动重传定时器**：
   - 不可靠传输（UDP）：`stun_timer_start()` 使用 `priv_compute_conncheck_timer()` 计算 Ta 超时（`MAX(Ta * waiting_in_progress_count, STUN_TIMER_DEFAULT_TIMEOUT)`）
   - 可靠传输（TCP）：`stun_timer_start_reliable()` 使用 `agent->stun_reliable_timeout`（更长超时）
7. **TCP-Active 特殊处理**：若 `sockptr->fileno == NULL` 且不是 UDP-TURN 且是 TCP-ACTIVE，调用 `nice_tcp_active_socket_connect()` 创建新连接套接字
8. **发送**：`agent_socket_send()` 通过套接字发送 STUN 消息缓冲区
9. **OC2007R2 兼容**：额外调用 `ms_ice2_legacy_conncheck_send()` 发送带正确 FINGERPRINT 的副本

**凭据格式（priv_create_username）**：
- RFC 5245：`remote_ufrag:local_ufrag`（outbound）/ `local_ufrag:remote_ufrag`（inbound）
- OC2007R2 / WLM2009：同上，但填充到 4 字节对齐
- Google：`remote_ufrag` + `local_ufrag`（无冒号分隔）
- MSN / OC2007：`base64(remote_ufrag)`:component_id:`base64(local_ufrag)`:component_id（含 base64 解码步骤）

#### priv_conn_check_initiate() -- 启动对指定候选对的检查

```c
static gboolean priv_conn_check_initiate (NiceAgent *agent, CandidateCheckPair *pair)
```

1. `SET_PAIR_STATE(agent, pair, NICE_CHECK_IN_PROGRESS)` 更新状态
2. 调用 `conn_check_send()` 发送 STUN 请求
3. 若发送失败，调用 `candidate_check_pair_fail()` 将候选对标记为 FAILED，并触发 `conn_check_update_check_list_state_for_ready()` 尝试推进组件状态

#### priv_conn_check_tick_agent_locked() -- 主定时器回调

```c
static gboolean priv_conn_check_tick_agent_locked (NiceAgent *agent, gpointer user_data)
```

这是连接检查的核心调度器，每个 Ta 定时周期执行一次，按优先级顺序处理四步：

1. **触发检查（Triggered Checks）** -- 最高优先级：从 `agent->triggered_check_queue` 取出候选对并执行 `priv_conn_check_initiate()`
2. **进行中的 STUN 事务处理** -- `priv_conn_check_tick_stream()`：遍历所有 IN_PROGRESS 候选对的 STUN 事务，调用 `stun_timer_refresh()` 检查重传/超时，重传超时耗尽则候选对失败
3. **普通检查（Ordinary Checks）** -- `priv_conn_check_ordinary_check()`：找到 WAITING 状态的最高优先级候选对，若无则先调用 `priv_conn_check_unfreeze_next()` 解冻下一批，然后发起检查
4. **提名处理** -- `priv_conn_check_tick_stream_nominate()`：评估停止条件并选择候选对进行提名

**重要设计原则**：每个定时器回调只发送一个 STUN 请求（通过 `stun_sent` 标志控制），以遵守 STUN pacing 约束。

**空闲超时**：当所有工作结束（`keep_timer_going == FALSE`）时，启动 `conncheck_ongoing_idle_delay` 宽限期计数器。当空闲时间达到 `agent->idle_timeout` 后，检查并标记失败组件，停止 conncheck 定时器，代理状态转换为 COMPLETED。

#### priv_conn_check_tick_stream() -- 流级 STUN 事务处理

```c
static gboolean priv_conn_check_tick_stream (NiceAgent *agent, NiceStream *stream)
```

遍历 `stream->conncheck_list` 中所有 IN_PROGRESS 候选对的活动 STUN 事务，调用 `stun_timer_refresh()`：
- `STUN_USAGE_TIMER_RETURN_TIMEOUT`：移除单个事务；非首个事务（index > 0）或重传被禁用时直接超时
- `STUN_USAGE_TIMER_RETURN_RETRANSMIT`：重新发送 STUN 消息，更新 `next_tick`
- `STUN_USAGE_TIMER_RETURN_SUCCESS`：更新 `next_tick`，增加 `remaining` 计数

当所有事务耗尽（`remaining == 0`）时，候选对被标记为 FAILED。

---

### 检查状态机

候选对状态转换流程（RFC 8445 Section 7.2.5.3.3）：

```
FROZEN ──(unfreeze)──> WAITING ──(initiate)──> IN_PROGRESS
                                                  │
                                    ┌─────────────┼─────────────┐
                                    │             │             │
                               (Success)     (Peer Refx)    (Timeout/Error)
                                    │             │             │
                                    v             v             v
                               SUCCEEDED     DISCOVERED      FAILED
                                    │             │
                                    └─────┬───────┘
                                     (Valid, nominated)
                                          │
                                          v
                                    [Selected Pair]
```

#### 解冻策略（Unfreezing）

**priv_conn_check_unfreeze_next()** -- RFC 8445 Section 6.1.4.2：
1. 如果已有 WAITING 状态的候选对存在，直接返回（不做解冻）
2. 否则，按 foundation 分组，每个不同的 foundation 解冻一个 FROZEN 对，将其置为 WAITING

**conn_check_unfreeze_related()** -- 检查成功后调用：
- 遍历所有流的所有候选对，找到 foundation 与成功对相同的所有 FROZEN 对，全部置为 WAITING
- 遵循 RFC 8445 Section 7.1 的简化解冻算法（相比 RFC 5245 简化，幂等）

**priv_conn_check_unfreeze_maybe()** -- 候选对创建时调用：
- 若新候选对初始状态为 FROZEN，检查同 foundation 是否已有 SUCCEEDED 对，若有则立即解冻

#### 候选对失败处理

`candidate_check_pair_fail()`：
1. 调用 `SET_PAIR_STATE(agent, p, NICE_CHECK_FAILED)`
2. 释放所有 STUN 事务（`priv_free_all_stun_transactions()`）
3. 级联失败关联的 `discovered_pair`（Regular nomination 模式下网络条件变化时的边界情况）

---

### 提名逻辑 (Nomination)

#### priv_conn_check_tick_stream_nominate() -- 提名主逻辑

此函数聚合流内和跨流的候选对计数器（FROZEN / IN_PROGRESS / WAITING / SUCCEEDED / DISCOVERED / NOMINATED / VALID），根据 Nomination Mode 和 Controlling Mode 决定提名策略：

**Regular Nomination（RFC 5245 8.1.1.1）**：
- 仅控制方执行
- 跨流/跨组件搜索已提名的候选对，用于保持各组件传输类型一致
- 停止条件（Stopping Criterion）优先级：
  1. 存在 HOST-HOST 有效对（直接停止）
  2. 存在 >= `NICE_MIN_NUMBER_OF_VALID_PAIRS`（2）个有效对
  3. 找到与其他组件已提名对传输兼容的有效对
  4. 找到与其他流已提名对传输兼容的有效对
  5. FROZEN / WAITING / IN_PROGRESS 全部耗尽（无更多检查可做）
- 选中候选对后设置 `use_candidate_on_next_check = TRUE`，加入触发队列

**Aggressive Nomination（RFC 5245 8.1.1.2）**：
- 控制方为每个组件选择最高优先级的 SUCCEEDED/DISCOVERED 对直接设置为 nominated
- 加入触发队列发起带 USE-CANDIDATE 的检查

**被控方提名**：
- 被控方由收到对端的 USE-CANDIDATE 触发（通过 `priv_mark_pair_nominated()` 处理）
- 同样检查其他流/组件来保持传输类型一致性

#### priv_mark_pair_nominated() -- 标记候选对为已提名

在收到入站 STUN 请求（带 USE-CANDIDATE 属性）时调用：
1. 在 `conncheck_list` 中找到匹配的候选对
2. 若候选对为 SUCCEEDED 且有 `discovered_pair`，切换到 discovered_pair（TCP 场景下的 peer-reflexive 对）
3. 若候选对在 `triggered_check_queue` 中或为 IN_PROGRESS，设置 `mark_nominated_on_response_arrival = TRUE`（响应到达时提名）
4. 若候选对已 valid，设置 `nominated = TRUE`，更新 selected pair，推进组件状态

---

### 触发检查 (Triggered Checks)

#### 触发检查队列

触发检查队列是 `NiceAgent` 中的 `GSList *triggered_check_queue`，提供以下操作：

- `priv_add_pair_to_triggered_check_queue()` -- 去重添加候选对到队列尾部，调用 `priv_schedule_next()` 启动定时器
- `priv_remove_pair_from_triggered_check_queue()` -- 从队列中移除候选对
- `priv_get_pair_from_triggered_check_queue()` -- 取出队列头部候选对并移除

#### priv_schedule_triggered_check() -- 调度触发检查

在收到远端 STUN Binding Request 后调用（RFC 8445 Section 7.2.1.4 "Triggered Checks"）：

1. 在 `conncheck_list` 中查找匹配的候选对（component_id + remote + sockptr 匹配）
2. 若匹配到 DISCOVERED 对，使用其 `succeeded_pair` 父对
3. 根据候选对状态执行不同操作：

| 状态 | 行为 |
|------|------|
| WAITING / FROZEN | 直接加入触发检查队列 |
| IN_PROGRESS | 仅当优先级高于当前 selected pair 时加入队列（重新发起检查） |
| FAILED | 仅当优先级高于 selected pair 时加入队列，并将组件状态回退到 CONNECTING/CONNECTED |
| SUCCEEDED | 无需操作（已完成） |

4. 若未找到匹配的候选对，检查 `local_candidates` 是否能匹配套接字，若能则在 conncheck_list 中创建新候选对（WAITING 状态）并加入触发队列

#### conn_check_remote_candidates_set() -- 延迟触发检查处理

处理场景（RFC 5245 Section 7.2）：对端在 SDP answer 到达之前就发来了 STUN Binding Request。这些请求被缓存在 `component->incoming_checks` 队列中。当远程凭据和候选就绪后，遍历缓存的 incoming checks：
1. 匹配 `local_socket` 找到对应的本地候选
2. 匹配 `from` 地址和远程候选
3. 调用 `priv_schedule_triggered_check()` 发起触发检查
4. 若 incoming check 携带 `use_candidate`，调用 `priv_mark_pair_nominated()` 处理提名

#### priv_reply_to_conn_check() -- 回复连接检查

收到 STUN Binding Request 后的回复流程：
1. 通过 `agent_socket_send()` 发送 STUN Binding Response
2. 若远程凭据已知（`stream->remote_ufrag[0]` 非空），调用 `priv_schedule_triggered_check()` 发起反向检查
3. 若请求携带 USE-CANDIDATE，调用 `priv_mark_pair_nominated()` 处理提名

#### priv_store_pending_check() -- 缓存未决检查

当远程候选尚未设置时，将入站 STUN 请求存储为 `IncomingCheck` 结构体，加入 `component->incoming_checks` 队列。数量上限为 `max_conn_checks * 2`，超过上限时丢弃最低优先级的检查。

---

### 连接检查生命周期

#### 启动（priv_schedule_next）

```c
static void priv_schedule_next (NiceAgent *agent)
```

当第一个候选对被加入 `conncheck_list` 时调用，同时启动两个定时器：
- **conncheck_timer_source**：`priv_conn_check_tick_agent_locked`，周期 `agent->timer_ta`
- **keepalive_timer_source**：`priv_conn_keepalive_tick_agent_locked`，周期 `agent->timer_ta`

若仍有未完成的候选发现（`discovery_unsched_items > 0`），发出警告但依然启动连接检查。

#### 停止（conn_check_stop）

```c
static void conn_check_stop (NiceAgent *agent)
```

销毁 `conncheck_timer_source`，清零 `conncheck_ongoing_idle_delay`。注意：保活定时器独立于 conncheck 定时器，即使连接检查完成后仍继续运行。

#### 清理

- `candidate_check_pair_free()` -- 从触发队列移除并释放单个候选对的 STUN 事务和内存
- `conn_check_free()` -- 释放所有流的所有候选对，停止 conncheck 定时器
- `conn_check_prune_stream()` -- 清除指定流的所有候选对，若所有流均无检查则停止定时器

---

### 角色冲突处理

#### priv_check_for_role_conflict()

当处理入站检查或出站检查的响应时，若检测到的角色（controlling/controlled）与当前 `agent->controlling_mode` 不一致：
1. 翻转 `agent->controlling_mode` 标志
2. 调用 `recalculate_pair_priorities()` 重算所有候选对的优先级
3. 按新优先级重新排序所有 `conncheck_list`

### priv_conn_keepalive_tick_unlocked() -- 保活逻辑

独立的保活定时器处理两种场景：

1. **会话已建立（Selected Pair 存在）**：
   - TCP 候选默认跳过保活（除非 `keepalive_conncheck` 或 `consent_freshness` 启用）
   - `NICE_AGENT_DO_KEEPALIVE_CONNCHECKS()` 为 TRUE 时：发送完整的 STUN 连接检查（含用户名/密码），启动 consent freshness 计时器（RFC 7675），使用随机抖动（0.8x ~ 1.2x `NICE_AGENT_TIMER_CONSENT_DEFAULT`）
   - 否则：发送简单的 STUN Binding Keepalive（无凭据）
   - OC2007R2 兼容：额外发送 MS-ICE2 legacy conncheck

2. **连接建立中**：定期向 STUN 服务器发送 Binding Request，保持 NAT 映射存活（`NICE_AGENT_TIMER_TR_DEFAULT` 周期，仅 UDP HOST 候选）

### conn_check_update_selected_pair() -- 更新 Selected Pair

当候选对的优先级高于当前 `component->selected_pair.priority` 时：
1. 构造 `CandidatePair cpair`
2. 调用 `nice_component_update_selected_pair()` 更新组件内部状态
3. 手动 tick 一次保活逻辑（`priv_conn_keepalive_tick_unlocked()`）
4. 发射 `new-selected-pair` 信号通知客户端

### conn_check_update_check_list_state_for_ready() -- 推进到 READY

当组件中存在 nominated 对时：
1. 统计 valid 和 nominated 数量
2. 若存在 nominated，调用 `priv_prune_pending_checks()` 裁剪低优先级对
3. 若无进行中的检查（prune 返回 0）：
   - FAILED / CONNECTING 状态推进到 CONNECTING
   - CONNECTED 状态推进到 CONNECTED（确保状态链完整）
   - 最终推进到 READY
4. 确保状态演变遵循 FAILED -> CONNECTING -> CONNECTED -> READY 的渐进路径

---

## 第六部分：连接检查 — 第二部分（响应处理与事务匹配）

### STUN 响应处理

#### priv_gen_username() / priv_create_username()
- **作用**: 为出站连接检查构造 STUN 用户名
- **关键逻辑**:
  - RFC5245 模式: `remote_ufrag:local_ufrag` 格式
  - WLM2009/OC2007R2 模式: 同上但 4 字节对齐填充
  - Google 模式: 直接拼接（无冒号分隔）
  - MSN/OC2007 模式: base64 解码后构造 `decoded_remote:comp_id:decoded_local:comp_id`

#### priv_get_password()
- **作用**: 获取出站连接检查的 STUN 密码
- **关键逻辑**: 优先使用候选的 `password`，否则使用流的 `remote_password`；Google 兼容模式返回空密码

#### conncheck_stun_validater()
- **作用**: 验证入站 STUN 消息的凭据（回调注册到 `StunAgent`）
- **关键逻辑**:
  - 遍历所有 local candidates 的 ufrag，与收到消息中的 username 做前缀匹配
  - 匹配成功则返回对应 candidate 的 password（或 stream 的 local_password）
  - MSN/OC2007 模式需要对 ufrag 和 password 做 base64 解码

#### priv_map_reply_to_conn_check_request()
- **作用**: 将收到的 STUN 响应匹配到发起检查的事务
- **关键逻辑**:
  1. 遍历所有 conncheck_list 中的候选对，比对其所有 STUN 事务的 transaction ID
  2. 若匹配成功，调用 `stun_usage_ice_conncheck_process()` 处理响应
  3. 校验响应来源地址是否与发送目标地址一致（不一致则 FAIL）
  4. 校验是否有对应的 remote_candidate（无则 FAIL）
  5. 成功分支: 调用 `priv_process_response_check_for_reflexive()` 提取 peer-reflexive
  6. 角色冲突分支: 通过 `priv_check_for_role_conflict()` 切换角色，将候选对放入触发队列重试
  7. 错误分支: 标记候选对为 FAILED
  8. 每次处理后调用 `conn_check_update_check_list_state_for_ready()` 推动状态机

#### priv_process_response_check_for_reflexive()
- **作用**: 从连接检查响应中提取 peer-reflexive 候选
- **关键逻辑**:
  1. 从响应中提取映射地址 (mapped address)，在 local candidates 中搜索是否已存在
  2. 若找到已存在的 local cand: 查找对应的已发现候选对，状态置 DISCOVERED；原候选对标记 SUCCEEDED
  3. 若未找到: 调用 `discovery_add_peer_reflexive_candidate()` 创建新的 peer-reflexive 本地候选
  4. 调用 `priv_add_peer_reflexive_pair()` 创建新的"已发现"候选对，标记为 VALID
  5. TCP-ACT/TCP-PASS 对始终创建 peer-reflexive 候选（因为端口号为 0）
  6. 原请求候选对标记 SUCCEEDED，清除 STUN 事务，从触发队列移除
  7. 有效对调用 `nice_component_add_valid_candidate()` 将 remote cand 加入有效列表

### USE-CANDIDATE 与提名 (Nomination)

#### ICE 提名标志更新 (位于 priv_map_reply_to_conn_check_request 内)
- **作用**: 处理 USE-CANDIDATE 标志和提名
- **关键逻辑**:
  - Controlling 端:
    - REGULAR 模式: 仅当 `use_candidate_on_next_check` 为 TRUE 时提名
    - AGGRESSIVE 模式: 首个成功响应的对即提名
  - Controlled 端: 当 `mark_nominated_on_response_arrival` 为 TRUE 时提名
  - 提名后调用 `conn_check_update_selected_pair()` 通知组件
  - 若状态未达 READY，先推进到 CONNECTED

#### priv_mark_pair_nominated()
- **作用**: 由入站检查触发，标记对端指定的候选对为 nominated
- **关键逻辑**:
  - 仅 controlled 端处理（controlling 端忽略）
  - 若对应的候选对状态为 SUCCEEDED 但有 discovered_pair，则替换为 discovered pair 进行提名
  - 若候选对在触发队列中或正在进行中，设置 `mark_nominated_on_response_arrival = TRUE` 延迟提名
  - 若候选对已 valid，直接标记 nominated 并更新 selected pair

### 响应分发到发现/刷新事务

#### priv_map_reply_to_discovery_request()
- **作用**: 匹配 STUN 响应到 srflx 发现事务
- **关键逻辑**:
  - 遍历 `agent->discovery_list`，仅处理类型为 `SERVER_REFLEXIVE` 的项
  - 调用 `stun_usage_bind_process()` 解析 Binding Response
  - SUCCESS: 调用 `discovery_add_server_reflexive_candidate()` 创建 srflx 候选，若 `use_ice_tcp` 则同时创建 TCP srflx
  - ALTERNATE_SERVER: 更新服务器地址，重新调度
  - ERROR: 标记 done

#### priv_map_reply_to_relay_request()
- **作用**: 匹配 STUN 响应到 TURN 分配事务
- **关键逻辑**:
  - 遍历 discovery_list，仅处理类型为 `RELAYED` 的项
  - 调用 `stun_usage_turn_process()` 解析 Allocate Response
  - RELAY_SUCCESS / MAPPED_SUCCESS:
    - 若应答同时包含映射地址: 创建 srflx 候选（UDP 情况）和 TCP srflx
    - 创建 relay 候选: 可靠 socket 下创建 TCP_ACTIVE + TCP_PASSIVE 两种候选；不可靠 socket 下创建 UDP 候选
    - 调用 `priv_add_new_turn_refresh()` 启动定期刷新计时器
    - 缓存 TURN 领域/随机数到 relay socket (用于后续请求认证)
  - ALTERNATE_SERVER: 重置所有同类型、同流的未完成发现项，指向新服务器
  - UNAUTHORIZED (stale nonce / 不匹配 realm): 保存响应消息用于重试认证，重新调度
  - ERROR: 标记 done

#### priv_map_reply_to_relay_refresh()
- **作用**: 匹配 STUN 响应到 TURN 刷新事务
- **关键逻辑**:
  - 遍历 `agent->refresh_list`，匹配事务 ID
  - SUCCESS: 重新设置定时器 (`priv_calc_turn_timeout(lifetime)`) 安排下次刷新
  - UNAUTHORIZED (stale nonce): 保存响应用于认证，立即重试刷新
  - ERROR: 释放 CandidateRefresh

#### priv_map_reply_to_relay_remove()
- **作用**: 匹配 STUN 响应到 TURN 移除（deallocate）事务
- **关键逻辑**: 仅处理 `disposing == TRUE` 的 refresh 项；收到有效响应（非 INVALID）后调用 `refresh_free()` 释放资源

#### priv_map_reply_to_keepalive_conncheck()
- **作用**: 处理收到对端的 consent freshness 保活检查
- **关键逻辑**: 更新 `component->selected_pair.remote_consent.last_received` 时间戳

### READY 判定

#### conn_check_update_check_list_state_for_ready()
- **作用**: 检查是否满足 READY 条件，推动组件状态机
- **关键逻辑**:
  1. 统计每个 component 的 valid 和 nominated 候选对数量
  2. 若存在 nominated 对:
     - 调用 `priv_prune_pending_checks()` 裁剪低优先级未完成检查
     - 若无进行中检查（prune 返回 0）: 逐步推进状态
  3. 状态演进路径: FAILED -> CONNECTING -> CONNECTED -> READY

#### priv_update_check_list_failed_components()
- **作用**: 检查并标记全部失败的组件
- **关键逻辑**:
  1. 若该流的发现仍在进行中（discovery_list 有未完成项），跳过
  2. 若 discovery_list 非空，跳过（等待所有发现完成）
  3. 遍历每个 component 的所有候选对: 若全部处于 FAILED/SUCCEEDED/DISCOVERED 且无 nominated 对，且 component 有 remote candidates，则推进到 FAILED 状态

### TURN 刷新管理

#### priv_add_new_turn_refresh()
- **作用**: 创建新的 TURN 分配定期刷新
- **关键逻辑**:
  - 维护从 CandidateRefresh 到 `agent->refresh_list` (storage) 以及定时器 (refresh_list)
  - lifetime > 0: 计算刷新间隔 `priv_calc_turn_timeout()`，设置秒级定时器，到期执行 `priv_turn_allocate_refresh_tick_agent_locked()`
  - lifetime == 0: 立即释放通过构件
  - 传递前一个 STUN 响应的认证信息（nonce/realm）到 refresh 对象
  - 跳过 OC2007/OC2007R2 的 TURN-TLS 刷新（不需要）

#### priv_turn_allocate_refresh_tick_unlocked()
- **作用**: 发送 TURN Refresh 请求
- **关键逻辑**:
  - 使用 `stun_usage_turn_create_refresh()` 构造刷新消息
  - lifetime 参数: `disposing ? 0 : -1`（移除时为 0，正常刷新为 -1 表示保持）
  - 启动 STUN 重传定时器并发送请求

#### CandidateRefresh 清理
- `refresh_free()` -- 从 refresh_list 和 pruning_refreshes 移除，销毁 timer/tick/destroy 源，调用 destroy 回调
- `refresh_prune_stream_async()` -- 异步移除流相关的所有 TURN 分配（收集 refresh 列表，错开时间逐一发送 deallocate）
- `refresh_prune_candidate()` / `refresh_prune_candidate_async()` -- 移除单个候选的刷新
- `refresh_prune_socket()` -- 移除指定 socket 关联的所有刷新

---

## 第七部分：候选发现 (Discovery)

### discovery.h / discovery.c (174 + 1478 = 1652 行)

#### 概述

`discovery.c` 实现 ICE 候选收集的完整流程，按 host -> srflx -> relay 三个阶段逐步发现本地候选。发现过程由定时器驱动的 `CandidateDiscovery` 状态机和 `CandidateRefresh` 刷新管理两部分组成。

#### CandidateDiscovery 结构体

```c
typedef struct {
  NiceCandidateType type;         // 发现类型: SERVER_REFLEXIVE 或 RELAYED
  NiceSocket *nicesock;           // 使用的底层 socket
  NiceAddress server;             // STUN/TURN 服务器地址
  gint64 next_tick;               // 下次 tick 时间戳
  gboolean pending;               // 是否正在处理中
  gboolean done;                  // 是否已完成
  guint stream_id;
  guint component_id;
  TurnServer *turn;               // TURN 服务器配置（仅 relay）
  StunAgent stun_agent;           // STUN 事务代理
  StunTimer timer;                // 重传定时器
  uint8_t stun_buffer[STUN_MAX_MESSAGE_SIZE_IPV6];
  StunMessage stun_message;       // 发出的 STUN 消息
  uint8_t stun_resp_buffer[STUN_MAX_MESSAGE_SIZE];
  StunMessage stun_resp_msg;      // 上次收到的 STUN 响应（用于认证重试）
} CandidateDiscovery;
```

#### Discovery 状态机

```
NOT_STARTED --[discovery_schedule()]-->
  GATHERING_HOST --[host candidates done]-->
  GATHERING_SRFLX --[STUN Binding 完成]-->
  GATHERING_RELAY --[TURN Allocate 完成]-->
  DONE --[agent_gathering_done()]
```

实际上三个阶段可以重叠。Discovery 在 `agent->streams` 和 `agent->discovery_list` 两个数据结构中协调: stream 负责建立 host candidate（在 `nice_agent_add_stream()` 和 `nice_agent_gather_candidates()` 中完成），discovery_list 负责 STUN/TURN 服务器的候选收集。

### 核心函数

#### discovery_add_local_host_candidate()
- **原型**: `HostCandidateResult discovery_add_local_host_candidate(NiceAgent *agent, guint stream_id, guint component_id, NiceAddress *address, NiceCandidateTransport transport, gboolean accept_duplicate, NiceCandidateImpl **outcandidate)`
- **作用**: 创建 HOST 类型本地候选
- **关键逻辑**:
  1. 根据 transport 类型创建对应 socket (UDP/TCP-ACTIVE/TCP-PASSIVE)
  2. 创建 `NiceCandidate` 对象，设置类型 HOST，计算 priority（按兼容模式选用不同算法）
  3. 生成凭证（MSN/OC2007: base64 随机用户名和密码; Google: 仅随机用户名）
  4. 分配 foundation（与已有相同类型+相同 base_addr 的 host 候选共享）
  5. 检查端口重复: 若同一端口已被其他流/组件的 host 候选使用，返回 `HOST_CANDIDATE_DUPLICATE_PORT`
  6. 调用 `priv_add_local_candidate_pruned()` 去重后加入 component 的 local_candidates 列表
  7. 将 socket 附加到 component，设置 TOS
  8. 返回值枚举: `SUCCESS`, `FAILED`, `CANT_CREATE_SOCKET`, `REDUNDANT`, `DUPLICATE_PORT`

#### discovery_add_server_reflexive_candidate()
- **原型**: `void discovery_add_server_reflexive_candidate(NiceAgent *agent, guint stream_id, guint component_id, NiceAddress *address, NiceCandidateTransport transport, NiceSocket *base_socket, const NiceAddress *server_address, gboolean nat_assisted)`
- **作用**: 创建 SERVER_REFLEXIVE 类型候选（由 STUN Binding 响应发现）
- **关键逻辑**:
  1. 创建 `NICE_CANDIDATE_TYPE_SERVER_REFLEXIVE` 候选
  2. 绑定到 `base_socket`，设置 `base_addr` 为 socket 地址
  3. 计算 priority（Google/Jingle、MSN、MS-ICE、标准 ICE 四种算法）
  4. 如果提供了 `server_address`，记录到 `c->stun_server`
  5. 生成凭证，分配 foundation（与同类型、同 base_addr、同 server 的已有候选共享）
  6. 去重后加入 local_candidates，发射 `new-candidate` 信号
  7. `nat_assisted` 标志影响 MS-ICE2 和标准 ICE 的优先级计算

#### discovery_add_relay_candidate()
- **原型**: `NiceCandidateImpl* discovery_add_relay_candidate(NiceAgent *agent, guint stream_id, guint component_id, NiceAddress *address, NiceCandidateTransport transport, NiceSocket *base_socket, TurnServer *turn, uint32_t *lifetime)`
- **作用**: 创建 RELAYED 类型候选（由 TURN Allocate 响应发现）
- **关键逻辑**:
  1. 创建 `NICE_CANDIDATE_TYPE_RELAYED` 候选，引用 `TurnServer` (`turn_server_ref()`)
  2. 创建 `NiceUDPSocket` (UDP TURN) 封装 relay 地址、base_socket、服务器地址和凭据
  3. 设置 transport (UDP/TCP_ACTIVE/TCP_PASSIVE)，计算 priority
  4. Google 兼容: 用 TURN 用户名覆盖候选用户名
  5. 去重后加入 local_candidates，将 relay_socket 附加到 component
  6. 冗余候选设置 `*lifetime = 0` 通知调用方无需刷新

#### discovery_add_peer_reflexive_candidate()
- **原型**: `NiceCandidate* discovery_add_peer_reflexive_candidate(NiceAgent *agent, guint stream_id, guint component_id, guint32 priority, NiceAddress *address, NiceSocket *base_socket, NiceCandidate *local, NiceCandidate *remote)`
- **作用**: 创建 PEER_REFLEXIVE 类型本地候选（由连接检查响应发现）
- **关键逻辑**:
  1. 创建 `NICE_CANDIDATE_TYPE_PEER_REFLEXIVE` 候选
  2. 传输类型推断: local 优先，其次 remote 反向匹配，最后根据 socket 类型判断
  3. priority 继承自发起 STUN 请求的候选对的值（RFC 5245 sect 7.1.3.2.1）
  4. foundation 与已有同类型、同base_addr 的候选共享
  5. MSN/OC2007 特有: 用户名由 local 和 remote 的 decoded usernames 合并后 base64 编码
  6. 去重后加入 local_candidates，不发射信号（内部候选）

#### discovery_learn_remote_peer_reflexive_candidate()
- **原型**: `NiceCandidate* discovery_learn_remote_peer_reflexive_candidate(NiceAgent *agent, NiceStream *stream, NiceComponent *component, guint32 priority, const NiceAddress *remote_address, NiceSocket *nicesock, NiceCandidate *local, NiceCandidate *remote)`
- **作用**: 从入站连接检查中学习对端 peer-reflexive 候选
- **关键逻辑**:
  1. 创建 `PEER_REFLEXIVE` 远程候选
  2. 传输类型: remote 优先，其次 local 反向匹配，最后 socket 类型判断
  3. priority: 使用 STUN 请求中的 PRIORITY 属性值；为 0 时按兼容模式计算
  4. foundation: 调用 `priv_assign_remote_foundation()` 与已有同类型同地址的远程候选共享
  5. MSN/OC2007: 用户名由 remote 和 local decoded usernames 合并
  6. 加入 `component->remote_candidates`，发射 `new-remote-candidate` 信号
  7. 不自动配对（RFC 7.2.1.3 规定）

#### discovery_discover_tcp_server_reflexive_candidates()
- **作用**: 为所有匹配 base_addr 的 TCP host 候选创建对应的 TCP srflx 候选
- **关键逻辑**:
  - 遍历 component 的所有 local_candidates
  - 对于每个 TCP (非 UDP) HOST 候选，其地址（端口为 0）与 base_addr 匹配时
  - 使用该候选的 transport 和 sockptr 创建 srflx

### 发现调度与定时器

#### priv_discovery_tick_unlocked()
- **作用**: 定时器驱动的发现进度回调（无锁版本）
- **关键逻辑**:
  1. 遍历 `agent->discovery_list`
  2. **新调度项** (pending == FALSE):
     - 设置 pending = TRUE，减少 `discovery_unsched_items`
     - srflx 类型: 用 `stun_usage_bind_create()` 构造 STUN Binding Request
     - relay 类型: 用 `stun_usage_turn_create()` 构造 TURN Allocate Request（含用户名/密码）
     - 发送 STUN 消息，启动重传定时器（可靠 socket 使用 `stun_timer_start_reliable`）
     - 记录 `next_tick`，递增 `need_pacing`（确保每轮只启动一个新发现，实现 Ta 步调）
  3. **进行中项** (pending == TRUE):
     - `stun_message.buffer == NULL`: 标记 done（发现被取消）
     - 重传超时 (`STUN_USAGE_TIMER_RETURN_TIMEOUT`): 标记 done，忘记事务
     - 需要重传 (`STUN_USAGE_TIMER_RETURN_RETRANSMIT`): 重新发送，更新 next_tick，递增 need_pacing
     - 等待中 (`STUN_USAGE_TIMER_RETURN_SUCCESS`): 更新 next_tick 继续等待
  4. 若 need_pacing 非零，跳出本次循环（确保 Ta 步调：每个 tick 最多启动一个新发现 + 处理一次重传）
  5. `not_done == 0`: 所有发现完成，调用 `discovery_free()` 和 `agent_gathering_done()`

#### priv_discovery_tick_agent_locked()
- **作用**: agent 锁定包装器，供 g_timeout_add 使用
- **关键逻辑**: 加锁调用 `priv_discovery_tick_unlocked()`，若返回 FALSE 则销毁 `discovery_timer_source`

#### discovery_schedule()
- **作用**: 启动/重新调度发现定时器
- **关键逻辑**:
  - 断言 `discovery_list` 非空
  - 若有未调度项 (`discovery_unsched_items > 0`) 且定时器未运行:
    - 立即执行首次 tick（`priv_discovery_tick_unlocked()`）
    - 若返回 TRUE（还有工作），启动周期定时器，间隔 `agent->timer_ta`

### 清理函数

#### discovery_free()
- **作用**: 释放 agent 的所有发现资源
- **关键逻辑**: `g_slist_free_full(discovery_list, discovery_free_item)`；销毁 `discovery_timer_source`；清零 `discovery_unsched_items`

#### discovery_free_item()
- **作用**: 释放单个 CandidateDiscovery
- **关键逻辑**: 若有关联的 TurnServer，`turn_server_unref()`；释放 CandidateDiscovery 内存

#### discovery_prune_stream()
- **作用**: 移除指定流的所有发现项
- **关键逻辑**: 遍历 discovery_list，移除 `stream_id` 匹配的项；若列表为空则调用 `discovery_free()`

#### discovery_prune_socket()
- **作用**: 移除指定 socket 关联的所有发现项
- **关键逻辑**: 遍历 discovery_list，移除 `nicesock` 匹配的项；若列表为空则调用 `discovery_free()`

### 内部辅助函数

#### priv_add_local_candidate_pruned()
- **作用**: 去重后添加本地候选到组件
- **关键逻辑**:
  1. 检查地址、base_addr 和 transport 完全相同的候选 -> 冗余
  2. 检查同类型 relay 候选（地址相同 + 端口不同 -> 冗余）
  3. 检查同类型 srflx 候选（地址相同 + 端口不同 -> 冗余）
  4. 通过检查后加入 `component->local_candidates`，并调用 `conn_check_add_for_local_candidate()` 与已存在的 remote candidates 配对

#### priv_assign_foundation()
- **作用**: 为本地候选分配 foundation
- **关键逻辑**: 遍历所有已存在的 local candidates，若同类型、同 transport、同 base_addr（relay 还需同 TURN server），共享 foundation；否则分配新 ID (`agent->next_candidate_id++`)

#### priv_assign_remote_foundation()
- **作用**: 为远程候选分配 foundation
- **关键逻辑**: 遍历 remote_candidates，匹配 type + transport + stream_id + base_addr；未匹配则分配 `"remoteN"` 格式的 foundation

#### priv_generate_candidate_credentials()
- **作用**: 为候选生成认证凭据
- **关键逻辑**:
  - MSN/OC2007: 32 字节随机用户名 + 16 字节随机密码，base64 编码
  - Google: 16 字节可打印随机字符串作为用户名，密码置 NULL

### 候选发现的初始化流程

候选发现在 `agent.c` 中通过以下函数初始化：

1. **`priv_add_new_candidate_discovery_stun()`**: 为每个 STUN server 地址创建 `CandidateDiscovery` (type=SERVER_REFLEXIVE)，初始化 `StunAgent`（短凭证 / 忽略凭证，按兼容模式调整），加入 `discovery_list`
2. **`priv_add_new_candidate_discovery_turn()`**: 为每个 TURN server 地址创建 `CandidateDiscovery` (type=RELAYED)，处理 UDP/TCP/TLS 连接方式，初始化 `StunAgent`（长凭证），加入 `discovery_list`
3. **`nice_agent_gather_candidates()`**: 创建 host candidates -> 解析 STUN/TURN 服务器地址 -> 创建 discovery items -> 调用 `discovery_schedule()` 启动发现

#### TURN 服务器连接的传输协议支持
- `NICE_RELAY_TYPE_TURN_UDP`: 直接使用 host socket
- `NICE_RELAY_TYPE_TURN_TCP` / `NICE_RELAY_TYPE_TURN_TLS`: 通过 `agent_create_tcp_turn_socket()` 建立 TCP/TLS 连接到 TURN 服务器

---

## 第八部分：I/O 接收与消息分派

### _nice_agent_recv() 消息接收流程

当底层 socket 收到数据时，`_nice_agent_recv()` (位于 `conncheck.c` 约第 4400 行起) 是中央分派点：

1. **读取数据**: 调用 `nice_socket_recv_messages()` 从 socket 接收
2. **STUN/TURN 消息过滤**: 检查数据是否以 STUN magic cookie 或兼容的 message type 开头
3. **消息分派**（按优先级依次尝试）:
   - `priv_map_reply_to_conn_check_request()` -- 连接检查响应
   - `priv_map_reply_to_discovery_request()` -- srflx 发现响应
   - `priv_map_reply_to_relay_request()` -- TURN 分配响应
   - `priv_map_reply_to_relay_refresh()` -- TURN 刷新响应
   - `priv_map_reply_to_relay_remove()` -- TURN 移除响应
   - `priv_map_reply_to_keepalive_conncheck()` -- 保活检查响应
   - 入站 STUN 连接检查处理（伪造 remote/peer-reflexive 候选，发送 response）
   - 入站 STUN Binding 请求响应（用于 consent freshness / old-style google stun）
4. **非 STUN 数据**: 转发给上层应用（通过 `nice_component_emit_io_callback()` 调用用户注册的回调）

---

## 第九部分：GIO 流封装

### 概述

`inputstream.c`、`outputstream.c` 和 `iostream.c` 三个文件将 `NiceAgent` 的可靠数据通道（reliable mode，即伪 TCP 模式）封装为标准 GIO 流接口，使基于 GIO 异步 I/O 模型的 GLib/GTK 应用程序能够以流式读写的方式使用 libnice 的 ICE 数据通道，无需直接调用 `nice_agent_recv()` / `nice_agent_send()`。

核心设计思想：
- **`NiceInputStream`** 继承 `GInputStream`，同时实现 `GPollableInputStream` 接口，支持同步/异步/非阻塞三种读取模式
- **`NiceOutputStream`** 继承 `GOutputStream`，同时实现 `GPollableOutputStream` 接口，支持同步/异步/非阻塞三种写入模式
- **`NiceIOStream`** 继承 `GIOStream`，将输入流和输出流组合为单一双向流对象
- 三个类均通过 `GWeakRef` 持有对 `NiceAgent` 的弱引用，避免循环引用
- 当底层 `NiceAgent` 的 stream 被移除时，通过 `"streams-removed"` 信号自动关闭对应的 GIO 流
- 所有类型均在 `Since: 0.1.5` 中引入，且均为稳定 API

### inputstream.c/h — NiceInputStream

**类型定义**：`NiceInputStream` 继承 `GInputStream`，并实现 `GPollableInputStream` 接口。私有结构体 `NiceInputStreamPrivate` 包含：
- `agent_ref` (`GWeakRef`)：对 `NiceAgent` 的弱引用
- `stream_id` / `component_id`：标识要包装的 stream 和 component

**构造属性**（均为 `G_PARAM_CONSTRUCT_ONLY`）：
- `NiceInputStream:agent` — 关联的 `NiceAgent`
- `NiceInputStream:stream-id` — stream ID
- `NiceInputStream:component-id` — component ID

---

**`nice_input_stream_new(NiceAgent *agent, guint stream_id, guint component_id)`**
```
原型: NiceInputStream *nice_input_stream_new(NiceAgent *agent, guint stream_id, guint component_id)
```
- **用途**: 创建一个新的 `NiceInputStream`，包装指定 agent 的给定 stream/component 对
- **关键逻辑**:
  1. 参数校验：`agent` 必须非空，`stream_id >= 1`，`component_id >= 1`
  2. 通过 `g_object_new()` 构造实例，三个构造属性在 `set_property` 中处理：
     - `agent` 通过 `g_weak_ref_set()` 存储弱引用，并连接 `"streams-removed"` 信号
     - `stream_id` 和 `component_id` 直接存储
  3. 注意：构造出的 `NiceInputStream` 不持有对 agent 的强引用；如果 agent 先于流销毁，后续所有操作返回 `G_IO_ERROR_CLOSED`

---

**`nice_input_stream_read(GInputStream *stream, void *buffer, gsize count, GCancellable *cancellable, GError **error)`**
```
原型: 虚函数，由 GInputStream 的 read_fn 指向
```
- **用途**: 同步阻塞读取数据
- **关键逻辑**:
  1. 检查流是否已关闭，已关闭则返回 0
  2. 通过 `g_weak_ref_get()` 获取 agent 强引用；若 agent 已被销毁，返回 `G_IO_ERROR_CLOSED`
  3. 调用 `nice_agent_recv(agent, stream_id, component_id, buffer, count, cancellable, error)` 执行阻塞读取
  4. 释放 agent 引用后返回读取字节数或错误

---

**`nice_input_stream_close(GInputStream *stream, GCancellable *cancellable, GError **error)`**
```
原型: 虚函数，由 GInputStream 的 close_fn 指向
```
- **用途**: 关闭读取流（仅关闭读方向）
- **关键逻辑**:
  1. 通过弱引用获取 agent；如果 agent 已消失，直接返回 TRUE（已经相当于关闭）
  2. 加 agent 锁，调用 `agent_find_component()` 定位 component
  3. 调用 `nice_component_shutdown(component, TRUE, FALSE)` —— `shutdown_read=TRUE, shutdown_write=FALSE`，只关闭 TCP 的读方向

---

**`nice_input_stream_is_readable(GPollableInputStream *stream)`**
```
原型: GPollableInputStream 接口实现
```
- **用途**: 检查是否有数据可读（非阻塞轮询）
- **关键逻辑**:
  1. 如果流已关闭或 agent 已消失，返回 FALSE
  2. 加 agent 锁后查找 component
  3. 如果是 reliable agent，首先检查 pseudo-TCP 输入缓冲区是否有待读取数据（`pseudo_tcp_socket_get_available_bytes(component->tcp) > 0`）
  4. 遍历 `component->socket_sources`，检查各 socket 的 fd 是否有 `G_IO_IN` 条件

---

**`nice_input_stream_read_nonblocking(GPollableInputStream *stream, void *buffer, gsize count, GError **error)`**
```
原型: GPollableInputStream 接口实现
```
- **用途**: 非阻塞读取数据
- **关键逻辑**:
  1. 检查流状态和 agent 有效性
  2. 调用 `nice_agent_recv_nonblocking(agent, stream_id, component_id, buffer, count, NULL, error)` —— 这是非阻塞版本的接收函数

---

**`nice_input_stream_create_source(GPollableInputStream *stream, GCancellable *cancellable)`**
```
原型: GPollableInputStream 接口实现
```
- **用途**: 创建 `GSource` 用于 GMainLoop 的 poll 集成
- **关键逻辑**:
  1. 如果流已关闭或 agent 已消失，创建一个虚拟 `GSource`（仅包含可取消源）
  2. 否则调用 `nice_component_input_source_new()` 创建实际的数据可读源
  3. 如果传入了 `cancellable`，将可取消源作为子源附加到主源上

---

**`streams_removed_cb(NiceAgent *agent, guint *stream_ids, gpointer user_data)`**
```
原型: NiceAgent::streams-removed 信号回调
```
- **用途**: 当底层 stream 被移除时自动关闭 GInputStream
- **关键逻辑**: 遍历 `stream_ids` 数组（以 0 结尾），检查是否包含自己的 `stream_id`，若是则调用 `g_input_stream_close()` 关闭流

---

**头文件 `inputstream.h` 导出**：
- GObject 类型宏（`NICE_TYPE_INPUT_STREAM`、`NICE_INPUT_STREAM(obj)` 等）
- `nice_input_stream_new()` 构造器
- `NiceInputStream` 和 `NiceInputStreamClass` 类型定义（均为空壳，仅继承父类）

---

### outputstream.c/h — NiceOutputStream

**类型定义**：`NiceOutputStream` 继承 `GOutputStream`，并实现 `GPollableOutputStream` 接口。私有结构体 `NiceOutputStreamPrivate` 包含：
- `agent_ref` (`GWeakRef`)：对 `NiceAgent` 的弱引用
- `stream_id` / `component_id`：标识 stream/component
- `closed_cancellable` (`GCancellable *`)：用于在流被移除时取消所有阻塞写入操作

**辅助结构体 `WriteData`**：
```
typedef struct {
  volatile gint ref_count;
  GCond cond;
  GMutex mutex;
  gboolean writable;
  gboolean cancelled;
} WriteData;
```
- 引用计数管理（`write_data_ref` / `write_data_unref`）
- 使用 `GCond` + `GMutex` 组合实现阻塞写入的等待/唤醒机制

---

**`nice_output_stream_new(NiceAgent *agent, guint stream_id, guint component_id)`**
```
原型: NiceOutputStream *nice_output_stream_new(NiceAgent *agent, guint stream_id, guint component_id)
```
- **用途**: 创建 `NiceOutputStream`，包装指定 agent 的 stream/component
- **关键逻辑**: 与 `NiceInputStream` 构造函数对称，参数校验后通过 `g_object_new()` 构造

---

**`nice_output_stream_write(GOutputStream *stream, const void *buffer, gsize count, GCancellable *cancellable, GError **error)`**
```
原型: 虚函数，由 GOutputStream 的 write_fn 指向
```
- **用途**: 同步阻塞写入数据。这是整个 GIO 流封装中最复杂的函数
- **关键逻辑**:
  1. 前置检查：流是否关闭、agent 是否存活
  2. 如果 `count == 0`，直接返回 0
  3. 创建 `WriteData` 结构体并初始化互斥锁和条件变量
  4. 注册取消回调（`write_cancelled_cb`）：当 `cancellable` 或 `closed_cancellable` 被取消时，锁定 mutex 并广播 `cond`，设置 `cancelled = TRUE`
  5. 注册 `"reliable-transport-writable"` 信号回调（`reliable_transport_writeable_cb`）：当伪 TCP 可写时，锁定 mutex 并广播 `cond`，设置 `writable = TRUE`
  6. 进入 do-while 循环，每次迭代：
     - 检查是否被取消
     - 重置 `writable = FALSE`，解锁 mutex
     - 调用 `nice_agent_send()` 发送数据
     - 重新锁定 mutex
     - 如果 `n_sent <= 0`，通过 `g_cond_wait()` 阻塞等待信号或取消
     - 如果 `n_sent > 0`，累加已发送字节数
     - 循环直到全部数据发送完毕
  7. 清理：断开信号连接、取消连接、释放 `WriteData`
  8. 如果没有发送任何数据且不是因为取消，返回错误
  9. **FIXME 注释**：`nice_agent_send()` 是非阻塞的（与 `nice_agent_recv()` 阻塞行为不对称），当前使用 `GCond` 的解决方案是一个不完美的过渡方案

---

**`write_cancelled_cb(GCancellable *cancellable, gpointer user_data)`**
```
原型: GCancellable 回调
```
- **用途**: 当写入操作被取消时唤醒阻塞的 write 调用
- **关键逻辑**: 锁定 `WriteData` 的 mutex，设置 `cancelled = TRUE`，广播 `cond` 唤醒等待线程

---

**`reliable_transport_writeable_cb(NiceAgent *agent, guint stream_id, guint component_id, gpointer user_data)`**
```
原型: NiceAgent::reliable-transport-writable 信号回调
```
- **用途**: 当伪 TCP 连接变为可写时唤醒阻塞的 write 调用
- **关键逻辑**: 锁定 mutex，设置 `writable = TRUE`，广播 `cond`

---

**`nice_output_stream_close(GOutputStream *stream, GCancellable *cancellable, GError **error)`**
```
原型: 虚函数，由 GOutputStream 的 close_fn 指向
```
- **用途**: 关闭写入流（仅关闭写方向）
- **关键逻辑**: 获取 agent，加锁找到 component，调用 `nice_component_shutdown(component, FALSE, TRUE)` —— `shutdown_read=FALSE, shutdown_write=TRUE`

---

**`nice_output_stream_is_writable(GPollableOutputStream *stream)`**
```
原型: GPollableOutputStream 接口实现
```
- **用途**: 检查流是否可写
- **关键逻辑**:
  1. 如果流已关闭或 agent 已消失，返回 FALSE
  2. 通过 `agent_find_component()` 定位 component 和 selected_pair
  3. 如果 `selected_pair.local` 不为空且 socket 不是可靠的（即不可靠 UDP 上的伪 TCP），检查 `pseudo_tcp_socket_can_send(component->tcp)` —— 伪 TCP 输出缓冲区是否有空间
  4. 如果 socket 本身是可靠的（如原生 TCP），直接检查 fd 的 `G_IO_OUT` 条件

---

**`nice_output_stream_write_nonblocking(GPollableOutputStream *stream, const void *buffer, gsize count, GError **error)`**
```
原型: GPollableOutputStream 接口实现
```
- **用途**: 非阻塞写入数据
- **关键逻辑**:
  1. 检查流状态和 agent 有效性
  2. 直接调用 `nice_agent_send(agent, stream_id, component_id, count, buffer)`
  3. 如果返回 -1，设置 `G_IO_ERROR_WOULD_BLOCK` 错误

---

**`nice_output_stream_create_source(GPollableOutputStream *stream, GCancellable *cancellable)`**
```
原型: GPollableOutputStream 接口实现
```
- **用途**: 创建用于 poll 的 `GSource`
- **关键逻辑**:
  1. 创建基础 `g_pollable_source_new()`，如果传入 cancellable，将其添加为子源
  2. 通过 `agent_find_component()` 获取 component
  3. 如果 `component->tcp_writable_cancellable` 存在，将其 cancellable source 作为子源附加（用于伪 TCP 可写通知）

---

**`streams_removed_cb()` (outputstream 版)**：
- 与 inputstream 版本类似，但在关闭输出流之前额外调用 `g_cancellable_cancel(self->priv->closed_cancellable)` 来唤醒所有因 `g_cond_wait()` 而阻塞的 write 操作

---

**头文件 `outputstream.h` 导出**：
- GObject 类型宏及 `nice_output_stream_new()` 构造器
- `NiceOutputStream` 和 `NiceOutputStreamClass` 类型定义

---

### iostream.c/h — NiceIOStream

**类型定义**：`NiceIOStream` 继承 `GIOStream`。私有结构体 `NiceIOStreamPrivate` 包含：
- `agent_ref` (`GWeakRef`)：弱引用 NiceAgent
- `stream_id` / `component_id`：标识 stream/component
- `input_stream` (`GInputStream *`)：持有的 `NiceInputStream`（延迟创建）
- `output_stream` (`GOutputStream *`)：持有的 `NiceOutputStream`（延迟创建）

---

**`nice_io_stream_new(NiceAgent *agent, guint stream_id, guint component_id)`**
```
原型: GIOStream *nice_io_stream_new(NiceAgent *agent, guint stream_id, guint component_id)
```
- **用途**: 创建双向 `NiceIOStream`
- **关键逻辑**:
  1. 参数校验：`stream_id > 0`，`component_id > 0`
  2. 通过 `g_object_new()` 构造，set_property 中存储弱引用并连接 `"streams-removed"` 信号
  3. `input_stream` 和 `output_stream` 此时尚未创建（延迟到首次访问时）

---

**`nice_io_stream_get_input_stream(GIOStream *stream)`**
```
原型: 虚函数，由 GIOStream 的 get_input_stream 指向
```
- **用途**: 获取输入流（延迟创建模式）
- **关键逻辑**:
  1. 如果 `input_stream` 为 NULL（首次调用），通过 `g_weak_ref_get()` 获取 agent
  2. 调用 `nice_input_stream_new(agent, stream_id, component_id)` 创建 `NiceInputStream` 实例
  3. 注意：agent 可能在此期间已被销毁（传入 NULL），`NiceInputStream` 支持 NULL agent 构造
  4. 后续调用直接返回已缓存的 `input_stream`

---

**`nice_io_stream_get_output_stream(GIOStream *stream)`**
```
原型: 虚函数，由 GIOStream 的 get_output_stream 指向
```
- **用途**: 获取输出流（延迟创建模式）
- **关键逻辑**:
  1. 如果 `output_stream` 为 NULL，获取 agent（可能为 NULL）
  2. 通过 `g_object_new(NICE_TYPE_OUTPUT_STREAM, ...)` 直接构造 `NiceOutputStream`（不同于 InputStream 使用 `_new()`）
  3. 后续调用直接返回已缓存的 `output_stream`

---

**`nice_io_stream_dispose(GObject *object)`**
```
原型: GObject dispose 虚函数
```
- **用途**: 清理资源
- **关键逻辑**:
  1. **关键顺序**：先确保 GIOStream 已关闭 —— 因为如果 input/output 流尚未延迟创建，在父类 `g_io_stream_dispose()` 中关闭时会触发延迟创建，但此时 agent 已可能无效导致崩溃
  2. 释放 `input_stream` 和 `output_stream`（unref 并置 NULL）
  3. 断开 `"streams-removed"` 信号，清除弱引用，调用父类 dispose

---

**`streams_removed_cb()` (iostream 版)**：
- 当底层 stream 被移除时，调用 `g_io_stream_close()` 关闭双向流（这会递归关闭内部的 input_stream 和 output_stream）

---

**头文件 `iostream.h` 导出**：
- GObject 类型宏
- `nice_io_stream_new()` 构造器（返回 `GIOStream *`）
- `NiceIOStream` 和 `NiceIOStreamClass` 类型定义
- 引入 `inputstream.h` 和 `outputstream.h` 以支持 getter 返回类型

---

### GIO 异步 I/O 模型集成

由于 `NiceInputStream` 和 `NiceOutputStream` 分别继承 `GInputStream` 和 `GOutputStream`，它们自动继承了基类的异步 I/O 支持。GLib 的 `GInputStream`/`GOutputStream` 基类会自动实现 `read_async()` / `write_async()` 等异步方法，这些方法在内部通过 `GTask` 在 `GMainContext` 线程池中执行同步的 `read_fn` / `write_fn`：

- **读取路径**：`g_input_stream_read_async()` 在内部创建 `GTask`，在线程池中调用 `nice_input_stream_read()`（即 `nice_agent_recv()`），完成后在 GMainContext 中回调用户提供的 `GAsyncReadyCallback`
- **写入路径**：`g_output_stream_write_async()` 同理，创建 `GTask`，在线程池中调用 `nice_output_stream_write()`（循环调用 `nice_agent_send()` 并阻塞等待），完成后回调

**线程安全注意事项**：
- `NiceInputStream` 和 `NiceOutputStream` 均使用 `GWeakRef` 而非强引用持有 agent，所以 agent 可以在任意线程被销毁而不会造成悬空指针
- `nice_agent_recv()` 是线程安全的同步接收函数
- `nice_output_stream_write()` 中的 `GCond`/`GMutex` 同步机制确保 `nice_agent_send()`（本质非阻塞）在 GIO 异步框架中表现为阻塞语义
- `closed_cancellable` 提供了一个额外的安全网：当 stream 被移除时，所有仍在 `g_cond_wait()` 的 write 调用都会被唤醒并返回错误
- `GPollableInputStream` / `GPollableOutputStream` 接口实现使流能够在非阻塞轮询场景中使用（如自定义事件循环集成）

---

## 第十部分：PseudoTCP -- 伪 TCP 可靠传输

### 概述

PseudoTCP 在不可靠的 UDP 之上实现类似 TCP 的可靠传输，用于 ICE reliable 模式，提供有序、无丢失的字节流。其设计灵感来自 libjingle（Google Talk 的 C++ 库）的 "pseudo TCP" 实现，后来被移植到 libnice 并重构为 GObject 类型。PseudoTCP 并非完整的 TCP 协议栈，而是一个精简的子集，包含：三次握手、序列号/确认号管理、拥塞控制（慢启动/拥塞避免/快速重传/快速恢复）、Nagle 算法、滑动窗口流控、FIN/RST 连接关闭等核心机制。

PseudoTCP 的线格式（wire format）使用 24 字节自定义头部（与 TCP 头部不同），在数据部分之前包含：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Conversation Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Acknowledgment Number                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |   |U|A|P|R|S|F|                               |
|    Control    |   |R|C|S|S|Y|I|            Window             |
|               |   |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Timestamp sending                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Timestamp receiving                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Conversation Number**（32 位）：唯一标识一个 PseudoTCP 连接的多路复用 ID。同一对等体之间可以运行多个独立的 PseudoTCP 连接，每个有不同的 conversation number。**Sequence Number**（32 位）：发送方为该段分配的序列号，用于接收方重组和确认。**Acknowledgment Number**（32 位）：对接收到的数据的累积确认。**Flags**（8 位）：目前使用 FLAG_FIN（bit0）、FLAG_CTL（bit1）、FLAG_RST（bit2），分别表示连接终止、控制消息和连接重置。**Window**（16 位）：接收方通告的窗口大小（经窗口缩放因子调整后的值）。**Timestamp sending / receiving**（各 32 位）：用于 RTT 测量的时间戳。

### PseudoTcpSocket 结构体（pseudotcp.h / pseudotcp.c）

#### GObject 类型定义

`PseudoTcpSocket` 是一个 GObject 类型，通过 `G_DEFINE_TYPE` 注册，包含 4 个 GObject 属性：`conversation`（构造时设置）、`callbacks`、`state`（只读）、`ack-delay`、`no-delay`、`rcv-buf`、`snd-buf`、`support-fin-ack`（构造时设置）。

```c
struct _PseudoTcpSocket {
    GObject parent;
    PseudoTcpSocketPrivate *priv;
};
```

注意：私有数据（`PseudoTcpSocketPrivate`）使用手动 `g_new0` 分配而非 `g_object_set_private`，因为 GObject 的 `g_slice_alloc` 无法分配超过 150KB 的私有数据块。

#### 关键私有字段分析

**连接状态**：
- `state`（`PseudoTcpState`）：当前 TCP 状态。完整状态枚举：`TCP_LISTEN`、`TCP_SYN_SENT`、`TCP_SYN_RECEIVED`、`TCP_ESTABLISHED`、`TCP_CLOSED`、`TCP_FIN_WAIT_1`、`TCP_FIN_WAIT_2`、`TCP_CLOSING`、`TCP_TIME_WAIT`、`TCP_CLOSE_WAIT`、`TCP_LAST_ACK`。这 11 个状态精确对应 RFC 793 定义的 TCP 状态机。
- `conv`（`guint32`）：会话 ID（conversation number）。
- `bReadEnable` / `bWriteEnable`：标记是否有可读/可写事件待通知。
- `bOutgoing`：标记上一次流量方向（用于判断是否有 pending 超时）。
- `last_traffic`：最后一次收发流量的时间戳。

**接收方数据**：
- `rlist`（`GList *`）：乱序到达的片段链表（`RSegment`，包含 `seq` 和 `len`），用于重组失序数据。
- `rbuf`（`PseudoTcpFifo`）：接收缓冲区（循环队列），存储已按序到达、待应用读取的数据。
- `rcv_nxt`：下一个期望接收的序列号（即以 `rcv_nxt-1` 为界的所有数据已按序确认）。
- `rcv_wnd`：当前接收窗口大小（动态变化，最大为 `rbuf_len`）。
- `rwnd_scale`：接收窗口缩放因子（0-14）。窗口大小通过 `rcv_wnd >> rwnd_scale` 转换为 16 位线格式值。
- `rcv_fin`：收到的 FIN 段的序列号（用于在失序到达时判断 FIN 是否已可被确认）。

**发送方数据**：
- `slist`（`GQueue`）：待确认的发送段队列（`SSegment`，包含 `seq`、`len`、`xmit` 重传计数、`flags`）。每次收到新的 ACK 时从中移除已确认部分。
- `unsent_slist`（`GQueue`）：尚未发送的段队列（指向 `slist` 中 `xmit == 0` 的段）。
- `sbuf`（`PseudoTcpFifo`）：发送缓冲区（循环队列），存储应用写入但尚未发送的数据。
- `snd_nxt`：下一个要发送的序列号。
- `snd_una`：最旧的未确认序列号（即发送窗口左边界）。
- `snd_wnd`：对端通告的接收窗口（经 `swnd_scale` 缩放还原后的实际值）。
- `swnd_scale`：发送窗口缩放因子（对端通告）。

**拥塞控制参数**：
- `cwnd`（`guint32`）：拥塞窗口（congestion window），限制已发送但未确认的数据量。初始值为 `2 * mss`。
- `ssthresh`（`guint32`）：慢启动阈值（slow start threshold）。cwnd < ssthresh 时执行慢启动，cwnd >= ssthresh 时执行拥塞避免。初始值为 `rbuf_len`。
- `mss`（`guint32`）：最大段大小（maximum segment size），由 MTU 减去协议开销得到。
- `msslevel`：当前 MTU 表索引。初始为 0，通过 `adjustMTU()` 动态调整。
- `largest`：已发送的最大段长度（用于统计/调整 MTU）。

**重传与 RTT 测量**：
- `rto_base`：当前重传定时器的起始时间（每次发送新数据时设置，全部确认后清零）。
- `rx_rto`：当前重传超时（RTO），根据 SRTT 和 RTTVAR 动态计算。
- `rx_srtt`：平滑往返时间（smoothed RTT），RFC 6298 指数加权移动平均。
- `rx_rttvar`：往返时间变化（RTT variance），用于 RTO 的安全裕度计算。

**快速重传/恢复**：
- `dup_acks`（`guint8`）：连续重复 ACK 计数。达到 3 时触发快速重传。
- `recover`：快速恢复期间记录的最大发送序列号（用于 NewReno 判断何时退出恢复）。
- `fast_recovery`：是否处于快速恢复阶段。

**定时器与 ACK 延迟**：
- `t_ack`：延迟 ACK 调度时间（0 表示无调度的 ACK）。
- `ack_delay`：延迟 ACK 超时时间（默认 100ms）。
- `last_acked_ts`：上一个被确认的段的发送时间戳（用于 NewReno 条件判断）。

**其他**：
- `use_nagling`：是否启用 Nagle 算法（默认启用），对应 `"no-delay"` 属性的取反。
- `support_wnd_scale`：是否支持窗口缩放（默认 TRUE，在连接协商时确定）。
- `support_fin_ack`：是否支持 FIN-ACK 扩展（默认 TRUE，在连接协商时确定）。
- `current_time`：显式设置的当前时间（用于单元测试；为 0 时使用 `g_get_monotonic_time()`）。
- `shutdown`（`Shutdown`）：旧版关闭模式（仅当 FIN-ACK 未启用时使用），取值 `SD_NONE`、`SD_GRACEFUL`、`SD_FORCEFUL`。

### 核心函数

#### pseudo_tcp_socket_new() -- 创建/初始化 PseudoTcp 实例

```c
PseudoTcpSocket *pseudo_tcp_socket_new(guint32 conversation, PseudoTcpCallbacks *callbacks);
```

通过 `g_object_new()` 创建 GObject 实例，传入 conversation 和 callbacks。内部的 `pseudo_tcp_socket_init()` 初始化所有字段：
- 接收/发送缓冲区分别初始化为 `DEFAULT_RCV_BUF_SIZE`（60KB）和 `DEFAULT_SND_BUF_SIZE`（90KB）。
- 连接状态为 `PSEUDO_TCP_LISTEN`。
- `cwnd = 2 * mss`（初始拥塞窗口为 2 个段），`ssthresh = rbuf_len`（初始慢启动阈值为接收缓冲区大小）。
- RTO 初始化为 `DEF_RTO`（1000ms）。

`PseudoTcpCallbacks` 结构体包含以下回调：
- `PseudoTcpOpened`：连接建立时调用。
- `PseudoTcpReadable`：有数据可读时调用。
- `PseudoTcpWritable`：有发送缓冲区空间时调用。
- `PseudoTcpClosed`：连接异常终止时调用（附带错误码）。
- `WritePacket`：需要发送数据包时调用（返回 `PseudoTcpWriteResult`），由上层负责将数据写入实际的传输层。

#### pseudo_tcp_socket_connect() -- 主动发起连接

```c
gboolean pseudo_tcp_socket_connect(PseudoTcpSocket *self);
```

仅在 `PSEUDO_TCP_LISTEN` 状态下有效。关键逻辑：
1. 设置状态为 `PSEUDO_TCP_SYN_SENT`。
2. 调用 `queue_connect_message()`：构造控制消息（`CTL_CONNECT`），附带 TCP 选项（窗口缩放因子 `TCP_OPT_WND_SCALE`、FIN-ACK 支持 `TCP_OPT_FIN_ACK`）。
3. 调用 `attempt_send()` 立即尝试发送。连接完成（进入 ESTABLISHED）后由 `process()` 中的状态机触发 `PseudoTcpOpened` 回调。

#### pseudo_tcp_socket_listen() -- 进入监听状态

`PseudoTcpSocket` 创建后默认即为 `PSEUDO_TCP_LISTEN` 状态，无需显式调用 listen。服务端通过接收对端的 SYN（`FLAG_CTL` + `CTL_CONNECT`）完成被动打开。

#### pseudo_tcp_socket_send() -- 发送数据

```c
gint pseudo_tcp_socket_send(PseudoTcpSocket *self, const char *buffer, guint32 len);
```

关键逻辑：
1. 状态检查：必须在 `PSEUDO_TCP_ESTABLISHED` 状态。如果已发送 FIN，返回 `EPIPE`；否则返回 `ENOTCONN`。
2. 检查发送缓冲区可用空间：若无空间，设置 `bWriteEnable = TRUE` 并返回 `EWOULDBLOCK`。
3. 调用 `queue()`：将数据写入 `sbuf` FIFO（可能截断至可用空间），并创建/追加 `SSegment` 到 `slist` 和 `unsent_slist`。如果最近一个段尚未发送且类型相同，则合并数据（避免过小段）。
4. 调用 `attempt_send()` 尝试立即发送。
5. 如果写入量小于请求量，设置 `bWriteEnable = TRUE` 以在后续空间可用时触发 `PseudoTcpWritable` 回调。

**MSS 分段逻辑**：`transmit()` 函数中，每次发送最多 `min(segment->len, priv->mss)` 字节。如果 `nTransmit < segment->len`，则将段拆分为两个子段：已发送部分和剩余部分。发送完成后 `snd_nxt += segment->len`，重传计数 `xmit += 1`，并设置 `rto_base`（如果尚未设置）。

**cwnd 限制与发送窗口**：`attempt_send()` 中计算：
- `nWindow = min(priv->snd_wnd, cwnd)`——实际发送窗口取对端通告窗口和拥塞窗口的较小值。
- `nInFlight = priv->snd_nxt - priv->snd_una`——已发送但未确认的数据量。
- `nUseable = (nInFlight < nWindow) ? (nWindow - nInFlight) : 0`——可用发送量。
- Limited Transmit（RFC 3042）：在 `dup_acks == 1 || dup_acks == 2` 时临时扩大 cwnd 以允许发送新数据来触发更多 ACK。
- Silly Window Syndrome（SWS）避免（RFC 813）：如果 `nUseable * 4 < nWindow`，则不发送（窗口太小不值得发送）。

**Nagle 算法**：如果 `use_nagling == TRUE` 且 `nAvailable < mss` 且有在途数据（`snd_nxt > snd_una`），则暂不发送——等待更多数据积累或 ACK 到达。

#### pseudo_tcp_socket_recv() -- 接收数据

```c
gint pseudo_tcp_socket_recv(PseudoTcpSocket *self, char *buffer, size_t len);
```

关键逻辑：
1. 如果已收到对端 FIN（`shutdown_reads == TRUE`），返回 0（表示 EOF，RFC 793 第 3.5 节 Case 2）。
2. 如果 FIN-ACK 未支持且连接已关闭，返回 0。
3. 如果 FIN-ACK 未支持且不在 ESTABLISHED 状态，返回 `ENOTCONN`。
4. 从 `rbuf` FIFO 读取数据。如果无数据且未收到 FIN，设置 `bReadEnable = TRUE` 并返回 `EWOULDBLOCK`。
5. **窗口更新**：读取后检查接收缓冲区是否恢复足够空间。如果 `available_space - rcv_wnd >= min(rbuf_len / 2, mss)`（即缓冲区空出至少半个缓冲区或一个 MSS），更新 `rcv_wnd`；如果之前窗口为 0（`bWasClosed`），立即发送 ACK 以通知对端窗口已打开。

#### pseudo_tcp_socket_notify_message() -- 处理收到的伪 TCP 消息

```c
gboolean pseudo_tcp_socket_notify_message(PseudoTcpSocket *self, NiceInputMessage *message);
```

这是 `notify_packet` 的优化版本，使用 `NiceInputMessage` 的双缓冲区结构（第一个缓冲区 24 字节头部，第二个缓冲区为数据负荷），避免数据拷贝。

关键逻辑：
1. 如果 message 只有一个缓冲区，回退到 `notify_packet()`。
2. 验证头部缓冲区大小 == `HEADER_SIZE`（24 字节）。
3. 检查包长度合法性（`MAX_PACKET` / `HEADER_SIZE` 边界）。
4. 调用 `parse()` 解析头部字段（conversation、seq、ack、flags、window、tsval、tsecr）。
5. 调用 `process()` 处理段，这是核心状态机和拥塞控制的入口。

**process() 函数的核心逻辑**：

1. **会话校验**：`seg->conv != priv->conv` 则返回 FALSE。
2. **CLOSED 状态处理**：如果已关闭且收到数据段，发送 RST（RFC 1122 第 4.2.2.13 节）。
3. **RST 处理**：收到 RST 则 `closedown(ECONNRESET, CLOSEDOWN_REMOTE)`。
4. **控制消息处理**：`FLAG_CTL` + `CTL_CONNECT` 为 SYN。在 LISTEN 状态收到 SYN 则进入 `SYN_RECEIVED` 并回复 SYN。在 SYN_SENT 状态收到 SYN 则直接进入 `ESTABLISHED`。
5. **RTT 计算**：如果收到的 ACK 带有有效的时间戳回复（`tsecr != 0`）：
   - `rtt = now - seg->tsecr`
   - 首次测量：`srtt = rtt`，`rttvar = rtt / 2`
   - 后续测量：`rttvar = (3 * rttvar + |rtt - srtt|) / 4`，`srtt = (7 * srtt + rtt) / 8`（RFC 6298 指数加权移动平均）。
   - `rto = bound(MIN_RTO, srtt + max(1, 4 * rttvar), MAX_RTO)`——RTO 下限 1000ms，上限 60000ms。
6. **ACK 处理**：
   - 有价值的 ACK（`seg->ack > snd_una && seg->ack <= snd_nxt`）：更新 `snd_una`、清理已确认的发送段、推进发送 FIFO 读取指针。
   - 检测 FIN-ACK（`nAcked == sbuf.data_length + 1` 且已发送 FIN）。
7. **拥塞控制更新**（详见下文拥塞控制章节）。
8. **FIN-ACK 状态机**：根据收到的 FIN/ACK 和当前状态执行 RFC 793 第 3.5 节定义的状态转换。
9. **接收数据插入**：将数据段写入 `rbuf` FIFO。如果段是顺序到达的（`seg->seq == rcv_nxt`），则推进 `rcv_nxt` 并重组 `rlist` 中所有可以连续拼接的乱序段。如果是乱序的，则创建 `RSegment` 并按序列号序插入 `rlist`。
10. **Readable 通知**：如果有新数据到达且 `bReadEnable == TRUE`，调用 `PseudoTcpReadable` 回调。
11. **Writable 通知**：如果发送缓冲区有足够空间，调用 `PseudoTcpWritable` 回调。

#### pseudo_tcp_socket_notify_clock() -- 定时器回调

```c
void pseudo_tcp_socket_notify_clock(PseudoTcpSocket *self);
```

定时器回调处理，由上层根据 `pseudo_tcp_socket_get_next_clock()` 返回的超时时间调度。关键逻辑：

1. **TIME-WAIT 状态**：收到时钟通知后直接关闭连接（`set_state_closed`）。
2. **LAST-ACK 状态**：重发 FIN（因为前一个 FIN 未被 ACK）。
3. **重传检查**：如果 `rto_base` 已设置且当前时间超过 `rto_base + rx_rto`：
   - 重传 `slist` 队首段（调用 `transmit()`）。
   - **指数退避**：`ssthresh = max(nInFlight / 2, 2 * mss)`，`cwnd = mss`，`rx_rto *= 2`（上限为 `MAX_RTO` 或连接建立前的 `DEF_RTO`）。
   - `recover = snd_nxt`。
   - 如果 `dup_acks >= 3`，退出快速恢复。
4. **零窗口探测**：如果 `snd_wnd == 0` 且超过 RTO 时间未收到新数据：
   - 如果距离上次收到数据超过 15000ms，认为连接已死，`closedown(ECONNABORTED)`。
   - 否则发送窗口探测包（`packet(snd_nxt - 1, ...)`），并指数退避 RTO。
5. **延迟 ACK 发送**：如果 `t_ack` 已设置且超过 `t_ack + ack_delay`，立即发送纯 ACK 包。

**RTO 计算总结**：`rx_rto = max(MIN_RTO(1000ms), SRTT + max(1, 4*RTTVAR))`，上限 `MAX_RTO(60000ms)`。

#### pseudo_tcp_socket_close() -- 连接关闭

```c
void pseudo_tcp_socket_close(PseudoTcpSocket *self, gboolean force);
```

两种关闭模式：

- **强制关闭**（`force = TRUE`）：立即调用 `closedown(self, ECONNABORTED, CLOSEDOWN_LOCAL)`，发送 RST 段并直接进入 CLOSED 状态（RFC 1122 第 4.2.2.13 节），丢弃所有待发送数据。
- **优雅关闭**（`force = FALSE`）：委托给 `pseudo_tcp_socket_shutdown(self, RDWR)`。

**pseudo_tcp_socket_shutdown()** 根据 FIN-ACK 支持情况采用不同路径：
- **FIN-ACK 不启用**：设置 `shutdown = SD_GRACEFUL`，由 `get_next_clock` 中的定时器逻辑驱动关闭。
- **FIN-ACK 启用**：
  - `SHUTDOWN_RD`：设置 `shutdown_reads = TRUE`，阻止后续读取。
  - `SHUTDOWN_WR` / `SHUTDOWN_RDWR`：
    - `ESTABLISHED` 状态：如果有未读接收数据，发 RST 并 `closedown`（RFC 1122 第 4.2.2.13 节）；否则发送 FIN，进入 `FIN_WAIT_1`（RFC 793 第 3.5 节 Case 1）。
    - `CLOSE_WAIT` 状态：发送 FIN，进入 `LAST_ACK`（RFC 793 第 3.5 节 Case 2）。

### 拥塞控制

PseudoTCP 实现了 TCP 拥塞控制的完整子集，包含以下四个核心阶段：

**慢启动（Slow Start）**：连接建立或超时重传后，`cwnd` 初始化为 `1 * MSS`（实际实现是 `mss`）。每收到一个新的 ACK，`cwnd += mss`，使拥塞窗口呈指数级增长（每 RTT 翻倍），直到 `cwnd >= ssthresh`。

**拥塞避免（Congestion Avoidance）**：当 `cwnd >= ssthresh` 时，每收到一个 ACK，`cwnd += max(1, mss * mss / cwnd)`，使拥塞窗口线性增长（每 RTT 增加约一个 MSS）。

**快速重传（Fast Retransmit）**：当收到 3 个连续的重复 ACK（`dup_acks == 3`）时，立即重传 `slist` 队首段，不等待超时。触发条件还包括 `snd_una >= recover` 或 `tsecr == last_acked_ts`（NewReno 条件，RFC 3782 第 3 节步骤 1A）。进入快速重传时设置：
- `recover = snd_nxt`
- `ssthresh = max(nInFlight / 2, 2 * mss)`
- `cwnd = ssthresh + 3 * mss`
- `fast_recovery = TRUE`

**快速恢复（Fast Recovery）**：在 `fast_recovery == TRUE` 期间：
- 每收到一个重复 ACK（`dup_acks > 3`）：`cwnd += mss`（允许发送新数据）。
- 收到恢复 ACK（`snd_una >= recover`）：`cwnd = min(ssthresh, max(nInFlight, mss) + mss)`，退出恢复，`dup_acks = 0`。
- 收到部分 ACK（`snd_una < recover`）：使用 NewReno 的重传策略（`transmit()` 重传队首段），`cwnd += (nAcked > mss ? mss : 0) - min(nAcked, cwnd)`。

**超时重传**：当重传定时器触发时：`ssthresh = max(nInFlight / 2, 2 * mss)`，`cwnd = mss`（退回到慢启动），`rx_rto *= 2`（指数退避），重置 `rto_base = now`。

流程图（简单版）：

```
          超时
cwnd=mss  <─── [任意阶段]
   │
   ▼
[慢启动] ── cwnd >= ssthresh ──▶ [拥塞避免]
   │                                 │
   │ 3 dup ACKs                      │ 3 dup ACKs
   ▼                                 ▼
[快速重传] ◀──────────────── [快速重传]
   │                                 │
   └── snd_una >= recover ──▶ 退出恢复
```

### 状态机

PseudoTCP 实现了完整的 TCP 连接状态机（RFC 793 第 23 页 + RFC 1122 第 4.2.2.8 节），共 11 个状态，所有状态转换通过 `set_state()` 函数以 `g_assert` 强制校验合法性。

```
                         主动打开                       被动打开
                            │                              │
                            │  send SYN                    │  rcv SYN
                            ▼                              ▼
CLOSED ──────────────▶ SYN_SENT                    LISTEN ─────▶ SYN_RCVD
                            │                              │          │
                            │  rcv SYN+ACK                 │  rcv ACK │
                            ▼                              ▼          │
                       ┌─ ESTABLISHED ◀───────────────────┘          │
                       │    │     │                                   │
                       │    │     │ rcv FIN                           │
      close() / send FIN    │     ▼                                   │
                       │    │  CLOSE_WAIT ── close() / send FIN       │
                       ▼    │     │              ▼                    │
                    FIN_WAIT_1 │  LAST_ACK ── rcv ACK ──▶ CLOSED      │
                       │    │                                        │
              rcv ACK  │    │                                        │
                       ▼    │                                        │
                    FIN_WAIT_2                                        │
                       │    │                                        │
                       │    │ rcv FIN                                │
                       ▼    ▼                                        │
                    TIME_WAIT ──── 定时器 ────▶ CLOSED               │
                       ▲                                              │
                       │    ▲                                         │
                    CLOSING ── rcv FIN (同时关闭)                      │
```

**被动打开的路径**：`CLOSED → LISTEN → SYN_RCVD → ESTABLISHED`。在 LISTEN 状态收到 SYN（FLAG_CTL + CTL_CONNECT）时，设置 `state = SYN_RCVD` 并回复 SYN。收到对端的 ACK 后进入 ESTABLISHED。

**主动打开的路径**：`CLOSED → SYN_SENT → ESTABLISHED`。在 SYN_SENT 状态收到 SYN（同时打开的情况），直接进入 ESTABLISHED。更常见的路径是收到对方的 SYN-ACK（实际上是 ACK + SYN）。

**优雅关闭**（启动方）：`ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`。发送 FIN 后进入 FIN_WAIT_1，收到 ACK 进入 FIN_WAIT_2，收到对端 FIN 后进入 TIME_WAIT，超时后进入 CLOSED。

**优雅关闭**（被动方）：`ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED`。收到对端 FIN 后进入 CLOSE_WAIT（通知应用 EOF），应用调用 close/shutdown 后发送 FIN 并进入 LAST_ACK，收到 ACK 后进入 CLOSED。

**同时关闭**：`ESTABLISHED → FIN_WAIT_1 → CLOSING → TIME_WAIT → CLOSED`。

**异常关闭**：任意非 CLOSED 状态收到 RST 段，或本地 `closedown()` 被调用，通过 `closedown()` 函数导航状态机（LISTEN/SYN_SENT 直接关；SYN_RCVD/ESTABLISHED 经 FIN_WAIT_1/FIN_WAIT_2；CLOSE_WAIT 经 LAST_ACK；TIME_WAIT/LAST_ACK 直接到 CLOSED）。

**TIME-WAIT 超时**：在 PseudoTCP 实现中，TIME_WAIT_TIMEOUT 被设置为 1ms（基本不存在），因为底层可以通过 conversation number 保证延迟段不会影响后续连接（RFC 1122 第 4.2.2.13 节的注释：`Since we can control the underlying layer's channel ID, we can guarantee delayed segments won't affect subsequent connections`）。

### PseudoTcpFifo -- 循环队列缓冲区

PseudoTCP 使用自定义的 `PseudoTcpFifo` 结构实现发送和接收缓冲区的循环队列，类似于 libjingle 中的 `FifoBuffer`。

核心操作：
- `pseudo_tcp_fifo_init(size)` / `pseudo_tcp_fifo_clear()`：分配/释放缓冲区（使用 `g_slice_alloc`/`g_slice_free1`）。
- `pseudo_tcp_fifo_read(buf, bytes)`：从 `read_position` 读取数据，自动处理回绕。
- `pseudo_tcp_fifo_write(buf, bytes)`：从写入位置写数据，自动处理回绕。
- `pseudo_tcp_fifo_read_offset(buf, bytes, offset)`：从指定偏移读取（不移动读指针），用于 `packet()` 读取即将发送的数据。
- `pseudo_tcp_fifo_write_offset(buf, bytes, offset)`：从指定偏移写入（不更新 `data_length`），用于 `process()` 处理乱序到达的数据段。
- `pseudo_tcp_fifo_set_capacity(size)`：动态调整缓冲区大小（用于调整 `rcv-buf` / `snd-buf` 属性）。如果缩小容量不足以容纳已有数据则返回 FALSE。
- `pseudo_tcp_fifo_consume_read_data(size)`：消费已确认的数据（移动读指针而不实际拷贝），用于 `process()` 中收到 ACK 后清理发送缓冲区。
- `pseudo_tcp_fifo_consume_write_buffer(size)`：增长已写入数据计数（用于 `process()` 中处理乱序到达后重组连续段）。

### 与 ICE Reliable 模式的集成

PseudoTCP 通过 Component 层与 NiceSocket 接口集成：

1. **创建阶段**：`nice_agent_add_stream()` 中，如果 agent 处于 reliable 模式（ICE 可靠模式，由 `NiceAgentOption` 的 `NICE_AGENT_OPTION_RELIABLE` 标志控制），为每个 component 调用 `pseudo_tcp_socket_new()` 创建 PseudoTcpSocket 实例，存储于 `component->tcp`。

2. **数据发送路径**（TCP over UDP）：
   ```
   nice_agent_send()
     └─ reliable: 写入 component->tcp_sbuf (PseudoTcpFifo)
           └─ PseudoTcpSocket 的 attempt_send()
                 └─ 调用 WritePacket 回调 → component 的 socket 发送 UDP 包
   ```
   数据先写入 PseudoTCP 的发送缓冲区，由 PseudoTCP 的发送逻辑进行分段（按 MSS）、拥塞控制、Nagle 算法处理后，通过 `WritePacket` 回调将 PseudoTCP 格式的数据包交给底层 UDP socket 发送。

3. **数据接收路径**：
   ```
   component 的 UDP socket 收到数据
     └─ pseudo_tcp_socket_notify_message() / notify_packet()
           └─ parse() + process()
                 ├─ 数据重组至 rbuf
                 ├─ ACK 生成
                 └─ PseudoTcpReadable 回调 → 通知应用可读
   ```
   底层 UDP 数据包先经过 PseudoTCP 的头部解析、序列号校验、乱序重组、ACK 生成等处理，重组为有序字节流后才通知应用读取。

4. **定时器集成**：`pseudo_tcp_socket_get_next_clock()` 返回下一次需要触发 `notify_clock()` 的时间，ICE agent 的主循环定时器（GLib 定时器）负责调度这个超时。`notify_clock()` 内部处理重传、零窗口探测、延迟 ACK 发送、TIME-WAIT 超时等定时事件。

5. **关闭集成**：`nice_component_close()` 中调用 `pseudo_tcp_socket_close()` 关闭 PseudoTCP 连接，完成 FIN 握手。

---

## agent/ 模块总结

### 文件清单与代码量

| 子模块 | 文件 | 行数 | 职责 |
|--------|------|------|------|
| core | agent.c/h/priv.h | ~10150 | GObject 类型 + 公共 API + 信号 |
| component | component.c/h | ~2169 | 组件状态管理 |
| conncheck | conncheck.c/h | ~5169 | 连接检查状态机 |
| discovery | discovery.c/h | ~1652 | 候选发现 |
| stream | stream.c/h | ~340 | 媒体流管理 |
| candidate | candidate.c/h/priv.h | ~920 | 候选结构 |
| address | address.c/h | ~786 | 网络地址封装 |
| interfaces | interfaces.c/h | ~1027 | 网络接口枚举 |
| I/O | inputstream/outputstream/iostream | ~1507 | GIO 流封装 |
| pseudotcp | pseudotcp.c/h | ~3245 | 伪 TCP |
| debug | debug.c/h | ~302 | 调试日志 |
| **总计** | **~26 files** | **~27267** | |

### 核心设计模式
1. GObject 事件驱动：ICE 状态机完全由 GLib 主循环驱动
2. 分层解耦：agent -> conncheck/discovery -> socket -> stun
3. 信号/回调分离：应用通过 g_signal_connect 或 nice_agent_attach_recv 接收事件
4. 两种接收模式：回调（事件驱动）vs 阻塞（线程安全）
