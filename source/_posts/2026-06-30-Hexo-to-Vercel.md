---
title: Hexo 博客迁移全记录：从 GitHub Pages 到 Vercel
date: 2026-06-30 10:00:00
categories:
  - 教程
tags:
  - DevOps
  - Hexo
  - Vercel
---

## 前言

我的个人博客（cloudetopia.top）原本部署在 GitHub Pages 上，一直存在几个痛点：部署流程繁琐、国内访问慢、图片缓存难控制等。最近趁着本地源文件丢失的机会，干脆做了一次彻底的技术栈升级——**从 GitHub Pages 迁移到 Vercel**。

这篇文章完整记录了迁移过程、踩坑和解决方案，供有同样需求的朋友参考。

## 旧架构的问题

迁移前的架构：

```
本地写 Markdown → hexo generate → hexo deploy → GitHub Pages
```

这个流程有几个明显的问题：

- **部署依赖本地环境**：换台电脑就得重新装 Node.js、Hexo CLI
- **国内访问慢**：GitHub Pages 服务器在海外
- **主题文件丢失**：本地源文件丢了，LiveForCode 主题只剩残片
- **无 CI/CD**：每次更新都要手动构建、推送

## 新架构：Vercel 方案

迁移后的架构：

```
本地写 Markdown → git push → Vercel 自动构建 → 全球 CDN 分发
```

一步到位，自动部署。

### 对比优势

| 维度 | GitHub Pages | Vercel |
|------|------------|--------|
| 部署方式 | 本地手动 | git push 自动 |
| 国内访问 | 慢 | Edge Network |
| CI/CD | 无 | 内置 |
| Serverless | 不支持 | 支持 |
| 成本 | 免费 | 免费 |

## 迁移过程

花了几个小时完成了整个迁移，核心步骤如下：

### 1. 配置 Vercel 部署

在项目根目录创建 `vercel.json`：

```json
{
  "buildCommand": "npx hexo generate",
  "outputDirectory": "public",
  "installCommand": "npm install",
  "framework": null
}
```

然后去 Vercel 导入 GitHub 仓库，绑定域名就好了。

### 2. 替换主题

原 LiveForCode 主题的模板文件已丢失，选了 **Hexo Fluid** 主题替代。安装很简单：

```bash
npm install --save hexo-theme-fluid hexo-renderer-ejs hexo-renderer-stylus
```

> ⚠️ 一定要装 `hexo-renderer-ejs` 和 `hexo-renderer-stylus`，否则 Fluid 主题会生成空白页面。

然后在 `_config.yml` 里设置 `theme: fluid`，再写一份 `_config.fluid.yml` 自定义配置。

### 3. 迁移图片

旧博客的头图存在 hexo-blog 仓库里，选了几张迁移过来：

```bash
mkdir -p source/images/hero
# 下载 01~09.jpg
```

在主题配置里指向新路径即可。

## 踩坑记录

### ❌ 部署后空白页面

Vercel 返回 HTTP 200 但 HTML 是空的——**缺少 EJS 渲染器**。

**解决**：安装 `hexo-renderer-ejs` 和 `hexo-renderer-stylus`。

### ❌ hexo: command not found (exit 127)

Vercel 默认把框架识别为 "Hexo"，内置构建命令不兼容。

**解决**：在 Vercel 项目 Settings → Build & Development Settings 里把 Framework Preset 改成 **Other**。

### ❌ 提交邮箱不匹配被阻止

Vercel 校验提交邮箱，`noreply` 邮箱被阻止。

**解决**：改成 GitHub 绑定的真实邮箱，关掉 Require Verified Commits。

### ❌ 自动部署没触发

Git 集成没正确配置。

**解决**：Vercel 项目 Settings → Git 确认仓库已连接，Production Branch 设为 `main`。

## 最终工作流

现在发布一篇文章只需要：

```bash
git add .
git commit -m "新文章"
git push origin main
# ✅ Vercel 自动构建部署
```

整个过程完全自动化，再也不需要本地 `hexo generate` 和 `hexo deploy` 了。

## 关于 Fluid 主题

Fluid 是一个非常成熟的中文 Hexo 主题，支持：

- 全屏 Banner 头图
- Material Design 卡片布局
- 代码高亮 + 复制按钮
- 中文搜索
- 文章目录导航
- 自动摘要

配置也很灵活，通过 `_config.fluid.yml` 可以定制导航、首页样式、文章元信息等。

## 后续计划

- [ ] 上传自定义头像
- [ ] 用 Vercel KV 实现阅读计数器
- [ ] 换一个更流畅的评论系统
- [ ] 集成站点统计

---

*以上就是这次迁移的完整记录。源码在 [MIutopia/hexo-blog-source](https://github.com/MIutopia/hexo-blog-source)，欢迎参考。*
