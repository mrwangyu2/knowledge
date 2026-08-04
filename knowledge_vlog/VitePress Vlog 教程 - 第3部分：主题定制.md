---
title: VitePress Vlog 教程 - 第3部分：主题定制
tags: [vitepress, vlog, 主题, 自定义]
date: 2026-07-17
---

# VitePress Vlog 从零开始搭建教程

## 第3部分：主题定制与样式优化

### 📌 本部分目标

- 自定义网站主题配色
- 创建专属的 Vlog 风格
- 优化视觉层次和排版
- 实现亮/暗色主题切换

---

## 第一步：创建自定义样式

### 1.1 编辑 `.vitepress/theme/styles/custom.css`

```css
/* ===== 颜色变量定义 ===== */
:root {
  /* 品牌色 */
  --vp-c-brand: #6366f1;
  --vp-c-brand-light: #818cf8;
  --vp-c-brand-dark: #4f46e5;
  --vp-c-brand-dimm: rgba(99, 102, 241, 0.05);

  /* 文本颜色 */
  --vp-c-text-1: #213547;
  --vp-c-text-2: #565656;
  --vp-c-text-3: #a9a9a9;

  /* 背景颜色 */
  --vp-c-bg: #ffffff;
  --vp-c-bg-soft: #f6f6f7;
  --vp-c-bg-mute: #f1f1f3;

  /* 边框颜色 */
  --vp-c-border: #e5e7eb;
  --vp-c-border-light: #f3f4f6;

  /* 其他颜色 */
  --vp-c-success: #10b981;
  --vp-c-warning: #f59e0b;
  --vp-c-danger: #ef4444;
}

/* 深色模式 */
html.dark {
  --vp-c-text-1: #f5f5f5;
  --vp-c-text-2: #d9d9d9;
  --vp-c-text-3: #999999;

  --vp-c-bg: #1a1a1a;
  --vp-c-bg-soft: #2a2a2a;
  --vp-c-bg-mute: #333333;

  --vp-c-border: #444444;
  --vp-c-border-light: #333333;
}

/* ===== 全局样式 ===== */

* {
  box-sizing: border-box;
}

html {
  scroll-behavior: smooth;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Helvetica Neue', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ===== 排版 ===== */

h1 {
  font-size: 2.5rem;
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1.2;
}

h2 {
  font-size: 2rem;
  font-weight: 600;
  letter-spacing: -0.01em;
  line-height: 1.3;
  margin-top: 48px;
  margin-bottom: 24px;
  padding-bottom: 12px;
  border-bottom: 1px solid var(--vp-c-border);
}

h3 {
  font-size: 1.5rem;
  font-weight: 600;
  line-height: 1.4;
  margin-top: 32px;
  margin-bottom: 16px;
}

h4 {
  font-size: 1.25rem;
  font-weight: 600;
  margin-top: 24px;
  margin-bottom: 12px;
}

p {
  line-height: 1.8;
  margin: 16px 0;
  color: var(--vp-c-text-2);
}

a {
  color: var(--vp-c-brand);
  text-decoration: none;
  transition: color 0.3s ease;
}

a:hover {
  color: var(--vp-c-brand-dark);
  text-decoration: underline;
}

code {
  background-color: var(--vp-c-bg-mute);
  color: var(--vp-c-brand);
  padding: 2px 6px;
  border-radius: 3px;
  font-family: 'Monaco', 'Courier New', monospace;
  font-size: 0.9em;
}

pre {
  background-color: var(--vp-c-bg-soft);
  border: 1px solid var(--vp-c-border);
  border-radius: 8px;
  padding: 16px;
  margin: 24px 0;
  overflow-x: auto;
}

/* ===== 列表样式 ===== */

ul, ol {
  margin: 16px 0;
  padding-left: 24px;
}

li {
  margin: 8px 0;
  line-height: 1.8;
}

/* ===== 引用块 ===== */

blockquote {
  border-left: 4px solid var(--vp-c-brand);
  padding: 12px 16px;
  background-color: var(--vp-c-brand-dimm);
  border-radius: 4px;
  margin: 24px 0;
  color: var(--vp-c-text-2);
}

/* ===== 表格 ===== */

table {
  width: 100%;
  border-collapse: collapse;
  margin: 24px 0;
}

thead {
  background-color: var(--vp-c-bg-soft);
}

th, td {
  padding: 12px 16px;
  text-align: left;
  border-bottom: 1px solid var(--vp-c-border);
}

tr:hover {
  background-color: var(--vp-c-bg-mute);
}

/* ===== 按钮 ===== */

.btn {
  display: inline-block;
  padding: 10px 20px;
  border-radius: 6px;
  font-weight: 500;
  text-decoration: none;
  cursor: pointer;
  transition: all 0.3s ease;
  border: none;
  font-size: 1rem;
}

.btn-primary {
  background-color: var(--vp-c-brand);
  color: white;
}

.btn-primary:hover {
  background-color: var(--vp-c-brand-dark);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(99, 102, 241, 0.3);
}

.btn-secondary {
  background-color: var(--vp-c-bg-soft);
  color: var(--vp-c-text-1);
  border: 1px solid var(--vp-c-border);
}

.btn-secondary:hover {
  background-color: var(--vp-c-bg-mute);
  border-color: var(--vp-c-brand);
}

/* ===== 卡片 ===== */

.card {
  background: var(--vp-c-bg-soft);
  border: 1px solid var(--vp-c-border);
  border-radius: 8px;
  padding: 20px;
  transition: all 0.3s ease;
}

.card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

/* ===== 响应式 ===== */

@media (max-width: 768px) {
  h1 {
    font-size: 2rem;
  }

  h2 {
    font-size: 1.5rem;
  }

  h3 {
    font-size: 1.25rem;
  }

  .content {
    padding: 16px;
  }
}
```

