# WebRTC 知识库 - Log

> 操作时间线，按时间倒序排列。使用 `grep "^## \[" log.md | head -20` 查看最近 20 条。

---

## [2026-05-12] tutorial | 创建 UDP 打洞流程详解

- 创建文件: `wiki/tutorials/tutorial-udp-hole-punching.md`
- 内容: UDP 打洞的完整深度讲解
  - 问题定义：NAT"只出不进"导致两端互不可达
  - 核心洞察：发包在自己 NAT 开洞，等对方从洞进来
  - 三步流程：报到→同时打洞→P2P 直连
  - NAT 适配：Full Cone/Restricted/Port-Restricted → 打洞可行，Symmetric → 失败
  - 四个"同时"问题：包时序竞争、映射端口可预测性、超时、中介信息时效
  - 与 ICE 关系：ICE = 标准化的打洞框架
- 更新 `index.md`，Wiki 页面数 11→12

---

## [2026-05-12] canvas | 扩展知识全景图 → 覆盖全部 15 个 Wiki 页面

- 更新文件: `wiki/protocols/protocols-relationship.canvas`
- 从 4 节点扩展到 16 节点 (全部 wiki/，排除 raw/)
- 新增 3 个分区组:
  - NAT穿透协议体系 (黄, 原有, 4 协议)
  - ICE 教程与实现详解 (绿, 新增, 4 教程: gathering → hole-punching → checks → nomination)
  - 信令与媒体传输 (青, 新增, SIP → SDP → RTP → RTP Profile)
  - 概念与对比速查 (紫, 新增, ICE/protocol/glossary + TURN vs Socket)
- 从 7 条边扩展到 27 条边
- UDP打洞教程为跨领域中心节点 (连接 NAT理论/信令/概念)
- 同时完成了 12 个文档到 tutorial-udp-hole-punching 的双向链接

---

## [2026-05-07] tasks | 创建日常任务库

- 创建文件: `wiki/daily-tasks.md`
- 整合工作目标（WebRTC RFC + libnice）、学习目标（WebRTC 第2阶段）、定时任务（3个 cronjobs）
- 更新 `index.md`，Wiki 页面数 10→11

---

## [2026-05-06] query | 阅读 RFC 3550 / RFC 3551 Wiki 页面

- 阅读文件: `wiki/protocols/protocol-rfc3550.md` (RTP/RTCP 协议分析)
- 阅读文件: `wiki/protocols/protocol-rfc3551.md` (RTP A/V Profile 负载类型映射)
- 更新 `index.md` 添加 RFC 3550/3551 Wiki 页面条目
- RFC 3550 总结: RTP=数据传输, RTCP=控制协议; SSRC=源标识, 序列号=丢包检测, 时间戳=播放同步
- RFC 3551 总结: PT 0-34 静态映射 (PCMU=0, PCMA=8, G722=9...), PT 96-127 动态分配 (Opus/VP8/H264)

---

## [2026-04-27] wiki | 扩充 STUN Message Type Flag 详解

- 更新文件: `wiki/tutorials/tutorial-ice-candidate-gathering.md`
- 新增内容: STUN Message Type 16位字段的完整位解析
  - Message Type 位字段详解: 详细说明 M11-M0 (12位方法) 和 C1-C0 (2位类别)
  - C0-C1 类别详解: 00=请求, 01=成功响应, 10=指示, 11=错误响应
  - M0-M11 方法详解: 12位方法值的计算公式，附常用 STUN/TURN 方法速查表
  - Type 字段分解示例: 0x0001, 0x0101, 0x0003 的完整二进制分解过程
  - 扩展消息类型对照表: 添加更多 TURN 方法 (ChannelBind, Send, Data 等)
  - Type 计算公式和完整生命周期流程图
