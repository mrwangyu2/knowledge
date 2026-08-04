---
title: WebRTC C++ API 使用方法总结
type: tutorial
tags: [webrtc, api, peerconnection, tutorial, cpp]
sources: [examples/peerconnection/client/, wiki/protocols/protocol-rfc8445, wiki/protocols/protocol-rfc3550]
created: 2026-06-04
updated: 2026-06-04
---

# WebRTC C++ API 使用方法总结

> 基于 `examples/peerconnection/client/` 官方示例，提炼 WebRTC Native API 的核心使用模式。读者应已了解 RTP/RTCP、ICE/STUN/TURN、SDP 基础。

---

## 架构总览

WebRTC API 采用**三层工厂模型**：

```
┌──────────────────────────────────────────────────┐
│  L1: PeerConnectionFactoryInterface               │
│      └─ 创建 PeerConnection + 音视频 Track        │
│               │                                  │
│  L2: PeerConnectionInterface                      │
│      └─ 管理 SDP 协商 + ICE 候选交换 + 媒体收发     │
│               │                                  │
│  L3: MediaStream / Track / RtpSender / RtpReceiver│
│      └─ 承载实际的音视频数据流                      │
│                                                  │
│  旁路: PeerConnectionObserver                     │
│      └─ 异步事件回调 (Track 到达、ICE 候选、状态变化) │
└──────────────────────────────────────────────────┘
```

**一句话**: `Factory` 造 `PeerConnection`，`PeerConnection` 管协商，`Track` 传数据，`Observer` 收事件。

---

## 完整流程：从启动到通信

```
 +--------+        +--------+        +--------+        +--------+
 | main   |        |Factory |        |  PC    |        |Observer|
 +---+----+        +---+----+        +---+----+        +---+----+
     │                  │                │                  │
     │ 1.CreateFactory   │                │                  │
     │─────────────────▶│                │                  │
     │                  │                │                  │
     │ 2.CreatePeerConnection()          │                  │
     │─────────────────────────────────▶│                  │
     │                  │                │                  │
     │ 3.AddTrack(audio+video)           │                  │
     │─────────────────────────────────▶│                  │
     │                  │                │                  │
     │ 4.CreateOffer/Answer (SDP协商)     │                  │
     │─────────────────────────────────▶│                  │
     │                  │                │                  │
     │ 5.SetLocalDescription + SetRemoteDescription        │
     │─────────────────────────────────▶│                  │
     │                  │                │                  │
     │ 6.ICE Candidate Discovery ────────────────────▶  │
     │                  │           OnIceCandidate(c)     │
     │                  │                │                  │
     │ 7.AddIceCandidate (交换远端候选)                      │
     │─────────────────────────────────▶│                  │
     │                  │                │                  │
     │ 8.ICE 连通 ───────────────────────────────────▶  │
     │                  │           OnAddTrack(track)     │
     │                  │                │                  │
     │                  │   RTP 媒体流直通    │                  │
     │                  │◀══════════════════════▶│        │
     +                  +                +                  +
```

**分步详解**:

---

## 第一步：创建 Factory

`PeerConnectionFactoryInterface` 是所有对象的"工厂车间"，只需创建一次。

```cpp
// 1. 准备依赖 (线程 + 编解码器)
auto signaling_thread = webrtc::Thread::CreateWithSocketServer();
signaling_thread->Start();

webrtc::PeerConnectionFactoryDependencies deps;
deps.signaling_thread = signaling_thread.get();              // 信令线程
deps.audio_encoder_factory = CreateBuiltinAudioEncoderFactory();  // 音频编码
deps.audio_decoder_factory = CreateBuiltinAudioDecoderFactory();  // 音频解码
deps.video_encoder_factory = /* VP8/VP9/H264/AV1 模板 */;         // 视频编码
deps.video_decoder_factory = /* VP8/VP9/H264/AV1 模板 */;         // 视频解码
EnableMedia(deps);                                           // 启用媒体

// 2. 创建 factory
auto factory = CreateModularPeerConnectionFactory(std::move(deps));
```

| 依赖项 | 作用 | 是否必须 |
|--------|------|:---:|
| `signaling_thread` | 执行所有回调的线程 | ✅ |
| `audio_encoder_factory` | 音频编码器工厂 | 需要音频时 |
| `audio_decoder_factory` | 音频解码器工厂 | 需要音频时 |
| `video_encoder_factory` | 视频编码器工厂 | 需要视频时 |
| `video_decoder_factory` | 视频解码器工厂 | 需要视频时 |

---

## 第二步：创建 PeerConnection

