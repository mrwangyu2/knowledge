# 04 -- 模块分析：socket/

## 模块概述

`socket/` 模块是 libnice 的套接字抽象层，其核心职责是在统一接口下封装多种底层传输协议。支持的传输类型包括：

- 原生 UDP（UDP/BSD）
- 原生 TCP（TCP/BSD）
- 主动 TCP（TCP/Active，由客户端发起连接）
- 被动 TCP（TCP/Passive，服务端监听）
- PseudoSSL（用于 TCP 链接上的 SSL/TLS 加密伪装，支持 GnuTLS 或 OpenSSL）
- UDP-over-TURN（通过 TURN 中继的 UDP）
- UDP-over-TURN-over-TCP（通过 TCP 承载的 TURN 中继 UDP）
- HTTP 代理（用于 TURN TCP 中继）
- SOCKS5 代理（用于 TURN TCP 中继）
- TCP_SO（基于已有 GSocket 封装）

模块共有 **18 个文件**（9 个 .c + 9 个 .h），代码量约 **7300 行**。

**设计模式**：采用"C 语言虚表（vtable）+ 工厂模式"。核心结构体 `NiceSocket` 包含一组函数指针，每个具体传输类型填充自己的实现函数，上层调用者通过统一接口分派，完全不知道底层细节。

---

## 第一部分：核心抽象层

### 文件：socket-priv.h（121 行）

`socket-priv.h` 定义了发送队列管理 API，是内部辅助接口，不对外开放。它提供了一套可靠数据排队发送的机制，用于 TCP 等可靠传输场景。

#### 内部结构体 NiceSocketQueuedSend

```c
struct _NiceSocketQueuedSend {
  guint8 *buf;      /* owned -- 已合并（compact）的消息缓冲区 */
  gsize length;     /* 缓冲区有效长度 */
  NiceAddress to;   /* 目标地址 */
};
```

该结构体在 `socket.c` 中定义，用于将一条完整的输出消息转换为单个连续缓冲区，加入发送队列。

#### 发送队列辅助函数

| 函数 | 作用 |
|------|------|
| `nice_socket_queue_send()` | 将多条消息压缩进发送队列，把 scatter-gather 缓冲区合并为连续内存块 |
| `nice_socket_queue_send_with_callback()` | 将单条部分消息加入队列（头或尾），同时创建可写 IO source 以通知上层可继续发送 |
| `nice_socket_flush_send_queue()` | 将队列中全部消息通过基 socket 可靠发送（用于 PseudoSSL 等场景） |
| `nice_socket_flush_send_queue_to_socket()` | 将队列消息直接发送到底层 GSocket，遇到 WOULD_BLOCK 则将未发送部分重新入队 |
| `nice_socket_free_send_queue()` | 释放队列中所有待发送项，不清空消息内容也不发送 |
| `nice_socket_type_to_string()` | 将枚举值转为字符串描述（`"udp"`、`"tcp"`、`"ssl"`、`"udp-turn"` 等） |

---

### 文件：socket.h（163 行）

`socket.h` 是 socket 模块的核心头文件，定义了 `NiceSocket` 虚表结构体、`NiceSocketType` 枚举以及全部公共 API。

#### NiceSocketType 枚举

```c
typedef enum {
  NICE_SOCKET_TYPE_UDP_BSD,           /* 原生 UDP Berkeley 套接字 */
  NICE_SOCKET_TYPE_TCP_BSD,           /* 原生 TCP Berkeley 套接字 */
  NICE_SOCKET_TYPE_PSEUDOSSL,         /* 伪 SSL/TLS 加密层 */
  NICE_SOCKET_TYPE_HTTP,              /* HTTP 隧道代理 */
  NICE_SOCKET_TYPE_SOCKS5,            /* SOCKS5 代理 */
  NICE_SOCKET_TYPE_UDP_TURN,          /* 基于 UDP 的 TURN 分配 */
  NICE_SOCKET_TYPE_UDP_TURN_OVER_TCP, /* 基于 TCP 的 TURN 分配 */
  NICE_SOCKET_TYPE_TCP_ACTIVE,        /* 主动 TCP（客户端发起 connect） */
  NICE_SOCKET_TYPE_TCP_PASSIVE,       /* 被动 TCP（服务端 accept） */
  NICE_SOCKET_TYPE_TCP_SO             /* 基于已有 GSocket 的 TCP（用于 GSocketListener 回调） */
} NiceSocketType;
```

**设计要点**：
- `TCP_ACTIVE` / `TCP_PASSIVE` / `TCP_SO` 是三类入口，它们最终都会创建 `TCP_BSD` 作为内层套接字来承载实际的数据收发。
- `PSEUDOSSL` 是装饰器模式：它包装一个内部 `NiceSocket`，在数据收发时加解密。
- `HTTP` / `SOCKS5` 是代理隧道层，也包装一个内部 `tcp-bsd`。

#### NiceSocket 虚表结构体

这是整个 socket 模块的核心设计：

```c
struct _NiceSocket
{
  /* ---- 基础属性 ---- */
  NiceAddress addr;        /* 本地绑定的地址 */
  NiceSocketType type;     /* socket 类型标记 */

  GSocket *fileno;         /* 底层 GLib GSocket 对象（文件描述符包装） */

  /* ---- 虚表函数指针 ---- */

  /* 非可靠批量接收：非阻塞，返回实际接收到的消息数 */
  gint (*recv_messages) (NiceSocket *sock,
      NiceInputMessage *recv_messages, guint n_recv_messages,
      NiceMessageExtraData *exdata);

  /* 非可靠批量发送：非阻塞，返回实际发送的消息数 */
  gint (*send_messages) (NiceSocket *sock, const NiceAddress *to,
      const NiceOutputMessage *messages, guint n_messages);

  /* 可靠批量发送：阻塞队列机制，保证全部发送 */
  gint (*send_messages_reliable) (NiceSocket *sock, const NiceAddress *to,
      const NiceOutputMessage *messages, guint n_messages);

  /* 查询传输是否可靠（TCP=true, UDP=false） */
  gboolean (*is_reliable) (NiceSocket *sock);

  /* 查询是否可以向给定地址发送（用于 ICE 发送前检查） */
  gboolean (*can_send) (NiceSocket *sock, NiceAddress *addr);

  /* 设置可写回调（用于 TCP 连接建立后或发送队列排空时通知） */
  void (*set_writable_callback) (NiceSocket *sock,
      NiceSocketWritableCb callback, gpointer user_data);

  /* 判断当前 socket 是否基于另一个 socket（用于 TURN/PseudoSSL 链式包装场景） */
  gboolean (*is_based_on) (NiceSocket *sock, NiceSocket *other);

  /* 关闭 socket */
  void (*close) (NiceSocket *sock);

  /* 私有数据指针，各实现使用自己的结构体 */
  void *priv;
};
```

**虚表各字段分析**：

| 字段 | 方向 | 语义 | 说明 |
|------|------|------|------|
| `recv_messages` | 虚表 | 接收数据 | 非阻塞批量接收，返回实际接收到的消息数量。必须支持 `n_recv_messages == 0` 的情况 |
| `send_messages` | 虚表 | 发送数据 | 非阻塞批量发送，返回实际发送的消息数量。支持 scatter-gather I/O（`GOutputVector` 数组） |
| `send_messages_reliable` | 虚表 | 可靠发送 | 通过内部队列保证数据全部发送，若立即不可写则排队等待 |
| `is_reliable` | 虚表 | 元数据查询 | TCP 系返回 TRUE，UDP 系返回 FALSE |
| `can_send` | 虚表 | 就绪检查 | 可选实现（NULL 则始终返回 TRUE），用于 ICE conncheck 前的发送可行性检查 |
| `set_writable_callback` | 虚表 | 异步通知 | 可选实现，用于 TCP 连接建立/排空队列后的可写通知 |
| `is_based_on` | 虚表 | 拓扑查询 | 可选实现，递归遍历包装链判断当前 socket 是否基于另一个。用于防止重复候选/回路 |
| `close` | 虚表 | 资源释放 | 每个实现清理自己的 `priv` 和内部资源 |
| `priv` | 存储 | 私有状态 | 各实现的具体状态（如 `UdpBsdSocketPrivate`、`TcpPriv` 等） |

**虚表机制的核心价值**：
- 上层代码（`agent/component.c`）只持有 `NiceSocket*` 指针，调用 `nice_socket_recv_messages(sock, ...)` 时完全不关心底层是 UDP 还是 TCP-over-TURN-over-SSL。
- 添加新传输类型只需实现这组函数指针并注册新的 `NiceSocketType` 枚举值，无需修改调用方代码。

#### 公共 API 原型

```c
/* --- 接收 --- */
G_GNUC_WARN_UNUSED_RESULT
gint nice_socket_recv_messages (NiceSocket *sock,
    NiceInputMessage *recv_messages, guint n_recv_messages,
    NiceMessageExtraData *exdata);

gssize nice_socket_recv (NiceSocket *sock, NiceAddress *from,
    gsize len, gchar *buf);

/* --- 发送 --- */
gint nice_socket_send_messages (NiceSocket *sock, const NiceAddress *addr,
    const NiceOutputMessage *messages, guint n_messages);

gint nice_socket_send_messages_reliable (NiceSocket *sock, const NiceAddress *addr,
    const NiceOutputMessage *messages, guint n_messages);

gssize nice_socket_send (NiceSocket *sock, const NiceAddress *to,
    gsize len, const gchar *buf);

gssize nice_socket_send_reliable (NiceSocket *sock, const NiceAddress *addr,
    gsize len, const gchar *buf);

/* --- 元数据查询 --- */
gboolean nice_socket_is_reliable (NiceSocket *sock);
gboolean nice_socket_can_send (NiceSocket *sock, NiceAddress *addr);

/* --- 异步通知 --- */
void nice_socket_set_writable_callback (NiceSocket *sock,
    NiceSocketWritableCb callback, gpointer user_data);

/* --- 拓扑查询 --- */
gboolean nice_socket_is_based_on (NiceSocket *sock, NiceSocket *other);

/* --- 生命周期 --- */
void nice_socket_free (NiceSocket *sock);
```

**API 层次说明**：
- `nice_socket_send()` / `nice_socket_recv()`（单缓冲区，返回 `gssize`）是对 `nice_socket_send_messages()` / `nice_socket_recv_messages()`（多缓冲区，返回消息数 `gint`）的便捷封装。单缓冲区版本构造一个 `NiceInputMessage` / `NiceOutputMessage` 后直接调用虚表。
- `recv_messages` 被标记为 `G_GNUC_WARN_UNUSED_RESULT`，调用者必须检查返回值（接收到的消息数）。
- `NiceMessageExtraData` 用于接收 IP_TOS / IPV6_TCLASS 等辅助控制消息，这是 2025 年引入的新功能。

#### 包含的子模块头文件

`socket.h` 在末尾包含所有子传输的头文件，形成一个完整的公共 API：

```c
#include "udp-bsd.h"
#include "tcp-bsd.h"
#include "tcp-active.h"
#include "tcp-passive.h"
#include "pseudossl.h"
#include "socks5.h"
#include "http.h"
#include "udp-turn.h"
#include "udp-turn-over-tcp.h"
```

这意味着上层包含 `socket.h` 即可获得所有 socket 类型的构造函数声明。

---

### 文件：socket.c（487 行）

`socket.c` 实现了 `socket.h` 中声明的所有公共 API，以及 `socket-priv.h` 中声明的发送队列管理函数。

#### 虚表分派实现

每个公共 API 函数都遵循相同的分派模式：

```c
gint
nice_socket_recv_messages (NiceSocket *sock,
    NiceInputMessage *recv_messages, guint n_recv_messages,
    NiceMessageExtraData *exdata)
{
  g_return_val_if_fail (sock != NULL, -1);
  g_return_val_if_fail (n_recv_messages == 0 || recv_messages != NULL, -1);
  return sock->recv_messages (sock, recv_messages, n_recv_messages, exdata);
}
```

即：参数校验后直接调用虚表对应的函数指针。这是典型的 C 语言多态实现。

#### 可选虚表字段的处理

并非所有虚表字段都是必须实现的。以下函数检查虚表指针是否为空，空则返回默认行为：

