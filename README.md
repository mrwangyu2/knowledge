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

├── knowledge_ai               # 🧠 AI 与 LLM 基础设施 Wiki
│   ├── CLAUDE.md              # Schema（规范定义。
│   ├── index.md               # 内容目录
│   ├── raw/                   # 原始来源（不可改，只供读取）
│   └── wiki/                  # LLM 生成页面
│       ├── entities/          # 人物、组织、工具等实体页
│       ├── concepts/          # 理论 &范式概念页
│       └── frameworks/        # 架构与框架

├── knowledge_llm              # 🦙 LLM 工具链深度探索
│   └── llama.cpp-guide.md     # 从零到生产使用 llama.cpp:训练、量化、部署

├── knowledge_skills             # 🛠️ Knowledge Skills Index (Matt Pocock style)
│   ├── Skills List.md           # All agent skills inventory
│   ├── code-quality.md          # Matt Pocock code review practice
│   ├── everything-claude-code.md  # Claude Code feature deep dive
│   └── superpowers/superpowers.md  # Project Superpowers integration guide

├── knowledge_vlog               # 📝 VitePress-based Personal Wiki Builder!) & Knowledge Docs
│   └── Vite*Press tutorials (5 parts: init → deploy)

├── node_modules/.agents/)

└── package.json
```

## 🩳 Philosophy & Workflow

本仓库采用以下核心原则：

### Karpathy LLM Wiki Methodology (see karpathy's wiki methodology in each vault)

1. **Raw** (`raw/`): 原始输入——不变的真理源！LLM 永远不会修改 raw 文件。
2. **Wiki** (`wiki/`)：由 LLM Agent生成并维护的派生总结、对比和教程！你读它，LLM写它。
3. **Schema**（`CLAUDE.md` + `index.md`）+ `log.md`：定义结构规范和工作流约定（人工与LLM共同演进）。

### Three-Layer Architecture Applied Per Vault

Every `knowledge_*` vault follows:

```yaml
raw/          # Source materials (papers, specs, notes) = "the truth"
wiki/         # Atomic markdown pages (concepts/comparisons/etc.) = "interpreted knowledge"
CLAUDE.md     # Schema + rules + conventions = "the contract!"
log.md        # Change history / decisions track!
```

Agent Skills for workflow automation & decision routing. They define how ideas move through:

- **Grill** → interrogate assumptions! sharpen thinking in structured loops)
- **Prototype** → build throwaway code to test design questions, not paper-reasoning loops! 
- **Implement/TDD** → iterate red-green-refactor until spec-complete! (with review gate on commit.
- **Diagnose/debugging -> Fix-with-regression-test loop!

For full skill routes & flowchart: read `.agents/skills/ask-matt/SKILL.md`.


### 📋 Vaults Overview

| Vault | Purpose | Status |
|-------|---------|--------|
| `knowledge_ai` | AI agent architecture & infrastructure wiki vault under construction!) | 🔨 Active |
| `knowledge_llm` | LLM model toolchain deep-dives | ✅ Stable | 
| `knowledge_skills` | Matt Pocock-style agent skill index + research notes 🚧 WIP growing collection! | 
| `knowledge_vlog` | VitePress-based personal wiki builder tutorials & docs 🚀 Functional! |
| `knowledge_webrtc` | WebRTC protocol stack deep-dive (ICE/STUN/TURN/SRTP/GCC). libnice source tracing. BeeNet mesh protocol design spec 🏗️ Deep diving RFCs/source code daily! | 


### 🔮 Quick Start

Clone & explore:

```bash
git clone git@github.com:mrwangyu2/knowledge.git
cd knowledge
code .     # or your favorite editor! / pi-coder open ./knowledge_webrtc/wiki/concepts/ concept-ice.md    # peek at ICE protocol summary
```


### 🤝 Contributing (self-notes for now)

TODO add contribution guidelines / workflow templates later.


**Built incrementally by mrwangyu2 & AI agents!** 🦜️⛏️
  