```cpp
// 1. 配置 (ICE 服务器 + SDP 语义)
webrtc::PeerConnectionInterface::RTCConfiguration config;
config.sdp_semantics = webrtc::SdpSemantics::kUnifiedPlan;  // 现代 SDP

webrtc::PeerConnectionInterface::IceServer server;
server.uri = "stun:stun.l.google.com:19302";               // STUN 地址
server.username = "...";                                    // TURN 用户名
server.password = "...";                                    // TURN 密码
config.servers.push_back(server);

// 2. 创建 (注入 Observer)
webrtc::PeerConnectionDependencies pc_deps(observer);  // observer 接收回调
auto result = factory->CreatePeerConnectionOrError(config, std::move(pc_deps));
auto peer_connection = std::move(result.value());
```

**Config 核心字段**:

| 字段 | 含义 | 默认 |
|------|------|------|
| `servers` | STUN/TURN 服务器列表 | 空 (仅 Host 候选) |
| `sdp_semantics` | `kPlanB` (旧) / `kUnifiedPlan` (新) | `kPlanB` |

> **UnifiedPlan vs PlanB**: UnifiedPlan 一条 PeerConnection 可收多路视频 (每路一个 `m=` 行)，是现代 WebRTC 的标准模式。

---

## 第三步：添加本地媒体轨道

```cpp
// 音频 Track
auto audio_source = factory->CreateAudioSource(AudioOptions());
auto audio_track = factory->CreateAudioTrack("audio_label", audio_source.get());
peer_connection->AddTrack(audio_track, {"stream_id"});

// 视频 Track (需要摄像头或文件输入)
auto video_source = /* 从摄像头或 FrameGenerator 创建 */;
auto video_track = factory->CreateVideoTrack(video_source, "video_label");
peer_connection->AddTrack(video_track, {"stream_id"});
```

**Track 层级**:

```
AudioSource ──▶ AudioTrack ──▶ RtpSender  ──▶ PeerConnection
VideoSource ──▶ VideoTrack ──▶ RtpSender  ──▶ PeerConnection
```

> `AddTrack()` 之后，后续 SDP 协商会自动带上这些 track 的信息。

---

## 第四步：SDP 协商 (Offer/Answer)

这是信令层的核心——通过信令服务器交换 SDP。

### 发起方 (Offerer)

```cpp
// 发起方: CreateOffer → 获得本地 SDP → 发送给对端
peer_connection->CreateOffer(observer, RTCOfferAnswerOptions());
// 回调:
void OnSuccess(SessionDescriptionInterface* desc) override {
    peer_connection->SetLocalDescription(dummy_observer, desc);
    SendToPeer(desc);   // 通过信令通道发给对端
}
```

### 应答方 (Answerer)

```cpp
// 应答方: 收到 Offer → SetRemote → CreateAnswer → 发回
void OnMessageFromPeer(SessionDescription* desc) override {
    peer_connection->SetRemoteDescription(dummy_observer, desc);
    if (desc->GetType() == SdpType::kOffer) {
        peer_connection->CreateAnswer(observer, RTCOfferAnswerOptions());
    }
}
```

**SDP 协商双向流**:

```
Offerer                              Answerer
   │                                    │
   │─ CreateOffer() ──────────────────▶ │
   │   SetLocalDescription(offer)       │
   │                                    │
   │──── 信令通道传 SDP ──────────────▶│
   │                                    │ SetRemoteDescription(offer)
   │                                    │ CreateAnswer()
   │                                    │ SetLocalDescription(answer)
   │◀─── 信令通道传 SDP ───────────────│
   │                                    │
   │ SetRemoteDescription(answer)        │
   │                                    │
   │  ◀══════ ICE 连通性检查 ═══════▶ │
```

> **关键**: `SetLocalDescription` 必须在发送 SDP 之前，`SetRemoteDescription` 必须在收到 SDP 之后。

---

## 第五步：ICE 候选交换

SDP 设置完成后，ICE 自动开始收集候选。每当新候选出现，`OnIceCandidate` 回调触发：

```cpp
void OnIceCandidate(const IceCandidate* candidate) override {
    // 将候选序列化为 JSON，通过信令通道发给对端
    Json::Value msg;
    msg["sdpMid"] = candidate->sdp_mid();
    msg["sdpMLineIndex"] = candidate->sdp_mline_index();
    msg["candidate"] = candidate->ToString();
    SendToPeer(msg);
}
```

收到对端候选后，添加到本地 PeerConnection：

```cpp
void OnRemoteCandidate(Json::Value msg) override {
    auto candidate = CreateIceCandidate(
        msg["sdpMid"], msg["sdpMLineIndex"], msg["candidate"]);
    peer_connection->AddIceCandidate(candidate.get());
}
```

> **Trickle ICE**: 候选逐个发送 (非等收集完再发)，减少连接建立延迟。这也是示例的默认行为。

---

## 第六步：接收远端媒体

当 ICE 连通后，对端的 Track 到达，触发 `OnAddTrack`：

