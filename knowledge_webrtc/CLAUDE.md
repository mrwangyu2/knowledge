# WebRTC 知识库 - CLAUDE.md

这是 WebRTC 知识库的 Schema 规范文档。

## 架构

本知识库采用 Karpathy LLM Wiki 三层架构：

```
raw/           ← 原始资料 (不可修改)
wiki/          ← LLM 生成的 wiki 页面
Schema 文件    ← 本文档 + index.md + log.md
```

## 三层定义

### 1. Raw Sources (raw/)
- **位置**: `raw/`
- **规则**: 原始文档只读不修改
- **子目录**:
  - `rfcs/` - RFC 标准文档
  - `papers/` - 学术论文
  - `articles/` - 网络文章
  - `assets/` - 原始图片/附件

### 2. Wiki (wiki/)
- **位置**: `wiki/`
- **规则**: LLM 生成和维护的内容
- **子目录**:
  - `concepts/` - 核心概念页 (如 ICE, STUN, TURN, SDP)
  - `protocols/` - 协议总结页
  - `comparisons/` - 对比分析页
  - `tutorials/` - 教程/指南

### 3. Schema 文件
- `CLAUDE.md` - 本文档，定义规范
- `index.md` - 内容目录
- `log.md` - 操作时间线

## 内容规范

### 页面命名
- 使用英文或中文描述性名称
- 概念页: `concept-ice.md`, `概念-stun.md`
- 协议页: `protocol-rfc8445.md`
- 对比页: `compare-ice-stun.md`

### Frontmatter 格式
```yaml
---
title: ICE (Interactive Connectivity Establishment)
type: concept
tags: [webrtc, ice, protocol]
sources: [rfc8445, rfc5245]
created: 2026-04-21
updated: 2026-04-21
---

### Wiki 链接格式
- 概念链接: [[wiki/concepts/概念-ice]]
- 协议链接: [[wiki/protocols/protocol-rfc8445]]
- Wiki 内部链接使用相对路径

### Raw 文档链接
- RFC 链接到原始 URL 或本地文件
- 示例: [RFC 8445](https://tools.ietf.org/html/rfc8445)

## 操作流程

### Ingest (摄入新资料)
1. 将原始文档放入 `raw/` 对应子目录
2. 运行 `/log` 记录: `## [2026-04-21] ingest | Article Title`
3. LLM 阅读原始文档，生成 wiki 页面
4. 更新 `index.md` 添加新页面条目
5. 更新相关概念页的交叉引用

### Query (查询)
1. 阅读 `index.md` 找到相关页面
2. 读取目标 wiki 页面
3. 必要时更新页面或创建新分析

### Lint (检查)
- 定期检查孤儿页面 (无入站链接)
- 检查过时的信息
- 补充缺失的交叉引用

## WebRTC 核心主题

- ICE (Interactive Connectivity Establishment)
- STUN (Session Traversal Utilities for NAT)
- TURN (Traversal Using Relays around NAT)
- SDP (Session Description Protocol)
- RTP/RTCP (Real-time Transport Protocol)
- DTLS/SRTP (Datagram Transport Layer Security / Secure RTP)
- SDP Offer/Answer Model
- Trickle ICE
- NAT Traversal Techniques

## 快捷命令

- `/log` - 追加日志条目
- `/index` - 更新目录
- `/lint` - 健康检查

---

## 网络配置 (RFC 下载)

### HTTP 代理
```
http://127.0.0.1:7890
```
使用代理下载命令示例：
```bash
curl -x http://127.0.0.1:7890 -sL "https://www.rfc-editor.org/rfc/rfc4566.txt" -o rfc4566.md
```

### 国内镜像 (备选)
```
https://www.rfc-editor.org/rfc/
```
- 直接访问可能受限
- 优先使用代理访问 rfc-editor.org

### 常用 RFC URL 格式
- 英文原文: `https://www.rfc-editor.org/rfc/rfc{NUMBER}.txt`
- HTML 版本: `https://www.rfc-editor.org/rfc/rfc{NUMBER}.html`
