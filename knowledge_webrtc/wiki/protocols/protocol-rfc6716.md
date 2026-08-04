---
title: RFC 6716 - Opus 音频编解码器
type: protocol
tags: [webrtc, audio, opus, codec, rfc]
sources: [rfc6716]
created: 2026-05-22
updated: 2026-05-22
---

# RFC 6716: Definition of the Opus Audio Codec

> **一句话**：Opus 是 IETF 标准化的交互式音频编解码器，覆盖 6~510 kbit/s，融合 SILK (语音) 和 CELT (音乐) 两种编码技术，是 WebRTC 的强制音频编解码器。

---

## 一句话理解

> **Opus = SILK (语音层) + CELT (音乐层)，一个编解码器同时搞定语音和音乐，码率从 6kbps 窄带语音到 510kbps 全频带立体声音乐。**

---

## Opus 解决了什么？

在 Opus 之前，语音和音乐需要不同的编解码器：

```
  语音通话 → SILK, G.729, AMR-WB (擅长人声，低码率)
  音乐传输 → AAC, Vorbis, MP3 (擅长音乐，高码率)

  问题: 一个视频会议里既有语音也有音乐分享怎么办？
       → 要么用语音编解码器（音乐听起来像垃圾）
       → 要么用音乐编解码器（浪费带宽在静音上）
```

**Opus 的方案：内部有两条编码路径，自动切换。**

---

## 架构：混合编解码器

```
┌─────────────────────────────────────────────────────────────┐
│                      Opus 编码器                              │
│                                                             │
│  输入音频                                                    │
│     │                                                       │
│     ├── LP 层 (SILK) ────────────┐                          │
│     │   基于线性预测               │                          │
│     │   擅长: 语音 (NB/MB/WB)      ├── 混合模式可同时启用       │
│     │   帧长: 10~60ms             │     (SWB/FB 下两者叠加)    │
│     │   延迟: +5ms                │                          │
│     │                             │                          │
│     └── MDCT 层 (CELT) ─────────┘                          │
│         基于改进离散余弦变换                                   │
│         擅长: 音乐 (NB/WB/SWB/FB)                             │
│         帧长: 2.5~20ms                                       │
│         延迟: +2.5ms                                          │
│                                                             │
│  混合模式: LP 编码 0~8kHz 低频 + MDCT 编码 8kHz+ 高频           │
│  两者使用同一个熵编码器，无缝拼接，零填充比特                      │
└─────────────────────────────────────────────────────────────┘
```

## 三种工作模式

| 模式 | 使用层 | 音频带宽 | 适用场景 |
|------|--------|---------|---------|
| **纯 SILK** | LP 层 | NB / MB / WB | 纯语音通话 |
| **纯 CELT** | MDCT 层 | NB / WB / SWB / FB | 纯音乐/非语音内容 |
| **Hybrid 混合** | LP + MDCT | SWB / FB | 宽频/全频带语音+音乐 |

> **自动切换**：编码器根据信号特征在三种模式间无缝切换，解码器无需重新协商。

---

## 音频带宽

| 缩写 | 名称 | 音频带宽 | 有效采样率 |
|------|------|:---:|:---:|
| NB | Narrowband 窄带 | 4 kHz | 8 kHz |
| MB | Medium-band 中带 | 6 kHz | 12 kHz |
| WB | Wideband 宽带 | 8 kHz | 16 kHz |
| SWB | Super-wideband 超宽带 | 12 kHz | 24 kHz |
| FB | Fullband 全频带 | 20 kHz | 48 kHz |

---

## 控制参数

| 参数 | 说明 | 范围/典型值 |
|------|------|-----------|
| **Bitrate 码率** | 编码后码率 | 6~510 kbit/s；NB 语音 8-12k，FB 立体声音乐 64-128k |
| **Channels 声道数** | 单声道/立体声 | 1 or 2 |
| **Audio Bandwidth 音频带宽** | 编码频率范围 | NB / MB / WB / SWB / FB |
| **Frame Duration 帧长** | 每帧时长 | 2.5 / 5 / 10 / 20 / 40 / 60 ms |
| **Complexity 复杂度** | 编码器计算量 | 0~10，越高品质越好但越慢 |
| **Packet Loss Resilience** | 丢包容忍度 | 影响编码决策，提前考虑丢包场景 |
| **FEC** | 前向纠错 | 在包内嵌入上一帧低码率副本 |
| **CBR / VBR** | 恒定/可变码率 | SILK 天生 VBR，CELT 天生 CBR，Hybrid 可做 CBR |
| **DTX** | 不连续传输 | 静音时不编码，节省带宽 |

