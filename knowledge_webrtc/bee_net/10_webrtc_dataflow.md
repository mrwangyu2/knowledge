# BeeNet WebRTC 数据流追踪

> 追踪日期：2026-06-04  
> 追踪链路：SDP 协商 → ICE 连接 → 音视频帧 → DataChannel → 关闭

---

## 线程模型（WebRTC 场景）

```
┌──────────────┐  ┌──────────────────┐  ┌───────────────────┐
│ Caller Thread │  │  Async Worker     │  │  WebRTC Internal   │
│ (用户/信令)    │  │  (BeeManager)     │  │  (signaling/worker)│
├──────────────┤  ├──────────────────┤  ├───────────────────┤
│ bee_send()   │  │  CmdQueue 消费     │  │  PeerConnection   │
│ (发送SDP)     │  │  Lua 协程调度      │  │  Observer 回调     │
│              │  │  curl 事件循环      │  │  Audio/Video Sink │
│              │  │                   │  │  ICE 候选收集      │
└──────────────┘  └──────────────────┘  └───────────────────┘
       │                  │                       │
       │    PostCmd        │    webrtc callback    │
       ├─────────────────→├←──────────────────────┤
       │    promise/future │    PostCmd(VideoFrame)│
       │←─────────────────├──────────────────────→│
```

**4 条线程**：
1. **Caller Thread** — 用户调用 API（bee_send 传 SDP/ICE）
2. **Async Worker** — BeeManager 工作线程（Lua 协程 + 命令消费）
3. **WebRTC Signaling Thread** — libwebrtc 信令线程（Observer 回调）
4. **WebRTC Worker Thread** — libwebrtc 工作线程（编码/传输）

---

## 阶段 1：创建 PeerConnection + SDP Offer

### Step 1.1：Lua 入口 → LuaCreate

Lua 脚本调用 `peerconnection.create(options)`：

```lua
local pc = peerconnection.create({
    stun = "stun:stun.example.com:3478",
    audio = true,
    video = true,
    video_width = 1280,
    video_height = 720,
    video_fps = 30,
})
```

**文件**：`peerconnection.cpp:LuaCreate()`

```cpp
int PeerConnection::LuaCreate(lua_State *L) {
    // ① 创建 shared_ptr<PeerConnection>
    auto shared_pc = std::make_shared<PeerConnection>();

    // ② 创建回调协程（所有 WebRTC 回调在此协程中执行 Lua）
    shared_pc->lua_co_ = lua_newthread(L);

    // ③ 解析 options table
    // STUN/TURN 服务器 URL
    const char *stun_url = lua_tostring(L, -1);  // "stun:stun.example.com:3478"

    // ④ 创建 PeerConnectionFactory（全局共享或新建）
    //    └─ ExternalAudioDeviceModule（自定义 ADM，支持外部音频注入）
    //    └─ CreateVideoEncoderFactory()（H.264/VP8/VP9 + 平台硬编）
    //    └─ CreateVideoDecoderFactory()（同上）
    //    └─ CreateBuiltinAudioEncoderFactory()
    //    └─ CreateBuiltinAudioDecoderFactory()
    webrtc::PeerConnectionFactoryDependencies pcf_deps;
    pcf_deps.task_queue_factory = CreateDefaultTaskQueueFactory();
    pcf_deps.audio_encoder_factory = CreateBuiltinAudioEncoderFactory();
    pcf_deps.audio_decoder_factory = CreateBuiltinAudioDecoderFactory();
    pcf_deps.video_encoder_factory = CreateVideoEncoderFactory();
    pcf_deps.video_decoder_factory = CreateVideoDecoderFactory();
    pcf_deps.adm = CreateExternalAudioDeviceModule(env, audio_input, nullptr);

    shared_pc->factory_ = CreateModularPeerConnectionFactory(std::move(pcf_deps));

    // ⑤ 配置 ICE 服务器
    webrtc::PeerConnectionInterface::RTCConfiguration config;
    config.servers.push_back(/* stun/turn */);

    // ⑥ 创建 PeerConnection（注册为 Observer）
    webrtc::PeerConnectionDependencies pc_deps(CSDO::Create(shared_pc));
    shared_pc->pc_ = shared_pc->factory_->CreatePeerConnection(
        config, std::move(pc_deps));

    // ⑦ 创建视频源（如果启用视频）
    if (video) {
        if (external_source_id)
            video_source = ExternalVideoSourceImpl::Get(id);
        else
            video_source = CameraVideoTrackSource::Create(0);

        video_source->Init(1280, 720, 30);
        // ... 创建 VideoTrack → pc_->AddTrack(sender, track)
    }

    // ⑧ 发起 SDP Offer
    shared_pc->pc_->CreateOffer(CSDO::Create(shared_pc), options);
    //    CSDO 持有 weak_ptr<PeerConnection>
    //    成功时 → OnLocalDescription(desc) 在 WebRTC Signaling Thread 中被回调
}
```

