# 🌙 MoonEdge Blog

[![Deploy Status](https://github.com/CentCord/MoonEdge/actions/workflows/deploy.yml/badge.svg)](https://github.com/CentCord/MoonEdge/actions/workflows/deploy.yml)

基于 [Fuwari](https://github.com/saicaca/fuwari) 构建的 Astro 静态博客。

## 🚀 技术栈

- **框架**: [Astro](https://astro.build/) v4+
- **包管理器**: pnpm
- **部署**: GitHub Actions → 阿里云 ECS
- **域名**: https://www.moonedge.cn

## 📦 本地开发

```bash
# 安装依赖
pnpm install

# 启动开发服务器
pnpm dev

# 构建生产版本
pnpm build

# 预览构建结果
pnpm preview
```

## 🔄 CI/CD 工作流

### 自动部署流程

1. **Push 到 main 分支** → 触发构建和部署
2. **Pull Request 到 main** → 仅构建，不部署（用于预览）

### 工作流程详情

```
检出代码 → 设置 Node.js 20 → 安装 pnpm → 缓存依赖 → 安装依赖 → 构建 → SSH 部署
```

## 🔐 配置 Secrets

在 GitHub 仓库 **Settings → Secrets and variables → Actions** 中配置：

| Secret | 说明 | 示例 |
|--------|------|------|
| `SSH_PRIVATE_KEY` | 服务器 SSH 私钥 | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `REMOTE_HOST` | 服务器 IP | `8.138.251.181` |
| `REMOTE_USER` | SSH 用户名 | `root` |
| `REMOTE_TARGET` | 部署目标路径 | `/var/www/moonedge.cn` |
| `REMOTE_PORT` | SSH 端口（可选）| `22` |

## 📝 目录结构

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml      # GitHub Actions 配置
├── src/                    # 源代码
├── public/                 # 静态资源
├── dist/                   # 构建输出（自动部署）
├── astro.config.mjs        # Astro 配置
└── package.json
```

## 🛠️ 故障排查

### 部署失败

1. 检查 GitHub Actions 日志
2. 验证 Secrets 是否正确配置
3. 测试 SSH 连接：
   ```bash
   ssh -i ~/.ssh/github_actions root@8.138.251.181
   ```

### 构建失败

1. 确保 `pnpm-lock.yaml` 已提交到仓库
2. 检查 Node.js 版本兼容性

---

*Last updated: 2026-03-12*
