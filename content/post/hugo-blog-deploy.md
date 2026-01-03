+++
date = '2026-01-03T22:35:27+08:00'
title = 'Hugo 博客本地搭建与 GitHub Pages 部署完整方案'
+++

# Hugo 博客本地搭建与 GitHub Pages 部署完整方案

本文提供从零开始搭建 Hugo 博客并部署到 GitHub Pages 的完整流程，包含 **本地开发、自动部署、避坑指南和常见问题排查**，适合新手快速上手。

> 💡 **推荐使用 GitHub Actions 自动部署**（无需手动管理 `public/` 目录），本文以该方式为主。

------

## 一、本地搭建 Hugo 博客

### 1. 安装 Hugo

首先需要安装 Hugo（**建议使用 extended 版本**，支持 SCSS 编译）。
根据你的操作系统选择安装方式：[Hugo 官方安装指南](https://gohugo.io/installation/)

### 2. 创建新站点

```bash
hugo new site my-blog
cd my-blog
```

此命令会创建 Hugo 站点的基本目录结构。

### 3. 初始化 Git 仓库

```bash
git init
echo "pektzz.github.io" > CNAME  # 可选：如果你计划用自定义域名
```

### 4. 添加主题（以 Ananke 为例）

```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo "theme = 'ananke'" >> hugo.toml
```

> ✅ 主题作为 submodule 管理，便于后续更新。

### 5. 创建内容

```bash
hugo new posts/my-first-post.md
```

> ⚠️ 注意：目录名是 **`posts`（带 s）**，这是 Hugo 默认的博客 section 名称。

编辑生成的文件，添加你的内容，并**删除或注释掉 `draft = true`**（否则文章不会发布）。

### 6. 本地预览

```bash
hugo server -D
```

访问 `http://localhost:1313` 查看效果。`-D` 参数用于显示草稿（仅本地）。

### 7. 配置站点

编辑 `hugo.toml`：

```toml
baseURL = 'https://pektzz.github.io/'
languageCode = 'zh-cn'
title = '我的博客'
theme = 'ananke'
```

> 🔑 **关键**：`baseURL` 必须与你的 GitHub Pages URL **完全一致**，且以 `/` 结尾。

### 8. 生成静态文件（仅测试用）

```bash
hugo
```

生成的文件位于 `public/` 目录。**但请勿将 `public/` 提交到 Git！**

------

## 二、部署到 GitHub Pages（使用 GitHub Actions）

### 1. 在 GitHub 创建仓库

仓库名必须为：`<your-username>.github.io`（例如 `pektzz.github.io`）

### 2. 配置 GitHub Actions 自动部署（推荐）

在项目根目录创建 `.github/workflows/hugo.yml`：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true
          
      - name: Build
        run: hugo --minify
        
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v3
```

> ✅ 使用 `@v3` 或 `@v4` 是当前最新稳定版。

### 3. 配置 `.gitignore`

创建 `.gitignore` 文件，防止提交构建产物：

```gitignore
# Hugo
/public/
/resources/_gen/
.hugo_build.lock

# OS
.DS_Store
Thumbs.db
```

### 4. 提交代码

```bash
git add .
git commit -m "Initial Hugo site with GitHub Actions"
git remote add origin git@github.com:pektzz/pektzz.github.io.git
git branch -M main
git push -u origin main
```

> 🔐 使用 SSH URL（`git@github.com:...`）可避免每次输入密码。

### 5. 在 GitHub 启用 Pages

1. 进入仓库 → **Settings → Pages**
2. **Source 选择 “GitHub Actions”**
3. 保存

> ⚠️ 如果下拉菜单没有 “GitHub Actions”，先手动运行一次 workflow（Actions 页面 → Run workflow），再刷新设置页。

### 6. 日常更新流程

```bash
# 创建新文章
hugo new posts/new-post.md

# 编辑后预览
hugo server -D

# 提交源码（无需 hugo 命令）
git add .
git commit -m "Add new post"
git push origin main
```

GitHub Actions 会自动构建并部署。

------

## 三、常见问题排查（实战经验）

### ❌ 问题 1：首页看不到文章

**原因**：文章 front matter 中有 `draft = true`
**解决**：删除该行或设为 `draft = false`

```toml
+++ 
# 删除这一行 ↓
# draft = true
title = "My Post"
+++
```

------

### ❌ 问题 2：文章列表为空，但文件存在

**原因**：目录名写成 `content/post/`（不带 s）
**解决**：重命名为 `content/posts/`（Hugo 默认只识别 `posts` section）

```bash
mv content/post content/posts
git add .
git commit -m "Fix: rename post → posts"
```

------

### ❌ 问题 3：点击文章链接 404

可能原因及解决方案：

#### a) **GitHub Pages Source 未设为 “GitHub Actions”**

→ 进入 Settings → Pages → Source 选择 **GitHub Actions**

#### b) **误将 `public/` 提交到仓库**

→ 从 Git 中移除并加入 `.gitignore`：

```bash
git rm -r --cached public/
echo "/public/" >> .gitignore
git add .gitignore && git commit -m "Remove public/"
```

#### c) **文件名含大写字母（如 `MyPost.md`）**

→ GitHub Pages（Linux）区分大小写，建议：

- 文件名全小写：`my-post.md`

- 或在 front matter 中指定 

  ```
  slug
  ```

  ：

  ```toml
  slug = "my-post"
  ```

#### d) **URL 缺少末尾 `/`**

→ 正确访问：`https://xxx.github.io/posts/my-post/`（带 `/`）
→ 错误访问：`https://xxx.github.io/posts/my-post`（404）

------

### ❌ 问题 4：资源（CSS/JS）加载失败

**原因**：`baseURL` 配置错误（缺少末尾 `/`）
**解决**：确保 `hugo.toml` 中为：

```toml
base URL = 'https://your-username.github.io/'  # 必须有 /
```

------

## 四、最佳实践总结

| 项目         | 推荐做法                             |
| ------------ | ------------------------------------ |
| **部署方式** | GitHub Actions（自动、安全）         |
| **主题管理** | Git submodule                        |
| **内容目录** | `content/posts/`（带 s）             |
| **草稿处理** | 本地用 `-D`，发布前删 `draft = true` |
| **构建产物** | **绝不提交 `public/`**               |
| **文件命名** | 全小写 + 连字符（如 `my-post.md`）   |
| **baseURL**  | 以 `/` 结尾                          |

------

## 五、后续优化建议

- 添加评论系统（如 [Giscus](https://giscus.app/)）
- 配置 SEO 元信息（Ananke 支持）
- 启用 RSS 订阅
- 自定义域名（添加 `CNAME` 文件）

------

现在，你的 Hugo 博客已经稳定运行在 GitHub Pages 上！🎉
只需专注写作，剩下的交给自动化流程。

> 本文档本身即通过上述流程部署，欢迎访问：[https://pektzz.github.io](https://pektzz.github.io/)

------

✅ **祝你写作愉快，技术精进！**

