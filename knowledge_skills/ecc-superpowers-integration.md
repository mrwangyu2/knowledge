---
created: 2026-07-02
tags:
  - ecc
  - superpowers
  - skills
  - workflow
  - integration
aliases:
  - ECC+Superpowers
  - 双剑合璧
---

# ECC + Superpowers 配合使用指南

> **一句话概括：** Superpowers 管流程（做什么、按什么顺序），ECC 管能力（用什么工具、怎么执行质量门禁），两者互补，合起来就是一套完整的 AI 开发工程体系。

## 为什么要配合使用？

| 单独用 | 问题 |
|--------|------|
| 只装 **Superpowers** | 流程清晰，但遇到具体执行时（修构建、做安全扫描）缺少专用工具，还得手动折腾 |
| 只装 **ECC** | 工具很全，但 AI 可能跳过需求澄清和计划阶段就直接改代码，缺乏流程纪律 |

| 合起来用 | 效果 |
|---------|------|
| **Superpowers 管流程** | brainstorming → writing-plans → 执行 → review → 收尾，一步不跳 |
| **ECC 管执行** | 在每个步骤里，用专用的 Agent 和 Skill 把活干得更专业 |

> **定位分工：Superpowers = 项目经理，ECC = 技术专家团队。**

---

## 安装顺序

```bash
# 1. 安装 Superpowers
/plugin install superpowers

# 2. 安装 ECC
/plugin marketplace add affaan-m/everything-claude-code
/plugin install everything-claude-code@everything-claude-code

# 3. 手动安装 ECC 的 rules（插件无法自动分发）
git clone https://github.com/affaan-m/everything-claude-code.git
cp -r everything-claude-code/rules/* ~/.claude/rules/

# 4. 重启生效
/restart
```

> 两个都装了就行，没有先后要求。

---

## 配合工作流全景

```
┌─────────────────────────────────────────────────────────────┐
│                    Superpowers 流程层                         │
│              （项目经理：把控做什么、按什么顺序）                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  brainstorming  ─────  ECC 提供：Explore Agent 快速调研       │
│      ↓                                                      │
│  writing-plans  ─────  ECC 提供：Planner Agent 辅助拆解       │
│      ↓                                                      │
│  subagent-driven-dev  ─────────────────────────────         │
│  │  ├── TDD 阶段    →  ECC 提供：TDD Guide Agent            │
│  │  ├── 代码实现    →  ECC 提供：语言 Patterns Skill          │
│  │  ├── 构建失败    →  ECC 提供：Build Error Resolver        │
│  │  └── 死代码      →  ECC 提供：Refactor Cleaner            │
│      ↓                                                      │
│  requesting-code-review                                     │
│  │  └── ECC 提供：Code Reviewer Agent 做深度审查              │
│      ↓                                                      │
│  verification-before-completion                             │
│  │  ├── ECC 提供：Security Reviewer 做安全扫描                │
│  │  ├── ECC 提供：E2E Runner 做验收测试                       │
│  │  └── ECC 提供：Test Coverage 做覆盖率检查                  │
│      ↓                                                      │
│  finishing-a-development-branch                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 分步详解

### 1. 需求阶段：brainstorming + ECC Explore Agent

| 角色 | 做什么 |
|------|--------|
| **Superpowers** | 苏格拉底式追问，澄清需求，输出设计文档 |
| **ECC** | Explore Agent 快速调研代码库现状，提供"现有代码中相关的部分"作为参考 |

**典型交互：**
```
你: /brainstorm 我想加一个用户通知系统
   → Superpowers 开始追问：通知类型？渠道？谁触发？
   → ECC 的 Explore Agent 扫描代码库：
     "找到了现有的 User 模型、已有的消息表结构、类似功能"
   → Superpowers 结合调研结果，输出更精确的设计文档
