---
title: "Hexo 博客迁移全记录：从 GitHub Pages 到 Vercel"
date: 2026-06-30 10:00:00
categories: ["DevOps"]
tags: ["Hexo", "Vercel", "GitHub Pages", "博客迁移", "CI/CD"]
cover: /images/hero/01.jpg
summary: "一次完整的 Hexo 博客迁移实战，从 GitHub Pages 到 Vercel，解决部署流程繁琐、国内访问慢、主题缺失等痛点。"
---

# Hexo 博客迁移全记录：从 GitHub Pages 到 Vercel

## 📝 背景与痛点

作为一个持续维护博客三年的技术博主，我遇到了很多困扰。迁移前，博客采用传统的 **Hexo + GitHub Pages** 架构，每次更新都需要本地执行构建和部署，流程非常繁琐。

### 旧架构流程

```
本地编辑 Markdown
      ↓
  hexo generate（生成静态文件）
      ↓
  hexo deploy（推送到 hexo-blog 仓库）
      ↓
  GitHub Pages 托管
```

### 遇到的痛点

| 痛点 | 影响 |
|------|------|
| **部署流程繁琐** | 每次更新需要本地执行构建推送，无法在 GitHub 上直接完成 |
| **构建依赖本地环境** | 换电脑需要重新搭建 Node.js + Hexo CLI 环境 |
| **国内访问慢** | GitHub Pages 服务器在海外，中国大陆访问体验差 |
| **无 CI/CD** | 没有自动化构建部署流水线 |
| **评论系统卡顿** | Gitalk 依赖 GitHub API，在中国大陆访问经常超时 |
| **本地源文件丢失** | 主题文件、配置文件等源文件在本地全部丢失 |

> **转折点**：一次意外导致本地源文件全部丢失，只剩 GitHub 上的源码仓库。这促使我下定决心进行彻底的迁移改造。

---

## 🏗️ 架构对比分析

经过调研，我选择了 **Vercel** 作为新的托管平台。以下是新旧方案的详细对比：

| 维度 | 旧方案 (GitHub Pages) | 新方案 (Vercel) |
|------|----------------------|------------------|
| **托管平台** | GitHub Pages | Vercel Edge Network |
| **构建部署** | 本地手动构建 + 推送 | `git push` 即自动部署 |
| **CI/CD** | ❌ 无 | ✅ 自动从 Git 仓库触发 |
| **Preview Deploy** | ❌ 无 | ✅ 每个 PR 生成预览链接 |
| **全球 CDN** | 🟡 第三方 | ✅ Vercel Edge Network |
| **Serverless 函数** | ❌ 不支持 | ✅ 可添加 API 函数 |
| **HTTPS 证书** | ✅ 自动管理 | ✅ 自动管理 |
| **回滚能力** | 🟡 手动回退 | ✅ 一键回滚 |
| **成本** | ✅ 免费 | ✅ Hobby 计划免费足够 |

### 迁移后的工作流

```
本地写文章 (Markdown)
      ↓
  git push
      ↓
  Vercel 自动检测到推送
      ↓
  执行 npm install → npx hexo generate
      ↓
  部署到 Vercel Edge Network
      ↓
  全球 CDN 分发
```

> **核心优势**：从 "本地构建 → 推送" 简化为 "git push"，一步到位！

---

## 🚀 迁移流程详解

### 第一步：配置 Vercel 部署

创建 `vercel.json` 配置文件，告诉 Vercel 如何构建项目：

```json
{
  "buildCommand": "npx hexo generate",
  "outputDirectory": "public",
  "installCommand": "npm install",
  "framework": null
}
```

同时添加 `.vercelignore`，排除不需要部署的文件：

```
node_modules/
.deploy_git/
db.json
README.md
```

### 第二步：在 Vercel 导入项目