### Step 1.2：VideoTrack 创建的设备链

```
CameraVideoTrackSource::Create(0)
  │
  ├─ webrtc::VideoCaptureFactory::CreateDeviceInfo()
  │    └─ 枚举系统摄像头（Windows: DirectShow/MF, Linux: V4L2, macOS: AVFoundation）
  │
  ├─ info->GetDeviceName(0, ...)  // "Front Camera" / "Built-in iSight"
  │
  └─ CameraVideoTrackSource(unique_name)
       │
       └─ Init(1280, 720, 30)
            │
            ├─ VideoCaptureFactory::Create(unique_name)
            │    └─ 打开摄像头设备
            │
            ├─ RegisterCaptureDataCallback(this)
            │    └─ this = CameraVideoTrackSource（继承 VideoTrackSource）
            │
            ├─ StartCapture({1280, 720, 30, kI420})
            │    └─ 摄像头开始推流 → OnFrame 回调
            │
            └─ 摄像头帧 → OnFrame(frame)
                          │
                          ├─ video_adapter_.AdaptFrameResolution(...)
                          │    └─ 根据 sink wants 缩放/跳帧
                          │
                          └─ broadcaster_.OnFrame(scaled_frame)
                               │
                               └─ → WebRTC VideoTrack → Encoder → 网络发送
```

---

## 阶段 2：SDP 协商

### Step 2.1：Offer 创建 → 本地 SDP 就绪

```
pc_->CreateOffer(CSDO, options)
  │  [WebRTC Signaling Thread 回调]
  ▼
CSDO::OnSuccess(SessionDescriptionInterface *desc)
  │  shared_pc = pc_.lock()  ← 检查 PeerConnection 是否仍存活
  ▼
PeerConnection::OnLocalDescription(desc)
  │
  ├─ pc_->SetLocalDescription(SSDO::Create(), desc)
  │    └─ 设置本地 SDP（WebRTC 内部操作）
  │
  ├─ std::string sdp;
  │   desc->ToString(&sdp);  // 序列化 SDP 文本
  │
  └─ 通过 Lua "ondescription" 回调传递 SDP
       │
       ├─ lua_rawgeti(lua_co_, REGISTRYINDEX, ref_on_description_)
       ├─ lua_pushstring(lua_co_, "offer")       // type
       ├─ lua_pushstring(lua_co_, sdp.c_str())   // SDP 文本
       └─ lua_pcall(lua_co_, 2, 0, 0)
            └─ Lua 回调函数收到 (type, sdp)
```

**此时 Lua 回调中的代码**（用户脚本）：
```lua
function pc.ondescription(type, sdp)
    -- 通过信令通道发送 SDP 到远端
    signaling_channel:send(json.encode({type=type, sdp=sdp}))
end
```

### Step 2.2：接收远端 SDP → SetRemoteDescription

远端的 SDP 通过信令通道传回，用户线程调用：

```cpp
// 假设信令消息到达 → 用户线程调用 bee_send 传递 SDP
// 实际流程：Lua 脚本调用 pc:set_remote_description(json_sdp)
```

```cpp
int PeerConnection::LuaSetRemoteDescription(lua_State *L) {
    // ① 解析 JSON：{"type":"answer", "sdp":"v=0\r\no=..."}
    const char *json = lua_tostring(L, 2);
    // 简单 JSON 解析 → 提取 type 和 sdp

    // ② 反序列化 SDP
    webrtc::SessionDescriptionInterface *desc = nullptr;
    webrtc::SdpParseError error;
    desc = webrtc::CreateSessionDescription(type, sdp, &error);

    // ③ 设置远端描述
    auto shared_pc = ...; // 从 Lua userdata 恢复
    shared_pc->pc_->SetRemoteDescription(SSDO::Create(), desc);
    //    └─ SSDO::OnSuccess() 空实现（由 OnConnectionChange 通知状态变更）
}
```

