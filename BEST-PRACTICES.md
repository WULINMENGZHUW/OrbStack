# OrbStack 极客最佳实践

> 目标：每个服务一个目录，全部用 YAML 声明，密钥不进 Git，数据永不丢失。

---

## 核心原则

| 原则 | 做法 |
|---|---|
| 配置即代码 | 所有服务用 `docker-compose.yml` 声明，不手动 `docker run` |
| 密钥隔离 | 密钥只写在 `.env`，`.env` 不进 Git |
| 数据永久化 | 持久数据用 named volume，不用 bind mount 到宿主机路径 |
| 配置文件版本化 | 服务配置文件（如 `config.yaml`）放在项目目录内，随 Git 管理 |
| 静态资源本地化 | Nginx html 等纯静态内容用 bind mount，但排除出 Git |

---

## 目录结构规范

每个服务目录的标准布局：

```
my-service/
├── docker-compose.yml   # 必须：服务声明（进 Git）
├── .env                 # 必须（如有密钥）：环境变量（不进 Git）
├── .env.example         # 推荐：.env 的模板，填占位值（进 Git）
└── config/              # 可选：服务配置文件，如 nginx.conf、config.yaml（进 Git）
```

**数据文件不放在这里**。数据由 Docker named volume 管理，存在 OrbStack 内部。

---

## 数据放在哪

### 三类数据，三种策略

```
数据类型           存储方式                  示例
─────────────────────────────────────────────────────────
结构化持久数据     Named Volume（Docker 管理）  Postgres、Redis、SQLite DB
服务配置文件       项目目录 bind mount          nginx.conf、litellm config.yaml
静态/媒体文件      项目目录 bind mount，排除 Git  Nginx html、上传文件
```

### Named Volume：持久数据的唯一正解

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # ✅ named volume

volumes:
  pgdata:   # Docker 自动管理路径，升级/迁移容器数据不丢
```

**不要用绝对路径 bind mount 存数据库文件：**

```yaml
volumes:
  - /Users/xuwei/data/postgres:/var/lib/postgresql/data   # ❌ 路径耦合宿主机
```

OrbStack 把 named volume 存在：
```
~/Library/Application Support/OrbStack/data/docker/volumes/
```

### 配置文件：bind mount 到项目目录

```yaml
services:
  proxy:
    volumes:
      - ./config.yaml:/app/config.yaml   # ✅ 相对路径，随项目目录走
```

`./` 表示 `docker-compose.yml` 所在目录，迁移机器时整个目录拷走即可。

### 引用已有 Volume（历史迁移场景）

如果 volume 是由旧的 compose 项目创建，名称已固定，用 `external` 引用：

```yaml
volumes:
  pgdata:
    external: true
    name: orbstack-services_pgdata   # 旧项目遗留的卷名
```

新项目应直接声明 volume，让 Docker 用 `<项目名>_<卷名>` 自动命名。

---

## 密钥管理：全用 .env

### 规范

所有密码、API Key、Token 只写在 `.env` 文件，绝不硬编码进 `docker-compose.yml`。

**compose 里这样引用：**

```yaml
services:
  app:
    env_file:
      - .env          # 整个文件注入
```

或只取其中几个变量（不污染容器环境）：

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}   # 从 .env 取单个值
```

### .env.example 模板（必须进 Git）

每个有 `.env` 的目录都要提供 `.env.example`，填占位值，方便别人或换机器时知道需要哪些变量：

```bash
# .env.example
POSTGRES_USER=root
POSTGRES_PASSWORD=change-me
API_KEY=sk-your-key-here
```

### .gitignore 配置

根目录的 `.gitignore` 已覆盖所有子目录：

```gitignore
.env
*.env
```

---

## 完整项目模板

新增一个服务，复制这个模板：

**`docker-compose.yml`**

```yaml
services:
  app:
    image: your-image:latest
    container_name: your-service
    restart: always
    ports:
      - "8888:8080"
    env_file:
      - .env
    volumes:
      - app_data:/data          # 持久数据 → named volume
      - ./config:/app/config    # 配置文件 → bind mount 项目目录
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    container_name: your-service-db
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  app_data:   # 自动命名为 your-service_app_data
  db_data:    # 自动命名为 your-service_db_data
```

**`.env`**（不进 Git）

```bash
POSTGRES_DB=mydb
POSTGRES_USER=myuser
POSTGRES_PASSWORD=change-me-in-prod
```

**`.env.example`**（进 Git）

```bash
POSTGRES_DB=mydb
POSTGRES_USER=myuser
POSTGRES_PASSWORD=
```

---

## 服务间通信

### 同一 compose 项目内（推荐）

同一 `docker-compose.yml` 里的服务直接用服务名访问：

```yaml
DATABASE_URL: postgresql://user:pass@db:5432/mydb
#                                     ↑ 服务名，不是 localhost
```

### 跨 compose 项目

用 `host.docker.internal` 访问宿主机上已暴露端口的服务：

```yaml
DATABASE_URL: postgresql://root:pass@host.docker.internal:5432/mydb
```

不要用 `localhost`，容器里的 localhost 是容器自身。

---

## 升级镜像的正确姿势

```bash
cd ~/OrbStack-services/litellm

# 1. 拉新镜像
docker compose pull

# 2. 重建容器（数据卷不受影响）
docker compose up -d

# 3. 清理旧镜像层
docker image prune -f
```

固定版本号（如 `litellm:1.86.0`）可控；用 `latest` 方便但可能引入破坏性更新。权衡选择。

---

## 当前项目目录与 .env.example 清单

| 目录 | 有 .env | 有 .env.example | 数据卷 |
|---|---|---|---|
| infrastructure | ✅ | ✅ | pgdata / redisdata / miniodata / metabasedata / meilidata |
| litellm | ✅ | ✅ | code_postgres_data（历史遗留名） |
| scrm | ✅ | ✅ | scrm-backup_scrm_data（历史遗留名） |
| scrm-handover | ✅ | ✅ | scrm-handover_pg_data |
| portkey | — | — | 无 |
| numa | — | — | 无 |
| stocks | — | — | stocks_stocks_data |