```c
/* can_send: 未实现 → 始终返回 TRUE */
gboolean
nice_socket_can_send (NiceSocket *sock, NiceAddress *addr)
{
  if (sock->can_send)
    return sock->can_send (sock, addr);
  return TRUE;
}

/* set_writable_callback: 未实现 → 无操作 */
void
nice_socket_set_writable_callback (NiceSocket *sock,
    NiceSocketWritableCb callback, gpointer user_data)
{
  if (sock->set_writable_callback)
    sock->set_writable_callback (sock, callback, user_data);
}

/* is_based_on: 未实现 → 仅当同一指针时返回 TRUE */
gboolean
nice_socket_is_based_on (NiceSocket *sock, NiceSocket *other)
{
  if (sock->is_based_on)
    return sock->is_based_on (sock, other);
  return (sock == other);
}
```

这种设计使得简单传输类型（如 UDP-BSD）只需实现核心的三个虚表字段（`recv_messages`, `send_messages`, `is_reliable`, `close`），而复杂传输类型（如 TURN-over-TCP-over-SSL）可以实现全部虚表字段来提供完整的链式语义。

#### 单缓冲区便捷封装

`nice_socket_recv()` 和 `nice_socket_send()` 是对批量接口的封装：

```c
gssize
nice_socket_recv (NiceSocket *sock, NiceAddress *from, gsize len, gchar *buf)
{
  GInputVector local_buf = { buf, len };
  NiceInputMessage local_message = { &local_buf, 1, from, 0};
  gint ret;

  ret = sock->recv_messages (sock, &local_message, 1, NULL);
  if (ret == 1)
    return local_message.length;
  return ret;
}
```

返回值转换：批量接口返回消息数（1 或 0 或 -1），单缓冲区接口将其转换为字节数（`local_message.length` 或 0 或 -1）。

#### nice_socket_free() -- 生命周期管理

```c
void
nice_socket_free (NiceSocket *sock)
{
  if (sock) {
    sock->close (sock);          /* 虚表分派，各实现清理自己的资源 */
    g_slice_free (NiceSocket, sock);  /* 释放 NiceSocket 自身 */
  }
}
```

**注意**：libnice 不提供 `nice_socket_new()` 工厂函数。每种 socket 类型有自己的构造函数（如 `nice_udp_bsd_socket_new()`、`nice_tcp_bsd_socket_new()`），它们负责分配 `NiceSocket` 内存并填充虚表指针。这与传统的工厂模式略有不同：分发点在调用方而不是工厂函数内部。

#### isnicesocketable 标志说明

在当前代码中，**没有名为 `isnicesocketable` 的布尔标志**。数据传输的就绪性由以下机制保证：

1. **IO source / GMainLoop 驱动**：UDP 和 TCP socket 都在创建时注册了 GLib IO source，当 socket 可读时由主循环回调通知上层。
2. **`can_send` 虚表函数**：部分实现提供了发送前检查（如 UDP-TURN 需要检查 TURN 权限是否已建立）。
3. **`set_writable_callback`**：TCP 系列实现在连接建立后通过此回调通知上层可写。

#### 发送队列机制详解

`socket.c` 实现了两个层次的发送队列：

**队列构建层**：
- `nice_socket_queue_send()` -- 将多条 `NiceOutputMessage` 压缩为 `NiceSocketQueuedSend` 链表。压缩指将 scatter-gather 的多个 `GOutputVector` 缓冲区 `memcpy` 到一块连续内存中。
- `nice_socket_queue_send_with_callback()` -- 支持部分消息的入队（指定 `message_offset`），同时创建可写 IO source。用于处理部分写入后的剩余数据。

**队列排空层**：
- `nice_socket_flush_send_queue()` -- 调用 `nice_socket_send_reliable()` 逐条发送队列中的消息且不重试（假设底层可靠）。
- `nice_socket_flush_send_queue_to_socket()` -- 直接调用 GLib 的 `g_socket_send_message()` 发送到原生 GSocket。遇到 `G_IO_ERROR_WOULD_BLOCK` 时将剩余数据重新入队头部并返回 FALSE（未排空），队列为空时返回 TRUE（排空完毕）。

---

### 具体实现的虚表模式示例

以 `tcp-bsd.c` 的构造函数为例，展示虚表如何填充：

```c
NiceSocket *
nice_tcp_bsd_socket_new_from_gsock (GMainContext *ctx, GSocket *gsock,
    NiceAddress *local_addr, NiceAddress *remote_addr, gboolean reliable)
{
  NiceSocket *sock;
  TcpPriv *priv;

  sock = g_slice_new0 (NiceSocket);      /* 分配基类内存 */
  sock->priv = priv = g_slice_new0 (TcpPriv);  /* 分配私有数据 */

  /* 填充基础属性 */
  sock->type = NICE_SOCKET_TYPE_TCP_BSD;
  sock->fileno = g_object_ref (gsock);
  sock->addr = *local_addr;

  /* 填充虚表 -- 这是多态的关键 */
  sock->send_messages = socket_send_messages;
  sock->send_messages_reliable = socket_send_messages_reliable;
  sock->recv_messages = socket_recv_messages;
  sock->is_reliable = socket_is_reliable;
  sock->can_send = socket_can_send;
  sock->set_writable_callback = socket_set_writable_callback;
  sock->close = socket_close;

  return sock;
}
```

`udp-bsd.c` 的构造函数结构完全相同，差异仅在于私有数据结构体类型（`UdpBsdSocketPrivate` vs `TcpPriv`）和具体的函数实现。

---

### NiceMessageExtraData -- 扩展数据机制

```c
struct _NiceMessageExtraData
{
  GSocketControlMessage *tos;  /* IP_TOS / IPV6_TCLASS 控制消息 */
};
```

该结构体在 `agent/agent-priv.h` 中定义，用于在批量接收接口 `nice_socket_recv_messages()` 中获取辅助数据。这是 libnice 近期提交引入的功能，使上层能够读取收到的 IP 数据包的服务类型（TOS）字段。

使用方式：调用者提供与 `recv_messages` 数组等长的 `NiceMessageExtraData` 数组，底层协议实现（如 `udp-bsd.c`）在接收数据时将 `GSocketControlMessage` 填充到对应位置。

---

## 调用关系

```
agent/component.c (I/O 回调: component_io_cb)
  │
  ├── nice_socket_recv_messages()  ──────► sock->recv_messages()
  ├── nice_socket_send_messages()  ──────► sock->send_messages()
  ├── nice_socket_send_messages_reliable() ──► sock->send_messages_reliable()
  ├── nice_socket_is_reliable()     ────► sock->is_reliable()
  ├── nice_socket_can_send()        ────► sock->can_send()
  ├── nice_socket_is_based_on()     ────► sock->is_based_on()
  └── nice_socket_free()            ────► sock->close()
         │
         └── socket/ 具体实现模块：
               ├── udp-bsd.c          (UDP 原生收发 + IP_TOS/TCLASS 接收)
               ├── udp-turn.c         (UDP-over-TURN 中继)
               ├── udp-turn-over-tcp.c (TCP 承载的 TURN 中继)
               ├── tcp-bsd.c          (TCP 原生收发 + 可靠队列)
               ├── tcp-active.c       (主动 TCP: connect + 升级为 tcp-bsd)
               ├── tcp-passive.c      (被动 TCP: accept + 升级为 tcp-bsd)
               ├── socks5.c           (SOCKS5 代理隧道)
               ├── http.c             (HTTP CONNECT 代理隧道)
               └── pseudossl.c        (SSL/TLS 加密封装，装饰器模式)

agent/discovery.c (候选收集: gather_candidates)
  │
  ├── nice_udp_bsd_socket_new()          → 主机候选（host）
  ├── nice_tcp_active_socket_new()       → TCP 主动主机候选
  ├── nice_tcp_passive_socket_new()      → TCP 被动主机候选（0 端口绑定）
  └── nice_udp_turn_socket_new()         → TURN 服务器反射候选（srflx/relay）

agent/agent.c (agent 初始化: _nice_agent_init())
  │
  └── nice_udp_turn_over_tcp_socket_new() → TURN-over-TCP 候选
         (组合使用 tcp-bsd + pseudossl + http 代理层)
```

### 调用流向总结

1. **上层入口**：`agent/component.c` 通过 IO 回调进入，获取 `NiceSocket*` 指针，调用 `nice_socket_recv_messages()` 接收数据或 `nice_socket_send_messages()` 发送数据。
2. **虚表分派**：`socket/socket.c` 进行参数校验后直接调用 `sock->recv_messages` 等虚表指针，无任何中间层。
3. **具体实现**：虚表函数在具体模块中实现，直接操作底层 GSocket 或执行协议逻辑（如 TURN 封装、SSL 加解密）。
4. **构造阶段**：`agent/discovery.c` 和 `agent/agent.c` 根据候选类型直接调用各子模块的构造函数。

### 包装/装饰链示例

TURN-over-TCP-over-SSL 的完整 socket 层级栈（由 `agent/agent.c` 构建）：

```
NiceSocket (UDP_TURN_OVER_TCP)  ← 最外层，提供 UDP 语义
  └── is_based_on →
      NiceSocket (PSEUDOSSL)    ← 加密层
        └── is_based_on →
            NiceSocket (HTTP)   ← HTTP 代理隧道（如果有代理）
              └── is_based_on →
                  NiceSocket (TCP_BSD)  ← 最内层，实际读写 TCP 字节流
```

上层组件调用 `nice_socket_is_based_on()` 可以递归遍历这条链，判断一个 socket 是否基于另一个（用于 ICE 去重逻辑，避免为同一底层 socket 创建多个候选）。

---

## 第二部分：UDP 传输实现

### udp-bsd.c / udp-bsd.h (597 行) -- 原生 UDP Socket

#### 概述

`udp-bsd.c` 实现了基于 BSD socket API 的原始 UDP 通信，是 libnice 中最简单、最高效的传输层。其职责是：创建 UDP socket、绑定地址、收发数据报、提供可写通知。该文件实现了 NiceSocket 虚表的全部 7 个函数指针，是研究虚表实现模式的最佳入口。

#### 内部数据结构

```c
struct UdpBsdSocketPrivate
{
  GMainContext *context;              /* 主上下文，用于 GLib IO source 附加 */
  NiceSocketWritableCb writable_cb;   /* 可写回调函数指针 */
  gpointer writable_data;             /* 可写回调的用户数据 */
  GMutex mutex;                       /* 保护以下字段 */
  NiceAddress niceaddr;               /* 缓存的上次发送目标地址 */
  GSocketAddress *gaddr;              /* 缓存的 GSocketAddress（由 niceaddr 构造） */
  GSource *io_source;                 /* 可写 IO source（异步通知用） */
};
```

#### 准确函数原型

| 函数 | 原型 |
|------|------|
| 构造函数 | `NiceSocket *nice_udp_bsd_socket_new (GMainContext *ctx, NiceAddress *addr, gboolean recv_tos, GError **error)` |

#### 核心函数分析

##### nice_udp_bsd_socket_new()

构造函数执行以下步骤：

1. **地址解析**：将 `NiceAddress` 转换为 `sockaddr_storage`，判断是 IPv4 (`AF_INET`) 还是 IPv6 (`AF_INET6`)，创建对应协议族的 `GSocket`（`G_SOCKET_TYPE_DATAGRAM` / `G_SOCKET_PROTOCOL_UDP`）。
2. **接口绑定**：如果系统支持 `IP_UNICAST_IF` / `IPV6_UNICAST_IF`（Linux），会通过 `nice_interfaces_get_if_index_by_addr()` 查找地址对应的接口索引并设置 socket option，确保多宿主机器上绑定正确的网卡。
3. **TOS 接收开关**：`recv_tos` 参数控制是否启用 IP_TOS / IPV6_TCLASS 辅助数据接收。IPv4 设置 `IP_RECVTOS`，IPv6 设置 `IPV6_RECVTCLASS`，需要 GLib >= 2.88 支持。
4. **非阻塞模式**：`g_socket_set_blocking (gsock, false)` 强制非阻塞。
5. **绑定**：`g_socket_bind()` 将 socket 绑定到指定地址和端口。
6. **获取实际地址**：通过 `g_socket_get_local_address()` 获取 OS 分配的端口（若传入 `addr = 0`），回填 `sock->addr`。
7. **虚表填充**：按标准模式填充所有 7 个虚表指针。

##### socket_recv_messages()

虚表 `recv_messages` 的实现，核心流程：

```c
static gint socket_recv_messages(NiceSocket *sock,
    NiceInputMessage *recv_messages, guint n_recv_messages,
    NiceMessageExtraData *exdata)
```

