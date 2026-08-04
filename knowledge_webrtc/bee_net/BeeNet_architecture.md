# BeeNet 架构分析

> 分析日期：2026-06-04  
> 仓库分支：v2  
> 分析范围：完整项目结构

---

## 1. 项目概述

BeeNet 是一个 **C++ 静态库**，对外暴露仿 POSIX 风格的纯 C API，内部使用 **Lua** 脚本驱动业务逻辑，底层通过 **libcurl** + **libwebrtc** 实现 HTTP、WebSocket、WebRTC 三种网络协议。

- 命名空间：`bee::net`
- 产物：`beenet.lib`（Windows）、`libbeenet.a`（其他平台）
- 平台支持：Windows、Linux、macOS、iOS、Android

---

## 2. 整体架构图

```
┌────────────────────────────────────────────┐
│                外部调用者                     │
│  bee_open / bee_read / bee_send / bee_close │
└──────────────────┬─────────────────────────┘
                   │ C API (interface.h)
┌──────────────────▼─────────────────────────┐
│              BeeManager (单例)               │
│  ┌──────────┐ ┌──────────┐ ┌────────────┐  │
│  │ CmdQueue │ │ Lua VM   │ │ DataCache[]│  │
│  │ (2条队列) │ │ (共享L)  │ │ (fd映射表)  │  │
│  └────┬─────┘ └────┬─────┘ └─────┬──────┘  │
│       │            │              │         │
│  ┌────▼────────────▼──────────────▼──────┐  │
│  │         Async Worker Thread           │  │
│  │  (消费命令, curl_multi 事件循环)         │  │
│  └───────────────────────────────────────┘  │
└──────────────────┬─────────────────────────┘
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
┌─────────┐ ┌──────────┐ ┌──────────────┐
│HTTPExec │ │ WSExec   │ │PeerConnection│
│(curl)   │ │(curl+WS) │ │(libwebrtc)   │
└────┬────┘ └────┬─────┘ └──────┬───────┘
     │           │               │
     └───────────┴───────┬───────┘
                         ▼
              ┌──────────────────┐
              │   第三方依赖       │
              │ curl / webrtc    │
              │ openssl / lua    │
              │ ffmpeg / opus    │
              └──────────────────┘
```

---

## 3. 公共 API 层 — `interface.h`

全部对外接口为纯 C 函数（`extern "C"`），分同步/异步两套：

### 生命周期管理

| 函数 | 功能 |
|---|---|
| `bee_logger_init(level, callback)` | 设置日志级别和回调 |
| `bee_env_init(opaque_json)` | 初始化 SDK 环境 |
| `bee_env_cleanup()` | 释放 SDK 资源 |
| `bee_set_user_id(uid)` | 设置用户 ID |
| `bee_set_app_version(ver)` | 设置应用版本 |
| `bee_set_sdk_version(ver)` | 设置 SDK 版本 |
| `bee_set_device_type(type)` | 设置设备类型 |
| `bee_set_lua_file(path)` | 设置 Lua 脚本路径 |

### 同步 API

| 函数 | 功能 |
|---|---|
| `bee_open(url, json)` → fd | 打开连接 |
| `bee_read(fd, buf, len)` → bytes | 读取数据 |
| `bee_send(fd, msg, len)` → result | 发送消息 |
| `bee_stat(fd, buf, len)` → length | 获取流状态 |
| `bee_seek(fd, offset, whence)` → offset | 流定位 |
| `bee_close(fd)` → err | 关闭连接 |

### 异步 API

| 函数 | 功能 |
|---|---|
| `bee_open_async(url, json, onopen, userp)` | 异步打开 |
| `bee_read_async(fd, bytes, onrecv, userp)` | 异步读取 |
| `bee_send_async(fd, msg, len, onsent, userp)` | 异步发送 |
| `bee_stat_async(fd, onstat, userp)` | 异步状态查询 |
| `bee_seek_async(fd, offset, whence, onseek, userp)` | 异步定位 |
| `bee_close_async(fd, onclose, userp)` | 异步关闭 |

