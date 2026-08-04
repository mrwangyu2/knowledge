---
created: 2026-06-30
source: Skills List.md
---

# improve-codebase-architecture

> **一句话概括：** 扫描你的代码库，找出那些"接口复杂但功能少"的浅层模块，帮你把它们重构为"接口简单但功能强"的深层模块。

## 这是什么？

这是 Matt Pocock 开源的一款 Claude Code 技能。它的核心思想是"**深度模块**"（Deep Module）——一个模块的接口越简单、内部做的事情越多，架构就越优秀。

## 它能做什么？

- 🔍 **扫描代码库**，自动发现架构层面的改进机会
- 📊 **生成 HTML 报告**，每个候选问题都附带：涉及文件、问题描述、解决方案、收益、推荐强度
- 💬 **和你对话式讨论**每个候选方案，帮你理清约束条件、依赖关系
- 📝 **自动更新术语表**（CONTEXT.md）和**架构决策记录**（ADR）

## 什么时候用？

| 场景 | 说明 |
|------|------|
| 代码库越来越大，感觉"很乱" | 让它扫描一遍，找到重构方向 |
| 想提升代码可测试性 | 深度模块天然更容易测试 |
| 感觉"到处都在重复" | 它能发现需要合并的散落模块 |
| 代码让 AI 读不懂 | 深度模块也是 AI 友好架构 |

## 怎么用？

1. 确保项目根目录有 `CONTEXT.md`（项目术语表）
2. 运行技能后，AI 会探索代码库并生成候选列表
3. 你挑选感兴趣的候选，AI 会和你逐个深入讨论
4. 最终确定方案后实施重构

## 安装

```bash
npx skills add https://github.com/mattpocock/skills --skill improve-codebase-architecture
```

## 小贴士

有人每周一早上对主力项目跑一遍这个技能，就像给代码库做一次"周体检"。你也可以试试！

---

*详细信息可参考 [SkillsMP](https://skillsmp.com/zh/skills/vamseeachanta-workspace-hub-claude-skills-development-improve-codebase-architecture-skill-md) 和 [Matt Pocock 的 GitHub](https://github.com/mattpocock/skills)。*