---

## 阶段 3：ICE 候选交换

### Step 3.1：本地 ICE 候选 → Lua 回调

WebRTC 内部收集 ICE 候选时：

```
[WebRTC Signaling Thread]
OnIceCandidate(const IceCandidateInterface *candidate)
  │
  ├─ std::string sdp_mid = candidate->sdp_mid();
  ├─ int sdp_mline_index = candidate->sdp_mline_index();
  ├─ std::string sdp;
  │   candidate->ToString(&sdp);  // "candidate:... 1 UDP 2122252543 192.168.1.1 12345 typ host"
  │
  └─ Lua "onicecandidate" 回调
       ├─ lua_rawgeti(lua_co_, REGISTRYINDEX, ref_on_ice_candidate_)
       ├─ lua_pushstring(lua_co_, json_str)  // JSON: {mid, index, candidate}
       └─ lua_pcall(lua_co_, 1, 0, 0)
```

### Step 3.2：接收远端 ICE 候选

```cpp
int PeerConnection::LuaAddRemoteCandidate(lua_State *L) {
    // ① 解析 JSON candidate
    // {"sdp_mid":"0", "sdp_mline_index":0, "candidate":"candidate:... 1 UDP ..."}
    
    // ② 创建 IceCandidateInterface
    webrtc::SdpParseError error;
    auto candidate = webrtc::CreateIceCandidate(mid, mline_index, candidate_str, &error);

    // ③ 添加到 PeerConnection
    shared_pc->pc_->AddIceCandidate(std::move(candidate), 
        [](webrtc::RTCError error) {
            // 异步回调：ICE 候选添加结果
        });
}
```

### Step 3.3：ICE 状态变更 → 连接建立

```
OnIceGatheringChange(new_state)
  │
  ├─ kIceGatheringNew       → 开始收集
  ├─ kIceGatheringGathering → 收集中
  └─ kIceGatheringComplete  → 收集完成
       └─ Lua "onicegatheringstatechange" 回调

OnConnectionChange(new_state)
  │
  ├─ kNew          → 初始
  ├─ kConnecting   → ICE 连接中
  ├─ kConnected    → ✓ 连接建立
  ├─ kDisconnected → 断开（可能重连）
  ├─ kFailed       → 失败
  └─ kClosed       → 已关闭
       └─ Lua "onconnectionstatechange" 回调(state_string)
```

---

## 阶段 4：媒体轨道建立

### Step 4.1：创建本地音频 Track

```cpp
int PeerConnection::LuaAddTrack(lua_State *L) {
    const char *type = lua_tostring(L, 2);  // "audio" or "video"
    
    if (audio) {
        // 创建音频源
        auto audio_source = shared_pc->factory_->CreateAudioSource(cricket::AudioOptions());
        
        // 创建音频 Track
        auto audio_track = shared_pc->factory_->CreateAudioTrack("audio0", audio_source);
        
        // 添加到 PeerConnection
        auto sender = shared_pc->pc_->AddTrack(audio_track, {"stream0"});
        
        // sender 可用于后续 RemoveTrack/ChangeTrack
    }
}
```

### Step 4.2：外部音频数据注入路径

```
用户代码（任意线程）
  │
  ├─ 创建 ExternalAudioSource 子类
  │    audio_source = MyAudioSource()
  │    source_id = audio_source:RegisterExternalSource()
  │
  └─ 持续推送 PCM 数据
       audio_source:IncomingData(pcm_data, 16, 44100, 2, 1024, kDefault)
         │
         ├─ ConvertToDefaultPCM(in, 16, 2, 1024, format, out)
         │    └─ 统一转换为 16-bit interleaved 平台默认字节序
         │
         ├─ ExternalAudioSourceImpl::Get(source_id)
         │    └─ 从全局 map 查找已注册的音频源
         │
         └─ DeliverRecordedData(pcm_16bit, 44100, 2, 1024)
              │
              ├─ FineAudioBuffer::DeliverRecordedData(...)
              │    ├─ 重采样/通道转换（如果需要）
              │    └─ 切分为 10ms WebRTC 标准帧长
              │         └─ 44100Hz × 0.01s = 441 samples/frame
              │
              ├─ audio_device_buffer_->SetRecordedBuffer(audio_frame)
              │
              └─ audio_device_buffer_->DeliverRecordedData()
                   │
                   └─ → AudioTransport → AudioProcessing → Encoder → 网络发送
```

