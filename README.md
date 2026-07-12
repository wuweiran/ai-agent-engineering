# AI Agent 工程师知识体系

这是一个使用 Jekyll、Just the Docs 和 Markdown 编写的中文技术知识库，发布为 GitHub Pages 项目站点。

## 目录结构

```text
.
├── _config.yml       # Jekyll 与主题配置
├── docs/             # 知识库文档
├── about.md          # 关于页面
├── index.md          # 首页
└── Gemfile           # 本地 Jekyll 依赖
```

## 添加一级文档

在 `docs` 目录中新建 Markdown 文件，并添加 YAML front matter：

```yaml
---
layout: default
title: 上下文工程
nav_order: 4
permalink: /docs/context-engineering/
---
```

`nav_order` 决定页面在左侧导航中的顺序。

## 添加树形子文档

父页面需要声明它包含子页面：

```yaml
---
layout: default
title: 上下文工程
nav_order: 4
has_children: true
---
```

子页面通过 `parent` 指向父页面：

```yaml
---
layout: default
title: Context Window
parent: 上下文工程
nav_order: 1
---
```

`parent` 必须与父页面的 `title` 完全一致。

## 本地预览

需要先安装 Ruby 和 Bundler，然后在项目目录运行：

```powershell
bundle install
bundle exec jekyll serve
```

本地访问地址通常为：

```text
http://127.0.0.1:4000/ai-agent-engineering/
```

## 发布到 GitHub Pages

提交修改并推送到 GitHub：

```powershell
git add .
git commit -m "Convert site to hierarchical documentation"
git push
```

仓库的 **Settings → Pages** 应设置为：

- Source：`Deploy from a branch`
- Branch：`main`
- Folder：`/(root)`

推送后可以在仓库的 **Actions** 页面查看构建状态，站点地址为：

```text
https://wuweiran.github.io/ai-agent-engineering/
```

如果以后修改仓库名，也要同步修改 `_config.yml` 中的 `baseurl`。