1. **批量接收循环**：遍历 `recv_messages` 数组，调用 `g_socket_receive_message()` 逐个接收。
2. **辅助数据提取**：当 `exdata != NULL` 且平台支持时，GSocket 返回的 control messages 中若包含 `G_IS_IP_TOS_MESSAGE` 或 `G_IS_IPV6_TCLASS_MESSAGE`，则 `g_object_ref()` 后存入 `exdata->tos`。
3. **发送方地址**：通过 `gaddr` 获取，转换为 `NiceAddress` 回填到 `recv_message->from`。
4. **错误处理**：`G_IO_ERROR_WOULD_BLOCK` 和 `G_IO_ERROR_CONNECTION_CLOSED` 视为正常（recvd = 0），`G_IO_ERROR_MESSAGE_TOO_LARGE` 视为数据截断（截取已有大小），其余为真正的错误。
5. **提前返回**：遇到 recvd <= 0 时跳出循环，返回已成功接收的消息数。

##### socket_send_messages()

虚表 `send_messages` 的实现：

```c
static gint socket_send_messages(NiceSocket *sock, const NiceAddress *to,
    const NiceOutputMessage *messages, guint n_messages)
```

关键设计：

1. **地址缓存优化**：使用 mutex 保护的 `priv->niceaddr` 和 `priv->gaddr` 缓存上次发送目标地址。如果当前目标与缓存相同，直接复用 `GSocketAddress` 对象（`g_object_ref`），避免重复的内存分配和地址转换。
2. **单消息快速路径**：当 `n_messages == 1` 时，直接调用 `g_socket_send_message()`。
3. **多消息高效路径**：当 `n_messages >= 2` 时，构造 `GOutputMessage` 数组并调用 `g_socket_send_messages()`，底层可能在支持的系统上使用 `sendmmsg()` 系统调用（一次系统调用发送多条消息）。
4. **WOULD_BLOCK 处理**：遇到 `G_IO_ERROR_WOULD_BLOCK` 时，创建 `G_IO_OUT` 类型的 IO source 并附加到主上下文。当 socket 变为可写时，`_udp_bsd_io_callback` 回调触发，清理 source 并调用用户的 `writable_cb` 通知上层可继续发送。

##### socket_send_messages_reliable()

```c
static gint socket_send_messages_reliable(NiceSocket *sock,
    const NiceAddress *to, const NiceOutputMessage *messages,
    guint n_messages)
{
  return -1;
}
```

**直接返回 -1**。UDP 是不可靠传输，不支持可靠发送语义。这个设计是合理的：调用者（`agent/component.c`）应先通过 `nice_socket_is_reliable()` 判断再决定是否调用此函数。

##### socket_is_reliable()

返回 `FALSE`。UDP 不保证送达。

##### socket_can_send()

返回 `TRUE`。UDP socket 无需 TURN 权限检查，始终可发送。

##### socket_set_writable_callback()

保存回调函数指针和用户数据到 `priv` 中。回调在 `_udp_bsd_io_callback` 中被调用。

##### socket_close()

清理流程：
1. `g_clear_object(&priv->gaddr)` 释放缓存的 GSocketAddress。
2. `g_mutex_clear(&priv->mutex)` 销毁互斥锁。
3. 销毁并释放 io_source（如果存在）。
4. 取消主上下文引用。
5. 释放 `UdpBsdSocketPrivate` 和 `NiceSocket` 内存。
6. 关闭底层 GSocket：`g_socket_close()` + `g_object_unref()`。

#### 关键设计：单端口复用

这是 libnice ICE 实现的关键优化：

```
多个 UDP component (RTP + RTCP) 共享同一个绑定到 INADDR_ANY 的 UDP socket
├── component 1 (RTP)  ──► NiceSocket (host candidate)  ──► UDP socket fd=5
├── component 2 (RTCP) ──► NiceSocket (host candidate)  ──► UDP socket fd=5
└── TURN srflx         ──► NiceSocket (server-reflexive) ──► UDP socket fd=5
```

服务器反射候选和主机候选使用相同的底层 BSD socket，因为 STUN 绑定请求从同一端口发出，NAT 映射后反射回的地址自然与 host 候选同端口。

---

### udp-turn.c / udp-turn.h (2377 行) -- TURN UDP 传输

#### 概述

`udp-turn.c` 是 `socket/` 模块中最大的文件（2291 行 C 代码 + 86 行头文件），实现了完整的 TURN 客户端生命周期。其职责是通过 TURN 服务器中继 UDP 数据，支持 5 种兼容模式：RFC5766（标准）、DRAFT9（草案）、GOOGLE（Google 非标准）、MSN（Microsoft TURN）、OC2007（Office Communications 2007）。

该文件不仅实现 NiceSocket 虚表接口，还包含：
- TURN 分配（Allocation）管理
- 权限（Permission）生命周期
- 信道（Channel Binding）绑定管理
- 发送/接收数据的分派逻辑
- 保活和刷新机制
- 定时器驱动的重传逻辑

#### 核心数据结构

```c
typedef struct {
  StunMessage message;          /* STUN 消息 */
  uint8_t buffer[STUN_MAX_MESSAGE_SIZE]; /* 序列化缓冲区 */
  StunTimer timer;              /* 重传定时器 */
} TURNMessage;

typedef struct {
  NiceAddress peer;             /* 对端地址 */
  uint16_t channel;             /* 信道号 (0x4000-0xFFFF) */
  gboolean renew;               /* 是否需要刷新 */
  GSource *timeout_source;      /* 超时 GSource */
} ChannelBinding;

/* 用于存储等待权限建立期间的发送数据 */
typedef struct {
  gchar *data;
  guint data_len;
  gboolean reliable;
} SendData;

typedef struct {
  StunTransactionId id;         /* 事务 ID */
  GSource *source;              /* 超时取消源 */
  UdpTurnPriv *priv;
} SendRequest;

typedef struct {
  GMainContext *ctx;
  StunAgent agent;                    /* STUN 代理（事务层） */
  GList *channels;                    /* ChannelBinding 链表 */
  GList *pending_bindings;            /* 待处理的信道绑定请求 */
  ChannelBinding *current_binding;    /* 当前进行中的绑定 */
  TURNMessage *current_binding_msg;   /* 当前绑定 STUN 消息 */
  GList *pending_permissions;         /* 待处理的 CreatePermission */
  GSource *tick_source_channel_bind;  /* 绑定重传定时器源 */
  GSource *tick_source_create_permission; /* 权限重传定时器源 */
  NiceSocket *base_socket;            /* 底层 socket（udp-bsd 或 tcp-bsd） */
  NiceAddress server_addr;            /* TURN 服务器地址 */
  uint8_t *username;                  /* 认证用户名 */
  gsize username_len;
  uint8_t *password;                  /* 认证密码 */
  gsize password_len;
  NiceTurnSocketCompatibility compatibility; /* 兼容模式 */
  GQueue *send_requests;              /* 待确认的 Send 请求队列 */
  GList *permissions;                 /* 已安装的权限（对端地址列表） */
  GList *sent_permissions;            /* 正在进行中的权限安装 */
  GHashTable *send_data_queues;       /* 每对端的数据发送队列 */
  GSource *permission_timeout_source; /* 权限过期定时器 */
  uint8_t *cached_realm;             /* 缓存的 realm（认证用） */
  uint16_t cached_realm_len;
  uint8_t *cached_nonce;             /* 缓存的 nonce（认证用） */
  uint16_t cached_nonce_len;
  GByteArray *fragment_buffer;        /* RFC4571 帧重组缓冲（可靠模式） */
  NiceAddress from;                   /* 分片缓冲时的源地址 */
  uint8_t *send_buffer;               /* 发送临时缓冲区 */
  /* MS-TURN 特有 */
  uint8_t ms_realm[STUN_MAX_MS_REALM_LEN + 1];
  uint8_t ms_connection_id[20];
  uint32_t ms_sequence_num;
  bool ms_connection_id_valid;
} UdpTurnPriv;
```

#### 准确函数原型（从 udp-turn.h）

```c
NiceSocket *nice_udp_turn_socket_new (GMainContext *ctx, NiceAddress *addr,
    NiceSocket *base_socket, const NiceAddress *server_addr,
    const gchar *username, const gchar *password,
    NiceTurnSocketCompatibility compatibility);

guint nice_udp_turn_socket_parse_recv_message (NiceSocket *sock,
    NiceSocket **from_sock, NiceInputMessage *message);

gsize nice_udp_turn_socket_parse_recv (NiceSocket *sock, NiceSocket **from_sock,
    NiceAddress *from, gsize len, guint8 *buf,
    const NiceAddress *recv_from, const guint8 *recv_buf, gsize recv_len);

gboolean nice_udp_turn_socket_set_peer (NiceSocket *sock, NiceAddress *peer);

void nice_udp_turn_socket_cache_realm_nonce (NiceSocket *sock, StunMessage *msg);

void nice_udp_turn_socket_set_ms_realm (NiceSocket *sock, StunMessage *msg);

void nice_udp_turn_socket_set_ms_connection_id (NiceSocket *sock, StunMessage *msg);
```

#### TURN 构造函数分析

##### nice_udp_turn_socket_new()

1. **STUN 代理初始化**：根据兼容模式选择不同的 STUN 代理配置：
   - `DRAFT9` / `RFC5766`：`STUN_COMPATIBILITY_RFC5389` + `LONG_TERM_CREDENTIALS`
   - `MSN`：`STUN_COMPATIBILITY_RFC3489` + `SHORT_TERM_CREDENTIALS` + `NO_INDICATION_AUTH`
   - `GOOGLE`：`STUN_COMPATIBILITY_RFC3489` + `SHORT_TERM_CREDENTIALS` + `IGNORE_CREDENTIALS`
   - `OC2007`：`STUN_COMPATIBILITY_OC2007` + `LONG_TERM_CREDENTIALS` + `NO_ALIGNED_ATTRIBUTES`
2. **凭据处理**：MSN/OC2007 模式下对 username/password 进行 Base64 解码；其他模式直接使用明文。
3. **数据队列**：创建 `send_data_queues`（按对端地址哈希的发送队列表），用于缓存等待权限建立时的发送数据。
4. **虚表填充**：注意 `sock->fileno = NULL`（TURN socket 没有自己的文件描述符，所有 I/O 皆通过 `base_socket` 间接进行）。
5. **注意**：构造函数不发送 TURN Allocate 请求。Allocate 由上层（`agent/discovery.c` 或 `agent/agent.c`）在 TURN candidate 发现阶段通过 STUN usage 层发起。

#### 数据发送路径

`socket_send_message()` 是核心发送分派函数，负责判断应该使用哪种封装方式：

```c
static gssize socket_send_message(NiceSocket *sock, const NiceAddress *to,
    const NiceOutputMessage *message, gboolean reliable)
```

**分派逻辑**（从内到外）：

```
socket_send_message(to, message)
  │
  ├── 已存在该 to 的 ChannelBinding？
  │   ├── YES (RFC5766/DRAFT9 模式):
  │   │   构造 ChannelData 帧:
  │   │   ┌────────────────────────────────┐
  │   │   │ channel number (2B) │ len (2B) │ data ...
  │   │   └────────────────────────────────┘
  │   │   仅 4 字节开销！直接 memcpy 到 send_buffer
  │   │   然后通过 base_socket 发出
  │   │
  │   └── YES (非 RFC5766 模式 => GOOGLE/MSN/OC2007):
  │       直接通过 base_socket 转发（这些协议只在接收端有信道概念）
  │
  └── NO ChannelBinding：
       │
       ├── RFC5766/DRAFT9 => Send Indication (STUN_IND_SEND)：
       │   构造 STUN 消息:
       │     stun_agent_init_indication(IND_SEND)
       │     → stun_message_append_xor_addr(XOR_PEER_ADDRESS, to)
       │     → stun_message_append_bytes(DATA, compacted_payload)
       │     → stun_agent_finish_message()
       │   开销：36 字节（STUN 头 20B + XOR_PEER_ADDRESS 属性 ~12B + DATA 属性头 4B）
       │
       └── GOOGLE/MSN/OC2007 => Send Request：
           构造 STUN 请求:
             stun_agent_init_request(SEND)
             → 附加 MAGIC_COOKIE, USERNAME, DESTINATION_ADDRESS
             → stun_agent_finish_message()
           然后进入 send_requests 队列等待响应
```

**ChannelData 4 字节 vs Send Indication 36 字节的权衡**：

| 封装方式 | 开销 | 使用场景 |
|----------|------|----------|
| ChannelData | 4 字节 | 高频数据传输；需要先通过 ChannelBind 建立绑定 |
| Send Indication | ~36 字节 | 低频/首次发送；无需预先绑定 |

对于 RTP 媒体流，32 字节/包的节省在大流量下非常显著。这就是为什么 ICE 连接检查完成后，libnice 会通过 `nice_udp_turn_socket_set_peer()` 主动建立 Channel Binding。

