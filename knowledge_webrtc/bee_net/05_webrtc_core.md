# BeeNet 模块5：WebRTC 核心

> 精读日期：2026-06-04  
> 覆盖文件：`peerconnection.h/cpp`, `media_frame.h/cpp`

---

## 文件依赖关系

```
peerconnection.cpp ──► webrtc_defines.h, camera_video.h, desktop_video.h, media_frame.h, manager.h
                       extensions/webrtc/* (自定义编解码工厂、外部源)
media_frame.cpp    ──► webrtc VideoFrame (libwebrtc)
```

PeerConnection 引用所有 WebRTC 扩展模块和媒体采集模块。

---

## 1. PeerConnection — WebRTC 对等连接

### 多重继承结构

```
PeerConnection :
  webrtc::PeerConnectionObserver     // ICE/连接/DataChannel 事件
  webrtc::AudioTrackSinkInterface    // 音频数据接收
  WebRTCVideoSinkInterface<VideoFrame> // 视频帧接收
  std::enable_shared_from_this       // 弱引用支持
```

`shared_from_this` 使得交叉线程回调可以使用 `weak_ptr` 安全访问 PeerConnection。

### 内部辅助类

#### `CSDO` (CreateSessionDescriptionObserver)
- **用途**: 处理 `CreateOffer` / `CreateAnswer` 的回调
- **核心逻辑**: 持有 `weak_ptr<PeerConnection>`，成功时调用 `pc->OnLocalDescription(desc)`

#### `SSDO` (SetSessionDescriptionObserver)
- **用途**: 处理 `SetLocalDescription` / `SetRemoteDescription` 的回调
- **核心逻辑**: OnSuccess 空实现；OnFailure 打日志

#### `RTCStatsObtainer` (RTCStatsCollectorCallback)
- **用途**: 处理 `GetStats()` 的回调
- **核心逻辑**: 持有 `weak_ptr<PeerConnection>`，成功时调用 `pc->OnRTCStatsObtained(report)`

### 构造与销毁

#### `PeerConnection()`
- **初始化**: 所有 Lua 回调引用（`ref_on_*`）设为 `LUA_NOREF`
- `lua_co_(nullptr)` — Lua 协程在 `LuaCreate` 时设置

#### `Destroy()`
- **调用流程**:
  1. 若 `pc_` 为空 → 已销毁，直接返回
  2. 遍历销毁所有 DataChannel 链表
  3. `pc_->Close()` 关闭 WebRTC 连接
  4. 清理所有 Lua 回调引用（unref + debug 日志）
  5. 释放 `pc_` 和 `factory_`

### `LuaCreate(L)` — 主入口
- **用途**: Lua 侧创建 PeerConnection
- **调用流程**:
  1. 在 Lua 堆上 placement new PeerConnection（通过 `shared_ptr` 持有）
  2. 设置 `lua_co_ = lua_newthread(L)` — 创建回调协程
  3. 解析 options table：
     - `stun` — STUN 服务器 URL
     - `turn` — TURN 服务器 URL（含 username:credential）
     - `audio` / `video` — 是否启用音视频
     - `video_width` / `video_height` / `video_fps` / `video_bitrate` — 视频参数
     - `audio_codec` / `video_codec` / `video_hardware` — 编解码器偏好
  4. 创建 `PeerConnectionFactory`:
     - 使用自定义 `ExternalAudioDeviceModule` (实时音频采集)
     - 自定义 `VideoEncoderFactory` / `VideoDecoderFactory`（支持 H.264/VP8/VP9 选择）
     - 内置音频编解码器工厂
  5. 创建 `PeerConnection` (libwebrtc API):
     - `webrtc::PeerConnectionDependencies` 绑定 CSDO 观察者
     - `webrtc::PeerConnectionInterface::RTCConfiguration` 配置 ICE 服务器
  6. 若启用视频 → 根据 source 类型创建视频源
     - 外部设备 → `ExternalVideoSourceImpl::Create(id)`
     - 本地摄像头 → `CameraVideo::Create()`
     - 桌面采集 → `DesktopVideo::Create()`
  7. `CreateOffer` 发起 SDP 协商

### SDP 协商

#### `OnLocalDescription(desc)` — 本地 SDP 就绪
- **调用流程**:
  1. `pc_->SetLocalDescription(SSDO::Create(), desc)` 设置本地描述
  2. 序列化 SDP → 通过 Lua `ondescription` 回调传递给用户
  3. 用户应通过信令服务器将 SDP 转发给远端

#### `LuaSetRemoteDescription(L)` — 设置远端 SDP
- **调用流程**:
  1. 解析 Lua 传来的 JSON SDP（type + sdp）
  2. `webrtc::CreateSessionDescription(type, sdp)` 反序列化
  3. `pc_->SetRemoteDescription(SSDO::Create(), desc)`

