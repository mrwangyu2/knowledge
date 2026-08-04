# mrwangyu2 的知识库

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) [![GitHub](https://img.shields.io/badge/GitHub-Repository-black?logo=github&logoColor=white)](https://github.com/mrwangyu2/knowledge)

个人知识库，使用 **Karpathy LLM Wiki 方法论**构建：**增量、原子化、无花哨的纯粹 Markdown**。整个仓库基于 AI Agent Skills/Tools 驱动。

---

## 📁 Structure

```
knowledge/
├── .agents/skills/              # AI Agent Skills（Matt Pocock 风格）
│   ├── ask-matt/               # Ask Matt：路由感知的技能分发器
│   ├── code-review/            # 代码审查（标准 + 规范双轴评审）
│   ├── diagnosing-bugs/        # Bug 诊断闭环
│   ├── domain-modeling/        # 领域建模与 ADRs
│   ├── grilling/               # 无情的访谈式追问原语
│   ├── implement/              # 功能实现驱动器
│   ├── prototype/              # 一次性原型（用于设计验证而非纸上推理）
│   ├── tdd/                      # 测试驱动开发闭环
│   └── ... (更多技能见 .agents/skills/)

├── knowledge_ai               # 🧠 AI与LLM基础设施 Wiki
│   ├── CLAUDE.md              # Schema（规范定义。
│   ├── index.md               # 内容目录
│   ├── raw/                   # 原始来源（不可改，只供读取）
│   └── wiki/                  # LLM 生成页面
│       ├── entities/          # 人物、组织、工具等实体页
│       ├── concepts/          # 理论与范式概念页
│       └── frameworks/        # 架构与框架

├── knowledge_llm              # 🦙 LLM 工具链深度探索
│   └── llama.cpp-guide.md     # 从零到生产使用 llama.cpp：训练、量化、部署

├── knowledge_skills             # 🛠️ Knowledge Skills Index (Matt Pocock style)
│   ├── Skills List.md           # All agent skills inventory
│   ├── code-quality.md          # Matt Pocock code review practice
│   ├── everything-claude-code.md  # Claude Code feature deep dive
│   └── superpowers/superpowers.md  # Project Superpowers integration guide

├── knowledge_webrtc            # 🌐 WebRTC 协议栈与 Libnice 知识库
│   ├── CLAUDE.md              # Schema（规范定义）
│   ├── index.md               # 内容目录
│   ├── raw/                   # 原始 RFC draft XEP 等规范文档（只读）
│   ├── wiki/                  # LLM生成的总结对比教程。
│   └── bee_net/               # 自研 BeeNet WebRTC 网状网络协议规范

├── node_modules/.agents/)

└── package.json
```


## 🧼 哲学与工作流

本仓库基于以下核心原则：

### Karpathy LLM Wiki 方法论（每个 vault 里均可见其文档）

1. **Raw** (`raw/`):   
   - 原始输入，不可变的“真相源”。  
   - LLM**只读不改**。
2. **Wiki** (`wiki/`):     
   - 由 LLM Agent生成并维护的派生的总结、对比和教程。     
   - 你负责阅读与提问，LLM负责书写与整理。 
3. **Schema**（`CLAUDE.md` + `index.md`）+ `log.md`:  
   - 定义结构规范和工作流约定。人工+LLM共同演进。


### Three-Layer Architecture Applied Per Vault

Every `knowledge_*` vault follows:

```yaml
raw/          # Source materials (papers, specs, notes) = "the truth"
wiki/         # Atomic markdown pages (concepts/comparisons/etc.) = "interpreted knowledge"
CLAUDE.md     # Schema + rules + conventions = "the contract!"
log.md        # Change history / decisions track!
```

Agent Skills for workflow automation & decision routing. They define how ideas move through:


### 🔍 Vaults Overview

| 金库 | 用途 | 状态 |
|------|------|------|
| `knowledge_ai` | AI Agent架构与基础设施的 Wiki vault（构建中！）| 🔨 Active |
| `knowledge_llm` | LLM模型工具链深度探索笔记。 ✅ Stable |/ 
| `knowledge_skills` | Matt Pocock式 agent skill 索引与研究笔记🚧 WIP growing collection! | 
| `knowledge_vlog` | 基于 VitePress的个人 Wiki Builder教程与文档 🚀 Functional! |
| `knowledge_webrtc` | WebRTC协议栈深度剖析（ICE/STUN/TURN/SRTP/GCC. libnice源码 tracing；自研 BeeNet 网状网络协议设计🏗️ Deep diving RFCs/source code daily!.


### 🔮 Quick Start


Clone & explore:

```bash
git clone git@github.com:mrwangyu2/knowledge.git
cd knowledge
code .     # or your favorite editor! / pi-coder open ./knowledge_webrtc/wiki/concepts/概念-ice.md    # peek at ICE protocol summary
```


### 🤝 Contributing (self-notes for now)

TODO add contribution guidelines / workflow templates later.

Built incrementally by mrwangyu2 & AI agents! 🦜️⛏️
  