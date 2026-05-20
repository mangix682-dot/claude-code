# Claude Code + Langfuse 本地部署指南

## 1. 部署 Langfuse（Docker 本地）

```bash
# 创建 docker-compose.yml
mkdir langfuse && cd langfuse
```

**方案 A（轻量，推荐）：Langfuse v2**

只需 PostgreSQL + Langfuse，资源占用小。

> 来源：https://github.com/langfuse/langfuse/blob/v2/docker-compose.yml

```yaml
# docker-compose.yml
services:
  langfuse-server:
    image: langfuse/langfuse:2
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/postgres
      - NEXTAUTH_SECRET=mysecret
      - SALT=mysalt
      - ENCRYPTION_KEY=0000000000000000000000000000000000000000000000000000000000000000
      - NEXTAUTH_URL=http://192.168.215.85:3000
      - TELEMETRY_ENABLED=${TELEMETRY_ENABLED:-true}
      - LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES=${LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES:-false}
      - LANGFUSE_INIT_ORG_ID=${LANGFUSE_INIT_ORG_ID:-}
      - LANGFUSE_INIT_ORG_NAME=${LANGFUSE_INIT_ORG_NAME:-}
      - LANGFUSE_INIT_PROJECT_ID=${LANGFUSE_INIT_PROJECT_ID:-}
      - LANGFUSE_INIT_PROJECT_NAME=${LANGFUSE_INIT_PROJECT_NAME:-}
      - LANGFUSE_INIT_PROJECT_PUBLIC_KEY=${LANGFUSE_INIT_PROJECT_PUBLIC_KEY:-}
      - LANGFUSE_INIT_PROJECT_SECRET_KEY=${LANGFUSE_INIT_PROJECT_SECRET_KEY:-}
      - LANGFUSE_INIT_USER_EMAIL=${LANGFUSE_INIT_USER_EMAIL:-}
      - LANGFUSE_INIT_USER_NAME=${LANGFUSE_INIT_USER_NAME:-}
      - LANGFUSE_INIT_USER_PASSWORD=${LANGFUSE_INIT_USER_PASSWORD:-}

  db:
    image: postgres:17
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 3s
      timeout: 3s
      retries: 10
      start_period: 30s
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=postgres
    ports:
      - 5432:5432
    volumes:
      - database_data:/var/lib/postgresql/data

volumes:
  database_data:
    driver: local
```

**方案 B（完整）：Langfuse v3**

v3 需要 ClickHouse + Redis + MinIO，占用更多资源（~2GB+ 内存）：

```yaml
# docker-compose.yml (v3 完整版)
services:
  langfuse-web:
    image: langfuse/langfuse:3
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_healthy
      redis:
        condition: service_healthy
      clickhouse:
        condition: service_healthy
    ports:
      - "3000:3000"
    environment: &langfuse-env
      NEXTAUTH_URL: http://192.168.215.85:3000
      NEXTAUTH_SECRET: any-random-secret-string
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/postgres
      SALT: any-random-salt-string
      ENCRYPTION_KEY: "0000000000000000000000000000000000000000000000000000000000000000"
      CLICKHOUSE_MIGRATION_URL: clickhouse://clickhouse:9000
      CLICKHOUSE_URL: http://clickhouse:8123
      CLICKHOUSE_USER: clickhouse
      CLICKHOUSE_PASSWORD: clickhouse
      LANGFUSE_S3_EVENT_UPLOAD_BUCKET: langfuse
      LANGFUSE_S3_EVENT_UPLOAD_REGION: auto
      LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID: minio
      LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY: miniosecret
      LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT: http://minio:9000
      LANGFUSE_S3_EVENT_UPLOAD_FORCE_PATH_STYLE: "true"
      LANGFUSE_S3_MEDIA_UPLOAD_BUCKET: langfuse
      LANGFUSE_S3_MEDIA_UPLOAD_REGION: auto
      LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID: minio
      LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY: miniosecret
      LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT: http://minio:9000
      LANGFUSE_S3_MEDIA_UPLOAD_FORCE_PATH_STYLE: "true"
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      REDIS_AUTH: myredissecret

  langfuse-worker:
    image: langfuse/langfuse-worker:3
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_healthy
      redis:
        condition: service_healthy
      clickhouse:
        condition: service_healthy
    environment:
      <<: *langfuse-env

  clickhouse:
    image: clickhouse/clickhouse-server
    restart: always
    user: "101:101"
    environment:
      CLICKHOUSE_DB: default
      CLICKHOUSE_USER: clickhouse
      CLICKHOUSE_PASSWORD: clickhouse
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:8123/ping || exit 1
      interval: 5s
      timeout: 5s
      retries: 10

  minio:
    image: minio/minio
    restart: always
    command: server --address ":9000" /data
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: miniosecret
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 3s
      timeout: 5s
      retries: 5

  minio-init:
    image: minio/mc
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      sh -c "mc alias set myminio http://minio:9000 minio miniosecret && mc mb --ignore-existing myminio/langfuse"

  redis:
    image: redis:7
    restart: always
    command: --requirepass myredissecret --maxmemory-policy noeviction
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 10s
      retries: 10

  postgres:
    image: postgres:17
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 3s
      timeout: 3s
      retries: 10

volumes:
  postgres_data:
  clickhouse_data:
  minio_data:
  redis_data:
```

