# libnice 源码分析路线图 — 设计文档

- **日期**: 2026-05-15
- **方案**: 混合结构（入口示例 + 模块文档 + 流程追踪）

## 目标

从浅到深、逐函数精读 libnice ICE 实现，产出 11 份分析文档。

## 输出目录

`/home/frank/knowledge/knowledge_webrtc/libnice/`

## 文档清单及顺序

### 入口（1 份）
```
01-entry-examples.md
```
按调用链分析三个 example：simple → sdp → threaded

### 模块分析（5 份，自底向上）
```
02-module-random.md     — random/ (2 文件, ~200 行)
03-module-stun.md       — stun/ (14 文件, ~3000 行, 含 usages/)
04-module-socket.md     — socket/ (18 文件, ~8000 行)
05-module-agent.md      — agent/ (24 文件, ~18000 行)
06-module-gst.md        — gst/ (4 文件, ~1500 行)
```

### 核心流程（4 份）
```
07-flow-candidate-gathering.md
08-flow-connectivity-checks.md
09-flow-nomination-data.md
10-flow-turn-allocation.md
```

### 索引
```
README.md — 总索引及阅读建议
```

## 内容规范

### 入口文档
逐行分析 `main()` 调用链，解释每个 API 参数和作用。

### 模块文档
- 先列文件概况（行数、职责、依赖）
- 逐函数：原型 → 作用 → 关键逻辑
- 关键数据结构单独框出
- 文件末尾附调用关系小结

### 流程文档
- 从 agent 触发点追踪到 socket 层和 stun 层
- 标注每个步骤的源文件和函数名
- ASCII art 状态机图
- 跨模块标注 "→ 详见 0X-module-xxx.md#section"

## 分析深度

逐函数精读：每个函数覆盖原型声明、一句话作用、关键控制流和数据流要点，不逐行翻译代码。