- 完善数据包结构 Type 说明: Binding Request/Success, Allocate Request/Success 均添加 M11-M0 和 C1-C0 位分解
- 修正 Type 说明格式: 统一使用 12 位 M11-M0 二进制格式 (0000 0000 0001)，修正 Binding Success/Allocate Success/完整对照表中的错误值，修正 ChannelBind 方法值 (0x000B)
- 新增 Transaction ID 详解: 96位随机数的作用、匹配规则、实际抓包示例、与 Magic Cookie 对比、请求-响应流程、防重放攻击机制
- 扩充 STUN Message Type: 新增 M (Method) 和 C (Class) 字母含义详解，包括概念解释、与 HTTP 类比、生活类比、M/C 组合示例
- 新增 FINGERPRINT 属性详解: 为什么需要 FINGERPRINT、计算方法（CRC32 XOR 0x5354554E）、属性格式、与 Magic Cookie 的对比、验证流程和计算示例
- 修正 Type 说明进制格式: 数据包结构"值"列同时显示二进制和十六进制 (如 `0000 0000 0000 0001` (0x0001))，M11-M0 说明列同时显示二进制和十六进制 (如 `0000 0000 0001 (0x001)`)

---

## [2026-04-23] translate | 完成 Karpathy LLM Wiki 方法论完整中英对照翻译

- 创建文件: `karpathy的wiki方法论_完整中英对照.md`
- 标题: LLM Wiki
- 作者: Andrej Karpathy
- 翻译内容: Karpathy LLM Wiki 方法论文档完整翻译
- 翻译格式: 中英文段落对照 `#### 原文` / `#### 中文`
- 涵盖内容:
  - The core idea (核心思想)
  - Architecture (架构) - Raw sources, Wiki, Schema 三层
  - Operations (操作) - Ingest, Query, Lint
  - Indexing and logging (索引和日志) - index.md, log.md
  - Optional CLI tools (可选 CLI 工具)
  - Tips and tricks (技巧)
  - Why this works (为什么这有效)
  - Note (注意)
- 更新 `index.md` 添加新翻译文件条目

---

## [2026-04-23] translate | 完成 RFC 3261 SIP 完整中英对照翻译

- 创建/更新文件: `raw/rfcs/rfc3261_完整中英对照.md`
- 标题: SIP: Session Initiation Protocol
- 作者: J. Rosenberg, H. Schulzrinne, G. Camarillo, A. Johnston, J. Peterson, R. Sparks, M. Handley, E. Schooler
- 翻译内容: RFC 3261 完整 28 个章节 + 附录
- 翻译格式: 中英文段落对照 `#### 原文` / `#### 中文`
- 涵盖内容:
  - Section 1-6: Introduction, Overview, Terminology, Operation, Protocol Structure, Definitions
  - Section 7-12: SIP Messages, UAC/UAS Behavior, Cancel, Registration, Capabilities, Dialogs
  - Section 13-19: Initiating/Modifying/Terminating Session, Proxy Behavior, Transactions, Transport, Common Components
  - Section 20-28: Header Fields, Response Codes, HTTP Authentication, S/MIME, Examples, BNF, Security, IANA, Changes
  - Appendix A-C: Timer Values, Normative/Informative References
- 更新 `index.md` 添加新翻译文件条目

---

## [2026-04-23] translate | 完成 RFC 4566 SDP 完整中英对照翻译

- 创建文件: `raw/rfcs/rfc4566_完整中英对照.md`
- 标题: SDP: Session Description Protocol
- 作者: M. Handley, V. Jacobson, C. Perkins
- 翻译内容: RFC 4566 完整 12 个章节 + 附录
- 翻译格式: 中英文段落对照 `#### 原文` / `#### 中文`
- 涵盖内容:
  - Section 1-5: Introduction, Terminology, Examples, Requirements, SDP Specification
  - Section 6: SDP Attributes (cat, keywds, tool, ptime, rtpmap, etc.)
  - Section 7-9: Security Considerations, IANA Considerations, SDP Grammar
  - Section 10-12: Changes from RFC 2327, Acknowledgements, References
- 更新 `index.md` 添加新翻译文件条目

---

## [2026-04-23] ingest | 下载 RFC 3551 RTP Profile

- 下载文件: `raw/rfcs/rfc3551.md` (2467 行, 108KB)
- 标题: RTP Profile for Audio and Video Conferences with Minimal Control
- 作者: H. Schulzrinne, S. Casner
- 更新 `index.md` 添加新 RFC 条目

---

## [2026-04-23] ingest | 下载 RFC 4566 和 RFC 3261

- 下载文件:
  - `raw/rfcs/rfc4566.md` (2747 行, 108KB) - SDP 会话描述协议
  - `raw/rfcs/rfc3261.md` (15067 行, 648KB) - SIP 会话发起协议