---

## 第二步：创建自定义布局组件

### 2.1 编辑 `.vitepress/theme/Layout.vue` 

创建文件：`.vitepress/theme/Layout.vue`

```vue
<template>
  <div class="layout">
    <DefaultLayout />
  </div>
</template>

<script setup>
import DefaultLayout from 'vitepress/theme'
</script>

<style scoped>
.layout {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}
</style>
```

### 2.2 创建 Footer 组件

创建文件：`.vitepress/theme/components/Footer.vue`

```vue
<template>
  <footer class="footer">
    <div class="footer-content">
      <div class="footer-section">
        <h4>关于</h4>
        <ul>
          <li><a href="/about/">关于我</a></li>
          <li><a href="/about/contact.html">联系我</a></li>
          <li><a href="/privacy.html">隐私政策</a></li>
        </ul>
      </div>
      
      <div class="footer-section">
        <h4>内容</h4>
        <ul>
          <li><a href="/vlog/">Vlog</a></li>
          <li><a href="/blog/">博客</a></li>
          <li><a href="/guide/">指南</a></li>
        </ul>
      </div>
      
      <div class="footer-section">
        <h4>社交</h4>
        <ul>
          <li><a href="https://github.com" target="_blank">GitHub</a></li>
          <li><a href="https://twitter.com" target="_blank">Twitter</a></li>
          <li><a href="https://youtube.com" target="_blank">YouTube</a></li>
        </ul>
      </div>
      
      <div class="footer-section">
        <h4>订阅</h4>
        <p>订阅获取最新内容</p>
        <form @submit.prevent="subscribe">
          <input 
            v-model="email" 
            type="email" 
            placeholder="输入你的邮箱"
            required
          />
          <button type="submit" class="btn-subscribe">订阅</button>
        </form>
      </div>
    </div>
    
    <div class="footer-bottom">
      <p>&copy; 2026 My Vlog. All rights reserved.</p>
    </div>
  </footer>
</template>

<script setup>
import { ref } from 'vue'

const email = ref('')

function subscribe() {
  console.log('Subscribe:', email.value)
  email.value = ''
}
</script>

<style scoped>
.footer {
  background: var(--vp-c-bg-soft);
  border-top: 1px solid var(--vp-c-border);
  margin-top: 80px;
  padding: 60px 20px 20px;
}

.footer-content {
  max-width: 1200px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 40px;
  margin-bottom: 40px;
}

.footer-section h4 {
  margin-top: 0;
  color: var(--vp-c-text-1);
  font-size: 1rem;
  font-weight: 600;
}

.footer-section ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.footer-section ul li {
  margin: 8px 0;
}

.footer-section a {
  color: var(--vp-c-text-2);
  transition: color 0.3s;
}

.footer-section a:hover {
  color: var(--vp-c-brand);
}

.footer-section form {
  display: flex;
  gap: 8px;
}

.footer-section input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid var(--vp-c-border);
  border-radius: 4px;
  background: var(--vp-c-bg);
  color: var(--vp-c-text-1);
}

.btn-subscribe {
  padding: 8px 16px;
  background: var(--vp-c-brand);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background 0.3s;
}

.btn-subscribe:hover {
  background: var(--vp-c-brand-dark);
}

.footer-bottom {
  text-align: center;
  padding-top: 20px;
  border-top: 1px solid var(--vp-c-border);
  color: var(--vp-c-text-3);
}

@media (max-width: 768px) {
  .footer-content {
    grid-template-columns: 1fr;
    gap: 24px;
  }
  
  .footer-section form {
    flex-direction: column;
  }
}
</style>
```