**同步/异步桥接机制**：同步 API 将操作封装为 `Bee*Cmd` 命令投递到工作线程，通过 `std::promise` 阻塞等待结果；异步 API 通过 `BeeAsync*Cmd` 投递，结果通过回调返回。

---

## 4. 核心运行时 — `BeeManager`（单例）

文件：`manager.h` / `manager.cpp`

`BeeManager` 是整个 SDK 的运行时「大总管」，全局唯一实例（`static BeeManager myself_`），持有：

| 组件 | 说明 |
|---|---|
| `cmd_queue_` | 常规命令队列（无锁 SPSC） |
| `urge_cmd_queue_` | 优先命令队列（处理 init_args 等紧急任务） |
| `async_worker_thread_` | WebRTC 线程，消费命令并驱动 curl 事件循环 |
| `lua_` | 共享 Lua VM（`shared_ptr<lua_State>`），所有 Executer 共用 |
| `fd2cache_vec_` | fd→DataCache 映射表（最多 1024 条） |
| `fd2cache_free_fds_` | 空闲 fd 环形缓冲区 |
| `executer_list_head_` | Executer 双向链表（活跃列表） |
| `poll_list_head_` | 待处理的 Executer 链表 |
| `curl_share_object_` | curl share handle（DNS/SSL 会话共享） |
| `curl_multi_handle_` | curl multi handle（事件驱动） |

关键静态方法：

| 方法 | 功能 |
|---|---|
| `Create()` / `GetInstance()` | 获取/创建单例 |
| `GetLuaContext()` | 获取共享 Lua 上下文 |
| `ChangeLuaContext(...)` | 运行时热替换 Lua 脚本 |
| `CreateCmd<T>(...)` | 创建命令（通过模板对象池） |
| `PostCmd(cmd)` | 投递命令到工作线程 |
| `CreateCache()` | 分配 fd 并创建 DataCache |
| `GetCache(fd)` | 通过 fd 查找 DataCache |
| `AddExecuter()` / `RemoveExecuter()` | 管理 Executer 链表 |

---

## 5. 命令系统 — `basecmd.h`

### 类层次

```
BaseCmd (虚基类，纯虚 Process())
CmdHandler (节点基类，含 next/rnext 原子指针)
   └── CmdMaker<T> (模板，多继承 T + CmdHandler)
         ├── 对象池 (ReuseQueue)：按模板类型独立复用内存
         └── Create()：优先从池中取，池空则 new (128字节对齐)
```

### CmdQueue — 无锁生产者/消费者队列

基于 `std::atomic<CmdHandler*>` 的 Michael-Scott 队列变体：

- `PutCmd(cmd)` — 生产者入队（CAS 更新 tail）
- `GetCmd()` — 消费者出队（CAS 更新 head）
- `Reset()` — 清空队列，释放所有节点（含并发安全保护）

### 关键命令类

所有命令定义在 `opencmd.h` 和 `initcmd.h`：

**同步命令**（含 `std::promise` 用于阻塞结果返回）：
- `BeeOpenCmd` — 打开连接
- `BeeReadCmd` — 读取数据
- `BeeSendCmd` — 发送消息
- `BeeStatCmd` — 获取状态
- `BeeSeekCmd` — 流定位
- `BeeCloseCmd` — 关闭连接

**异步命令**（含回调函数指针）：
- `BeeAsyncOpenCmd` — 异步打开
- `BeeAsyncReadCmd` — 异步读取
- `BeeAsyncSendCmd` — 异步发送
- `BeeAsyncStatCmd` — 异步状态查询
- `BeeAsyncSeekCmd` — 异步定位
- `BeeAsyncCloseCmd` — 异步关闭

**初始化命令**：
- `EnvInitCmd` — 执行环境初始化（调用 Lua `bee_env_init`）
- `RuntimeInitCmd` — 初始化 Lua 运行时（含 DRM keyid/key）

---

## 6. 协议处理器 — `Executer` 继承体系