### Step 4.3：外部视频数据注入路径

```
用户代码
  │
  └─ video_source:IncomingFrame(rgb_data, len, 1920, 1080, kRGB24)
       │
       ├─ ExternalVideoSourceImpl::Get(source_id)
       │
       └─ OnFrame(VideoFrame)
            │
            ├─ 格式转换（如 RGB24 → I420 via libyuv）
            │
            ├─ video_adapter_.AdaptFrameResolution(...)
            │    └─ 根据编码器需求缩放/裁剪
            │
            └─ broadcaster_.OnFrame(i420_frame)
                 │
                 └─ → VideoTrack → VideoEncoder → RTP Packetizer → 网络发送
```

---

## 阶段 5：远端媒体接收

### Step 5.1：远端 Track 到达

```
[WebRTC Signaling Thread]
OnTrack(transceiver)
  │
  ├─ transceiver->receiver()->track()
  │    ├─ 音频 track → 通过 AudioTrackSinkInterface 接收
  │    └─ 视频 track → 通过 VideoSinkInterface 接收
  │
  └─ 注册 Sink：
       video_track->AddOrUpdateSink(this, wants)
       // this = PeerConnection（实现了 VideoSinkInterface）
```

### Step 5.2：视频帧到达 → 跨线程投递

**[WebRTC Worker/Decoder Thread]**
```
远端视频帧解码完成
  │
  ▼
PeerConnection::OnFrame(const VideoFrame &frame)    ← VideoSinkInterface 回调
  │  [在 WebRTC 内部线程中执行！不能直接操作 Lua]
  │
  ├─ 创建 VideoFrameCmd
  │    cmd->weak_pc = shared_from_this()    // weak_ptr 安全引用
  │    cmd->frame = frame;                  // 拷贝视频帧数据
  │
  └─ BeeManager::PostCmd(cmd)              // 投递到工作线程
       │
       ▼
[Async Worker Thread]
VideoFrameCmd::Process()
  │
  ├─ shared_pc = weak_pc.lock()            // 检查 PeerConnection 是否存活
  │
  ├─ shared_pc->OnVideoFrame(frame)         // 现在在工作线程中执行
  │    │
  │    ├─ 创建 VideoFrame Lua 对象
  │    │    lua_newuserdata(L, sizeof(VideoFrame))
  │    │    new(frame_obj) VideoFrame(frame)  // 再次拷贝
  │    │
  │    └─ Lua "onvideoframe" 回调
  │         ├─ lua_rawgeti(lua_co_, REGISTRYINDEX, ref_on_video_frame_)
  │         ├─ lua_pushlightuserdata(...) → VideoFrame userdata
  │         └─ lua_pcall(lua_co_, 1, 0, 0)
  │              └─ Lua 回调函数收到 videoframe 对象
```

**视频帧的 2 次拷贝**：
1. WebRTC 解码输出 → `VideoFrameCmd::frame`（第一次拷贝）
2. `VideoFrameCmd::frame` → `new VideoFrame(frame)`（第二次拷贝，在 Lua 堆中）

### Step 5.3：音频帧到达 → 跨线程投递

```
[WebRTC Audio Thread]
远端音频解码完成
  │
  ▼
PeerConnection::OnData(audio_data, bits, rate, channels, frames)  ← AudioTrackSinkInterface
  │  [在 WebRTC 内部线程中执行]
  │
  ├─ 创建 AudioDataCmd
  │    cmd->weak_pc = shared_from_this()
  │    cmd->data = make_unique<uint8_t[]>(size)  // 拷贝音频数据
  │    memcpy(cmd->data.get(), audio_data, size)
  │    cmd->bits_per_sample = bits
  │    cmd->sample_rate = rate
  │    cmd->number_of_channels = channels
  │    cmd->number_of_frames = frames
  │
  └─ BeeManager::PostCmd(cmd)
       │
       ▼
[Async Worker Thread]
AudioDataCmd::Process()
  │
  ├─ shared_pc = weak_pc.lock()
  │
  └─ shared_pc->OnAudioData(move(data), bits, rate, channels, frames)
       │
       ├─ 创建 AudioFrame Lua 对象
       │    lua_newuserdata(L, sizeof(AudioFrame))
       │    new(frame_obj) AudioFrame(data.get(), bits, rate, channels, frames)
       │
       └─ Lua "onaudioframe" 回调
            └─ Lua 回调函数收到 audioframe 对象
```

