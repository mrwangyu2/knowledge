# 01 -- 入口：从例子开始

## simple-example.c (439 行)

### 概述

`simple-example.c` 是 libnice 的入门示例，演示了两个端点之间通过 ICE 协议建立 UDP 连接并交换数据的完整过程。程序在命令行下运行，通过标准输入/输出来交换 ICE 候选信息（不借助任何信令服务器），连接建立后可在两个进程之间发送文本消息。

两个实例分别以 "controlling"（控制方，参数 0）和 "controlled"（被控方，参数 1）角色运行，这是 ICE 协议中决定候选对优先级的一种机制。

### 调用链分析

按 `main()` 中的调用顺序，逐个分析：

#### 1. nice_agent_new()

- **原型**: `NiceAgent * nice_agent_new (GMainContext *ctx, NiceCompatibility compat);`
- **作用**: 创建一个 NiceAgent 实例，这是所有 ICE 操作的入口对象。
- **参数说明**:
  - `ctx`: GLib 主循环上下文，所有定时器（连接检查、keepalive 等）都在此上下文中调度。示例传入 `g_main_loop_get_context(gloop)`，即 gloop 的上下文。传入 `NULL` 则使用全局默认上下文。
  - `compat`: ICE 兼容模式，示例使用 `NICE_COMPATIBILITY_RFC5245`，即标准 RFC 5245 (ICE-UDP) + RFC 6544 (ICE-TCP) 兼容模式。
- **返回值**: 新创建的 NiceAgent GObject，失败返回 NULL。使用后需通过 `g_object_unref()` 释放。

#### 2. g_object_set() -- GObject 属性

示例通过 GObject 属性系统（而非函数调用）设置了三个关键属性：

- **`"stun-server"`**: 设置 STUN 服务器地址（IPv4 字符串）。这个 STUN 服务器用于 Server Reflexive 候选的发现。如果不设置，代理将不会进行 STUN 绑定请求。
- **`"stun-server-port"`**: STUN 服务器端口，默认为 3478（STUN 标准端口）。
- **`"controlling-mode"`**: 布尔值，TRUE 表示本端是 ICE 控制方（controlling），FALSE 表示被控方（controlled）。在 ICE 协议中，这决定了有冲突时哪一方具有最终决定权。Controlling 方通过发送 `USE-CANDIDATE` 属性来提名候选对。

#### 3. g_signal_connect() -- 信号连接

示例连接了三个关键的 NiceAgent 信号，这些信号是事件驱动编程的核心：

- **`"candidate-gathering-done"`**: 候选收集完成时触发。对应的回调 `cb_candidate_gathering_done` 负责打印本地候选信息供对端使用。
- **`"new-selected-pair"`**: 当 ICE 选择一个新的候选对进行连接检查时触发。回调 `cb_new_selected_pair` 仅打印调试信息。
- **`"component-state-changed"`**: 组件的 ICE 状态发生改变时触发。回调 `cb_component_state_changed` 是最核心的回调，负责在状态变为 CONNECTED 时启动数据交换，在状态变为 FAILED 时退出主循环。

#### 4. nice_agent_add_stream()

- **原型**: `guint nice_agent_add_stream (NiceAgent *agent, guint n_components);`
- **作用**: 向代理中添加一个数据流（stream），并指定该流包含的组件数量。
- **参数说明**:
  - `agent`: NiceAgent 实例。
  - `n_components`: 组件数量。值为 1 表示只有一个 RTP 组件（component_id = 1）。常见的 WebRTC 用法会传 2，分别对应 RTP 和 RTCP 两个组件。
- **返回值**: 新创建的 stream ID（正整数），失败返回 0。stream_id 在后续所有操作中用于标识该流。

#### 5. nice_agent_attach_recv()

- **原型**: `gboolean nice_agent_attach_recv (NiceAgent *agent, guint stream_id, guint component_id, GMainContext *ctx, NiceAgentRecvFunc func, gpointer data);`
- **作用**: 将组件的底层套接字挂载到 GLib 主循环上下文中，以便在数据到达时触发回调。**这是 ICE 连接建立的关键前提** -- 没有这个调用，代理无法接收 STUN 包，候选收集和连接检查将无法完成。
- **参数说明**:
  - `stream_id`: 流 ID。
  - `component_id`: 组件 ID，示例中为 1（即唯一的组件）。
  - `ctx`: 用于监听 I/O 的 GLib 主循环上下文，示例传入 gloop 的上下文。
  - `func`: 数据到达时的回调函数，类型为 `NiceAgentRecvFunc`（见下文回调分析）。注意：STUN 包不会传递给此回调，代理内部会自动处理。
  - `data`: 传递给回调的用户数据，示例传 NULL。
- **返回值**: TRUE 表示成功，FALSE 表示 stream/component ID 无效。

#### 6. nice_agent_gather_candidates()

- **原型**: `gboolean nice_agent_gather_candidates (NiceAgent *agent, guint stream_id);`
- **作用**: 启动本地候选地址收集过程。代理会在所有可用网络接口上分配端口、创建套接字，并向已配置的 STUN 服务器发送绑定请求以获取服务器反射候选。收集完成后发出 `"candidate-gathering-done"` 信号。
- **参数说明**:
  - `agent`: NiceAgent 实例。
  - `stream_id`: 要收集候选的流 ID。
