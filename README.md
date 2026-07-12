# AI Agent 工程师知识体系

这是一个使用 Jekyll 和 Markdown 编写的中文技术博客，计划发布为 GitHub Pages 项目站点。

## 目录结构

```text
.
├── _config.yml       # Jekyll 配置
├── _posts/           # 博客文章
├── about.md          # 关于页面
├── index.md          # 首页与文章列表
└── Gemfile           # 本地 Jekyll 依赖
```

## 添加文章

在 `_posts` 目录中新建 Markdown 文件，文件名必须符合：

```text
YYYY-MM-DD-title.md
```

例如：

```text
2026-07-13-context-engineering.md
```

文章开头需要包含 YAML front matter：

```yaml
---
layout: post
title: "文章标题"
date: 2026-07-13 10:00:00 +0800
---
```

front matter 之后就可以使用普通 Markdown 写作。

## 本地预览

需要先安装 Ruby 和 Bundler，然后在项目目录运行：

```powershell
bundle install
bundle exec jekyll serve
```

项目站点配置了 `baseurl`，本地访问地址通常为：

```text
http://127.0.0.1:4000/ai-agent-engineering/
```

## 发布到 GitHub Pages

1. 在 GitHub 创建一个名为 `ai-agent-engineering` 的空仓库，不要额外生成 README、`.gitignore` 或 License。
2. 按 GitHub 页面给出的地址添加远程仓库并推送：

   ```powershell
   git remote add origin https://github.com/YOUR-USERNAME/ai-agent-engineering.git
   git push -u origin main
   ```

3. 进入 GitHub 仓库的 **Settings → Pages**。
4. 在 **Build and deployment** 中选择 **Deploy from a branch**。
5. 选择 `main` 分支和 `/(root)` 目录，然后保存。
6. 等待 GitHub 构建完成后访问：

   ```text
   https://YOUR-USERNAME.github.io/ai-agent-engineering/
   ```

如果以后修改仓库名，也要同步修改 `_config.yml` 中的 `baseurl`。
