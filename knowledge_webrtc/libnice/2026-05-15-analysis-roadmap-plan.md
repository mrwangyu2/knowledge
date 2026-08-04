# libnice 源码分析路线图 — 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 产出 11 份 libnice ICE 实现源码分析文档，覆盖全部模块（random → stun → socket → agent → gst）的逐函数精读及 4 条核心跨模块流程追踪。

**Architecture:** 自底向上分模块分析 + 跨模块核心流程追踪。先在入口文档从 example 建立感性认识，再逐模块精读底层实现，最后通过流程文档将模块间调用关系串联。每份模块文档按文件拆分，每个函数分析原型、作用、关键逻辑。

**Tech Stack:** C (GLib/GObject), Meson 构建系统

**Source:** `/data/libnice`
**Output:** `/home/frank/knowledge/knowledge_webrtc/libnice/`

---

### Task 1: 入口文档 — simple-example.c

**Files:**
- Read: `examples/simple-example.c` (439 行)
- Write: `01-entry-examples.md` (section: simple-example)

- [ ] **Step 1: 阅读 simple-example.c**

Read `examples/simple-example.c` 全文。理解 `main()` 的调用序列：
`nice_agent_new_full()` → `nice_agent_add_stream()` → `nice_agent_gather_candidates()` → `nice_agent_attach_recv()` → `nice_agent_set_remote_candidates()` → `nice_agent_get_local_candidates()` → `nice_agent_send()` → 主循环 → `nice_agent_remove_stream()`

- [ ] **Step 2: 分析每个 API 调用**

对 `main()` 中每个 `nice_agent_*()` 调用，写出：函数原型（从 `agent/agent.h` 查找）、各参数含义、返回值含义。穿插读到 `cb_candidate_gathering_done()`、`cb_component_state_changed()`、`cb_nice_recv()` 三个回调时分析回调参数和作用。

- [ ] **Step 3: 写入 01-entry-examples.md**

```markdown
# 01 — 入口：从例子开始

## simple-example.c (439 行)

### 概述
simple-example 是 libnice 最简示例：创建 agent，收集候选，交换 SDP，发送/接收数据。

### 调用链分析

#### 1. nice_agent_new_full()
原型: NiceAgent *nice_agent_new_full(GMainContext *ctx, NiceCompatibility compat, NiceAgentOption flags)
作用: 创建 ICE agent 实例
...

#### 2. nice_agent_add_stream()
...
```

对 simple-example 的每个 API 调用和回调写入分析。

- [ ] **Step 4: Commit**

```bash
git -C /home/frank/knowledge/knowledge_webrtc add libnice/01-entry-examples.md
git -C /home/frank/knowledge/knowledge_webrtc commit -m "docs: add simple-example analysis"
```

---

### Task 2: 入口文档 — sdp-example.c + threaded-example.c

**Files:**
- Read: `examples/sdp-example.c` (284 行), `examples/threaded-example.c` (461 行)
- Write: append to `01-entry-examples.md`

- [ ] **Step 1: 阅读并分析 sdp-example.c**

与 simple-example 对比，重点分析不同的部分：SDP 生成/解析 API (`nice_agent_generate_local_sdp()`, `nice_agent_parse_remote_sdp()`)、ICE 协商循环 (`nice_agent_get_stream_status()`, `g_main_loop`)。

- [ ] **Step 2: 阅读并分析 threaded-example.c**

分析多线程模式：`nice_agent_recv_messages()` 阻塞接收 vs `nice_agent_attach_recv()` 回调模式、线程安全注意事项。

- [ ] **Step 3: 追加到 01-entry-examples.md**

在 simple-example 节后追加 `## sdp-example.c` 和 `## threaded-example.c` 分析。

- [ ] **Step 4: Commit**

---

### Task 3: 模块文档 — random/

**Files:**
- Read: `random/random.c` (132 行), `random/random.h` (79 行), `random/random-glib.c` (108 行), `random/random-glib.h` (56 行)
- Write: `02-module-random.md`

- [ ] **Step 1: 阅读 random/ 全部文件**

理解 `NiceRNG` 结构和 `nice_rng_generate_bytes()` 等 API。注意它是如何封装 GRand 的。

- [ ] **Step 2: 写入 02-module-random.md**