文件：`executer.h` / `executer.cpp`

### 基类 Executer

```
Executer (抽象基类)
├── 封装 CURL* easy handle (unique_ptr)
├── 关联一个 Lua 协程 (lua_co_)
├── 事件类型：EV_NONE / EV_READ / EV_WRITE / EV_CLOSE
├── 虚函数接口：
│   ├── Destroy() noexcept = 0
│   ├── OnErrorEvent(error) = 0
│   ├── OnCompleteEvent() → bool = 0
│   ├── OnReadEvent() → const char*
│   ├── OnWriteEvent() → const char*
│   └── OnCloseEvent()
└── BeeManager 维护双向链表（prev_/next_）
```

### HTTPExecuter

文件：`http_exec.h` / `http_exec.cpp`

- 协议：HTTP/HTTPS（通过 curl，支持 HTTP/2、HTTP/3）
- 状态机：`INIT → HEAD → BODY → COMPLETE → BREAK → DESTROY`
- 功能：
  - 分块下载（通过 curl 回调 `GetHeader` / `GetBody`）
  - 进度回调（`ProgressCallback`）
  - 自定义 HTTP headers（`curl_slist`）
  - POST multipart form（`curl_mime`）
  - 业务逻辑由 Lua 协程中的 `on_header` / `on_body` / `on_progress` 回调处理

### WSExecuter

文件：`ws_exec.h` / `ws_exec.cpp`

- 协议：WebSocket（RFC 6455）
- 状态机：`INIT → HTTP(upgrade) → WEBSOCKET → DESTROY`
- 状态位：`S_ERROR / S_WS_OK / S_CLOSE_1(主动关闭) / S_CLOSE_2(被动关闭)`
- 功能：
  - 完整的 WebSocket 帧解析与构造
  - 文本/二进制消息发送与接收
  - Ping/Pong 心跳
  - 主动/被动关闭握手
  - 自定义 headers
- 使用独立的 `send_buffer_` 和 `recv_buffer_`（`io_buffer`）

### CAExecuter

文件：`ca_exec.h` / `ca_exec.cpp`

- 功能：证书颁发机构相关操作
- 细节较少，继承自 Executer 基类

---

## 7. WebRTC 子系统

### PeerConnection

文件：`peerconnection.h` / `peerconnection.cpp`

`PeerConnection` 多重继承：
- `webrtc::PeerConnectionObserver` — ICE/连接/DataChannel 事件
- `webrtc::AudioTrackSinkInterface` — 音频数据接收
- `WebRTCVideoSinkInterface<VideoFrame>` — 视频帧接收
- `std::enable_shared_from_this` — 支持弱引用

**Lua API**（通过 `LuaOpenPeerConnectionLib` 注册）：

| Lua 函数 | 功能 |
|---|---|
| `Create()` | 创建 PeerConnection |
| `Close()` | 关闭连接 |
| `SetCallback()` | 设置事件回调 |
| `SetRemoteDescription()` | 设置远端 SDP |
| `AddRemoteCandidate()` | 添加 ICE 候选 |
| `CreateOffer()` / `CreateAnswer()` | SDP 协商 |
| `AddTrack()` / `RemoveTrack()` / `ChangeTrack()` | 音视频轨道管理 |
| `CreateDataChannel()` | 创建数据通道 |
| `GetStats()` | 获取统计信息 |
| `Destroy()` | 销毁 |

**回调引用**（Lua registry ref）：
- `ref_on_description_` — SDP 就绪
- `ref_on_ice_candidate_` — ICE 候选
- `ref_on_ice_gathering_state_change_` — ICE 收集状态
- `ref_on_connection_state_change_` — 连接状态变更
- `ref_on_data_channel_` — DataChannel 创建
- `ref_on_video_frame_` — 视频帧到达
- `ref_on_audio_frame_` — 音频帧到达
- `ref_on_rtc_stats_` — 统计报告

### DataChannel

文件：`peerconnection.h`（内嵌于 PeerConnection 声明）

