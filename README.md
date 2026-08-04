# mrwangyu2 知识库

[![许可证：MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) [![GitHub](https://img.shields.io/badge/GitHub-Repository-black?logo=github&logoColor=white)](https://github.com/mrwangyu2/knowledge)

个人知识库，基于 **Karpathy LLM Wiki 方法论**构建：**增量式、原子化、不花哨的纯 Markdown**。整个仓库由 AI Agent Skills / Tools 驱动。

---

## 📁 目录结构

```
knowledge/
├── .agents/skills/              # AI Agent Skills（Matt Pocock 风格）
│   ├── ask-matt/               # Ask Matt：路由感知型技能分发器
│   ├── code-review/            # 代码审查（同时按“编码规范”与“需求规格”双轴评审）
│   ├── diagnosing-bugs/        # Bug 诊断闭环流程
│   ├── domain-modeling/        # 领域建模与 ADRs（架构决策记录）
│   ├── grilling/               # 追问式盘问原语（无情拷问思路）
│   ├── implement/              # 功能实现驱动器
│   ├── prototype/              # 一次性原型（用于设计验证，而非纸面推理）
│   ├── tdd/                    # 测试驱动开发闭环流程
│   └── ...                     # （更多技能请见 .agents/skills/ 目录）

├── knowledge_ai                # 🧠 AI与LLM基础设施 Wiki
│   ├── CLAUDE.md              # Schema（规范定义文件）
│   ├── index.md               # 内容目录索引
│   ├── raw/                   # 原始资料区（只读不改的“真值源”）
│   └── wiki/                  # LLM 生成的总结 / 对比 / 教程
│       ├── entities/          # 人物、组织、工具等实体页
│       ├── concepts/          # 理论与范式概念页
│       └── frameworks/        # 架构与框架

├── knowledge_llm               # 🦙 LLM 工具链深度探索笔记
│   └── llama.cpp-guide.md     # llama.cpp：从入门到生产环境（训练、量化、部署）

├── knowledge_skills            # 🛠️ Knowledge Skills 索引 (Matt Pocock 风格)
│   ├── Skills List.md           # AI Agent 技能清单汇总
│   ├── code-quality.md          # Matt Pocock 代码审查实践笔记
│   ├── everything-claude-code.md  # Claude Code 功能深度剖析
│   └── superpowers/superpowers.md  # Project Superpowers 集成指南

├── knowledge_webrtc            # 🌐 WebRTC 协议栈与 Libnice 知识库
│   ├── CLAUDE.md              # Schema（规范定义文件）
│   ├── index.md               # 内容目录索引
│   ├── raw/                   # RFC / draft / XEP 等标准文档（只读）
│   ├── wiki/                  # LLM 生成的总结、对比与教程
│   └── bee_net/               # 自研 BeeNet WebRTC 网状网络协议规范

├── node_modules/.agents        # Node agents 相关技能扩展入口示例

└── package.json                # 项目配置文件
```


## 🧼 核心理念与工作流

本仓库遵循以下核心原则：

### Karpathy LLM Wiki 方法论（每个 Vault 中均有对应说明文档）

1. **Raw（原始资料区）** (`raw/`)  
   - 存放原始输入，作为不可变的“真值源”。
   - LLM 仅可以**只读引用**，不得直接修改。

2. **Wiki（知识库页）** (`wiki/`)  
   - 由 LLM Agent 自动生成并维护的派生知识：包括总结、对比与教程等页面。
   - 你负责阅读与提出问题；LLM 负责撰写与整理内容。

3. **Schema（架构契约）**（`CLAUDE.md` + `index.md`）以及 `log.md`  
   - 定义目录结构规范与工作流约定，由人工和 LLM 共同逐步演进维护。


### 每个 Vault 的三层架构

每一个 `knowledge_*` Vault 均遵循以下分层结构：

```yaml
raw/          # 原始资料（论文、标准文档、手记等）——“真值源”
wiki/         # 原子化 Markdown 文章（概念页 / 对比页等）——“经过解读的知识”
CLAUDE.md     # Schema + 规则与约定 —— “契约（规范文档）！”
log.md        # 变更记录与决策追踪日志！
```

此外配套 **AI Agent Skills**，用于工作流自动化与路由判断。它们定义了某个想法或任务如何在整个系统中流转处理。


### 🔍 Vaults 概览

| Vault名称         | 用途说明                                                                                      | 状态           |
|-------------------|---------------------------------------------------------------------------------------------|----------------|
| `knowledge_ai`       | AI Agent 系统架构与基础设施 Wiki（正在持续建设中）                                                  | 🔨 Active      |
| `knowledge_llm`      | LLM模型工具链与落地实践的深入剖析笔记                                                               | ✅ Stable      |
| `knowledge_skills`   | Matt Pocock 风格 agent skill 索引与研究笔记                                                          | 🚧 WIP         |
| `knowledge_vlog`     | 基于 VitePress 构建的个人 Wiki Builder 教程与文档系统                                                   | 🚀 Functional! |
| `knowledge_webrtc`   | WebRTC协议栈深度剖析（ICE/STUN/TURN/SRTP/GCC，libnice源码追踪；以及自研 BeeNet P2P网状网协议设计）          | 🏗️ 持续深潜中    |


### 🔮 快速上手

克隆仓库并查看内容：

```bash
git clone git@github.com:mrwangyu2/knowledge.git
cd knowledge
code .                                          # 用 VS Code（或你喜欢的编辑器）打开
# 示例：查看 ICE 协议总结页面：
pi-coder open ./knowledge_webrtc/wiki/concepts/概念-ice.md
```


### 🤝 贡献说明（目前仅限个人备忘使用）

TODO：后续再补充正式的贡献指南与工作流模板。

本仓库采用增量式演进方式持续维护，作者：mrwangyu2 & AI agents! 🦜️⛏️
