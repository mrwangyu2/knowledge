---
created: 2026-07-02
tags:
  - ecc
  - everything-claude-code
  - skills
  - agents
aliases:
  - ECC
  - everything-claude-code
---

# Everything Claude Code（ECC）

> **一句话概括：** 一套让 Claude Code 从"临时助手"升级为"虚拟工程团队"的完整增强系统——内置 61 个子代理、246 个技能、76 个斜杠命令，Anthropic 黑客松冠军作品。

## 项目背景

**Everything Claude Code（简称 ECC）** 由开发者 **Affaan Mustafa** 创建，在 **Anthropic x Forum Ventures 黑客松** 中获得冠军。经过 **10 个月+** 高强度生产环境打磨，GitHub 上获得 **50K+ Star**，是目前最完整的 Claude Code 配置增强系统。

它不是一个简单的提示词合集，而是一套 **AI 编程 Agent 的工程化增强系统（Agent Harness Performance Optimization System）**。通过 **Agents、Skills、Commands、Hooks、Rules、MCP 配置** 六层架构，把 Claude Code 武装成一个完整的"虚拟工程团队"。

核心理念：**前期投入时间构建可复用组件，后期每次使用都在收益——AI 开发中的复利效应。**

---

## 六大核心模块总览

| 模块 | 数量 | 作用 | 类比 |
|------|------|------|------|
| **Agents（子代理）** | 61 个 | 专业角色代理，各司其职 | 你的虚拟工程团队成员 |
| **Skills（技能）** | 246 个 | 领域知识和工作流 | 团队的 SOP 手册 |
| **Commands（斜杠命令）** | 76 个 | 一键启动工作流 | 快捷指令按钮 |
| **Hooks（钩子）** | 事件驱动 | 自动执行的规则脚本 | 24 小时在岗的质检员 |
| **Rules（规则）** | 8+N 个 | 强制执行的开发规范 | 团队的宪法 |
| **MCP 配置** | 多服务 | 外部工具集成 | 团队的通讯录 |

---

## 一、Agents（子代理）— 你的虚拟工程团队

Agents 是 ECC 的核心执行单元。每个子代理专注特定开发任务，形成完整"工程团队"。

### 核心 Agents 列表

| Agent | 职责 | 什么时候用 |
|-------|------|-----------|
| **Planner** | 功能实现规划，拆解需求为可执行步骤 | 开始一个新功能前 |
| **Architect** | 系统架构设计，技术决策与权衡分析 | 涉及重大架构变更 |
| **TDD Guide** | 测试驱动开发，强制 RED→GREEN→REFACTOR | 写新代码或修 bug |
| **Code Reviewer** | 代码质量与可维护性审查 | 写完代码后、合并前 |
| **Security Reviewer** | 漏洞分析（OWASP Top 10、SQL注入、XSS） | 处理用户输入、认证、API |
| **Build Error Resolver** | 构建/编译错误修复 | 构建失败时 |
| **E2E Runner** | Playwright 端到端测试生成与执行 | 需要验收测试时 |
| **Refactor Cleaner** | 死代码清理、重复代码消除 | 代码库需要清理时 |
| **Doc Updater** | 文档与架构图同步更新 | 代码变化后更新文档 |
| **Database Reviewer** | 数据库/Supabase 查询优化与模式审查 | 写 SQL 或迁移脚本时 |

### 5 阶段 Agent 编排模式

ECC 定义了明确的**工程流水线**，任何任务都按这 5 步走：

```
Phase 1: RESEARCH  ──  Explore Agent 调研        → 输出 research-summary.md
Phase 2: PLAN      ──  Planner Agent 做规划        → 输出 plan.md
Phase 3: IMPLEMENT ──  TDD Guide Agent 写代码      → 输出代码改动
Phase 4: REVIEW    ──  Code Reviewer 做审查        → 输出 review-comments.md
Phase 5: VERIFY    ──  有问题？Build Resolver 修复 → 循环直到通过
```

**关键规则：**
- 每个 Agent **只有一个输入**，产出**一个输出**
- 输出是下一阶段的输入
- **永远不要跳过阶段**
- 阶段之间用 `/clear` 清理上下文

---

## 二、Skills（技能/工作流）— 团队的 SOP 手册

Skills 是 ECC 的"知识引擎"，覆盖 12 种编程语言与全栈开发场景。每个 Skill 是一份结构化的领域知识文档。

### 流程类 Skills