- 封装 `webrtc::DataChannelInterface`
- 支持文本/二进制消息
- 双向链表管理（`prev_`/`next_`，由 PeerConnection 持有头节点）

### 音视频命令 — 跨线程数据传递

- `VideoFrameCmd` — 将视频帧从 WebRTC 线程传递到工作线程
- `AudioDataCmd` — 将音频数据从 WebRTC 线程传递到工作线程
- 两者均通过 `weak_ptr<PeerConnection>` 弱引用目标对象

---

## 8. WebRTC 扩展 (`extensions/webrtc/`)

libwebrtc 的自定义实现，用于注入外部媒体源：

| 文件 | 功能 |
|---|---|
| `external_audio_source_impl.*` | 外部 PCM 音频注入适配器 |
| `external_audio_device_module.*` | 自定义音频设备模块接口 |
| `external_audio_capture_module.*` | 自定义音频采集模块 |
| `external_video_source_impl.*` | 外部视频帧注入适配器 |
| `video_encoder_factory.*` | 编解码器工厂（H.264/VP8/VP9 选择） |
| `video_decoder_factory.*` | 解码器工厂 |
| `apple/video_toolbox_encoder.*` | macOS/iOS VideoToolbox 硬编码 |
| `apple/video_toolbox_decoder.*` | macOS/iOS VideoToolbox 硬解码 |
| `apple/video_toolbox_nalu_rewriter.*` | NALU 重写（H.264 Annex B ↔ AVCC） |
| `apple/video_capture/*` | macOS/iOS 摄像头采集 |

---

## 9. 外部媒体注入接口

### ExternalAudioSource

文件：`external_audio.h`

- 基类：用户继承并实现音频数据源
- `RegisterExternalSource()` — 注册到 WebRTC，返回 source ID
- `IncomingData(data, bits, rate, channels, samples, format)` — 推送 PCM 数据
- `ConvertToDefaultPCM()` — 将任意 PCM 格式转换为 16-bit interleaved 标准格式
- 支持的格式标志：`kFloat`、`kNonInterleaved`、`kBigEndian`

### ExternalVideoSource

文件：`external_video.h`

- 基类：用户继承并实现视频数据源
- `RegisterExternalSource()` — 注册到 WebRTC，返回 source ID
- `IncomingFrame(data, length, width, height, type)` — 推送视频帧
- 支持 14 种视频格式：`kI420`、`kRGB24`、`kARGB`、`kYUY2`、`kMJPEG`、`kNV12` 等

### 媒体帧封装

文件：`media_frame.h`

- `VideoFrame`：封装 `webrtc::VideoFrame`，暴露 Lua API（id/width/height/rotation）
- `AudioFrame`：封装 PCM 音频数据，暴露 Lua API（bits_per_sample/sample_rate/channels/samples）

---

## 10. 数据缓冲层

### io_buffer

文件：`iobuffer.h` / `iobuffer.cpp`

高性能读写缓冲区（GPL 许可，源于独立组件）：

- 支持网络字节序整数的编码/解码（`EncodeInt16/32/64`、`DecodeInt16/32/64`）
- 模板化内存分配（`alloc<T>()` 返回 `io_buffer_view<T>`）
- `put_byte/word/int32/int64` — 写入操作
- `get_byte/word/int32/int64` — 读取操作
- `put_vstring/put_string` — 格式化写入
- `get_line` — 按分隔符读取一行
- `release(n)` / `clear()` / `chop()` / `chomp()` — 缓冲区指针管理

### DataCache

文件：`datacache.h` / `datacache.cpp`

每个 fd 对应一个 DataCache 实例，是外部 API 与 Lua 协程之间的数据桥梁：

**状态机**：
```
INIT → READY → FINISHED → SUCCESS / FAILED / LUAERR → DESTROY
                  ↑            │
                  └── seek ────┘
```

**Lua API**（通过 `LuaOpenDataCacheLib` 注册）：

