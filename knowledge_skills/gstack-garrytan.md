---
created: 2026-07-07
tags:
  - gstack
  - garry-tan
  - claude-code
  - skills
  - workflow
aliases:
  - garrytan/gstack
  - Garry Tan gstack
---

# gstack — Garry Tan 的 Claude Code 工程框架

> **一句话概括：** Y Combinator CEO Garry Tan 开源的 Claude Code 配置框架，通过 23 个专家角色和 8 个能力工具，把一个 AI 助手变成一支完整的软件工程团队，实现"每周一万行代码"。

## 项目背景

**gstack** 由 **Y Combinator CEO Garry Tan** 创建并开源，是他个人实际使用的 Claude Code 配置，GitHub 上达到 **71,000~85,000+ Star**。

> "Peter Steinberger 基本上靠 AI Agents 独自构建了 OpenClaw（247K GitHub Stars）。革命已经来临。一个有正确工具的独立开发者，可以比一支传统团队跑得更快。"
> —— Garry Tan

**核心理念：** 把 Claude Code 从"通用助手"升级为"有明确分工的虚拟工程团队"，不是让 AI 随意发挥，而是用严格的角色和流程约束它。

支持平台：Claude Code、Codex、Cursor、OpenClaw 及其他 Agent 宿主。

---

## 核心概念：Think → Plan → Build → Review → Test → Ship → Reflect

gstack 的工作不是让 AI "随便写代码"，而是强制走一个固定的 7 步交付循环：

```
Think（重构问题）
    ↓
Plan（锁定计划）
    ↓
Build（编写代码）
    ↓
Review（代码审查）
    ↓
Test（浏览器 QA）
    ↓
Ship（干净发布）
    ↓
Reflect（复盘总结）
```

每一步都有对应的专家角色和斜杠命令，强制 AI 在正确的"模式"下工作。

---

## 两大核心组件

| 组件 | 说明 |
|------|------|
| **23 个专家角色（Skills）** | 每个角色对应一个 `/命令`，切换到专属人格、优先级和约束 |
| **8 个能力工具（Power Tools）** | 持久化浏览器、代码审查、QA 测试、发布管理等执行能力 |

---

## 一、23 个专家角色

每个角色是一个独立的 Markdown 技能文件，定义了：人格、优先级、约束、输出格式。

### 产品 & 战略类

| 角色 | 命令 | 职责 |
|------|------|------|
| **CEO** | `/ceo` | 重构问题，挑战你的前提假设，从高层视角审视产品方向 |
| **Product Manager** | `/pm` | 六个逼问问题，从用户视角审查功能，输出设计文档 |
| **Strategy Advisor** | `/strategy` | 商业模式分析，竞争态势，市场定位 |

### 工程 & 架构类

| 角色 | 命令 | 职责 |
|------|------|------|
| **Engineering Manager** | `/em` | 技术规划，团队协调，工程决策把控 |
| **Architect** | `/architect` | 系统架构设计，技术选型，权衡分析 |
| **Senior Engineer** | `/senior` | 高质量代码实现，代码规范，性能优化 |
| **Security Officer** | `/security` | 安全审查，漏洞检测，合规检查 |
| **Database Expert** | `/db` | 数据库设计，查询优化，迁移策略 |

### 质量 & 测试类

| 角色 | 命令 | 职责 |
|------|------|------|
| **QA Lead** | `/qa` | 测试策略，边界情况，验收测试 |
| **Code Reviewer** | `/review` | 代码审查，重构建议，最佳实践检查 |
| **Performance Engineer** | `/perf` | 性能分析，瓶颈定位，优化方案 |

### 设计 & 文档类

| 角色 | 命令 | 职责 |
|------|------|------|
| **Designer** | `/design` | UI/UX 设计，用户体验，视觉规范 |
| **Doc Engineer** | `/docs` | 技术文档撰写，API 文档，用户指南 |
| **Release Manager** | `/release` | 发布流程，版本管理，发布清单 |

### 其他专家

- `/devops` — DevOps 工程师，CI/CD、容器化、基础设施
- `/data` — 数据工程师，数据管道、分析、BI
- `/mobile` — 移动端专家，iOS/Android 最佳实践
- `/frontend` — 前端专家，React/Vue/CSS 深度优化
- `/backend` — 后端专家，API 设计、服务架构
- `/ml` — 机器学习工程师，模型集成、推理优化
- `/sre` — 站点可靠性工程师，可观测性、告警
- `/growth` — 增长工程师，A/B 测试、用户转化
- `/retrospective` — 复盘主持人，Sprint 回顾，经验提炼

---

## 二、8 个能力工具（Power Tools）

| 工具 | 命令 | 功能 |
|------|------|------|
| **浏览器自动化** | `/browse` | 持久化 Headless Chromium，70+ 命令，~100ms/次 |
| **代码审查** | `/review` | 全面代码质量审查，可配置严格程度 |
| **计划锁定** | `/plan` | 制定并确认开发计划，防止随意扩展 |
| **发布管理** | `/ship` | 一条命令完成干净发布 |
| **QA 测试** | `/qa` | 浏览器驱动的端到端 QA 测试 |
| **复盘** | `/retro` | Sprint 复盘，提取经验教训 |
| **范围管理** | `/scope` | 四种模式：扩展、选择性扩展、保持范围、缩减 |
| **设计文档** | `/design-doc` | 产出喂给下游技能的设计文档 |

