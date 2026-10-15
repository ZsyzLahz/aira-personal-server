# Aira Personal Server — Docker 自动构建部署

本仓库跟踪上游 [mason173/aira-browser](https://github.com/mason173/aira-browser) 的 `services/personal-server`，由 GitHub Actions 自动构建 Docker 镜像并发布到 GitHub Container Registry（GHCR），部署端通过 Watchtower 自动拉取更新。

## 工作机制

```
上游 mason173/aira-browser (main)
        │  每天 UTC 18:00 定时同步（也可手动 workflow_dispatch）
        ▼
本仓库 services/personal-server  ──有变化──▶  GitHub Actions 构建
        │                                        │
        └── 无变化则跳过                          ▼
                                        ghcr.io/<你的用户名>/aira-personal-server:latest
                                                 │
                                                 ▼  Watchtower 每 5 分钟检查
                                        服务器上 docker compose 自动滚动更新
```

- **自动同步**：定时任务拉取上游最新代码，有变化才提交并构建
- **自动构建**：推送到 `main` 或上游有更新时，构建并推送 `latest` 和 `sha-xxxxxxx` 两个 tag
- **自动更新**：部署端 Watchtower 监听新镜像，自动 `pull` 并重建容器，数据卷不受影响

## 部署（目标服务器上执行）

只需一个 `docker-compose.yml`，无需克隆本仓库：

```bash
mkdir -p ~/aira-server && cd ~/aira-server
# 下载仓库中的 docker-compose.yml 和 .env.example
curl -fsSLO https://raw.githubusercontent.com/<你的用户名>/aira-personal-server/main/docker-compose.yml
curl -fsSLo .env https://raw.githubusercontent.com/<你的用户名>/aira-personal-server/main/.env.example

# 编辑 .env 填写对外访问地址（可选）
vi .env

docker compose up -d
```

查看配对码（客户端 `Settings > Sync > Personal Server` 中使用）：

```bash
docker compose exec aira-server cat /data/setup-code
```

### 端口与安全

- 默认只绑定 `127.0.0.1:8787`，公网访问请前置 Caddy/Nginx 反代 + HTTPS：

  ```caddyfile
  sync.example.com {
    reverse_proxy 127.0.0.1:8787
  }
  ```

  反代 request-body 上限保持 12 MiB 以上，并保留 `Authorization` 头。
- 局域网直接访问：把 compose 中的 `"127.0.0.1:8787:8787"` 改为 `"8787:8787"`。

### 镜像包可见性

若 GHCR 仓库为 private，目标服务器需要先 `docker login ghcr.io`；推荐在仓库 Packages 设置中把该包改为 **public**，无需登录即可拉取。

## 手动操作

- 手动触发同步+构建：仓库 Actions 页面 → `Build & Publish Personal Server Image` → `Run workflow`
- 停用自动更新：`docker compose stop watchtower`
- 备份/恢复：见上游文档 `services/personal-server/docs/operations.md`

## 许可

上游源码遵循 GPL-3.0-only，详见 `services/personal-server/LICENSE`。
