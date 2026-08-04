# 06 -- 模块分析：gst/ (GStreamer 插件)

## 概述

`gst/` 是 libnice 的可选 GStreamer 插件模块，提供了两个 GStreamer 元素：**nicesrc**（数据源，用于从 ICE 连接接收数据）和 **nicesink**（数据汇，用于通过 ICE 连接发送数据）。该模块使得 libnice 的 ICE 能力可以无缝集成到 GStreamer 多媒体 pipeline 中。

- **总代码量**：约 1350 行（gstnice.c 62 行 + gstnicesrc.c 487 行 + gstnicesrc.h 89 行 + gstnicesink.c 577 行 + gstnicesink.h 95 行）
- **依赖**：`gstreamer-1.0`、`gstnet-1.0`（GstNetControlMessageMeta）
- **构建**：编译为独立的 `libgstnice.so` 共享库，安装到 `$libdir/gstreamer-1.0/` 目录
- **构建系统**：Meson（`gst/meson.build`），编译时定义 `-DGST_USE_UNSTABLE_API` 以使用 GstNetControlMessageMeta
- **许可证**：MPL 1.1 / LGPL 2.1 双许可证，与 libnice 主体一致

## 文件: gstnice.c / gstnice.h (102 行)

### 插件注册

`gstnice.c` 是 GStreamer 插件的入口文件，负责将 nicesrc 和 nicesink 两个元素注册到 GStreamer 插件系统中。

**`plugin_init()`**
```c
static gboolean plugin_init (GstPlugin *plugin)
```
- 依次调用 `gst_element_register_nicesrc()` 和 `gst_element_register_nicesink()` 注册两个元素
- 任一元素注册失败则返回 FALSE，插件加载失败

**`GST_PLUGIN_DEFINE` 宏**
- 插件名称：`"nice"`
- 描述：`"Interactive UDP connectivity establishment"`
- 版本：使用 libnice 的 `VERSION` 宏
- 许可证：`"LGPL"`
- 包名：`PACKAGE_NAME`
- 官网：`"https://nice.freedesktop.org/"`

`gstnice.h` 仅包含两个子元素的头文件引用 (`#include "gstnicesrc.h"` 和 `#include "gstnicesink.h"`)，无其他公共定义。

## 文件: gstnicesrc.c / gstnicesrc.h (576 行) -- nicesrc 元素

### 概述

`GstNiceSrc` 继承自 `GstPushSrc`（GStreamer 基类中的 push-mode source），负责从 ICE 连接接收数据并将其作为 `GstBuffer` 输出到下游 GStreamer pipeline。它是一个 live source（通过 `gst_base_src_set_live(TRUE)` 标记），使用自己的 `GMainLoop` 在线程中阻塞等待数据。

- **类型宏**：`GST_TYPE_NICE_SRC`
- **Pad**：仅有 `src` pad（`GST_PAD_SRC`，格式为 `GST_STATIC_CAPS_ANY`）

### GstNiceSrc 结构体

```c
struct _GstNiceSrc {
  GstPushSrc parent;
  GstPad *srcpad;
  NiceAgent *agent;        // 绑定的 ICE 代理
  guint stream_id;         // 要读取的流 ID
  guint component_id;      // 要读取的组件 ID
  GMainContext *mainctx;   // 自有主上下文（隔离其他 GLib 事件）
  GMainLoop *mainloop;     // 自有主循环（阻塞等待数据到达）
  GstBufferList *outbufs;  // 已收到待输出的缓冲区列表
  gboolean unlocked;       // unlock 状态标志
  GSource *idle_source;    // 用于 unlock 的高优先级 idle source
};
```

**关键设计**：nicesrc 拥有自己的 `GMainContext` 和 `GMainLoop`（在 `gst_nice_src_init()` 中创建），这意味着 `nice_agent_attach_recv_ex()` 的回调将在独立于 GStreamer 流线程的上下文中执行。数据到达时，回调将 buffer 添加到 `outbufs` 列表并退出主循环，create 函数随即被唤醒，将缓冲区列表提交给 GStreamer。

### GObject 属性

| 属性 | 类型 | 读写 | 说明 |
|------|------|------|------|
| `agent` | `NiceAgent*` | RW (仅可设置一次) | 绑定的 ICE 代理 |
| `stream` | `guint` | RW | 要读取的流 ID |
| `component` | `guint` | RW | 要读取的组件 ID |

### 核心函数

