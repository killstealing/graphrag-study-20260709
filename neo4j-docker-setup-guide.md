# Neo4j Docker 安装与连接指南 (WSL Ubuntu)

## 环境说明

- Windows 11 + WSL2 Ubuntu 24.04
- Docker Engine 安装在 WSL Ubuntu 内部（非 Docker Desktop）

---

## 1. 安装 Docker Engine

```bash
# 卸载旧版（如果有）
sudo apt remove docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 添加 Docker 官方 GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 添加 Docker APT 仓库
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 免 sudo 使用 docker
sudo usermod -aG docker $USER
```

---

## 2. 启动 Docker 守护进程

WSL 中 systemd 可能有兼容性问题（`exit code 203/EXEC`），直接手动启动：

```bash
sudo nohup dockerd &>/tmp/dockerd.log &
```

验证：

```bash
docker ps
```

---

## 3. 拉取并启动 Neo4j

```bash
# 拉取镜像
docker pull neo4j:latest

# 启动容器
docker run -d \
  --name neo4j-graphrag \
  -p 7474:7474 \
  -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/graphrag2024 \
  -v neo4j-data:/data \
  -v neo4j-logs:/logs \
  neo4j:latest
```

### 端口说明

| 端口 | 用途 |
|------|------|
| `7474` | HTTP 浏览器访问 |
| `7687` | Bolt 协议（Python 连接用） |

### 验证启动

```bash
docker logs neo4j-graphrag | grep -E "Bolt enabled|Started"
```

看到 `Bolt enabled on 0.0.0.0:7687` 和 `Started` 即表示成功。

---

## 4. 配置免密码 sudo（dockerd 专用）

```bash
echo 'ALL ALL=(ALL) NOPASSWD: /usr/bin/dockerd' | sudo tee /etc/sudoers.d/dockerd
sudo chmod 440 /etc/sudoers.d/dockerd
```

---

## 5. 配置开机自动启动

在 `~/.bashrc` 末尾添加：

```bash
# ===== 自动启动 Docker 和 Neo4j =====
if ! pgrep -x dockerd > /dev/null; then
    sudo nohup dockerd &>/tmp/dockerd.log &
    sleep 2
fi

if ! docker ps --format '{{.Names}}' | grep -q neo4j-graphrag; then
    docker start neo4j-graphrag 2>/dev/null
fi
# ===== Docker / Neo4j 自动启动结束 =====
```

以后每次打开 Ubuntu 终端，dockerd 和 Neo4j 都会自动运行。

---

## 6. 常用管理命令

```bash
docker stop neo4j-graphrag    # 停止 Neo4j
docker start neo4j-graphrag   # 启动 Neo4j
docker restart neo4j-graphrag # 重启 Neo4j
docker logs neo4j-graphrag    # 查看日志
docker rm -f neo4j-graphrag   # 删除容器（数据卷保留）
```

---

## 7. 连接信息

| 参数 | 值 |
|------|-----|
| URI | `bolt://localhost:7687` |
| 用户名 | `neo4j` |
| 密码 | `graphrag2024` |
| 默认数据库 | `neo4j` |
| 浏览器访问 | `http://localhost:7474` |

---

## 8. `.env` 配置

```env
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=graphrag2024
NEO4J_DATABASE=neo4j
```

---

## 9. Python 连接示例

### 原生 neo4j 驱动

```python
import os
from dotenv import load_dotenv
from neo4j import GraphDatabase

load_dotenv()

driver = GraphDatabase.driver(
    os.getenv("NEO4J_URI"),          # bolt://localhost:7687
    auth=(
        os.getenv("NEO4J_USERNAME"),  # neo4j
        os.getenv("NEO4J_PASSWORD"),  # graphrag2024
    ),
)

# 测试连接
driver.verify_connectivity()
print("Neo4j 连接成功")
driver.close()
```

### LangChain Neo4jGraph

```python
import os
from dotenv import load_dotenv
from langchain_community.graphs import Neo4jGraph

load_dotenv()

graph = Neo4jGraph(
    url=os.getenv("NEO4J_URI"),
    username=os.getenv("NEO4J_USERNAME"),
    password=os.getenv("NEO4J_PASSWORD"),
)
```

---

## 10. 已知问题与解决

| 问题 | 原因 | 解决 |
|------|------|------|
| `docker ps` 一直卡住 | Docker daemon 没启动 | `sudo nohup dockerd &` |
| `systemctl start docker` 报 203/EXEC | WSL systemd 兼容性 | 手动启动 dockerd，不用 systemctl |
| 关闭终端后 Neo4j 不可用 | WSL 闲置后自动关机 | 配置 `~/.bashrc` 自动启动 |
