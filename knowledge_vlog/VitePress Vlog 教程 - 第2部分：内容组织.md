---
title: VitePress Vlog 教程 - 第2部分：内容组织
tags: [vitepress, vlog, 教程, 内容管理]
date: 2026-07-17
---

# VitePress Vlog 从零开始搭建教程

## 第2部分：内容组织与视频展示

### 📌 本部分目标

- 创建视频 Vlog 内容结构
- 实现视频卡片和展示组件
- 建立分类和标签系统
- 创建视频索引页面

---

## 第一步：创建视频组件

### 1.1 创建 VideoCard 组件

创建文件：`.vitepress/theme/components/VideoCard.vue`

```vue
<template>
  <div class="video-card">
    <div class="video-thumbnail">
      <img :src="thumbnail" :alt="title" />
      <div class="duration">{{ duration }}</div>
      <div class="play-button">▶</div>
    </div>
    <div class="video-info">
      <h3 class="video-title">{{ title }}</h3>
      <p class="video-date">{{ formatDate(date) }}</p>
      <p class="video-description">{{ description }}</p>
      <div class="video-tags">
        <span 
          v-for="tag in tags" 
          :key="tag"
          class="tag"
        >
          #{{ tag }}
        </span>
      </div>
      <a :href="videoLink" class="read-more">
        查看详情 →
      </a>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Props {
  title: string
  description: string
  thumbnail: string
  date: string
  duration: string
  tags: string[]
  videoLink: string
}

withDefaults(defineProps<Props>(), {})

function formatDate(date: string): string {
  const d = new Date(date)
  return d.toLocaleDateString('zh-CN', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.video-card {
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s, box-shadow 0.3s;
  background: var(--vp-c-bg-soft);
}

.video-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.15);
}

.video-thumbnail {
  position: relative;
  width: 100%;
  padding-bottom: 56.25%;
  overflow: hidden;
  background: #000;
}

.video-thumbnail img {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.duration {
  position: absolute;
  bottom: 8px;
  right: 8px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: bold;
}

.play-button {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 60px;
  height: 60px;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
  opacity: 0;
  transition: opacity 0.3s;
}

.video-card:hover .play-button {
  opacity: 1;
}

.video-info {
  padding: 20px;
}

.video-title {
  margin: 0 0 8px 0;
  font-size: 18px;
  font-weight: 600;
  color: var(--vp-c-text-1);
}

.video-date {
  margin: 0 0 12px 0;
  font-size: 14px;
  color: var(--vp-c-text-3);
}

.video-description {
  margin: 0 0 12px 0;
  font-size: 14px;
  color: var(--vp-c-text-2);
  line-height: 1.6;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.video-tags {
  margin: 12px 0;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.tag {
  display: inline-block;
  background: var(--vp-c-bg-alt);
  color: var(--vp-c-brand);
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
}

.read-more {
  display: inline-block;
  margin-top: 12px;
  color: var(--vp-c-brand);
  text-decoration: none;
  font-weight: 500;
  transition: color 0.3s;
}

.read-more:hover {
  color: var(--vp-c-brand-dark);
}
</style>
```

### 1.2 创建 VideoGallery 网格组件

创建文件：`.vitepress/theme/components/VideoGallery.vue`

```vue
<template>
  <div class="video-gallery">
    <div class="gallery-grid">
      <VideoCard
        v-for="video in videos"
        :key="video.id"
        :title="video.title"
        :description="video.description"
        :thumbnail="video.thumbnail"
        :date="video.date"
        :duration="video.duration"
        :tags="video.tags"
        :video-link="video.link"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import VideoCard from './VideoCard.vue'

interface Video {
  id: string
  title: string
  description: string
  thumbnail: string
  date: string
  duration: string
  tags: string[]
  link: string
}

interface Props {
  videos: Video[]
}

withDefaults(defineProps<Props>(), {})
</script>

<style scoped>
.video-gallery {
  width: 100%;
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 24px;
  margin: 24px 0;
}

@media (max-width: 768px) {
  .gallery-grid {
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 16px;
  }
}
</style>
```

---

## 第二步：创建 Vlog 页面

### 2.1 编辑 `docs/vlog/index.md`

