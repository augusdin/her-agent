# GitHub CI/CD 自动部署指南

> 创建日期：2026-02-01
>
> 本文档记录了如何配置 GitHub Actions 实现代码自动构建并部署到服务器的完整流程。

## 目录

- [部署架构](#部署架构)
- [前置条件](#前置条件)
- [配置步骤](#配置步骤)
  - [步骤 1: 生成 SSH 密钥对](#步骤-1-生成-ssh-密钥对)
  - [步骤 2: 配置 GitHub Secrets](#步骤-2-配置-github-secrets)
  - [步骤 3: 配置服务器 Docker 登录](#步骤-3-配置服务器-docker-登录)
  - [步骤 4: 更新服务器 docker-compose.yml](#步骤-4-更新服务器-docker-composeyml)
- [遇到的问题与解决方案](#遇到的问题与解决方案)
- [验证部署](#验证部署)
- [常用命令](#常用命令)

---

## 部署架构

```
本地代码 → git push → GitHub main 分支
                           ↓
                    GitHub Actions 触发
                           ↓
                    构建 Docker 镜像
                           ↓
                    推送到 ghcr.io/augusdin/her-agent:server_latest
                           ↓
                    SSH 连接到 xiaozhi-self 服务器
                           ↓
                    拉取新镜像并重启容器
                           ↓
                    健康检查验证（最多 60 秒）
                           ↓
                    部署完成 ✅
```

---

## 前置条件

1. GitHub 仓库：`augusdin/her-agent`
2. 服务器：`xiaozhi-self`（IP: `107.173.38.186`）
3. 服务器已安装 Docker 和 docker-compose
4. 本地已配置 SSH 访问服务器（`ssh xiaozhi-self`）

---

## 配置步骤

### 步骤 1: 生成 SSH 密钥对

在本地生成专用于 CI/CD 部署的 SSH 密钥：

```bash
# 生成密钥对（不设置密码）
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""

# 将公钥添加到服务器
cat ~/.ssh/github_actions_deploy.pub | ssh xiaozhi-self 'cat >> ~/.ssh/authorized_keys'

# 验证公钥已添加
ssh xiaozhi-self 'cat ~/.ssh/authorized_keys | grep github-actions-deploy'
```

### 步骤 2: 配置 GitHub Secrets

访问 GitHub 仓库设置页面：`https://github.com/augusdin/her-agent/settings/secrets/actions`

添加以下 **Repository secrets**：

| Secret 名称 | 值 | 说明 |
|------------|------|------|
| `TOKEN` | GitHub Personal Access Token | 用于登录 ghcr.io 推送镜像 |
| `SSH_HOST` | `107.173.38.186` | 服务器 IP 地址 |
| `SSH_USER` | `root` | SSH 用户名 |
| `SSH_PRIVATE_KEY` | SSH 私钥内容 | 用于 SSH 连接服务器 |

#### 获取 SSH 私钥内容

```bash
# 在本地执行，复制输出内容到 SSH_PRIVATE_KEY
cat ~/.ssh/github_actions_deploy
```

**重要**：复制时必须包含完整内容，包括：
```
-----BEGIN OPENSSH PRIVATE KEY-----
...密钥内容...
-----END OPENSSH PRIVATE KEY-----
```

#### 获取 GitHub Personal Access Token

1. 访问：https://github.com/settings/tokens/new
2. 配置：
   - **Note**: `ghcr-deploy`
   - **Expiration**: 选择合适的过期时间
   - **Select scopes**: 勾选 `write:packages`（会自动勾选 `read:packages`）
3. 点击 **Generate token**
4. 复制生成的 token（以 `ghp_` 开头）

### 步骤 3: 配置服务器 Docker 登录

服务器需要登录 ghcr.io 才能拉取私有镜像：

```bash
# 将 YOUR_TOKEN 替换为你的 GitHub Personal Access Token
ssh xiaozhi-self 'echo "YOUR_TOKEN" | docker login ghcr.io -u augusdin --password-stdin'
```

成功后会显示：`Login Succeeded`

### 步骤 4: 更新服务器 docker-compose.yml

将服务器上的 docker-compose.yml 镜像地址改为你自己的镜像：

```bash
ssh xiaozhi-self 'cat > /opt/xiaozhi-deployment/her-agent/docker-compose.yml << EOF
version: "3"
services:
  her-agent:
    image: ghcr.io/augusdin/her-agent:server_latest
    container_name: her-agent
    restart: always
    security_opt:
      - seccomp:unconfined
    environment:
      - TZ=Asia/Shanghai
    ports:
      - "8000:8000"
      - "8003:8003"
    volumes:
      - ./data:/opt/xiaozhi-esp32-server/data
      - ./models/SenseVoiceSmall/model.pt:/opt/xiaozhi-esp32-server/models/SenseVoiceSmall/model.pt
      - ./tmp:/opt/xiaozhi-esp32-server/tmp
EOF'
```

---

## 遇到的问题与解决方案

### 问题 1: Build 成功但 Deploy 失败 - "Password required"

**错误信息**：
```
Error: Password required
```

**原因**：GitHub Secrets 中没有添加 `TOKEN`，导致登录 ghcr.io 时密码为空。

**解决方案**：
1. 访问 https://github.com/augusdin/her-agent/settings/secrets/actions
2. 添加名为 `TOKEN` 的 secret，值为 GitHub Personal Access Token

### 问题 2: Deploy 失败 - "ssh: no key found"

**错误信息**：
```
ssh.ParsePrivateKey: ssh: no key found
```

**原因**：`SSH_PRIVATE_KEY` secret 的内容格式不正确或不完整。

**解决方案**：
1. 在本地执行 `cat ~/.ssh/github_actions_deploy` 获取完整私钥
2. 确保复制时包含 `-----BEGIN OPENSSH PRIVATE KEY-----` 和 `-----END OPENSSH PRIVATE KEY-----`
3. 更新 GitHub Secrets 中的 `SSH_PRIVATE_KEY`

### 问题 3: Deploy 失败 - "publickey authentication failed"

**错误信息**：
```
ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain
```

**原因**：服务器上没有对应的公钥。

**解决方案**：
```bash
# 将公钥添加到服务器
cat ~/.ssh/github_actions_deploy.pub | ssh xiaozhi-self 'cat >> ~/.ssh/authorized_keys'
```

### 问题 4: 健康检查失败但服务实际正常

**错误信息**：
```
curl: (56) Recv failure
Process exited with status 56
```

**原因**：服务启动需要时间，固定的 `sleep 15` 不够。

**解决方案**：修改 workflow 中的健康检查逻辑，使用重试机制（最多等待 60 秒）：

```yaml
script: |
  cd /opt/xiaozhi-deployment/her-agent
  docker-compose pull
  docker-compose up -d
  echo "Waiting for service to start..."
  for i in {1..12}; do
    sleep 5
    if curl -s -o /dev/null -w "%{http_code}" http://localhost:8003/xiaozhi/ota/ | grep -q "200"; then
      echo "Service is ready!"
      docker ps | grep her-agent
      exit 0
    fi
    echo "Attempt $i: Service not ready yet..."
  done
  echo "Service health check failed after 60 seconds"
  docker ps | grep her-agent
  docker logs --tail 50 her-agent
  exit 1
```

---

## 验证部署

### 1. 检查 GitHub Actions 状态

访问：https://github.com/augusdin/her-agent/actions

查看最新的 workflow 运行状态，确保 Build 和 Deploy 都成功。

### 2. 检查服务器容器状态

```bash
ssh xiaozhi-self 'docker ps | grep her-agent'
```

应该看到类似输出：
```
85d96a7487fb   ghcr.io/augusdin/her-agent:server_latest   "python app.py"   Up X minutes   0.0.0.0:8000->8000/tcp, 0.0.0.0:8003->8003/tcp   her-agent
```

### 3. 检查健康状态

```bash
ssh xiaozhi-self 'curl -s http://localhost:8003/xiaozhi/ota/'
```

### 4. 使用测试页面验证

访问：http://107.173.38.186:8003/test/test_page.html

1. OTA 地址填入：`http://107.173.38.186:8003/xiaozhi/ota/`
2. 点击"连接"按钮
3. 发送文本消息测试

---

## 常用命令

### 服务器操作

```bash
# 查看容器状态
ssh xiaozhi-self 'docker ps | grep her-agent'

# 查看容器日志
ssh xiaozhi-self 'docker logs -f her-agent'

# 查看最近 100 行日志
ssh xiaozhi-self 'docker logs --tail 100 her-agent'

# 重启容器
ssh xiaozhi-self 'cd /opt/xiaozhi-deployment/her-agent && docker-compose restart'

# 停止容器
ssh xiaozhi-self 'cd /opt/xiaozhi-deployment/her-agent && docker-compose down'

# 启动容器
ssh xiaozhi-self 'cd /opt/xiaozhi-deployment/her-agent && docker-compose up -d'

# 手动拉取最新镜像
ssh xiaozhi-self 'cd /opt/xiaozhi-deployment/her-agent && docker-compose pull'
```

### 手动触发 CI/CD

```bash
# 通过 GitHub API 触发（需要替换 YOUR_TOKEN）
curl -X POST \
  -H "Authorization: token YOUR_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/augusdin/her-agent/actions/workflows/docker-image.yml/dispatches \
  -d '{"ref":"main"}'
```

### 查看 CI/CD 状态

```bash
# 查看最新 workflow 运行状态（需要替换 YOUR_TOKEN）
curl -s -H "Authorization: token YOUR_TOKEN" \
  "https://api.github.com/repos/augusdin/her-agent/actions/runs?per_page=1" | \
  jq '.workflow_runs[0] | {id, status, conclusion, name}'
```

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `.github/workflows/docker-image.yml` | GitHub Actions workflow 配置 |
| `Dockerfile-server` | Docker 镜像构建文件 |
| 服务器 `/opt/xiaozhi-deployment/her-agent/docker-compose.yml` | 服务器 docker-compose 配置 |
| 服务器 `/opt/xiaozhi-deployment/her-agent/data/.config.yaml` | 服务配置文件 |

---

## 注意事项

1. **SSH 密钥安全**：部署用的 SSH 私钥只用于此 CI/CD，不要泄露或用于其他用途
2. **Token 安全**：GitHub Token 有权限推送镜像，请妥善保管
3. **配置文件保留**：服务器上的 `data/.config.yaml` 和模型文件不会被 CI/CD 覆盖
4. **首次部署**：配置完成后需要手动推送一次代码触发 CI/CD

---

## HTTPS 配置（使用 Caddy）

为了让测试页面能够使用麦克风功能，需要配置 HTTPS。浏览器的安全策略要求 `getUserMedia` API 只能在安全上下文（HTTPS 或 localhost）中使用。

### 前置条件

1. 一个域名（如 `hermind.top`）
2. 域名 DNS 的 A 记录指向服务器 IP

### 步骤 1: 安装 Caddy

Caddy 是一个自动 HTTPS 的 Web 服务器，会自动申请和续期 Let's Encrypt 证书。

```bash
ssh xiaozhi-self 'apt update && apt install -y debian-keyring debian-archive-keyring apt-transport-https curl'
ssh xiaozhi-self 'curl -1sLf "https://dl.cloudsmith.io/public/caddy/stable/gpg.key" | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg'
ssh xiaozhi-self 'curl -1sLf "https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt" | tee /etc/apt/sources.list.d/caddy-stable.list'
ssh xiaozhi-self 'apt update && apt install -y caddy'
```

### 步骤 2: 配置 Caddy

创建 Caddyfile 配置反向代理：

```bash
ssh xiaozhi-self 'cat > /etc/caddy/Caddyfile << '\''EOF'\''
hermind.top {
    # WebSocket proxy for device communication
    @websocket {
        path /xiaozhi/v1/*
    }
    handle @websocket {
        reverse_proxy localhost:8000
    }

    # HTTP API proxy (OTA, vision, test page, etc.)
    handle {
        reverse_proxy localhost:8003
    }
}
EOF'
```

**说明**：
- `hermind.top` - 替换为你的域名
- `/xiaozhi/v1/*` - WebSocket 路径，代理到端口 8000
- 其他路径代理到端口 8003（HTTP API、测试页面等）

### 步骤 3: 启动 Caddy

```bash
# 验证配置
ssh xiaozhi-self 'caddy validate --config /etc/caddy/Caddyfile'

# 重启 Caddy 服务
ssh xiaozhi-self 'systemctl restart caddy'

# 查看状态（应该能看到证书申请成功的日志）
ssh xiaozhi-self 'systemctl status caddy'
```

Caddy 会自动：
- 申请 Let's Encrypt 证书
- 配置 HTTPS（端口 443）
- HTTP 自动重定向到 HTTPS
- 证书自动续期

### 步骤 4: 更新服务配置

修改服务器上的 `data/.config.yaml`，将 WebSocket 和视觉分析地址改为 HTTPS：

```bash
# 更新 websocket 地址
ssh xiaozhi-self "sed -i 's|websocket: ws://.*|websocket: wss://hermind.top/xiaozhi/v1/|' /opt/xiaozhi-deployment/her-agent/data/.config.yaml"

# 更新 vision_explain 地址
ssh xiaozhi-self "sed -i 's|vision_explain: http://.*|vision_explain: https://hermind.top/mcp/vision/explain|' /opt/xiaozhi-deployment/her-agent/data/.config.yaml"

# 重启容器使配置生效
ssh xiaozhi-self 'cd /opt/xiaozhi-deployment/her-agent && docker-compose restart'
```

### 步骤 5: 验证 HTTPS

```bash
# 测试 HTTPS OTA 接口
curl -s https://hermind.top/xiaozhi/ota/

# 应该返回类似：
# OTA接口运行正常，向设备发送的websocket地址是：wss://hermind.top/xiaozhi/v1/
```

### HTTPS 服务地址

配置完成后，可以使用以下地址：

| 服务 | 地址 |
|------|------|
| 测试页面 | https://hermind.top/test/test_page.html |
| OTA 接口 | https://hermind.top/xiaozhi/ota/ |
| WebSocket | wss://hermind.top/xiaozhi/v1/ |
| 视觉分析 | https://hermind.top/mcp/vision/explain |

### 为什么需要 HTTPS

浏览器的安全策略（Secure Context）要求某些敏感 API 只能在安全上下文中使用：

| 来源 | 是否安全上下文 | 麦克风可用 |
|------|---------------|-----------|
| `https://任何地址` | ✅ 是 | ✅ 可用 |
| `http://localhost` | ✅ 是 | ✅ 可用 |
| `http://其他地址` | ❌ 否 | ❌ 不可用 |

所以通过 HTTP 访问远程服务器的测试页面时，会出现 "音频初始化错误: Cannot read properties of undefined (reading 'getUserMedia')" 错误。

### Caddy 常用命令

```bash
# 查看 Caddy 状态
ssh xiaozhi-self 'systemctl status caddy'

# 重启 Caddy
ssh xiaozhi-self 'systemctl restart caddy'

# 查看 Caddy 日志
ssh xiaozhi-self 'journalctl -u caddy -f'

# 重新加载配置（不中断服务）
ssh xiaozhi-self 'caddy reload --config /etc/caddy/Caddyfile'

# 查看证书信息
ssh xiaozhi-self 'caddy list-certificates'
```

### 配置文件位置

| 文件 | 说明 |
|------|------|
| `/etc/caddy/Caddyfile` | Caddy 配置文件 |
| `~/.local/share/caddy/` | 证书存储位置 |
| 服务器 `data/.config.yaml` | 服务配置（websocket、vision_explain 地址） |

### 注意事项

1. **域名解析**：确保域名 DNS 已正确解析到服务器 IP，否则证书申请会失败
2. **端口开放**：确保服务器防火墙开放 80 和 443 端口
3. **配置持久化**：`data/.config.yaml` 中的 HTTPS 配置是部署环境相关的，不会被 CI/CD 覆盖
4. **证书续期**：Caddy 会自动续期证书，无需手动操作