- 来源: https://www.rfc-editor.org/rfc/
- 更新 `index.md` 添加新 RFC 条目

---

## [2026-04-23] wiki | 更新 WebRTC 缩写字典

- 更新文件: `wiki/concepts/concept-glossary.md`
- 新增内容:
  - EIM/EDM 缩写条目（NAT 穿透协议部分）
  - NAT 类型分类表（Full Cone/Restricted/Port-Restricted/Symmetric）
  - NAT 类型速查（EIM vs EDM P2P 友好性）
  - 更新"快速区分"部分，添加 EIM vs EDM
  - 添加 RFC 5128 相关链接

---

## [2026-04-23] wiki | 更新 RFC 5128 EIM/EDM 详解

- 更新文件: `wiki/protocols/protocol-rfc5128.md`
- 新增内容: EIM vs EDM 详细解释
  - 一句话核心区别
  - 生活化类比: 宿舍电话 vs 快递柜
  - 具体通信示例: EIM 成功 vs EDM 困难
  - 完整对比表

---

## [2026-04-23] wiki | 更新 RFC 5128 NAT 过滤规则详解

- 更新文件: `wiki/protocols/protocol-rfc5128.md`
- 新增内容: NAT 过滤规则详解
  - Full Cone: 无限制过滤
  - Restricted Cone: 仅限制 IP (手机通讯录白名单)
  - Port-Restricted: 限制 IP + 端口 (精确通讯录)
  - 具体打洞场景示例
  - 过滤规则总结对比表

---

## [2026-04-23] wiki | 创建 TURN vs Socket 对比分析页

- 创建文件: `wiki/comparisons/comparison-turn-rtp-socket.md`
- 内容: TURN 转发 RTP/RTCP 与原生 Socket 的详细对比
  - 核心问题: TURN 能否转发 RTP/RTCP (答案: 可以)
  - RFC 5766 证据: Section 2.8 RTP Support、EVEN-PORT 属性、透明转发机制
  - 详细对比表: 数据完整性、协议开销、NAT 穿透、权限控制等
  - 协议开销详解: ChannelData (4字节) vs Send Indication (36字节)
  - TURN 独特优势: 单地址多对端复用、Permission 机制、ICE 原生集成
  - 适用场景决策树
- 更新: `index.md` 添加新页面条目

---

## [2026-04-22] wiki | 创建 WebRTC 缩写字典

- 创建文件: `wiki/concepts/concept-glossary.md`
- 内容: NAT 穿透相关缩写全称对照表、快速区分、常用缩写

---

## [2026-04-22] translate | 完成 RFC 5128 完整中英对照翻译

- 创建文件: `raw/rfcs/rfc5128_完整中英对照.md`
- 标题: "State of Peer-to-Peer (P2P) Communication across Network Address Translators (NATs)"
- 作者: P. Srisuresh, B. Ford, D. Kegel
- 翻译内容: RFC 5128 完整 8 个章节
- 翻译格式: 中英文段落对照
- 涵盖内容:
  - Section 1: Introduction (简介)
  - Section 2: Terminology (术语)
  - Section 3: NAT分类与行为
  - Section 4: P2P 通信方法
  - Section 5: EIM-NAT 行为分析
  - Section 6: 端点相关映射 (EDM-NAT)
  - Section 7: UDP 保持存活
  - Section 8: 建议与结论
- 相关链接: [[wiki/protocols/protocol-rfc5389]], [[wiki/protocols/protocol-rfc5766]]

---

## [2026-04-22] wiki | 创建协议关系图谱

- 创建文件: `wiki/protocols/protocols-relationship.canvas`
- 内容: WebRTC NAT穿透协议关系图谱
  - 节点: RFC 5128, RFC 5389, RFC 5766, RFC 8445
  - 边关系: 理论基础、协议扩展、ICE 协调
  - 颜色区分: 理论(紫)、STUN(绿)、TURN(橙)、ICE(青)

---

## [2026-04-22] wiki | 创建 RFC 5128 协议分析页

- 创建文件: `wiki/protocols/protocol-rfc5128.md`
- 内容: P2P 跨 NAT 通信状态分析
  - 语义层: NAT 类型分类、P2P 通信方法、EIM/EDM 概念
  - 语法层: NAT 分类体系、打洞技术分类
  - 时序层: UDP 打洞流程、Hairpinning、存活机制
