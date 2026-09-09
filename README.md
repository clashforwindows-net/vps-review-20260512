# VPS 自托管 Git 与 CI/CD 一体化实战：Gitea / Woodpecker CI / GitLab 全对比

> 代码放 GitHub/GitLab 云端当然方便，但私有仓库有合规、隐私与成本考量。本仓库教你用一台 VPS 搭出「代码托管 + 镜像仓库 + CI/CD + Webhook 自动部署」的完整研发平台：Gitea 轻量托管、Woodpecker CI 流水线、GitLab CE 全家桶对比，含备份迁移、SSH 安全与高可用要点。

## 目录

- [为什么自托管 Git 服务](#为什么自托管-git-服务)
- [平台选型：Gitea vs GitLab CE vs Gogs](#平台选型gitea-vs-gitlab-ce-vs-gogs)
- [部署前准备：域名、证书与存储规划](#部署前准备域名证书与存储规划)
- [Gitea 极速部署（Docker Compose）](#gitea-极速部署docker-compose)
- [基础配置：SSH、HTTP 与反向代理](#基础配置sshhttp-与反向代理)
- [CI 流水线：接入 Woodpecker CI](#ci-流水线接入-woodpecker-ci)
- [Webhook 自动部署到服务器](#webhook-自动部署到服务器)
- [镜像仓库：随附 Registry 与 Harbor](#镜像仓库随附-registry-与-harbor)
- [GitLab CE 方案对比与迁移](#gitlab-ce-方案对比与迁移)
- [备份、恢复与实例迁移](#备份恢复与实例迁移)
- [安全加固：SSH 密钥、令牌与审计](#安全加固ssh-密钥令牌与审计)
- [性能调优与容量规划](#性能调优与容量规划)
- [常见问题 FAQ](#常见问题-faq)
- [相关资源与推荐入口](#相关资源与推荐入口)
- [免责声明](#免责声明)

## 为什么自托管 Git 服务

| 考量 | 公有云托管 | 自托管 |
|------|-----------|--------|
| 私有仓库免费额度 | 有限 | 无限 |
| 代码合规（数据出境） | 受限 | 完全自主 |
| 单仓库大小限制 | 常见 1–2GB | 自己定 |
| CI 分钟数 | 免费额度少 | 用自己的 VPS，无限 |
| 维护成本 | 零 | 需要你（本仓库帮你降到最低） |
| 适合 | 开源/个人公开项目 | 商业代码、大文件、内网合规 |

## 平台选型：Gitea vs GitLab CE vs Gogs

| 维度 | Gitea | GitLab CE | Gogs |
|------|-------|-----------|------|
| 内存占用（空载） | ~300–500M | ~2–4G | ~200–300M |
| 内置 CI | 需插件（Woodpecker） | 内置强大 | 无 |
| 安装复杂度 | 低 | 高 | 最低 |
| 功能面 | 够用（PR/Issue/Actions 生态） | 全套 DevOps | 基础 |
| 升级活跃度 | 高 | 高 | 低 |
| 推荐场景 | **个人/小团队首选** | 需要一体化 DevOps 的中队 | 极简备份仓库 |

**结论**：1–10 人团队强烈推荐 Gitea + Woodpecker；需要开箱即用的完整 DevOps（含 SAST、容器扫描、Epics）再上 GitLab CE；Gogs 仅作归档冷仓。

## 部署前准备：域名、证书与存储规划

1. **域名**：`git.example.com`（Web）、`ci.example.com`（CI）、`git-ssh.example.com`（SSH 用，便于换端口）。
2. **证书**：acme.sh 签发 Let's Encrypt 泛域名证书，或交给反向代理（Caddy/Traefik）自动签。
3. **存储规划**（务必单独分区/磁盘）：

```bash
sudo mkdir -p /srv/git/{data,backup}
# /srv/git 建议独立挂载大容量数据盘，git 仓库目录建议 XFS/ ext4
df -h /srv/git
```

> 估算：1GB 仓库含 .git 历史约膨胀 1.5–2 倍；10 人团队两年数据量通常 20–100GB，硬盘按此规划并留 30% 余量。

## Gitea 极速部署（Docker Compose）

`/srv/git/docker-compose.yml`：

```yaml
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__server__DOMAIN=git.example.com
      - GITEA__server__SSH_DOMAIN=git-ssh.example.com
      - GITEA__server__ROOT_URL=https://git.example.com/
      - GITEA__server__SSH_PORT=2222
      - GITEA__database__DB_TYPE=sqlite3
    volumes:
      - /srv/git/data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "2222:22"     # SSH 走 2222，避免与系统 SSH 冲突
      - "3000:3000"   # Web（由反代转发，不直接暴露公网）
    restart: unless-stopped
```

启动并初始化：

```bash
cd /srv/git && docker compose up -d
# 访问 https://git.example.com 完成网页安装向导
# 注册第一个账号（即管理员）
```

### 常用运维命令

```bash
docker compose logs -f gitea          # 日志
docker compose exec gitea gitea admin user create --username dev1 --password 'xxx' --email dev1@example.com  # 命令行建用户
docker compose exec gitea gitea dump --file /data/backup/gitea-dump.zip   # 官方备份工具
```

## 基础配置：SSH、HTTP 与反向代理

### SSH 端口与密钥

容器内 22 映射宿主 2222，用户克隆地址为 `ssh://git@git-ssh.example.com:2222/owner/repo.git`。为兼容默认 `git@` 短格式，可配置 `~/.ssh/config`：

```sshconfig
Host git-ssh.example.com
    HostName git-ssh.example.com
    Port 2222
    User git
    IdentityFile ~/.ssh/id_ed25519
```

### Caddy 反代（自动 HTTPS）

```caddy
git.example.com {
    reverse_proxy 127.0.0.1:3000
}
ci.example.com {
    reverse_proxy 127.0.0.1:8000
}
```

同时配置 Git 的 LFS 与上传大小上限（`app.ini`）：

```ini
[server]
LFS_START_SERVER = true
LFS_MAX_FILE_SIZE = 2048
[repository]
MAX_CREATION_LIMIT = 0
```

## CI 流水线：接入 Woodpecker CI

Woodpecker 是轻量开源 CI，与 Gitea 深度集成。Compose 追加：

```yaml
  woodpecker-server:
    image: woodpeckerci/woodpecker-server:latest
    container_name: woodpecker-server
    environment:
      - WOODPECKER_OPEN=true
      - WOODPECKER_GITEA=true
      - WOODPECKER_GITEA_URL=http://gitea:3000
      - WOODPECKER_GITEA_CLIENT=你的OAuth客户端ID
      - WOODPECKER_GITEA_SECRET=你的OAuth密钥
      - WOODPECKER_HOST=https://ci.example.com
      - WOODPECKER_AGENT_SECRET=共享密钥(与agent一致)
    volumes:
      - /srv/git/woodpecker-server:/var/lib/woodpecker
    ports:
      - "8000:8000"
    restart: unless-stopped

  woodpecker-agent:
    image: woodpeckerci/woodpecker-agent:latest
    container_name: woodpecker-agent
    environment:
      - WOODPECKER_SERVER=woodpecker-server:9000
      - WOODPECKER_AGENT_SECRET=共享密钥
      - WOODPECKER_MAX_WORKFLOWS=2
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - woodpecker-server
    restart: unless-stopped
```

在 Gitea 后台「应用 → 创建 OAuth2 应用」填入上述 Client/Secret 即完成对接。

### 示例流水线 `.woodpecker.yml`

```yaml
when:
  event: push

steps:
  build:
    image: node:20
    commands:
      - npm ci
      - npm run build
      - tar -czf dist.tar.gz dist/
  notify:
    image: appleboy/drone-telegram
    settings:
      token: {from_secret: tg_token}
      to: {from_secret: tg_chat}
    when:
      status: [success, failure]
```

## Webhook 自动部署到服务器

「推送到 git 主干 → 服务器自动拉取并重启服务」是自托管最实用的一环：

### 方式一：Gitea 内建 Webhook + 简易部署脚本

服务器上放部署脚本 `/srv/deploy/app-deploy.sh`：

```bash
#!/usr/bin/env bash
set -e
cd /srv/apps/myapp
git pull --ff-only origin main
docker compose build --pull
docker compose up -d
curl -fsS -X POST "https://api.telegram.org/bot${TG_TOKEN}/sendMessage" \
  -d chat_id="${TG_CHAT}" -d text="✅ myapp 部署完成 $(date '+%F %T')"
```

Gitea 仓库 → 设置 → Web 钩子 → 添加 Gitea 类型钩子，URL 指向一个迷你接收端（如 `webhookd`、`adnanh/webhook`），触发时执行上述脚本。

### 方式二：Woodpecker 部署步骤（推荐，含回滚）

```yaml
  deploy:
    image: docker:cli
    commands:
      - docker compose -f /srv/apps/myapp/docker-compose.yml up -d --build
    volumes:
      - /srv/apps/myapp:/srv/apps/myapp
      - /var/run/docker.sock:/var/run/docker.sock
    when:
      branch: main
      event: push
```

> 部署三板斧：先打镜像 tag（commit sha）、部署后健康检查（curl /healthz）、失败自动回滚上一 tag——三行脚本就能让上线基本无感。

## 镜像仓库：随附 Registry 与 Harbor

Gitea 自带容器镜像仓库（`gitea.example.com/owner/image`），开箱即用：

```bash
# 登录与推送
docker login git.example.com
docker tag myapp:latest git.example.com/owner/myapp:1.0.0
docker push git.example.com/owner/myapp:1.0.0
```

规模化/多团队时上 Harbor（带漏洞扫描与 RBAC）：

```bash
# Harbor 需求：2C4G 起，依赖 PostgreSQL/Redis，用其官方 installer 部署
```

## GitLab CE 方案对比与迁移

需要一体化 DevOps 时 GitLab CE 是 Gitea 的「大号」替代。Omnibus 一键包安装：

```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo GITLAB_ROOT_PASSWORD='强密码' EXTERNAL_URL='https://gitlab.example.com' apt install -y gitlab-ce
```

| 能力 | Gitea+Woodpecker | GitLab CE |
|------|-----------------|-----------|
| 资源占用 | 低（合计 1G 内） | 高（建议 4G+） |
| 内置 CI 生态 | Woodpecker（兼容 GitHub Actions 语法习惯） | 自家 CI/CD（功能全） |
| 安全扫描 | 无内置 | 内置 SAST/DAST（CE 部分受限） |
| 维护负担 | 低 | 中高（升级需谨慎） |

迁移路径：两端都支持 `git clone --mirror` + 推送，Issue/PR 可借 `gitlab/github` 类工具迁移，Wiki 手动搬。

## 备份、恢复与实例迁移

### 备份（Gitea 官方 dump）

```bash
docker compose exec gitea gitea dump --file /data/backup/gitea-dump-$(date +%F).zip
# 备份内容：git 仓库、数据库(sqlite)、LFS、附件、配置
```

异地同步（推荐对象存储或第二台 VPS）：

```bash
rclone copy /srv/git/backup remote:git-backup/ --transfers 4
```

### 恢复演练（每季度一次）

```bash
docker compose exec gitea gitea restore --from /data/backup/gitea-dump-xxx.zip
```

### 完整迁移到新 VPS

1. 新机装同版本 Gitea（版本必须 ≥ 旧版本，sqlite 向后兼容性有限）。
2. rsync `/srv/git/data` 全量过去（先停容器保证一致）：`rsync -a --delete /srv/git/data/ user@新机:/srv/git/data/`。
3. 新机 `docker compose up -d`，改 DNS 即完成。

## 安全加固：SSH 密钥、令牌与审计

1. **全员 ed25519 密钥，禁用密码登录 Git**；Gitea 管理面板可强制「仅 SSH 密钥」。
2. **令牌最小权限**：CI 用的 access token 只授「仓库读写」；部署密钥用 Deploy Key（只读单仓库）。
3. **两因素认证**：开启 TOTP，管理员账号强制。
4. **审计**：定期导出操作日志；敏感仓库加「受保护分支」，仅 Maintainer 可合并。
5. **SSH 与 Web 分离暴露面**：git SSH 只开 2222 并限来源 IP；Web 走反代 + 防火墙。
6. **容器更新**：Gitea/Woodpecker 镜像每月升级，关注官方安全通告。

## 性能调优与容量规划

```ini
# app.ini 调优参考
[server]
START_SSH_SERVER = true

[repository]
DEFAULT_PRIVATE = private

[database]
DB_TYPE = sqlite3        # 10 人以下够用；更大团队换 PostgreSQL
```

| 规模 | 建议配置 | 数据库 | 说明 |
|------|---------|--------|------|
| ≤5 人 | 2C2G + 50G | SQLite | 默认即可 |
| 5–30 人 | 4C4G + 200G | PostgreSQL | 加 Woodpecker agent ×2 |
| 30+ 人或 CI 重 | 4C8G+ + 500G | PostgreSQL | 拆分 CI 到独立 agent 机 |

## 常见问题 FAQ

**Q: Gitea 推送大文件卡顿？**
A: 启用 Git LFS 并把 LFS 放独立存储路径；HTTP 上传大小上限调大；确认反代没限制 body 大小。

**Q: CI 构建很慢？**
A: 先看瓶颈是拉镜像还是编译；加 agent 并发、用缓存卷挂 `~/.npm` 或 `~/.cache`、构建依赖用多阶段镜像。

**Q: Woodpecker 收不到 Gitea 事件？**
A: 检查 OAuth 回调地址是否 `https://ci.example.com/authorize`；Gitea 与 Woodpecker 网络是否互通；Webhook 用内网地址 `http://gitea:3000` 时确认 `WOODPECKER_GITEA_URL` 同步。

**Q: 数据库选 SQLite 还是 PostgreSQL？**
A: 个人/小团队 SQLite 零维护；一旦并发写多（多人同时开 PR/跑 CI 上报）就换 PostgreSQL，迁移工具官方提供。

**Q: 如何让外网同事也能用？**
A: 反代 + HTTPS 即可公网访问；建议开启注册邀请制，或用 OAuth 对接企业微信/GitHub 登录。

**Q: 磁盘被 .git 撑爆？**
A: 开仓库级 `git gc` 定时任务；教育团队用 LFS 管理二进制；`git count-objects -vH` 排查大仓库。

**Q: 与 GitHub Actions 语法兼容吗？**
A: Woodpecker 语法与 Actions 相似但不同（基于 pipeline 步骤）；可用 `woodpecker` 官方迁移工具辅助，复杂 workflow 需手改。

## 相关资源与推荐入口

- https://vpsvip.net - VPSVIP 官网（研发型 VPS 选型与优惠）
- https://clashvip.net - ClashVIP 官网
- https://nav.clashvip.net - ClashVIP 精选导航
- https://clashhub.net - ClashHub 社区
- https://bbs.clashhub.net - ClashHub 论坛
- https://clash-for-windows.net - Clash for Windows 官方网站
- https://docs.gitea.com - Gitea 官方文档
- https://woodpecker-ci.org - Woodpecker CI 官方文档
- https://about.gitlab.com/install - GitLab CE 安装文档

## 免责声明

1. 本仓库内容仅供技术学习与信息参考。
2. 自托管服务请自行做好备份与安全加固，数据无价。
3. 请遵守所在地法律法规与软件许可协议。
4. 第三方脚本与镜像使用前请自行审查。

## 许可证

MIT License

---
更新时间：2026-09-09