- **返回值**: FALSE 表示 stream ID 无效或无法在请求的接口/端口上分配 host 候选；TRUE 表示成功启动。
- **注意**: 如果没有预先调用 `nice_agent_add_local_address()` 添加地址，代理会自动检测本地 IP 地址（通过 `nice_interfaces_get_local_ips()`）。

#### 7. g_main_loop_run()

- **原型**: `void g_main_loop_run (GMainLoop *loop);`
- **作用**: 运行 GLib 主循环。此时程序进入事件驱动模式，所有后续操作都在信号回调中异步完成。主循环会一直运行直到 `g_main_loop_quit()` 被调用。

以下 API 不在 `main()` 中直接调用，而是在回调中调用：

#### 8. nice_agent_get_local_credentials()

- **原型**: `gboolean nice_agent_get_local_credentials (NiceAgent *agent, guint stream_id, gchar **ufrag, gchar **pwd);`
- **作用**: 获取本地 ICE 凭据（username fragment 和 password），用于随本地候选一起发送给对端。
- **参数说明**:
  - `ufrag`: (out) 调用方分配的字符串指针，用于接收 ICE username fragment。使用后需 `g_free()` 释放。
  - `pwd`: (out) 同上，用于接收 ICE password。
- **返回值**: TRUE 成功，FALSE 失败。
- **调用位置**: 在 `print_local_data()` 中 -- `cb_candidate_gathering_done` 回调内。

#### 9. nice_agent_get_local_candidates()

- **原型**: `GSList * nice_agent_get_local_candidates (NiceAgent *agent, guint stream_id, guint component_id);`
- **作用**: 获取指定流组件的所有本地候选地址列表。
- **参数说明**:
  - `stream_id`: 流 ID。
  - `component_id`: 组件 ID。
- **返回值**: (transfer full) 包含 `NiceCandidate` 元素的 `GSList`，调用者拥有返回值及列表元素的所有权。使用后需用 `g_slist_free_full()` 配合 `nice_candidate_free` 释放。
- **调用位置**: 在 `print_local_data()` 中。

#### 10. nice_agent_set_remote_credentials()

- **原型**: `gboolean nice_agent_set_remote_credentials (NiceAgent *agent, guint stream_id, const gchar *ufrag, const gchar *pwd);`
- **作用**: 设置对端 ICE 凭据，用于验证对端的 STUN 消息。
- **参数说明**:
  - `ufrag`: 对端的 ICE username fragment（长度 22-256 字符）。
  - `pwd`: 对端的 ICE password（长度 4-256 字符）。
- **返回值**: TRUE 成功，FALSE 失败。
- **调用位置**: 在 `parse_remote_data()` 中 -- `stdin_remote_info_cb` 回调内。

#### 11. nice_agent_set_remote_candidates()

- **原型**: `int nice_agent_set_remote_candidates (NiceAgent *agent, guint stream_id, guint component_id, const GSList *candidates);`
- **作用**: 设置对端的候选地址列表。**此调用会触发 ICE 连接检查的启动**。
- **参数说明**:
  - `stream_id`: 流 ID。
  - `component_id`: 组件 ID。
  - `candidates`: (transfer none) 包含 `NiceCandidate` 元素的 `GSList`。代理不会取得元素所有权，调用者仍需负责释放。
- **返回值**: 成功添加的候选数量，负数表示错误。
- **注意**: 注释明确指出 "this will trigger the start of negotiation"。

#### 12. nice_agent_send()

- **原型**: `gint nice_agent_send (NiceAgent *agent, guint stream_id, guint component_id, guint len, const gchar *buf);`
- **作用**: 通过指定流组件发送数据载荷。
- **参数说明**:
  - `stream_id`: 流 ID。
  - `component_id`: 组件 ID。
  - `len`: 发送字节数。
  - `buf`: 数据缓冲区。
- **返回值**: 发送的字节数，负数表示错误。在 reliable 模式下，-1 表示未连接或发送缓冲区满（等价于 EWOULDBLOCK）。
- **前置条件**: 组件状态必须为 `NICE_COMPONENT_STATE_READY`。示例实际上在 CONNECTED 状态下就开始发送（通过 `stdin_send_data_cb`），这是可行的，因为 CONNECTED 表示至少有一个可用的候选对。
- **调用位置**: 在 `stdin_send_data_cb()` 中。

#### 13. nice_agent_get_selected_pair()

- **原型**: `gboolean nice_agent_get_selected_pair (NiceAgent *agent, guint stream_id, guint component_id, NiceCandidate **local, NiceCandidate **remote);`
- **作用**: 获取当前选中的候选对（用于实际媒体传输的本地和远端候选）。
- **参数说明**:
  - `local`: (out) 指向本地候选的指针。
  - `remote`: (out) 指向远端候选的指针。
- **返回值**: TRUE 表示存在已选中的候选对，FALSE 表示无。
- **调用位置**: 在 `cb_component_state_changed()` 中，状态变为 CONNECTED 时调用，用于打印连接详情。

### 回调函数分析

#### cb_candidate_gathering_done()

