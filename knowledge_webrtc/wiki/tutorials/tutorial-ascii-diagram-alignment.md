---
title: Obsidian 字符图表对齐方法论
type: tutorial
tags: [obsidian, ascii-art, diagram, alignment, unicode, east-asian-width]
created: 2026-05-27
updated: 2026-05-27
---

# Obsidian 字符图表对齐方法论

## 问题起源

在 `[[wiki/protocols/protocol-rfc3550]]` 中重绘 "RTP 协议体系架构" 图表时，反复遇到右边界 `│` 或 `║` 无法对齐的问题。

## 核心发现：East Asian Width Ambiguous 陷阱

### 关键事实

Unicode 把所有 Box Drawing 字符（`─│┌┐└┘╔╗║═╠╣├┤`，U+2500–U+257F）标注为 **East Asian Width = Ambiguous (A)**。

| 属性 | 含义 | 中文 Windows 行为 |
|------|------|-------------------|
| **N (Neutral)** | 永远是半角 | 宽度 = 1 列 |
| **W (Wide)** | 永远是全角 | 宽度 = 2 列 |
| **A (Ambiguous)** | 取决于上下文 | **中文系统 = 全角 (2 列)** |

### 后果

在中文 Windows 的 Obsidian 中：

```
─ (U+2500) → 渲染宽度 = 2 列
R (U+0052) → 渲染宽度 = 1 列
```

这导致：

- **水平边框行**（全是 `─` 或 `═`）的渲染宽度 ≈ 内容行 × 2
- **不可能用 Unicode 箱线字符做出同时包含半角内容和边框的对齐图表**

## 解决方案：纯 ASCII 边框

### 字符选择

| 用途 | Unicode（不可靠） | ASCII（可靠） |
|------|-------------------|---------------|
| 外框角 | `╔╗╚╝` | `+` |
| 外框水平 | `═` | `-` 或 `=` |
| 外框垂直 | `║` | `|` |
| 内框角 | `┌┐└┘` | `+` |
| 内框水平 | `─` | `-` |
| 内框垂直 | `│` | `|` |
| 视觉分隔 | `╠╣` | `=` |

ASCII 字符（U+0020–U+007E）的 East Asian Width 全部是 **Na (Narrow)** 或 **N (Neutral)**，在所有系统上宽度 = 1，保证对齐。

### 对齐公式

每条线结构：

```
| + 空格 + 内容 + 填充空格 + 空格 + |
  ↑                         ↑
  内缩                      内缩
```

显示宽度约束：

```
1 + dw(内容) + 填充空格数 + 1 = 外框宽度
```

填充公式：

```
填充空格 = 外框宽度 - dw(内容) - 2
```

其中 `dw()` 计算显示宽度：**CJK 字符 = 2，ASCII 字符 = 1**。

### 嵌套盒子

内层盒子内容统一宽度：

```
maxInner = max(所有内层文字DW)
ibox(text) = '| ' + text + padding + ' |'
// ibox DW = 4 + maxInner（恒定值）
```

外层盒子宽度必须容纳内层盒子：

```
outerDW = max(titleDW + 2, iboxDW + 2)
```

### 验证方法

用脚本逐行计算并对比 `dw()`，确保每行相等：

```javascript
for (const line of lines) {
    const content = line.slice(1, -1); // 去掉外层边框
    if (dw(content) !== outerDW) {
        console.log('不对齐!');
    }
}
```

## 适用范围

- ✅ Obsidian 中文 Windows 环境
- ✅ 任何需要中英文混排的 ASCII 图表
- ✅ 嵌套盒子（双层边框）结构

## 结论

> **在中文 Windows 上做字符图表：永远用 ASCII 边框（`+ - | =`），永远不要用 Unicode 箱线字符（`─│┌┐└┘╔╗║═`）。**
