---
title: VitePress Vlog 快速参考
tags: [vitepress, 快速参考, 命令, 代码片段]
date: 2026-07-17
---

# VitePress Vlog 快速参考手册

这是一份快速查询和复制粘贴的参考资料，包含常用命令、配置和代码片段。

---

## 🔧 常用命令速查

### 项目管理

```bash
# 初始化新项目
npm init vitepress@latest .

# 安装依赖
npm install

# 本地开发
npm run docs:dev

# 构建项目
npm run docs:build

# 预览构建结果
npm run docs:preview

# 更新依赖
npm update
```

### Git 操作

```bash
# 初始化仓库
git init
git remote add origin https://github.com/your-username/repo.git

# 查看状态
git status

# 提交更改
git add .
git commit -m "message"

# 推送
git push origin main

# 创建分支
git checkout -b feature/name
```

### 文件操作

```bash
# 查看文件大小
du -sh .vitepress/dist/

# 清理缓存
rm -rf node_modules package-lock.json
rm -rf .vitepress/cache .vitepress/dist

# 查找文件
find . -name "*.md" -type f

# 搜索内容
grep -r "keyword" docs/
```

---

## ⚙️ 关键配置片段

### 最小化 config.ts

```typescript
import { defineConfig } from 'vitepress'

export default defineConfig({
  title: '我的 Vlog',
  description: '视频博客网站',
  themeConfig: {
    nav: [
      { text: '首页', link: '/' },
      { text: 'Vlog', link: '/vlog/' }
    ],
    socialLinks: [
      { icon: 'github', link: 'https://github.com' }
    ]
  }
})
```

### 完整 config.ts 模板

```typescript
import { defineConfig } from 'vitepress'

export default defineConfig({
  title: '我的 Vlog',
  description: '分享生活、技术和创意的视频博客',
  lang: 'zh-CN',
  base: '/',
  
  head: [
    ['meta', { charset: 'utf-8' }],
    ['meta', { name: 'viewport', content: 'width=device-width, initial-scale=1' }],
    ['link', { rel: 'icon', href: '/favicon.ico' }]
  ],
  
  themeConfig: {
    nav: [
      { text: '首页', link: '/' },
      { text: 'Vlog', link: '/vlog/' },
      { text: '博客', link: '/blog/' },
      { text: '关于', link: '/about/' }
    ],
    
    sidebar: {
      '/vlog/': [
        {
          text: 'Vlog 集合',
          items: [
            { text: '所有视频', link: '/vlog/' }
          ]
        }
      ]
    },
    
    socialLinks: [
      { icon: 'github', link: 'https://github.com' },
      { icon: 'twitter', link: 'https://twitter.com' }
    ],
    
    footer: {
      message: '分享知识，传播快乐',
      copyright: 'Copyright © 2026 My Vlog'
    },
    
    search: {
      provider: 'local'
    }
  },
  
  markdown: {
    lineNumbers: true,
    theme: 'github-dark'
  }
})
```

---

## 📝 Markdown 快速语法

### 基础

```markdown
# 一级标题
## 二级标题
### 三级标题

**粗体** *斜体* ~~删除线~~

- 列表项 1
- 列表项 2

1. 有序项 1
2. 有序项 2

[链接](url)
![图片](image.png)

> 引用
> 第二行
```

### VitePress 扩展

```markdown
<!-- 信息框 -->
::: info
信息
:::

::: warning
警告
:::

::: danger
危险
:::

::: details 标题
内容
:::

<!-- 代码高亮 -->
```javascript [title]
const code = 'here'  // [!code highlight]
const old = 'line'   // [!code --]
const new = 'line'   // [!code ++]
```

<!-- Front Matter -->
---
title: 页面标题
description: 页面描述
---
```

---

## 🎨 样式代码片段

### CSS 变量

```css
:root {
  --vp-c-brand: #6366f1;
  --vp-c-brand-dark: #4f46e5;
  --vp-c-text-1: #213547;
  --vp-c-bg: #ffffff;
  --vp-c-border: #e5e7eb;
}

html.dark {
  --vp-c-text-1: #f5f5f5;
  --vp-c-bg: #1a1a1a;
  --vp-c-border: #444444;
}
```

### 常用类名

```css
/* 按钮 */
.btn {
  padding: 10px 20px;
  border-radius: 6px;
  cursor: pointer;
}

.btn-primary {
  background: var(--vp-c-brand);
  color: white;
}

/* 卡片 */
.card {
  background: var(--vp-c-bg-soft);
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}

/* 网格 */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 24px;
}

/* 响应式 */
@media (max-width: 768px) {
  .grid {
    grid-template-columns: 1fr;
  }
}
```