#### `gst_nice_src_class_init()` -- 类初始化

- 设置 `GstPushSrcClass->create` 指向 `gst_nice_src_create()`
- 设置 `GstBaseSrcClass->unlock` / `unlock_stop` 用于状态切换时的线程安全中断
- 设置 `GstElementClass->change_state` 用于状态转换的验证和回调注册
- 添加 src pad 模板（`"src"`, `GST_PAD_SRC`, `GST_PAD_ALWAYS`, `GST_STATIC_CAPS_ANY`）
- 注册三个 GObject 属性（agent、stream、component）

#### `gst_nice_src_init()` -- 实例初始化

- 标记为 live source (`TRUE`) 和 TIME 格式
- 创建独立的 `GMainContext` 和 `GMainLoop`
- 初始化 `outbufs` 为空的 `GstBufferList`
- 将 `unlocked` 设为 FALSE

#### `gst_nice_src_set_property()` / `gst_nice_src_get_property()`

- `PROP_AGENT`：仅允许设置一次，重复设置会打印 `GST_ERROR`；get 返回当前 agent
- `PROP_STREAM` / `PROP_COMPONENT`：可随时设置和读取

#### `gst_nice_src_create()` -- 核心：拉取数据并创建 GstBuffer

```c
static GstFlowReturn gst_nice_src_create(GstPushSrc *basesrc, GstBuffer **buffer)
```

**执行流程**：
1. 检查 `unlocked` 标志，若为 TRUE 则返回 `GST_FLOW_FLUSHING`
2. 检查 `outbufs` 列表是否为空
3. 若为空，释放对象锁，调用 `g_main_loop_run(nicesrc->mainloop)` **阻塞等待**数据到达
4. 当回调函数 `gst_nice_src_read_callback()` 将 buffer 加入列表并退出主循环后，`create()` 被唤醒
5. 通过 `gst_base_src_submit_buffer_list()` 将整个 `outbufs` 列表提交给基类处理
6. 创建新的空 `outbufs` 列表用于后续数据
7. GStreamer 1.26 之前版本使用 `gst_base_src_wait_playing()` 作为 workaround（解决 NULL buffer + OK 返回值的 bug）

#### `gst_nice_src_read_callback()` -- ICE 数据到达回调

```c
static void gst_nice_src_read_callback(NiceAgent *agent, guint stream_id,
    guint component_id, guint len, gchar *buf, NiceMessageExtraData *exdata, gpointer data)
```

- 由 `nice_agent_attach_recv_ex()` 注册的回调函数，在各 ICE 数据到达时被调用
- 通过 `gst_buffer_new_allocate()` + `gst_buffer_fill()` 将原始数据拷贝到新的 `GstBuffer` 中
- 调用 `gst_nice_src_get_timestamp()` 获取 pipeline 时钟时间戳，同时设置 `GST_BUFFER_DTS` 和 `GST_BUFFER_PTS`
- **GstNetControlMessageMeta 处理**：如果 `exdata` 非空，调用 `nice_message_extra_data_get_tos()` 获取 `GSocketControlMessage`（包含 IP_TOS/IPV6_TCLASS 信息），然后通过 `gst_buffer_add_net_control_message_meta()` 将其附加到 buffer 的元数据中。这允许下游元素（如网络 sink）还原 DSCP 标记
- 将 buffer 添加到 `outbufs` 列表，调用 `g_main_loop_quit()` 唤醒阻塞在 `create()` 中的线程

#### `gst_nice_src_get_timestamp()` -- 时间戳获取

- 使用 `GST_ELEMENT_CLOCK()` 获取 pipeline 时钟
- 计算当前时钟时间减去 `base_time`，同时用作 PTS 和 DTS
- 若无时钟，返回 `GST_CLOCK_TIME_NONE`

#### `gst_nice_src_unlock()` / `gst_nice_src_unlock_stop()`

- **unlock**：设置 `unlocked = TRUE`，退出主循环；同时创建一个高优先级 (`G_PRIORITY_HIGH`) 的 idle source 并附加到 `mainctx`，该 source 在 `unlocked` 仍为 TRUE 时持续退出主循环，防止 `create()` 在 unlock 和 unlock_stop 之间重新进入主循环
- **unlock_stop**：清除 `unlocked` 标志，销毁并释放 idle source
- 这是 GStreamer 基类要求实现的标准线程安全中断机制

#### `gst_nice_src_change_state()` -- 状态转换处理

