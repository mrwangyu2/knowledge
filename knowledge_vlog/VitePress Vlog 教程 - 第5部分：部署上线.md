---
title: VitePress Vlog 教程 - 第5部分：部署上线
tags: [vitepress, vlog, 部署, 上线]
date: 2026-07-17
---

# VitePress Vlog 从零开始搭建教程

## 第5部分：部署上线与持续发布

### 📌 本部分目标

- 本地构建和测试
- GitHub 仓库配置
- CI/CD 自动部署
- 多个部署平台对比
- 自定义域名配置
- 性能监控和优化

---

## 第一步：本地构建和测试

### 1.1 构建项目

```bash
npm run docs:build
```

输出目录：`.vitepress/dist/`

### 1.2 本地预览构建结果

```bash
npm run docs:preview
```

访问 `http://localhost:4173` 查看最终效果

### 1.3 检查构建输出

```bash
# 查看生成的文件
ls -la .vitepress/dist/

# 查看文件大小
du -sh .vitepress/dist/

# 测试所有链接
npx broken-link-checker http://localhost:4173
```

---

## 第二步：GitHub 仓库设置

### 2.1 初始化 Git 仓库

```bash
# 初始化 Git
git init

# 添加远程仓库
git remote add origin https://github.com/your-username/my-vlog.git

# 创建 .gitignore
cat > .gitignore << EOF
node_modules/
.vitepress/dist/
.vitepress/cache/
.env
.env.local
dist/
*.log
.DS_Store
EOF

# 首次提交
git add .
git commit -m "Initial commit: VitePress Vlog setup"
git branch -M main
git push -u origin main
```

### 2.2 配置 GitHub Pages

在 GitHub 仓库设置中：
1. 进入 **Settings → Pages**
2. Source 选择 **Deploy from a branch**
3. Branch 选择 **main** 和 **/root** 目录
4. 点击 **Save**

---

## 第三步：GitHub Actions 自动部署

### 3.1 创建自动部署工作流

创建文件：`.github/workflows/deploy.yml`

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. 检出代码
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      # 2. 设置 Node.js
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      # 3. 安装依赖
      - name: Install dependencies
        run: npm ci

      # 4. 构建网站
      - name: Build site
        run: npm run docs:build

      # 5. 部署到 GitHub Pages
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./.vitepress/dist
          cname: yourdomain.com  # 如果使用自定义域名

      # 6. 部署通知（可选）
      - name: Deployment notification
        if: always()
        run: |
          echo "Deployment status: ${{ job.status }}"
```

### 3.2 提交工作流文件

```bash
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Actions deployment workflow"
git push
```

监控部署过程：访问 GitHub 仓库 → **Actions** 标签

---

## 第四步：Vercel 部署（推荐）

### 4.1 连接 Vercel

1. 访问 [vercel.com](https://vercel.com)
2. 使用 GitHub 账号登录
3. 点击 **New Project**
4. 导入你的 GitHub 仓库
5. 项目配置自动识别

### 4.2 配置 Vercel

创建文件：`vercel.json`

```json
{
  "buildCommand": "npm run docs:build",
  "outputDirectory": ".vitepress/dist"
}
```

### 4.3 环境变量（如需要）

在 Vercel 项目设置中添加环境变量：

```
ANALYTICS_ID=your-analytics-id
API_KEY=your-api-key
```

### 4.4 Vercel 优势

- ✅ 极快的全球 CDN
- ✅ 自动 HTTPS
- ✅ 预览部署
- ✅ 性能监控
- ✅ 自动回滚

---

## 第五步：Netlify 部署

### 5.1 连接 Netlify

1. 访问 [netlify.com](https://netlify.com)
2. 点击 **Add new site → Import an existing project**
3. 选择 GitHub 仓库
4. 配置自动识别

### 5.2 配置文件

创建文件：`netlify.toml`

```toml
[build]
command = "npm run docs:build"
publish = ".vitepress/dist"

[build.environment]
NODE_VERSION = "18"

# 重定向规则
[[redirects]]
from = "/*"
to = "/index.html"
status = 200

# 缓存规则
[[headers]]
for = "/_nuxt/*"
[headers.values]
Cache-Control = "public, max-age=31536000, immutable"
```

---

## 第六步：自定义域名配置

### 6.1 GitHub Pages 自定义域名

#### 方式 1：通过 GitHub 设置

1. Settings → Pages
2. 在 **Custom domain** 输入你的域名
3. 点击 **Save**

#### 方式 2：手动配置 DNS

1. 在 GitHub 仓库根目录创建 `CNAME` 文件

```bash
echo "yourdomain.com" > docs/public/CNAME
```

2. 在域名提供商配置 DNS 记录

对于 Vercel/Netlify：
```
A 记录: 76.76.19.89（Vercel）或相应的 IP
CNAME: 指向提供商的服务
```

### 6.2 HTTPS 证书

所有现代部署平台都自动提供 HTTPS：
- GitHub Pages：自动
- Vercel：自动
- Netlify：自动

---

## 第七步：性能监控和优化

### 7.1 使用 Lighthouse CI

创建文件：`.github/workflows/lighthouse.yml`

```yaml
name: Lighthouse CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      
      - run: npm ci
      - run: npm run docs:build
      
      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          uploadArtifacts: true
          temporaryPublicStorage: true