**额外的 TURN 封装层**：

在 `socket_send_message()` 构造好 TURN 消息后，实际发送通过 `_socket_send_messages_wrapped()` 进行：

- **不可靠 base_socket（udp-bsd）**：直接用 `nice_socket_send_messages()` 发出。
- **可靠 base_socket（tcp-bsd）**：在 TURN 消息前加上 RFC4571 帧头（2 字节长度，`htons(message_len)`），然后通过可靠发送发出。这是 ICE-TCP over TURN 的关键适配。

**权限检查**：

```c
if (priv->compatibility == NICE_TURN_SOCKET_COMPATIBILITY_RFC5766 &&
    !priv_has_permission_for_peer(priv, to)) {
    // 先发送 CreatePermission
    if (!priv_has_sent_permission_for_peer(priv, to)) {
        priv_send_create_permission(priv, to);
    }
    // 数据入队等待
    socket_enqueue_data(priv, to, msg_len, buffer, reliable);
}
```

RFC5766 要求：启动 `Send Indication` 前必须先安装 Permission。数据在权限建立前被缓存在 `send_data_queues` 中。

**可靠发送限制**：

```c
// socket_send_messages_reliable 中的检查
if (priv->base_socket->type == NICE_SOCKET_TYPE_UDP_BSD) {
    return -1;  // TURN over UDP 不支持可靠发送
}
```

可靠发送仅在 TURN-over-TCP 场景下可用（因为只有 TCP 能保证送达）。

#### 数据接收路径

`socket_recv_messages()` 和 `nice_udp_turn_socket_parse_recv()` 协同工作：

**接收流程**：

```
socket_recv_messages()
  │
  ├── 1. 首先处理 fragment_buffer 中缓存的数据
  │      (TCP 可靠模式下 RFC4571 帧可能被分散在多次 recv 中)
  │
  ├── 2. 从 base_socket 接收原始数据
  │
  └── 3. 对每条消息调用 nice_udp_turn_socket_parse_recv()
         │
         ├── 可靠模式？剥离 RFC4571 帧头 (2字节)
         │
         ├── 来源是 TURN server？
         │   │
         │   ├── STUN 消息验证通过？
         │   │   │
         │   │   ├── IND_DATA (Data Indication)：
         │   │   │   从 XORMAPPEDADDRESS / REMOTEADDRESS 提取 peer addr
         │   │   │   从 DATA 属性提取原始数据
         │   │   │   memmove 到输出缓冲区
         │   │   │   → 返回数据长度（> 0）
         │   │   │
         │   │   ├── SEND (Send Response)：
         │   │   │   匹配 send_requests 中的请求，释放请求记录
         │   │   │   GOOGLE 模式：检查 OPTIONS 标志，可能触发信道锁定
         │   │   │   → 返回 0（控制消息，已消费）
         │   │   │
         │   │   ├── CHANNELBIND (ChannelBind Response)：
         │   │   │   匹配 current_binding_msg，处理成功/失败
         │   │   │   失败：处理 UNAUTHORIZED/STALE_NONCE → 缓存 realm/nonce 并重试
         │   │   │   成功：将 binding 添加到 channels 列表，启动刷新定时器
         │   │   │   → 返回 0（控制消息）
         │   │   │
         │   │   ├── CREATEPERMISSION (CreatePermission Response)：
         │   │   │   匹配 pending_permissions 列表
         │   │   │   成功：添加到 permissions 列表，dequeue 缓存数据
         │   │   │   失败：同 ChannelBind 的错误处理逻辑
         │   │   │   → 返回 0（控制消息）
         │   │   │
         │   │   └── OLD_SET_ACTIVE_DST (MS-TURN)：
         │   │       匹配 current_binding_msg
         │   │       成功：触发信道锁定 → 返回 0
         │   │
         │   └── STUN 验证失败 → 进入 recv 标签
         │
         └── 来源不是 TURN server？
             │
             └── 在 channels 列表中查找匹配的 ChannelBinding
                 (RFC5766: 通过 channel number 匹配; 非 RFC5766: 第一个即匹配)
                 │
                 ├── 找到 binding：
                 │   from = binding->peer
                 │   RFC5766: 剥离 4 字节 ChannelData 头，memmove 数据
                 │   → 返回数据长度（> 0）
                 │
                 └── 未找到：
                     from = recv_from (保留原始来源)
                     memmove 原始数据
                     → 返回数据长度（> 0）
```

##### socket_can_send()

委托给 `priv->base_socket` 的 `can_send()`。这意味着 TURN socket 的可发送性取决于底层传输的状态：UDP 始终可发送；TCP 需要连接已建立。

##### socket_is_based_on()

递归遍历包装链：

```c
return (sock == other) ||
    (priv && nice_socket_is_based_on(priv->base_socket, other));
```

例如 `nice_socket_is_based_on(turn_sock, udp_bsd_sock)` 返回 TRUE，因为 turn_sock 底层包装了 udp_bsd_sock。

#### 权限（Permission）管理

RFC5766 要求：每个想要通过 TURN 中继发送数据的对端地址，都需要先安装 Permission。

**安装流程**：

```
priv_send_create_permission(peer)
  │
  ├── 1. 注册 peer 为 sent_permission（防重复发送）
  │
  ├── 2. 通过 STUN usage 层构造 CreatePermission 请求
  │     stun_usage_turn_create_permission() 自动处理认证
  │
  ├── 3. 发送请求（先尝试可靠发送，失败则非可靠发送）
  │
  ├── 4. 启动重传定时器
  │     - 可靠模式: STUN_TIMER_DEFAULT_RELIABLE_TIMEOUT
  │     - 非可靠模式: STUN_TIMER_DEFAULT_TIMEOUT + DEFAULT_MAX_RETRANSMISSIONS
  │
  └── 5. 添加到 pending_permissions 列表
```

**响应处理**（在 `nice_udp_turn_socket_parse_recv()` 中）：

- **成功 (`STUN_RESPONSE`)**：将 peer 从 `sent_permissions` 移到 `permissions`，排空该 peer 的发送队列（`socket_dequeue_all_data`）。首次成功时安装 `permission_timeout_source`（240 秒后刷新）。
- **认证失败 (`STUN_ERROR`)**：缓存新的 realm/nonce，重新发送 CreatePermission。
- **超时**（`priv_retransmissions_create_permission_tick_unlocked`）：达到最大重试次数后，假定服务器不支持权限（回退），仍然将 peer 加入 `permissions` 并排空队列，让连接检查自行决定成败。

**刷新机制**：

- `permission_timeout_source` 在 240 秒（`STUN_PERMISSION_TIMEOUT`）后触发 `priv_permission_timeout()`。
- 刷新时清除全部 `permissions`。下次发送数据时会触发新的 CreatePermission。

#### 信道绑定（Channel Binding）

**信道号分配**（RFC5766/DRAFT9）：

```c
uint16_t channel = 0x4000;
// 遍历已有 channels，找到第一个未使用的信道号
for (i = priv->channels; i; i = i->next) {
    if (channel == b->channel) {
        channel++;
        i = priv->channels; // 重新检查
    }
}
```

RFC5766 规范要求信道号范围 `0x4000-0x7FFF`。代码中使用了更大范围 `0x4000-0xFFFF`。

**绑定流程**：

1. `priv_add_channel_binding()` 被调用（来自 `nice_udp_turn_socket_set_peer()` 或内部 pending 处理）。
2. 如果当前有活跃绑定（`priv->current_binding != NULL`），则将请求加入 `pending_bindings` 队列等待。
3. 否则：
   - **RFC5766/DRAFT9**：分配信道号，调用 `priv_send_channel_bind()` 发送 ChannelBind Request，设置 `current_binding` 为临时对象。
   - **MSN/OC2007**：发送 `OLD_SET_ACTIVE_DST` 请求（Microsoft 特定）。
   - **GOOGLE**：无需发送请求，直接设置 `current_binding`（Google TURN 无显式绑定）。
4. `priv_send_turn_message()` 发送并启动重传定时器。

**刷新机制**：

- 成功绑定的 channel 在 540 秒（`STUN_BINDING_TIMEOUT`）后触发 `priv_binding_timeout()`，设置 `b->renew = TRUE`。
- 同时安装 60 秒（`STUN_EXPIRE_TIMEOUT`）后的到期定时器 `priv_binding_expired_timeout`。
- 如果到期前未刷新成功，binding 被移除，peer 重新加入 pending 队列。

#### 重传和定时器系统

TURN 协议需要可靠的消息传输（ChannelBind、CreatePermission）。libnice 使用两层定时器：

1. **ChannelBind 重传**：`tick_source_channel_bind` + `priv_retransmissions_tick()`
2. **CreatePermission 重传**：`tick_source_create_permission` + `priv_retransmissions_create_permission_tick()`

调度策略（`priv_schedule_tick()`）：

```
priv_schedule_tick()
  │
  ├── 检查 current_binding_msg 的 StunTimer
  │   如果 time_remaining > 0：创建定时器等待
  │   如果 time_remaining == 0：立即重传
  │
  └── 遍历所有 pending_permissions
      为每个计算 StunTimer 剩余时间
      取最小值 min_timeout
      创建单个定时器（在 min_timeout 后触发，一次处理所有到期项）
```

重传结果处理：

- `STUN_USAGE_TIMER_RETURN_RETRANSMIT`：重新发送消息，保留定时器。
- `STUN_USAGE_TIMER_RETURN_TIMEOUT`：重试耗尽。
  - ChannelBind：释放 current_binding，`priv_process_pending_bindings()` 处理下一个。
  - CreatePermission：假定服务器不支持权限，回退处理。
- `STUN_USAGE_TIMER_RETURN_SUCCESS`：消息得到响应，正常流程已处理。

#### RFC4571 帧处理（可靠模式）

当 `base_socket` 是可靠的（TCP）时，还需处理 TCP 字节流的帧边界问题：

**接收端**：TURN 消息在 TCP 流中由 RFC4571 帧封装（2 字节长度 + STUN 数据）。TCP 的 recv 可能返回不完整的帧，这时需要 `fragment_buffer`（`GByteArray`）来缓存和重组。

**发送端**：`_socket_send_messages_wrapped()` 检查 `nice_socket_is_reliable()`，如果是可靠传输则自动添加 RFC4571 帧头（`htons(message_len)`）。

---

### udp-turn-over-tcp.c / udp-turn-over-tcp.h (523 行) -- TURN over TCP

#### 概述

`udp-turn-over-tcp.c` 实现了通过 TCP 连接使用 TURN 服务的适配层。其职责是：在 TCP 字节流上提供 TURN 消息的帧边界处理（RFC4571 帧协议），使得上层（`udp-turn.c`）可以像操作 UDP 一样操作 TCP 上的 TURN 消息。这是一个相对轻量的适配层（469 行 C 代码），核心逻辑是帧边界的解析和组装。

#### 内部数据结构

```c
typedef struct {
  NiceTurnSocketCompatibility compatibility;
  union {
    guint8 u8[65536];        /* 接收缓冲区（64KB，TURN 最大消息尺寸） */
    guint16 u16[32768];
  } recv_buf;
  gsize recv_buf_len;         /* 接收缓冲区已使用长度 */
  guint expecting_len;        /* 当前期待的帧完整长度 */
  NiceSocket *base_socket;    /* 底层 TCP socket */
} TurnTcpPriv;
```

#### 准确函数原型

```c
NiceSocket *nice_udp_turn_over_tcp_socket_new (NiceSocket *base_socket,
    NiceTurnSocketCompatibility compatibility);
```

#### 构造函数

```c
NiceSocket *nice_udp_turn_over_tcp_socket_new(
    NiceSocket *base_socket,
    NiceTurnSocketCompatibility compatibility)
```

1. **包装 base_socket**：不创建自己的 GSocket，而是直接引用 `base_socket->fileno` 和 `base_socket->addr`。
2. **虚表填充**：与 `udp-bsd.c` 一致的模式，但额外实现 `is_based_on`（用于链式遍历）。
3. **注意**：构造函数不建立 TCP 连接。连接由上层在构建包装链时通过 `tcp-active` 或 `tcp-passive` 建立，最终以 TCP socket 传入。

#### 与 udp-turn.c 的关键差异