```markdown
# 02 — 模块分析：random/

## 概述
random/ 是对 GLib GRand 的轻量封装，用于生成 STUN 事务 ID。

## 文件: random.h / random.c

### nice_rng_new()
原型: NiceRNG *nice_rng_new(void)
作用: 创建随机数生成器
关键逻辑: 调用 g_rand_new() 初始化 GRand 实例

### nice_rng_generate_bytes()
...
```

- [ ] **Step 3: Commit**

---

### Task 4: 模块文档 — stun/ message 层

**Files:**
- Read: `stun/stunmessage.c` (761 行), `stun/stunmessage.h` (1027 行), `stun/constants.h` (203 行), `stun/stun5389.c` (115 行), `stun/stun5389.h` (68 行)
- Write: `03-module-stun.md` (section: message format)

- [ ] **Step 1: 阅读 stunmessage.h 数据结构**

理解核心结构：`StunMessage`（消息结构体）、`StunAttribute`（属性 TLV）、`StunMethod` / `StunClass`（方法/类别枚举）、`StunMsgId`（事务 ID）、Magic Cookie。对照 `stun/constants.h` 查看常量定义。

- [ ] **Step 2: 阅读 stunmessage.c 所有函数**

逐函数分析消息构造和解析：
- `stun_message_init()` — 消息初始化
- `stun_message_append_attr()` / `stun_message_append_addr()` / `stun_message_append_string()` — 属性追加
- `stun_message_append_error()` — 错误响应
- `stun_message_serialize()` — 序列化为线格式缓冲区
- `stun_message_parse()` — 从线格式解析
- `stun_message_has_cookie()` — Magic Cookie 验证
- `stun_message_find_attr()` / `stun_message_get_class()` / `stun_message_get_method()` — 查询方法

- [ ] **Step 3: 阅读 stun5389.c**

分析 `stun_5389_compat_*()` 函数族，理解 RFC 5389 与 RFC 3489 的兼容处理。

- [ ] **Step 4: 写入 03-module-stun.md 前半部分**

写入文件概述和 message 格式分析，包含 StunMessage 结构体字段详解、属性 TLV 格式说明。

- [ ] **Step 5: Commit**

---

### Task 5: 模块文档 — stun/ agent 层

**Files:**
- Read: `stun/stunagent.c` (746 行), `stun/stunagent.h` (537 行), `stun/rand.c` (75 行), `stun/rand.h` (50 行), `stun/debug.c` (127 行), `stun/debug.h` (102 行), `stun/utils.c` (132 行), `stun/utils.h` (79 行)
- Append to: `03-module-stun.md`

- [ ] **Step 1: 阅读 stunagent.h**

理解 `StunAgent` 结构和事务层 API：`stun_agent_init()`、`stun_agent_build_request()`、`stun_agent_build_response()`、`stun_agent_validate()`。

- [ ] **Step 2: 阅读 stunagent.c 所有函数**

逐函数分析事务管理：
- 消息构建（多种重载的 `_build_*()` 函数）
- 消息验证（`stun_agent_validate()` — 指纹检查、HMAC 验证、事务 ID 匹配）
- 重传计时器集成

- [ ] **Step 3: 阅读辅助模块**

`rand.c` — 生成事务 ID；`debug.c` — 调试日志；`utils.c` — 地址/端口工具函数。

- [ ] **Step 4: 追加到 03-module-stun.md**

追加 agent 层分析。

- [ ] **Step 5: Commit**

---

### Task 6: 模块文档 — stun/ crypto 层

**Files:**
- Read: `stun/stunhmac.c` (303 行), `stun/stunhmac.h` (70 行), `stun/stuncrc32.c` (162 行), `stun/stuncrc32.h` (59 行)
- Append to: `03-module-stun.md`

- [ ] **Step 1: 阅读 stunhmac.c**

分析 `stun_hmac_*()` 函数族：MESSAGE-INTEGRITY 属性的 HMAC-SHA1 计算和验证流程。理解 long-term / short-term credential 两种认证模式下的 HMAC key 生成。

- [ ] **Step 2: 阅读 stuncrc32.c**

分析 FINGERPRINT 属性的 CRC32 计算：`stun_crc32()` 与 Magic Cookie 的 XOR 关系（0x5354554E）。

- [ ] **Step 3: 追加到 03-module-stun.md**

追加 HMAC 和 CRC32 层分析。

- [ ] **Step 4: Commit**

---

### Task 7: 模块文档 — stun/ usages 层

**Files:**
- Read: `stun/usages/bind.c` (595 行), `stun/usages/ice.c` (413 行), `stun/usages/turn.c` (455 行), `stun/usages/timer.c` (159 行) 及对应头文件
- Append to: `03-module-stun.md`

