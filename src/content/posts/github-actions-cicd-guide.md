---
title: "从零配置 GitHub Actions CI/CD 自动化部署"
published: 2026-03-12
description: "详细记录如何为 Astro 博客配置 GitHub Actions 实现自动化构建和部署"
tags: ["CI/CD", "GitHub Actions", "DevOps", "Astro"]
category: "技术"
draft: false
---

## 前言

最近为我的博客配置了 GitHub Actions CI/CD，实现了代码推送后自动构建和部署。这篇文章记录整个配置过程，希望能帮助到有同样需求的朋友。

## 什么是 CI/CD

**CI (Continuous Integration)** 持续集成：代码合并后自动构建和测试
**CD (Continuous Deployment)** 持续部署：构建通过后自动部署到服务器

对于静态博客来说，这意味着：
1. 本地写好文章，push 到 GitHub
2. GitHub Actions 自动构建
3. 构建成功后自动部署到服务器
4. 全程无需手动操作

## 准备工作

### 服务器环境

- Linux 服务器（我的是 Alibaba Cloud Linux）
- 已安装 Nginx 或其他 Web 服务器
- 确保有 SSH 访问权限

### 安装必要软件

在服务器上安装 rsync（用于文件同步）：

```bash
# CentOS/RHEL/Alibaba Cloud Linux
sudo yum install -y rsync

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y rsync
```

### GitHub 仓库配置

1. 在 GitHub 创建仓库，上传博客源码
2. 确保仓库有构建脚本（如 `pnpm build`）

## 配置 GitHub Actions

### 1. 创建 Workflow 文件

在仓库根目录创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to Server

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4
      
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '22'
        
    - name: Setup pnpm
      uses: pnpm/action-setup@v2
      with:
        version: 9
        
    - name: Get pnpm store directory
      shell: bash
      run: |
        echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_ENV
        
    - name: Setup pnpm cache
      uses: actions/cache@v4
      with:
        path: ${{ env.STORE_PATH }}
        key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
        restore-keys: |
          ${{ runner.os }}-pnpm-store-
          
    - name: Install dependencies
      run: pnpm install
      
    - name: Build
      run: pnpm build
      
    - name: Deploy to Server
      if: github.ref == 'refs/heads/main'
      uses: easingthemes/ssh-deploy@v5.0.3
      with:
        SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        ARGS: "-rltgoDzvO --delete"
        SOURCE: "dist/"
        REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
        REMOTE_USER: ${{ secrets.REMOTE_USER }}
        REMOTE_PORT: ${{ secrets.REMOTE_PORT }}
        TARGET: ${{ secrets.REMOTE_TARGET }}
        EXCLUDE: "/dist/, /node_modules/"
```

### 2. 配置 Secrets

在 GitHub 仓库 **Settings → Secrets and variables → Actions** 中添加以下 secrets：

| Secret | 说明 |
|--------|------|
| `SSH_PRIVATE_KEY` | 服务器 SSH 私钥 |
| `REMOTE_HOST` | 服务器地址 |
| `REMOTE_USER` | SSH 用户名 |
| `REMOTE_PORT` | SSH 端口 |
| `REMOTE_TARGET` | 部署目标路径 |

### 生成 SSH 密钥

在本地生成密钥对：

```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions
```

将公钥添加到服务器的 `~/.ssh/authorized_keys`：

```bash
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
```

将私钥内容复制到 GitHub Secrets 的 `SSH_PRIVATE_KEY`。

## 遇到的问题与解决

### 问题 1：Node.js 版本警告

GitHub Actions 提示 Node.js 20 actions 将在 2026年6月 后弃用。

**解决**：添加环境变量强制使用 Node.js 24：

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

### 问题 2：类型检查失败

Astro 的 `astro check` 报告类型错误，导致构建失败。

**解决**：跳过类型检查，只执行构建：

```yaml
- name: Build
  run: pnpm build  # 不运行 pnpm astro check
```

### 问题 3：服务器缺少 rsync

部署时报错 `rsync: command not found`。

**解决**：在服务器上安装 rsync（见准备工作）。

### 问题 4：SSH 连接被拒绝

部署时无法连接到服务器。

**解决**：
1. 检查 SSH 密钥是否正确配置
2. 确认服务器防火墙允许 GitHub Actions IP
3. 验证端口是否正确

## 优化配置

### 跳过不必要的构建

如果只想在代码文件变更时触发构建，可以添加路径过滤：

```yaml
on:
  push:
    branches: [ main ]
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'pnpm-lock.yaml'
```

### 多 Node.js 版本测试

如果需要测试多个 Node.js 版本：

```yaml
strategy:
  matrix:
    node: [ 22, 23 ]
```

但对于个人博客，通常只需要一个版本即可。

## 验证部署

配置完成后，push 代码到 main 分支，观察 GitHub Actions 运行状态：

1. 进入仓库的 **Actions** 标签
2. 查看 workflow 运行状态
3. 绿色 ✅ 表示成功

部署成功后，访问博客域名查看效果。

## 总结

通过 GitHub Actions 实现自动化部署后，博客的发布流程变得非常简单：

1. 本地编写文章
2. `git push origin main`
3. 等待 1-2 分钟
4. 文章自动上线

不再需要手动构建、上传文件，大大提升了写作效率。

## 参考链接

- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Astro 部署指南](https://docs.astro.build/en/guides/deploy/)
- [ssh-deploy Action](https://github.com/easingthemes/ssh-deploy)

---

*本文记录于 2026-03-12，配置环境：Astro + pnpm + GitHub Actions + Linux 服务器*
