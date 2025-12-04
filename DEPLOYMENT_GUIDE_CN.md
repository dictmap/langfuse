# Langfuse 生产环境部署指南

## 目录
1. [服务器要求](#1-服务器要求)
2. [环境准备](#2-环境准备)
3. [安装 Docker 和 Docker Compose](#3-安装-docker-和-docker-compose)
4. [下载和配置 Langfuse](#4-下载和配置-langfuse)
5. [配置环境变量](#5-配置环境变量)
6. [启动服务](#6-启动服务)
7. [初始化和验证](#7-初始化和验证)
8. [配置域名和反向代理](#8-配置域名和反向代理)
9. [安全加固](#9-安全加固)
10. [备份和维护](#10-备份和维护)
11. [故障排查](#11-故障排查)
12. [升级指南](#12-升级指南)

---

## 1. 服务器要求

### 1.1 最低硬件配置
- **CPU**: 4 核心（推荐 8 核心）
- **内存**: 8GB RAM（推荐 16GB 或更高）
- **存储**: 100GB SSD（取决于数据量，推荐 500GB+）
- **网络**: 稳定的互联网连接

### 1.2 推荐配置（生产环境）
- **CPU**: 8 核心或更高
- **内存**: 16GB 或更高
- **存储**: 500GB NVMe SSD 或更高
- **备份**: 配置自动备份存储

### 1.3 操作系统
- **推荐**: Ubuntu 22.04 LTS 或 Ubuntu 24.04 LTS
- **其他支持**: Debian 11/12, CentOS 8+, RHEL 8+
- **架构**: x86_64 (amd64) 或 ARM64

### 1.4 端口要求
需要开放以下端口：
- **3000**: Langfuse Web 应用（对外）
- **9090**: MinIO S3 存储（对外访问文件上传）
- 其他端口（5432/PostgreSQL, 6379/Redis, 8123/ClickHouse）仅内部访问

---

## 2. 环境准备

### 2.1 更新系统
```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y

# CentOS/RHEL
sudo yum update -y
```

### 2.2 安装基础工具
```bash
# Ubuntu/Debian
sudo apt install -y curl wget git vim openssl net-tools

# CentOS/RHEL
sudo yum install -y curl wget git vim openssl net-tools
```

### 2.3 配置防火墙
```bash
# Ubuntu/Debian (ufw)
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 3000/tcp  # Langfuse Web
sudo ufw allow 9090/tcp  # MinIO
sudo ufw enable

# CentOS/RHEL (firewalld)
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```

---

## 3. 安装 Docker 和 Docker Compose

### 3.1 安装 Docker

#### Ubuntu/Debian
```bash
# 卸载旧版本
sudo apt remove -y docker docker-engine docker.io containerd runc

# 安装依赖
sudo apt install -y ca-certificates curl gnupg lsb-release

# 添加 Docker 官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 设置 Docker 仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 启动 Docker
sudo systemctl enable docker
sudo systemctl start docker
```

#### CentOS/RHEL
```bash
# 卸载旧版本
sudo yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

# 安装依赖
sudo yum install -y yum-utils

# 添加 Docker 仓库
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装 Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 启动 Docker
sudo systemctl enable docker
sudo systemctl start docker
```

### 3.2 配置 Docker（可选但推荐）
```bash
# 将当前用户添加到 docker 组，避免每次都用 sudo
sudo usermod -aG docker $USER

# 重新登录使更改生效，或运行：
newgrp docker

# 验证 Docker 安装
docker --version
docker compose version
docker run hello-world
```

### 3.3 配置 Docker 镜像加速（中国大陆用户）
```bash
# 创建或编辑 Docker daemon 配置
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF

# 重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 4. 下载和配置 Langfuse

### 4.1 创建部署目录
```bash
# 创建 Langfuse 部署目录
sudo mkdir -p /opt/langfuse
cd /opt/langfuse

# 设置目录权限
sudo chown -R $USER:$USER /opt/langfuse
```

### 4.2 下载配置文件
```bash
# 下载 docker-compose.yml
curl -o docker-compose.yml https://raw.githubusercontent.com/langfuse/langfuse/main/docker-compose.yml

# 下载环境变量模板
curl -o .env.example https://raw.githubusercontent.com/langfuse/langfuse/main/.env.prod.example
```

### 4.3 复制环境变量文件
```bash
# 创建实际的环境变量文件
cp .env.example .env

# 设置文件权限（保护敏感信息）
chmod 600 .env
```

---

## 5. 配置环境变量

这是最关键的步骤！使用你喜欢的编辑器编辑 `.env` 文件：

```bash
vim .env
# 或者
nano .env
```

### 5.1 必须配置的核心变量

#### 数据库连接
```bash
# PostgreSQL 数据库连接
DATABASE_URL="postgresql://langfuse_user:你的强密码@postgres:5432/langfuse"

# PostgreSQL 数据库凭证（用于 docker-compose 中的 postgres 服务）
POSTGRES_USER=langfuse_user
POSTGRES_PASSWORD=你的强密码
POSTGRES_DB=langfuse
```

#### 认证密钥（必须修改！）
```bash
# NextAuth 密钥 - 生成方法：openssl rand -base64 32
NEXTAUTH_SECRET="在这里粘贴生成的密钥"

# API 密钥加密盐值 - 生成方法：openssl rand -base64 32
SALT="在这里粘贴生成的盐值"

# 数据加密密钥 - 生成方法：openssl rand -hex 32
ENCRYPTION_KEY="在这里粘贴64位16进制字符串"
```

**生成这些密钥的命令：**
```bash
echo "NEXTAUTH_SECRET=\"$(openssl rand -base64 32)\""
echo "SALT=\"$(openssl rand -base64 32)\""
echo "ENCRYPTION_KEY=\"$(openssl rand -hex 32)\""
```

#### 应用 URL
```bash
# 将 localhost 替换为你的域名或服务器 IP
NEXTAUTH_URL="http://你的域名或IP:3000"
# 例如：
# NEXTAUTH_URL="https://langfuse.yourdomain.com"
# 或者：
# NEXTAUTH_URL="http://192.168.1.100:3000"
```

#### ClickHouse 配置
```bash
# ClickHouse 连接
CLICKHOUSE_URL="http://clickhouse:8123"
CLICKHOUSE_MIGRATION_URL="clickhouse://clickhouse:9000"
CLICKHOUSE_USER=clickhouse_user
CLICKHOUSE_PASSWORD=你的ClickHouse强密码
CLICKHOUSE_DB=default
CLICKHOUSE_CLUSTER_ENABLED=false
```

#### Redis 配置
```bash
# Redis 连接
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_AUTH=你的Redis强密码
REDIS_TLS_ENABLED=false
```

#### MinIO (S3) 配置
```bash
# MinIO 凭证
MINIO_ROOT_USER=langfuse_minio
MINIO_ROOT_PASSWORD=你的MinIO强密码

# S3 事件上传配置
LANGFUSE_S3_EVENT_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_EVENT_UPLOAD_REGION=auto
LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=langfuse_minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=你的MinIO强密码
LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT=http://minio:9000
LANGFUSE_S3_EVENT_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_EVENT_UPLOAD_PREFIX=events/

# S3 媒体上传配置
LANGFUSE_S3_MEDIA_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_MEDIA_UPLOAD_REGION=auto
LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID=langfuse_minio
LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY=你的MinIO强密码
LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT=http://localhost:9090
LANGFUSE_S3_MEDIA_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_MEDIA_UPLOAD_PREFIX=media/

# S3 批量导出配置
LANGFUSE_S3_BATCH_EXPORT_ENABLED=false
LANGFUSE_S3_BATCH_EXPORT_BUCKET=langfuse
LANGFUSE_S3_BATCH_EXPORT_PREFIX=exports/
LANGFUSE_S3_BATCH_EXPORT_REGION=auto
LANGFUSE_S3_BATCH_EXPORT_ENDPOINT=http://minio:9000
LANGFUSE_S3_BATCH_EXPORT_EXTERNAL_ENDPOINT=http://localhost:9090
LANGFUSE_S3_BATCH_EXPORT_ACCESS_KEY_ID=langfuse_minio
LANGFUSE_S3_BATCH_EXPORT_SECRET_ACCESS_KEY=你的MinIO强密码
LANGFUSE_S3_BATCH_EXPORT_FORCE_PATH_STYLE=true
```

### 5.2 初始用户配置（推荐）
```bash
# 创建初始组织和项目
LANGFUSE_INIT_ORG_ID=your-org-id
LANGFUSE_INIT_ORG_NAME=你的组织名称
LANGFUSE_INIT_PROJECT_ID=your-project-id
LANGFUSE_INIT_PROJECT_NAME=你的项目名称

# 创建初始用户
LANGFUSE_INIT_USER_EMAIL=admin@yourdomain.com
LANGFUSE_INIT_USER_NAME=管理员
LANGFUSE_INIT_USER_PASSWORD=你的初始密码

# API 密钥（可选，如果不设置会自动生成）
# LANGFUSE_INIT_PROJECT_PUBLIC_KEY=pk-lf-xxxxxxxx
# LANGFUSE_INIT_PROJECT_SECRET_KEY=sk-lf-xxxxxxxx
```

### 5.3 可选配置

#### 邮件配置（用于密码重置等）
```bash
EMAIL_FROM_ADDRESS=noreply@yourdomain.com
SMTP_CONNECTION_URL=smtp://username:password@smtp.gmail.com:587
```

#### OAuth 登录（可选）
```bash
# Google OAuth
AUTH_GOOGLE_CLIENT_ID=你的Google客户端ID
AUTH_GOOGLE_CLIENT_SECRET=你的Google客户端密钥
AUTH_GOOGLE_ALLOW_ACCOUNT_LINKING=false

# GitHub OAuth
AUTH_GITHUB_CLIENT_ID=你的GitHub客户端ID
AUTH_GITHUB_CLIENT_SECRET=你的GitHub客户端密钥
AUTH_GITHUB_ALLOW_ACCOUNT_LINKING=false
```

#### 功能开关
```bash
# 启用实验性功能
LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES=false

# 遥测（匿名使用数据收集）
TELEMETRY_ENABLED=true

# 日志级别
LANGFUSE_LOG_LEVEL=info
LANGFUSE_LOG_FORMAT=text
```

### 5.4 安全建议

**重要提示：** 所有标记为 `# CHANGEME` 的值都必须修改！

**密码强度要求：**
- 至少 16 个字符
- 包含大小写字母、数字和特殊字符
- 不要使用字典单词

**生成强密码的方法：**
```bash
# 生成 32 字符随机密码
openssl rand -base64 32

# 生成 64 位十六进制密钥
openssl rand -hex 32
```

---

## 6. 启动服务

### 6.1 拉取 Docker 镜像
```bash
cd /opt/langfuse

# 拉取所有镜像
docker compose pull
```

### 6.2 启动服务
```bash
# 启动所有服务（后台运行）
docker compose up -d

# 查看日志
docker compose logs -f

# 只查看特定服务的日志
docker compose logs -f langfuse-web
docker compose logs -f langfuse-worker
```

### 6.3 检查服务状态
```bash
# 查看所有容器状态
docker compose ps

# 应该看到所有服务都是 "Up" 状态：
# - langfuse-web
# - langfuse-worker
# - postgres
# - clickhouse
# - redis
# - minio
```

### 6.4 等待服务完全启动
```bash
# 观察 web 服务日志，等待看到 "ready started server"
docker compose logs -f langfuse-web

# 等待数据库迁移完成
# 你应该看到类似这样的日志：
# "Applying database migrations..."
# "Migration applied successfully"
```

---

## 7. 初始化和验证

### 7.1 访问 Web 界面
在浏览器中访问：
```
http://你的服务器IP:3000
或
http://你的域名:3000
```

### 7.2 首次登录
如果你配置了 `LANGFUSE_INIT_USER_*` 变量：
- **邮箱**: 你设置的 `LANGFUSE_INIT_USER_EMAIL`
- **密码**: 你设置的 `LANGFUSE_INIT_USER_PASSWORD`

如果没有配置初始用户，点击 "Sign up" 注册新账户。

### 7.3 验证功能

#### 检查数据库连接
```bash
# 连接到 PostgreSQL 检查数据
docker compose exec postgres psql -U langfuse_user -d langfuse -c "\dt"

# 应该看到多个表已创建
```

#### 检查 ClickHouse
```bash
# 检查 ClickHouse 表
docker compose exec clickhouse clickhouse-client --user clickhouse_user --password 你的密码 --query "SHOW TABLES"
```

#### 检查 Redis
```bash
# 检查 Redis 连接
docker compose exec redis redis-cli -a 你的Redis密码 ping

# 应该返回 PONG
```

#### 检查 MinIO
在浏览器访问 MinIO 控制台：
```
http://你的服务器IP:9091
```
- **用户名**: 你设置的 `MINIO_ROOT_USER`
- **密码**: 你设置的 `MINIO_ROOT_PASSWORD`

### 7.4 测试功能
1. 登录 Langfuse Web 界面
2. 创建新项目
3. 获取 API 密钥
4. 尝试发送测试追踪数据

---

## 8. 配置域名和反向代理

### 8.1 使用 Nginx 反向代理

#### 安装 Nginx
```bash
# Ubuntu/Debian
sudo apt install -y nginx

# CentOS/RHEL
sudo yum install -y nginx
```

#### 配置 Nginx
创建配置文件：
```bash
sudo vim /etc/nginx/sites-available/langfuse
```

粘贴以下配置（修改域名）：
```nginx
# HTTP 配置（用于重定向到 HTTPS）
server {
    listen 80;
    listen [::]:80;
    server_name langfuse.yourdomain.com;

    # 用于 Let's Encrypt 验证
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # 重定向到 HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS 配置
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name langfuse.yourdomain.com;

    # SSL 证书（稍后配置）
    ssl_certificate /etc/letsencrypt/live/langfuse.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/langfuse.yourdomain.com/privkey.pem;

    # SSL 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # 安全头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # 客户端上传大小限制
    client_max_body_size 100M;

    # 代理到 Langfuse Web
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}

# MinIO S3 API（可选，如果需要外部访问）
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name minio.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/minio.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/minio.yourdomain.com/privkey.pem;

    client_max_body_size 500M;

    location / {
        proxy_pass http://localhost:9090;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 启用配置
```bash
# 创建软链接
sudo ln -s /etc/nginx/sites-available/langfuse /etc/nginx/sites-enabled/

# 测试配置
sudo nginx -t

# 如果测试通过，重启 Nginx
sudo systemctl restart nginx
```

### 8.2 配置 SSL 证书（Let's Encrypt）

#### 安装 Certbot
```bash
# Ubuntu/Debian
sudo apt install -y certbot python3-certbot-nginx

# CentOS/RHEL
sudo yum install -y certbot python3-certbot-nginx
```

#### 获取证书
```bash
# 为 Langfuse 域名获取证书
sudo certbot --nginx -d langfuse.yourdomain.com

# 如果需要为 MinIO 也获取证书
sudo certbot --nginx -d minio.yourdomain.com

# Certbot 会自动配置 Nginx 并设置自动续期
```

#### 验证自动续期
```bash
# 测试续期
sudo certbot renew --dry-run

# 查看定时任务
sudo systemctl status certbot.timer
```

### 8.3 更新环境变量
修改 `.env` 文件，更新 URL：
```bash
NEXTAUTH_URL="https://langfuse.yourdomain.com"
LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT="https://minio.yourdomain.com"
LANGFUSE_S3_BATCH_EXPORT_EXTERNAL_ENDPOINT="https://minio.yourdomain.com"
```

重启服务：
```bash
docker compose down
docker compose up -d
```

---

## 9. 安全加固

### 9.1 限制端口访问

#### 更新 docker-compose.yml
修改 `/opt/langfuse/docker-compose.yml`，确保非必要端口只绑定到 localhost：

```yaml
services:
  postgres:
    ports:
      - 127.0.0.1:5432:5432  # 仅本地访问

  redis:
    ports:
      - 127.0.0.1:6379:6379  # 仅本地访问

  clickhouse:
    ports:
      - 127.0.0.1:8123:8123  # 仅本地访问
      - 127.0.0.1:9000:9000  # 仅本地访问

  minio:
    ports:
      - 127.0.0.1:9090:9000  # 如果用 Nginx 代理，改为仅本地
      - 127.0.0.1:9091:9001  # 仅本地访问控制台
```

### 9.2 配置防火墙规则
```bash
# 只允许必要端口
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

### 9.3 设置环境变量文件权限
```bash
chmod 600 /opt/langfuse/.env
chown $USER:$USER /opt/langfuse/.env
```

### 9.4 启用 Docker 用户命名空间（可选）
编辑 `/etc/docker/daemon.json`：
```json
{
  "userns-remap": "default"
}
```

重启 Docker：
```bash
sudo systemctl restart docker
```

### 9.5 定期更新
```bash
# 设置自动更新（Ubuntu/Debian）
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 10. 备份和维护

### 10.1 备份策略

#### 数据库备份脚本
创建 `/opt/langfuse/backup.sh`：
```bash
#!/bin/bash

BACKUP_DIR="/backup/langfuse"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# 备份 PostgreSQL
docker compose exec -T postgres pg_dump -U langfuse_user langfuse | gzip > $BACKUP_DIR/postgres_$DATE.sql.gz

# 备份 ClickHouse
docker compose exec -T clickhouse clickhouse-client --user clickhouse_user --password 你的密码 --query "SELECT * FROM system.tables FORMAT TabSeparatedWithNames" | gzip > $BACKUP_DIR/clickhouse_$DATE.tsv.gz

# 备份环境变量文件
cp .env $BACKUP_DIR/env_$DATE

# 备份 MinIO 数据
docker compose exec -T minio mc mirror /data $BACKUP_DIR/minio_$DATE/

# 删除 30 天前的备份
find $BACKUP_DIR -type f -mtime +30 -delete

echo "Backup completed: $DATE"
```

设置权限并添加到 crontab：
```bash
chmod +x /opt/langfuse/backup.sh

# 编辑 crontab
crontab -e

# 添加每天凌晨 2 点备份
0 2 * * * /opt/langfuse/backup.sh >> /var/log/langfuse-backup.log 2>&1
```

### 10.2 日志轮转
创建 `/etc/logrotate.d/langfuse`：
```
/var/log/langfuse*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    missingok
    create 644 root root
}
```

### 10.3 监控磁盘空间
```bash
# 检查 Docker 卷使用情况
docker system df -v

# 清理未使用的数据（谨慎使用）
docker system prune -a --volumes
```

### 10.4 定期维护任务

#### PostgreSQL 维护
```bash
# 每周运行一次 VACUUM
docker compose exec postgres psql -U langfuse_user -d langfuse -c "VACUUM ANALYZE;"
```

#### ClickHouse 维护
```bash
# 优化表
docker compose exec clickhouse clickhouse-client --user clickhouse_user --password 你的密码 --query "OPTIMIZE TABLE traces FINAL"
```

---

## 11. 故障排查

### 11.1 常见问题

#### 问题：容器启动失败
```bash
# 查看详细日志
docker compose logs langfuse-web
docker compose logs langfuse-worker

# 检查端口占用
sudo netstat -tulpn | grep LISTEN

# 重启服务
docker compose down
docker compose up -d
```

#### 问题：数据库连接失败
```bash
# 检查 PostgreSQL 是否运行
docker compose ps postgres

# 测试数据库连接
docker compose exec postgres psql -U langfuse_user -d langfuse -c "SELECT 1"

# 检查环境变量
docker compose exec langfuse-web env | grep DATABASE
```

#### 问题：ClickHouse 迁移失败
```bash
# 查看 ClickHouse 日志
docker compose logs clickhouse

# 手动运行迁移
docker compose exec langfuse-web sh -c "cd packages/shared && sh clickhouse/scripts/up.sh"
```

#### 问题：MinIO 无法访问
```bash
# 检查 MinIO 状态
docker compose ps minio

# 检查 bucket 是否创建
docker compose exec minio mc ls /data/

# 测试连接
curl http://localhost:9090/minio/health/live
```

### 11.2 性能优化

#### 增加 PostgreSQL 性能
修改 docker-compose.yml，添加 PostgreSQL 配置：
```yaml
postgres:
  command:
    - "postgres"
    - "-c"
    - "shared_buffers=256MB"
    - "-c"
    - "max_connections=200"
    - "-c"
    - "work_mem=16MB"
```

#### 增加 ClickHouse 性能
创建 clickhouse 配置文件并挂载到容器。

### 11.3 日志级别调整
修改 `.env`：
```bash
LANGFUSE_LOG_LEVEL=debug  # 用于故障排查
# 或
LANGFUSE_LOG_LEVEL=error  # 用于生产环境
```

### 11.4 重置数据库（谨慎！）
```bash
# 完全重置（会删除所有数据）
docker compose down -v
docker compose up -d
```

---

## 12. 升级指南

### 12.1 升级前准备
```bash
# 1. 备份数据
/opt/langfuse/backup.sh

# 2. 查看当前版本
docker compose exec langfuse-web cat package.json | grep version

# 3. 查看更新日志
# 访问: https://github.com/langfuse/langfuse/releases
```

### 12.2 升级步骤
```bash
cd /opt/langfuse

# 1. 停止服务
docker compose down

# 2. 备份当前 docker-compose.yml
cp docker-compose.yml docker-compose.yml.backup

# 3. 更新镜像版本
# 编辑 docker-compose.yml，修改镜像标签
# 例如：langfuse/langfuse:3 -> langfuse/langfuse:3.x.x

# 4. 拉取新镜像
docker compose pull

# 5. 启动服务（会自动运行数据库迁移）
docker compose up -d

# 6. 查看日志确认升级成功
docker compose logs -f langfuse-web

# 7. 验证功能正常
```

### 12.3 回滚
```bash
# 如果升级出现问题
docker compose down

# 恢复旧版本配置
cp docker-compose.yml.backup docker-compose.yml

# 启动旧版本
docker compose up -d

# 如果需要，恢复数据库备份
# 谨慎操作！
```

---

## 附录 A：完整的 .env 配置示例

```bash
# 数据库
DATABASE_URL="postgresql://langfuse_user:StrongPassword123!@postgres:5432/langfuse"
POSTGRES_USER=langfuse_user
POSTGRES_PASSWORD=StrongPassword123!
POSTGRES_DB=langfuse

# NextAuth
NEXTAUTH_URL="https://langfuse.yourdomain.com"
NEXTAUTH_SECRET="生成的密钥"
SALT="生成的盐值"
ENCRYPTION_KEY="生成的加密密钥"

# ClickHouse
CLICKHOUSE_URL="http://clickhouse:8123"
CLICKHOUSE_MIGRATION_URL="clickhouse://clickhouse:9000"
CLICKHOUSE_USER=clickhouse_user
CLICKHOUSE_PASSWORD=ClickHousePass123!
CLICKHOUSE_DB=default
CLICKHOUSE_CLUSTER_ENABLED=false

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_AUTH=RedisPass123!
REDIS_TLS_ENABLED=false

# MinIO
MINIO_ROOT_USER=langfuse_minio
MINIO_ROOT_PASSWORD=MinIOPass123!

LANGFUSE_S3_EVENT_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_EVENT_UPLOAD_REGION=auto
LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=langfuse_minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=MinIOPass123!
LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT=http://minio:9000
LANGFUSE_S3_EVENT_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_EVENT_UPLOAD_PREFIX=events/

LANGFUSE_S3_MEDIA_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_MEDIA_UPLOAD_REGION=auto
LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID=langfuse_minio
LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY=MinIOPass123!
LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT=https://minio.yourdomain.com
LANGFUSE_S3_MEDIA_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_MEDIA_UPLOAD_PREFIX=media/

# 初始化
LANGFUSE_INIT_ORG_ID=my-org
LANGFUSE_INIT_ORG_NAME=我的组织
LANGFUSE_INIT_PROJECT_ID=my-project
LANGFUSE_INIT_PROJECT_NAME=我的项目
LANGFUSE_INIT_USER_EMAIL=admin@yourdomain.com
LANGFUSE_INIT_USER_NAME=管理员
LANGFUSE_INIT_USER_PASSWORD=AdminPass123!

# 功能
LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES=false
TELEMETRY_ENABLED=true
LANGFUSE_LOG_LEVEL=info
```

---

## 附录 B：常用命令速查

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose down

# 查看日志
docker compose logs -f

# 重启特定服务
docker compose restart langfuse-web

# 查看运行状态
docker compose ps

# 进入容器
docker compose exec langfuse-web sh

# 查看资源使用
docker stats

# 清理未使用资源
docker system prune -a

# 备份数据
/opt/langfuse/backup.sh

# 更新镜像
docker compose pull && docker compose up -d
```

---

## 附录 C：资源链接

- **官方文档**: https://langfuse.com/docs
- **自托管指南**: https://langfuse.com/docs/deployment/self-host
- **GitHub 仓库**: https://github.com/langfuse/langfuse
- **社区论坛**: https://langfuse.com/discord
- **问题追踪**: https://github.com/langfuse/langfuse/issues

---

## 结语

这份部署指南应该能帮助你成功部署 Langfuse。如果遇到任何问题：

1. 首先检查日志：`docker compose logs -f`
2. 查看官方文档的故障排查部分
3. 在 GitHub Issues 或 Discord 社区寻求帮助

**重要提醒：**
- 定期备份数据
- 定期更新系统和 Docker 镜像
- 使用强密码
- 启用 HTTPS
- 监控系统资源

祝你部署顺利！
