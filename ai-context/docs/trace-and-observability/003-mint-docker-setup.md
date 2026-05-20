# Linux Mint 22.x 安装 Docker

基于 Ubuntu 24.04 (Noble)。

## 1. 卸载旧版本（如有）

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc 2>/dev/null | cut -f1) 2>/dev/null
```

## 2. 安装依赖 + 添加 Docker 官方源

> 参考：https://docs.docker.com/engine/install/ubuntu/

```bash
# 安装依赖
sudo apt update
sudo apt install -y ca-certificates curl

# 添加 Docker 官方 GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 添加 apt 源（DEB822 格式）
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

> **注意**：Mint 22.x 基于 Ubuntu 24.04，`Suites` 必须写 `noble`。官方文档用 `$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")`，但 Mint 的 `VERSION_CODENAME` 是 `wilma` 不是 `noble`，所以这里**硬编码**。

## 3. 安装 Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 4. 免 sudo 运行

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 5. 验证

```bash
docker --version
docker compose version
docker run --rm hello-world
```

看到 `Hello from Docker!` 即成功。

---

## 中国网络环境注意事项（FlClash 代理）

假设 FlClash 开在本机，HTTP 代理端口为 `7890`（根据你的实际端口调整）。

### A. apt 下载 + curl 添加 GPG Key（安装阶段）

终端临时设置代理：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

设置后再执行上面的步骤 2、3 即可。安装完成后可 `unset http_proxy https_proxy`。

### B. Docker 拉取镜像（最关键）

Docker daemon 拉镜像走的是自己的网络，**不读终端的环境变量**，需要单独配置。

> 参考：https://docs.docker.com/engine/daemon/proxy/

**方法 1（官方推荐）：daemon.json**

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "proxies": {
    "http-proxy": "http://127.0.0.1:7890",
    "https-proxy": "http://127.0.0.1:7890",
    "no-proxy": "localhost,127.0.0.1"
  }
}
EOF

sudo systemctl restart docker
```

**方法 2：systemd drop-in 文件**

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
```

> **注意**：两种方法二选一。daemon.json 方式优先级更高。

验证：

```bash
sudo systemctl show --property=Environment docker
docker info | grep -i proxy
```

### C. 容器内访问外网（Langfuse 容器不需要）

如果容器内需要访问外网，在 `~/.docker/config.json` 中添加：

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://host.docker.internal:7890",
      "httpsProxy": "http://host.docker.internal:7890",
      "noProxy": "localhost,127.0.0.1"
    }
  }
}
```

> Langfuse 容器只需连本地 PostgreSQL，不需要访问外网，所以这步一般不需要。

### 总结

| 场景 | 需要代理 | 配置位置 |
|---|---|---|
| apt 安装 Docker | ✅ | 终端 `export` 环境变量 |
| docker pull 拉镜像 | ✅ **最重要** | `/etc/systemd/system/docker.service.d/proxy.conf` |
| 容器内访问外网 | 一般不需要 | `~/.docker/config.json` |
