# MIutopia's Blog - AI 协作规范

## 项目概述
Hexo 博客源码仓库，使用 Fluid 主题，部署在 Vercel。
- 域名: https://www.cloudetopia.top
- 框架: Hexo 7.x + hexo-theme-fluid
- 部署: Vercel (自动构建，git push 触发)
- 文章目录: source/_posts/

## 文章规范
- 文件名格式: YYYY-MM-DD-标题.md
- frontmatter 必须包含: title, date, categories, tags, description
- categories 使用一级分类: 教程 / 云原生 / 云计算 / 数据库 / 存储 / 负载均衡 / 架构
- tags 使用具体技术名: Docker, Kubernetes, Nginx, MySQL 等
- AI 系列文章统一添加 AI Agent 标签

## 安全规则
- 每次批量修改前先创建 checkpoint commit
- 高风险操作(git reset --hard, git push -f)前必须确认
- 不要删除 source/images/ 目录下的图片
- 不要修改 _config.yml 和 _config.fluid.yml 中的域名和 URL 配置

## 构建命令
- 本地预览: npx hexo server
- 生成静态文件: npx hexo generate
- 新建文章: npx hexo new "标题"