---
title: VitePress Vlog 教程 - 第1部分：项目初始化
tags: [vitepress, vlog, 教程, 视频博客]
date: 2026-07-17
status: 完成
---

# VitePress Vlog 从零开始搭建教程

## 第1部分：项目初始化与基础设置

### 📌 本教程目标

通过 VitePress 搭建一个现代化的 Vlog（视频博客）网站，包含：
- 📹 视频展示和播放
- 📝 文章和笔记
- 🏷️ 分类和标签系统
- 🔍 全文搜索
- 📊 访问统计
- 💬 评论系统
- 🌙 深色/浅色主题

### 🔧 环境要求

```bash
- Node.js 18.0 或以上
- npm 或 yarn（包管理工具）
- Git（版本控制）
- 代码编辑器（VS Code 推荐）
```

检查环境：
```bash
node --version    # v18.0.0 或更高
npm --version     # 9.0.0 或更高
git --version     # 2.30.0 或更高
```

---

## 第一步：创建项目

### 1.1 初始化项目

选择工作目录，运行初始化命令：

```bash
# 创建项目目录
mkdir my-vlog
cd my-vlog

# 使用 npm 初始化 VitePress
npm init vitepress@latest .
```

### 1.2 交互式配置

按照提示进行选择：

```
✔ Project name: my-vlog
✔ Project root: .
✔ Theme: Use default theme
✔ TypeScript: Yes
✔ Add VitePress npm scripts: Yes
✔ Add Prettier for code formatting: Yes
```

### 1.3 安装依赖

```bash
npm install
```

### 1.4 验证安装

```bash
npm run docs:dev
```

打开浏览器访问 `http://localhost:5173`，看到欢迎页面表示安装成功。

---

## 第二步：项目结构规划

### 2.1 创建完整的项目目录结构

```bash
mkdir -p docs/{guide,vlog,blog,resources,about,public/{images,videos}}
```

完整的项目结构：

```
my-vlog/
├── .vitepress/
│   ├── config.ts              # 网站配置
│   ├── theme/
│   │   ├── index.ts           # 主题入口
│   │   ├── Layout.vue         # 自定义布局
│   │   ├── components/        # 自定义组件
│   │   │   ├── VideoCard.vue
│   │   │   ├── VideoGallery.vue
│   │   │   └── Comment.vue
│   │   └── styles/
│   │       └── custom.css
│   └── dist/                  # 构建输出（自动生成）
│
├── docs/
│   ├── index.md               # 首页
│   ├── vlog/                  # Vlog 视频集合
│   │   ├── index.md
│   │   ├── 2026-07-01-first-video.md
│   │   └── 2026-07-15-my-story.md
│   ├── blog/                  # 文字博客
│   │   ├── index.md
│   │   └── how-to-build-vlog.md
│   ├── guide/                 # 指南文档
│   │   ├── index.md
│   │   └── getting-started.md
│   ├── resources/             # 资源合集
│   │   ├── index.md
│   │   └── tools.md
│   ├── about/                 # 关于页面
│   │   └── index.md
│   ├── public/                # 静态资源
│   │   ├── images/
│   │   │   ├── logo.png
│   │   │   ├── banner.jpg
│   │   │   └── avatar.jpg
│   │   └── videos/
│   │       ├── video-1.mp4
│   │       └── video-2.webm
│   └── .vitepress/            # VitePress 配置（同上）
│
├── package.json
├── tsconfig.json
└── README.md
```

### 2.2 创建基础目录和文件

```bash
# 创建目录
mkdir -p docs/vlog docs/blog docs/guide docs/resources docs/about
mkdir -p docs/public/images docs/public/videos
mkdir -p .vitepress/theme/components .vitepress/theme/styles

# 创建初始文件
touch docs/vlog/index.md
touch docs/blog/index.md
touch docs/guide/index.md
touch docs/resources/index.md
touch docs/about/index.md
```

---

## 第三步：配置 VitePress

### 3.1 编辑 `.vitepress/config.ts`