- [ ] **Step 1: 阅读 usages/bind.c**

分析 STUN Binding 用法：`stun_usage_bind_create()` 构造 Binding Request、`stun_usage_bind_process()` 解析 Binding Response、提取 XOR-MAPPED-ADDRESS。

- [ ] **Step 2: 阅读 usages/ice.c**

分析 ICE STUN 扩展用法：`stun_usage_ice_conncheck_create()` 构造连接检查请求（含 PRIORITY/USE-CANDIDATE/ICE-CONTROLLING/ICE-CONTROLLED 属性）、`stun_usage_ice_conncheck_process()` 解析响应。

- [ ] **Step 3: 阅读 usages/turn.c**

分析 TURN 用法：Allocate、Refresh、CreatePermission、ChannelBind、Send/Data Indication 的消息构建和响应处理函数。

- [ ] **Step 4: 阅读 usages/timer.c**

分析 STUN 事务重传定时器：RTO 计算、指数退避、最大重传次数。

- [ ] **Step 5: 追加到 03-module-stun.md 并收尾**

追加 usages 层分析，在文件末尾添加整个 stun/ 模块的调用关系小结和文件依赖图。

- [ ] **Step 6: Commit**

---

### Task 8: 模块文档 — socket/ core 抽象层

**Files:**
- Read: `socket/socket.c` (487 行), `socket/socket.h` (163 行), `socket/socket-priv.h` (121 行)
- Write: `04-module-socket.md` (section: core)

- [ ] **Step 1: 阅读 socket-priv.h 和 socket.h**

理解 `NiceSocket` 虚表结构：`send()`、`recv()`、`close()` 等函数指针。理解 `NiceSocketType` 枚举（UDP/BSD、TCP/BSD、UDP/TURN 等）和 `NiceAddress`。

- [ ] **Step 2: 阅读 socket.c 所有函数**

逐函数分析通用 socket 操作：
- `nice_socket_new()` — 根据类型工厂创建 socket
- `nice_socket_send()` / `nice_socket_recv()` — 统一收发接口
- `nice_socket_send_messages()` / `nice_socket_recv_messages()` — 批量收发
- `nice_socket_close()` — 统一关闭
- `nice_socket_is_reliable()` — 可靠性查询
- `nice_socket_set_extra_data()` / `nice_socket_recv_messages_extra()` — 附加数据（TOS/TCLASS）

- [ ] **Step 3: 写入 04-module-socket.md 第一部分**

写入 socket 抽象层分析，包含 NiceSocket 虚表机制和工厂模式详解。

- [ ] **Step 4: Commit**

---

### Task 9: 模块文档 — socket/ UDP 传输

**Files:**
- Read: `socket/udp-bsd.c` (544 行), `socket/udp-turn.c` (2291 行), `socket/udp-turn-over-tcp.c` (469 行) 及对应头文件
- Append to: `04-module-socket.md`

- [ ] **Step 1: 阅读 udp-bsd.c**

逐函数分析原生 UDP socket：
- BSD socket 创建（`socket()`, `bind()`）
- `socket_send()` / `socket_recv()` — 直接 UDP 收发
- IP_TOS / IPV6_TCLASS ancillary message 处理（近期提交关注点）
- `socket_send_messages()` / `socket_recv_messages()` — GSocket 批量收发

- [ ] **Step 2: 阅读 udp-turn.c**

逐函数分析 TURN UDP 传输（最大文件 2291 行）：
- TURN 分配生命周期（Allocate → Refresh → 数据传输 → 释放）
- Permission 管理（CreatePermission 请求/响应）
- ChannelData 封装/解封装（4 字节头 vs Send Indication 36 字节开销）
- `socket_send()` — 通过 ChannelData 或 Send Indication 发送
- `socket_recv()` — 从 ChannelData 或 Data Indication 接收
- TURN 定时器（分配刷新、权限刷新、信道绑定刷新）

- [ ] **Step 3: 阅读 udp-turn-over-tcp.c**

分析 TURN over TCP：TCP 连接建立、SOCKS5/HTTP 代理支持、帧边界处理（2 字节长度前缀）、与 udp-turn 的差异。

- [ ] **Step 4: 追加到 04-module-socket.md**

- [ ] **Step 5: Commit**

---

### Task 10: 模块文档 — socket/ TCP 传输