---

## 包格式：TOC 字节

每个 Opus 包的第一个字节是 **TOC (Table of Contents)**：

```
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
| config |s| c  |
+-+-+-+-+-+-+-+-+

  config (5 bits): 编码模式索引 (0~31)，决定 SILK/CELT/Hybrid + 带宽
  s (1 bit):      立体声标志 (0=mono, 1=stereo)
  c (2 bits):     帧数编码 (0~3)，一个包里可以打包多帧
```

**帧打包 (Frame Packing)**：一个 RTP 包里可以包含多个 Opus 帧（最大 120ms 音频），这是 Opus 降低 RTP 开销的机制。

| Code | 含义 |
|:---:|------|
| 0 | 包内 1 帧 |
| 1 | 包内 2 帧，等长 |
| 2 | 包内 2 帧，不等长 |
| 3 | 包内 N 帧，帧数由额外字节指示 |

---

## 编解码器内部要点

### 解码器 (Section 4)

解码器是**规范性**（normative）的——附录 A 的 C 参考实现定义了解码行为：

```
Opus 包 → TOC 解析 → 配置确定
    │
    ├── SILK 帧 → SILK 解码器 → WB 以下低频
    │                                 │
    ├── CELT 帧 → CELT 解码器 → 全频带  │
    │                                 │
    └── Hybrid → 两个解码器并行 ───────┘
                                    │
                                相加 → 输出 PCM
```

### 编码器 (Section 5)

编码器**不是规范性**的——开发者可以自由设计编码器，只要输出符合解码器规范即可。这给了编码器很大的优化空间。

### 一致性 (Section 6)

通过比较解码器输出与参考实现的输出（逐比特相等）来验证一致性。

---

## 在 WebRTC 中的角色

Opus 是 **WebRTC 强制实现（MTI）的音频编解码器**：

```
WebRTC 音频编解码器:
  必需: Opus (RFC 6716)
  可选: G.711 (PCMU/PCMA)

Opus 的优势在 WebRTC 中:
  ✅ 一个编解码器覆盖所有场景（语音/音乐/混合）
  ✅ 支持 FEC 抗丢包
  ✅ 支持 DTX 省带宽
  ✅ 支持立体声（音乐教学、屏幕共享带音频）
  ✅ 动态码率调整（拥塞控制友好）
  ✅ 低延迟 (最小 5ms 算法延迟)
```

---

## 与 RFC 3550 (RTP) 的关系

Opus 作为 RTP 负载，定义在 [RFC 7587](https://www.rfc-editor.org/rfc/rfc7587) 中。关键点：

- Opus 包可以包含多帧（利用 TOC 的帧打包）
- RTP 时间戳对应第一帧的采样时刻
- 支持 FEC 和 PLC (Packet Loss Concealment)
- 推荐使用 20ms 帧长作为默认值

---

## 核心要点

1. **混合架构**：SILK (LP) 处理语音低频，CELT (MDCT) 处理音乐和语音高频，Hybrid 模式下两者协同
2. **超宽码率范围**：6 ~ 510 kbit/s，从 NB 窄带到 FB 全频带
3. **解码器是标准**：附录 A C 代码是规范性定义；编码器可以自由优化
4. **WebRTC MTI**：所有 WebRTC 实现必须支持 Opus
5. **极低延迟**：算法延迟最低 5ms，适合实时交互

---

## 相关文档

- [[wiki/protocols/protocol-rfc3550]] - RTP 基础协议
- [[raw/rfcs/rfc6716]] - RFC 6716 英文原版
- [[raw/rfcs/rfc6716_完整中英对照]] - RFC 6716 中英对照
- [RFC 7587](https://www.rfc-editor.org/rfc/rfc7587) - RTP Payload Format for Opus
