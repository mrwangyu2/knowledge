# Log - 操作日志

> 按时间顺序的追加记录：ingests, queries, lint passes

---

## [2026-04-23] init | Vault 初始化

- 基于 Karpathy 的 LLM Wiki 方法论创建了 knowledge_ai vault
- 创建了目录结构：`raw/`, `wiki/entities`, `wiki/concepts`, `wiki/sources`, `wiki/summaries`, `wiki/comparisons`, `wiki/syntheses`
- 创建了 Schema 文件：`CLAUDE.md`
- 创建了索引文件：`index.md`
- 创建了日志文件：`log.md`

---

## [2026-04-23] ingest | the-shortform-guide.md

- 来源：https://github.com/affaan-m/everything-claude-code/blob/main/the-shortform-guide.md
- 保存位置：`raw/the-shortform-guide.md`
- 内容：Claude Code 使用指南（skills, hooks, subagents, MCPs, plugins, 键盘快捷键等）
- 标签：#claude-code #tutorial #workflow

---

*日志格式：`## [YYYY-MM-DD] type | description`*