| 特性 | udp-turn.c | udp-turn-over-tcp.c |
|------|-----------|---------------------|
| 底层传输 | UDP（数据报） | TCP（字节流） |
| 帧边界 | 天然存在（每个 UDP 数据报即一个消息） | 需要 RFC4571 2 字节长度前缀 |
| uint8_t send_buffer | 每消息分配（TURNMessage.buffer） | 无（直接使用 base_socket） |
| fileno | NULL | 指向 base_socket->fileno |
| 接收缓冲区 | 无（逐消息处理） | 64KB recv_buf + 分帧逻辑 |
| is_based_on | 实现（递归到 base_socket） | 实现（递归到 base_socket） |
| 包头处理 | STUN 直接解析 | 先读帧头，再读 STUN 体 |

#### TCP 帧边界处理

TCP 是流式协议，没有消息边界。TURN over TCP 使用 RFC4571 标准解决此问题：

```
┌──────────────┬──────────────────────────────────────────┐
│ length (2B)  │ TURN/STUN message body (length bytes)   │
└──────────────┴──────────────────────────────────────────┘
```

`socket_recv_message()` 实现了一个两阶段状态机：

**阶段 1 -- 读取帧头**（`expecting_len == 0`）：

```c
if (priv->expecting_len == 0) {
    // 根据兼容模式确定头部长度
    headerlen = (DRAFT9|RFC5766|OC2007) ? 4 : (GOOGLE ? 2 : 0);

    // 从 base_socket 读取直到凑齐 headerlen 字节
    ret = nice_socket_recv_messages(priv->base_socket, ...);
    priv->recv_buf_len += ret;

    if (priv->recv_buf_len < headerlen)
        return 0;  // 还需要更多数据
}
```

头部解析（根据兼容模式）：

- **DRAFT9 / RFC5766**：读取 4 字节。前 2 字节是 "magic" 字段。如果 `< 0x4000`，是 STUN 消息，期望长度 = `20 + packetlen`（STUN 头 20 字节）。如果 `>= 0x4000`，是 ChannelData 消息，期望长度 = `4 + packetlen`（ChannelData 头 4 字节）。最后还需对齐到 4 字节边界（`padlen = (expecting_len % 4) ? 4 - (expecting_len % 4) : 0`）。
- **GOOGLE**：读取 2 字节长度。`expecting_len = compat_len`。
- **OC2007**：读取 4 字节。检查第一个字节的 payload 类型（`MS_TURN_CONTROL_MESSAGE = 2` 或 `MS_TURN_END_TO_END_DATA = 3`）。保留 RFC4571 帧格式让 NiceAgent 解帧。

**阶段 2 -- 读取完整消息**（`expecting_len > 0`）：

```c
// 从 base_socket 读取补齐 expecting_len + padlen 字节
ret = nice_socket_recv_messages(priv->base_socket, ...);
priv->recv_buf_len += ret;

if (priv->recv_buf_len == priv->expecting_len + padlen) {
    // 帧完整！memcpy 到输出缓冲区
    memcpy_buffer_to_input_message(recv_message, priv->recv_buf.u8, priv->recv_buf_len);
    priv->expecting_len = 0;  // 重置状态机，准备下一个帧
    priv->recv_buf_len = 0;
}
```

#### 发送端帧封装

`socket_send_message()` 在发送前根据兼容模式添加帧头：

- **GOOGLE**：添加 2 字节长度前缀（`htons(message_size)`）。
- **DRAFT9 / RFC5766**：添加 4 字节对齐的 padding（因为 STUN 消息已自带 STUN 头，不需要再添加 RFC4571 帧头；TURN-over-TCP 的 STUN 直接承载在 TCP 流的 STUN 帧中）。
- **OC2007**：检查消息中是否包含 STUN Magic Cookie（在偏移 24 字节处）。如果包含 → `MS_TURN_CONTROL_MESSAGE`，否则 → `MS_TURN_END_TO_END_DATA`。添加 2 字节头（`pt + zero`）。
- **MSN**：不添加帧头。

#### 兜底机制

ICE 规范（RFC 6544）要求：当 UDP 候选不可用时，可以通过 TCP 执行 TURN 分配，从而获得 UDP-over-TURN 的连通性。`udp-turn-over-tcp.c` 就是这个兜底机制的核心。

在 libnice 中，该 socket 被用作整个 TURN over TCP 协议栈的"外壳"，内部可以包装多层：
```
NiceSocket (UDP_TURN_OVER_TCP)  ← 最外层外壳
  └── udp-turn (内部逻辑复用)   ← TURN 协议逻辑
       └── tcp-active/passive   ← TCP 传输
```

#### 虚表函数特点

几乎所有虚表函数都采用**委托模式**，转发给 `priv->base_socket`：

| 虚表函数 | 实现方式 |
|----------|----------|
| `recv_messages` | 自实现：两阶段帧解析 + 委托 base_socket |
| `send_messages` | 自实现：添加帧头 + 委托 base_socket |
| `send_messages_reliable` | 自实现：添加帧头 + 委托 base_socket（可靠发送） |
| `is_reliable` | 委托：`nice_socket_is_reliable(priv->base_socket)` |
| `can_send` | 委托：`nice_socket_can_send(priv->base_socket, addr)` |
| `set_writable_callback` | 委托：`nice_socket_set_writable_callback(priv->base_socket, cb, data)` |
| `is_based_on` | 递归：`sock == other \|\| nice_socket_is_based_on(priv->base_socket, other)` |
| `close` | 清理 priv + `nice_socket_free(priv->base_socket)` |

---

## 第三部分：TCP 传输实现

### tcp-bsd.c / tcp-bsd.h (555 行) -- 基础 TCP Socket

#### 概述

`tcp-bsd.c` 实现了基于 BSD socket API 的基础 TCP 通信，是 TCP 系列 socket 的"叶子节点" -- 所有其他 TCP 类型（tcp-active、tcp-passive、TCP_SO）最终都会创建一个 tcp-bsd 实例来承载实际的数据收发。其职责是：管理已连接 TCP socket 的收发、维护发送队列、处理可靠/不可靠两种发送语义、提供可写回调通知。

#### 内部数据结构

```c
typedef struct {
  NiceAddress remote_addr;          /* 对端地址 */
  GQueue send_queue;                /* 发送队列（NiceSocketQueuedSend 链表） */
  GMainContext *context;            /* 主上下文 */
  GSource *io_source;               /* 可写 IO source（排空队列用） */
  gboolean error;                   /* 连接错误标志（对端关闭等） */
  gboolean reliable;                /* 可靠模式标志 */
  NiceSocketWritableCb writable_cb; /* 可写回调 */
  gpointer writable_data;           /* 可写回调用户数据 */
  NiceSocket *passive_parent;       /* 父级 passive socket（用于清理通知） */
} TcpPriv;
```

#### 准确函数原型（从 tcp-bsd.h）

```c
NiceSocket * nice_tcp_bsd_socket_new (GMainContext *ctx,
    NiceAddress *remote_addr, NiceAddress *local_addr, gboolean reliable);

NiceSocket * nice_tcp_bsd_socket_new_from_gsock (GMainContext *ctx,
    GSocket *gsock, NiceAddress *remote_addr, NiceAddress *local_addr,
    gboolean reliable);

void nice_tcp_bsd_socket_set_passive_parent (NiceSocket *socket,
    NiceSocket *passive_parent);

NiceSocket * nice_tcp_bsd_socket_get_passive_parent (NiceSocket *socket);
```

#### 核心函数分析

##### nice_tcp_bsd_socket_new_from_gsock()

"从现有 GSocket 构造"的工厂函数，是三个 TCP 入口（tcp-active、tcp-passive、TCP_SO）最终汇聚点：
1. 分配 `NiceSocket` 和 `TcpPriv`。
2. 存储 `remote_addr`、`context`、`reliable` 标志。
3. 设置 `sock->type = NICE_SOCKET_TYPE_TCP_BSD`。
4. 引用传入的 `gsock` 作为 `sock->fileno`。
5. 填充全部 6 个虚表函数指针（`send_messages`、`send_messages_reliable`、`recv_messages`、`is_reliable`、`can_send`、`set_writable_callback`、`close`）。

##### nice_tcp_bsd_socket_new()

独立的构造函数，自主创建 GSocket 并建立 TCP 连接：
1. 根据 `remote_addr` 的地址族创建 `G_SOCKET_TYPE_STREAM` / `G_SOCKET_PROTOCOL_TCP` 类型的 GSocket。
2. 设置 `TCP_NODELAY` 选项（禁用 Nagle 算法，避免小包合并延迟）。
3. 设置非阻塞模式（`g_socket_set_blocking(gsock, false)`）。
4. 调用 `g_socket_connect()` 发起连接。`G_IO_ERROR_PENDING` 是正常情况（非阻塞 connect），其他错误则失败。
5. 调用 `g_socket_bind()` 绑定本地地址。
6. 最终调用 `nice_tcp_bsd_socket_new_from_gsock()` 完成封装。

##### socket_recv_messages()

```c
static gint socket_recv_messages (NiceSocket *sock,
    NiceInputMessage *recv_messages, guint n_recv_messages,
    NiceMessageExtraData *exdata)
```

逐条接收的批量接口：
1. 检查 `priv->error` 标志，若已出错则直接返回 -1。
2. 遍历 `recv_messages` 数组，对每条调用 `g_socket_receive_message()`。
3. **连接关闭检测**：`len == 0` 表示对端执行了 shutdown，设置 `priv->error = TRUE` 并跳出循环。若首条消息就遇到此情况，返回 -1 通知上层销毁 IO source。
4. **WOULD_BLOCK 处理**：将 `len` 设为 0，正常返回已接收的消息数。
5. **源地址回填**：将 `priv->remote_addr` 写入 `recv_messages[i].from`（TCP 连接的对端地址是固定的）。

##### socket_send_message()

核心发送函数（内部，被 `socket_send_messages` 和 `socket_send_messages_reliable` 共用）：

```c
static gssize socket_send_message (NiceSocket *sock,
    const NiceOutputMessage *message, gboolean reliable)
```

发送队列管理逻辑（互斥锁保护）：

1. **队列为空时直接发送**：调用 `g_socket_send_message()` 尝试立即发送。
   - 全部发送成功：直接返回。
   - **部分发送**（`ret < message_len`）：将剩余数据入队（头部插入），创建可写 IO source。
   - **WOULD_BLOCK / NOT_CONNECTED / FAILED**：完整消息入队（尾部追加），创建可写 IO source。
2. **队列非空时**：
   - **可靠模式**（`reliable == TRUE`）：消息入队尾部，等待排空。
   - **非可靠模式**：不入队，直接返回 0（丢弃消息）。

**可靠 vs 非可靠的关键区别**：可靠模式下消息总是入队等待（即使队列非空），保证不丢数据；非可靠模式下队列非空时直接丢弃新消息，避免积压。

##### socket_send_messages()

批量非可靠发送：遍历消息数组，调用 `socket_send_message(message, FALSE)`。遇到错误或 WOULD_BLOCK 时停止并返回已发送数。

##### socket_send_messages_reliable()

批量可靠发送：遍历消息数组，调用 `socket_send_message(message, TRUE)`。遇到错误返回 -1，否则保证全部入队。

##### socket_is_reliable()

返回 `priv->reliable`。注意：TCP 本身是可靠协议，但 libnice 仍保留此标志以允许不可靠模式的使用场景（如某些测试或特殊的代理场景）。

##### socket_can_send()

返回 `g_queue_is_empty(&priv->send_queue)`。只有当发送队列为空时才允许发送 -- 这确保了背压控制：队列中有待发送数据时不接受新消息，直到 `socket_send_more` 排空队列并触发 `writable_cb`。

##### socket_set_writable_callback()

保存回调函数指针和用户数据。当发送队列排空时，`socket_send_more()` 会调用此回调通知上层可以继续发送。

##### socket_send_more() -- 队列排空回调

```c
static gboolean socket_send_more (GSocket *gsocket,
    GIOCondition condition, gpointer data)
```

当 socket 变为可写时由 GLib 主循环触发：
1. 调用 `nice_socket_flush_send_queue_to_socket()` 将队列中的数据实际写入 GSocket。
2. 若排空完毕（返回 TRUE）或连接挂起（`G_IO_HUP`）：销毁 IO source，调用 `writable_cb` 通知上层。
3. 若未排空（返回 FALSE）：保持 IO source 继续等待下次可写事件。

##### socket_close()

清理流程：
1. 关闭并释放底层 GSocket。
2. 销毁 IO source（如果存在）。
3. 若有关联的 passive_parent，调用 `nice_tcp_passive_socket_remove_connection()` 从被动 socket 的连接表中移除自己。
4. 释放发送队列（`nice_socket_free_send_queue`）。
5. 释放主上下文引用和 TcpPriv 内存。

---

### tcp-active.c / tcp-active.h (370 行) -- 主动 TCP Socket