```cpp
void OnAddTrack(
    scoped_refptr<RtpReceiverInterface> receiver,
    const vector<scoped_refptr<MediaStreamInterface>>& streams) override {
    auto* track = receiver->track().get();
    if (track->kind() == MediaStreamTrackInterface::kVideoKind) {
        auto* video_track = static_cast<VideoTrackInterface*>(track);
        renderer->StartRemoteRenderer(video_track);  // 渲染远端视频
    }
}
```

> **无需手动"接收"** — ICE 连通后 RTP 流自动到达，你只需要渲染到达的 Track。

---

## Observer 回调速查

| 回调 | 触发时机 | 你需要做什么 |
|------|---------|------------|
| `OnAddTrack` | ICE 连通后对端 Track 到达 | 渲染音视频 |
| `OnRemoveTrack` | 对端停止发送 | 停止渲染 |
| `OnIceCandidate` | 本地 ICE 候选就绪 | 发候选给对端 (信令) |
| `OnIceConnectionChange` | ICE 状态变化 | 更新 UI 连接状态 |
| `OnIceGatheringChange` | 候选收集状态变化 | 可忽略或跟踪进度 |
| `OnSignalingChange` | SDP 状态变化 | 调试用 |
| `OnDataChannel` | 对端创建数据通道 | 绑定 DataChannel 事件 |

---

## 完整生命周期

```
┌──────────┐   CreateFactory    ┌──────────┐   CreatePC    ┌──────────┐
│  main()  │ ─────────────────▶ │ Factory  │ ───────────▶ │    PC    │
└──────────┘                    └──────────┘               └──────────┘
                                                               │
  1. AddTrack(audio+video) ──────────────────────────────────▶ │
  2. CreateOffer/CreateAnswer ───────────────────────────────▶ │
  3. SetLocalDescription (回调 OnSuccess) ◀──────────────────── │
  4. 信令交换 SDP ──────────────────────────────────────────── │
  5. SetRemoteDescription ───────────────────────────────────▶ │
  6. OnIceCandidate ──▶ 信令交换候选 ──▶ AddIceCandidate ────── │
  7. ICE 连通 ◀──────────────────────────────────────────────── │
  8. OnAddTrack ──▶ 渲染远端媒体                                │
                                                               │
  9. 通信结束: DeletePeerConnection ──────────────────────────▶ │
     (先设 peer_connection_ = nullptr, 再 factory = nullptr)    │
```

**析构顺序必须是**：`PeerConnection` → `PeerConnectionFactory`。因为 Factory 创建的对象持有 Factory 的引用，PC 必须先释放。

---

## 关键 API 速查表

| 接口 | 核心方法 | 用途 |
|------|---------|------|
| `PeerConnectionFactoryInterface` | `CreatePeerConnectionOrError()` | 创建 PC |
| | `CreateAudioTrack()` | 创建音频轨道 |
| | `CreateVideoTrack()` | 创建视频轨道 |
| | `CreateAudioSource()` | 创建音频源 |
| `PeerConnectionInterface` | `CreateOffer()` / `CreateAnswer()` | SDP 协商 |
| | `SetLocalDescription()` | 设置本地 SDP |
| | `SetRemoteDescription()` | 设置远端 SDP |
| | `AddTrack()` | 添加媒体轨道 |
| | `AddIceCandidate()` | 添加远端 ICE 候选 |
| | `GetSenders()` / `GetReceivers()` | 查询发送/接收器 |
| `PeerConnectionObserver` | `OnAddTrack()` | 远端 Track 到达 |
| | `OnIceCandidate()` | 本地候选就绪 |
| | `OnIceConnectionChange()` | ICE 状态变化 |
| `CreateSessionDescriptionObserver` | `OnSuccess(SDP*)` | Offer/Answer 创建完成 |
| | `OnFailure(RTCError)` | SDP 创建失败 |

---

## 一句话总结

> **Factory 造 PC → PC 管 SDP+ICE → Observer 收事件 → Track 传媒体**。信令 (SDP+ICE候选) 走外挂通道，媒体 (RTP) 走 P2P 直连。AddTrack → CreateOffer → SetLocalDescription → 信令传 SDP → SetRemoteDescription → 交换 ICE 候选 → ICE 连通 → OnAddTrack → 渲染。

---

## 相关链接

- [[wiki/tutorials/tutorial-ice-candidate-gathering]] — ICE 候选收集详细流程
- [[wiki/tutorials/tutorial-ice-connectivity-checks]] — ICE 连接性检查与提名详解
- [[wiki/protocols/protocol-rfc8445]] — ICE 协议详细分析
- [[wiki/protocols/protocol-rfc3550]] — RTP/RTCP 协议分析
- [[wiki/protocols/protocol-rfc4566]] — SDP 协议分析
- [[wiki/libnice/libnice-error-handling]] — libnice 错误处理分析
- [[raw/rfcs/rfc4585_完整中英对照]] — RTP/AVPF 中英对照
