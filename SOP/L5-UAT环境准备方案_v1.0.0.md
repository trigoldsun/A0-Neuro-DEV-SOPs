# UAT 环境（阿里云ECS）准备方案

> **版本**: v1.0.0
> **更新时间**: 2026-05-12
> **说明**: 从原《非生产环境（Dev/Test/UAT）准备方案》拆分，聚焦 UAT 环境。Dev/Test 环境请参考《本地开发环境准备指南》。

## 1. 引言

本方案旨在为验收（UAT）环境在阿里云ECS上的准备提供详细指导。方案严格遵循《系统开发和文档管理指导手册 v2.2.0》[1]（以下简称“《一号文件》”）的各项规范，并结合Docker容器化技术，以实现环境的标准化、高效部署和资源优化。本方案将涵盖ECS实例选型、基础环境配置、Docker安装与配置，以及核心基础设施组件的部署。

## 2. 部署目标

*   建立一套**高度一致且可复现**的 UAT 环境，确保验收流程的顺畅。
*   通过Docker实现**资源隔离与高效利用**，降低运营成本。
*   确保所有环境配置符合《一号文件》中关于技术栈、端口、资源限制、安全基线等强制要求。
*   为微服务架构提供稳定、高效的运行基础。

## 3. ECS实例选型与配置

根据《一号文件》中“非生产环境允许单副本、单实例部署以节省资源”的原则，以及“单服务CPU限制在4核及以下，内存8G及以下”的要求，我们建议选择以下ECS实例配置作为基准：

| 配置项 | 建议值 | 备注 |
| :--- | :--- | :--- |
| **实例规格** | 4核8G 或 8核16G | 视微服务数量和并发压力而定，初期可选择4核8G，后续可升级。 |
| **操作系统** | Ubuntu 22.04 LTS | 稳定、社区支持良好，Docker兼容性最佳。 |
| **系统盘** | 100GB SSD | 满足操作系统、Docker镜像、容器日志及少量数据存储需求。 |
| **数据盘** | 视情况而定 | 如果自建数据库数据量较大，可额外挂载数据盘用于持久化存储。 |
| **网络带宽** | 1Mbps - 5Mbps | 非生产环境对外带宽需求较低，可按需选择。 |
| **安全组** | 最小权限原则 | 仅开放SSH (22)、HTTP/HTTPS (80/443，如果网关对外)、以及30000-50000端口段中服务可能使用的端口。 |

## 4. 宿主机基础环境配置

### 4.1 操作系统初始化

1.  **更新系统**: 登录ECS后，首先更新系统软件包。
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
2.  **安装常用工具**: 安装 `git`, `curl`, `wget` 等常用工具。
    ```bash
    sudo apt install git curl wget -y
    ```
3.  **配置时区**: 设置为上海时区。
    ```bash
    sudo timedatectl set-timezone Asia/Shanghai
    ```

### 4.2 Docker环境安装

按照Docker官方推荐方式安装Docker Engine和Docker Compose插件。这将确保您获得最新且稳定的Docker版本。

1.  **卸载旧版本 (如果存在)**:
    ```bash
    for pkg in docker.io docker-doc docker-compose docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-engine; do sudo apt remove $pkg; done
    ```