#### 概述

`tcp-active.c` 实现了 TCP 连接的主动发起方（客户端），对应 ICE-TCP 中的 "Controlling" 角色。其核心设计采用**两阶段模式**：

1. **阶段一 -- 占位 socket**：`nice_tcp_active_socket_new()` 创建一个 `fileno == NULL` 的空壳 NiceSocket，此时所有收发操作均返回 -1。它的作用是在候选收集阶段（`gather_candidates`）快速创建一个占位符，让 ICE 协商过程可以启动。
2. **阶段二 -- 实际连接**：当 ICE 确定需要向某个对端发起 TCP 连接时，调用 `nice_tcp_active_socket_connect()` 创建真正的 GSocket、执行 `connect()`、然后包装为 `tcp-bsd` 并返回。此后所有 I/O 通过返回的 tcp-bsd socket 进行。

这种两阶段设计符合 ICE 规范：候选收集时不立即建立连接，等到连接检查阶段才按需连接。

#### 内部数据结构

```c
typedef struct {
  GSocketAddress *local_addr;  /* 本地绑定地址（预解析为 GSocketAddress） */
  GMainContext *context;       /* 主上下文 */
} TcpActivePriv;
```

数据结构极简，因为实际 I/O 功能完全委托给后续创建的 tcp-bsd。

#### 准确函数原型（从 tcp-active.h）

```c
NiceSocket * nice_tcp_active_socket_new (GMainContext *ctx, NiceAddress *addr);
NiceSocket * nice_tcp_active_socket_connect (NiceSocket *socket, NiceAddress *addr);
```

#### 核心函数分析

##### nice_tcp_active_socket_new()

创建占位 socket：
1. 解析本地地址 `addr`，将端口设为 0（让 OS 自动分配端口）。
2. 将地址转换为 `GSocketAddress` 并保存到 `priv->local_addr` 中，供后续 `connect()` 时使用。
3. 分配 NiceSocket，设置 `sock->type = NICE_SOCKET_TYPE_TCP_ACTIVE`。
4. **关键**：`sock->fileno = NULL` -- 此时还没有真正的 socket 文件描述符。
5. 填充虚表函数指针，但所有 I/O 函数均返回 -1，`can_send` 返回 FALSE，`set_writable_callback` 为空操作。

##### nice_tcp_active_socket_connect()

执行真正的 TCP 连接，将占位 socket 升级为可用的 tcp-bsd socket：

1. 解析目标地址 `addr`，根据地址族创建 `GSocket`（`SOCK_STREAM` / `TCP`）。
2. 设置非阻塞模式和 `TCP_NODELAY`。
3. 绑定到 `priv->local_addr`（即构造函数中保存的本地地址）。
4. 调用 `g_socket_connect()` 发起连接。`G_IO_ERROR_PENDING` 是正常的（非阻塞 connect 需要等待）。
5. 获取 OS 分配的实际本地地址（`g_socket_get_local_address()`）。
6. **创建 tcp-bsd**：调用 `nice_tcp_bsd_socket_new_from_gsock(priv->context, gsock, &local_addr, addr, TRUE)` 返回新的 tcp-bsd socket。
7. 释放临时 GSocket 引用（tcp-bsd 已持有引用）。

**与 tcp-bsd 构造函数的关系**：`nice_tcp_active_socket_connect()` 和 `nice_tcp_bsd_socket_new()` 都创建 GSocket 并 connect，但前者保留了占位 socket 的上下文和地址配置，后者是独立的完整流程。在 ICE 流程中，前者是标准路径（先创建占位候选，后按需连接），后者是简化路径。

##### 虚表函数特点

| 虚表函数 | 实现 |
|----------|------|
| `recv_messages` | 返回 -1（占位阶段不可接收） |
| `send_messages` | 返回 -1（占位阶段不可发送） |
| `send_messages_reliable` | 返回 -1 |
| `is_reliable` | 返回 TRUE |
| `can_send` | 返回 FALSE（连接未建立） |
| `set_writable_callback` | 空操作 |
| `close` | 释放 context 和 local_addr |

所有 I/O 操作都在占位 socket 上不可用。上层必须调用 `nice_tcp_active_socket_connect()` 获取实际可用的 tcp-bsd socket。

#### 与 tcp-bsd 的关系

```
tcp-active (占位)                              tcp-bsd (实际 I/O)
  sock->type = TCP_ACTIVE        connect()      sock->type = TCP_BSD
  sock->fileno = NULL         ─────────────►    sock->fileno = <real fd>
  recv_messages() → -1                          recv_messages() → g_socket_receive_message()
  send_messages() → -1                          send_messages() → g_socket_send_message()
  can_send() → FALSE                            can_send() → send_queue 为空时 TRUE
```

tcp-active 是"发起者"，tcp-bsd 是"执行者"。连接建立后，上层完全使用返回的 tcp-bsd socket，旧的 tcp-active 占位 socket 仅作为配置模板保留。

---

### tcp-passive.c / tcp-passive.h (400 行) -- 被动 TCP Socket

#### 概述

`tcp-passive.c` 实现了 TCP 连接的被动监听方（服务端），对应 ICE-TCP 中的 "Controlled" 角色。其职责是：在指定地址上监听 TCP 连接、接受新连接并为每个连接创建独立的 tcp-bsd socket、管理所有已接受连接的路由表。它是三个 TCP socket 类型中唯一持有 `GSocket` 文件描述符并直接参与 I/O 多路复用的。

与 tcp-active 的两阶段模式不同，tcp-passive 的 `socket_recv_messages()` 始终返回 -1（它本身不从网络读取数据），但 `socket_send_messages()` 会通过内部连接表查找到正确的子 socket 并委托发送。

#### 内部数据结构

```c
typedef struct {
  GMainContext *context;              /* 主上下文 */
  GHashTable *connections;            /* NiceAddress → NiceSocket* 连接映射表 */
  NiceSocketWritableCb writable_cb;   /* 可写回调 */
  gpointer writable_data;             /* 可写回调用户数据 */
} TcpPassivePriv;
```

**`connections` 哈希表**：以对端地址（`NiceAddress`）为键，对应的 tcp-bsd socket 为值。这是 tcp-passive 实现"一对多"通信的核心 -- 单个监听 socket 背后对应多个已接受的连接，发送数据时通过目标地址路由到正确的子 socket。

#### 准确函数原型（从 tcp-passive.h）

```c
NiceSocket * nice_tcp_passive_socket_new (GMainContext *ctx,
    NiceAddress *addr, GError **gerror);

NiceSocket * nice_tcp_passive_socket_accept (NiceSocket *socket);

void nice_tcp_passive_socket_remove_connection (NiceSocket *socket,
    const NiceAddress *to);
```

#### 核心函数分析

##### nice_tcp_passive_socket_new()

创建监听 socket：
1. 根据 `addr` 的地址族创建 `G_SOCKET_TYPE_STREAM` / `G_SOCKET_PROTOCOL_TCP` 类型的 GSocket。
2. 设置非阻塞模式。
3. 调用 `g_socket_bind()` + `g_socket_listen()` 绑定并开始监听。
4. 获取 OS 分配的实际地址（`g_socket_get_local_address()`），回填 `sock->addr`。
5. 分配 NiceSocket，设置 `sock->type = NICE_SOCKET_TYPE_TCP_PASSIVE`。
6. **关键**：`sock->fileno = gsock` -- 保存监听 socket 的文件描述符。上层（agent/component.c）会为此 fd 注册 `G_IO_IN` 回调，当有新连接到达时调用 `nice_tcp_passive_socket_accept()`。
7. 初始化空的 `connections` 哈希表。
8. 填充虚表函数指针。

##### nice_tcp_passive_socket_accept()

接受新连接并创建子 socket：

1. 调用 `g_socket_accept(sock->fileno)` 接受新连接，获得新的 GSocket。
2. 设置非阻塞模式和 `TCP_NODELAY`。
3. 通过 `g_socket_get_remote_address()` 获取对端地址。
4. **创建 tcp-bsd 子 socket**：调用 `nice_tcp_bsd_socket_new_from_gsock(priv->context, gsock, &sock->addr, &remote_addr, TRUE)`。
5. **建立父子关系**：
   - 调用 `nice_tcp_bsd_socket_set_passive_parent(new_socket, sock)` 告知子 socket 其父级。
   - 调用 `nice_socket_set_writable_callback(new_socket, _child_writable_cb, sock)` 使子 socket 可写时通知父级。
6. **注册到连接表**：以 `remote_addr` 的副本为键，将新 socket 插入 `connections` 哈希表。
7. 返回新的 tcp-bsd socket。上层（agent/component.c）会为该子 socket 的 fd 注册 `G_IO_IN` 回调以接收数据。

##### socket_send_messages() / socket_send_messages_reliable()

通过连接表路由发送：

```c
static gint socket_send_messages (NiceSocket *sock, const NiceAddress *to,
    const NiceOutputMessage *messages, guint n_messages)
{
  TcpPassivePriv *priv = sock->priv;
  if (to) {
    NiceSocket *peer_socket = g_hash_table_lookup (priv->connections, to);
    if (peer_socket)
      return nice_socket_send_messages (peer_socket, to, messages, n_messages);
  }
  return -1;
}
```

发送流程：根据目标地址 `to` 在 `connections` 表中查找对应的子 socket，找到则委托其发送，找不到则返回 -1。可靠发送版本逻辑相同但委托 `nice_socket_send_messages_reliable()`。

##### socket_recv_messages()

直接返回 -1。tcp-passive 本身不接收数据 -- 每个已接受的连接有自己独立的 tcp-bsd socket 和 IO source，接收由子 socket 独立完成。

##### socket_can_send()

在 `connections` 表中查找目标地址对应的子 socket，委托其 `can_send()` 判断。未找到则返回 FALSE。

##### socket_set_writable_callback()

保存回调函数。当子 socket 的可写回调触发时，`_child_writable_cb()` 将通知传播到父 socket 的用户回调，使上层感知到"passive socket 变为可写"。

##### _child_writable_cb()

内部回调，当子 socket（tcp-bsd）的发送队列排空时被调用，转发给 tcp-passive 注册的用户回调。这使上层可以通过监听 passive socket 的可写状态来获知所有子 socket 的可写变化。

##### socket_close()

清理流程：
1. 关闭监听 socket 并释放引用。
2. 释放主上下文引用。
3. 销毁 `connections` 哈希表（会释放所有键的内存和值对象）。
4. 释放 `TcpPassivePriv` 内存。

##### nice_tcp_passive_socket_remove_connection()

由 tcp-bsd 的 `socket_close()` 调用，将已关闭的连接从 `connections` 表中移除。这是父子 socket 之间生命周期联动的一部分。

---

### TCP 三种模式的对比

| 特性 | tcp-bsd | tcp-active | tcp-passive |
|------|---------|------------|-------------|
| 角色 | 已连接的通用基础 | 主动连接发起方 | 被动监听方 |
| 连接方式 | 已连接（或 connect 中） | `connect()` | `listen()` + `accept()` |
| ICE 角色 | 被上层使用 | Controlling | Controlled |
| `fileno` | 有效 GSocket | NULL（占位阶段） | 监听 GSocket |
| `recv_messages` | 实际接收 | 返回 -1 | 返回 -1 |
| `send_messages` | 实际发送 | 返回 -1 | 查表委托子 socket |
| `can_send` | send_queue 为空 | FALSE | 查表委托 |
| `is_reliable` | priv->reliable | TRUE | TRUE |
| 子 socket 管理 | 无 | 无 | connections 哈希表 |
| 典型使用 | 被 active/passive 创建 | ICE-TCP 主动连接 | ICE-TCP 被动监听 |
| socket_close 联动 | 通知 passive_parent | 仅清理自身 | 清理全部子连接 |

### TCP socket 生命周期图

```
tcp-active (占位)                        tcp-passive (监听)
  │                                        │
  │ nice_tcp_active_socket_new()           │ nice_tcp_passive_socket_new()
  │   type=TCP_ACTIVE, fileno=NULL         │   type=TCP_PASSIVE, fileno=<listen_fd>
  │                                        │
  │ nice_tcp_active_socket_connect()       │ nice_tcp_passive_socket_accept()
  │   socket() → bind() → connect()       │   g_socket_accept() → 新的 GSocket
  │   └→ nice_tcp_bsd_socket_new_from_gsock│   └→ nice_tcp_bsd_socket_new_from_gsock()
  │                                        │       └→ 注册到 connections 表
  ▼                                        ▼
tcp-bsd (已连接, 实际 I/O)               tcp-bsd (已连接, 实际 I/O)
  type=TCP_BSD, fileno=<conn_fd>           type=TCP_BSD, fileno=<accept_fd>
  recv_messages() → 实际收包               recv_messages() → 实际收包
  send_messages() → 实际发包               send_messages() → 实际发包
  passive_parent = NULL                    passive_parent = tcp-passive
```