- **原型**: `static void cb_candidate_gathering_done(NiceAgent *agent, guint _stream_id, gpointer data);`
- **触发时机**: 当指定 stream 的本地候选收集完成时，由 NiceAgent 内部发出 `"candidate-gathering-done"` 信号触发。
- **作用**:
  1. 调用 `print_local_data()` 打印本地 ufrag、password 以及所有候选（格式化输出到 stdout）。
  2. 提示用户拷贝此信息到对端。
  3. 添加对 stdin 的监听 (`stdin_remote_info_cb`)，等待用户输入对端的候选数据。

#### cb_component_state_changed()

- **原型**: `static void cb_component_state_changed(NiceAgent *agent, guint _stream_id, guint component_id, guint state, gpointer data);`
- **触发时机**: 组件的 ICE 状态发生变化时由 `"component-state-changed"` 信号触发。可能的状态转换路径：
  - `DISCONNECTED` -> `GATHERING` （开始收集候选）
  - `GATHERING` -> `CONNECTING` （开始连接检查）
  - `CONNECTING` -> `CONNECTED` （至少一个候选对可用）
  - `CONNECTED` -> `READY` （ICE 协商完成，选定的候选对最终确定）
  - 任何状态 -> `FAILED` （连接检查完成但未建立连接）
- **作用**:
  - 当状态变为 `NICE_COMPONENT_STATE_CONNECTED` 时：
    1. 通过 `nice_agent_get_selected_pair()` 获取当前选中的候选对。
    2. 打印本地和远端的 IP:port 信息。
    3. 添加对 stdin 的监听 (`stdin_send_data_cb`)，让用户可以输入文本发送给对端。
  - 当状态变为 `NICE_COMPONENT_STATE_FAILED` 时：
    1. 调用 `g_main_loop_quit(gloop)` 退出主循环，程序终止。

#### cb_nice_recv()

- **原型**: `static void cb_nice_recv(NiceAgent *agent, guint _stream_id, guint component_id, guint len, gchar *buf, gpointer data);`
- **触发时机**: 当组件收到非 STUN 的用户数据时由 `nice_agent_attach_recv()` 注册的回调触发。STUN 包由代理内部处理，不会到此回调。
- **作用**:
  1. 检查是否收到的是一个 `\0` 字节（1 字节）。如果是，调用 `g_main_loop_quit()` 退出（对端 Ctrl-D 信号）。
  2. 否则将接收到的数据打印到 stdout。

#### cb_new_selected_pair()

- **原型**: `static void cb_new_selected_pair(NiceAgent *agent, guint _stream_id, guint component_id, gchar *lfoundation, gchar *rfoundation, gpointer data);`
- **触发时机**: ICE 代理在连接检查过程中选择新的候选对时。
- **作用**: 仅打印调试信息（本地 foundation 和远端 foundation），便于追踪 ICE 候选对的选择过程。

#### stdin_remote_info_cb()

- **原型**: `static gboolean stdin_remote_info_cb(GIOChannel *source, GIOCondition cond, gpointer data);`
- **触发时机**: 由 `g_io_add_watch()` 注册，当 stdin 有数据可读时触发。在 `cb_candidate_gathering_done` 中首次注册。
- **作用**:
  1. 从 stdin 读取一行文本。
  2. 调用 `parse_remote_data()` 解析该行。
  3. 解析成功则返回 FALSE 以停止 stdin 监听，否则提示用户重新输入。
- **返回值**: TRUE 表示继续监听，FALSE 表示移除监听。

#### stdin_send_data_cb()

- **原型**: `static gboolean stdin_send_data_cb(GIOChannel *source, GIOCondition cond, gpointer data);`
- **触发时机**: 由 `g_io_add_watch()` 注册，当 stdin 有数据可读时。在 `cb_component_state_changed`（状态变为 CONNECTED）中注册。
- **作用**:
  1. 从 stdin 读取一行文本。
  2. 调用 `nice_agent_send()` 将文本发送给对端。
  3. 如果读取失败（如 Ctrl-D），发送一个 `\0` 字节表示结束，然后调用 `g_main_loop_quit()` 退出。
- **返回值**: TRUE 继续监听。

### 辅助函数分析

#### print_local_data()

- **作用**: 获取本地 ICE 凭据和候选列表，并按固定格式打印：
  ```
  <ufrag> <pwd> <foundation>,<priority>,<addr>,<port>,<type> ...
  ```
  每行包含 ufrag、password 以及用空格分隔的多个候选序列化字符串。

#### parse_remote_data()

- **作用**: 解析对端的候选数据行（反序列化 `print_local_data()` 的输出），依次调用：
  1. `nice_agent_set_remote_credentials()` 设置远端凭据。
  2. `nice_agent_set_remote_candidates()` 设置远端候选（此调用触发连接检查）。

#### parse_candidate()

- **作用**: 将单个候选的序列化字符串解析为 `NiceCandidate` 对象。格式为：
  ```
  <foundation>,<priority>,<addr>,<port>,<type>
  ```
  字段间用逗号分隔，type 为 `host`、`srflx`、`prflx` 或 `relay` 之一。

### 主循环

`g_main_loop_run(gloop)` 是 GLib 事件驱动编程的核心。进入主循环后：

1. **定时器驱动 ICE 状态机**: NiceAgent 的所有定时器（候选收集超时、连接检查重传、keepalive 等）都在主循环上下文中调度和触发。
2. **I/O 事件处理**: `nice_agent_attach_recv()` 注册的 I/O 回调（接收 STUN 包和用户数据）以及 `g_io_add_watch()` 注册的 stdin 监听都在主循环中处理。
3. **信号发射**: 所有 GObject 信号（`candidate-gathering-done`、`component-state-changed` 等）都在主循环上下文内同步触发。