---

## 第三步：自定义导航栏

### 3.1 编辑 `.vitepress/config.ts` 中的 nav

```typescript
export default defineConfig({
  themeConfig: {
    nav: [
      { text: '首页', link: '/' },
      { text: 'Vlog', link: '/vlog/', activeMatch: '^/vlog' },
      { text: '博客', link: '/blog/', activeMatch: '^/blog' },
      {
        text: '更多',
        items: [
          { text: '指南', link: '/guide/' },
          { text: '资源', link: '/resources/' },
          { text: '关于', link: '/about/' }
        ]
      }
    ],
    
    // 搜索配置
    search: {
      provider: 'local',
      options: {
        locales: {
          'zh-CN': {
            translations: {
              button: {
                buttonText: '搜索',
                buttonAriaLabel: '搜索文档'
              },
              modal: {
                noResultsText: '无法找到相关结果',
                resetButtonTitle: '清除查询条件',
                footer: {
                  selectText: '选择',
                  navigateText: '导航',
                  closeText: '关闭'
                }
              }
            }
          }
        }
      }
    }
  }
})
```

---

## 第四步：深色模式支持

### 4.1 在 config.ts 中配置深色模式

```typescript
export default defineConfig({
  appearance: 'dark', // 'light' | 'dark' | 'auto'
  
  themeConfig: {
    // 主题配置
  }
})
```

### 4.2 创建深色模式样式

在 `custom.css` 中已包含深色模式的颜色变量定义。

---

## 第五步：响应式设计优化

### 5.1 添加移动端适配

编辑 `.vitepress/theme/styles/custom.css`，添加移动端特定样式：

```css
/* ===== 移动设备优化 ===== */

@media (max-width: 768px) {
  :root {
    font-size: 14px;
  }

  h1 {
    font-size: 1.75rem;
  }

  h2 {
    font-size: 1.25rem;
  }

  .container {
    padding: 16px;
  }
}

@media (max-width: 480px) {
  h1 {
    font-size: 1.5rem;
  }

  h2 {
    font-size: 1.1rem;
  }

  p {
    font-size: 0.9rem;
  }
}
```

---

## 第六步：添加过渡和动画

### 6.1 创建动画样式

在 `custom.css` 中添加：

```css
/* ===== 过渡和动画 ===== */

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateX(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.fade-in {
  animation: fadeIn 0.5s ease-out;
}

.slide-in {
  animation: slideIn 0.5s ease-out;
}

/* 页面过渡 */
.view {
  animation: fadeIn 0.3s ease-out;
}
```

---

## 📋 验证清单

完成此部分后，检查：

- ✅ 自定义颜色和品牌色生效
- ✅ 深色模式正常切换
- ✅ 移动端显示良好
- ✅ 导航栏样式符合预期
- ✅ 卡片和按钮样式一致
- ✅ 所有过渡和动画流畅

---

## 🔗 下一步

→ **[[VitePress Vlog 教程 - 第4部分：功能扩展]]**

---

最后更新：2026-07-17
