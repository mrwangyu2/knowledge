---
created: 2026-06-30
source: Skills List.md
aliases:
  - grill-me
  - grill-with-docs
  - 创业压力测试
---

# startup-pressure-test

> **一句话概括：** 让 AI 扮演一个"较真的面试官"，对你的创业想法或技术方案一轮接一轮地追问，直到把每一个隐藏的假设和盲区都翻出来。

## 这是什么？

在 Matt Pocock 的 120K+ Star 技能库中，`startup-pressure-test` 对应的就是大名鼎鼎的 **`/grill-me`** 技能。它的灵感来自"苏格拉底式追问"——不给你答案，只不断地问问题，直到你想清楚。

## 它能做什么？

- 🗣️ **一轮接一轮地追问**，直到每个分支都考虑清楚
- 🔍 **发现方案中的盲区**——你没想到的边界情况、依赖风险、假设漏洞
- 💰 **帮你省下返工时间**——据说平均省下至少 3 小时的改方案时间
- 📄 **`/grill-with-docs` 进阶版** — 追问的同时还更新项目术语表（CONTEXT.md）和架构决策记录（ADR）

## 什么时候用？

| 场景 | 用什么版本 |
|------|-----------|
| 刚有一个创业想法，还没写代码 | `/grill-me` |
| 准备做一个新功能，需求还不清晰 | `/grill-me` |
| 有代码库，想验证方案是否靠谱 | `/grill-with-docs` |
| 团队讨论有分歧，需要一个"中立追问者" | `/grill-me` |

## 它会问什么类型的问题？

- "这个方案的用户是谁？你怎么验证他们真的需要这个？"
- "如果这个依赖下个月不维护了，你的备选是什么？"
- "你说的'快速'具体是多快？怎么衡量？"
- "这个假设如果错了，代价是什么？"
- "竞争对手如果明天复制这个功能，你的护城河在哪？"

## 怎么装？

```bash
# 安装 grill-me（纯追问版）
npx skills@latest add mattpocock/skills/grill-me

# 或安装 grill-with-docs（带文档更新的进阶版）
npx skills@latest add mattpocock/skills/grill-with-docs
```

> 💡 **建议**：第一次用的时候，挑一个你正在犹豫不决的小决定来试。你会惊讶于有多少盲区是自己没发现的。

---

*详细信息可参考 [Matt Pocock/skills](https://github.com/mattpocock/skills) 和 [Best of JS](https://bestofjs.org/projects/mattpocock-skills)。*
