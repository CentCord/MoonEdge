# GitHub Actions 配置说明

## 自动部署工作流

### 触发条件
- `push` 到 `main` 分支
- `pull_request` 到 `main` 分支（仅构建，不部署）

### 工作流程
1. **检出代码** - 从 GitHub 拉取最新代码
2. **设置 Node.js 20** - 安装 Node.js 环境
3. **设置 pnpm** - 安装 pnpm 包管理器
4. **缓存依赖** - 加速后续构建
5. **安装依赖** - `pnpm install`
6. **构建** - `pnpm build`
7. **部署** - 通过 SSH 将 `dist/` 目录上传到服务器

---

## 需要配置的 Secrets

在 GitHub 仓库的 **Settings → Secrets and variables → Actions** 中添加以下 secrets：

| Secret 名称 | 说明 | 示例值 |
|------------|------|--------|
| `SSH_PRIVATE_KEY` | 服务器 SSH 私钥 | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `REMOTE_HOST` | 服务器 IP 或域名 | `8.138.251.181` |
| `REMOTE_USER` | SSH 用户名 | `root` |
| `REMOTE_PORT` | SSH 端口（可选，默认22） | `22` |
| `REMOTE_TARGET` | 服务器上的目标路径 | `/var/www/moonedge.cn` |

---

## 生成 SSH 密钥对

在本地或服务器上执行：

```bash
# 生成密钥对（不要设置密码，直接回车）
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions

# 查看私钥（复制到 GitHub Secrets）
cat ~/.ssh/github_actions

# 查看公钥（添加到服务器的 authorized_keys）
cat ~/.ssh/github_actions.pub
```

---

## 服务器配置

### 1. 添加公钥到服务器

```bash
# 在服务器上执行
echo "你的公钥内容" >> ~/.ssh/authorized_keys
```

### 2. 确保目标目录存在

```bash
# 创建目标目录（根据你的实际配置调整）
mkdir -p /var/www/moonedge.cn

# 设置权限
chown -R $USER:$USER /var/www/moonedge.cn
```

### 3. Nginx 配置示例

```nginx
server {
    listen 80;
    server_name moonedge.cn www.moonedge.cn;
    
    root /var/www/moonedge.cn;
    index index.html;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## 手动触发部署

如果需要在 GitHub 网页上手动触发：

1. 进入仓库的 **Actions** 标签
2. 选择 **Deploy to Server** 工作流
3. 点击 **Run workflow**

---

## 故障排查

### 部署失败

1. **检查 Secrets 是否正确配置**
   - 私钥是否包含换行符
   - 路径是否正确

2. **检查 SSH 连接**
   ```bash
   # 本地测试
   ssh -i ~/.ssh/github_actions root@8.138.251.181
   ```

3. **检查目录权限**
   ```bash
   # 在服务器上
   ls -la /var/www/
   ```

4. **查看 Action 日志**
   - GitHub → Actions → 选择失败的运行 → 查看详细日志

---

## 可选：添加状态徽章

在 `README.md` 中添加：

```markdown
![Deploy Status](https://github.com/CentCord/MoonEdge/actions/workflows/deploy.yml/badge.svg)
```