程序退出主循环的两种路径：
- **失败**: `cb_component_state_changed` 检测到 `NICE_COMPONENT_STATE_FAILED`，调用 `g_main_loop_quit()`。
- **用户退出**: 用户按 Ctrl-D，`stdin_send_data_cb` 发送 `\0`；对端 `cb_nice_recv` 收到 `\0` 后调用 `g_main_loop_quit()`。

### 流程图

```
main()
  |
  +-- g_networking_init()                   初始化网络
  +-- g_main_loop_new()                     创建主循环
  +-- nice_agent_new()                      创建 ICE 代理
  +-- g_object_set("stun-server", ...)      配置 STUN 服务器
  +-- g_object_set("controlling-mode", ...) 配置控制方/被控方角色
  +-- g_signal_connect("candidate-gathering-done", ...)    注册信号
  +-- g_signal_connect("new-selected-pair", ...)
  +-- g_signal_connect("component-state-changed", ...)
  +-- stream_id = nice_agent_add_stream(agent, 1)          创建流(1个组件)
  +-- nice_agent_attach_recv(agent, stream_id, 1, ...)     挂接接收回调
  +-- nice_agent_gather_candidates(agent, stream_id)        启动候选收集
  +-- g_main_loop_run(gloop)                                进入主循环
        |
        |   [异步 -- 候选收集过程中]
        |   new-selected-pair 信号可能多次触发
        |
        +-- candidate-gathering-done 信号
        |     |
        |     +-- cb_candidate_gathering_done()
        |           |
        |           +-- print_local_data() -> stdout   打印本地候选
        |           +-- g_io_add_watch(stdin, ...)     等待用户输入远端候选
        |                 |
        |                 +-- stdin_remote_info_cb()
        |                       |
        |                       +-- parse_remote_data()
        |                             |
        |                             +-- nice_agent_set_remote_credentials()
        |                             +-- nice_agent_set_remote_candidates()
        |                                   |
        |                                   |   [触发连接检查开始]
        |                                   |
        |       [异步 -- 连接检查过程中]      |
        |       new-selected-pair 信号可能触发|
        |       component-state-changed 信号   |
        |              |                      |
        |              +-- NICE_COMPONENT_STATE_CONNECTED
        |              |     |
        |              |     +-- cb_component_state_changed()
        |              |           |
        |              |           +-- nice_agent_get_selected_pair()  获取选中候选对
        |              |           +-- g_io_add_watch(stdin, ...)      等待用户输入消息
        |              |                 |
        |              |                 +-- stdin_send_data_cb()
        |              |                       |
        |              |                       +-- nice_agent_send() -> 发送到对端
        |              |                       |        |
        |              |                       |        +-- 对端 cb_nice_recv() 收到数据
        |              |                       |
        |              |                       +-- [Ctrl-D] nice_agent_send("\0")
        |              |                              g_main_loop_quit()
        |              |
        |              +-- NICE_COMPONENT_STATE_FAILED
        |                    |
        |                    +-- cb_component_state_changed()
        |                          g_main_loop_quit()
        |
  +-- g_main_loop_unref(gloop)
  +-- g_object_unref(agent)
  +-- g_io_channel_unref(io_stdin)
  return EXIT_SUCCESS
```

### 关键设计要点

1. **事件驱动架构**: libnice 完全基于 GLib 主循环的事件驱动模型。`main()` 函数仅做初始化，所有 ICE 协商和数据收发逻辑都在信号回调中完成。

2. **候选交换方式**: 示例使用最简单的 "手动拷贝粘贴" 方式交换候选信息。在生产环境中，这一步应通过信令服务器（如 SIP、WebSocket 上的 SDP）完成。libnice 也提供了 SDP 辅助函数（如 `nice_agent_generate_local_sdp()` 和 `nice_agent_parse_remote_sdp()`）来简化这一过程。

3. **attach_recv 的双重作用**: `nice_agent_attach_recv()` 不仅用于接收用户数据，更是 ICE 连接检查的前提 -- 没有它，代理无法接收 STUN 包，连接检查将无法进行。

4. **角色模式**: controlling/controlled 模式由 `"controlling-mode"` 属性设置，决定了 ICE 候选对提名的策略。Controlling 方主动提名，controlled 方被动响应。

5. **单组件流**: 示例只使用了 1 个组件（component_id=1），即 `NICE_COMPONENT_TYPE_RTP`。实际 WebRTC 场景通常需要 2 个组件（RTP + RTCP）。

## sdp-example.c (284 行)

### 概述

`sdp-example.c` 演示了使用 SDP（Session Description Protocol）字符串来交换 ICE 候选信息的用法。与 simple-example 的"手动打印/解析候选"方式不同，sdp-example 使用 `nice_agent_generate_local_sdp()` 将本地凭据和所有候选序列化为单个 SDP 字符串，使用 `nice_agent_parse_remote_sdp()` 从 SDP 字符串中解析远端凭据和候选。这是生产环境中更常见的集成方式（如 SIP、WebRTC 信令）。

### 与 simple-example 的关键区别

#### 1. 线程架构

sdp-example 引入了双线程模型，**但这并非 libnice 的功能，而是为了解耦输入和事件处理**：