- 相关链接: [[wiki/protocols/protocol-rfc5389]], [[wiki/protocols/protocol-rfc5766]]

---

## [2026-04-22] ingest | 下载 RFC 5128

- 下载文件: `raw/rfcs/rfc5128.md`
- 标题: "State of Peer-to-Peer (P2P) Communication across Network Address Translators (NATs)"
- 作者: P. Srisuresh, B. Ford, D. Kegel
- 类型: Informational
- 内容: P2P 通信穿越 NAT 的各种方法概述
- 包含: NAT, STUN, TURN, ICE, SDP, DTLS, SRTP, RTP, RTCP 等

---

## [2026-04-22] wiki | 更新 RFC 5389 XOR 编码说明

- 更新文件: `wiki/protocols/protocol-rfc5389.md`
- 新增内容: XOR 地址编码详解
  - 为什么需要 XOR 编码（防止 NAT ALG 误改）
  - XOR 运算原理
  - MAPPED-ADDRESS vs XOR-MAPPED-ADDRESS 对比
  - TURN 中的 XOR 地址属性
- 相关链接: [[wiki/protocols/protocol-rfc5766]]

---

## [2026-04-22] wiki | 更新 RFC 5766 多对端场景

- 更新文件: `wiki/protocols/protocol-rfc5766.md`
- 新增内容: TURN 多对端通信场景
  - 群聊场景：Client 与多个 Peer 同时通信
  - 完整时序图
  - SOCKS vs TURN 对比
  - 核心原因：XOR-PEER-ADDRESS 区分收件人

---

## [2026-04-22] wiki | 更新 RFC 5766 协议分析页

- 更新文件: `wiki/protocols/protocol-rfc5766.md`
- 新增内容: TURN 与 STUN 关系通俗解释
  - STUN vs TURN 能力对照表
  - TURN 扩展 STUN 的方法、属性、概念
  - 具体例子：STUN 发现地址 vs TURN 中继数据
  - 一句话总结：指南针 vs 快递柜 vs 指挥官

---

## [2026-04-22] wiki | 创建 RFC 5766 协议分析页

- 创建文件: `wiki/protocols/protocol-rfc5766.md`
- 内容: 按语义/语法/时序三层面分析 TURN 协议
  - 语义层: TURN Client/Server/Peer、Allocation、Permission、Channel
  - 语法层: 6个新 STUN 方法、TURN 属性、ChannelData 格式
  - 时序层: 创建分配→创建权限→数据传输→刷新维护
- 相关链接: [[wiki/protocols/protocol-rfc5389]], [[wiki/protocols/protocol-rfc8445]]

---

## [2026-04-22] wiki | 更新 RFC 8445 协议分析页

- 更新文件: `wiki/protocols/protocol-rfc8445.md`
- 内容: 按语义/语法/时序三层面重新组织
  - 语义层: ICE Agent、候选类型、角色、实现类型
  - 语法层: 候选优先级公式、候选对公式、STUN 扩展属性
  - 时序层: 五阶段流程、候选对状态机、会话状态、定时器

---

## [2026-04-22] translate | 完成 RFC 6887 完整中英对照翻译

- 创建文件: `raw/rfcs/rfc6887_完整中英对照.md`
- 标题: "Port Control Protocol (PCP)"
- 作者: D. Wing, S. Cheshire, M. Boucadair, R. Penno, P. Selkirk
- 翻译内容: RFC 6887 完整 21 个章节 + 附录
- 翻译格式: 中英文段落对照
- 涵盖内容:
  - Section 1-5: Introduction, Scope, Terminology, PCP Server Relationship, Fixed-Size Addresses
  - Section 6-9: Protocol Design Note, Common Header Format, Options, Result Codes, General Operation
  - Section 10-12: MAP Opcode, PEER Opcode, Options (THIRD_PARTY, PREFER_FAILURE, FILTER)
  - Section 13-16: Rapid Recovery, Mapping Lifetime, Implementation Considerations
  - Section 17-21: Deployment, Security, IANA, Acknowledgments, References
  - Appendix A: NAT-PMP Transition

---

## [2026-04-22] wiki | 创建 RFC 5389 协议分析页