| Skill | 功能 |
|-------|------|
| **TDD Workflow** | 测试驱动开发方法论，RED→GREEN→REFACTOR 循环 |
| **Security Review** | 安全审查清单，覆盖 10 大安全领域 |
| **Verification Loop** | 持续验证循环，6 阶段 QA 系统 |
| **Continuous Learning** | 从会话中自动提取模式，积累到记忆库 |
| **Strategic Compact** | 手动压缩建议，优化上下文使用 |

### 语言与框架类 Skills

| Skill | 覆盖内容 |
|-------|---------|
| **Python Patterns** | 类型提示、dataclasses、async/await 惯用法 |
| **Golang Patterns** | Go 惯用语、并发模式、错误处理 |
| **Rust Patterns** | 所有权、生命周期、unsafe 使用规范 |
| **Frontend Patterns** | React/Next.js 组件模式、状态管理 |
| **Backend Patterns** | API 设计、数据库访问、缓存策略 |
| **Django Patterns** | Django 安全、ORM、TDD、生产部署 |
| **React Patterns** | Hook 规范、性能优化、SSR/CSR |
| **Vue Patterns** | Composition API、响应式陷阱、Pinia |

### 进阶 Skills

| Skill | 功能 |
|-------|------|
| **Continuous Learning v2** | 基于"本能"的带置信评分学习系统 |
| **Eval Harness** | 验证循环评测（pass@k、pass^k 指标） |
| **Iterative Retrieval** | 子代理的渐进式上下文精炼 |

---

## 三、Commands（斜杠命令）— 一键启动工作流

76 个斜杠命令，输入 `/xxx` 即刻触发专业工作流。

### 核心命令

| 命令 | 功能 |
|------|------|
| `/plan` | 创建实施计划，确认后开始执行 |
| `/tdd` | 启动 TDD 模式，先写测试再写实现 |
| `/code-review` | 审查未提交的代码变更 |
| `/build-fix` | 修复构建和 TypeScript 错误 |
| `/e2e` | 生成 Playwright 端到端测试 |
| `/refactor-clean` | 清理死代码和未使用的依赖 |
| `/verify` | 运行完整的验证流程 |
| `/checkpoint` | 保存当前的验证状态 |
| `/test-coverage` | 检查测试覆盖率 |
| `/learn` | 从当前会话提取可复用模式 |
| `/eval` | 运行评估测试 |

### 高级命令（v1.7.0+）

| 命令 | 功能 |
|------|------|
| `/orchestrate` | 协调多个 Agent 完成复杂任务 |
| `/multi-plan` | 多 Agent 任务分解与分配 |
| `/multi-execute` | 编排的多 Agent 工作流执行 |
| `/session` | 会话历史管理 |
| `/skill-create` | 从 Git 提交历史生成技能 |
| `/pm2` | PM2 服务生命周期管理 |

---

## 四、Hooks（钩子）— 自动执行的规则

Hooks 是事件驱动的自动化系统，**不需要手动触发**，在特定事件发生时自动运行脚本。

### Hook 事件类型

| 事件 | 触发时机 | 可以用来自动做什么 |
|------|---------|------------------|
| **PreToolUse** | 工具执行前 | 验证参数、提醒注意事项 |
| **PostToolUse** | 工具执行后 | 自动格式化、触发反馈 |
| **UserPromptSubmit** | 用户提交消息时 | 预处理输入 |
| **Stop** | AI 完成响应时 | 保存会话摘要 |
| **SessionStart** | 会话开始时 | 加载历史记忆 |
| **SessionEnd** | 会话结束时 | 持久化上下文、保存记忆 |
| **PreCompact** | 上下文压缩前 | 提取关键信息再压缩 |

### 实用场景

| 场景 | Hook 配置 |
|------|----------|
| **自动格式化** | 每次 Edit 后自动运行 Prettier / Ruff / gofmt |
| **阻止敏感操作** | 拒绝修改 `.env`、`secrets/` 等文件 |
| **操作审计** | 记录所有工具调用到日志文件 |
| **记忆持久化** | 会话结束时保存上下文，新会话自动加载 |
| **自动 Review** | 代码变更后自动触发 Code Review |

---

## 五、Rules（规则）— 团队的宪法

Rules 是**始终生效的开发规范**，强制约束 AI 的生成行为，确保输出质量底线。

### 8 大核心规则

| 规则文件 | 核心要点 |
|---------|---------|
| **security.md** | 禁止硬编码密钥、输入验证、SQL 注入防护 |
| **testing.md** | 最低 80% 测试覆盖率、TDD 优先 |
| **coding-style.md** | 不可变性优先、文件≤800行、函数≤50行 |
| **git-workflow.md** | 约定式提交、功能分支、PR 审查 |
| **agents.md** | 定义什么情况委托给哪个子 Agent |
| **performance.md** | 模型选择策略、上下文预算管理 |
| **patterns.md** | 设计模式应用指南、避免过度设计 |
| **hooks.md** | 钩子使用规范 |