- **NULL -> READY**：验证 agent、stream_id、component_id 均已设置（不为 NULL 或 0），否则返回 `GST_STATE_CHANGE_FAILURE`
- **PAUSED -> READY**：调用 `nice_agent_attach_recv_ex()` 传入 NULL 回调以**卸载**回调，清空 `outbufs`
- **READY -> PAUSED**：调用 `nice_agent_attach_recv_ex()` **注册** `gst_nice_src_read_callback` 回调，开始接收数据

#### `gst_nice_src_dispose()` -- 资源清理

- 释放 agent 引用
- 销毁 `mainloop`、`mainctx`、`outbufs`、`idle_source`（如果存在）
- 调用父类 dispose

## 文件: gstnicesink.c / gstnicesink.h (672 行) -- nicesink 元素

### 概述

`GstNiceSink` 继承自 `GstBaseSink`，是 pipeline 的末端元素。它接收上游产生的 `GstBuffer`（或 `GstBufferList`），将其数据通过 ICE 连接发送出去。

- **类型宏**：`GST_TYPE_NICE_SINK`
- **Pad**：仅有 `sink` pad（`GST_PAD_SINK`，格式为 `GST_STATIC_CAPS_ANY`）
- **发送模式**：使用 `nice_agent_send_messages_nonblocking()` 进行非阻塞批量发送

### GstNiceSink 结构体

```c
struct _GstNiceSink {
  GstBaseSink parent;
  GstPad *sinkpad;
  NiceAgent *agent;         // 绑定的 ICE 代理
  guint stream_id;          // 发送目标的流 ID
  guint component_id;       // 发送目标的组件 ID
  gboolean reliable;        // 可靠模式标志（从 agent 属性读取）
  GCond writable_cond;      // 条件变量：等待 ICE 传输变为可写
  gulong writable_id;       // "reliable-transport-writable" 信号的 handler ID
  gboolean flushing;        // 正在 flush 中（对应 unlock 状态）

  /* 预分配的临时空间，避免每次 render 都 malloc */
  GOutputVector *vecs;      // 输出向量数组（指向 buffer 内存）
  guint n_vecs;             // vecs 数组容量
  GstMapInfo *maps;         // 内存映射信息数组
  guint n_maps;             // maps 数组容量
  NiceOutputMessage *messages;  // ICE 输出消息数组
  guint n_messages;         // messages 数组容量
};
```

**关键设计**：nicesink 在 `_init()` 中预分配了 `vecs`、`maps`、`messages` 数组（基于 `gst_buffer_get_max_memory()`），并在每次 send 前按需动态扩容（使用 `GST_ROUND_UP_16` 对齐），避免了 render 热路径上的频繁内存分配。

### GObject 属性

| 属性 | 类型 | 读写 | 说明 |
|------|------|------|------|
| `agent` | `NiceAgent*` | RW (仅可设置一次) | 绑定的 ICE 代理；设置时自动读取 reliable 属性并连接 `reliable-transport-writable` 信号 |
| `stream` | `guint` | RW | 发送目标的流 ID |
| `component` | `guint` | RW | 发送目标的组件 ID；变更时广播 writable_cond |

### 核心函数

#### `gst_nice_sink_class_init()` -- 类初始化

- 设置 `GstBaseSinkClass->render` 指向 `gst_nice_sink_render()`
- 设置 `GstBaseSinkClass->render_list` 指向 `gst_nice_sink_render_list()`（批量发送优化）
- 设置 `unlock` / `unlock_stop` 用于 flushing 状态管理
- 设置 `GstElementClass->change_state` 用于启动前验证
- 添加 sink pad 模板（`"sink"`, `GST_PAD_SINK`, `GST_PAD_ALWAYS`, `GST_STATIC_CAPS_ANY`）
- 注册三个 GObject 属性（agent、stream、component）

#### `gst_nice_sink_init()` -- 实例初始化

- 初始化 `writable_cond` 条件变量
- 根据 `gst_buffer_get_max_memory()` 返回的最大内存块数，预分配 `vecs`、`maps`、`messages` 数组
- 在 GStreamer >= 1.12 时调用 `gst_base_sink_set_drop_out_of_segment(FALSE)` 禁用段外丢帧（确保所有数据都尝试发送）

#### `gst_nice_sink_render()` -- 单 buffer 渲染入口