#### `LuaAddRemoteCandidate(L)` — 添加远端 ICE 候选
- **调用流程**:
  1. 解析 JSON：`sdp_mid`、`sdp_mline_index`、`candidate`
  2. `webrtc::CreateIceCandidate(...)` → `pc_->AddIceCandidate(...)`

### ICE 与连接事件

#### `OnIceCandidate(candidate)`
- **用途**: 本地 ICE 候选就绪
- **调用流程**: 序列化为 JSON → 通过 Lua `onicecandidate` 回调传递

#### `OnIceGatheringChange(new_state)`
- **用途**: ICE 收集状态变更
- **调用流程**: 通过 Lua `onicegatheringstatechange` 回调通知

#### `OnConnectionChange(new_state)`
- **用途**: 对等连接状态变更
- **调用流程**: 通过 Lua `onconnectionstatechange` 回调通知
- **状态**: `new` / `connecting` / `connected` / `disconnected` / `failed` / `closed`

### 音视频轨道

#### `LuaAddTrack(L)` — 添加音视频 track
- **用途**: 创建音视频发送器
- **调用流程**:
  1. 解析 track 类型（audio/video）
  2. 音频 → `factory_->CreateAudioTrack(...)` + `pc_->AddTrack(...)`
  3. 视频 → `factory_->CreateVideoTrack(...)` + `pc_->AddTrack(...)`

#### `LuaRemoveTrack(L)` — 移除 track
- **调用流程**: 通过 sender 从 PeerConnection 移除

#### `LuaChangeTrack(L)` — 切换 track 源
- **用途**: 运行时切换音视频源（如切换摄像头）
- **调用流程**: 通过 sender 的 `SetTrack` 方法替换源

### 音视频数据处理

#### `OnFrame(frame)` — 视频帧回调
- **用途**: WebRTC 视频 sink 接口，接收远端视频帧
- **调用流程**: 创建 `VideoFrame` 对象 → 通过 Lua `onvideoframe` 回调传递
- **注意**: 通过 `VideoFrameCmd` 命令投递到工作线程，避免在 WebRTC 回调线程中直接操作 Lua

#### `OnData(audio_data, bits, rate, channels, frames)` — 音频数据回调
- **用途**: WebRTC 音频 sink 接口，接收远端音频帧
- **调用流程**: 拷贝音频数据 → 创建 `AudioFrame` 对象 → 通过 Lua `onaudioframe` 回调传递
- **同样通过命令投递**: `AudioDataCmd` 跨线程传递

#### `OnVideoFrame(frame)` / `OnAudioData(...)` (public method)
- **用途**: 被 `VideoFrameCmd::Process()` / `AudioDataCmd::Process()` 调用
- **核心逻辑**: Resume Lua 协程，执行用户注册的 onvideoframe / onaudioframe 回调

### DataChannel 管理

#### `OnDataChannel(dc)`
- **用途**: 远端创建 DataChannel 时回调
- **调用流程**: 创建 `DataChannel` 对象 → 头插法加入 `dc_list_head_` 链表 → 通过 Lua `ondatachannel` 回调通知

#### `LuaCreateDataChannel(L)`
- **用途**: 本端创建 DataChannel
- **调用流程**: `pc_->CreateDataChannel(label, config)` → 创建 `DataChannel` 并加入链表

### RTC 统计

#### `LuaGetStats(L)` — 获取统计报告
- **调用流程**: `pc_->GetStats(RTCStatsObtainer::Create(pc))` — 异步获取

#### `OnRTCStatsObtained(report)` — 统计就绪
- **调用流程**: 遍历所有统计条目，构造 JSON → 通过 Lua `onrtcstats` 回调传递

### Lua 绑定

| 注册函数 | Lua 模块名 | 说明 |
|---|---|---|
| `LuaOpenPeerConnectionLib` | `peerconnection` | 创建和管理 PeerConnection |
| `LuaOpenDataChannelLib` | `datachannel` | DataChannel 操作 |

**PeerConnection Lua API**:
| 方法 | C 函数 |
|---|---|
| `pc:close()` | `LuaClose` |
| `pc.onxxx = func` | `LuaSetCallback`（支撑 7 种回调） |
| `pc:set_remote_description(sdp)` | `LuaSetRemoteDescription` |
| `pc:add_remote_candidate(candidate)` | `LuaAddRemoteCandidate` |
| `pc:create_offer()` | `LuaCreateOffer` |
| `pc:create_answer()` | `LuaCreateAnswer` |
| `pc:add_track(type, source)` | `LuaAddTrack` |
| `pc:remove_track(track)` | `LuaRemoveTrack` |
| `pc:change_track(track, source)` | `LuaChangeTrack` |
| `pc:create_data_channel(label)` | `LuaCreateDataChannel` |
| `pc:get_stats()` | `LuaGetStats` |

---