---

## 阶段 6：DataChannel 数据流

### Step 6.1：创建本地 DataChannel

```cpp
int PeerConnection::LuaCreateDataChannel(lua_State *L) {
    const char *label = lua_tostring(L, 2);

    // ① 创建 webrtc::DataChannel
    webrtc::DataChannelInit config;
    auto dc = shared_pc->pc_->CreateDataChannel(label, &config);

    // ② 创建 BeeNet DataChannel 包装
    auto bee_dc = std::make_shared<DataChannel>(shared_pc.get(), dc);

    // ③ 加入 PeerConnection 的 DataChannel 链表
    bee_dc->next_ = shared_pc->dc_list_head_;
    shared_pc->dc_list_head_ = bee_dc;

    // ④ 注册为 Observer
    dc->RegisterObserver(bee_dc.get());
}
```

### Step 6.2：远端 DataChannel 到达

```
[WebRTC Signaling Thread]
OnDataChannel(DataChannelInterface *dc)
  │
  ├─ auto bee_dc = make_shared<DataChannel>(this, dc)
  ├─ 加入 dc_list_head_ 链表
  │
  └─ Lua "ondatachannel" 回调
       ├─ lua_rawgeti(lua_co_, REGISTRYINDEX, ref_on_data_channel_)
       ├─ lua_pushlightuserdata → DataChannel userdata
       └─ lua_pcall(lua_co_, 1, 0, 0)
```

### Step 6.3：DataChannel 消息收发

**发送**（Lua 侧 `dc:sendmsg("hello")`）：

```
Lua dc:sendmsg("hello")
  │
  ▼
DataChannel::LuaSendMessage(L)
  │
  ├─ webrtc::DataBuffer buffer("hello")
  │
  └─ dc_->Send(buffer)
       │
       └─ → SCTP → DTLS → ICE → 网络 → 远端
```

**接收**（远端发来消息）：

```
[WebRTC Signaling Thread]
DataChannel::OnMessage(const DataBuffer &buffer)
  │
  ├─ buffer.binary ? "binary" : "text"
  ├─ buffer.data.data<char>() → 消息内容
  │
  └─ Lua "onmessage" 回调
       ├─ lua_rawgeti(pc->lua_co_, REGISTRYINDEX, ref_on_message_)
       ├─ lua_pushstring("text")
       ├─ lua_pushlstring(data, size)
       └─ lua_pcall(lua_co_, 2, 0, 0)
```

**注意**：DataChannel 的 `OnMessage` 回调直接在 WebRTC Signaling Thread 中执行 Lua（不是在 Async Worker 中）。这与音频/视频帧不同，意味着 DataChannel 消息处理**不能有阻塞操作**。

---

## 阶段 7：RTC 统计获取

```
Lua pc:get_stats()
  │
  ▼
PeerConnection::LuaGetStats(L)
  │
  └─ pc_->GetStats(RTCStatsObtainer::Create(shared_pc))
       │  [异步操作]
       │
       ▼
[WebRTC Signaling Thread]
RTCStatsObtainer::OnStatsDelivered(report)
  │
  └─ shared_pc->OnRTCStatsObtained(report)
       │
       ├─ 遍历所有 stats 条目
       │    for (auto &stat : *report) {
       │         stat.type()    // "inbound-rtp", "outbound-rtp", "candidate-pair", ...
       │         stat.id()
       │         stat.values()  // 键值对
       │    }
       │
       ├─ 构造 JSON
       │
       └─ Lua "onrtcstats" 回调
            └─ lua_pcall(lua_co_, 1, 0, 0)
```

---

## 阶段 8：连接关闭

```
Lua pc:close()
  │
  ▼
PeerConnection::LuaClose(L)
  │
  └─ shared_pc->Destroy()
       │
       ├─ 遍历销毁所有 DataChannel
       │    while (dc_list_head_) dc_list_head_->Destroy()
       │    │
       │    └─ DataChannel::Destroy()
       │         ├─ dc_->UnregisterObserver()
       │         ├─ dc_->Close()
       │         ├─ 从 pc 链表中移除自身
       │         └─ 清理所有 Lua 回调引用
       │
       ├─ pc_->Close()
       │    └─ 触发 OnConnectionChange(kClosed)
       │
       ├─ 清理 Lua 引用：
       │    ├─ ref_on_description_
       │    ├─ ref_on_ice_candidate_
       │    ├─ ref_on_ice_gathering_state_change_
       │    ├─ ref_on_connection_state_change_
       │    ├─ ref_on_data_channel_
       │    ├─ ref_on_video_frame_
       │    ├─ ref_on_audio_frame_
       │    ├─ ref_on_rtc_stats_
       │    └─ ref_cb_co_ (回调协程)
       │
       ├─ pc_.reset()
       └─ factory_.reset()
```

