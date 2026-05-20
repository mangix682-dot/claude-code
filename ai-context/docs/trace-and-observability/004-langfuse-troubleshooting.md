# Langfuse 常见问题排查

## 1. SDK v3 与 Server v2 不兼容（数据静默丢失）

**现象**：Hook 脚本日志显示 `Flush completed successfully`，但 Langfuse UI 无任何 trace。服务端日志只有 `Read from postgres only`，没有 ingestion 请求。

**原因**：`pip install langfuse` 默认安装 SDK v3，而 Langfuse Server 用的是 v2 镜像（`langfuse/langfuse:2`）。SDK v3 的数据格式与 Server v2 不兼容，请求被静默丢弃，**不报错**。

**解决**：

```bash
pip install "langfuse<3"
```

**验证**：

```bash
pip show langfuse  # 确认版本 2.x.x
```

**同时注意**：SDK v2 和 v3 的 API 也不同：

| | SDK v2 | SDK v3 |
|---|---|---|
| 创建 trace | `lf.trace(name=...)` | `lf.create_trace(name=...)` |
| 嵌套观测 | `trace.generation(...)` / `trace.span(...)` | `lf.start_as_current_observation(...)` |
| 上下文传播 | 不需要 | `propagate_attributes(...)` |

如果 hook 脚本是为 SDK v3 编写的（使用 `start_as_current_observation`），降级 SDK 后脚本也需要改为 v2 API。

---

## 2. 环境变量设置后不生效

**现象**：`[Environment]::SetEnvironmentVariable(...)` 设置了 User 级变量，但 Claude Code 读不到。

**原因**：User 级环境变量只对**新启动的进程**生效。已打开的终端、IDE、Claude Code 不会自动刷新。

**解决**：设置后**关闭并重新打开**终端/IDE，再启动 Claude Code。

**验证**（在新终端中）：

```powershell
$env:LANGFUSE_PUBLIC_KEY  # 应输出 pk-lf-xxx
```

---

## 3. Langfuse v3 Server 需要 ClickHouse

**现象**：`docker compose up -d` 后容器启动失败，日志报错：

```
Error: CLICKHOUSE_URL is not configured. Migrating from V2? Check out migration guide
```

**原因**：`langfuse/langfuse:latest` 已经是 v3，v3 需要 ClickHouse + Redis + MinIO。

**解决**：

- 方案 A：改用 v2 镜像 `langfuse/langfuse:2`（只需 PostgreSQL）
- 方案 B：使用完整 v3 docker-compose（含 ClickHouse/Redis/MinIO，参见 `langfuse-setup.md`）

---

## 4. VM 网络不通（VirtualBox）

**现象**：Windows 浏览器访问 `http://<VM-IP>:3000` 超时。

**排查**：

```powershell
# Windows 侧测试
Invoke-WebRequest http://<VM-IP>:3000 -UseBasicParsing -TimeoutSec 5
```

**常见原因**：

- VirtualBox 网络模式为 NAT（改为**桥接模式**）
- VM 防火墙挡了端口（`sudo ufw allow 3000/tcp`）
- VM IP 变了（`ip addr | grep 192.168` 确认）
