# wg-easy amd64 安全部署模板

这个仓库用于部署和持续构建 `wg-easy/wg-easy` 的 `linux/amd64` 镜像。

它只保存部署模板、GitHub Actions 工作流和上游 commit 记录，不保存真实 `.env`、WireGuard 私钥、客户端配置、管理员密码或 GitHub token。

## 当前镜像

默认镜像：

```text
ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
```

当前已发布的固定版本：

```text
ghcr.io/leolabtec/wg-easy-source-amd64:source-78b2838-amd64
```

对应官方源码：

```text
https://github.com/wg-easy/wg-easy/commit/78b283849895b35ad45c6f83fa01228c0f1a6774
```

Release：

```text
https://github.com/leolabtec/wg-easy-source-amd64/releases/tag/source-78b2838-amd64
```

## 安全边界

公网只应该开放 WireGuard UDP：

```text
51820/udp
```

Web UI 端口不要暴露到公网：

```text
51821/tcp
```

本仓库的 `docker-compose.yml` 已把 Web UI 绑定到 `${LAN_BIND_IP}:51821`，而不是 `0.0.0.0:51821`。这意味着 Web UI 只监听你指定的内网地址。

推荐部署形态：

```text
公网 / 路由器 / 云防火墙
  只放行 51820/udp

内网
  允许访问 服务器内网IP:51821
```

不要把 `network_mode: host` 加进去。当前 bridge 网络模式更容易控制端口暴露面。

## GHCR 可见性

GitHub 仓库 public 不等于 GHCR package public。

如果服务器执行：

```bash
docker compose pull
```

出现：

```text
unauthorized
```

说明 `ghcr.io/leolabtec/wg-easy-source-amd64` 这个 container package 仍是 private。

解决方式二选一：

1. 把 GHCR package 改成 public。
2. 在服务器上 `docker login ghcr.io` 后再 pull。

package 页面：

```text
https://github.com/users/leolabtec/packages/container/package/wg-easy-source-amd64
```

进入 `Package settings` 后改的是 package visibility，不是 repository visibility。

## 部署

在服务器创建目录：

```bash
mkdir -p /opt/wg-easy
cd /opt/wg-easy
```

准备文件：

```bash
wget -O docker-compose.yml https://raw.githubusercontent.com/leolabtec/wg-easy-source-amd64/main/docker-compose.yml
wget -O .env.example https://raw.githubusercontent.com/leolabtec/wg-easy-source-amd64/main/.env.example
cp .env.example .env
```

编辑 `.env`：

```bash
nano .env
```

至少修改 `LAN_BIND_IP` 为服务器内网 IP，例如：

```text
LAN_BIND_IP=192.168.1.10
```

默认跟随自动构建的最新 amd64 镜像：

```text
WG_EASY_IMAGE=ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
```

如果你要固定到审计过的版本，改成：

```text
WG_EASY_IMAGE=ghcr.io/leolabtec/wg-easy-source-amd64:source-78b2838-amd64
```

检查最终配置：

```bash
docker compose config
```

你应该看到：

```text
51820:51820/udp
192.168.1.10:51821:51821/tcp
```

启动：

```bash
docker compose up -d
```

查看状态：

```bash
docker compose ps
docker compose logs -f
```

## Web UI 初始化

只从内网访问：

```text
http://服务器内网IP:51821
```

首次进入 Web UI 后创建管理员账号。

建议：

- 使用高强度管理员密码。
- 启用 2FA。
- 不要把一次性链接用于公网场景。
- 不要启用 OAuth 自动注册，除非你严格限制允许的域名。
- 不要把 metrics 暴露到公网。

## 防火墙

UFW 示例：

```bash
ufw allow 51820/udp
ufw allow from 192.168.1.0/24 to any port 51821 proto tcp
ufw deny 51821/tcp
```

路由器或云防火墙只做公网端口转发：

```text
51820/udp -> 服务器IP:51820/udp
```

不要转发：

```text
51821/tcp
```

## 更新

使用 `latest-amd64` 时：

```bash
cd /opt/wg-easy
docker compose pull
docker compose up -d
```

使用固定 tag 时，先改 `.env` 的 `WG_EASY_IMAGE`，再执行：

```bash
docker compose pull
docker compose up -d
```

## GHCR private 时的临时部署

如果 package 还没改 public，可以从 Release 下载 tarball：

```bash
wget https://github.com/leolabtec/wg-easy-source-amd64/releases/download/source-78b2838-amd64/wg-easy-source-78b2838-amd64.tar.gz
docker load < wg-easy-source-78b2838-amd64.tar.gz
docker compose up -d
```

也可以服务器登录 GHCR：

```bash
docker login ghcr.io -u leolabtec
docker compose pull
docker compose up -d
```

不要把个人 GitHub token 写进仓库文件。

## 自动构建

工作流：

```text
.github/workflows/build-upstream.yml
```

触发方式：

- 每周一 UTC 03:23 自动检查官方仓库。
- 支持手动 `workflow_dispatch`。

构建逻辑：

1. 查询 `wg-easy/wg-easy` 当前 HEAD。
2. 和 `.github/upstream-wg-easy.sha` 记录的 commit 对比。
3. 如果有新 commit，构建 `linux/amd64` 镜像。
4. 推送到 GHCR：

```text
ghcr.io/leolabtec/wg-easy-source-amd64:latest-amd64
ghcr.io/leolabtec/wg-easy-source-amd64:source-<short-sha>-amd64
```

5. 创建 GitHub Release，并上传 Docker image tarball 和 sha256 文件。
6. 更新 `.github/upstream-wg-easy.sha`。

工作流只使用 GitHub Actions 内置的 `GITHUB_TOKEN`。如果推送 GHCR 时报 `403 Forbidden`，到 package settings 里给这个仓库添加 Actions 写权限。

## 本仓库不包含的内容

不会提交：

- `.env`
- WireGuard 私钥
- 客户端配置
- Web UI 管理员密码
- GitHub token
- 服务器 IP 以外的真实敏感配置

真实运行数据在服务器 Docker volume：

```text
etc_wireguard:/etc/wireguard
```

这个 volume 里会有 WireGuard 配置和密钥，不要备份到公开位置。