---

## 🎯 Vue 组件模板

### 基础组件

```vue
<template>
  <div class="my-component">
    <h2>{{ title }}</h2>
    <button @click="handleClick">点击</button>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface Props {
  title: string
}

withDefaults(defineProps<Props>(), {
  title: '默认标题'
})

const count = ref(0)

function handleClick() {
  count.value++
}
</script>

<style scoped>
.my-component {
  padding: 20px;
}
</style>
```

### 列表组件

```vue
<template>
  <div class="item-list">
    <div
      v-for="item in items"
      :key="item.id"
      class="item"
    >
      {{ item.name }}
    </div>
  </div>
</template>

<script setup lang="ts">
interface Item {
  id: number
  name: string
}

defineProps<{
  items: Item[]
}>()
</script>

<style scoped>
.item-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.item {
  padding: 12px;
  background: var(--vp-c-bg-soft);
  border-radius: 4px;
}
</style>
```

---

## 📦 npm 包速查

### 常用包

```bash
# 开发工具
npm install -D typescript ts-node

# 工具库
npm install axios dayjs
npm install @vueuse/core

# 测试
npm install -D vitest @testing-library/vue

# 构建
npm install -D vite vitepress
```

---

## 🚀 部署相关

### GitHub Actions 模板

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci && npm run docs:build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./.vitepress/dist
```

### vercel.json

```json
{
  "buildCommand": "npm run docs:build",
  "outputDirectory": ".vitepress/dist"
}
```

### netlify.toml

```toml
[build]
command = "npm run docs:build"
publish = ".vitepress/dist"
```

---

## 🔍 SEO 配置

### Head 标签

```typescript
head: [
  ['meta', { charset: 'utf-8' }],
  ['meta', { name: 'viewport', content: 'width=device-width, initial-scale=1' }],
  ['meta', { name: 'description', content: '描述' }],
  ['meta', { property: 'og:title', content: '标题' }],
  ['meta', { property: 'og:image', content: '/image.jpg' }],
  ['link', { rel: 'icon', href: '/favicon.ico' }],
  ['link', { rel: 'canonical', href: 'https://yourdomain.com' }]
]
```

### Robots.txt

```
User-agent: *
Allow: /
Disallow: /admin

Sitemap: https://yourdomain.com/sitemap.xml
```

---

## 🛠️ 调试技巧

### 常见错误解决

```bash
# 端口被占用
lsof -i :5173        # 查看进程
kill -9 <PID>        # 杀死进程

# 清理缓存
rm -rf node_modules package-lock.json
npm install

# 检查配置
npm run docs:build -- --debug

# 查看版本
npm list vitepress
```

### 性能优化

```bash
# 分析包大小
npm run build -- --analyze

# 生成报告
npm run build -- --report

# 检查性能
npm install -D lighthouse
npx lighthouse https://yourdomain.com
```

---

## 📋 内容结构模板

### Vlog 页面

```markdown
---
title: 视频标题
description: 视频描述
tags: [标签1, 标签2]
date: 2026-07-17
videoUrl: https://example.com/video.mp4
thumbnail: /images/thumb.jpg
duration: 12:34
---

# 视频标题

<VideoPlayer :url="videoUrl" />

## 📝 视频摘要

简介...

## 📚 相关资源

- [资源1](link)

<Comment />
```

### 博客页面

```markdown
---
title: 文章标题
description: 文章描述
author: 作者
date: 2026-07-17
tags: [标签]
---

# 文章标题

内容...

<ShareButtons :title="title" />

<Comment />
```

---

## 🔗 重要链接速查

| 资源 | URL |
|------|-----|
| VitePress 官网 | https://vitepress.dev |
| Vue 3 官网 | https://vuejs.org |
| GitHub Pages | https://pages.github.com |
| Vercel | https://vercel.com |
| Netlify | https://netlify.com |

---

## 📞 快速排查清单

部署失败？检查以下项目：

- [ ] Node.js 版本 >= 18
- [ ] npm 依赖已安装
- [ ] config.ts 语法正确
- [ ] Markdown 文件路径正确
- [ ] 没有 TypeScript 错误
- [ ] 构建命令成功执行
- [ ] .vitepress/dist 目录存在
- [ ] 部署平台配置正确

---

## 💡 快速提示

- 使用 `npm run docs:dev` 实时开发
- 在 config.ts 中编辑导航和侧边栏
- 在 markdown 中使用 Vue 组件
- 在 theme/styles/custom.css 中自定义样式
- 定期 `npm update` 更新依赖
- 推荐使用 GitHub + Vercel 的组合

---

**收藏本页以快速查询常用命令和代码片段！**

最后更新：2026-07-17