- **main 线程**: 运行 GLib 主循环 (`g_main_loop_run()`)，这是 NiceAgent 事件处理所必需的。
- **example_thread**: 执行所有 ICE 操作：创建代理、收集候选、通过 stdin 交换数据。

两个线程通过 `GMutex` + `GCond` 同步，信号回调通过设置标志 + `g_cond_signal()` 通知工作线程继续。

```c
// 全局同步变量
static GMutex gather_mutex, negotiate_mutex;
static GCond gather_cond, negotiate_cond;
static gboolean exit_thread, candidate_gathering_done, negotiation_done;

// 信号回调: 设置标志并通知工作线程
static void cb_candidate_gathering_done(...) {
  g_mutex_lock(&gather_mutex);
  candidate_gathering_done = TRUE;
  g_cond_signal(&gather_cond);
  g_mutex_unlock(&gather_mutex);
}

// 工作线程: 等待信号
g_mutex_lock(&gather_mutex);
while (!exit_thread && !candidate_gathering_done)
  g_cond_wait(&gather_cond, &gather_mutex);
g_mutex_unlock(&gather_mutex);
```

这种模式的优势：ICE 操作（候选交换、stdin I/O）放在非 GLib 线程中，可以在这些操作中使用阻塞 I/O 而不阻塞主循环。

#### 2. 新增 API: nice_agent_set_stream_name()

- **原型**: `gboolean nice_agent_set_stream_name (NiceAgent *agent, guint stream_id, const gchar *name);`
- **作用**: 为流指定一个媒体类型名称，有效值为 `"audio"`, `"video"`, `"text"`, `"application"`, `"image"` 和 `"message"`。
- **重要性**: SDP 生成依赖此名称。`nice_agent_generate_local_sdp()` 使用它填充 SDP 的 `m=` 行。如果流没有名称，`m=` 行会显示 `-`，且对端 `nice_agent_parse_remote_sdp()` 将无法正确解析。
- **参数说明**:
  - `stream_id`: 流 ID。
  - `name`: 媒体类型字符串（如 `"text"`），传入 `NULL` 无效。
- **返回值**: TRUE 成功；FALSE 表示 stream_id 无效或名称重复。
- **调用时机**: 必须在 `nice_agent_generate_local_sdp()` 之前调用。

#### 3. 新增 API: nice_agent_generate_local_sdp()

- **原型**: `gchar * nice_agent_generate_local_sdp (NiceAgent *agent);`
- **作用**: 生成一个 SDP 字符串，包含代理中所有流和组件的本地候选地址及 ICE 凭据。这是 `nice_agent_get_local_credentials()` + `nice_agent_get_local_candidates()` 的高级封装，将所有信息序列化为标准 SDP 格式。
- **参数说明**:
  - `agent`: NiceAgent 实例。
- **返回值**: (transfer full) SDP 字符串，使用后需 `g_free()` 释放。
- **SDP 内容说明**: 生成的 SDP 不包含 codec 行（如 `a=rtpmap:`），`m=` 行不列出任何 payload type。它仅包含 ICE 相关信息：
  - `a=ice-ufrag:` -- ICE username fragment
  - `a=ice-pwd:` -- ICE password
  - `a=candidate:` -- 每个候选的 SDP 序列化（包含 foundation、component-id、transport、priority、address、port、type 等属性）
  - 默认候选（`m=` 行的 `c=` 行）会选择第一个组件中优先级最低的候选。
- **调用时机**: 候选收集完成后。

sdp-example 中将生成的 SDP 做了 Base64 编码以便通过单行传输：
```c
sdp = nice_agent_generate_local_sdp(agent);
sdp64 = g_base64_encode((const guchar *)sdp, strlen(sdp));
printf("\n  %s\n", sdp64);
```

libnice 还提供更细粒度的 SDP API：
- `nice_agent_generate_local_stream_sdp()` -- 生成单个流的 SDP
- `nice_agent_generate_local_candidate_sdp()` -- 生成单个候选的 SDP 行

#### 4. 新增 API: nice_agent_parse_remote_sdp()

- **原型**: `int nice_agent_parse_remote_sdp (NiceAgent *agent, const gchar *sdp);`
- **作用**: 解析远端 SDP 字符串，从中提取候选地址和 ICE 凭据，并设置到代理上。这是 `nice_agent_set_remote_credentials()` + `nice_agent_set_remote_candidates()` 的高级封装。**此调用会触发 ICE 连接检查的启动**。
- **参数说明**:
  - `agent`: NiceAgent 实例。
  - `sdp`: 远端 SDP 字符串（纯文本，非 Base64）。
- **返回值**: 成功添加的候选数量，负数表示错误（如 SDP 格式无效或流名不匹配）。
- **前置条件**: 本地流的名称必须与远端 SDP 中 `m=` 行的名称匹配。如果本地流未命名，此函数将无法匹配。
- **调用时机**: 同 simple-example 的 `parse_remote_data()`，在接收到远端候选数据后。

sdp-example 中先对 Base64 解码再解析：
```c
sdp = (gchar *)g_base64_decode(line, &sdp_len);
if (sdp && nice_agent_parse_remote_sdp(agent, sdp) > 0) {
  // 解析成功，触发连接检查
}
```