- 创建文件: `wiki/protocols/protocol-rfc5389.md`
- 内容: 从语义/语法/时序三层面分析 STUN 协议
  - 语义层: 地址发现、连接检查、保活功能
  - 语法层: 20字节头 + TLV属性、Magic Cookie、Binding 方法
  - 时序层: Binding 事务、重传机制、认证握手流程
- 相关链接: [[wiki/concepts/concept-protocol]]

---

## [2026-04-22] wiki | 创建"协议通俗理解"概念页

- 创建文件: `wiki/concepts/concept-protocol.md`
- 内容: 协议的三层抽象（语义/语法/时序）、状态机概念、分层结构、学习协议的三个问题
- 相关链接: [[wiki/concepts/concept-ice]], [[raw/rfcs/rfc7350]]

---

## [2026-04-22] ingest | 下载 RFC 7350

- 下载文件: `raw/rfcs/rfc7350.md`
- 标题: "Datagram Transport Layer Security (DTLS) as Transport for Session Traversal Utilities for NAT (STUN)"
- 作者: M. Petit-Huguenin, G. Salgueiro
- 更新: RFC 5389 和 RFC 5928

---

## [2026-04-22] translate | 完成 RFC 7350 完整中英对照翻译

- 创建文件: `raw/rfcs/rfc7350_完整中英对照.md`
- 翻译内容: RFC 7350 完整 8 个章节 + 附录
- 翻译格式: 中英文段落对照
- 涵盖内容:
  - Section 1: Introduction (简介)
  - Section 2: Terminology (术语)
  - Section 3: DTLS as Transport for STUN (DTLS 作为 STUN 的传输协议)
  - Section 4: STUN Usages (STUN 用法)
    - 4.1-4.5: NAT Discovery, Connectivity Check, Media/SIP Keep-Alive, NAT Behavior Discovery
    - 4.6: TURN Usage (TURN 用法)
  - Section 5: Security Considerations (安全考虑)
  - Section 6: IANA Considerations (IANA 注意事项)
  - Section 7: Acknowledgements (致谢)
  - Section 8: References (参考文献)
  - Appendix A: Examples (示例)

---

## [2026-04-21] translate | 完成 RFC 5766 完整中英对照翻译

- 创建文件: `raw/rfcs/rfc5766_完整中英对照.md`
- 翻译内容: RFC 5766 完整 21 个章节
- 翻译格式: 中英文段落对照
- 涵盖内容:
  - Section 1: Introduction (简介)
  - Section 2: Overview of Operation (操作概述)
  - Section 3: Terminology (术语)
  - Section 4: General Behavior (一般行为)
  - Section 5-7: Allocations, Creating/Refreshing (分配、创建/刷新)
  - Section 8: Permissions (权限)
  - Section 9: CreatePermission (创建权限)
  - Section 10: Send and Data Methods (发送和数据方法)
  - Section 11: Channels (通道)
  - Section 12: IP Header Fields (IP 头字段)
  - Section 13-15: New STUN Methods/Attributes/Error Codes
  - Section 16: Detailed Example (详细示例)
  - Section 17-21: Security, IANA, IAB, Acknowledgements, References

---

## [2026-04-21] translate | 完成 RFC 5389 完整中英对照翻译

- 创建/更新文件: `raw/rfcs/rfc5389_完整中英对照.md`
- 翻译内容: RFC 5389 完整 22 个章节 + 附录 + 作者地址 + 版权声明
- 翻译格式: 中英文段落对照
- 涵盖内容:
  - Section 1-6: Introduction, Evolution, Overview, Terminology, Definitions, Message Structure
  - Section 7-9: Base Procedures, FINGERPRINT, DNS Discovery
  - Section 10-14: Authentication, ALTERNATE-SERVER, Backwards Compatibility, Server, Usages
  - Section 15: STUN Attributes (MAPPED-ADDRESS, XOR-MAPPED-ADDRESS, USERNAME, MESSAGE-INTEGRITY, FINGERPRINT, ERROR-CODE, etc.)
  - Section 16-22: Security, IAB, IANA, Changes, Contributors, Acknowledgements, References
  - Appendix A: C Snippet

---

## [2026-04-21] translate | 完成 RFC 8445 完整中英对照翻译