---

## 跨线程回调执行位置总览

| 回调事件 | 执行线程 | Lua 调用方式 |
|---|---|---|
| OnIceCandidate | WebRTC Signaling | 直接 lua_pcall (在 webrtc 线程) |
| OnIceGatheringChange | WebRTC Signaling | 直接 lua_pcall |
| OnConnectionChange | WebRTC Signaling | 直接 lua_pcall |
| OnDataChannel | WebRTC Signaling | 直接 lua_pcall |
| OnTrack | WebRTC Signaling | 直接 lua_pcall |
| DataChannel::OnMessage | WebRTC Signaling | 直接 lua_pcall |
| OnFrame (视频帧) | WebRTC Worker/Decoder | PostCmd → Async Worker → lua_pcall |
| OnData (音频帧) | WebRTC Audio | PostCmd → Async Worker → lua_pcall |
| OnRTCStatsObtained | WebRTC Signaling | 直接 lua_pcall |

**重要发现**：ICE/连接/DataChannel 消息的回调在 **WebRTC 信令线程**中直接调用 `lua_pcall`，而音视频帧通过 `PostCmd` 转发到 **Async Worker 线程**。这意味着：
- 信令回调中的 Lua 代码也有「不能阻塞」的约束
- 视频/音频帧的跨线程转发引入了一帧延迟（PostCmd + 事件循环）

---

## 完整时间线（Offer-Answer 场景）

```
T0:  Lua: pc = peerconnection.create({stun="...", video=true})
T1:    ├─ PeerConnection created
T2:    ├─ PeerConnectionFactory created (ADM + codec factories)
T3:    ├─ CameraVideoTrackSource created → Init(1280,720,30)
T4:    ├─ VideoTrack created → AddTrack
T5:    └─ CreateOffer() → CSDO

T6:  [WebRTC Signaling] CSDO::OnSuccess(offer_sdp)
T7:    └─ OnLocalDescription(offer_sdp)
T8:        ├─ SetLocalDescription
T9:        └─ Lua ondescription("offer", sdp) → 信令通道发送

T10: [经由网络/信令服务器]
     --- 远端收到 offer, 创建 answer ---

T11: Lua: pc:set_remote_description(json_answer)     [Caller Thread]
T12:   └─ CreateSessionDescription → SetRemoteDescription

T13: [WebRTC Signaling] ICE 候选开始
T14: OnIceCandidate(candidate_x_N)
T15:   └─ Lua onicecandidate(json) → 信令通道发送

T16: Lua: pc:add_remote_candidate(json)  × N          [Caller Thread]
T17:   └─ CreateIceCandidate → AddIceCandidate

T18: [WebRTC Signaling] OnConnectionChange(kConnected)
T19:   └─ Lua onconnectionstatechange("connected")

T20: [WebRTC Signaling] OnTrack(transceiver)
T21:   └─ AddOrUpdateSink(this) → 注册视频/音频 Sink

T22: [WebRTC Decoder] 远端视频帧解码完成
T23:   └─ OnFrame(frame)
T24:       └─ PostCmd(VideoFrameCmd) → Async Worker
T25:           └─ Lua onvideoframe(vf) → 用户收到视频帧
```

---

## 设计要点

| 要点 | 说明 |
|---|---|
| **weak_ptr 安全** | 所有 WebRTC 内部回调通过 `weak_ptr` 访问 PeerConnection，防止对象已销毁 |
| **帧跨线程投递** | 音视频帧通过 BaseCmd 体系从 WebRTC 线程投递到工作线程 |
| **DataChannel 直接回调** | 消息回调在信令线程直接 lua_pcall，无额外转发 |
| **2 次帧拷贝** | 视频帧：解码输出→VideoFrameCmd→Lua VideoFrame（2 次深拷贝） |
| **FineAudioBuffer** | 将任意长度 PCM 切分为 10ms WebRTC 标准帧长 |
| **全局源注册表** | 外部音视频源通过 ID→对象映射实现跨模块注入绑定 |