libnice 还提供更细粒度的解析 API：
- `nice_agent_parse_remote_stream_sdp()` -- 解析单个流的 SDP 段
- `nice_agent_parse_remote_candidate_sdp()` -- 解析单个 `a=candidate:` 行

#### 5. 协商完成的等待方式

sdp-example 使用 `NICE_COMPONENT_STATE_READY` 作为协商完成的标志（而非 simple-example 的 `CONNECTED`）：

```c
static void cb_component_state_changed(..., guint state, ...) {
  if (state == NICE_COMPONENT_STATE_READY) {
    // 协商最终完成
    g_mutex_lock(&negotiate_mutex);
    negotiation_done = TRUE;
    g_cond_signal(&negotiate_cond);
    g_mutex_unlock(&negotiate_mutex);
  } else if (state == NICE_COMPONENT_STATE_FAILED) {
    g_main_loop_quit(gloop);
  }
}
```

`READY` 状态表示提名完成、选定的候选对已最终确定，比 `CONNECTED`（仅表示至少有一个可用候选对）更可靠。

### 流程图

```
main()                                      example_thread()
  |                                             |
  +-- g_networking_init()                       |
  +-- g_main_loop_new()                         |
  +-- g_thread_new("example thread", ...) ------+
  |                                             |
  |  g_main_loop_run(gloop) -- [主循环线程]      |
  |        |                                    |
  |        | [代理事件处理]                       |  nice_agent_new()
  |        |                                    |  g_object_set("stun-server", ...)
  |        |                                    |  g_object_set("controlling-mode", ...)
  |        |                                    |  g_signal_connect("candidate-gathering-done", ...)
  |        |                                    |  g_signal_connect("component-state-changed", ...)
  |        |                                    |
  |        |                                    |  stream_id = nice_agent_add_stream(agent, 1)
  |        |                                    |  nice_agent_set_stream_name(agent, stream_id, "text")
  |        |                                    |  nice_agent_attach_recv(agent, ...)
  |        |                                    |  nice_agent_gather_candidates(agent, stream_id)
  |        |                                    |
  |        |                                    |  [GCond 等待 candidate_gathering_done]
  |        |                                    |
  |        +-- candidate-gathering-done SIGNAL -+
  |        |   cb_candidate_gathering_done()     |
  |        |     g_cond_signal(gather_cond) ----+
  |        |                                    |
  |        |                                    |  sdp = nice_agent_generate_local_sdp(agent)
  |        |                                    |  printf(Base64(sdp)) -> stdout
  |        |                                    |  [从 stdin 读取远端 Base64 SDP]
  |        |                                    |  nice_agent_parse_remote_sdp(agent, sdp)
  |        |                                    |       |
  |        |                                    |       +-- [触发连接检查]
  |        |                                    |
  |        |                                    |  [GCond 等待 negotiation_done]
  |        |                                    |
  |        +-- component-state-changed SIGNAL --+
  |        |   NICE_COMPONENT_STATE_READY        |
  |        |     g_cond_signal(negotiate_cond) -+
  |        |                                    |
  |        |                                    |  nice_agent_get_selected_pair() 获取连接详情
  |        |                                    |  [stdin 循环: 读取 -> nice_agent_send()]
  |        |                                    |
  |        +-- cb_nice_recv()  收到数据 -> stdout|
  |        +-- cb_nice_recv()  收到 \0 -> quit   |
  |        |                                    |
  |  g_main_loop_quit                            |
  |                                             |  g_object_unref(agent)
  +-- g_thread_join() <------------------------ +
  +-- g_main_loop_unref(gloop)
```

### sdp-example 的关键设计要点

1. **SDP 作为接口契约**: `nice_agent_generate_local_sdp()` 和 `nice_agent_parse_remote_sdp()` 提供的是完整的序列化/反序列化，它们之间的关系类似 `nice_agent_get_local_credentials()` + `nice_agent_get_local_candidates()` 和 `nice_agent_set_remote_credentials()` + `nice_agent_set_remote_candidates()`，但只需一次函数调用即可完成。

2. **流命名是关键**: 如果没有调用 `nice_agent_set_stream_name()`，生成的 SDP 将无法被 `nice_agent_parse_remote_sdp()` 正确解析。流名称用于匹配 SDP 中的 `m=` 行。

3. **Base64 编码不是必须的**: sdp-example 进行 Base64 编码纯粹是为了方便通过单行 stdin 传输（避免换行符截断）。在实际信令协议中（如 SIP），SDP 通常作为消息体直接传输，不需要 Base64。

4. **线程同步模式**: sdp-example 使用 `GMutex` + `GCond` 实现了"信号回调通知工作线程"的同步模式。这是将 GLib 事件驱动模型与非 GLib 线程结合时的标准做法。

## threaded-example.c (461 行)

### 概述

`threaded-example.c` 演示了如何将 libnice 集成到多线程应用中的模式。与 sdp-example 一样，它使用一个工作线程执行 ICE 操作，主线程运行 GLib 主循环。候选交换方式与 simple-example 完全相同（手动格式：ufrag + password + 逗号分隔的候选），但整个 ICE 流程在非 GLib 线程中完成。关键学习点是：**libnice 本身不要求多线程，但当你需要在非 GLib 线程中使用它时，信号回调仍然在 GLib 主循环的线程中执行，需要同步机制来桥接**。

### 与 simple-example 的关键区别