- 创建文件: `raw/rfcs/rfc8445_完整中英对照.md`
- 翻译内容: RFC 8445 完整 22 个章节 + 3 个附录 + 致谢 + 作者地址
- 翻译行数: 2514 行（原文约 5135 行）
- 翻译格式: 中英文段落对照
- 涵盖内容:
  - Section 1-4: Introduction, Overview, ICE Usage, Terminology
  - Section 5-7: Candidate Gathering, Processing, Connectivity Checks
  - Section 8-12: Concluding ICE, Restarts, Option, Keepalives, Data Handling
  - Section 13-18: Extensibility, Ta/RTO, Examples, STUN Extensions, Operational, IAB
  - Section 19-22: Security, IANA, Changes, References
  - Appendix A-C: Lite/Full, Design, Bandwidth

---

## [2026-04-21] init | 初始化 WebRTC 知识库

- 根据 Karpathy LLM Wiki 方法论创建三层架构
- 创建目录结构: `raw/`, `wiki/`
- 移动 `协议/收集协议.md` → `raw/rfcs/收集协议.md`
- 创建 `CLAUDE.md` (Schema 规范)
- 创建 `index.md` (内容目录)
- 创建 `log.md` (本时间线)

---

## [2026-04-21] init | 安装 obsidian-skills

- 从 https://github.com/kepano/obsidian-skills 安装 5 个 skills
- 安装路径: `.claude/skills/`
- Skills: obsidian-markdown, obsidian-bases, json-canvas, obsidian-cli, defuddle

---

---

## [2026-04-21] ingest | 下载 RFC 文档

- 使用国内镜像 https://www.rfc-editor.org/ 下载 RFC HTML 文件
- 下载目录: 
- 下载的 RFC: rfc8445.html (293K), rfc5389.html (160K), rfc7675.html (38K), rfc5766.html (206K), rfc3550.html (300K), rfc6544.html (98K), rfc6887.html (259K)
- 移除 old RFC: rfc5245, rfc3489, rfc6886
- 更新  链接

---

---

## [2026-04-21] ingest | 下载 RFC 文档

- 使用国内镜像 https://www.rfc-editor.org/ 下载 RFC HTML 文件
- 下载目录: raw/rfcs/
- 下载的 RFC: rfc8445.html (293K), rfc5389.html (160K), rfc7675.html (38K), rfc5766.html (206K), rfc3550.html (300K), rfc6544.html (98K), rfc6887.html (259K)
- 移除 old RFC: rfc5245, rfc3489, rfc6886
- 更新 raw/rfcs/收集协议.md 链接

---

---

## [2026-04-21] convert | HTML to Markdown 转换

- 使用 turndown 将 `raw/rfcs/` 下的 HTML 文件转换为 Markdown
- 转换结果:
  - rfc3550.md (271K)
  - rfc5389.md (140K)
  - rfc5766.md (183K)
  - rfc6544.md (84K)
  - rfc6887.md (233K)
  - rfc7675.md (30K)
  - rfc8445.md (258K)
- 保留原始 HTML 文件

---

---

## [2026-04-21] ingest | 创建 RFC 8445 Wiki 页面

- 阅读 `raw/rfcs/rfc8445.md` 并提取关键内容
- 创建 `wiki/protocols/protocol-rfc8445.md`
- 内容包含：摘要、候选类型、工作流程、优先级公式、实现类型、与 RFC 5245 的变化
- 更新 `index.md` 添加新页面条目

---

---

## [2026-04-21] ingest | 创建 ICE 通俗理解概念页

- 基于 RFC 8445 创建通俗易懂的 ICE 概念解释
- 创建 `wiki/concepts/concept-ice.md`
- 内容包含：通俗类比、三阶段流程、关键概念速查
- 更新 `index.md` 添加页面条目

---

---

## [2026-04-21] translate | RFC 8445 中文翻译

- 创建 `raw/rfcs/rfc8445_中文.md`
- 完整翻译 RFC 8445 为中文
- 包含：摘要、术语、工作流程、示例、安全考虑等所有主要章节
- 更新 `index.md` 添加翻译文件条目

---

---

## [2026-04-21] translate | RFC 8445 中英对照翻译

- 创建 `raw/rfcs/rfc8445_中英对照.md`
- 采用英文段落与中文段落交替对照的格式
- 包含所有主要章节的对照翻译
- 更新 `index.md` 添加翻译文件条目

---
