# LLM Wiki
# LLM Wiki（LLM 维基）

---

## The core idea
## 核心思想

#### 原文

Most people's experience with LLMs and documents looks like RAG: you upload a collection of files, the LLM retrieves relevant chunks at query time, and generates an answer. This works, but the LLM is rediscovering knowledge from scratch on every question. There's no accumulation. Ask a subtle question that requires synthesizing five documents, and the LLM has to find and piece together the relevant fragments every time. Nothing is built up. NotebookLM, ChatGPT file uploads, and most RAG systems work this way.

The idea here is different. Instead of just retrieving from raw documents at query time, the LLM **incrementally builds and maintains a persistent wiki** — a structured, interlinked collection of markdown files that sits between you and the raw sources. When you add a new source, the LLM doesn't just index it for later retrieval. It reads it, extracts the key information, and integrates it into the existing wiki — updating entity pages, revising topic summaries, noting where new data contradicts old claims, strengthening or challenging the evolving synthesis. The knowledge is compiled once and then *kept current*, not re-derived on every query.

This is the key difference: **the wiki is a persistent, compounding artifact.** The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything you've read. The wiki keeps getting richer with every source you add and every question you ask.

You never (or rarely) write the wiki yourself — the LLM writes and maintains all of it. You're in charge of sourcing, exploration, and asking the right questions. The LLM does all the grunt work — the summarizing, cross-referencing, filing, and bookkeeping that makes a knowledge base actually useful over time.

#### 中文

大多数人与 LLM 和文档的交互方式类似于 RAG：你上传一堆文件，LLM 在查询时检索相关片段，然后生成答案。这种方式可以工作，但 LLM 每次回答问题都要从头重新发现知识。没有积累。问一个需要综合五个文档的微妙问题，LLM 每次都必须找到并拼凑相关碎片。没有东西被建立起来。NotebookLM、ChatGPT 文件上传和大多数 RAG 系统都是这样工作的。

这里的想法不同。LLM 不只是在查询时从原始文档中检索，而是在查询时，LLM **增量构建和维护一个持久的 wiki**——一个结构化的、互相链接的 markdown 文件集合，位于你和原始资料之间。当你添加新资料时，LLM 不只是索引它以供以后检索。它会阅读它，提取关键信息，并将其整合到现有的 wiki 中——更新实体页面、修改主题摘要、注意新数据与旧说法矛盾的地方、加强或挑战正在发展的综合认知。知识被编译一次，然后*保持最新*，而不是每次查询时重新派生。

关键区别在于：**wiki 是一个持久的、复合的产物。** 交叉引用已经在那里了。矛盾已经被标记了。综合已经反映了你所读的一切。Wiki 随着你添加的每个资料和每个问题而变得更加丰富。

你从不（或很少）自己写 wiki——LLM 编写和维护所有内容。你负责资料来源、探索和提出正确的问题。LLM 做所有的苦力工作——总结、交叉引用、整理和簿记，使知识库随着时间的推移真正有用。

---

## Architecture
## 架构

#### 原文

There are three layers:

**Raw sources** — your curated collection of source documents. Articles, papers, images, data files. These are immutable — the LLM reads from them but never modifies them. This is your source of truth.

**The wiki** — a directory of LLM-generated markdown files. Summaries, entity pages, concept pages, comparisons, an overview, a synthesis. The LLM owns this layer entirely. It creates pages, updates them when new sources arrive, maintains cross-references, and keeps everything consistent. You read it; the LLM writes it.

**The schema** — a document (e.g. CLAUDE.md for Claude Code or AGENTS.md for Codex) that tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow when ingesting sources, answering questions, or maintaining the wiki. This is the key configuration file — it's what makes the LLM a disciplined wiki maintainer rather than a generic chatbot. You and the LLM co-evolve this over time as you figure out what works for your domain.

#### 中文

有三层：

**原始资料 (Raw sources)** —— 你策划的源文档集合。文章、论文、图片、数据文件。这些是不可变的——LLM 从中读取但从不修改。这是你的真相来源。

**Wiki** —— 由 LLM 生成的 markdown 文件目录。摘要、实体页面、概念页面、比较、概述、综合。LLM 完全拥有这一层。它创建页面、在新资料到达时更新它们、维护交叉引用并保持一切一致。你阅读它；LLM 编写它。

**Schema** —— 一个文档（如 Claude Code 的 CLAUDE.md 或 Codex 的 AGENTS.md），告诉 LLM wiki 是如何组织的、有哪些约定，以及在摄取资料、回答问题或维护 wiki 时遵循什么工作流程。这是关键配置文件——它使 LLM 成为有纪律的 wiki 维护者而不是通用聊天机器人。你和 LLM 随着时间的推移共同发展这个文档，因为你弄清楚什么对你的领域有效。