### 与上层 agent 的交互

在 `agent/discovery.c` 的候选收集阶段：
- `nice_tcp_active_socket_new()` 创建 TCP 主动主机候选（Controlling 端）
- `nice_tcp_passive_socket_new()` 创建 TCP 被动主机候选（Controlled 端）

在 `agent/component.c` 的 I/O 回调中：
- 对于 tcp-passive 的监听 fd：注册 `G_IO_IN` 回调，触发时调用 `nice_tcp_passive_socket_accept()` 创建子 socket
- 对于 tcp-bsd 的已连接 fd：注册 `G_IO_IN` 回调，触发时调用 `nice_socket_recv_messages()` 接收数据
- 对于 tcp-active 的占位 socket：不注册任何回调（`fileno == NULL`）

发送路径总是一致的：`component.c` 持有 `NiceSocket*` 指针，调用 `nice_socket_send_messages()`，虚表自动分派到正确的实现（tcp-bsd 直接发送，tcp-passive 查表路由到子 tcp-bsd）。

---

## 第四部分：代理与加密层

### socks5.c / socks5.h (555 行) -- SOCKS5 代理

#### 概述

`socks5.c` 实现了通过 SOCKS5 代理（RFC 1928）建立 TCP 隧道的适配层。其职责是：在已连接的 TCP 基础上执行 SOCKS5 握手协议，使上层 TURN over TCP 流量能够穿透 SOCKS5 代理到达 TURN 服务器。这是一个典型的"握手后透明转发"模式。

#### 内部数据结构

```c
typedef enum {
  SOCKS_STATE_INIT,
  SOCKS_STATE_AUTH,
  SOCKS_STATE_CONNECT,
  SOCKS_STATE_CONNECTED,
  SOCKS_STATE_ERROR
} SocksState;

typedef struct {
  SocksState state;
  NiceSocket *base_socket;
  NiceAddress addr;
  gchar *username;
  gchar *password;
  GQueue send_queue;
} Socks5Priv;
```

#### 准确函数原型（从 socks5.h）

```c
NiceSocket *nice_socks5_socket_new (NiceSocket *base_socket,
    NiceAddress *addr, gchar *username, gchar *password);
```

#### SOCKS5 握手流程

```
Client (libnice)                       SOCKS5 Proxy Server
  │                                         │
  │ —— 1. 认证协商 ——                       │
  │  VER=5, NMETHODS, METHODS[]             │
  │  (支持无认证 + 用户名/密码)               │
  │                                         │ —— 2. 选择方法 ——
  │                                         │  VER=5, METHOD
  │                                         │
  │ [若 METHOD=0x02 且提供了用户名/密码]       │
  │ —— 3. 用户名/密码认证 ——                  │
  │  VER=1, ULEN, UNAME, PLEN, PASSWD       │
  │                                         │ —— 4. 认证结果 ——
  │                                         │  VER=1, STATUS
  │                                         │
  │ —— 5. CONNECT 请求 ——                    │
  │  VER=5, CMD=1(CONNECT), RSV=0,          │
  │  ATYP=1(IPv4)/4(IPv6), DST.ADDR, DST.PORT│
  │                                         │ —— 6. CONNECT 响应 ——
  │                                         │  VER=5, REP, RSV, ATYP, BND.ADDR, BND.PORT
  │                                         │
  │ === 隧道建立，后续数据透明转发 ===          │
```

#### 构造函数：nice_socks5_socket_new()

```c
NiceSocket *nice_socks5_socket_new (NiceSocket *base_socket,
    NiceAddress *addr, gchar *username, gchar *password)
```

1. 包装已有的 `base_socket`（通常是已连接的 TCP socket），共享其 `fileno` 和 `addr`。
2. 填充全部 7 个虚表函数指针。
3. **立即发送认证协商消息**：VER=0x05, NMETHODS=1（仅无认证）或 NMETHODS=2（增加用户名/密码认证，METHOD=0x02）。
4. 状态设为 `SOCKS_STATE_INIT`，等待服务器响应。

#### 握手状态机

握手完全在 `socket_recv_messages()` 中以状态机形式驱动：

**SOCKS_STATE_INIT -- 解析认证方法选择**：

接收服务器 2 字节响应 `(VER, METHOD)`：
- `METHOD == 0x00`：无需认证，跳转到 `send_connect` 发送 CONNECT 请求。
- `METHOD == 0x02`：需要用户名/密码认证。构造认证消息 `VER=1, ULEN, USERNAME, PLEN, PASSWORD`，发送后进入 `SOCKS_STATE_AUTH`。用户名和密码长度均限制为 255 字节。
- 其他 METHOD：不支持，进入 ERROR。

**SOCKS_STATE_AUTH -- 解析认证结果**：

接收 2 字节响应 `(VER, STATUS)`：
- `VER=1, STATUS=0x00`：认证成功，跳转 `send_connect`。
- 其他：认证失败，进入 ERROR。

**SOCKS_STATE_CONNECT -- 解析 CONNECT 响应**：

```c
send_connect:
  msg[len++] = 0x05;  /* VER=5 */
  msg[len++] = 0x01;  /* CMD=1 (CONNECT) */
  msg[len++] = 0x00;  /* RSV=0 */
  msg[len++] = 0x01/0x04;  /* ATYP (IPv4/IPv6) */
  /* + 4/16 bytes 地址 + 2 bytes 端口 */
```

接收服务器响应：
1. 先读取 4 字节 `(VER, REP, RSV, ATYP)`。
2. 根据 ATYP 继续读取绑定地址：
   - `0x01`（IPv4）：再读 6 字节（4 字节地址 + 2 字节端口）。
   - `0x04`（IPv6）：再读 18 字节（16 字节地址 + 2 字节端口）。
3. 检查 `REP == 0x00`（成功）：状态切换为 `SOCKS_STATE_CONNECTED`，刷新发送队列（`flush_send_queue`）排空握手期间积压的数据。

SOCKS5 错误码对照（REP 字段）：
- `0x01`：通用 SOCKS 服务器失败
- `0x02`：规则集不允许连接
- `0x03`：网络不可达
- `0x04`：主机不可达
- `0x05`：连接被拒绝
- `0x06`：TTL 过期
- `0x07`：命令不支持
- `0x08`：地址类型不支持

**SOCKS_STATE_CONNECTED -- 透明转发**：

所有 `recv_messages` 和 `send_messages` 直接委托给 `base_socket`，SOCKS5 层完全透明。

#### 虚表函数特点

与 socks5、http、pseudossl 共享统一的委托模式：

| 虚表函数 | 实现方式 |
|----------|----------|
| `recv_messages` | 自实现：状态机驱动握手（INIT/AUTH/CONNECT）或透明转发（CONNECTED） |
| `send_messages` | 委托：CONNECTED 时直接转发；其他状态返回 0（不可发送）；ERROR 返回 -1 |
| `send_messages_reliable` | 委托：CONNECTED 时转发；其他状态**入队**等待握手完成后自动排空 |
| `is_reliable` | 委托 `nice_socket_is_reliable(priv->base_socket)` |
| `can_send` | 委托 `nice_socket_can_send(priv->base_socket, addr)` |
| `set_writable_callback` | 委托 `nice_socket_set_writable_callback(priv->base_socket, ...)` |
| `is_based_on` | 递归：`sock == other \|\| nice_socket_is_based_on(base_socket, other)` |
| `close` | 释放 base_socket、username、password、send_queue |

#### 发送队列机制

关键设计：握手期间的非可靠 `send_messages` 直接返回 0（丢弃），但**可靠发送 `send_messages_reliable` 会将消息入队**（`nice_socket_queue_send`）。当握手完成进入 CONNECTED 状态时，`flush_send_queue` 将积压的数据一次性排出。这确保 TURN 分配等关键 STUN 消息不会丢失，而应用层数据在握手完成前不会发送。

---

### http.c / http.h (723 行) -- HTTP CONNECT 代理

#### 概述

`http.c` 实现了通过 HTTP CONNECT 方法（RFC 2817）建立 TCP 隧道的适配层。其职责是：在已连接的 TCP 基础上发送 HTTP CONNECT 请求并解析响应，使上层 TURN over TCP 流量能够穿透 HTTP 代理到达 TURN 服务器。与 SOCKS5 类似采用"握手后透明转发"模式，但状态机更复杂（需解析 HTTP 响应头中的 Content-Length）。

#### 内部数据结构

```c
typedef enum {
  HTTP_STATE_INIT,
  HTTP_STATE_HEADERS,
  HTTP_STATE_BODY,
  HTTP_STATE_CONNECTED,
  HTTP_STATE_ERROR
} HttpState;

typedef struct {
  HttpState state;
  NiceSocket *base_socket;
  NiceAddress addr;
  gchar *username;
  gchar *password;
  GQueue send_queue;

  /* 环形缓冲区，用于接收和解析 HTTP 头部 */
  guint8 *recv_buf;
  gsize recv_buf_length;   /* 缓冲区分配大小 */
  gsize recv_buf_pos;      /* 缓冲区 0 字节的偏移 */
  gsize recv_buf_fill;     /* 缓冲区中已占用的字节数 */

  /* 从 Content-Length 头部解析出的值 */
  gsize content_length;
} HttpPriv;
```

环形缓冲区（ring buffer）是 http.c 的核心数据结构，用于在非阻塞 I/O 下增量接收和解析 HTTP 响应。缓冲区初始大小为 0（延迟分配），首次需要时分配 1024 字节，每次填满后 double（`MAX(priv->recv_buf_length * 2, 1024)`）。

#### 准确函数原型（从 http.h）

```c
NiceSocket *nice_http_socket_new (NiceSocket *base_socket,
    NiceAddress *addr, gchar *username, gchar *password,
    GHashTable *extra_headers);
```

#### HTTP CONNECT 流程

```
Client (libnice)                       HTTP Proxy Server
  │                                         │
  │ —— CONNECT host:port HTTP/1.0 ——         │
  │  Host: host                             │
  │  User-Agent: libnice                    │
  │  Content-Length: 0                      │
  │  Proxy-Connection: Keep-Alive            │
  │  Connection: Keep-Alive                  │
  │  Cache-Control: no-cache                │
  │  Pragma: no-cache                       │
  │  [Proxy-Authorization: Basic <b64>]      │
  │  [extra_headers...]                     │
  │                                         │ —— HTTP 响应 ——
  │                                         │  HTTP/1.x 200 OK
  │                                         │  [Content-Length: xxx]
  │                                         │  [body bytes]
  │                                         │
  │ === 隧道建立，后续数据透明转发 ===          │
```

#### 构造函数：nice_http_socket_new()

```c
NiceSocket *nice_http_socket_new (NiceSocket *base_socket,
    NiceAddress *addr, gchar *username, gchar *password,
    GHashTable *extra_headers)
```

1. 包装已有的 `base_socket`（通常是已连接的 TCP socket），共享其 `fileno` 和 `addr`。
2. 初始化环形缓冲区字段为全零（延迟分配）。
3. 填充全部 7 个虚表函数指针。
4. **立即构造并发送 HTTP CONNECT 请求**：
   - 使用 `GString` 构造请求头，包含 `CONNECT host:port HTTP/1.0`、`Host`、`User-Agent: libnice`、`Content-Length: 0`、`Proxy-Connection: Keep-Alive`、`Connection: Keep-Alive`、`Cache-Control: no-cache`、`Pragma: no-cache`。
   - 如果提供了 `extra_headers` 哈希表，遍历追加（通过 `_append_extra_header()` 回调）。
   - 如果提供了 `username`，构造 `Proxy-Authorization: Basic <base64(user:pass)>` 头。
   - 以 `\r\n\r\n` 结束。
   - 通过可靠发送一次性发送完整请求。
5. 状态设为 `HTTP_STATE_INIT`，等待服务器响应。

#### 握手状态机

握手完全在 `socket_recv_messages()` 中以状态机形式驱动，与 socks5 不同之处在于需要一个环形缓冲区来增量解析 HTTP 响应。

**HTTP_STATE_INIT -- 解析状态行**：

1. 跳过前导空白。
2. 检查响应以 `HTTP/1.` 开头（支持 `HTTP/1.0` 和 `HTTP/1.1`）。
3. 跳过空格，解析 3 位状态码，要求为 `2xx`（成功）。任何非 2xx 状态码视为错误。
4. 消耗状态行到 `\r\n`。
5. 将 `content_length` 重置为 0，切换到 `HTTP_STATE_HEADERS`。