```typescript
import { defineConfig } from 'vitepress'

export default defineConfig({
  // ===== 基础配置 =====
  title: '我的 Vlog',
  description: '分享生活、技术和创意的视频博客',
  lang: 'zh-CN',
  base: '/',
  
  // ===== 构建配置 =====
  outDir: '../dist',
  cacheDir: '../.vitepress_cache',
  
  // ===== 主题配置 =====
  themeConfig: {
    // 导航栏
    nav: [
      { text: '首页', link: '/' },
      { text: 'Vlog', link: '/vlog/' },
      { text: '博客', link: '/blog/' },
      { text: '指南', link: '/guide/' },
      { text: '关于', link: '/about/' }
    ],
    
    // 侧边栏
    sidebar: {
      '/vlog/': [
        {
          text: 'Vlog 集合',
          items: [
            { text: '所有视频', link: '/vlog/' },
            { text: '最新上传', link: '/vlog/latest' },
            { text: '分类浏览', link: '/vlog/categories' }
          ]
        }
      ],
      '/blog/': [
        {
          text: '博客文章',
          items: [
            { text: '所有文章', link: '/blog/' },
            { text: '技术分享', link: '/blog/tech' },
            { text: '生活思考', link: '/blog/life' }
          ]
        }
      ],
      '/guide/': [
        {
          text: '使用指南',
          items: [
            { text: '入门指南', link: '/guide/' },
            { text: '常见问题', link: '/guide/faq' }
          ]
        }
      ]
    },
    
    // 社交链接
    socialLinks: [
      { icon: 'github', link: 'https://github.com' },
      { icon: 'twitter', link: 'https://twitter.com' },
      { icon: 'youtube', link: 'https://youtube.com' }
    ],
    
    // 页脚
    footer: {
      message: '分享知识，传播快乐',
      copyright: 'Copyright © 2026 My Vlog. All rights reserved.'
    },
    
    // 右侧大纲
    outline: {
      level: 'deep',
      label: '目录'
    }
  },
  
  // ===== Markdown 扩展 =====
  markdown: {
    lineNumbers: true,
    theme: 'github-dark',
    config: (md) => {
      // 可在此添加自定义 markdown 插件
    }
  }
})
```

---

## 第四步：创建首页

### 4.1 编辑 `docs/index.md`

```markdown
---
layout: home

hero:
  name: "我的 Vlog"
  text: "分享生活、技术和创意"
  tagline: "欢迎来到我的视频博客世界
  actions:
    - theme: brand
      text: 开始观看
      link: /vlog/
    - theme: alt
      text: 了解更多
      link: /about/
  image:
    src: /images/banner.jpg
    alt: Banner

features:
  - icon: 🎬
    title: 最新视频
    details: 每周更新的精彩视频内容
  - icon: 📚
    title: 深度文章
    details: 配套的详细文字说明和思考
  - icon: 🔍
    title: 智能搜索
    details: 快速找到你想要的内容
  - icon: 💡
    title: 干货分享
    details: 技术、生活、创意的完整分享
---
```

---

## 第五步：运行开发服务器

### 5.1 启动开发环境

```bash
npm run docs:dev
```

看到输出：
```
VITE v4.x.x  ready in xxx ms

➜  Local:   http://localhost:5173/
➜  press any key to stop
```

### 5.2 测试页面

- 访问 `http://localhost:5173/`
- 看到自定义的首页
- 点击导航栏链接测试

---

## 📋 检查清单

完成此部分后，检查以下项目：

- ✅ Node.js 环境已安装
- ✅ 项目通过 `npm init vitepress` 初始化
- ✅ 依赖已安装 (`npm install`)
- ✅ 项目目录结构已创建
- ✅ `.vitepress/config.ts` 已配置
- ✅ `docs/index.md` 首页已创建
- ✅ 开发服务器可正常启动和访问

---

## 🔗 下一步

→ **[[VitePress Vlog 教程 - 第2部分：内容组织]]**

---

最后更新：2026-07-17