### 多语言规则（v1.4.0+）

```
rules/
├── common/          ← 语言无关的通用规则
├── typescript/      ← TypeScript 专用规则
├── python/          ← Python 专用规则
├── golang/          ← Go 专用规则
└── rust/            ← Rust 专用规则
```

---

## 六、核心方法论

### 1. CLI + Skills 替代 MCP

很多 MCP 工具（GitHub、Supabase、Vercel）可以用 CLI 命令 + Skills 替代，**节省上下文窗口和 Token 消耗**。只有真正需要实时交互的服务才保留 MCP。

### 2. Token 优化策略

| 模型 | 适用场景 | 占比 |
|------|---------|------|
| **Haiku** | 探索搜索、简单编辑、文档撰写 | ~5% |
| **Sonnet** | 多文件实现、PR 审查、日常编码 | ~90% |
| **Opus** | 复杂架构、安全分析、跨 5+ 文件任务 | ~5% |

### 3. 验证循环指标

ECC 定义了两种验证指标：

| 指标         | 含义                 | 例            |
| ---------- | ------------------ | ------------ |
| **pass@k** | k 次尝试中**至少 1 次**成功 | pass@3 = 91% |
| **pass^k** | k 次尝试**必须全部**成功    | pass^3 = 34% |

pass@k 适合宽松场景，pass^k 适合安全关键场景。

### 4. Git Worktrees 并行化

多个 Claude 实例在不同工作树上并行工作：

```bash
git worktree add ../project-feature-a feature-a
cd ../project-feature-a && claude
```

可以同时开 3-4 个实例各做各的功能，互不干扰。

### 5. 双实例启动模式

对于一个复杂任务，同时启动两个 Claude 实例：
- **实例 A**：搭建项目骨架和基础架构
- **实例 B**：做深度研究和方案调研

完成后合并成果。

---

## 七、安装方法

### 方式一：插件安装（推荐）

```bash
# 在 Claude Code 中执行
/plugin marketplace add affaan-m/everything-claude-code
/plugin install everything-claude-code@everything-claude-code

# 然后手动安装 rules（插件无法自动分发 rules）
git clone https://github.com/affaan-m/everything-claude-code.git
cp -r everything-claude-code/rules/* ~/.claude/rules/
```

### 方式二：手动安装

```bash
git clone https://github.com/affaan-m/everything-claude-code.git
cp everything-claude-code/agents/*.md ~/.claude/agents/
cp everything-claude-code/commands/*.md ~/.claude/commands/
cp -r everything-claude-code/skills/* ~/.claude/skills/
cp everything-claude-code/rules/* ~/.claude/rules/
```

---

## 八、注意事项

| 注意点 | 说明 |
|--------|------|
| **Context Window** | 200K 上下文窗口，开启太多工具可能缩水到 70K |
| **不要叠加安装** | 不要同时用插件安装 + install.sh，会重复加载 |
| **按需选规则** | 不要全量复制所有 rules，选项目需要的即可 |
| **安全扫描** | 用 `npx ecc-agentshield scan` 扫描配置漏洞 |

---

## ECC vs Superpowers

| 对比维度 | ECC | Superpowers |
|---------|-----|-------------|
| **作者** | Affaan Mustafa | Jesse Vincent (obra) |
| **规模** | 61 Agents + 246 Skills + 76 Commands | 14 个核心技能 |
| **定位** | 完整工程化增强系统 | 轻量级工作流方法论 |
| **学习曲线** | 较陡，功能极其丰富 | 平缓，容易上手 |
| **适合场景** | 需要完整工程体系的中大型项目 | 个人开发者、小团队快速上手 |
| **共同点** | 都强调"流程大于提示"，都用 Agent 编排工作流 | |

> 两者不是对立关系。详细配合方法见 [[ecc-superpowers-integration]]。

---

## 延伸阅读

- [[ecc-superpowers-integration]] — ECC + Superpowers 配合使用指南
- [[superpowers/superpowers]] — Superpowers 14 个技能详解

*详细信息参考 [GitHub - affaan-m/ECC](https://github.com/affaan-m/everything-claude-code)、[中文翻译版](https://github.com/xu-xiang/everything-claude-code-zh) 和 [知乎详解](https://zhuanlan.zhihu.com/p/1997978582736204425)。*