#### 1. 双线程模型

```
main 线程:                    example_thread (工作线程):
  g_main_loop_new()            |
  g_thread_new() --------------+-- 启动
  g_main_loop_run()            |
    |                           +-- nice_agent_new(g_main_loop_get_context(gloop), ...)
    |                           |   // 代理事件在主循环上下文中调度
    |                           +-- g_signal_connect(...)  // 信号回调在 main 线程执行
    |                           +-- nice_agent_add_stream()
    |                           +-- nice_agent_attach_recv()  // I/O 回调在 main 线程执行
    |                           +-- nice_agent_gather_candidates()
    |                           |
  [信号回调]                    |   [GCond 等待 candidate_gathering_done]
  cb_candidate_gathering_done() |
    g_cond_signal() ----------->+-- 继续: 打印本地候选
    |                           +-- 等待 stdin 输入远端候选
    |                           +-- parse_remote_data()
    |                           |     +-- nice_agent_set_remote_credentials()
    |                           |     +-- nice_agent_set_remote_candidates()
    |                           |           |
    |                           |           +-- [触发连接检查]
    |                           |
  [信号回调]                    |   [GCond 等待 negotiation_done]
  cb_component_state_changed()  |
    NICE_COMPONENT_STATE_READY  |
    g_cond_signal() ----------->+-- 继续: 打印连接详情
    |                           +-- stdin 循环: nice_agent_send()
    |                           |
  cb_nice_recv()                |
    (收到 \0 则 g_main_loop_quit)|
    |                           |
  g_main_loop 退出              |
  g_thread_join() <-------------+-- g_object_unref(agent)
```

#### 2. 接收模式: 仍然是回调

threaded-example **并未使用** `nice_agent_recv_messages()` 阻塞式 API。它仍然使用 `nice_agent_attach_recv()` 注册回调（同 simple-example）：

```c
nice_agent_attach_recv(agent, stream_id, 1,
    g_main_loop_get_context(gloop), cb_nice_recv, NULL);
```

`cb_nice_recv()` 回调在 main 线程（GLib 主循环所在的线程）中触发，回调中打印接收到的数据。

#### 3. 同步机制: GMutex + GCond

由于信号回调在 main 线程中执行，而 ICE 操作流程在工作线程中按顺序执行，必须通过同步原语来等待异步事件：

| 事件 | 源 | 等待方 | 同步方式 |
|------|-----|--------|---------|
| 候选收集完成 | `cb_candidate_gathering_done` (main 线程) | 工作线程 | `gather_mutex` + `gather_cond` |
| ICE 协商完成/失败 | `cb_component_state_changed` (main 线程) | 工作线程 | `negotiate_mutex` + `negotiate_cond` |
| 程序退出 | stdin Ctrl-D | main 线程退出主循环 | `exit_thread` 标志 |

**关键模式**:
```c
// 信号回调（在 main 线程中执行）- 设置标志并通知
static void cb_component_state_changed(..., guint state, ...) {
  if (state == NICE_COMPONENT_STATE_READY) {
    g_mutex_lock(&negotiate_mutex);
    negotiation_done = TRUE;
    g_cond_signal(&negotiate_cond);
    g_mutex_unlock(&negotiate_mutex);
  } else if (state == NICE_COMPONENT_STATE_FAILED) {
    g_main_loop_quit(gloop);  // 失败直接退出
  }
}

// 工作线程中的等待循环
g_mutex_lock(&negotiate_mutex);
while (!exit_thread && !negotiation_done)
  g_cond_wait(&negotiate_cond, &negotiate_mutex);
g_mutex_unlock(&negotiate_mutex);
```

注意 `while` 而非 `if` 的使用 -- 这是条件变量的标准用法，防止虚假唤醒。

#### 4. 真正的阻塞式接收: nice_agent_recv_messages()（未在此示例中使用）

虽然此示例使用了回调模式，但 libnice 提供了真正的线程安全阻塞式接收 API，适合不需要 GLib 主循环的场景：

**nice_agent_recv_messages()**:
- **原型**: `gint nice_agent_recv_messages (NiceAgent *agent, guint stream_id, guint component_id, NiceInputMessage *messages, guint n_messages, GCancellable *cancellable, GError **error);`
- **作用**: 阻塞等待从指定组件接收数据，直到恰好 `n_messages` 条消息被填满、流被关闭或操作被取消。STUN 包由代理内部处理，不传递给调用者。
- **关键特性**:
  - 在多线程环境中安全使用，可从任意线程调用。
  - **不能**与 `nice_agent_attach_recv()` 在同一组件上混用。
  - 支持通过 `GCancellable` 从另一个线程取消阻塞操作。
  - 调用者预分配 `NiceInputMessage` 数组，收到后每个消息的 `length` 字段指示实际字节数。
- **返回值**: 成功写入 `messages` 的消息数（>0 或 n_messages==0），0 表示对端关闭流，-1 表示错误。
- **适用场景**: 非 GLib 应用、不想运行主循环的应用、需要完全控制接收线程的应用。

**nice_agent_recv_messages_nonblocking()** 是相应的非阻塞版本，行为类似但不会阻塞；如果接收会阻塞，返回 -1 并设置 `G_IO_ERROR_WOULD_BLOCK`。

