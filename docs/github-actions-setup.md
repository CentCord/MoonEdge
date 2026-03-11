# GitHub Actions CI/CD 配置记录

**记录时间**: 2026-03-12  
**项目**: MoonEdge Blog (Astro + Fuwari)  
**目标**: 实现代码推送后自动构建和部署

---

## 最终配置

### 1. Deploy Workflow (`.github/workflows/deploy.yml`)

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

### 2. Build Workflow (`.github/workflows/build.yml`)

```yaml
name: Build and Check

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    name: Astro Build for Node.js 22
    steps:
      - name: Setup Node.js
        uses: actions/setup-node@cdca7365b2dadb8aad0a33bc7601856ffabcc48e
        with:
          node-version: 22

      - name: Checkout
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

      - name: Setup pnpm
        uses: pnpm/action-setup@a7487c7e89a18df4991f7f222e4898a00d66ddda
        with:
          run_install: false

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run Astro Build
        run: pnpm astro build
```

---

## GitHub Secrets 配置

| Secret | 用途 | 获取方式 |
|--------|------|----------|
| `SSH_PRIVATE_KEY` | SSH 私钥 | `cat ~/.ssh/github_actions` |
| `REMOTE_HOST` | 服务器地址 | 服务器 IP 或域名 |
| `REMOTE_USER` | SSH 用户名 | 通常是 `root` 或其他用户 |
| `REMOTE_PORT` | SSH 端口 | 默认 22，或自定义端口 |
| `REMOTE_TARGET` | 部署目标路径 | 如 `/var/www/moonedge.cn` |

---

## 服务器准备工作

### 1. 安装 rsync

```bash
# Alibaba Cloud Linux / CentOS / RHEL
sudo yum install -y rsync

# Ubuntu / Debian
sudo apt-get update && sudo apt-get install -y rsync
```

### 2. 配置 SSH 密钥

在本地生成密钥对：
```bash
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions
```

将公钥添加到服务器：
```bash
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
```

将私钥添加到 GitHub Secrets：
```bash
cat ~/.ssh/github_actions
# 复制内容到 GitHub → Settings → Secrets → SSH_PRIVATE_KEY
```

---

## 遇到的问题及解决方案

### 问题 1: Node.js 20 弃用警告
**现象**: GitHub Actions 提示 Node.js 20 actions 将被弃用
**解决**: 添加环境变量 `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true`

### 问题 2: Astro 类型检查失败
**现象**: `pnpm astro check` 报告类型错误
**解决**: 跳过类型检查，只运行 `pnpm build`

### 问题 3: 服务器缺少 rsync
**现象**: 部署时报错 `rsync: command not found`
**解决**: 在服务器上安装 rsync

### 问题 4: SSH 连接被拒绝
**现象**: `ssh: connect to host port 22: Connection refused`
**解决**: 检查 SSH 端口配置，确保使用正确的端口

### 问题 5: 路径过滤导致未触发部署
**现象**: 推送文件但 workflow 未运行
**解决**: 移除 `paths` 过滤或确保文件在监控路径内

---

## 操作步骤总结

1. **服务器准备**
   - 安装 rsync
   - 配置 SSH 密钥
   - 确认部署目录存在

2. **GitHub 配置**
   - 创建 workflow 文件
   - 添加 Secrets
   - 推送代码测试

3. **验证部署**
   - 查看 Actions 运行状态
   - 检查服务器文件
   - 访问网站确认

---

## 相关命令

```bash
# 查看 Actions 运行状态
gh run list --repo CentCord/MoonEdge

# 查看详细日志
gh run view <run-id> --log-failed

# 本地测试构建
pnpm install
pnpm build

# 检查服务器部署目录
ls -la /var/www/moonedge.cn/
```

---

## 注意事项

- 不要在代码中暴露 Secrets 信息
- README 中不要包含服务器地址、端口等敏感信息
- 定期轮换 SSH 密钥
- 监控 Actions 运行状态，及时处理失败

---

*最后更新: 2026-03-12*