```c
static GstFlowReturn gst_nice_sink_render(GstBaseSink *basesink, GstBuffer *buffer)
```
- 检查 buffer 的内存块数，若有数据则委托给 `gst_nice_sink_render_buffers()` 处理
- 空 buffer（0 内存块）直接返回 `GST_FLOW_OK`

#### `gst_nice_sink_render_list()` -- buffer 列表渲染入口

```c
static GstFlowReturn gst_nice_sink_render_list(GstBaseSink *basesink, GstBufferList *buffer_list)
```
- 将 `GstBufferList` 展开为 buffer 指针数组和对应的内存块数数组
- 空列表直接返回 `GST_FLOW_OK`
- 使用 `g_newa`（栈上分配）避免堆分配
- 委托给 `gst_nice_sink_render_buffers()` 进行批量发送

#### `gst_nice_sink_render_buffers()` -- 核心：批量发送缓冲区

```c
static GstFlowReturn gst_nice_sink_render_buffers(GstNiceSink *sink,
    GstBuffer **buffers, guint num_buffers, guint8 *mem_nums, guint total_mem_num)
```

**执行流程**：

1. **预分配检查与扩容**：对比所需的 vecs/maps/messages 数量与当前容量，不足时按 `GST_ROUND_UP_16` 对齐扩容

2. **填充输出向量**：遍历所有 buffer，对每个 buffer 调用 `fill_vectors()` -- 通过 `gst_memory_map()` 将 buffer 中的每个 `GstMemory` 块映射到对应的 `GOutputVector`（零拷贝，直接指向 GstMemory 的底层数据）

3. **发送循环**（在 `GST_OBJECT_LOCK` 下）：
   - 调用 `nice_agent_send_messages_nonblocking()` 尝试发送所有消息
   - 返回值 > 0 表示成功发送了部分消息，更新 `written` 计数
   - **可靠模式**（`sink->reliable == TRUE`）：若未全部发送且错误是 `G_IO_ERROR_WOULD_BLOCK`，在 `writable_cond` 上调用 `g_cond_wait()` 等待 ICE 传输变为可写（由 `_reliable_transport_writable()` 信号回调广播）
   - **非可靠模式**：如遇严重错误（非 WOULD_BLOCK），停止发送以防止无限循环，打印 WARNING
   - 检查 `flushing` 标志，若已置位则返回 `GST_FLOW_FLUSHING`
   - 错误 `GError` 在每次循环迭代中通过 `g_clear_error()` 释放

4. **清理**：对所有已映射的内存调用 `gst_memory_unmap()`

#### `fill_vectors()` -- 将 GstBuffer 内存填充到输出向量

```c
static gsize fill_vectors(GOutputVector *vecs, GstMapInfo *maps, guint n, GstBuffer *buf)
```
- 遍历 buffer 的 n 个 `GstMemory` 块
- 对每个块调用 `gst_memory_map(mem, &maps[i], GST_MAP_READ)`
- 将映射后的数据指针和大小填入 `GOutputVector`
- 返回所有内存块的总大小

#### `_reliable_transport_writable()` -- 可写通知回调

```c
static void _reliable_transport_writable(NiceAgent *agent, guint stream_id,
    guint component_id, GstNiceSink *sink)
```
- 响应 NiceAgent 的 `"reliable-transport-writable"` 信号
- 验证 stream_id 和 component_id 匹配当前 sink
- 广播 `writable_cond`，唤醒所有等待在发送循环中的线程

#### `gst_nice_sink_unlock()` / `gst_nice_sink_unlock_stop()`

- **unlock**：设置 `flushing = TRUE`，广播 `writable_cond` 唤醒等待线程，使 `render_buffers()` 返回 `GST_FLOW_FLUSHING`
- **unlock_stop**：恢复 `flushing = FALSE`

#### `gst_nice_sink_set_property()` -- 属性设置

- `PROP_AGENT`：仅允许设置一次。设置后自动读取 agent 的 `"reliable"` 属性存入 `sink->reliable`，并连接 `"reliable-transport-writable"` 信号
- `PROP_STREAM`：在锁保护下设置（因为 render 线程同时读取）
- `PROP_COMPONENT`：在锁保护下设置；若 component_id 发生变化则广播 `writable_cond`（因为不同的 component 可能有不同的可写状态）

#### `gst_nice_sink_change_state()` -- 状态转换处理

- **NULL -> READY**：验证 agent、stream_id、component_id 均已设置且非 0
- 其他状态转换委托给父类，无额外处理

#### `gst_nice_sink_dispose()` / `gst_nice_sink_finalize()`