---

## Operations
## 操作

#### 原文

**Ingest.** You drop a new source into the raw collection and tell the LLM to process it. An example flow: the LLM reads the source, discusses key takeaways with you, writes a summary page in the wiki, updates the index, updates relevant entity and concept pages across the wiki, and appends an entry to the log. A single source might touch 10-15 wiki pages.

**Query.** You ask questions against the wiki. The LLM searches for relevant pages, reads them, and synthesizes an answer with citations. Answers can take different forms depending on the question — a markdown page, a comparison table, a slide deck (Marp), a chart (matplotlib), a canvas. The important insight: **good answers can be filed back into the wiki as new pages.**

**Lint.** Periodically, ask the LLM to health-check the wiki. Look for: contradictions between pages, stale claims that newer sources have superseded, orphan pages with no inbound links, important concepts mentioned but lacking their own page, missing cross-references, data gaps that could be filled with a web search.

#### 中文

**摄取 (Ingest)。** 你将新资料放入原始集合并告诉 LLM 处理它。一个示例流程：LLM 阅读资料，与你讨论关键收获，在 wiki 中写一个摘要页面，更新索引，更新整个 wiki 中相关的实体和概念页面，并在日志中追加一个条目。一个资料可能涉及 10-15 个 wiki 页面。

**查询 (Query)。** 你针对 wiki 提出问题。LLM 搜索相关页面，阅读它们，并用引用综合答案。答案可以根据问题的不同采取不同的形式——一个 markdown 页面、一个比较表、一个幻灯片演示（Marp）、一个图表（matplotlib）、一个画布。重要的洞察：**好的答案可以作为新页面存档回 wiki。**

**整理 (Lint)。** 定期让 LLM 对 wiki 进行健康检查。查找：页面之间的矛盾、新资料已取代的过时说法、没有入站链接的孤立页面、被提及但缺乏自己页面的重要概念、缺失的交叉引用、数据空白（可以通过网络搜索填补）。

---

## Indexing and logging
## 索引和日志

#### 原文

Two special files help the LLM (and you) navigate the wiki as it grows. They serve different purposes:

**index.md** is content-oriented. It's a catalog of everything in the wiki — each page listed with a link, a one-line summary, and optionally metadata like date or source count. Organized by category (entities, concepts, sources, etc.). The LLM updates it on every ingest. When answering a query, the LLM reads the index first to find relevant pages, then drills into them.

**log.md** is chronological. It's an append-only record of what happened and when — ingests, queries, lint passes. A useful tip: if each entry starts with a consistent prefix (e.g. `## [2026-04-02] ingest | Article Title`), the log becomes parseable with simple unix tools — `grep "^## \[" log.md | tail -5` gives you the last 5 entries.

#### 中文

两个特殊文件帮助 LLM（和你）浏览不断增长的 wiki。它们服务于不同的目的：

**index.md** 是面向内容的。它是 wiki 中所有内容的目录——每个页面列出一个链接、一行摘要，以及可选的元数据如日期或来源数量。按类别组织（实体、概念、来源等）。LLM 在每次摄取时更新它。在回答查询时，LLM 首先阅读索引以找到相关页面，然后深入阅读它们。

**log.md** 是按时间顺序的。它是发生什么事情和什么时候发生的追加记录——摄取、查询、整理过程。一个有用的技巧：如果每个条目以一致的前缀开始（例如 `## [2026-04-02] ingest | Article Title`），日志可以用简单的 unix 工具解析——`grep "^## \[" log.md | tail -5` 给你最后 5 个条目。

---

## Optional: CLI tools
## 可选：CLI 工具

#### 原文