2.  **安装Docker Engine**: 参照[Docker官方文档](https://docs.docker.com/engine/install/ubuntu/)进行安装。
    ```bash
    # 添加Docker的官方GPG密钥
    sudo apt update
    sudo apt install ca-certificates curl gnupg -y
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # 添加Docker存储库
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt update

    # 安装Docker Engine、CLI和Containerd
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```
3.  **验证安装**: 运行 `hello-world` 镜像验证Docker是否正常工作。
    ```bash
    sudo docker run hello-world
    ```
4.  **配置非root用户使用Docker**: 将当前用户添加到 `docker` 用户组，以便无需 `sudo` 即可运行Docker命令。
    ```bash
    sudo usermod -aG docker $USER
    # 重新登录或执行 newgrp docker 使更改生效
    newgrp docker
    ```

### 4.3 项目目录结构

在ECS上创建统一的项目根目录，用于存放所有微服务的代码、Docker Compose文件、配置文件和持久化数据。

```
/opt/your_project_name/
├── docker-compose.yml          # Docker Compose主配置文件
├── .env                        # 环境变量文件，存放敏感信息或环境特定配置
├── services/                   # 存放各个微服务的代码和Dockerfile
│   ├── service_a/
│   │   ├── Dockerfile
│   │   └── ... (Go/Gin代码)
│   ├── service_b/
│   │   ├── Dockerfile
│   │   └── ... (Go/Gin代码)
│   └── ...
├── configs/                    # 存放Nginx、Vector等服务的配置文件
│   ├── nginx/nginx.conf
│   └── vector/vector.toml
├── data/                       # 存放数据库、MinIO等持久化数据，通过Docker Volume挂载
│   ├── postgres/
│   ├── redis/
│   └── minio/
├── logs/                       # 存放宿主机或共享日志，可通过Vector收集
└── db_migrations/              # 存放数据库迁移工具或脚本
    └── Dockerfile
    └── migrate_script.sh
```

## 5. 核心基础设施组件部署（Tier1）

本节将指导如何通过Docker Compose部署《一号文件》中提及的核心基础设施组件。这些组件将以**全局共享**的方式部署，为所有微服务提供基础服务。实际部署请参考 `docker-compose.infra.yml`。

### 5.1 Tier1 组件清单

| 组件 | 镜像 | 端口 | 资源限制 |
|:---|:---|:---|:---|
| Nginx Gateway | nginx:alpine | 20000 | CPU 0.2, 内存 128M |
| PostgreSQL | postgres:18-alpine | 20002 | CPU 0.5, 内存 512M |
| Redis | redis:7-alpine | 20003 | CPU 0.2, 内存 128M |
| MinIO | minio/minio:latest | 20004/20005 | CPU 0.5, 内存 128M |
| VictoriaMetrics | victoriametrics/v1.105.0 | 20010 | CPU 0.2, 内存 100M |

> 注意：本文档中的示例端口为示意，UAT 平台实际端口以部署配置为准（基础设施 20000-29999，项目服务 30000-49999）。

### 5.2 配置文件示例

*   **`configs/nginx/nginx.conf`**: 用于配置API网关的路由规则，将请求转发到不同的微服务。
    ```nginx
    worker_processes auto;
    events {
        worker_connections 1024;
    }
    http {
        include       mime.types;
        default_type  application/octet-stream;
        sendfile        on;
        keepalive_timeout  65;

        upstream backend_service_a {
            server service_a:8080; # 假设微服务A在容器内监听8080端口
        }
        upstream backend_service_b {
            server service_b:8080; # 假设微服务B在容器内监听8080端口
        }

        server {
            listen 80;
            server_name localhost;

            location /api/v1/service_a/ {
                proxy_pass http://backend_service_a/api/v1/service_a/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }

            location /api/v1/service_b/ {
                proxy_pass http://backend_service_b/api/v1/service_b/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }

            # 其他路由规则...
        }
    }
    ```
*   **`configs/vector/vector.toml`**: 用于配置Vector收集容器日志和宿主机日志，并发送到VictoriaMetrics（或Loki、Elasticsearch等）。
    ```toml
    # sources: 从哪里收集日志
    [sources.docker_logs]
    type = "docker_logs"
    include_containers = [".*"] # 收集所有容器的日志

    [sources.host_syslog]
    type = "syslog"
    address = "0.0.0.0:514"
    protocol = "udp"

    # sinks: 将日志发送到哪里
    [sinks.victoriametrics_logs]
    type = "http"
    inputs = ["docker_logs", "host_syslog"]
    uri = "http://victoriametrics:8428/api/v1/import/prometheus/" # 示例：发送到VictoriaMetrics
    encoding.codec = "json"
    # 其他配置，如认证、缓冲等
    ```

## 6. 部署流程

### 6.1 基础设施部署（一次性）

1.  **准备ECS实例**: 按照第3节和第4.1节的指导，完成ECS实例的选型、创建和操作系统初始化。
2.  **安装Docker**: 按照第4.2节的指导，在ECS上安装Docker Engine和Docker Compose。
3.  **克隆平台仓库**: 将包含 `docker-compose.infra.yml` 的平台管理仓库克隆到 `/opt/neuro-devsituat-platform`。
    ```bash
    git clone <platform_repo_url> /opt/neuro-devsituat-platform
    cd /opt/neuro-devsituat-platform
    ```
4.  **配置环境变量**: 创建 `.env` 和 `.env.secrets` 文件。
5.  **启动Tier1基础设施**:
    ```bash
    docker compose -f docker-compose.infra.yml up -d
    ```
6.  **验证基础设施**: 确认所有Tier1服务正常运行。

### 6.2 项目部署（按需）

1.  **创建项目**（自动分配端口/数据库/用户/桶）:
    ```bash
    platformctl project create <项目编号> <项目描述名>
    ```
2.  **登记资源**: 在对应附表中登记端口、路由、容器名等。
3.  **配置网关路由**:
    ```bash
    docker exec gateway-nginx nginx -s reload
    ```
4.  **启动项目服务**:
    ```bash
    platformctl project start <项目编号>
    ```
5.  **数据库迁移**: 使用 Goose 或 golang-migrate 执行迁移。
6.  **环境验证**: 通过网关访问项目API，验证各服务健康状态。

## 7. 安全与合规

*   **密钥管理**: 敏感配置（如数据库密码、API密钥）**禁止明文写入代码或配置文件**。存储在 `/opt/neuro-devsituat-platform/.env.secrets`（权限600），通过环境变量注入容器。
*   **端口规范**: 严格遵守《一号文件》中30000-50000的端口段要求，非整百端口用于非生产环境。
*   **资源限制**: 在 `docker-compose.yml` 中为每个服务配置CPU和内存限制，防止资源滥用。
*   **非root用户**: 容器内的应用应以非root用户运行，降低安全风险。
*   **环境隔离**: 明确标识非生产环境，严禁接入生产流量或存储生产敏感数据。

## 8. 参考文献

[1] 系统开发和文档管理指导手册 v2.2.0. (2026). [本地文件: /home/ubuntu/upload/01-系统开发和文档管理指导手册（简称：一号文件）_v2.2.0.md]
[2] 阿里云ECS开发、测试、验收环境Docker部署分析报告. (2026). [本地文件: /home/ubuntu/docker_deployment_analysis.md]