1. 访问 [vercel.com/new](https://vercel.com/new)
2. 选择 **Import Git Repository** → `MIutopia/hexo-blog-source`
3. 确认配置，点击 **Deploy**
4. 添加自定义域名 `cloudetopia.top`

### 第三步：清理旧 GitHub Pages

删除 `hexo-blog` 仓库中的 `CNAME` 文件，并关闭 GitHub Pages 功能，避免 DNS 冲突。

---

## 🎨 主题迁移与配置

### 主题选型

由于旧主题 **LiveForCode** 的核心模板文件已丢失，我选择了 **Hexo Fluid** 主题，原因如下：

| 对比项 | 原 LiveForCode | Fluid |
|--------|---------------|-------|
| 首页头图 | 全屏轮播 | 全屏 Banner + 动态背景 |
| 文章卡片 | 简洁卡片 | Material Design 卡片 |
| 社区维护 | 不可考 | 活跃社区，持续更新 |
| 安装方式 | 源码内嵌 | `npm install hexo-theme-fluid` |

### 安装过程

```bash
# 安装主题
npm install --save hexo-theme-fluid

# 安装缺失渲染器（关键！否则生成空白页面）
npm install --save hexo-renderer-ejs hexo-renderer-stylus
```

> **踩坑提示**：缺少渲染器时，Hexo 会直接把原始模板文件复制到输出目录，生成空 HTML。这是一个非常容易踩的坑！

### 主题配置

`_config.fluid.yml` 关键配置：

```yaml
navbar:
  blog_title: CloudSheep
  menu:
    - { key: '首页', link: '/', icon: 'iconfont icon-home-fill' }
    - { key: '归档', link: '/archives/', icon: 'iconfont icon-archive-fill' }
    - { key: '分类', link: '/categories/', icon: 'iconfont icon-category-fill' }
    - { key: '标签', link: '/tags/', icon: 'iconfont icon-tags-fill' }
    - { key: '关于', link: '/about/', icon: 'iconfont icon-user-fill' }

index:
  banner_img: /images/hero/01.jpg
  banner_img_height: 70
  slogan:
    enable: true
    text: "CloudSheep's Blog — 云原生 & DevOps 技术笔记"

code:
  highlightjs:
    enable: true
    theme: github
  copy_btn: true
```

---

## 🖼️ 图片资源迁移

### 迁移操作

```bash
# 创建图片目录
mkdir -p source/images/hero

# 下载选定的图片
for i in $(seq -w 1 9); do
  curl -k -sL -o "source/images/hero/${i}.jpg" \
    "https://raw.githubusercontent.com/MIutopia/hexo-blog/master/images/Lycoris-Recoil-a/${i}.jpg"
done
```

> **优化建议**：图片单张控制在 1MB 以内，超过 5MB 的图片建议压缩后再提交。

---

## ⚠️ 常见问题排查

### 问题 1：部署后页面为空白

**症状**：Vercel 返回 HTTP 200，但 HTML 内容为空（0 字节）。

**原因**：缺少 `hexo-renderer-ejs` 和 `hexo-renderer-stylus` 渲染器。

**解决**：
```bash
npm install --save hexo-renderer-ejs hexo-renderer-stylus
```

### 问题 2：Vercel 构建成功但显示旧内容

**原因**：Vercel CDN 缓存。

**解决**：在 Vercel Dashboard 中触发 **Redeploy**。

### 问题 3：自定义域名 DNS 冲突

**症状**：域名解析到旧的 GitHub Pages。

**解决**：删除 `hexo-blog` 仓库中的 `CNAME` 文件，关闭 GitHub Pages 功能。

---

## 📋 最终文件结构

```
hexo-blog-source/
├── source/
│   ├── _posts/               # 博文 (15 篇)
│   └── images/               # 图片资源
├── _config.yml               # Hexo 主配置
├── _config.fluid.yml         # Fluid 主题配置
├── vercel.json               # Vercel 部署配置
└── package.json              # 依赖管理
```

---

## 🚀 后续优化方向

| 优先级 | 任务 | 方案 |
|--------|------|------|
| 🔴 | 上传自定义头像 | 替换默认头像 |
| 🟡 | 迁移博文中的外链图片 | 将图床图片迁入本地 |
| 🟡 | 配置阅读计数器 | 使用 Vercel KV + Edge Function |
| 🟢 | 更换评论系统 | 评估 Twikoo / Waline |
| 🟢 | 添加站点统计 | 集成 Vercel Analytics |

---

## 📖 参考资料

- [Hexo 官方文档](https://hexo.io/docs/)
- [Fluid 主题文档](https://hexo.fluid-dev.com/docs/)
- [Vercel 文档](https://vercel.com/docs)
- [源码仓库](https://github.com/MIutopia/hexo-blog-source)

---

> **写在最后**：这次迁移不仅解决了旧架构的痛点，还让我对 Hexo 和 Vercel 有了更深入的理解。如果你也在使用 GitHub Pages 托管 Hexo 博客，强烈推荐尝试迁移到 Vercel，体验自动化部署的便捷！

> 欢迎在评论区分享你的博客迁移经验，或者提出任何问题！