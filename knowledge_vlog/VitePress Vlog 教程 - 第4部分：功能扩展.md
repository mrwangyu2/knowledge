---
title: VitePress Vlog 教程 - 第4部分：功能扩展
tags: [vitepress, vlog, 功能, 扩展]
date: 2026-07-17
---

# VitePress Vlog 从零开始搭建教程

## 第4部分：功能扩展与高级特性

### 📌 本部分目标

- 添加评论系统
- 实现访问统计
- 配置 SEO 优化
- 集成社交分享
- 实现阅读进度条

---

## 第一步：添加评论系统

### 1.1 创建 Comment 组件

创建文件：`.vitepress/theme/components/Comment.vue`

```vue
<template>
  <div class="comment-section">
    <h3>💬 评论</h3>
    
    <div class="comment-form">
      <textarea
        v-model="newComment.content"
        placeholder="分享你的想法..."
        rows="4"
      />
      <div class="form-row">
        <input
          v-model="newComment.author"
          type="text"
          placeholder="你的名字"
        />
        <input
          v-model="newComment.email"
          type="email"
          placeholder="你的邮箱（不会公开）"
        />
      </div>
      <button @click="submitComment" class="btn-submit">
        发布评论
      </button>
    </div>

    <div class="comments-list">
      <div
        v-for="comment in comments"
        :key="comment.id"
        class="comment-item"
      >
        <div class="comment-header">
          <span class="comment-author">{{ comment.author }}</span>
          <span class="comment-time">{{ formatTime(comment.time) }}</span>
        </div>
        <p class="comment-content">{{ comment.content }}</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const newComment = ref({
  author: '',
  email: '',
  content: ''
})

const comments = ref([
  {
    id: 1,
    author: '张三',
    time: '2026-07-15 10:30',
    content: '很棒的视频，学到了很多！'
  },
  {
    id: 2,
    author: '李四',
    time: '2026-07-14 15:45',
    content: '期待更多内容'
  }
])

function submitComment() {
  if (!newComment.value.author || !newComment.value.content) {
    alert('请填写名字和评论内容')
    return
  }

  comments.value.unshift({
    id: comments.value.length + 1,
    author: newComment.value.author,
    time: new Date().toLocaleString('zh-CN'),
    content: newComment.value.content
  })

  newComment.value = { author: '', email: '', content: '' }
}

function formatTime(time) {
  return time
}
</script>

<style scoped>
.comment-section {
  margin-top: 60px;
  padding-top: 40px;
  border-top: 2px solid var(--vp-c-border);
}

.comment-section h3 {
  margin-top: 0;
}

.comment-form {
  background: var(--vp-c-bg-soft);
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 30px;
}

textarea, input {
  width: 100%;
  padding: 12px;
  margin-bottom: 12px;
  border: 1px solid var(--vp-c-border);
  border-radius: 6px;
  background: var(--vp-c-bg);
  color: var(--vp-c-text-1);
  font-family: inherit;
  font-size: 1rem;
}

textarea:focus, input:focus {
  outline: none;
  border-color: var(--vp-c-brand);
  box-shadow: 0 0 0 2px var(--vp-c-brand-dimm);
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.form-row input {
  margin-bottom: 0;
}

.btn-submit {
  padding: 10px 24px;
  background: var(--vp-c-brand);
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.3s;
}

.btn-submit:hover {
  background: var(--vp-c-brand-dark);
}

.comments-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.comment-item {
  background: var(--vp-c-bg-soft);
  padding: 16px;
  border-radius: 6px;
  border-left: 3px solid var(--vp-c-brand);
}

.comment-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  font-size: 0.9rem;
}

.comment-author {
  font-weight: 600;
  color: var(--vp-c-text-1);
}

.comment-time {
  color: var(--vp-c-text-3);
}

.comment-content {
  margin: 0;
  color: var(--vp-c-text-2);
  line-height: 1.6;
}

@media (max-width: 768px) {
  .form-row {
    grid-template-columns: 1fr;
  }
}
</style>
```

---

## 第二步：添加阅读进度条

### 2.1 创建 ReadingProgress 组件

创建文件：`.vitepress/theme/components/ReadingProgress.vue`

```vue
<template>
  <div class="reading-progress">
    <div class="progress-bar" :style="{ width: progress + '%' }"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const progress = ref(0)

function calculateProgress() {
  const scrollTop = window.scrollY
  const docHeight = document.documentElement.scrollHeight - window.innerHeight
  const scrollPercent = docHeight > 0 ? (scrollTop / docHeight) * 100 : 0
  progress.value = Math.min(scrollPercent, 100)
}

onMounted(() => {
  window.addEventListener('scroll', calculateProgress)
})

onUnmounted(() => {
  window.removeEventListener('scroll', calculateProgress)
})
</script>

<style scoped>
.reading-progress {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 3px;
  background: transparent;
  z-index: 999;
  pointer-events: none;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(
    90deg,
    var(--vp-c-brand) 0%,
    var(--vp-c-brand-light) 100%
  );
  transition: width 0.1s ease;
}
</style>
```

---

## 第三步：SEO 优化

### 3.1 更新 config.ts 中的 head 配置

