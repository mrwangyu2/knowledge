# BeeNet 模块6：外部媒体注入

> 精读日期：2026-06-04  
> 覆盖文件：`external_audio.h`, `external_video.h`, `external_media_source_pusher.cpp`

---

## 文件依赖关系

```
external_audio.h ── (独立，纯头文件 + 实现)
external_video.h ── (独立，纯头文件 + 实现)
external_media_source_pusher.cpp ──► (独立辅助)
```

这些文件定义用户可继承的外部媒体源基类，用于在不使用本地摄像头/麦克风时注入自定义音视频数据。

---

## 1. ExternalAudioSource — 外部音频注入

文件：`external_audio.h`（纯头文件，inline 实现）

### 字节序自动检测

通过预处理器自动判断目标平台的字节序：
- Windows → 固定小端
- `__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__` → GCC/Clang 小端
- `__GLIBC__` → 检查 `<endian.h>`
- 备选：`_LITTLE_ENDIAN` / `_BIG_ENDIAN` 宏

### `AudioPCMFormat` 枚举

| 标志 | 值 | 含义 |
|---|---|---|
| `kFloat` | 0x01 | 浮点采样（非整数 PCM） |
| `kNonInterleaved` | 0x02 | 非交错布局（按通道分离存储） |
| `kBigEndian` | 0x04 | 大端字节序 |
| `kDefault` | 小端=0 / 大端=kBigEndian | 平台默认格式 |

### `ConvertToDefaultPCM(in, bits, channels, samples, format, out)` → bool
- **签名**: `bool ConvertToDefaultPCM(const uint8_t* const* in, int nBitsPerSample, size_t nChannels, size_t nSamples, int format, void* out)`
- **用途**: 将任意 PCM 格式转换为 16-bit interleaved 默认格式
- **核心逻辑**:
  - `format & kNonInterleaved` → 输入 `in` 是二维数组（`in[channel][data]`）
  - 否则 → 仅 `in[0]` 有效（交错布局）
  - 目标：16-bit 整数 + 平台默认字节序 + 交错布局

### `ExternalAudioSource()` / `~ExternalAudioSource()`
- **构造函数**: `is_registered_(false)`, PcmBuffer 分配
- **析构函数**: 虚析构（允许子类安全删除）

### `RegisterExternalSource()` → string
- **用途**: 注册音频源到 WebRTC，返回唯一 source ID
- **核心逻辑**: 生成 UUID → 存储到全局注册表 → 标记 `is_registered_ = true`

### `id()` → string
- **用途**: 返回已注册的 source ID
- **核心逻辑**: 从全局注册表查找

### `IncomingData(data, bits, rate, channels, samples, format)`
- **用途**: 子类调用此方法推送 PCM 音频数据
- **调用流程**: `ConvertToDefaultPCM` → 查全局注册表 → 转发到 `ExternalAudioDeviceModule`

---

## 2. ExternalVideoSource — 外部视频注入

文件：`external_video.h`（纯头文件，inline 实现）

### `VideoType` 枚举 — 支持的视频格式

| 格式 | 说明 |
|---|---|
| `kI420` | YUV 4:2:0 平面格式 |
| `kIYUV` | YUV 4:2:0 (同 I420，顺序不同) |
| `kRGB24` | RGB 8:8:8 |
| `kBGR24` | BGR 8:8:8 |
| `kARGB` | ARGB 8:8:8:8 |
| `kABGR` | ABGR 8:8:8:8 |
| `kRGB565` | RGB 5:6:5 |
| `kYUY2` | YUYV 4:2:2 |
| `kYV12` | YVU 4:2:0 |
| `kUYVY` | UYVY 4:2:2 |
| `kMJPEG` | Motion JPEG |
| `kBGRA` | BGRA 8:8:8:8 |
| `kNV12` | Y + UV interleaved 4:2:0 |

共 14 种格式，覆盖大多数视频采集场景。

### `RegisterExternalSource()` → string
- **用途**: 注册视频源到 WebRTC，返回唯一 source ID
- **核心逻辑**: 同音频——生成 UUID → 全局注册表

### `IncomingFrame(data, length, width, height, type)`
- **用途**: 子类调用此方法推送视频帧
- **调用流程**: 查全局注册表 → 根据 VideoType 转换为 I420 → 转发到 `ExternalVideoSourceImpl`

---

## 3. external_media_source_pusher.cpp — 媒体源推送辅助

文件为空，可能是预留的扩展接口，或实现已合并到其他文件中。

---

## 模块6 设计要点

| 要点 | 说明 |
|---|---|
| **纯头文件设计** | Audio/Video 源均为纯头文件，方便用户直接 include 继承 |
| **14 种视频格式** | 覆盖 I420/NV12/RGB24/MJPEG 等常见原始和压缩格式 |
| **自动字节序检测** | 编译期检测平台字节序，无需用户关心 |
| **交错/非交错音频** | 支持通道分离存储的音频布局 |
| **全局注册表** | 通过 source ID 将外部源与 WebRTC track 绑定 |