```bash
cd ~/langfuse
docker compose down
# 如果修改了 docker-compose.yml 中的 NEXTAUTH_URL
docker compose up -d
```

访问 http://192.168.215.85:3000 → 注册账号 → 创建 Project → Settings 中获取 API Keys。

---

## 2. 配置 Claude Code

有两种方式：**内置 OTel 集成**（源码内置）和 **Hook 脚本**（官方推荐的外部方式）。

### 方式一：内置 OTel 集成（环境变量即生效）

claude-code 源码内置了 `@langfuse/otel` 集成（`src/services/langfuse/client.ts`），只需设置环境变量：

```powershell
# 全局环境变量（永久生效）
[Environment]::SetEnvironmentVariable("LANGFUSE_PUBLIC_KEY", "pk-lf-xxx", "User")
[Environment]::SetEnvironmentVariable("LANGFUSE_SECRET_KEY", "sk-lf-xxx", "User")
[Environment]::SetEnvironmentVariable("LANGFUSE_BASE_URL", "http://192.168.215.85:3000", "User")
```

或在项目级配置 `.claude/settings.local.json`：

```json
{
  "env": {
    "LANGFUSE_PUBLIC_KEY": "pk-lf-xxx",
    "LANGFUSE_SECRET_KEY": "sk-lf-xxx",
    "LANGFUSE_BASE_URL": "http://192.168.215.85:3000"
  }
}
```

**可选环境变量**：
| 变量 | 默认值 | 说明 |
|---|---|---|
| `LANGFUSE_FLUSH_AT` | 20 | 批量发送阈值 |
| `LANGFUSE_FLUSH_INTERVAL` | 10 | 发送间隔(ms) |
| `LANGFUSE_TRACING_ENVIRONMENT` | development | 环境标签 |
| `LANGFUSE_EXPORT_MODE` | batched | batched / immediate |
| `LANGFUSE_TIMEOUT` | 5 | 超时(s) |

**判断是否启用**：代码中 `isLangfuseEnabled()` 仅检查 `LANGFUSE_PUBLIC_KEY` 和 `LANGFUSE_SECRET_KEY` 是否非空。

### 方式二：Hook 脚本（langfuse.com 官方方案）

通过 claude-code 的 Stop hook 在每轮响应结束后解析 JSONL 并上报。

**步骤 1**：安装依赖

```bash
pip install langfuse
```

**步骤 2**：创建 hook 脚本 `~/.claude/hooks/langfuse_hook.py`

脚本来源：https://langfuse.com/integrations/other/claude-code

核心逻辑：读取 session JSONL → 增量解析新消息 → 按 turn 分组 → 上报到 Langfuse。

**步骤 3**：注册全局 hook `~/.claude/settings.json`

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/langfuse_hook.py"
          }
        ]
      }
    ]
  }
}
```

**步骤 4**：按项目启用（`.claude/settings.local.json`）

```json
{
  "env": {
    "TRACE_TO_LANGFUSE": "true",
    "LANGFUSE_PUBLIC_KEY": "pk-lf-xxx",
    "LANGFUSE_SECRET_KEY": "sk-lf-xxx",
    "LANGFUSE_BASE_URL": "http://192.168.215.85:3000"
  }
}
```

---

## 3. 两种方式对比

| | 内置 OTel | Hook 脚本 |
|---|---|---|
| 数据来源 | 应用运行时直接埋点 | 读取磁盘 JSONL 文件 |
| 数据丰富度 | 最高 | 仅 transcript 内容 |
| System Prompt | ✅ 包含 | ❌ 不含 |
| Tools Definition | ✅ 包含（name + description + input_schema） | ❌ 不含 |
| Messages（对话历史） | ✅ 完整 | ✅ 完整 |
| Token 用量 | ✅ 含 cache 细分 | ⚠️ 需从 JSONL 提取 |
| 延迟 | 异步批量，几乎无感 | 每轮结束后同步执行 |
| 侵入性 | 零（纯环境变量） | 需要安装 Python + 脚本 |
| Windows 兼容 | ✅ | ⚠️ 需适配路径和 fcntl |

**关键区别**：内置 OTel 集成在 API 调用点（`src/services/api/claude.ts:2975`）直接捕获 `systemPrompt` 和 `toolSchemas`，通过 `convertMessagesToLangfuse(messagesForAPI, systemPrompt)` 和 `convertToolsToLangfuse(toolSchemas)` 转换后上报。Hook 脚本只能读磁盘上的 JSONL，而 system prompt 和 tools definition 不写入 JSONL。

**推荐**：优先使用内置 OTel 集成，数据最完整且零配置。

---

## 4. 验证

启动 claude-code 后在 Langfuse UI 中查看：

- **Traces** — 每个 turn 一条 trace
- **Sessions** — 按 session_id 分组
- **Generations** — LLM 调用（含 model、token usage）
- **Tools** — 工具调用 span（name、input、output）

调试：设置 `CC_LANGFUSE_DEBUG=true`，日志写入 `~/.claude/state/langfuse_hook.log`。
