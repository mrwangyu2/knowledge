# libnice 源码分析

> 从浅到深、逐函数精读 libnice ICE (RFC 8445) 实现。

## 推荐阅读顺序

### 入门
1. [01-entry-examples.md](01-entry-examples.md) — 从例子开始：simple → sdp → threaded

### 模块分析（自底向上）
2. [02-module-random.md](02-module-random.md) — random/ 随机数模块 (~200 行)
3. [03-module-stun.md](03-module-stun.md) — stun/ STUN 协议实现 (~7200 行)
4. [04-module-socket.md](04-module-socket.md) — socket/ 套接字抽象层 (~7300 行)
5. [05-module-agent.md](05-module-agent.md) — agent/ ICE 代理核心 (~27000 行)
6. [06-module-gst.md](06-module-gst.md) — gst/ GStreamer 插件 (~1350 行)

### 核心流程（跨模块追踪）
7. [07-flow-candidate-gathering.md](07-flow-candidate-gathering.md) — 候选收集（host → srflx → relay）
8. [08-flow-connectivity-checks.md](08-flow-connectivity-checks.md) — 连接检查（候选对状态机）
9. [09-flow-nomination-data.md](09-flow-nomination-data.md) — 提名与数据收发
10. [10-flow-turn-allocation.md](10-flow-turn-allocation.md) — TURN 分配与中继

## 模块总览

| 模块 | 文件数 | 代码量 | 职责 |
|------|--------|--------|------|
| random/ | 4 | ~200 行 | 随机数生成 (GRand 封装) |
| stun/ | 14 | ~7200 行 | STUN 协议消息/事务/usages |
| socket/ | 18 | ~7300 行 | 多传输类型 socket 抽象 |
| agent/ | 24 | ~27000 行 | ICE 代理核心逻辑 |
| gst/ | 4 | ~1350 行 | GStreamer 插件 |

## 关键数据结构速查

| 结构 | 定义位置 | 用途 |
|------|---------|------|
| StunMessage | stun/stunmessage.h | STUN 消息结构 |
| StunAgent | stun/stunagent.h | STUN 事务代理 |
| NiceSocket | socket/socket-priv.h | Socket 虚表 |
| NiceAgent | agent/agent-priv.h | ICE 代理私有结构 |
| Component | agent/component.h | 组件状态 |
| Stream | agent/stream.h | 媒体流 |
| NiceCandidate | agent/candidate.h | ICE 候选 |
| CandidateCheckPair | agent/conncheck.c | 候选对 |
| ConnCheck | agent/conncheck.h | 连接检查状态 |
| PseudoTcpSocketPrivate | agent/pseudotcp.c | 伪 TCP 状态 |

## 架构层次

```
应用层 (examples/)
  └── agent/ (ICE 代理)
        ├── conncheck (连接检查)
        ├── discovery (候选发现)
        ├── component (组件)
        ├── stream (流)
        ├── candidate (候选)
        └── pseudotcp (可靠传输)
              └── socket/ (传输抽象)
                    ├── udp-bsd (原生 UDP)
                    ├── udp-turn (TURN 中继)
                    ├── udp-turn-over-tcp (TURN over TCP)
                    ├── tcp-bsd/active/passive (TCP)
                    ├── socks5/http (代理)
                    └── pseudossl (加密)
                          └── stun/ (STUN 协议)
                                ├── stunmessage (消息格式)
                                ├── stunagent (事务管理)
                                ├── usages/bind (Binding)
                                ├── usages/ice (ICE 扩展)
                                ├── usages/turn (TURN 扩展)
                                ├── usages/timer (重传)
                                └── crypto (HMAC/CRC32)
                                      └── random/ (随机数)
```
