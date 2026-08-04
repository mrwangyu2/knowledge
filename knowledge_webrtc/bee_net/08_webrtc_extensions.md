# BeeNet 模块8：WebRTC 扩展

> 精读日期：2026-06-04  
> 覆盖目录：`extensions/webrtc/`（10 个文件）

---

## 文件概览

```
extensions/webrtc/
├── video_encoder_factory.h/cpp    (自定义视频编码器工厂)
├── video_decoder_factory.h/cpp    (自定义视频解码器工厂)
├── external_audio_source_impl.h/cpp   (外部音频源实现)
├── external_audio_device_module.h/cpp (自定义音频设备模块)
├── external_audio_capture_module.h/cpp (自定义音频采集模块) [已废弃]
├── external_video_source_impl.h/cpp    (外部视频源实现)
└── apple/
    ├── video_toolbox_encoder.h/cpp     (Apple VT 硬编码)
    ├── video_toolbox_decoder.h/cpp     (Apple VT 硬解码)
    ├── video_toolbox_nalu_rewriter.h/cpp (H.264 NALU 格式转换)
    ├── video_capture/device_info.h     (iOS/macOS 摄像头设备信息)
    ├── video_capture/video_capture.h   (iOS/macOS 摄像头采集)
    ├── AVCaptureSession+DevicePosition.h (摄像头前后切换)
    ├── UIDevice+H264Profile.h          (iOS H.264 Profile 查询)
    └── macos_supported.h               (macOS 平台能力检测)
```

---

## 1. VideoEncoderFactory — 自定义编码器工厂

文件：`video_encoder_factory.h/cpp`

### `CreateVideoEncoderFactory()` → `unique_ptr<VideoEncoderFactory>`
- **用途**: 创建 BeeNet 自定义的视频编码器工厂，支持编解码器偏好选择
- **核心逻辑**:
  1. 创建内部的 `VideoEncoderFactory` 实现
  2. 按优先级支持 `H.264 → VP9 → VP8`
  3. **硬件加速优先级**: VideoToolbox (Apple) > MediaFoundation (Windows) > 软件 OpenH264
  4. 若 `WEBRTC_USE_H264` 未定义 → 跳过 H.264，仅支持 VP8/VP9
- **与 libwebrtc 的关系**: 替换默认的 `CreateBuiltinVideoEncoderFactory()`，注入到 `PeerConnectionFactory` 创建过程中

---

## 2. VideoDecoderFactory — 自定义解码器工厂

文件：`video_decoder_factory.h/cpp`

### `CreateVideoDecoderFactory()` → `unique_ptr<VideoDecoderFactory>`
- **用途**: 创建 BeeNet 自定义的视频解码器工厂
- **核心逻辑**:
  1. 支持 `H.264 → VP9 → VP8` 解码
  2. **硬件加速**: VideoToolbox (Apple 硬解) 优先
  3. 回退到软件实现
- **与 encoder 对称**: 同样在 `PeerConnectionFactory` 创建时注入

---

## 3. ExternalAudioSourceImpl — 外部音频源实现

文件：`external_audio_source_impl.h/cpp`

### 类层次（当前版本）

```
ExternalAudioSourceImpl : AudioInput (自定义基类)
  - 实现 AudioInput 接口（Recording/AttachAudioBuffer 等）
  - 被 ExternalAudioDeviceModule 管理
```

### `Create(id)` → ExternalAudioSourceImpl*
- **用途**: 创建或获取音频源（通过全局 map 按 ID 缓存）
- **核心逻辑**: 若已存在同 ID 源 → 返回现有；否则创建新的

### `DeliverRecordedData(audio_data, sample_rate, channels, samples)`
- **用途**: 外部推送 16-bit PCM 音频数据
- **调用流程**:
  1. `FineAudioBuffer` 将输入数据重采样为 10ms 帧（WebRTC 标准帧长）
  2. 通过 `audio_device_buffer_->SetRecordedBuffer(...)` 注入
  3. `audio_device_buffer_->DeliverRecordedData()` 通知 WebRTC
- **FineAudioBuffer**: WebRTC 内置工具，负责将任意长度的 PCM 数据切分为 10ms 帧并放入环形缓冲区

### `StartRecording()` / `StopRecording()`
- **用途**: 控制录制状态
- **核心逻辑**: 设置 `recording_` 标志 + 构造/释放 `FineAudioBuffer`

---

## 4. ExternalAudioDeviceModule — 自定义音频设备模块

文件：`external_audio_device_module.h/cpp`

### 设计思路

将 WebRTC 的 `AudioDeviceModule`（ADM）拆分为两部分：

```
AudioInput  (抽象接口，含 Init/StartRecording/StopRecording/AttachAudioBuffer 等)
AudioOutput (抽象接口，含 Init/StartPlayout/StopPlayout/AttachAudioBuffer 等)
```

### `CreateExternalAudioDeviceModule(task_queue_factory, input, output)` → ADM*
- **用途**: 组合 AudioInput 和 AudioOutput 创建一个完整的 AudioDeviceModule
- **核心逻辑**: 内部实现转发所有 ADM 方法到对应的 input/output 组件
- **WebRTC 分支兼容**:
  - `WEBRTC_BRANCH_6367`: 传 `TaskQueueFactory*`
  - 新版本: 传 `const webrtc::Environment&`