```

### 2. 规划阶段：writing-plans + ECC Planner Agent

| 角色 | 做什么 |
|------|--------|
| **Superpowers** | 把设计拆成 2-5 分钟的细粒度步骤，写出计划文档 |
| **ECC** | Planner Agent 辅助拆解，建议更合理的任务粒度 |

### 3. 执行阶段：subagent-driven-dev + ECC 多种 Agent

这是配合最密集的阶段：

| 子代理工作 | Superpowers 负责 | ECC 负责 |
|-----------|-----------------|---------|
| 写测试 | 确保先写测试（TDD 约束） | TDD Guide Agent 提供标准化测试模板 |
| 写实现代码 | 确保步骤粒度合理不跑偏 | 对应语言的 Patterns Skill 提供最佳实践 |
| 构建失败 | 记录 blocker，决定是否继续 | Build Error Resolver 自动定位修复 |
| 有死代码 | 标记为清理项 | Refactor Cleaner 自动清理 |

### 4. 审查阶段：requesting-code-review + ECC Code Reviewer

| 角色 | 做什么 |
|------|--------|
| **Superpowers** | 触发 code review，按 Critical/Important/Minor 分级要求修复 |
| **ECC** | Code Reviewer Agent 做 5 路并行检查：质量、安全、可维护性、性能、测试覆盖 |

**ECC 审查比 Superpowers 自带审查强在哪？**
- 5 路并行检查，不仅看代码风格
- 提供安全扫描（AgentShield）
- 自动生成结构化审查报告

### 5. 验证阶段 + ECC 安全/测试工具

| ECC 工具 | 在 Superpowers 流程中的位置 |
|---------|--------------------------|
| **Security Reviewer** | 在 `verification-before-completion` 之前做安全扫描 |
| **E2E Runner** | 验证核心用户流程是否正常 |
| **Test Coverage** | 确认测试覆盖率达标（>=80%） |
| **AgentShield** | 扫描配置漏洞和密钥泄露 |

### 6. 收尾阶段

Superpowers 的 `finishing-a-development-branch` 负责收尾决策（合并/PR/保留/丢弃）。ECC 在这个阶段不参与，但之前的验证结果会作为收尾决策的依据。

---

## 什么时候只用其中一个？

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 个人项目，快速迭代 | **Superpowers 就够了** | 流程轻量，够用 |
| 中大型项目，团队协作 | **ECC + Superpowers 一起上** | 流程纪律 + 专业工具都需要 |
| 构建频繁失败，想快速修 | **只装 ECC** | Build Error Resolver 是它最大的价值之一 |
| 代码质量差，需要安全审计 | **只装 ECC** | Security Reviewer 和 AgentShield 是独有功能 |
| 需求经常变，需要流程管控 | **只装 Superpowers** | brainstorming 和 writing-plans 才是你需要的 |

---

## 常见问题

### 问：两个都装会不会冲突？

**不会。** 它们作用在不同层面——Superpowers 管流程顺序，ECC 提供具体执行能力。就像项目经理和技术专家各司其职。

### 问：AI 会不会搞混用哪个？

AI 会根据当前任务自动选择。在 Superpowers 的 `subagent-driven-dev` 阶段，当需要做安全审查时，AI 会自动调用 ECC 的 Security Reviewer Agent，不需要你手动指定。

### 问：token 消耗会不会翻倍？

比单独用一个多，但不是翻倍。因为：
- Superpowers 主要在**主对话**里活动（流程管控）
- ECC 的 Agents 在**子代理**里执行（独立上下文，不污染主会话）
- 两者各自有独立上下文，互不干扰

### 问：先学哪个？

**建议顺序：**
1. 先装 Superpowers，掌握完整的流程概念
2. 熟悉后，再装 ECC，在每个步骤里利用 ECC 的专用工具提升效率

---

## 推荐阅读

- [[superpowers/superpowers]] — Superpowers 14 个技能详解
- [[everything-claude-code]] — ECC 六大核心模块详解
- [[Skills List.md]] — 技能总览

---

*详细信息参考 [drose-yu/claude_code（Superpowers+ECC 融合项目）](https://github.com/drose-yu/claude_code) 和 [腾讯云 ECC 解读](https://cloud.tencent.com.cn/developer/article/2674429)。*
