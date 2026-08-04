# mrwangyu2's Knowledge Base

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) [![GitHub](https://img.shields.io/badge/GitHub-Repository-black?logo=github&logoColor=white)](https://github.com/mrwangyu2/knowledge)

个人知识库，使用 **Karpathy LLM Wiki 方法论**构建：**增量、原子化、无花哨的纯粹 Markdown**，所有文件夹基于 AI-agent skills/tools。

---

## 📁 Structure

```
knowledge/
├── .agents/skills/              # AI Agent Skills (Matt Pocock style)
│   ├── ask-matt/               # Ask Matt: route-aware skill dispatcher
│   ├── code-review/            # Code review (Standards + Spec axes)
│   ├── diagnosing-bugs/        # Bug diagnosis loop
│   ├── domain-modeling/        # Domain language & ADRs
│   ├── grilling/               # Relentless interview primitive
│   ├── implement/              # Feature implementation driver
│   ├── prototype/              # Throwaway prototypes, for design Qs not paper reasoning)
│   ├── tdd/                      # Test-driven development loop
│   └── ... (more skills: see .agents/skills/)

├── knowledge_ai               # 🧠 AI & LLM Infrastructure Wiki
│   ├── CLAUDE.md              # Schema definition
│   ├── index.md               # Content directory
│   ├── raw/                   # Raw sources (immutable)
│   └── wiki/                  # LLM-generated pages
│       ├── entities/          # People, orgs, tools
│       ├── concepts/          # Theories & paradigms
│       └── frameworks/        # Architectures & patterns

├── knowledge_llm              # 🦙 LLM Toolchain Deep-Dives
│   └── llama.cpp-guide.md     # 从零到生产使用 llama.cpp: 训练、量化、部署

├── knowledge_skills             # 🛠️ Knowledge Skills Index (Matt Pocock style)
│   ├── Skills List.md           # All agent skills inventory
│   ├── code-quality.md          # Matt Pocock code review practice
│   ├── everything-claude-code.md  # Claude Code feature deep dive
│   └── superpowers/superpowers.md  # Project Superpowers integration guide

├── knowledge_vlog               # 📝 VitePress-based Personal Wiki Builder) & Knowledge Docs
│   └── Vite*Press tutorials (5 parts: init → deploy)

├── knowledge_webrtc            # 🌐 WebRTC Protocol & Libnice 知识库
│   ├── CLAUDE.md              # Schema definition
│   ├── index.md               # Content directory
│   ├── raw/                   # Original RFCs, drafts, XEP specs (read-only!)
│   ├── wiki/                  # LLM-generated summaries/comparisons/tutorials
│   └── bee_net/               # Self-designed BeeNet webrtc mesh protocol spec

├── node_modules/.agents/)

└── package.json
```

## 🩳 Philosophy & Workflow

本仓库采用以下核心原则：

### Karpathy LLM Wiki Methodology (see karpathy's wiki methodology in each vault)

1. **Raw** (`raw/`): original inputs — immutable source of truth! The LLM never mutates raw files.
2. **Wiki** (`wiki/`): derived summaries, comparisons, tutorials generated & maintained by the LLM agent! 你读它，LLM写它。
3. **Schema** (`CLAUDE.md` + `index.md` + `log.md`) 定义结构规范与工作流约定（人工+LLM共同演进）。

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
  