### 使用场景
- `ExternalAudioSourceImpl` 作为 `AudioInput` 提供外部 PCM 注入
- 若需要音频播放，可提供自定义 `AudioOutput` 实现

---

## 5. ExternalAudioCaptureModule — [已废弃]

文件：`external_audio_capture_module.h/cpp`

整个文件被 `#if 0` 包围。这是一个更早期的实现，直接继承完整的 `webrtc::AudioDeviceModule` 接口。

- **废弃原因**: 接口过于复杂（90+ 个方法），多数方法只需返回默认值
- **替代方案**: `AudioInput` / `AudioOutput` 分离设计 + `CreateExternalAudioDeviceModule` 组合器

---

## 6. ExternalVideoSourceImpl — 外部视频源实现

文件：`external_video_source_impl.h/cpp`

### 类层次

```
ExternalVideoSourceImpl :
  Notifier<VideoTrackSourceInterface>   // WebRTC 视频源接口
  VideoSinkInterface<VideoFrame>        // 接收 IncomingFrame 回调
```

### `Get(id)` → ExternalVideoSourceImpl*
- **用途**: 按 ID 获取视频源（全局 map 缓存）

### `OnFrame(VideoFrame)` — 视频帧接收
- **用途**: 被 `ExternalVideoSource::IncomingFrame` 调用
- **调用流程**:
  1. 检查 `video_adapter_.AdaptFrameResolution(...)` — 自适应分辨率
  2. 若需要缩放 → `CropAndScaleFrom` 裁剪缩放
  3. 通过 `broadcaster_.OnFrame(frame)` 广播给 WebRTC

### Sink 管理
- `AddOrUpdateSink` / `RemoveSink` — 委托给 `broadcaster_`
- `video_adapter_` 根据 `broadcaster_.wants()` 动态调整编码参数

### 与 CameraVideoTrackSource 的相似性
两者架构几乎相同（broadcaster + video_adapter + scaled_buffer），区别在于：
- CameraVideoTrackSource 自己采集（VideoCaptureModule）
- ExternalVideoSourceImpl 接收外部推送

---

## 7. Apple VideoToolbox 硬编解码

目录：`extensions/webrtc/apple/`

### VideoToolboxEncoder — H.264 硬编码
- **用途**: 使用 Apple VideoToolbox 进行 H.264 硬件编码
- **核心功能**: `CreateVideoToolboxEncoder(cricket::VideoCodec)` 工厂方法
- **特性**: 支持 CBP (Constrained Baseline Profile)、Main、High Profile
- **码率控制**: 支持 CBR/VBR 模式
- **重配**: 运行时码率/分辨率调整

### VideoToolboxDecoder — H.264 硬解码
- **用途**: 使用 Apple VideoToolbox 进行 H.264 硬件解码
- **核心功能**: `CreateVideoToolboxDecoder()` 工厂方法
- **特性**: 异步解码 + 重排序（处理 B 帧）

### VideoToolboxNaluRewriter — NALU 格式转换
- **用途**: Annex B ↔ AVCC 格式互转
- **核心逻辑**:
  - Annex B: `0x00000001 NALU 0x00000001 NALU ...`（起始码分隔）
  - AVCC: `[length] NALU [length] NALU ...`（长度前缀）
  - VideoToolbox 编码器输出 AVCC，解码器输入 Annex B

---

## 8. Apple 视频采集

目录：`extensions/webrtc/apple/video_capture/`

### DeviceInfoIos / DeviceInfoObjc
- **用途**: iOS/macOS 摄像头设备枚举（替代 libwebrtc 默认实现）
- **核心功能**: `NumberOfDevices()` / `GetDeviceName()` / `GetDeviceUniqueName()`

### VideoCaptureIos / VideoCaptureObjc
- **用途**: iOS/macOS 摄像头采集实现
- **核心功能**: `Create(unique_name)` 工厂 → `StartCapture(capability)` → 回调 `OnFrame`

### AVCaptureSession+DevicePosition
- **用途**: Category 扩展,快速切换前后摄像头 (`devicePosition` 属性)

### UIDevice+H264Profile
- **用途**: 查询设备 H.264 硬件编码能力（Profile + Level）

### macos_supported.h
- **用途**: macOS 版本检测 (`@available` / `__builtin_available`)

---

## 模块8 设计要点

| 要点 | 说明 |
|---|---|
| **编解码器选择** | 完全替换 libwebrtc 默认编解码工厂，支持 H.264/VP8/VP9 + 平台硬编解码 |
| **ADM 拆分设计** | `AudioInput`/`AudioOutput` 分离 → `CreateExternalAudioDeviceModule` 组合，比直接继承 ADM 更简洁 |
| **全局 ID 缓存** | ExternalAudioSourceImpl 和 ExternalVideoSourceImpl 均通过全局 map 按 ID 缓存和查找 |
| **FineAudioBuffer** | 利用 WebRTC 内置工具做 10ms 帧切分和采样率转换 |
| **NALU 格式桥接** | Annex B ↔ AVCC 转换适配 Apple VideoToolbox 与 WebRTC 内部格式差异 |
| **平台条件编译** | Apple 相关代码通过 `WEBRTC_MAC` / `WEBRTC_IOS` 宏隔离，桌面采集通过 `!WEBRTC_IOS` 隔离 |