```markdown
---
title: Vlog 集合
layout: page
---

# 📹 我的 Vlog 集合

欢迎来到我的视频博客库。在这里分享生活点滴、技术干货和创意内容。

## 🎯 按分类浏览

::: tabs
== 全部视频
<VideoGallery :videos="allVideos" />

== 技术教程
<VideoGallery :videos="techVideos" />

== 生活分享
<VideoGallery :videos="lifeVideos" />

== 创意作品
<VideoGallery :videos="creativVideos" />
:::

<script setup>
const allVideos = [
  {
    id: '1',
    title: '如何用 VitePress 搭建个人博客',
    description: '从零开始，手把手教你搭建一个专业的个人博客网站。',
    thumbnail: '/images/vlog-1.jpg',
    date: '2026-07-15',
    duration: '12:34',
    tags: ['教程', 'VitePress', 'Web开发'],
    link: '/vlog/2026-07-15-vitepress-tutorial.md'
  },
  {
    id: '2',
    title: '我的远程工作日常',
    description: '分享在家工作的真实经历和最佳实践。',
    thumbnail: '/images/vlog-2.jpg',
    date: '2026-07-12',
    duration: '8:45',
    tags: ['生活', '工作', '日常'],
    link: '/vlog/2026-07-12-remote-work.md'
  },
  {
    id: '3',
    title: '我用过最棒的开发工具',
    description: '推荐 10 个提升开发效率的必用工具。',
    thumbnail: '/images/vlog-3.jpg',
    date: '2026-07-08',
    duration: '15:20',
    tags: ['技术', '工具', '生产力'],
    link: '/vlog/2026-07-08-dev-tools.md'
  }
]

const techVideos = allVideos.filter(v => v.tags.includes('教程'))
const lifeVideos = allVideos.filter(v => v.tags.includes('生活'))
const creativVideos = allVideos.filter(v => v.tags.includes('创意'))
</script>
```

### 2.2 创建单个视频页面

创建文件：`docs/vlog/2026-07-15-vitepress-tutorial.md`

```markdown
---
title: 如何用 VitePress 搭建个人博客
description: 从零开始，手把手教你搭建一个专业的个人博客网站。
tags: [教程, VitePress, Web开发]
date: 2026-07-15
duration: 12:34
videoUrl: https://example.com/videos/vitepress-tutorial.mp4
thumbnail: /images/vlog-1.jpg
---

# 如何用 VitePress 搭建个人博客

<VideoPlayer :url="videoUrl" />

## 📝 视频摘要

本视频将教你如何从零开始搭建一个专业的个人博客。

### 主要内容

- ✅ 环境设置和项目初始化
- ✅ 配置和自定义主题
- ✅ 内容组织和部署
- ✅ SEO 优化技巧

## 📚 相关资源

- [VitePress 官方文档](https://vitepress.dev)
- [视频源代码](https://github.com/example/vitepress-blog)
- [完整项目演示](https://example.com)

## 💬 评论

<Comment />

<script setup>
const videoUrl = 'https://example.com/videos/vitepress-tutorial.mp4'
</script>
```

---

## 第三步：创建视频播放器组件

### 3.1 创建 VideoPlayer 组件

创建文件：`.vitepress/theme/components/VideoPlayer.vue`

```vue
<template>
  <div class="video-player-container">
    <video
      ref="videoElement"
      class="video-player"
      controls
      :poster="poster"
    >
      <source :src="url" type="video/mp4" />
      您的浏览器不支持 HTML5 视频播放
    </video>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

interface Props {
  url: string
  poster?: string
  autoplay?: boolean
}

withDefaults(defineProps<Props>(), {
  autoplay: false
})

const videoElement = ref<HTMLVideoElement>()
</script>

<style scoped>
.video-player-container {
  width: 100%;
  margin: 24px 0;
  border-radius: 8px;
  overflow: hidden;
  background: #000;
}

.video-player {
  width: 100%;
  height: auto;
  display: block;
  max-width: 100%;
  aspect-ratio: 16 / 9;
}
</style>
```

---

## 第四步：注册全局组件

### 4.1 编辑 `.vitepress/theme/index.ts`

```typescript
import { h } from 'vue'
import type { Theme } from 'vitepress'
import DefaultTheme from 'vitepress/theme'
import './styles/custom.css'

// 导入自定义组件
import VideoCard from './components/VideoCard.vue'
import VideoGallery from './components/VideoGallery.vue'
import VideoPlayer from './components/VideoPlayer.vue'

export default {
  extends: DefaultTheme,
  Layout: () => {
    return h(DefaultTheme.Layout, null, {})
  },
  enhanceApp({ app }) {
    // 注册全局组件
    app.component('VideoCard', VideoCard)
    app.component('VideoGallery', VideoGallery)
    app.component('VideoPlayer', VideoPlayer)
  }
} satisfies Theme
```

---

## 第五步：创建搜索和分类页面

### 5.1 创建 `docs/vlog/categories.md`

```markdown
---
title: Vlog 分类浏览
---

# 📂 按分类浏览

## 技术教程

展示所有技术相关的视频内容...

## 生活分享

展示所有生活相关的视频内容...

## 创意作品

展示所有创意相关的视频内容...
```

---

## 📋 测试清单

- ✅ 视频组件正常显示
- ✅ 视频卡片有响应式布局
- ✅ 播放器可以正常播放视频
- ✅ 标签和分类功能正常工作
- ✅ 页面在移动端显示良好

---

## 🔗 下一步

→ **[[VitePress Vlog 教程 - 第3部分：主题定制]]**

---

最后更新：2026-07-17
