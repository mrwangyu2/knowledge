# BeeNet 模块7：媒体采集

> 精读日期：2026-06-04  
> 覆盖文件：`camera_video.h/cpp`, `desktop_video.h/cpp`

---

## 文件依赖关系

```
camera_video.cpp  ──► webrtc_defines.h, camera_video.h (libwebrtc VideoCaptureModule)
desktop_video.cpp ──► webrtc_defines.h, desktop_video.h (libwebrtc DesktopCapturer, libyuv)
```

---

## 1. CameraVideoTrackSource — 摄像头采集

文件：`camera_video.h/cpp`  
继承：`webrtc::VideoTrackSource`（非远程源）

### `Create(uint32_t id)` → CameraVideoTrackSource*
- **用途**: 按设备序号创建摄像头源
- **调用流程**:
  1. 创建 `VideoCaptureModule::DeviceInfo` → 平台适配
     - macOS/iOS: `DeviceInfoIos` 自定义实现
     - 其他: `VideoCaptureFactory::CreateDeviceInfo()`
  2. `info->NumberOfDevices()` 检查摄像头数量
  3. id 超出范围 → 默认使用 0
  4. `info->GetDeviceName(id, ...)` 获取设备名称和唯一 ID
  5. 返回 `CameraVideoTrackSource(unique_name)`

### `Create(fuzzy_name)` → CameraVideoTrackSource*
- **用途**: 模糊匹配设备名称（`strcasestr` 查找）
- **调用流程**: 遍历设备列表，通过大小写不敏感匹配找到目标设备

### `Init(width, height, fps)` → bool
- **用途**: 启动摄像头采集
- **调用流程**:
  1. 创建 `VideoCaptureModule`：
     - macOS/iOS: `VideoCaptureIos::Create(unique_name)`
     - 其他: `VideoCaptureFactory::Create(unique_name)`
  2. 检查设备是否已被占用（`CaptureStarted()`）
  3. 注册回调 `RegisterCaptureDataCallback(this)`
  4. 设置采集参数：`VideoCaptureCapability{width, height, fps, kI420}`
  5. `vcm_->StartCapture(capability)`
  6. 验证 `CaptureStarted()` 确认启动成功
- **采集格式强制 I420**: `capability.videoType = kI420`

### `OnFrame(frame)` — 摄像头帧回调
- **用途**: `VideoCaptureModule` 的数据回调
- **调用流程**:
  1. 检查 `broadcaster_.frame_wanted()`（是否有消费者需要帧）
  2. `video_adapter_.AdaptFrameResolution(...)` — 自适应分辨率/帧率
  3. 若需要缩放：
     - 创建或复用 `scaled_buffer_`（I420Buffer）
     - `CropAndScaleFrom` 裁剪并缩放
     - 传递 `update_rect`（脏矩形，用于编码优化）
  4. 通过 `broadcaster_.OnFrame(frame)` 广播给所有 sink

### `AddOrUpdateSink / RemoveSink` — Sink 管理
- **用途**: WebRTC track 的 sink 注册/注销
- **核心逻辑**: 委托给 `broadcaster_` + 同步 `video_adapter_` 的 sink wants

### 析构
- **调用流程**: `vcm_->StopCapture()` → `DeRegisterCaptureDataCallback()` → 释放 VCM
- **调试日志**: 记录设备名称

---

## 2. DesktopVideoTrackSource — 桌面采集

文件：`desktop_video.h/cpp`  
条件编译：`!defined(WEBRTC_IOS)`（iOS 无桌面采集）  
继承：`webrtc::VideoTrackSource` + `webrtc::DesktopCapturer::Callback`

### `Create()` → DesktopVideoTrackSource*
- **用途**: 创建桌面采集源（极简单例化）

### `SetFrameRate(fps)` — 设置帧率
- **用途**: 在 `Init()` 之前设置期望帧率

### `Init(with_cursor)` → bool
- **用途**: 启动桌面采集
- **调用流程**:
  1. 创建独立的 `capture_thread_`（WebRTC 线程）避免阻塞主线程
  2. 在线程中执行初始化：
     - 创建 `DesktopCaptureOptions`
       - Linux: `set_allow_pipewire(true)`（Wayland 支持）
       - Windows: `set_allow_wgc_screen_capturer(true)`（Windows Graphics Capture）
       - macOS: `set_allow_iosurface(true)`（零拷贝）
     - 创建采集器：
       - `with_cursor=true` → `DesktopAndCursorComposer`（含鼠标光标）
       - `with_cursor=false` → `DesktopCapturer::CreateScreenCapturer()`
     - `capturer_->Start(this)` 启动
     - 调用 `CaptureFrame()` 开始第一次采集

### `CaptureFrame()` — 定时采集循环
- **调用流程**: `capturer_->CaptureFrame()` → `PostDelayedTask(CaptureFrame, 1000/fps)` 递归调度
- **macOS 特殊处理**: 每次采集后 `CFRunLoopRunInMode(kCFRunLoopDefaultMode, 0, true)` — 驱动屏幕采集的回调事件

### `OnCaptureResult(result, frame)` — 桌面帧回调
- **用途**: `DesktopCapturer` 的数据回调
- **调用流程**:
  1. 检查 result == SUCCESS
  2. `AdaptFrame(...)` 自适应分辨率/帧率（获取 crop 区域和适配尺寸）
  3. **ARGB → I420 转换**: `libyuv::ConvertToI420(frame->data(), 0, y, u, v, crop_x, crop_y, width, height, crop_w, crop_h, libyuv::kRotate0, libyuv::FOURCC_ARGB)`
  4. **HiDPI 缩放**: 若 `scale_factor > 0 && != 1`（如 Retina 屏 2x）
     - `libyuv::I420Scale(...)` 使用双线性滤波缩放到目标分辨率
  5. `OnFrame(VideoFrame(i420_buffer, 0, 0, kVideoRotation_0))` 广播

### 格式转换技术栈

```
DesktopFrame (ARGB/ABGR, 平台相关)
  │  libyuv::ConvertToI420
  ▼
I420 (原始分辨率)
  │  libyuv::I420Scale (kFilterBilinear) [仅 HiDPI]
  ▼
I420 (适配分辨率) → broadcaster_.OnFrame()
```

---

## 模块7 设计要点

| 要点 | 说明 |
|---|---|
| **平台适配** | 摄像头（iOS 用自定义 DeviceInfoIos）、桌面（Windows WGC / macOS IOSurface / Linux PipeWire） |
| **自适应分辨率** | `video_adapter_` / `AdaptFrame` 根据 sink wants 动态调整分辨率和帧率 |
| **脏矩形优化** | 传递 `update_rect` 给编码器，仅编码变化区域（移动端省电） |
| **ARGB→I420** | 桌面帧统一使用 libyuv 转换为 I420 再进行缩放/广播 |
| **HiDPI 支持** | 检测 `scale_factor`，对高 DPI 屏幕做像素级缩放后再输出 |
| **定时采集** | 桌面采集用 `PostDelayedTask` 实现固定帧率循环 |