At some point you may want to build small tools that help the LLM operate on the wiki more efficiently. A search engine over the wiki pages is the most obvious one — at small scale the index file is enough, but as the wiki grows you want proper search. [qmd](https://github.com/tobi/qmd) is a good option: it's a local search engine for markdown files with hybrid BM25/vector search and LLM re-ranking, all on-device. It has both a CLI (so the LLM can shell out to it) and an MCP server (so the LLM can use it as a native tool).

#### 中文

在某些时候，你可能想要构建帮助 LLM 更高效地操作 wiki 的小工具。在 wiki 页面上构建搜索引擎是最明显的——在小规模下索引文件就足够了，但随着 wiki 的增长，你需要适当的搜索。[qmd](https://github.com/tobi/qmd) 是一个很好的选择：它是一个本地 markdown 搜索引擎，具有混合 BM25/向量搜索和 LLM 重排序，全部在设备上。它既有 CLI（所以 LLM 可以调用它），也有 MCP 服务器（所以 LLM 可以将其作为原生工具使用）。

---

## Tips and tricks
## 技巧

#### 原文

- **Obsidian Web Clipper** is a browser extension that converts web articles to markdown. Very useful for quickly getting sources into your raw collection.
- **Download images locally.** In Obsidian Settings → Files and links, set "Attachment folder path" to a fixed directory (e.g. `raw/assets/`). Then in Settings → Hotkeys, search for "Download" to find "Download attachments for current file" and bind it to a hotkey.
- **Obsidian's graph view** is the best way to see the shape of your wiki — what's connected to what, which pages are hubs, which are orphans.
- **Marp** is a markdown-based slide deck format. Obsidian has a plugin for it. Useful for generating presentations directly from wiki content.
- **Dataview** is an Obsidian plugin that runs queries over page frontmatter.
- The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free.

#### 中文

- **Obsidian Web Clipper** 是一个浏览器扩展，可以将网页文章转换为 markdown。对于快速将资料放入原始集合非常有用。
- **本地下载图片。** 在 Obsidian 设置 → 文件和链接中，将"附件文件夹路径"设置为固定目录（例如 `raw/assets/`）。然后在设置 → 热键中，搜索"下载"找到"下载当前文件的附件"并绑定到热键。
- **Obsidian 的图形视图** 是查看 wiki 形状的最佳方式——什么连接到什么，哪些页面是中心，哪些是孤立的。
- **Marp** 是一个基于 markdown 的幻灯片格式。Obsidian 有它的插件。对于直接从 wiki 内容生成演示文稿很有用。
- **Dataview** 是一个在页面 frontmatter 上运行查询的 Obsidian 插件。
- Wiki 只是一个 markdown 文件的 git 仓库。你免费获得版本历史、分支和协作。

---

## Why this works
## 为什么这有效

#### 原文

The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims, maintaining consistency across dozens of pages. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass. The wiki stays maintained because the cost of maintenance is near zero.

The human's job is to curate sources, direct the analysis, ask good questions, and think about what it all means. The LLM's job is everything else.

The idea is related in spirit to Vannevar Bush's Memex (1945) — a personal, curated knowledge store with associative trails between documents. Bush's vision was closer to this than to what the web became: private, actively curated, with the connections between documents as valuable as the documents themselves. The part he couldn't solve was who does the maintenance. The LLM handles that.

#### 中文

维护知识库繁琐的部分不是阅读或思考，而是簿记。更新交叉引用、保持摘要最新、注意新数据何时与旧说法矛盾、在数十个页面之间保持一致性。人类放弃 wiki 是因为维护负担增长快于价值。LLM 不会感到无聊、不会忘记更新交叉引用、可以一次触及 15 个文件。Wiki 保持维护，因为维护成本接近于零。

人类的工作是策划资料、引导分析、提出好问题、思考这一切意味着什么。LLM 的工作是其他一切。

这个想法在精神上与 Vannevar Bush 的 Memex（1945）相关——一个具有文档之间关联路径的个人策划知识库。Bush 的愿景更接近于此，而不是 Web 最终变成的样子：私人、积极策划，文档之间的连接与文档本身一样有价值。他无法解决的部分是谁来做维护。LLM 处理那个。

---

## Note
## 注意

#### 原文

This document is intentionally abstract. It describes the idea, not a specific implementation. The exact directory structure, the schema conventions, the page formats, the tooling — all of that will depend on your domain, your preferences, and your LLM of choice. Everything mentioned above is optional and modular — pick what's useful, ignore what isn't. For example: your sources might be text-only, so you don't need image handling at all. Your wiki might be small enough that the index file is all you need, no search engine required. The right way to use this is to share it with your LLM agent and work together to instantiate a version that fits your needs. The document's only job is to communicate the pattern. Your LLM can figure out the rest.

#### 中文

这份文档有意抽象。它描述了想法，而不是具体的实现。确切的目录结构、schema 约定、页面格式、工具——所有这些都取决于你的领域、你的偏好和你的 LLM 选择。上文提到的所有内容都是可选的和模块化的——选择有用的，忽略无用的。例如：你的资料可能只有文本，所以你完全不需要图片处理。你的 wiki 可能很小，索引文件就是你需要的，没有搜索引擎。正确使用它的方式是将其分享给你的 LLM agent 并共同合作来实例化一个适合你需求的版本。这份文档的唯一工作是传达这个模式。你的 LLM 可以想出其余的部分。

---

*Karpathy LLM Wiki 方法论完整中英对照翻译完成*

*翻译日期：2026-04-23*
