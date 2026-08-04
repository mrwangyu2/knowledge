---
title: VitePress 完整指南
tags: [vitepress, 文档生成器, 静态网站, vue, vite]
created: 2026-07-17
updated: 2026-07-17
---

# VitePress 完整指南

## 📌 概述

VitePress 是一个**静态网站生成器（SSG）**，专为构建**文档网站**而设计。由 Vue.js 团队开发，基于 Vite 构建工具。

**官方链接**: [vitepress.dev](https://vitepress.dev)

---

## 🎯 核心理念

### 设计哲学
- **极速开发体验** - 基于 Vite 的快速热更新（HMR）
- **开箱即用** - 最小化配置，专注内容
- **Vue 原生** - 支持在 Markdown 中使用 Vue 3 组件
- **性能优先** - 静态生成，CDN 友好，零 JavaScript 依赖

### 核心价值
1. 快速迭代 - 毫秒级热更新
2. 低学习曲线 - Markdown 为主，配置简单
3. 灵活扩展 - 支持自定义主题和组件
4. 生产就绪 - 优化的输出，SEO 友好

---

## ⚙️ 工作原理

### 编译流程

```
Markdown 文件
    ↓
[解析] → 提取 frontmatter + 内容
    ↓
[转换] → 转换为 HTML + Vue 组件
    ↓
[主题应用] → 应用文档主题
    ↓
[静态生成] → 预渲染完整 HTML 文件
    ↓
输出目录 (.vitepress/dist)
    ↓
[部署] → CDN/静态托管
```

### 关键技术栈

| 组件 | 技术 | 作用 |
|------|------|------|
| **构建工具** | Vite | 快速编译和热更新 |
| **框架** | Vue 3 | 组件系统和主题 |
| **Markdown 处理** | markdown-it | Markdown 解析 |
| **输出** | 静态 HTML | 生产就绪的网站 |

---

## 🚀 快速开始

### 1. 初始化项目

```bash
npm init vitepress
```

交互式创建项目，选择：
- 项目名称
- 主题（默认推荐）
- TypeScript 支持（可选）

### 2. 本地开发

```bash
npm run docs:dev
```

- 启动本地开发服务器（通常 http://localhost:5173）
- 实时热更新（保存即刷新）
- 浏览器自动刷新

### 3. 构建输出

```bash
npm run docs:build
```

- 生成优化的静态网站
- 输出到 `.vitepress/dist/`
- 包含所有必要的 HTML、CSS、JS 文件

### 4. 部署

```bash
npm run docs:preview  # 本地预览构建结果
```

部署 `.vitepress/dist` 目录到：
- GitHub Pages
- Vercel
- Netlify
- 任何静态托管服务

---

## 📁 项目结构

```
my-docs/
├── .vitepress/
│   ├── config.js              # 网站配置（必须）
│   ├── config.ts              # TypeScript 版本
│   ├── theme/
│   │   ├── index.ts           # 主题入口
│   │   ├── Layout.vue         # 自定义布局
│   │   └── custom.css         # 自定义样式
│   └── dist/                  # 构建输出（自动生成）
├── docs/
│   ├── index.md               # 首页
│   ├── guide/
│   │   ├── index.md
│   │   ├── getting-started.md
│   │   └── installation.md
│   ├── api/
│   │   ├── index.md
│   │   └── reference.md
│   └── public/
│       ├── logo.svg
│       └── favicon.ico
├── package.json
└── README.md
```

---

## ⚙️ 配置详解

### 基础配置（`.vitepress/config.js`）

```javascript
import { defineConfig } from 'vitepress'

export default defineConfig({
  // 基础信息
  title: 'My Documentation',
  description: 'A VitePress Site',
  
  // URL 配置
  base: '/',                    // 部署基础路径
  lang: 'zh-CN',               // 语言
  
  // 构建选项
  outDir: '../docs-dist',
  cacheDir: '../.vitepress_cache',
  
  // 主题配置
  themeConfig: {
    // 顶部导航
    nav: [
      { text: '首页', link: '/' },
      { text: '指南', link: '/guide/' },
      { text: 'API', link: '/api/' },
      {
        text: '更多',
        items: [
          { text: '博客', link: '/blog/' },
          { text: '更新日志', link: '/changelog/' }
        ]
      }
    ],
    
    // 侧边栏
    sidebar: {
      '/guide/': [
        {
          text: '介绍',
          collapsed: false,
          items: [
            { text: '什么是 VitePress?', link: '/guide/' },
            { text: '快速开始', link: '/guide/getting-started' }
          ]
        },
        {
          text: '进阶',
          items: [
            { text: '配置', link: '/guide/configuration' },
            { text: '扩展', link: '/guide/extending' }
          ]
        }
      ],
      '/api/': [
        { text: 'API 参考', link: '/api/' }
      ]
    },
    
    // 右侧目录
    outline: 'deep',            // 显示深度
    
    // 页脚
    footer: {
      message: 'Released under the MIT License',
      copyright: 'Copyright © 2026'
    },
    
    // 深色模式
    darkModeSwitcher: true,
    
    // 搜索
    search: {
      provider: 'local'         // 本地搜索
    }
  },
  
  // Markdown 扩展
  markdown: {
    lineNumbers: true,          // 代码行号
    theme: 'github-dark'        // 代码主题
  }
})
```

### TypeScript 类型支持

```typescript
import { defineConfig } from 'vitepress'
import type { DefaultTheme } from 'vitepress'

export default defineConfig({
  title: 'My Docs',
  themeConfig: {
    nav: [
      { text: 'Home', link: '/' }
    ] as DefaultTheme.NavItem[]
  }
})
```

---

## 📝 Markdown 特性

### 基础语法

```markdown
# 一级标题
## 二级标题
### 三级标题

**粗体** 和 *斜体*

- 列表项 1
- 列表项 2

1. 有序项 1
2. 有序项 2

[链接](https://example.com)
![图片](./image.png)

> 引用块
> 第二行
```

### VitePress 扩展语法

#### 代码块高亮

````markdown
```javascript
const greeting = 'Hello, VitePress!'
console.log(greeting)          // [!code highlight]
```

```python
def hello():
    print('Hello')             # [!code ++]
    # 旧代码                    # [!code --]
```
````

#### 容器（Callout）

```markdown
::: info
这是一个信息提示框
:::

::: warning
这是一个警告提示框
:::

::: danger
这是一个危险提示框
:::

::: details 点击展开
隐藏的内容
:::
```

#### Front Matter（页面元数据）

```yaml
---
title: 页面标题
description: 页面描述
layout: doc
lastUpdated: true
---
```

#### 使用 Vue 组件

```markdown
# 在 Markdown 中使用 Vue

<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

点击次数: {{ count }}
<button @click="count++">增加</button>
```

---

## 🎨 主题自定义

### 使用内置主题变量

```css
/* .vitepress/theme/custom.css */

:root {
  --vp-c-brand: #3b82f6;
  --vp-c-brand-dark: #2563eb;
  --vp-c-bg: #ffffff;
  --vp-c-text-1: #213547;
  --vp-c-text-2: #565656;
}
```

### 创建自定义主题

```typescript
// .vitepress/theme/index.ts
import { h } from 'vue'
import type { Theme } from 'vitepress'
import DefaultTheme from 'vitepress/theme'
import './custom.css'

export default {
  extends: DefaultTheme,
  Layout: () => {
    return h(DefaultTheme.Layout, null, {
      // 插槽定制
    })
  },
  enhanceApp({ app }) {
    // 全局组件注册
  }
} satisfies Theme
```

---

## 🔍 功能对比

### VitePress vs 其他工具

| 特性 | VitePress | Docusaurus | Hugo | Nuxt |
|------|-----------|-----------|------|------|
| **学习曲线** | ⭐⭐ 简单 | ⭐⭐⭐ 中等 | ⭐⭐⭐ 中等 | ⭐⭐⭐⭐ 陡峭 |
| **构建速度** | ⭐⭐⭐⭐⭐ 极快 | ⭐⭐⭐⭐ 快 | ⭐⭐⭐⭐⭐ 极快 | ⭐⭐⭐ 中等 |
| **Vue 集成** | ⭐⭐⭐⭐⭐ 原生 | ⭐ 无 | ⭐ 无 | ⭐⭐⭐⭐⭐ 完美 |
| **组件支持** | ⭐⭐⭐⭐⭐ 完全 | ⭐⭐⭐⭐ 完全 | ⭐⭐ 有限 | ⭐⭐⭐⭐⭐ 完全 |
| **SEO** | ⭐⭐⭐⭐ 好 | ⭐⭐⭐⭐ 好 | ⭐⭐⭐⭐⭐ 极好 | ⭐⭐⭐⭐ 好 |
| **配置复杂度** | ⭐ 最简 | ⭐⭐⭐ 中等 | ⭐⭐ 简单 | ⭐⭐⭐⭐ 复杂 |

---

## ✅ 使用场景

### 最适合使用 VitePress

✅ 项目文档（API、指南、教程）  
✅ 技术博客和知识库  
✅ 产品文档网站  
✅ 开源项目文档  
✅ 团队知识库  
✅ 需要 Vue 组件的文档  

### 不适合使用 VitePress

❌ 需要服务端渲染的应用  
❌ 高度交互的应用  
❌ 实时更新的内容（需要 API）  
❌ 大规模电商网站  
❌ CMS 驱动的站点  

---

## 🛠️ 常见任务

### 添加搜索功能

```javascript
// config.js
export default defineConfig({
  themeConfig: {
    search: {
      provider: 'local'
    }
  }
})
```

### 支持多语言

```javascript
export default defineConfig({
  locales: {
    root: {
      label: '中文',
      lang: 'zh-CN'
    },
    en: {
      label: 'English',
      lang: 'en',
      link: '/en/'
    }
  }
})
```

### 自定义导航栏

```javascript
themeConfig: {
  nav: [
    { text: '首页', link: '/' },
    { text: '文档', link: '/docs/' },
    {
      text: '链接',
      items: [
        { text: 'GitHub', link: 'https://github.com' },
        { text: '论坛', link: 'https://forum.example.com' }
      ]
    }
  ]
}
```

### 添加社交链接

```javascript
themeConfig: {
  socialLinks: [
    { icon: 'github', link: 'https://github.com/your-org' },
    { icon: 'twitter', link: 'https://twitter.com/your-handle' },
    { icon: 'discord', link: 'https://discord.gg/...' }
  ]
}
```

---

## 📦 部署示例

### GitHub Pages

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm ci
      - run: npm run docs:build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./.vitepress/dist
```

### Vercel

```json
// vercel.json
{
  "buildCommand": "npm run docs:build",
  "outputDirectory": ".vitepress/dist"
}
```

### Netlify

```toml
# netlify.toml
[build]
command = "npm run docs:build"
publish = ".vitepress/dist"
```

---

## 📚 学习资源

- [官方文档](https://vitepress.dev)
- [GitHub 仓库](https://github.com/vuejs/vitepress)
- [Vue.js 文档](https://vuejs.org)（使用 VitePress 构建）

---

## 🔗 相关笔记

- [[Vite 构建工具]]
- [[Vue 3 框架]]
- [[静态网站生成器对比]]
- [[Markdown 完整指南]]

---

**最后更新**: 2026-07-17
