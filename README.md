# Oh My Media
Oh My Media 是一个面向媒体库管理与在线播放的部署工具，支持目录扫描、元数据/NFO 管理、标签、分类、系列、演员、收藏、资源查询、在线播放与定时扫描等功能。

## 快速开始

推荐使用 Docker Compose 部署。镜像支持 `linux/amd64` 和 `linux/arm64`。

### 1. 准备 docker-compose.yaml

将发布包中的 `docker-compose.yaml` 放入部署目录；如果发布包未附带该文件，可使用以下模板创建：

```yaml
services:
  md-center:
    image: ghcr.io/dbonline666/oh-my-media:latest
    container_name: md-center
    restart: unless-stopped
    environment:
      TZ: Asia/Shanghai
    ports:
      - "8001:8001"
    volumes:
      - ./data:/app/data
      - /path/to/media:/media
```

生产环境建议将 `latest` 替换为具体版本号，例如 `ghcr.io/dbonline666/oh-my-media:2.0.10`，便于后续升级和回滚。

### 2. 启动

```bash
docker compose up -d
```

### 3. 访问

- 浏览器访问：`http://<服务器IP>:8001`
- 健康检查：`curl http://127.0.0.1:8001/api/health`

如果 `8001` 已被占用，可修改 compose 中的端口映射，例如 `8080:8001`，然后访问 `http://<服务器IP>:8080`。

## 数据目录与媒体目录

Compose 中两个挂载点说明：

| 宿主机路径 | 容器内路径 | 说明 |
| --- | --- | --- |
| `./data` | `/app/data` | 持久化配置、数据库、日志和数据库备份 |
| `/path/to/media` | `/media` | 媒体库根目录，需要改为实际路径 |

- 首次启动时，程序会在 `/app/data/config.yaml` 自动生成默认配置，对应宿主机的 `./data/config.yaml`。
- 在 Web UI 添加媒体目录时，填写容器内路径，例如 `/media/电影`，不要填写宿主机路径。
- 媒体目录默认使用读写挂载。应用可能需要在媒体目录中写入 NFO、海报、缩略图、外挂字幕等文件，若改为只读挂载，这些功能会受限。

## 配置

主要配置文件为 `./data/config.yaml`。修改后执行：

```bash
docker compose restart
```

常见配置项：

- `server.port`：服务监听端口，默认 `8001`
- `translation`：翻译服务地址、API Key、模型和开关
- `dbonline_api`：资源查询服务地址、API Key 和开关
- `ffmpeg`：转码与媒体信息相关配置
- `schedule`：定时扫描开关和时间

不推荐把 API Key 写入公开文档或提交到仓库，请在部署目录的 `config.yaml` 中配置。

## ffmpeg

官方镜像当前未内置 `ffmpeg`/`ffprobe`。未提供时，目录扫描、库浏览和部分直连播放仍可使用，但媒体信息探测与转码播放会降级。

如需启用转码，请将兼容的静态版 `ffmpeg`/`ffprobe` 放到宿主机目录，并挂载进容器，例如：

```yaml
    volumes:
      - ./data:/app/data
      - /path/to/media:/media
      - ./ffmpeg/bin:/opt/ffmpeg:ro
```

然后在 `config.yaml` 中配置：

```yaml
ffmpeg:
  ffmpeg_path: /opt/ffmpeg/ffmpeg
  ffprobe_path: /opt/ffmpeg/ffprobe
  enabled: true
```

## 升级

1. 先备份 `./data` 目录。
2. 修改 `docker-compose.yaml` 中的镜像版本号。
3. 执行：

```bash
docker compose pull
docker compose up -d
```

升级前建议查看 release 说明，确认是否存在需要手动处理的配置变更。

## 备份与迁移

- 应用数据主要在 `./data` 目录中，包括 `config.yaml`、`media_manager.db`、`logs` 和数据库备份。
- 建议在服务停止后备份：

```bash
docker compose stop
cp -r ./data ./data-backup
docker compose start
```

Windows PowerShell 可用：

```powershell
docker compose stop
Copy-Item -Path .\data -Destination .\data-backup -Recurse
docker compose start
```

- 迁移时保持媒体挂载点一致（例如仍使用 `/media`），否则数据库中记录的媒体路径会失效。迁移后如需调整媒体目录，应在 Web UI 中更新目录路径并重新扫描。

## 日志与排障

```bash
# 查看容器日志
docker compose logs -f md-center

# 健康检查
curl http://127.0.0.1:8001/api/health
```

常见问题：

- 容器启动成功但页面打不开：确认端口映射和防火墙是否放行。
- Web UI 中看不到媒体：确认媒体目录已正确挂载，并在目录管理中使用容器内路径。
- 日志提示 `ffmpeg/ffprobe 未找到`：按上文提供 ffmpeg 并重启。
- 升级后数据库或路径异常：恢复升级前备份，检查媒体挂载路径是否保持一致。

## 停止服务

```bash
# 停止容器，保留数据
docker compose down

# 删除 compose 管理的容器和网络
docker compose down --remove-orphans
```

请勿随意使用 `docker compose down -v`，该命令会删除 compose 中定义的命名卷。示例使用宿主机目录挂载，删除前仍需确认数据已有备份。