```typescript
export default defineConfig({
  // ... 其他配置

  head: [
    // 基础 meta 标签
    ['meta', { charset: 'utf-8' }],
    ['meta', { name: 'viewport', content: 'width=device-width, initial-scale=1' }],
    ['meta', { name: 'description', content: '分享生活、技术和创意的视频博客' }],
    ['meta', { name: 'keywords', content: 'vlog, 视频博客, 教程, 技术分享' }],
    ['meta', { name: 'author', content: '你的名字' }],

    // Open Graph
    ['meta', { property: 'og:type', content: 'website' }],
    ['meta', { property: 'og:title', content: '我的 Vlog' }],
    ['meta', { property: 'og:description', content: '分享生活、技术和创意的视频博客' }],
    ['meta', { property: 'og:image', content: '/images/og-image.jpg' }],

    // Twitter Card
    ['meta', { name: 'twitter:card', content: 'summary_large_image' }],
    ['meta', { name: 'twitter:title', content: '我的 Vlog' }],
    ['meta', { name: 'twitter:description', content: '分享生活、技术和创意的视频博客' }],

    // Favicon
    ['link', { rel: 'icon', href: '/favicon.ico' }],
    ['link', { rel: 'apple-touch-icon', href: '/images/apple-touch-icon.png' }],

    // Preload 关键资源
    ['link', { rel: 'preload', as: 'font', href: '/fonts/main.woff2', crossorigin: true }],
  ]
})
```

### 3.2 在页面 frontmatter 中添加 SEO 信息

```markdown
---
title: 视频标题
description: 视频描述信息
image: /images/vlog-thumbnail.jpg
tags: [标签1, 标签2]
author: 你的名字
date: 2026-07-17
---
```

---

## 第四步：社交分享功能

### 4.1 创建 ShareButtons 组件

创建文件：`.vitepress/theme/components/ShareButtons.vue`

```vue
<template>
  <div class="share-buttons">
    <span class="share-label">分享：</span>
    <a
      href="#"
      @click.prevent="shareOn('twitter')"
      class="share-btn twitter"
      title="分享到 Twitter"
    >
      𝕏
    </a>
    <a
      href="#"
      @click.prevent="shareOn('facebook')"
      class="share-btn facebook"
      title="分享到 Facebook"
    >
      f
    </a>
    <a
      href="#"
      @click.prevent="shareOn('linkedin')"
      class="share-btn linkedin"
      title="分享到 LinkedIn"
    >
      in
    </a>
    <a
      href="#"
      @click.prevent="shareOn('copy')"
      class="share-btn copy"
      title="复制链接"
    >
      🔗
    </a>
  </div>
</template>

<script setup>
const props = defineProps({
  title: String,
  url: String
})

function shareOn(platform) {
  const url = props.url || window.location.href
  const title = props.title || document.title
  let shareUrl = ''

  switch (platform) {
    case 'twitter':
      shareUrl = `https://twitter.com/intent/tweet?url=${encodeURIComponent(url)}&text=${encodeURIComponent(title)}`
      break
    case 'facebook':
      shareUrl = `https://www.facebook.com/sharer/sharer.php?u=${encodeURIComponent(url)}`
      break
    case 'linkedin':
      shareUrl = `https://www.linkedin.com/sharing/share-offsite/?url=${encodeURIComponent(url)}`
      break
    case 'copy':
      navigator.clipboard.writeText(url)
      alert('链接已复制到剪贴板')
      return
  }

  window.open(shareUrl, '_blank')
}
</script>

<style scoped>
.share-buttons {
  display: flex;
  align-items: center;
  gap: 12px;
  margin: 30px 0;
  padding: 16px;
  background: var(--vp-c-bg-soft);
  border-radius: 8px;
}

.share-label {
  font-weight: 600;
  color: var(--vp-c-text-1);
}

.share-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: var(--vp-c-bg);
  border: 2px solid var(--vp-c-border);
  color: var(--vp-c-text-2);
  font-weight: bold;
  transition: all 0.3s;
}

.share-btn:hover {
  transform: scale(1.1);
}

.share-btn.twitter:hover {
  background: #000;
  border-color: #000;
  color: white;
}

.share-btn.facebook:hover {
  background: #1877f2;
  border-color: #1877f2;
  color: white;
}

.share-btn.linkedin:hover {
  background: #0a66c2;
  border-color: #0a66c2;
  color: white;
}

.share-btn.copy:hover {
  background: var(--vp-c-brand);
  border-color: var(--vp-c-brand);
  color: white;
}
</style>
```

---

## 第五步：访问统计集成

### 5.1 添加 Google Analytics

在 config.ts 的 head 中添加：

```typescript
head: [
  // ... 其他 head 配置
  
  // Google Analytics
  ['script', { async: true, src: 'https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX' }],
  ['script', {}, `
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', 'G-XXXXXXXXXX');
  `]
]
```

### 5.2 使用 Vercel Analytics

在 `.vitepress/theme/index.ts` 中：

```typescript
import { h } from 'vue'
import type { Theme } from 'vitepress'
import DefaultTheme from 'vitepress/theme'
import './styles/custom.css'

import { injectSpeedInsights } from '@vercel/speed-insights/vue'

export default {
  extends: DefaultTheme,
  Layout: () => h(DefaultTheme.Layout),
  enhanceApp({ app }) {
    injectSpeedInsights(app)
  }
} satisfies Theme
```

---

## 第六步：性能优化

### 6.1 图片懒加载

在 Markdown 中使用：

```markdown
![描述](image.jpg?loading=lazy)
```

### 6.2 代码分割和预加载

在 config.ts 中配置：

```typescript
export default defineConfig({
  vite: {
    build: {
      rollupOptions: {
        output: {
          manualChunks: {
            vue: ['vue'],
            vitepress: ['vitepress']
          }
        }
      }
    }
  }
})
```

---

## 📋 功能检查清单

完成此部分后，检查：

- ✅ 评论系统可以添加和显示评论
- ✅ 阅读进度条实时更新
- ✅ SEO meta 标签正确生成
- ✅ 社交分享按钮正常工作
- ✅ 访问统计正确跟踪
- ✅ 页面加载性能良好

---

## 🔗 下一步

→ **[[VitePress Vlog 教程 - 第5部分：部署上线]]**

---

最后更新：2026-07-17