---

## 三、特色功能：持久化浏览器

这是 gstack 的独特竞争力，其他技能框架几乎没有同等能力。

### 架构

```
Claude Code
    ↓
/browse 命令（CLI 二进制）
    ↓
持久化 Headless Chromium 守护进程（基于 Playwright）
    ↓
真实浏览器操作（导航、填表、截图、检查控制台错误）
```

### 为什么持久化很重要

| 冷启动浏览器（每次命令） | 持久化浏览器（gstack 方案） |
|------------------------|--------------------------|
| 每次等待 3-5 秒 | ~100-200ms 每次调用 |
| 每次丢失 cookie 和登录状态 | 保留登录状态、标签页 |
| 无法跨命令维持上下文 | 支持复杂多步骤自动化 |
| 消耗大量 context token | 零 context token 开销 |

### 浏览器能力

- 页面导航、元素点击、表单填写
- 截图（可视化验证）
- 控制台错误检查
- Real Browser 模式（Chrome 侧边栏 + Claude PTY）
- ngrok 对代理流（pair-agent flow）
- 多层提示注入防护

---

## 四、标准开发流程示例

```
你: /ceo 我想做一个 AI 简历筛选工具
    └── CEO 角色：挑战你的假设，提六个逼问问题
            "HR 愿意为 AI 筛选结果负责吗？"
            "你的护城河是什么？"
    ↓
你: /pm 输出产品需求文档
    └── PM 角色：产出设计文档（喂给下游技能）
    ↓
你: /architect 设计系统架构
    └── Architect 角色：输出技术架构和选型决策
    ↓
你: /plan 锁定开发计划
    └── 确认后不允许随意扩展范围
    ↓
你: /senior 实现核心功能（Claude 按计划写代码）
    ↓
你: /review 代码审查
    └── Code Reviewer 角色：全面审查，输出修复建议
    ↓
你: /qa 浏览器 QA 测试
    └── 用持久化浏览器自动跑测试，截图验证
    ↓
你: /ship 发布
    └── Release Manager 角色：干净发布流程
    ↓
你: /retro 复盘
    └── 提取本次迭代的经验教训
```

---

## 五、安装方法

```bash
# 方法一：一行安装（推荐）
npx gstack install

# 方法二：手动安装
git clone https://github.com/garrytan/gstack.git
cp -r gstack/.claude/* ~/.claude/

# 验证安装
claude /ceo --help
```

安装后在 Claude Code 中输入 `/` 即可看到所有 gstack 命令。

---

## 六、gstack vs Superpowers vs ECC

| 对比维度 | **gstack** | **Superpowers** | **ECC** |
|---------|-----------|-----------------|---------|
| **作者** | Garry Tan（YC CEO） | Jesse Vincent（obra） | Affaan Mustafa |
| **GitHub Stars** | 71K-85K | ~16K | ~50K |
| **核心特色** | 持久化浏览器 + 角色扮演 | 流程方法论 | 完整工程体系 |
| **专家角色** | 23 个 | — | 61 个 Agent |
| **浏览器自动化** | ✅ 原生（杀手级功能） | ❌ | 有 E2E Runner |
| **学习曲线** | 中等 | 平缓 | 较陡 |
| **定位** | 角色扮演 + QA 自动化 | 流程纪律 | 完整工程体系 |
| **适合场景** | 需要浏览器操作 + 多角色协作 | 流程控制 | 企业级工程体系 |

### 三者配合使用方案

```
Superpowers（流程纪律）
    ├── brainstorming → writing-plans → executing
    │
gstack（角色专业化 + 浏览器）
    ├── /ceo → 战略确认
    ├── /browse → 浏览器 QA
    ├── /retro → 复盘
    │
ECC（质量工具链）
    ├── Code Reviewer → 代码审查
    ├── Security Reviewer → 安全扫描
    └── Build Error Resolver → 构建修复
```

---

## 七、注意事项

| 注意点 | 说明 |
|--------|------|
| **浏览器需要单独安装** | `/browse` 依赖编译好的 CLI 二进制，需要系统有 Chromium |
| **角色不是万能的** | 每个角色只能在自己的专业域内做好决策 |
| **设计文档是核心** | `/pm` 产出的设计文档会喂给所有下游技能，要认真写 |
| **不要跳过 /plan** | 计划锁定后 AI 才会约束自己不随意扩展功能 |

---

## 推荐阅读

- [[ecc-superpowers-integration]] — ECC + Superpowers 配合使用
- [[superpowers/superpowers]] — Superpowers 14 个技能详解
- [[everything-claude-code]] — ECC 六大核心模块

---

*详细信息参考 [GitHub - garrytan/gstack](https://github.com/garrytan/gstack)、[MindStudio 详解](https://www.mindstudio.ai/blog/g-stack-garry-tan-ai-engineering-team) 和 [Augment Code 指南](https://www.augmentcode.com/learn/garry-tan-gstack-claude-code)。*
