---
created: 2026-06-30
source: Skills List.md
---

# impeccable

> **一句话概括：** 让你的 AI 生成的前端界面告别千篇一律的"AI 风"（Inter 字体、紫蓝渐变、毛玻璃），变成真正有设计感的、生产可用的 UI。

## 这是什么？

Impeccable 是目前最流行的 Claude Code 前端设计技能（排名第一）。它解决了一个普遍痛点：AI 生成的 UI 看起来都一个样——因为训练数据里好的设计就那么几套。

它从三个方面解决这个问题：

1. 📚 **7 份设计参考文件** — 涵盖色彩、排版、布局等
2. 💬 **教学协议** (`/teach-impeccable`) — 你告诉它你的设计偏好，它记住
3. 🛠️ **20+ 个设计命令** — 覆盖设计的方方面面

## 常用命令

| 命令 | 作用 |
|------|------|
| `/impeccable audit` | 审计当前 UI，打分并列出问题 |
| `/impeccable critique` | 给出一份详细的设计评审报告 |
| `/impeccable polish` | 自动打磨 UI 细节 |
| `/impeccable bolder` | 让页面更有视觉冲击力 |
| `/impeccable quieter` | 让页面更温和内敛 |
| `/impeccable colorize` | 优化配色方案 |
| `/impeccable typeset` | 优化排版层级 |
| `/impeccable arrange` | 优化布局和间距 |
| `/impeccable delight` | 加一点"惊喜感" |
| `/impeccable extract` | 提取当前 UI 的设计 token |
| `/impeccable adapt` | 适配不同的设计系统 |

## 它禁止什么？（反模式库）

Impeccable 内置了一个"禁令清单"，明确告诉 AI **不要**做什么：
- ❌ 不要用 Inter 字体（用系统字体栈）
- ❌ 不要用紫蓝渐变
- ❌ 不要用毛玻璃效果（glassmorphism）
- ❌ 不要用过度圆角
- ❌ 不要用大段模糊的阴影
- ❌ 不要让所有东西都居中
- ❌ 不要用默认的 Tailwind 配色

## 怎么装？

```bash
# 一键安装
npx impeccable init

# 或手动安装
npx skills add pbakaus/impeccable
```

## 工作流推荐

1. `/frontend-design` → AI 生成第一版 UI
2. `/impeccable critique` → 获得设计评审
3. `/impeccable polish` → 自动打磨
4. `/impeccable audit` → 确认评分提升

> 一位开发者的实测：审计评分从 **13/20 提升到 18/20**，而且肉眼可见地更好看了。

---

*详细信息可参考 [Apidog 博客](https://apidog.com/blog/impeccable-claude-code-skill/)、[DevelopersIO 实测](https://dev.classmethod.jp/en/articles/claude-code-impeccable-skill-ai-slop-removal/) 和 [npm 包](https://www.npmjs.com/package/impeccable)。*