**Files:**
- Read: `socket/tcp-bsd.c` (493 行), `socket/tcp-active.c` (319 行), `socket/tcp-passive.c` (343 行) 及对应头文件
- Append to: `04-module-socket.md`

- [ ] **Step 1: 阅读 tcp-bsd.c**

分析基础 TCP socket：BSD TCP socket 创建、connect/bind/listen/accept 流程。

- [ ] **Step 2: 阅读 tcp-active.c 和 tcp-passive.c**

分析主动/被动 TCP 模式：tcp-active 主动发起连接、tcp-passive 监听接受连接。两者如何通过 NiceSocket 虚表统一接口。

- [ ] **Step 3: 追加到 04-module-socket.md**

- [ ] **Step 4: Commit**

---

### Task 11: 模块文档 — socket/ 代理与加密层

**Files:**
- Read: `socket/socks5.c` (501 行), `socket/http.c` (668 行), `socket/pseudossl.c` (332 行) 及对应头文件
- Append to: `04-module-socket.md`

- [ ] **Step 1: 阅读 socks5.c**

分析 SOCKS5 代理握手：认证协商、CONNECT 命令、地址类型处理（IPv4/IPv6/域名）。

- [ ] **Step 2: 阅读 http.c**

分析 HTTP CONNECT 代理：HTTP 代理隧道建立（CONNECT 方法、407 Proxy Auth）。

- [ ] **Step 3: 阅读 pseudossl.c**

分析伪 SSL 层：对所选加密库（GnuTLS 或 OpenSSL）的封装，DTLS/TLS 握手和数据传输抽象。

- [ ] **Step 4: 追加到 04-module-socket.md 并收尾**

追加代理/加密层分析，在文件末尾添加 socket/ 模块调用关系小结和依赖图。

- [ ] **Step 5: Commit**

---

### Task 12: 模块文档 — agent/ core（agent.c 第一部分）

**Files:**
- Read: `agent/agent.h` (1834 行), `agent/agent-priv.h` (359 行), `agent/agent.c` (7957 行)
- Write: `05-module-agent.md` (section: agent core + public API)

- [ ] **Step 1: 阅读 agent.h 公共 API**

理解公共类型和 API 分类：
- `NiceAgent` GObject 类型定义
- `NiceAgentOption` 位标志（ICE-LITE、ICE-TCP、RELIABLE 等）
- `NiceCompatibility` 兼容模式（RFC 5245、RFC 8445 等）
- 生命周期：`nice_agent_new_full()` / `nice_agent_new()` / `g_object_unref()`
- Stream 管理：`nice_agent_add_stream()` / `nice_agent_remove_stream()`
- 候选管理：`nice_agent_gather_candidates()` / `nice_agent_set_remote_candidates()` / `nice_agent_get_local_candidates()`
- 数据收发：`nice_agent_send()` / `nice_agent_attach_recv()` / `nice_agent_recv_messages()`
- 回调信号：`candidate-gathering-done`、`component-state-changed`、`new-candidate`、`new-remote-candidate` 等
- SDP：`nice_agent_generate_local_sdp()` / `nice_agent_parse_remote_sdp()`

- [ ] **Step 2: 阅读 agent-priv.h**

理解内部结构 `NiceAgent` 私有字段：streams 列表、components、local/remote candidates、conncheck 状态、discovery 状态、GLib 定时器、GMainContext。

- [ ] **Step 3: 阅读 agent.c 公共 API 实现**

逐函数分析生命周期、stream 管理、候选管理的实现：
- `nice_agent_new_full()` — GObject 构造，选项应用
- `nice_agent_class_init()` / `nice_agent_init()` — GObject 类/实例初始化
- `nice_agent_dispose()` / `nice_agent_finalize()` — 清理
- `nice_agent_add_stream()` / `nice_agent_remove_stream()` — Stream 增删
- `nice_agent_gather_candidates()` — 触发候选收集入口
- `nice_agent_set_remote_candidates()` — 远端候选注入
- `nice_agent_get_local_candidates()` — 本地候选查询

- [ ] **Step 4: 写入 05-module-agent.md 第一部分**

写入 agent core 概述、GObject 类型系统集成、公共 API 分析。

- [ ] **Step 5: Commit**

---

### Task 13: 模块文档 — agent/ core（agent.c 第二部分 + debug）

