+++
date = '2026-01-03T22:35:27+08:00'
title = 'Hugo Blog Deploy'
+++

# Hugo 博客本地搭建与 GitHub Pages 部署完整方案

## 一、本地搭建 Hugo 博客

### 1. 安装 Hugo

首先需要安装 Hugo（建议使用 extended 版本）。根据你的操作系统选择相应的安装方式。 [1](#0-0) 

### 2. 创建新站点

```bash
hugo new site my-blog
cd my-blog
```

这个命令会创建一个新的 Hugo 站点骨架结构。 [2](#0-1) 

### 3. 初始化 Git 仓库

```bash
git init
``` [3](#0-2) 

### 4. 添加主题

以 Ananke 主题为例（你可以选择其他主题）：

```bash
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo "theme = 'ananke'" >> hugo.toml
``` [4](#0-3) 

### 5. 创建内容

```bash
hugo new content content/posts/my-first-post.md
```

编辑生成的文件，添加你的内容。 [5](#0-4) 

### 6. 本地预览

```bash
hugo server -D
```

访问终端显示的 URL（通常是 `http://localhost:1313`）查看你的博客。 [6](#0-5) 

### 7. 配置站点

编辑 `hugo.toml` 文件，设置你的站点信息：

```toml
baseURL = 'https://your-username.github.io/'
languageCode = 'zh-cn'
title = '我的博客'
theme = 'ananke'
```

注意：`baseURL` 需要设置为你的 GitHub Pages 地址。 [7](#0-6) 

### 8. 生成静态文件

```bash
hugo
```

这将在 `public` 目录中生成所有静态文件。 [8](#0-7) 

## 二、部署到 GitHub Pages（使用 SSH 方式）

### 1. 配置 SSH 密钥

如果还没有配置 SSH 密钥，需要先生成并添加到 GitHub：

```bash
# 生成 SSH 密钥（如果还没有）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 启动 ssh-agent
eval "$(ssh-agent -s)"

# 添加私钥
ssh-add ~/.ssh/id_ed25519
```

然后将公钥（`~/.ssh/id_ed25519.pub` 的内容）添加到 GitHub：
- 登录 GitHub
- 进入 Settings → SSH and GPG keys → New SSH key
- 粘贴公钥内容并保存

### 2. 在 GitHub 创建仓库

创建一个新的仓库，命名为 `your-username.github.io`（如果要部署到用户主页）或其他名称（如果要部署到项目页面）。

### 3. 配置部署方式

**方法一：使用 GitHub Actions 自动部署（推荐）**

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

defaults:
  run:
    shell: bash

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
        uses: actions/upload-pages-artifact@v2
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
        uses: actions/deploy-pages@v2
```

**方法二：手动部署到 gh-pages 分支**

```bash
# 初始化 public 目录为独立的 git 仓库
cd public
git init
git remote add origin git@github.com:your-username/your-username.github.io.git
git add .
git commit -m "Initial commit"
git branch -M main
git push -u origin main
```

### 4. 推送代码（使用 SSH）

回到项目根目录，添加远程仓库（使用 SSH URL）：

```bash
cd ..
git remote add origin git@github.com:your-username/your-username.github.io.git
git add .
git commit -m "Initial Hugo site"
git branch -M main
git push -u origin main
```

**注意：** 使用 `git@github.com:` 开头的 SSH URL，而不是 `https://github.com/` 开头的 HTTPS URL。

### 5. 在 GitHub 配置 Pages

如果使用 GitHub Actions 部署：
1. 进入仓库的 Settings → Pages
2. Source 选择 "GitHub Actions"

如果使用 gh-pages 分支部署：
1. 进入仓库的 Settings → Pages
2. Source 选择 "Deploy from a branch"
3. Branch 选择 `main` 或 `gh-pages`（取决于你的配置）

### 6. 日常更新流程

```bash
# 创建新文章
hugo new content content/posts/new-post.md

# 编辑文章后预览
hugo server -D

# 生成静态文件
hugo

# 提交并推送（使用 SSH）
git add .
git commit -m "Add new post"
git push origin main
```

## 三、.gitignore 配置建议

在项目根目录创建 `.gitignore`：

```
# Hugo
/public/
/resources/_gen/
.hugo_build.lock

# OS
.DS_Store
Thumbs.db
```

如果使用方法一（GitHub Actions），`public` 目录不需要提交。

## Notes

- **SSH vs HTTPS**: 使用 SSH 方式（`git@github.com:...`）的优势是不需要每次推送都输入用户名和密码，配置好 SSH 密钥后会更方便。
- **GitHub Actions 推荐**: 推荐使用 GitHub Actions 自动部署，这样你只需要推送源代码，GitHub 会自动构建和部署，不需要手动管理 `public` 目录。
- **主题管理**: 主题作为 git submodule 添加，更新主题时使用 `git submodule update --remote`。
- **baseURL 配置**: 确保 `hugo.toml` 中的 `baseURL` 与你的 GitHub Pages URL 完全一致，否则可能导致资源加载失败。
- **自定义域名**: 如果要使用自定义域名，需要在仓库根目录添加 `CNAME` 文件，并在域名 DNS 设置中添加相应记录。