| Lua 函数 | 功能 |
|---|---|
| `ready()` | 标记就绪 |
| `fail()` | 标记失败 |
| `stuff_finish()` | 标记数据接收完成 |
| `append_data()` | 追加接收数据 |
| `append_offset()` | 追加接收偏移 |
| `seek()` | 流定位 |
| `wait()` / `wakeup()` | 协程等待/唤醒（读写同步） |
| `buffer_flush()` | 刷新缓冲区 |
| `buffered_amount()` | 查询缓冲区数据量 |
| `set_callback()` | 设置回调 |
| `destroy()` | 销毁 |

---

## 11. 其他核心文件

| 文件 | 功能 |
|---|---|
| `logger.h/cpp` | 日志系统（BEE_LOG_FATAL/ERROR/WARN/INFO/DEBUG/TRACE 宏） |
| `os.h/cpp` | 平台抽象层 |
| `camera_video.h/cpp` | 摄像头视频采集 |
| `desktop_video.h/cpp` | 桌面/屏幕视频采集 |
| `deflua.h/c` | 默认 Lua 脚本（加密常量定义） |
| `cacert.c` | 内置 CA 证书 |
| `external_media_source_pusher.cpp` | 外部媒体源推送辅助 |

---

## 12. 第三方依赖 (`third_party/`)

### 网络传输栈

| 库 | 平台 | 说明 |
|---|---|---|
| libcurl | 全平台 | HTTP/HTTPS 客户端，通过 nghttp2/nghttp3/ngtcp2 支持 H2/H3 |
| nghttp2 | 全平台 | HTTP/2 协议实现 |
| nghttp3 | 全平台 | HTTP/3 协议实现 |
| ngtcp2 | 全平台 | QUIC 传输层 |
| c-ares | 全平台 | 异步 DNS 解析 |

### WebRTC 与媒体

| 库 | 平台 | 说明 |
|---|---|---|
| libwebrtc (Google) | 全平台 + winos | WebRTC 实现（含 abseil-cpp、libyuv、perfetto） |
| FFmpeg | Linux/macOS/Windows | 媒体编解码（avcodec/avformat/avutil/swresample/swscale） |
| x264 | Linux/macOS/Windows | H.264 软件编码 |
| fdk-aac | Linux/macOS/Windows | AAC 音频编解码 |
| opus | Linux/Windows | Opus 音频编解码 |

### 硬件加速

| 库 | 平台 | 说明 |
|---|---|---|
| AMF | Linux/Windows | AMD Advanced Media Framework |
| mfx (Intel Media SDK) | Linux/Windows | Intel 硬件编解码 |
| libva | Linux | VA-API 硬件加速 |

### TLS 与加密

| 库 | 平台 | 说明 |
|---|---|---|
| OpenSSL | 全平台 | TLS/SSL（也被 ngtcp2 使用） |
| xxtea | 全平台 | XXTEA 加密算法 |

### 脚本与数据

| 库 | 平台 | 说明 |
|---|---|---|
| Lua | 全平台 | Lua 5.x 脚本运行时 |
| lua_cjson | 源码 | JSON 解析/生成（fpconv + strbuf） |
| lua_zlib | 源码 | zlib 压缩/解压绑定 |
| lua-squish | 源码 | Lua 脚本合并工具 |
| jsoncpp | Linux/macOS/Windows | C++ JSON 库 |

### 其他工具库

| 库 | 平台 | 说明 |
|---|---|---|
| zlib | 全平台 + winos | 数据压缩 |
| brotli | Windows | Brotli 压缩 |
| librtmp | Linux/macOS/Windows | RTMP 协议 |
| libyuv | Linux | YUV 格式转换 |
| libcaca | Linux/macOS | 彩色 ASCII 渲染 |
| ncurses | Linux | 终端 UI |
| getopt | Windows | 命令行参数解析 |
| vcpkg | Windows | 包管理器 |

---

## 13. 平台构建体系 (`builder/`)