## 2. DataChannel — WebRTC 数据通道

### 类结构
```
DataChannel : webrtc::DataChannelObserver
  - 双向链表 (shared_ptr<DataChannel> prev_/next_)
  - 持有 PeerConnection 指针和 DataChannelInterface
```

### `DataChannel(pc, dc)`
- **用途**: 构造并注册为 DataChannel 观察者
- **调用流程**: `dc->RegisterObserver(this)` → 初始化 Lua 回调引用

### `Destroy()`
- **调用流程**: `dc_->UnregisterObserver()` → `dc_->Close()` → 从 pc 的链表中移除 → 清理 Lua 引用

### `OnStateChange()` — 状态变更
- **状态**: `kOpen` → 调用 `onopen` 回调（一次性，调用后 unref）
- `kClosed` → 调用 `onclose` 回调（一次性）

### `OnMessage(buffer)` — 收到消息
- **调用流程**: 调用 Lua `onmessage` 回调，传递 `(type, data)` — type 为 "text" 或 "binary"

### Lua 绑定

| 方法 | C 函数 |
|---|---|
| `dc:close()` | `LuaClose` |
| `dc:sendmsg(text)` | `LuaSendMessage` |
| `dc:send(binary)` | `LuaSendBinary` |
| `dc:label()` | `LuaGetLabel` |
| `dc:id()` | `LuaGetID` |
| `dc.onopen = func` | `LuaSetCallback` |
| `dc.onclose = func` | `LuaSetCallback` |
| `dc.onmessage = func` | `LuaSetCallback` |

---

## 3. VideoFrame / AudioFrame — 媒体帧封装

### VideoFrame

继承 `webrtc::VideoFrame`，添加 Lua 绑定层：

| Lua 属性 | C 函数 | 返回值 |
|---|---|---|
| `frame.id` | `LuaGetID` | `webrtc::VideoFrame::id()` |
| `frame.width` | `LuaGetWidth` | `webrtc::VideoFrame::width()` |
| `frame.height` | `LuaGetHeight` | `webrtc::VideoFrame::height()` |
| `frame.rotation` | `LuaGetRotation` | `webrtc::VideoFrame::rotation()` |

**元表**: `videoframe` — 仅暴露只读属性，无构造器（由 PeerConnection 内部创建）。

### AudioFrame

纯 BeeNet 类（不继承 webrtc 类型），封装 PCM 音频数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `audio_data_` | `unique_ptr<const uint8_t[]>` | PCM 采样数据 |
| `bits_per_sample_` | `int` | 每个采样的位深 |
| `sample_rate_` | `int` | 采样率（Hz） |
| `channels_` | `size_t` | 通道数 |
| `nb_samples_` | `size_t` | 采样帧数 |

**数据大小计算**: `size() = channels_ * nb_samples_ * bits_per_sample_ / 8`

| Lua 属性 | 返回值 |
|---|---|
| `frame.bits_per_sample` | 位深 |
| `frame.sample_rate` | 采样率 |
| `frame.channels` | 通道数 |
| `frame.samples` | 帧数 |

### 跨线程命令转发

```cpp
VideoFrameCmd : BaseCmd  // 持有 webrtc::VideoFrame 副本 + weak_ptr<PeerConnection>
AudioDataCmd  : BaseCmd  // 持有 unique_ptr<uint8_t[]> + weak_ptr<PeerConnection>
```

**设计意图**: WebRTC 回调线程与 BeeManager 工作线程是不同线程。视频/音频帧到达时：
1. WebRTC 线程创建 `VideoFrameCmd` / `AudioDataCmd`（拷贝帧数据）
2. 通过 `BeeManager::PostCmd` 投递到工作线程
3. 工作线程在 `Process()` 中通过 `weak_ptr.lock()` 检查 PeerConnection 是否仍存活
4. 若存活 → 调用 `OnVideoFrame` / `OnAudioData` → resume Lua 协程执行用户回调

---

## 模块5 设计要点总结

| 要点 | 说明 |
|---|---|
| **weak_ptr 安全** | 所有 WebRTC 内部回调通过 `weak_ptr` 访问 PeerConnection，避免悬垂指针 |
| **命令跨线程** | 音视频帧通过 BaseCmd 体系从 WebRTC 线程投递到工作线程 |
| **自定义编解码** | 替换默认编解码工厂，支持 H.264/VP8/VP9 + 硬件加速 |
| **SDP 序列化** | 通过 JSON 在 Lua 和 C++ 之间传递 SDP/ICE 候选 |
| **共享 PeerConnectionFactory** | 所有 PeerConnection 共享同一个 Factory（静态单例模式） |
| **回调一次性** | DataChannel 的 onopen/onclose 调用后自动 unref，防止重复触发 |
| **递归销毁保护** | `Destroy()` 先检查 `pc_` 是否为空，防止重复销毁 |
