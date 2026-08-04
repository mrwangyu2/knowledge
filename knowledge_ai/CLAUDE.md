# knowledge_ai - LLM Wiki Vault

> 这是一个基于 Karpathy 的 Wiki 方法论构建的个人知识库。LLM 增量构建和维护 wiki，人类负责资料来源和提问。

---

## 架构

### 三层结构

1. **Raw Sources** (`raw/`)
   - 原始资料集合（文章、论文、图片、数据文件）
   - **不可变** - LLM 只读取，不修改
   - 这是你的真相来源

2. **Wiki** (`wiki/`)
   - LLM 生成的 markdown 文件集合
   - LLM 完全拥有此层 - 创建、更新、维护
   - 你阅读它，LLM 编写它

3. **Schema** (本文件)
   - 定义 wiki 结构、约定和工作流程
   - 你和 LLM 共同演进

### Wiki 子目录

```
wiki/
├── entities/      # 实体页面 - 人物、地点、组织、概念
├── concepts/      # 概念页面 - 主题、理论、方法
├── sources/       # 来源页面 - 原始资料的摘要和引用
├── summaries/     # 摘要页面 - 特定主题的综合总结
├── comparisons/   # 比较页面 - 对比分析
└── syntheses/     # 综合页面 - 跨主题的深度综合
```

---

## 工作流程

### Ingest - 摄取新资料

当你添加新资料时，执行以下步骤：

1. **读取原始资料** - 阅读 `raw/` 中的新文件
2. **讨论关键收获** - 与用户确认核心信息
3. **创建源页面** - 在 `wiki/sources/` 创建摘要页
4. **更新相关页面** - 更新相关的 entities 和 concepts
5. **更新索引** - 在 `index.md` 添加新页面
6. **记录日志** - 在 `log.md` 追加条目

**一个资料可能涉及 10-15 个 wiki 页面。**

### Query - 查询

1. **阅读索引** - 先看 `index.md` 找到相关页面
2. **深入阅读** - 读取相关 wiki 页面
3. **综合答案** - 用引用综合回答
4. **存档** - 好的答案可以作为新页面存档回 wiki

### Lint - 整理

定期检查：
- 页面之间的矛盾
- 被新资料取代的过时说法
- 没有入站链接的孤立页面
- 被提及但缺乏自己页面的重要概念
- 缺失的交叉引用
- 可通过网络搜索填补的数据空白

---

## 文件约定

### Wiki 页面格式

```markdown
---
title: 页面标题
type: entity | concept | source | summary | comparison | synthesis
tags: [tag1, tag2]
created: 2026-04-23
updated: 2026-04-23
sources: [相关来源列表]
---

# 页面标题

内容...
```

### 交叉引用

- 使用 Obsidian 的双链语法：`[[页面名称]]`
- 链接到其他 wiki 页面
- 来源链接使用：`[@source-key]` 格式

### 命名约定

- 文件名使用 kebab-case：`machine-learning-intro.md`
- 实体页面：`entity-name.md`
- 概念页面：`concept-name.md`
- 来源页面：`source-title-slug.md`

---

## 特殊文件

### index.md

面向内容的目录，列出所有 wiki 页面：
- 每个页面的链接
- 一行摘要
- 可选元数据（日期、来源数量）

按类别组织：`entities`, `concepts`, `sources`, `summaries`, `comparisons`, `syntheses`

### log.md

按时间顺序的追加记录，格式：

```
## [2026-04-23] ingest | 文章标题
- 创建了源页面
- 更新了 3 个相关概念页面
- 添加了交叉引用

## [2026-04-23] query | 如何学习机器学习
- 回答了问题
- 综合了 5 个 wiki 页面的内容
```

前缀格式：`## [YYYY-MM-DD] type | description`

---

## 工具提示

- **Obsidian Web Clipper** - 将网页转换为 markdown
- **本地下载图片** - 设置 `raw/assets/` 为附件文件夹
- **图形视图** - 查看 wiki 结构和连接
- **Dataview** - 在 frontmatter 上运行查询

---

## 核心原则

1. **Wiki 是持久的复合产物** - 交叉引用已存在，矛盾已标记，综合已完成
2. **LLM 做簿记工作** - 总结、交叉引用、整理和更新
3. **人类做创造性工作** - 资料来源、探索、提问、思考意义
4. **知识编译一次，保持最新** - 不在每次查询时重新派生

---

*本 vault 由 Claude 维护*
