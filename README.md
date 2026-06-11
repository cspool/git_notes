# Git Notes

一个使用 Jekyll 和 GitHub Pages 发布的博客。

## 发布

1. 在 GitHub 创建仓库，并把本目录推送到仓库的 `main` 分支。
2. 打开仓库的 `Settings` -> `Pages`。
3. 在 `Build and deployment` 下，将 `Source` 设置为 `GitHub Actions`。
4. 打开 `Actions` 页面查看 `Deploy Jekyll site to Pages` 工作流。

首次部署完成后，博客地址通常为：

- 项目仓库：`https://<username>.github.io/<repository>/`
- 用户站点仓库 `<username>.github.io`：`https://<username>.github.io/`

## 写文章

在 `_posts` 中新增 `YYYY-MM-DD-title.md` 文件。每篇文章需要以 Jekyll
Front Matter 开头：

```yaml
---
layout: post
title: "文章标题"
date: 2026-06-11 16:30:00 +0800
categories: notes
---
```

推送到 `main` 分支后，GitHub Actions 会自动构建并发布。

站点标题、简介、导航和链接格式可在 `_config.yml` 中修改。