| 平台 | 构建目录 | 构建系统 | 产物 |
|---|---|---|---|
| Windows | `builder/windows/` | Makefile + MSVC | `beenet.lib` |
| Windows UNC | `builder/windows_uncrossed/` | Makefile + vcpkg | `beenet.lib` |
| Linux | `builder/linux/` | Makefile + GCC/Clang | `libbeenet.a` |
| macOS | `builder/macos/` | Makefile + Clang | `libbeenet.a` |
| iOS | `builder/ios/` | Makefile + Xcode | `libbeenet.a` |
| Android | `android/` | Gradle + NDK | AAR |

构建变量：
- `BUILD`：`debug` / `release`
- `ARCH`：平台相关（`x64`、`x86`、`arm64`、`armv7`）
- `SYS`（Linux）：目标发行版

依赖引用路径：`third_party/<lib>/<platform>/output/<BUILD>/<ARCH>/`

---

## 14. 关键预处理器宏

| 宏 | 用途 |
|---|---|
| `WEBRTC_WIN` | Windows WebRTC 构建标识 |
| `WEBRTC_USE_H264` | 启用 H.264 编解码器 |
| `RTC_DISABLE_CHECK_MSG` | 屏蔽 WebRTC 断言消息 |
| `RTC_DISABLE_LOGGING` | 屏蔽 WebRTC 内部日志 |
| `_DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR` | MSVC STL 兼容性修复 |
| `CURL_STATICLIB` | 静态链接 curl |
| `USE_BUILTIN_CA` | 使用内置 CA 证书 |

---

## 15. 设计模式总结

| 模式 | 应用位置 | 说明 |
|---|---|---|
| **单例** | `BeeManager` | 全局唯一运行时 |
| **生产者/消费者** | `CmdQueue` | 无锁命令投递与消费 |
| **模板+对象池** | `CmdMaker<T>` | 减少内存分配，按类型池化 |
| **策略模式** | `Executer` 继承体系 | 不同协议统一事件接口 |
| **观察者/回调** | Lua registry ref (`ref_on_*`) | 事件驱动的 Lua 回调机制 |
| **协程协作** | Lua `yield`/`resume` | 异步 I/O 的同步化写法 |
| **桥接模式** | 同步 API → Cmd + promise | C 同步接口与内部异步实现的桥接 |

---

## 16. 目录结构一览

```
BeeNet/
├── interface.h/cpp          # 公共 C API
├── manager.h/cpp            # BeeManager 单例运行时
├── executer.h/cpp           # Executer 抽象基类
├── basecmd.h                # CmdQueue + CmdMaker 命令系统
├── opencmd.h                # 同步/异步操作命令
├── initcmd.h                # 初始化命令
├── datacache.h/cpp          # fd 数据缓存
├── iobuffer.h/cpp           # 读写缓冲区
├── http_exec.h/cpp          # HTTP 协议处理器
├── ws_exec.h/cpp            # WebSocket 协议处理器
├── ca_exec.h/cpp            # CA 证书处理器
├── peerconnection.h/cpp     # WebRTC PeerConnection
├── media_frame.h/cpp        # 音视频帧封装
├── external_audio.h         # 外部音频注入接口
├── external_video.h         # 外部视频注入接口
├── external_media_source_pusher.cpp
├── camera_video.h/cpp       # 摄像头采集
├── desktop_video.h/cpp      # 桌面采集
├── logger.h/cpp             # 日志系统
├── os.h/cpp                 # 平台抽象
├── deflua.h/c               # 默认 Lua 脚本常量
├── cacert.c                 # 内置 CA 证书
├── extensions/webrtc/       # WebRTC 扩展实现
│   ├── external_audio_*.h/cpp
│   ├── external_video_*.h/cpp
│   ├── video_*_factory.h/cpp
│   └── apple/              # Apple 平台硬编解码
├── third_party/            # 35+ 预编译第三方库
├── builder/                # 各平台构建文件
├── windows/                # Windows 输出 + 测试
├── linux/                  # Linux 输出 + 测试
├── macos/                  # macOS 输出 + 测试
├── ios/                    # iOS 工程
└── android/                # Android Gradle 工程
```