```

### 7.2 Web Vitals 监控

在 config.ts 中添加：

```typescript
export default defineConfig({
  head: [
    ['script', {}, `
      // 监控 Web Vitals
      window.addEventListener('web-vital', (event) => {
        console.log(event.detail)
        // 发送到分析服务
        fetch('/api/analytics', {
          method: 'POST',
          body: JSON.stringify(event.detail)
        })
      })
    `]
  ]
})
```

---

## 第八步：发布流程和最佳实践

### 8.1 标准发布流程

```bash
# 1. 创建功能分支
git checkout -b feature/new-video

# 2. 编写和测试内容
npm run docs:dev

# 3. 本地构建验证
npm run docs:build
npm run docs:preview

# 4. 提交更改
git add .
git commit -m "feat: add new vlog video about VitePress"

# 5. 推送到远程
git push origin feature/new-video

# 6. 创建 Pull Request
# 在 GitHub 上创建 PR，等待自动检查

# 7. 合并到 main
# 获得审批后合并

# 8. 自动部署触发
# GitHub Actions 自动构建和部署
```

### 8.2 快速发布脚本

创建文件：`scripts/publish.sh`

```bash
#!/bin/bash

# 快速发布脚本
set -e

echo "🚀 开始发布流程..."

# 本地测试
echo "📝 运行本地测试..."
npm run docs:build > /dev/null 2>&1

echo "✅ 本地构建成功"

# 获取提交信息
read -p "输入提交信息: " message

# 提交并推送
git add .
git commit -m "$message"
git push

echo "🎉 已推送到 GitHub，自动部署中..."
echo "📊 查看进度: https://github.com/your-username/my-vlog/actions"
```

使用方法：

```bash
chmod +x scripts/publish.sh
./scripts/publish.sh
```

---

## 第九步：故障排查

### 常见问题

#### 部署失败：找不到 npm 包

```bash
# 清理缓存并重新安装
rm -rf node_modules package-lock.json
npm install
npm run docs:build
```

#### 构建超时

增加 Node.js 内存：

```bash
NODE_OPTIONS=--max_old_space_size=4096 npm run docs:build
```

#### 页面 404

检查 base 路径配置：

```typescript
// config.ts
export default defineConfig({
  base: '/my-vlog/'  // 如果部署在子路径
})
```

#### CSS/JS 没有加载

清除缓存和重新部署：

```bash
rm -rf .vitepress/cache .vitepress/dist
npm run docs:build
```

---

## 第十步：部署平台对比

| 特性 | GitHub Pages | Vercel | Netlify |
|------|-------------|--------|---------|
| **部署速度** | 中等 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **性能** | 中等 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **CDN** | 有 | 全球CDN | 全球CDN |
| **免费套餐** | 完全免费 | 有限制 | 有限制 |
| **自定义域名** | ✅ | ✅ | ✅ |
| **HTTPS** | 自动 | 自动 | 自动 |
| **环境变量** | ❌ | ✅ | ✅ |
| **预览部署** | ❌ | ✅ | ✅ |
| **分析工具** | ❌ | ✅ | ✅ |

**推荐方案**：Vercel（最佳体验）或 GitHub Pages（完全免费）

---

## 📋 部署检查清单

完成此部分后，检查：

- ✅ 本地构建和预览正常
- ✅ GitHub 仓库已配置
- ✅ CI/CD 工作流已启用
- ✅ 首次部署成功
- ✅ 自定义域名已配置
- ✅ HTTPS 正常工作
- ✅ 性能监控已设置
- ✅ 能快速发布新内容

---

## 🎯 发布你的第一个 Vlog！

现在你已经拥有一个完整的 Vlog 平台，可以开始：

1. 📹 上传你的第一个视频
2. ✍️ 编写视频说明和相关文章
3. 🏷️ 添加标签和分类
4. 📤 发布到网络
5. 📊 监控访问数据
6. 💬 与观众互动

---

## 🔗 相关资源

- [VitePress 官方文档](https://vitepress.dev)
- [GitHub Pages 文档](https://pages.github.com)
- [Vercel 文档](https://vercel.com/docs)
- [Netlify 文档](https://docs.netlify.com)

---

## 💡 下一步建议

- 定期更新内容
- 优化 SEO
- 建立读者社区
- 寻求反馈
- 持续改进设计和功能

---

最后更新：2026-07-17

**完成整个教程！🎉**