- **dispose**：断开 `reliable-transport-writable` 信号，释放 agent 引用，清理条件变量
- **finalize**：释放预分配的 `vecs`、`maps`、`messages` 数组

## GStreamer Pipeline 集成

### 典型 Pipeline 结构

**接收端（nicesrc）**：
```
nicesrc agent=X stream=1 component=1 ! rtprtxqueue ! application/x-rtp,... ! rtpjitterbuffer ! rtpopusdepay ! opusdec ! audioconvert ! audioresample ! pulsesink
```

**发送端（nicesink）**：
```
pulsesrc ! audioconvert ! opusenc ! rtpopuspay ! application/x-rtp,... ! rtprtxsend ! nicesink agent=X stream=1 component=1
```

### 使用要点

1. **NiceAgent 必须已在外部创建**并通过 `g_object_set()` 传入 nicesrc/nicesink。nicesrc/nicesink 不负责创建 agent
2. **ICE 协商**（gathering、SDP 交换、连接检查）必须在 pipeline 外部完成；nicesrc/nicesink 仅负责数据面的收发
3. **流和组件**通过 `stream` 和 `component` 属性指定。对于 RTP/RTCP 的典型场景，RTP 使用 component 1，RTCP 使用 component 2
4. **nicesrc** 作为 live source，时间戳来自 pipeline 时钟，因此 pipeline 中必须有时钟提供者（通常由 audiosrc 或 videosrc 提供）
5. **nicesink** 禁用 `drop_out_of_segment`，因为在实时流场景中，段外 buffer（如重传包）也需要发送

### 可靠/非可靠模式

- nicesink 在设置 `agent` 属性时自动从 agent 读取 `"reliable"` 属性
- 可靠模式（`reliable == TRUE`）：通过伪 TCP 发送，TCP 拥塞控制可能使发送阻塞，nicesink 等待 `reliable-transport-writable` 信号后重试
- 非可靠模式（`reliable == FALSE`）：UDP 模式，WOULD_BLOCK 错误直接放弃等待；但遇到严重错误时会停止发送以避免死循环

## 关键设计要点

1. **Push/Pull 模型分离**：nicesrc 基于 `GstPushSrc`（push-mode），nicesink 基于 `GstBaseSink`（pull-mode）。两者分别对应 ICE 连接的接收和发送方向

2. **独立 GMainContext 隔离**：nicesrc 拥有独立的 `GMainContext` 和 `GMainLoop`，将 ICE 数据接收回调的执行与 GStreamer 流线程隔离，避免了线程安全问题

3. **零拷贝数据路径**：nicesink 通过 `gst_memory_map()` 直接映射 GstBuffer 的底层内存到 `GOutputVector`，数据无需拷贝即可送入 ICE 发送函数。但 nicesrc 仍通过 `gst_buffer_fill()` 拷贝数据

4. **预分配与批量处理**：nicesink 在初始化时预分配 `GOutputVector`、`GstMapInfo`、`NiceOutputMessage` 数组，render 热路径中按需扩容，避免频繁堆分配。同时支持 `render_list` 批量处理多个 buffer，减少系统调用次数

5. **NiceAgent 外部化**：agent 通过 GObject 属性注入，nicesrc/nicesink 不负责 ICE 生命周期的管理。这保持了关注点分离，使得 ICE 信令和媒体传输可以独立控制

6. **TOS/TCLASS 元数据透传**（2025 年新增）：nicesrc 在接收回调中通过 `NiceMessageExtraData` 提取 `GSocketControlMessage`（包含 IP_TOS/IPV6_TCLASS），并使用 `gst_buffer_add_net_control_message_meta()` 将其附加到输出 buffer 的元数据中。这使得下游元素（如其他网络 sink 或 nicesink 本身）可以还原 QoS 标记

7. **可靠模式重试机制**：nicesink 在可靠模式下使用条件变量 (`GCond`) 实现阻塞等待，由 `reliable-transport-writable` 信号触发唤醒。非可靠模式下遇到严重错误会停止发送，防止无限循环。两类模式都正确处理了 `flushing` 状态，确保 pipeline 关闭时能及时退出

8. **线程安全设计**：nicesrc 的 `outbufs` 操作和 nicesink 的发送循环均在 `GST_OBJECT_LOCK` 保护下进行，状态标志（`unlocked`、`flushing`）也在锁下原子检查，确保 `unlock()`/`unlock_stop()` 与数据路径的正确同步
