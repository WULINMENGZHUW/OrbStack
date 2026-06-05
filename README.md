# OrbStack Services 运维手册

每个子目录是一个独立的 Docker Compose 项目，可以单独启停，互不影响。

---

## 目录结构

```
orbstack-services/
├── infrastructure/          # 基础中间件（Postgres、Redis、Nginx、MinIO、Metabase、Meilisearch）
├── litellm/                 # AI 代理网关（端口 4000）
├── scrm/                    # SCRM 客户系统（端口 3000）
├── scrm-handover/           # SCRM 交接版本，含 Nginx（端口 8080）
├── portkey/                 # Portkey Gateway（端口 8787）
├── numa/                    # Numa（无固定端口）
└── stocks/                  # 股票看板（端口 8000）
```

---

## 当前服务状态

| 服务 | 目录 | 端口 | 数据卷 | 备注 |
|---|---|---|---|---|
| PostgreSQL 16 | infrastructure | 5432 | `orbstack-services_pgdata` | 常驻，共享数据库 |
| Redis | infrastructure | 6379 | `orbstack-services_redisdata` | 常驻 |
| Nginx | infrastructure | 8080 | — | 静态文件服务 |
| MinIO | infrastructure | 9000 / 9001 | `orbstack-services_miniodata` | 9001 为 Web 控制台 |
| Metabase | infrastructure | 3000 | `orbstack-services_metabasedata` | 数据看板 |
| Meilisearch | infrastructure | 7700 | `orbstack-services_meilidata` | 搜索引擎 |
| LiteLLM Proxy | litellm | 4000 | `code_postgres_data` | AI 模型路由，含配置文件 |
| SCRM | scrm | 3000 | `scrm-backup_scrm_data` | 按需启动 |
| SCRM Handover | scrm-handover | 8080 | `scrm-handover_pg_data` | 按需启动 |
| Portkey Gateway | portkey | 8787 | — | 按需启动 |
| Numa | numa | — | — | 按需启动 |
| 股票看板 | stocks | 8000 | `stocks_stocks_data` | 按需启动 |

---

## 日常运维

### 启动 / 停止 / 重启

```bash
# 进入对应目录操作
cd ~/OrbStack-services/litellm

docker compose up -d          # 后台启动
docker compose down           # 停止并移除容器（数据卷不受影响）
docker compose restart        # 重启所有容器
docker compose restart proxy  # 重启单个服务
```

### 查看状态和日志

```bash
docker compose ps             # 查看该项目的容器状态
docker compose logs -f        # 实时跟踪日志
docker compose logs -f proxy  # 只看某个服务的日志
docker ps                     # 查看所有运行中的容器（全局）
```

### 更新镜像

```bash
docker compose pull           # 拉取最新镜像
docker compose up -d          # 重新创建容器（旧容器自动替换）
```

### 进入容器调试

```bash
docker compose exec proxy sh  # 进入容器 shell
docker compose exec db psql -U litellm  # 直接连数据库
```

---

## 数据卷管理

数据卷是持久化的核心，**所有 compose 文件均用 `external: true` 引用已有卷，`docker compose down` 不会删除数据**。

```bash
docker volume ls              # 列出所有卷
docker volume inspect orbstack-services_pgdata  # 查看卷详情（含实际路径）
```

OrbStack 将卷存储在：`~/Library/Application Support/OrbStack/data/...`

**备份某个卷的数据：**

```bash
# 把 Postgres 数据导出为 SQL
docker exec pgsql-16 pg_dumpall -U root > backup_$(date +%Y%m%d).sql

# 把任意卷打包为 tar
docker run --rm -v orbstack-services_pgdata:/data -v $(pwd):/backup \
  alpine tar czf /backup/pgdata_$(date +%Y%m%d).tar.gz -C /data .
```

**恢复数据：**

```bash
# 恢复 SQL
cat backup_20260605.sql | docker exec -i pgsql-16 psql -U root

# 恢复 tar
docker run --rm -v orbstack-services_pgdata:/data -v $(pwd):/backup \
  alpine tar xzf /backup/pgdata_20260605.tar.gz -C /data
```

---

## 定期清理（避免磁盘爆满）

```bash
# 删除所有已停止的容器（不影响运行中的）
docker container prune -f

# 删除没有容器使用的镜像（不删除有标签且被引用的）
docker image prune -f

# 一键清理容器 + 网络 + 悬空镜像（不碰卷）
docker system prune -f

# 查看磁盘占用
docker system df
```

> 不要执行 `docker system prune -a` 或 `docker volume prune`，前者会删除所有未使用的镜像（包括停止的服务镜像），后者会删除数据卷。

---

## 新增项目指引

### 1. 创建目录

```bash
mkdir ~/OrbStack-services/my-new-service
cd ~/OrbStack-services/my-new-service
```

### 2. 编写 docker-compose.yml

**如果是全新镜像（无持久数据）：**

```yaml
services:
  app:
    image: some-image:latest
    container_name: my-new-service
    restart: always
    ports:
      - "XXXX:XXXX"
    environment:
      SOME_KEY: some_value
```

**如果需要持久数据卷：**

```yaml
services:
  app:
    image: some-image:latest
    container_name: my-new-service
    restart: always
    ports:
      - "XXXX:XXXX"
    volumes:
      - app_data:/data

volumes:
  app_data:   # 首次 up 会自动创建，名称为 my-new-service_app_data
```

**如果需要复用 infrastructure 里的 Postgres：**

```yaml
services:
  app:
    image: some-image:latest
    environment:
      DATABASE_URL: postgresql://root:mysecretpassword@host.docker.internal:5432/yourdb
```

> 用 `host.docker.internal` 连宿主机上的 Postgres，无需和 infrastructure 共享网络。

### 3. 有环境变量 / API Key 时

创建 `.env` 文件，compose 自动读取同目录的 `.env`：

```bash
# .env
API_KEY=sk-xxxx
DATABASE_URL=postgresql://...
```

在 compose 里引用：

```yaml
services:
  app:
    env_file:
      - .env
```

**.env 不要提交到 Git。** 在目录下加 `.gitignore`：

```
.env
```

### 4. 启动前检查端口冲突

先查下方「端口速查」表，再用命令确认端口实际是否空闲：

```bash
# 检查单个端口是否被占用（有输出 = 已占用）
lsof -iTCP:8787 -sTCP:LISTEN

# 同时检查多个端口
lsof -iTCP -sTCP:LISTEN | grep -E ':(3000|4000|8080|8787)'

# 查看所有正在监听的端口（全局概览）
lsof -iTCP -sTCP:LISTEN | awk 'NR>1 {print $9, $1, $2}' | sort
```

端口被占用时，输出示例：

```
node    12345  xuwei  ...  TCP *:3000 (LISTEN)
```

确认空闲后再写进 compose，避免启动失败。

### 5. 启动验证

```bash
docker compose up -d
docker compose ps       # 确认 Status 为 Up
docker compose logs -f  # 确认无报错
```

### 6. 更新本文件

在上方「当前服务状态」表格里补一行，记录端口和数据卷名称。

---

## 端口速查（已占用）

| 端口 | 服务 |
|---|---|
| 3000 | Metabase / SCRM（二选一，注意冲突） |
| 4000 | LiteLLM Proxy |
| 5432 | PostgreSQL 16 |
| 6379 | Redis |
| 7700 | Meilisearch |
| 8000 | 股票看板 |
| 8080 | Nginx / SCRM Handover |
| 8787 | Portkey Gateway |
| 9000 | MinIO API |
| 9001 | MinIO Console |

新增服务选端口前先在此表确认无冲突。