**HTTP_STATE_HEADERS -- 解析响应头**：

逐行解析头部：
- 识别 `Content-Length` 头（大小写不敏感）：提取数字值（通过 `g_ascii_digit_value()` 逐字符解析，支持环形缓冲区）。溢出检查：`content_length > G_MAXSIZE / 10` 时丢弃该头部值。
- 其他头部直接跳过（消耗到 `\r\n`）。
- 遇到空行（仅 `\r\n`，即 `pos == 2`）时切换到 `HTTP_STATE_BODY`。

**HTTP_STATE_BODY -- 消耗响应体**：

根据 `content_length` 的值消耗对应数量的字节：
- 如果 `content_length == 0`：直接切换到 `HTTP_STATE_CONNECTED`。
- 否则：从环形缓冲区消耗 `MIN(content_length, recv_buf_fill)` 字节，递减 `content_length`。循环直到 `content_length == 0` 后切换为 CONNECTED。

**HTTP_STATE_CONNECTED -- 透明转发**：

环形缓冲区中残留的数据（已超过头部和 body 的部分）通过 `memcpy_ring_buffer_to_input_messages()` 复制到用户的 recv_messages 缓冲区。同时调用 `flush_send_queue` 排空握手期间积压的可靠发送队列。

**错误处理**：任何解析失败设置 `state = HTTP_STATE_ERROR`，释放 base_socket。

#### 环形缓冲区操作

`memcpy_ring_buffer_to_buffer()` 是核心工具函数，支持环形缓冲区的"绕回"语义：
- 如果数据未绕回（`recv_buf_pos + recv_buf_fill <= recv_buf_length`）：从 `pos` 处直接拷贝。
- 如果数据已绕回：分两段拷贝 -- 先拷贝 `pos` 到缓冲区末尾，再拷贝缓冲区开头到剩余位置。

`memcpy_ring_buffer_to_input_messages()` 将环形缓冲区的数据批量分发到用户提供的多个 `NiceInputMessage` 中（遍历 messages 和每个 message 的 buffers）。

#### 虚表函数特点

与 socks5.c 完全一致的委托模式：

| 虚表函数 | 实现方式 |
|----------|----------|
| `recv_messages` | 自实现：状态机（INIT/HEADERS/BODY/CONNECTED），使用环形缓冲区增量解析 |
| `send_messages` | 委托：CONNECTED 时转发；ERROR 返回 -1；其他返回 0 |
| `send_messages_reliable` | 委托：CONNECTED 时转发；ERROR 返回 -1；其他入队等待 |
| `is_reliable` | 委托 `nice_socket_is_reliable(priv->base_socket)` |
| `can_send` | 委托 `nice_socket_can_send(priv->base_socket, addr)` |
| `set_writable_callback` | 委托 `nice_socket_set_writable_callback(priv->base_socket, ...)` |
| `is_based_on` | 递归委托 |
| `close` | 释放 base_socket、username、password、recv_buf、send_queue |

---

### pseudossl.c / pseudossl.h (399 行) -- 伪 SSL 层

#### 概述

`pseudossl.c` 实现了"伪 SSL"握手机制，用于兼容 Google TURN 和 Microsoft Office Communicator (MSOC) / Lync 服务器的非标准 TLS 封装。其名称中的"pseudo"是因为它并非真正的 SSL/TLS 实现（不使用 GnuTLS 或 OpenSSL 库），而是通过硬编码的握手字节序列与服务器进行"假的"SSL 握手 -- 客户端发送预定义的 ClientHello，服务器回复预定义的 ServerHello，验证通过后透明转发数据。

#### 内部数据结构

```c
typedef struct {
  gboolean handshaken;
  NiceSocket *base_socket;
  GQueue send_queue;
  NicePseudoSSLSocketCompatibility compatibility;
} PseudoSSLPriv;
```

#### 兼容性枚举（从 pseudossl.h）

```c
typedef enum {
  NICE_PSEUDOSSL_SOCKET_COMPATIBILITY_GOOGLE = 0,
  NICE_PSEUDOSSL_SOCKET_COMPATIBILITY_MSOC,
} NicePseudoSSLSocketCompatibility;
```

#### 准确函数原型（从 pseudossl.h）

```c
NiceSocket *nice_pseudossl_socket_new (NiceSocket *base_socket,
    NicePseudoSSLSocketCompatibility compatibility);
```

#### 硬编码握手数据

文件中包含四段硬编码的字节序列，作为握手"模板"：

| 握手方向 | 兼容模式 | 长度 | 说明 |
|---------|---------|------|------|
| ClientHello | GOOGLE | 64 字节 | 固定 TLS ClientHello（SSL 版本 3.1、TLS_RSA_WITH_RC4_128_SHA 等） |
| ServerHello | GOOGLE | 75 字节 | 期望的服务器响应 |
| ClientHello | MSOC | 42 字节 | MSOC/Lync 兼容的 ClientHello |
| ServerHello | MSOC | 79 字节 | 期望的 MSOC 服务器响应 |

#### 构造函数：nice_pseudossl_socket_new()

```c
NiceSocket *nice_pseudossl_socket_new (NiceSocket *base_socket,
    NicePseudoSSLSocketCompatibility compatibility)
```

1. 根据 `compatibility` 选择对应的 ClientHello 序列。
2. 如果 `compatibility` 不是 GOOGLE 或 MSOC，返回 NULL（不支持的兼容模式）。
3. 包装已有的 `base_socket`，共享其 `fileno` 和 `addr`。
4. 填充全部 7 个虚表函数指针。
5. **立即发送 ClientHello**：通过 `nice_socket_send_reliable()` 发送预定义的握手字节序列。
6. `handshaken = FALSE`，等待服务器 ServerHello。

#### 握手验证：server_handshake_valid()

```c
static gboolean server_handshake_valid(NiceSocket *sock, GInputVector *data, guint length)
```

- **GOOGLE 模式**：长度必须等于 `sizeof(SSL_SERVER_GOOGLE_HANDSHAKE)`，且 `memcmp` 全等。
- **MSOC 模式**：长度必须等于 `sizeof(SSL_SERVER_MSOC_HANDSHAKE)`。MSOC 的特殊处理 -- 先将接收数据中偏移 11 和 44 处的各 32 字节清零（这些是随机数区域），然后与模板比较。

#### 握手流程（socket_recv_messages 驱动）

```
Client (libnice)                       TURN Server
  │                                         │
  │ —— 预定义 ClientHello ——                 │
  │  (GOOGLE: 64B / MSOC: 42B)              │
  │                                         │ —— 预定义 ServerHello ——
  │                                         │  (GOOGLE: 75B / MSOC: 79B)
  │                                         │
  │ handshaken = TRUE                       │
  │ === 后续数据透明转发 ===                   │
```

1. 如果 `handshaken == TRUE`：直接委托 base_socket 接收（快速路径）。
2. 如果 `handshaken == FALSE`：
   - 根据兼容模式确定期望的 ServerHello 长度，在栈上分配对应大小的缓冲区。
   - 从 base_socket 接收数据。
   - 调用 `server_handshake_valid()` 验证。验证失败则释放 base_socket 并返回 -1。
   - 验证成功后：`handshaken = TRUE`，刷新发送队列。

#### 虚表函数特点

| 虚表函数 | 实现方式 |
|----------|----------|
| `recv_messages` | 自实现：handshake 模式下验证 ServerHello；handshaken 后透明转发 |
| `send_messages` | 委托：handshaken 时转发；否则返回 0 |
| `send_messages_reliable` | 委托：handshaken 时转发；否则**入队**等待握手完成 |
| `is_reliable` | 委托 base_socket |
| `can_send` | 委托 base_socket |
| `set_writable_callback` | 委托 base_socket |
| `is_based_on` | 递归委托 |
| `close` | 释放 base_socket 和 send_queue |

#### 与真正的 SSL/TLS 的区别

"伪 SSL"与真正的 TLS 实现的关键差异：

| 特性 | 真正的 TLS (GnuTLS/OpenSSL) | 伪 SSL (pseudossl.c) |
|------|---------------------------|---------------------|
| 握手 | 动态协商（密钥交换、证书验证） | 硬编码字节序列比对 |
| 加密 | 对称加密（AES/ChaCha20 等） | 无加密，仅握手模拟 |
| 证书 | X.509 证书链验证 | 无 |
| 密钥协商 | DH/ECDH 密钥交换 | 无 |
| 会话恢复 | Session ID / Session Ticket | 不支持 |
| 使用场景 | 标准 TLS 连接 | Google TURN / MSOC/Lync 兼容 |

pseudossl.c 的"握手机制"纯粹是协议兼容层 -- 它模拟了 Google Talk 和 Microsoft Lync 服务器要求的 TLS 外观，但握手后数据仍然是**明文传输**的。真正的加密由可选的 `socket/tcp-bsd.c` + GnuTLS/OpenSSL 层提供（在 socket 包装链中，pseudossl 可能被替换为真正的 TLS 包装）。

---

## socket/ 模块总结

### 文件清单与代码量

| 子模块 | 文件 | 行数 | 职责 |
|--------|------|------|------|
| core | socket.c / socket.h / socket-priv.h | 771 | 虚表抽象 + 工厂 |
| UDP | udp-bsd.c / udp-bsd.h | 597 | 原生 UDP |
| UDP | udp-turn.c / udp-turn.h | 2377 | TURN UDP 中继 |
| UDP | udp-turn-over-tcp.c / udp-turn-over-tcp.h | 523 | TURN over TCP |
| TCP | tcp-bsd.c / tcp-bsd.h | 554 | 基础 TCP |
| TCP | tcp-active.c / tcp-active.h | 370 | 主动 TCP |
| TCP | tcp-passive.c / tcp-passive.h | 400 | 被动 TCP |
| Proxy | socks5.c / socks5.h | 555 | SOCKS5 代理 |
| Proxy | http.c / http.h | 723 | HTTP 代理 |
| SSL | pseudossl.c / pseudossl.h | 399 | TLS/DTLS 加密 |
| **总计** | **20 files** | **~7269** | |

### 传输类型选择流程

```
ice-tcp? → tcp-active / tcp-passive
ice-udp? → turn-server? → udp-turn (or udp-turn-over-tcp via http/socks5)
         → no-turn → udp-bsd
tls/dtls? → pseudossl wraps above
```

### 代理/SSL 的包装链

socks5、http 和 pseudossl 都是"包装器"类型的 socket -- 它们不直接创建文件描述符，而是在已有 TCP 连接上增加握手/认证层，握手完成后透明转发。典型的嵌套结构：

```
udp-turn-over-tcp                    ← TCP 上的 TURN 帧处理
  └── udp-turn                       ← TURN 协议逻辑
       └── pseudossl                 ← 伪 SSL 握手（可选）
            └── http / socks5        ← 代理握手（可选，SOCKS5 或 HTTP）
                 └── tcp-active / tcp-passive  ← TCP 传输
                      └── tcp-bsd   ← 实际 TCP socket
```

### 关键设计要点

1. **虚表多态**：NiceSocket 的虚表机制让上层 agent 无需关心底层传输类型，所有 socket 类型通过统一接口（`socket_send_messages`、`socket_recv_messages` 等）操作。
2. **单端口复用**：UDP BSD socket 一个端口承载多个 component 的 host + srflx 候选。
3. **分层包装**：socket 可以嵌套（如 TCP -> HTTP Proxy -> TURN，或 TCP -> pseudossl -> TURN），每层通过 `base_socket` 指针向下委托。
4. **发送队列**：可靠 socket 维护发送队列，不可靠 socket 直接发送。代理和 SSL 层在握手期间将可靠发送入队，握手完成后统一排空。
5. **TURN 开销优化**：ChannelData（4 字节）vs Send Indication（~36 字节）的自动选择。
6. **异步 I/O 模型**：基于 GLib GSource 的非阻塞 I/O + 可写回调。握手状态机完全由 `socket_recv_messages()` 的被动回调驱动，无需额外定时器。
7. **握手后透明转发**：socks5、http、pseudossl 三个包装器共享相同模式——握手期间状态机驱动解析，握手完成后所有 `send/recv` 直接委托 `base_socket`，性能开销为零。
8. **环形缓冲区**：http.c 使用动态扩容的环形缓冲区（初始 1KB，double 增长）处理非阻塞 I/O 下的增量 HTTP 响应解析，核心算法处理了数据绕回（wrapping）和溢出防护。