**重要**: 使用阻塞式 API 时，由于代理本身不主动轮询消息，**必须持续调用这些函数**来驱动 STUN 消息的接收和处理，ICE 连接建立才能正常工作。

### 流程图

```
threaded-example 完整流程

main()                          main thread (gloop)          example_thread
  |                                                             |
  +-- g_networking_init()                                       |
  +-- g_main_loop_new()                                         |
  +-- g_thread_new() -----------+-- [main 线程运行主循环]        |
  |                             |                               |
  g_main_loop_run(gloop)        |   [事件处理中...]              |
  |                             |                               |
  |                             |                               +-- nice_agent_new()
  |                             |                               +-- nice_agent_add_stream(1)
  |                             |                               +-- nice_agent_attach_recv()
  |                             |                               +-- nice_agent_gather_candidates()
  |                             |                               |
  |                             |                               |   [等待 COND]
  |   "candidate-gathering-done"|  SIGNAL                       |
  |   cb: g_cond_signal() ------+-------------------------------+
  |                             |                               |
  |                             |                               +-- print_local_data()
  |                             |                               |   -> nice_agent_get_local_credentials()
  |                             |                               |   -> nice_agent_get_local_candidates()
  |                             |                               +-- stdin 读取远端候选
  |                             |                               +-- parse_remote_data()
  |                             |                               |     -> nice_agent_set_remote_credentials()
  |                             |                               |     -> nice_agent_set_remote_candidates()
  |                             |                               |         |
  |                             |                               |         +-- [触发协商]
  |                             |                               |
  |                             |                               |   [等待 COND]
  |   "component-state-changed" |  SIGNAL: READY                |
  |   cb: g_cond_signal() ------+-------------------------------+
  |                             |                               |
  |                             |                               +-- nice_agent_get_selected_pair()
  |                             |                               +-- [stdin 循环: nice_agent_send()]
  |                             |                               |
  |   "cb_nice_recv"            |  (收到对端数据)                |
  |   printf() -> stdout        |                               |
  |                             |                               |
  |   "cb_nice_recv"            |  (收到 \0)                    |
  |   g_main_loop_quit()        |                               |
  |   [gloop 退出]               |                               |
  |                             |                               |
  +-- g_thread_join() ----------+-- [线程结束] -----------------+
  +-- g_main_loop_unref(gloop)
  return EXIT_SUCCESS
```

### threaded-example 的关键设计要点

1. **Agent 必须在主循环线程中创建**: `nice_agent_new()` 需要 `GMainContext`，定时器和 I/O 事件在该上下文中调度。即使 Agent 对象在工作线程中创建，其事件处理仍发生在传入的 `GMainContext` 所属的线程（即运行 `g_main_loop_run()` 的线程）。

2. **两个示例使用相同的候选交换格式**: threaded-example 的 `print_local_data()` 和 `parse_remote_data()` 与 simple-example 完全相同（ufrag + password + 逗号分隔候选的文本格式）。这意味着它与 simple-example 可以互通。

3. **退出机制的三层协调**:
   - `exit_thread` 布尔标志：main 线程退出主循环后设置，通知工作线程停止。
   - `g_main_loop_quit()`: 由 `NICE_COMPONENT_STATE_FAILED` 信号或接收到 `\0` 触发。
   - `g_thread_join()`: main 线程等待工作线程结束后才释放资源。

4. **两种接收模式的对比**:

   | 特性 | `nice_agent_attach_recv()` 回调模式 | `nice_agent_recv_messages()` 阻塞模式 |
   |------|--------------------------------------|---------------------------------------|
   | 需要 GMainLoop | 是 | 否（但 ICE 状态机仍需要） |
   | 调用线程要求 | 信号回调在 GMainContext 所属线程 | 可从任意线程调用 |
   | 接收方式 | 回调推送 | 主动调用拉取 |
   | 多消息接收 | 每次回调一条消息 | 一次阻塞可接收多条 |
   | 可取消性 | 无法取消 | 支持 GCancellable |
   | 与对方 API 互斥 | 与 recv_messages 互斥 | 与 attach_recv 互斥 |

## 三种例子的适用场景对比

| 特性 | simple-example | sdp-example | threaded-example |
|------|---------------|-------------|-----------------|
| 候选交换格式 | 自定义文本（ufrag pwd cand...） | SDP (Base64 编码传输) | 自定义文本（同 simple-example） |
| 交换 API | `get_local_credentials` + `get_local_candidates` | `generate_local_sdp` | `get_local_credentials` + `get_local_candidates` |
| 解析 API | `set_remote_credentials` + `set_remote_candidates` | `parse_remote_sdp` | `set_remote_credentials` + `set_remote_candidates` |
| 流命名 | 不涉及 | 必须调用 `set_stream_name` | 不涉及 |
| 线程模型 | 单线程（全部在 main 线程） | 双线程（main + worker） | 双线程（main + worker） |
| 接收模式 | 回调 (`attach_recv`) | 回调 (`attach_recv`) | 回调 (`attach_recv`) |
| 信号同步 | 直接处理 | GMutex + GCond | GMutex + GCond |
| 协商完成标志 | `CONNECTED` | `READY` | `READY` |
| 新 API 暴露 | 无（基线） | SDP 生成/解析 系列 API | 线程安全同步模式 |
| 适用场景 | 学习调试、简单点对点 | SIP/WebRTC 信令集成 | 需要非 GLib 线程的应用 |