**Files:**
- Read: `agent/agent.c` (继续), `agent/debug.c` (176 行), `agent/debug.h` (126 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 agent.c 信号和回调部分**

分析信号发射机制：`nice_agent_attach_recv()` 的回调管理、`new-candidate` / `new-remote-candidate` / `candidate-gathering-done` / `component-state-changed` 等信号的发射时机。

- [ ] **Step 2: 阅读 agent.c 定时器驱动逻辑**

分析 GLib 定时器如何驱动 ICE 状态机：conncheck 定时器、discovery 定时器、保活定时器（`g_timer_*()` / `g_timeout_add()`）。

- [ ] **Step 3: 阅读 debug.c**

分析 `nice_debug()` 函数和 `NICE_DEBUG` 环境变量处理：`G_LOG_DOMAIN` 设置、调试级别。

- [ ] **Step 4: 追加到 05-module-agent.md**

- [ ] **Step 5: Commit**

---

### Task 14: 模块文档 — agent/ component

**Files:**
- Read: `agent/component.c` (1795 行), `agent/component.h` (374 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 component.h**

理解 `Component` 结构：component ID、状态（`NiceComponentState`: DISCONNECTED → GATHERING → CONNECTING → CONNECTED → READY → FAILED）、关联的 socket、候选列表、选中的候选对。

- [ ] **Step 2: 阅读 component.c 所有函数**

逐函数分析 component 状态管理：
- `component_new()` / `component_free()` — 创建/销毁
- `component_set_state()` — 状态转换和信号发射
- `component_find_selected_pair()` — 选中候选对查询
- `component_io_callback()` — I/O 事件回调分发
- `component_schedule_io_callback()` — GSource 集成
- socket 绑定和切换逻辑

- [ ] **Step 3: 追加到 05-module-agent.md**

- [ ] **Step 4: Commit**

---

### Task 15: 模块文档 — agent/ stream + candidate + address

**Files:**
- Read: `agent/stream.c` (201 行), `agent/stream.h` (139 行), `agent/candidate.c` (497 行), `agent/candidate.h` (296 行), `agent/candidate-priv.h` (127 行), `agent/address.c` (449 行), `agent/address.h` (337 行), `agent/interfaces.c` (932 行), `agent/interfaces.h` (95 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 stream.c**

分析 Stream 结构：stream ID、components 数组、创建/销毁、流状态查询。

- [ ] **Step 2: 阅读 candidate.c + candidate-priv.h**

分析 `NiceCandidate` 结构：候选类型（host/srflx/relay/peer-reflexive）、传输类型（UDP/TCP）、优先级计算、foundation 概念、候选对（`NiceCandidatePair`）结构。

- [ ] **Step 3: 阅读 address.c**

分析 `NiceAddress` 封装：IPv4/IPv6 统一处理、地址比较、字符串转换、端口操作。

- [ ] **Step 4: 阅读 interfaces.c**

分析网络接口枚举：`nice_interfaces_get_local_interfaces()` — 获取本地 IP 地址列表（用于 host 候选收集）、IPv4/IPv6 过滤。

- [ ] **Step 5: 追加到 05-module-agent.md**

- [ ] **Step 6: Commit**

---

### Task 16: 模块文档 — agent/ conncheck（第一部分）

**Files:**
- Read: `agent/conncheck.c` (5035 行), `agent/conncheck.h` (134 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 conncheck.h**

理解连接检查阶段的核心类型：`ConnCheck` 结构、检查状态枚举、定时器常量。

- [ ] **Step 2: 阅读 conncheck.c 前 2500 行**

重点函数（按调用顺序）：
- `conn_check_start()` — 启动连接检查阶段
- `conn_check_handle()` — 单次检查调度入口
- `conn_check_send()` — 构造并发送 STUN Binding Request（通过 socket 层）
- `priv_map_reply_to_conn_check_request()` — 响应匹配

- [ ] **Step 3: 追加到 05-module-agent.md**

- [ ] **Step 4: Commit**

---

### Task 17: 模块文档 — agent/ conncheck（第二部分 + discovery）

**Files:**
- Read: `agent/conncheck.c` (续), `agent/discovery.c` (1478 行), `agent/discovery.h` (174 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 conncheck.c 后半部分**

重点函数：
- `priv_conn_check_unfreeze_related()` — 候选对解冻（RFC 8445 Section 7.1 的 frozen/frozen 状态管理）
- `priv_conn_check_process_response()` — 处理 STUN 响应
- `priv_update_check_list_state_for_ready()` — 检查列表状态更新
- `priv_mark_pair_nominated()` — 提名标记
- `priv_conn_check_tick_stream()` — 定时器驱动的检查循环
- 候选对状态机：WAITING → IN_PROGRESS → SUCCEEDED / FAILED / FROZEN

- [ ] **Step 2: 阅读 discovery.c**

逐函数分析候选发现：
- `discovery_start()` — 启动候选收集
- `discovery_add_local_host_candidate()` — 添加 host 候选
- `discovery_add_server_reflexive_candidate()` — 添加 srflx 候选（通过 STUN Binding 发现）
- `discovery_add_relay_candidate()` — 添加 relay 候选（通过 TURN Allocate）
- `discovery_tick()` — 定时器驱动发现进度
- `discovery_schedule()` — 调度下一次发现步骤

- [ ] **Step 3: 追加到 05-module-agent.md**

- [ ] **Step 4: Commit**

---

### Task 18: 模块文档 — agent/ I/O streams

**Files:**
- Read: `agent/inputstream.c` (494 行), `agent/inputstream.h` (83 行), `agent/outputstream.c` (661 行), `agent/outputstream.h` (82 行), `agent/iostream.c` (352 行), `agent/iostream.h` (83 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 inputstream.c**

分析 `NiceInputStream` GObject：基于 GInputStream 的 ICE 接收流封装、`read()` / `read_async()` / `skip()` 实现、如何对接 GIO 异步 I/O 模型。

- [ ] **Step 2: 阅读 outputstream.c**

分析 `NiceOutputStream` GObject：基于 GOutputStream 的 ICE 发送流封装、`write()` / `write_async()` 实现。

- [ ] **Step 3: 阅读 iostream.c**

分析 `NiceIOStream`：组合 InputStream + OutputStream 的 GIO 双向流。

- [ ] **Step 4: 追加到 05-module-agent.md**

- [ ] **Step 5: Commit**

---

### Task 19: 模块文档 — agent/ pseudotcp

**Files:**
- Read: `agent/pseudotcp.c` (2646 行), `agent/pseudotcp.h` (599 行)
- Append to: `05-module-agent.md`

- [ ] **Step 1: 阅读 pseudotcp.h**

理解 PseudoTcp 结构体：状态枚举（TCP_LISTEN → TCP_ESTABLISHED → TCP_CLOSED）、拥塞控制参数（cwnd、ssthresh）、重传定时器、分段/重组缓冲。

- [ ] **Step 2: 阅读 pseudotcp.c 核心逻辑**

逐函数分析伪 TCP 实现：
- `pseudo_tcp_connect()` / `pseudo_tcp_listen()` — 连接建立
- `pseudo_tcp_send()` — 数据分段和发送
- `pseudo_tcp_recv()` — 数据重组和接收
- 拥塞控制：慢启动、拥塞避免、快速重传
- 重传机制：RTO 计算、指数退避、最大重传次数
- `pseudo_tcp_op_connect()` / `pseudo_tcp_op_recv() / `pseudo_tcp_notify_message()` — 内部消息处理

- [ ] **Step 3: 追加到 05-module-agent.md 并收尾**

追加 pseudotcp 分析，在文件末尾添加 agent/ 模块整体调用关系小结和文件依赖图。

- [ ] **Step 4: Commit**

---

### Task 20: 模块文档 — gst/

**Files:**
- Read: `gst/gstnice.c` (62 行), `gst/gstnice.h` (40 行), `gst/gstnicesrc.c` (487 行), `gst/gstnicesrc.h` (89 行), `gst/gstnicesink.c` (577 行), `gst/gstnicesink.h` (95 行)
- Write: `06-module-gst.md`

- [ ] **Step 1: 阅读 gstnice.c**

分析 GStreamer 插件注册：`gst_nice_register()` → `gst_element_register()`。

- [ ] **Step 2: 阅读 gstnicesrc.c**

分析 `nicesrc` GstElement：数据接收 → GstBuffer 转换、`GstNetControlMessageMeta` 附加元数据（近期提交）、pad 模板和协商。

- [ ] **Step 3: 阅读 gstnicesink.c**

分析 `nicesink` GstElement：GstBuffer → `nice_agent_send()` 发送。

- [ ] **Step 4: 写入 06-module-gst.md**

- [ ] **Step 5: Commit**

---

### Task 21: 流程文档 — 候选收集

**Files:**
- Read: `agent/discovery.c`, `agent/agent.c` (gather 部分), `stun/usages/bind.c`, `stun/usages/turn.c`, `socket/udp-bsd.c`, `socket/udp-turn.c`
- Write: `07-flow-candidate-gathering.md`

- [ ] **Step 1: 追踪 host 候选收集**

从 `nice_agent_gather_candidates()` → `discovery_start()` → `discovery_add_local_host_candidate()` → `interfaces.c` 枚举本地 IP → 创建 `NiceCandidate` → 信号 `new-candidate` 发射。

- [ ] **Step 2: 追踪 srflx 候选收集**

host 候选创建后 → `discovery_tick()` → 向 STUN server 发送 Binding Request → `stun/usages/bind.c` 构造消息 → `socket/udp-bsd.c` 发送 → 接收 Binding Response → 提取 XOR-MAPPED-ADDRESS → `discovery_add_server_reflexive_candidate()` → 信号 `new-candidate`。

- [ ] **Step 3: 追踪 relay 候选收集**

srflx 候选收集完成后 → 向 TURN server 发送 Allocate Request → `stun/usages/turn.c` 构造 → `socket/udp-turn.c` 或 `socket/udp-turn-over-tcp.c` 发送 → 接收 Allocate Success Response → 提取 XOR-RELAYED-ADDRESS → `discovery_add_relay_candidate()` → 信号 `new-candidate`。

- [ ] **Step 4: 绘制状态机图**

ASCII art 展示候选收集阶段的状态转换：NOT_STARTED → GATHERING_HOST → GATHERING_SRFLX → GATHERING_RELAY → DONE。

- [ ] **Step 5: 写入 07-flow-candidate-gathering.md**

- [ ] **Step 6: Commit**

---

### Task 22: 流程文档 — 连接检查

**Files:**
- Read: `agent/conncheck.c`, `stun/usages/ice.c`, `stun/usages/timer.c`
- Write: `08-flow-connectivity-checks.md`

- [ ] **Step 1: 追踪连接检查启动**

candidate gathering done → `conn_check_start()` → 构造候选对列表 → 计算优先级 → 按优先级排序 → 设置初始 frozen 状态。

- [ ] **Step 2: 追踪单次检查周期**

`conn_check_handle()` → `conn_check_send()` → `stun/usages/ice.c` `stun_usage_ice_conncheck_create()` 构造 Binding Request（含 PRIORITY/USE-CANDIDATE/ICE-CONTROLLING/ICE-CONTROLLED 属性）→ socket 发送 → 接收响应 → `priv_conn_check_process_response()` 解析 → 触发对端检查 → `priv_conn_check_unfreeze_related()` 解冻相关对。

- [ ] **Step 3: 追踪检查完成判定**

`priv_update_check_list_state_for_ready()` → 所有对 SUCCEEDED 或 FAILED → 选定最佳对 → 进入 READY 状态 → 发射 `component-state-changed` 信号。

- [ ] **Step 4: 绘制候选对状态机**

WAITING → FROZEN → IN_PROGRESS → SUCCEEDED / FAILED 的完整状态转换图。

- [ ] **Step 5: 写入 08-flow-connectivity-checks.md**

- [ ] **Step 6: Commit**

---

### Task 23: 流程文档 — 提名与数据收发

**Files:**
- Read: `agent/conncheck.c` (nomination 部分), `agent/component.c`, `agent/agent.c` (send/recv 部分), `socket/socket.c`, `socket/udp-bsd.c`
- Write: `09-flow-nomination-data.md`

- [ ] **Step 1: 追踪常规提名流程**

Controlling agent 端：选定有效对 → 发送带 USE-CANDIDATE 标志的 Binding Request → 标记该对为 nominated。Controlled agent 端：收到带 USE-CANDIDATE 的 Binding Request → `priv_mark_pair_nominated()` → 确认选中对。

- [ ] **Step 2: 追踪激进提名流程**

ICE-LITE 或特定配置下的激进提名：无需等待常规检查完成，直接提名第一个有效对。

- [ ] **Step 3: 追踪数据发送路径**

`nice_agent_send()` → 查找 component 的 selected pair → 通过 `NiceSocket` 虚表调用 → `socket/udp-bsd.c` 或 `socket/udp-turn.c` 的 `send()` → 线格式发送。

- [ ] **Step 4: 追踪数据接收路径**

socket 收到数据 `recv()` → `component_io_callback()` → 回调分发：STUN 消息 → conncheck 处理；应用数据 → `cb_nice_recv()` 用户回调或 `nice_agent_recv_messages()` 缓冲区。

- [ ] **Step 5: 写入 09-flow-nomination-data.md**

- [ ] **Step 6: Commit**

---

### Task 24: 流程文档 — TURN 分配

**Files:**
- Read: `socket/udp-turn.c`, `stun/usages/turn.c`, `stun/usages/timer.c`, `socket/socks5.c`, `socket/http.c`
- Write: `10-flow-turn-allocation.md`

- [ ] **Step 1: 追踪 TURN 分配创建**

从 `socket/udp-turn.c` 的 socket 创建入口 → Allocate Request 构造（`stun/usages/turn.c`）→ 发送 → 接收 Allocate Success/Error Response → 提取 relayed address。

- [ ] **Step 2: 追踪 TURN 权限和信道**

Permission 创建流程（CreatePermission Request/Response）、Channel 绑定流程（ChannelBind Request/Response）、ChannelData 格式（4 字节头：channel number + length）。

- [ ] **Step 3: 追踪 TURN 数据中继**

发送路径：应用数据 → ChannelData 封装 → TURN server → Data Indication → peer。接收路径：peer → TURN server → ChannelData 或 Data Indication → socket recv → 解封装 → 上交。

- [ ] **Step 4: 追踪 TURN 保活和回收**

Refresh 定时器 → 发送 Refresh Request → 更新 allocation 生命周期 → 超时未刷新 → 释放 allocation。Permission 和 Channel 的定时刷新。

- [ ] **Step 5: 追踪代理路径**

TCP TURN 场景下的 SOCKS5 和 HTTP CONNECT 代理握手流程集成。

- [ ] **Step 6: 写入 10-flow-turn-allocation.md**

- [ ] **Step 7: Commit**

---

### Task 25: 总索引 README.md

**Files:**
- Write: `README.md`

- [ ] **Step 1: 写入 README.md**

```markdown
# libnice 源码分析

从浅到深、逐函数精读 libnice ICE (RFC 8445) 实现。

## 文档导航

### 推荐阅读顺序

1. [01-entry-examples.md](01-entry-examples.md) — 从例子开始，建立感性认识
2. [02-module-random.md](02-module-random.md) — random/ 随机数模块
3. [03-module-stun.md](03-module-stun.md) — stun/ STUN 协议实现
4. [04-module-socket.md](04-module-socket.md) — socket/ 套接字抽象层
5. [05-module-agent.md](05-module-agent.md) — agent/ ICE 代理核心
6. [06-module-gst.md](06-module-gst.md) — gst/ GStreamer 插件
7. [07-flow-candidate-gathering.md](07-flow-candidate-gathering.md) — 流程：候选收集
8. [08-flow-connectivity-checks.md](08-flow-connectivity-checks.md) — 流程：连接检查
9. [09-flow-nomination-data.md](09-flow-nomination-data.md) — 流程：提名与数据收发
10. [10-flow-turn-allocation.md](10-flow-turn-allocation.md) — 流程：TURN 分配与中继

### 模块总览

| 模块 | 文件数 | 代码量 | 职责 |
|------|--------|--------|------|
| random/ | 2 | ~200 行 | 随机数生成 |
| stun/ | 14 | ~7200 行 | STUN 协议消息/事务/usages |
| socket/ | 18 | ~7300 行 | 多传输类型 socket 抽象 |
| agent/ | 24 | ~18000 行 | ICE 代理核心逻辑 |
| gst/ | 4 | ~1350 行 | GStreamer 插件 |

### 关键数据结构速查

- StunMessage (stunmessage.h) — STUN 消息结构
- NiceSocket (socket-priv.h) — socket 虚表
- NiceAgent (agent-priv.h) — ICE 代理私有结构
- Component (component.h) — 组件状态
- NiceCandidate (candidate.h) — ICE 候选
- ConnCheck (conncheck.h) — 连接检查状态
- PseudoTcp (pseudotcp.h) — 伪 TCP 状态
```

- [ ] **Step 2: Commit**

---

### 验收标准

- [ ] 10 份分析文档 + 1 份 README 全部写入 `/home/frank/knowledge/knowledge_webrtc/libnice/`
- [ ] 每个模块的每个文件都有对应分析章节
- [ ] 每个公共和关键内部函数都有原型+作用+关键逻辑分析
- [ ] 4 份流程文档标注了源文件和函数名
- [ ] 所有跨模块引用标注了 "→ 详见 0X-module-xxx.md"
