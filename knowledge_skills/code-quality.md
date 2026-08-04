---
created: 2026-06-30
source: Skills List.md
---

# code-quality

> **一句话概括：** 在 AI 写代码时强制执行代码质量守则——命名规范、函数精简、错误处理、拒绝花哨——让输出代码更像人写的、可维护的代码。

## 这是什么？

这是一个给 Claude Code 设定"代码质量底线"的技能。它告诉 AI："别只顾着写能跑的代码，要写**好**代码。"

## 它强制哪些规矩？

不同版本侧重点略有不同，但核心思想一致：

| 规则 | 说明 |
|------|------|
| ✅ 有意义的命名 | 变量名、函数名让人一看就懂 |
| ✅ 函数不超过 50 行 | 太长就拆 |
| ✅ 外部调用必须做错误处理 | 不能假设网络/数据库永远正常 |
| ✅ 代码自文档化 | 能写清楚的代码就别加注释来解释 |
| ✅ 没有魔法数字 | 常量该提取就提取 |
| ✅ 提前 return 减少嵌套 | 还你一双健康的眼睛 |
| ✅ 低圈复杂度 | if 嵌套不超过 2 层 |
| ✅ 最小化原则 | 不写多余的代码 |
| ❌ 拒绝花哨 | 不要炫技，不要过早优化，不要多余抽象层 |

## 怎么安装？

```bash
# 轻量版（推荐入门）
npx skills add https://github.com/agricidaniel/claude-code-essentials --skill code-quality

# 完整版（质量门禁+安全检查）
npx skills add https://github.com/T1nker-1220/.claude --skill code-quality-standards
```

## 什么时候用？

- 📝 **每次 AI 写完代码**，让它用这套标准自检一遍
- 🔄 **Code Review 前**，先让质量门禁过滤一遍低级问题
- 🚀 **部署前**，作为质量检查清单

> 核心思想：**在 AI 写代码的时代，质量底线不能交给 AI 自觉，要靠技能来约束。**

---

*详细信息可参考 [SkillsMP](https://skillsmp.com/skills/agricidaniel-claude-code-essentials-vs-code-templates-skills-code-quality-skill-md) 和 [Svenja-dev 的 GitHub](https://github.com/Svenja-dev/claude-code-skills)。*
