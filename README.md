# 云原生智造

基于 Hugo 和 FixIt 的技术博客，发布于 GitHub Pages。

## 发布文章

在 `content/posts/` 新建 Markdown 文件，填写 front matter，例如：

```yaml
---
title: "文章标题"
date: 2026-11-20
categories: ["AI Infra"]
tags: ["Kubernetes"]
---

文章内容。
```

提交并推送到 `main` 后，GitHub Actions 会构建并发布网站。专题导航在 `content/topics/index.md`，站点配置在 `hugo.toml`。

本地运行需要 Hugo Extended 0.170.1 和 Dart Sass；克隆仓库时使用 `git clone --recurse-submodules`。
