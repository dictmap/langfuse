# Langfuse Windows 本地部署指南

## 目录
1. [系统要求](#1-系统要求)
2. [安装 Docker Desktop](#2-安装-docker-desktop)
3. [下载 Langfuse 配置文件](#3-下载-langfuse-配置文件)
4. [配置环境变量](#4-配置环境变量)
5. [启动 Langfuse](#5-启动-langfuse)
6. [访问和使用](#6-访问和使用)
7. [常用操作](#7-常用操作)
8. [故障排查](#8-故障排查)
9. [数据备份](#9-数据备份)

---

## 1. 系统要求

### 1.1 硬件要求
- **CPU**: 4 核心或更高（推荐）
- **内存**: 至少 8GB RAM（推荐 16GB）
- **硬盘**: 至少 20GB 可用空间（SSD 更佳）

### 1.2 软件要求
- **操作系统**:
  - Windows 10 64位：专业版、企业版或教育版（版本 1903 或更高）
  - Windows 11 64位：所有版本
- **必须启用**:
  - WSL 2（Windows Subsystem for Linux 2）
  - 虚拟化功能（在 BIOS 中启用）

### 1.3 检查虚拟化是否启用
1. 按 `Ctrl + Shift + Esc` 打开任务管理器
2. 点击"性能"选项卡
3. 选择"CPU"
4. 查看右下角"虚拟化"是否显示"已启用"

**如果显示"已禁用"**，需要在 BIOS 中启用：
- 重启电脑
- 进入 BIOS（通常按 F2、F10、Del 或 Esc 键）
- 找到 "Virtualization Technology"、"Intel VT-x" 或 "AMD-V"
- 设置为 "Enabled"
- 保存并退出

---

## 2. 安装 Docker Desktop

### 2.1 下载 Docker Desktop

访问官网下载：
```
https://www.docker.com/products/docker-desktop/
```

或直接下载链接（Windows）：
```
https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe
```

### 2.2 安装 Docker Desktop

1. **运行安装程序**
   - 双击 `Docker Desktop Installer.exe`
   - 如果弹出 UAC 提示，点击"是"

2. **配置选项**（保持默认即可）
   - ✅ Use WSL 2 instead of Hyper-V（推荐）
   - ✅ Add shortcut to desktop

3. **完成安装**
   - 点击"OK"开始安装
   - 等待安装完成
   - 点击"Close and restart"重启电脑

### 2.3 启动和配置 Docker Desktop

1. **首次启动**
   - 重启后，Docker Desktop 会自动启动
   - 接受服务协议
   - 可以跳过登录（点击 "Continue without signing in"）

2. **等待 Docker 启动**
   - 查看系统托盘，Docker 图标从橙色变为绿色
   - 表示 Docker 已完全启动

3. **验证安装**
   - 按 `Win + R`，输入 `cmd`，回车打开命令提示符
   - 运行以下命令：
   ```cmd
   docker --version
   docker compose version
   ```
   - 应该看到版本信息

### 2.4 配置 Docker Desktop（可选但推荐）

1. **右键点击托盘中的 Docker 图标**
2. **选择 "Settings"（设置）**
3. **推荐配置**：

#### Resources（资源）
- **CPUs**: 分配 4 个或更多核心
- **Memory**: 分配至少 6GB（推荐 8GB）
- **Disk image size**: 保持默认或根据需要调整

#### Docker Engine（可选，中国大陆用户）
添加镜像加速器，提高下载速度：

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

点击 "Apply & restart" 应用设置。

---

## 3. 下载 Langfuse 配置文件

### 3.1 创建项目文件夹

1. **选择一个位置创建文件夹**，例如：
   ```
   C:\Users\你的用户名\langfuse
   ```
   或
   ```
   D:\projects\langfuse
   ```

2. **打开该文件夹**

### 3.2 下载配置文件

#### 方法一：使用浏览器下载

1. **下载 docker-compose.yml**
   - 访问：https://raw.githubusercontent.com/langfuse/langfuse/main/docker-compose.yml
   - 右键点击页面 → "另存为" → 保存到 langfuse 文件夹
   - 文件名：`docker-compose.yml`

2. **下载环境变量模板**
   - 访问：https://raw.githubusercontent.com/langfuse/langfuse/main/.env.prod.example
   - 右键点击页面 → "另存为" → 保存到 langfuse 文件夹
   - 文件名：`.env`（注意：文件名以点开头）

#### 方法二：使用命令行（PowerShell）

1. **按 `Win + X`，选择 "Windows PowerShell"**

2. **切换到 langfuse 文件夹**：
   ```powershell
   cd C:\Users\你的用户名\langfuse
   ```

3. **下载文件**：
   ```powershell
   # 下载 docker-compose.yml
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/langfuse/langfuse/main/docker-compose.yml" -OutFile "docker-compose.yml"

   # 下载环境变量模板
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/langfuse/langfuse/main/.env.prod.example" -OutFile ".env"
   ```

### 3.3 验证文件

确保 langfuse 文件夹中有以下两个文件：
- `docker-compose.yml`
- `.env`

---

## 4. 配置环境变量

### 4.1 生成安全密钥

#### 使用 PowerShell 生成（推荐）

1. **打开 PowerShell**（按 `Win + X` → 选择 "Windows PowerShell"）

2. **运行以下命令生成密钥**：

```powershell
# 生成 NEXTAUTH_SECRET（base64，32字节）
$nextauthSecret = -join ((1..32 | ForEach-Object { [byte[]](Get-Random -Maximum 256) }) | ForEach-Object { [Convert]::ToBase64String([byte[]]@($_)) })
Write-Host "NEXTAUTH_SECRET=$nextauthSecret"

# 生成 SALT（base64，32字节）
$salt = -join ((1..32 | ForEach-Object { [byte[]](Get-Random -Maximum 256) }) | ForEach-Object { [Convert]::ToBase64String([byte[]]@($_)) })
Write-Host "SALT=$salt"

# 生成 ENCRYPTION_KEY（hex，32字节=64字符）
$encryptionKey = -join ((1..32 | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) }))
Write-Host "ENCRYPTION_KEY=$encryptionKey"
```

或者使用简化版本：

```powershell
# 一次性生成所有密钥
Write-Host "NEXTAUTH_SECRET=" -NoNewline; Write-Host ([Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 })))
Write-Host "SALT=" -NoNewline; Write-Host ([Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 })))
Write-Host "ENCRYPTION_KEY=" -NoNewline; Write-Host (-join (1..32 | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) }))
```

**复制生成的三个值，稍后会用到。**

#### 使用在线工具生成（备选）

如果 PowerShell 命令不工作，可以使用在线工具：
- Base64（NEXTAUTH_SECRET 和 SALT）: https://www.random.org/strings/?num=1&len=32&digits=on&upperalpha=on&loweralpha=on&unique=on&format=html&rnd=new
- Hex（ENCRYPTION_KEY）: https://www.random.org/cgi-bin/randbyte?nbytes=32&format=h

### 4.2 编辑 .env 文件

1. **使用记事本或其他文本编辑器打开 `.env` 文件**
   - 推荐使用：VS Code、Notepad++、或 Windows 记事本
   - 在文件夹中右键 `.env` → "打开方式" → 选择编辑器

2. **修改以下关键配置**：

```bash
# ========================================
# 数据库配置
# ========================================
DATABASE_URL="postgresql://langfuse:langfuse123@postgres:5432/langfuse"
POSTGRES_USER=langfuse
POSTGRES_PASSWORD=langfuse123
POSTGRES_DB=langfuse

# ========================================
# 应用 URL（本地部署使用 localhost）
# ========================================
NEXTAUTH_URL="http://localhost:3000"

# ========================================
# 安全密钥（粘贴刚才生成的值）
# ========================================
NEXTAUTH_SECRET="粘贴刚才生成的NEXTAUTH_SECRET"
SALT="粘贴刚才生成的SALT"
ENCRYPTION_KEY="粘贴刚才生成的ENCRYPTION_KEY"

# ========================================
# ClickHouse 配置
# ========================================
CLICKHOUSE_URL="http://clickhouse:8123"
CLICKHOUSE_MIGRATION_URL="clickhouse://clickhouse:9000"
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=clickhouse123
CLICKHOUSE_DB=default
CLICKHOUSE_CLUSTER_ENABLED=false

# ========================================
# Redis 配置
# ========================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_AUTH=redis123
REDIS_TLS_ENABLED=false

# ========================================
# MinIO (S3) 配置
# ========================================
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=minio123

LANGFUSE_S3_EVENT_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_EVENT_UPLOAD_REGION=auto
LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=minio123
LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT=http://minio:9000
LANGFUSE_S3_EVENT_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_EVENT_UPLOAD_PREFIX=events/

LANGFUSE_S3_MEDIA_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_MEDIA_UPLOAD_REGION=auto
LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY=minio123
LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT=http://localhost:9090
LANGFUSE_S3_MEDIA_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_MEDIA_UPLOAD_PREFIX=media/

# ========================================
# 初始用户配置（可选但推荐）
# ========================================
LANGFUSE_INIT_ORG_ID=my-org
LANGFUSE_INIT_ORG_NAME=我的组织
LANGFUSE_INIT_PROJECT_ID=my-project
LANGFUSE_INIT_PROJECT_NAME=我的项目
LANGFUSE_INIT_USER_EMAIL=admin@example.com
LANGFUSE_INIT_USER_NAME=管理员
LANGFUSE_INIT_USER_PASSWORD=admin123

# ========================================
# 功能开关
# ========================================
LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES=true
TELEMETRY_ENABLED=true
LANGFUSE_LOG_LEVEL=info
```

3. **保存文件**（Ctrl + S）

### 4.3 配置说明

**本地开发环境的简化配置**：
- 密码可以使用简单的值（如 `langfuse123`），因为只在本地访问
- 如果是生产环境，必须使用强密码
- `NEXTAUTH_URL` 保持为 `http://localhost:3000`
- 所有服务通过 Docker 内部网络通信

---

## 5. 启动 Langfuse

### 5.1 打开命令行

1. **在 langfuse 文件夹中按住 Shift 键 + 右键**
2. **选择 "在此处打开 PowerShell 窗口"**

或者：
1. **按 `Win + R`**
2. **输入 `cmd` 或 `powershell`**
3. **运行**：
   ```cmd
   cd C:\Users\你的用户名\langfuse
   ```

### 5.2 拉取 Docker 镜像

**第一次启动需要下载镜像，可能需要 5-15 分钟**，取决于网速。

```cmd
docker compose pull
```

你会看到类似这样的输出：
```
[+] Pulling 6/6
 ✔ langfuse-web Pulled
 ✔ langfuse-worker Pulled
 ✔ postgres Pulled
 ✔ clickhouse Pulled
 ✔ redis Pulled
 ✔ minio Pulled
```

### 5.3 启动所有服务

```cmd
docker compose up -d
```

参数说明：
- `up`：启动服务
- `-d`：后台运行（detached mode）

你会看到：
```
[+] Running 6/6
 ✔ Container langfuse-postgres-1         Started
 ✔ Container langfuse-clickhouse-1       Started
 ✔ Container langfuse-redis-1            Started
 ✔ Container langfuse-minio-1            Started
 ✔ Container langfuse-langfuse-worker-1  Started
 ✔ Container langfuse-langfuse-web-1     Started
```

### 5.4 查看启动日志

```cmd
docker compose logs -f
```

**等待看到以下关键信息**：
- `Applying database migrations...`（应用数据库迁移）
- `Migration applied successfully`（迁移成功）
- `ready started server on 0.0.0.0:3000`（Web 服务器启动成功）

**按 `Ctrl + C` 停止查看日志**（不会停止服务）

### 5.5 检查服务状态

```cmd
docker compose ps
```

**所有服务的 STATUS 应该是 "Up"**：
```
NAME                         STATUS
langfuse-clickhouse-1        Up (healthy)
langfuse-langfuse-web-1      Up
langfuse-langfuse-worker-1   Up
langfuse-minio-1             Up (healthy)
langfuse-postgres-1          Up (healthy)
langfuse-redis-1             Up (healthy)
```

---

## 6. 访问和使用

### 6.1 访问 Langfuse Web 界面

在浏览器中访问：
```
http://localhost:3000
```

### 6.2 首次登录

#### 如果配置了初始用户（推荐）

使用 `.env` 文件中设置的凭据：
- **邮箱**: `LANGFUSE_INIT_USER_EMAIL` 的值（例如：admin@example.com）
- **密码**: `LANGFUSE_INIT_USER_PASSWORD` 的值（例如：admin123）

#### 如果没有配置初始用户

1. 点击 "Sign up"（注册）
2. 输入邮箱和密码创建账户
3. 登录

### 6.3 访问 MinIO 控制台（可选）

MinIO 是 S3 兼容的对象存储，用于存储文件。

在浏览器中访问：
```
http://localhost:9091
```

登录凭据：
- **用户名**: `.env` 中的 `MINIO_ROOT_USER`（例如：minio）
- **密码**: `.env` 中的 `MINIO_ROOT_PASSWORD`（例如：minio123）

### 6.4 开始使用

1. **创建项目**
   - 登录后会自动创建项目（如果配置了 `LANGFUSE_INIT_PROJECT_*`）
   - 或者手动创建新项目

2. **获取 API 密钥**
   - 在项目设置中找到 API 密钥
   - 复制 Public Key 和 Secret Key

3. **集成到你的应用**
   - 使用 Python SDK 或 TypeScript SDK
   - 参考官方文档：https://langfuse.com/docs

---

## 7. 常用操作

### 7.1 停止 Langfuse

在 langfuse 文件夹中运行：

```cmd
docker compose down
```

这会停止所有容器，但**数据会保留**。

### 7.2 重新启动 Langfuse

```cmd
docker compose up -d
```

### 7.3 查看日志

```cmd
# 查看所有服务日志
docker compose logs -f

# 只查看 Web 服务日志
docker compose logs -f langfuse-web

# 只查看 Worker 服务日志
docker compose logs -f langfuse-worker

# 查看最后 100 行日志
docker compose logs --tail=100
```

### 7.4 重启特定服务

```cmd
# 重启 Web 服务
docker compose restart langfuse-web

# 重启 Worker 服务
docker compose restart langfuse-worker
```

### 7.5 查看资源使用情况

```cmd
docker stats
```

显示每个容器的 CPU、内存使用情况。按 `Ctrl + C` 退出。

### 7.6 进入容器内部（高级）

```cmd
# 进入 Web 容器
docker compose exec langfuse-web sh

# 进入 PostgreSQL 容器
docker compose exec postgres psql -U langfuse -d langfuse

# 退出容器
exit
```

---

## 8. 故障排查

### 8.1 常见问题

#### 问题 1：Docker Desktop 无法启动

**症状**：Docker 图标一直是橙色，或显示错误

**解决方案**：
1. 确保 WSL 2 已安装：
   ```powershell
   wsl --install
   wsl --set-default-version 2
   ```
2. 重启 Docker Desktop
3. 检查 Windows 更新，安装最新补丁
4. 查看 Docker Desktop 日志：Settings → Troubleshoot → Get diagnostic

#### 问题 2：端口被占用

**症状**：启动时显示 "port is already allocated"

**解决方案**：
1. 查找占用端口的程序：
   ```cmd
   netstat -ano | findstr :3000
   ```
2. 关闭占用端口的程序，或修改 `docker-compose.yml` 中的端口映射

#### 问题 3：容器启动失败

**症状**：`docker compose ps` 显示某些容器 "Exited"

**解决方案**：
1. 查看该容器的日志：
   ```cmd
   docker compose logs 容器名
   ```
2. 常见原因：
   - 内存不足：在 Docker Desktop Settings 中增加内存
   - 配置错误：检查 `.env` 文件

#### 问题 4：数据库迁移失败

**症状**：日志中显示 "Migration failed"

**解决方案**：
1. 完全重置（**会删除所有数据**）：
   ```cmd
   docker compose down -v
   docker compose up -d
   ```
2. 查看详细日志：
   ```cmd
   docker compose logs langfuse-web
   ```

#### 问题 5：无法访问 http://localhost:3000

**症状**：浏览器显示"无法访问此网站"

**解决方案**：
1. 确认服务正在运行：
   ```cmd
   docker compose ps
   ```
2. 检查防火墙是否阻止了 Docker
3. 尝试使用 `http://127.0.0.1:3000`
4. 查看 Web 服务日志：
   ```cmd
   docker compose logs langfuse-web
   ```

### 8.2 重置所有数据

**警告：这会删除所有数据！**

```cmd
# 停止并删除所有容器和卷
docker compose down -v

# 清理所有 Docker 资源（可选）
docker system prune -a --volumes

# 重新启动
docker compose up -d
```

### 8.3 更新到最新版本

```cmd
# 停止服务
docker compose down

# 拉取最新镜像
docker compose pull

# 重新启动
docker compose up -d
```

---

## 9. 数据备份

### 9.1 备份数据库

#### 备份 PostgreSQL

```cmd
# 导出数据库
docker compose exec -T postgres pg_dump -U langfuse langfuse > backup_postgres.sql
```

#### 恢复 PostgreSQL

```cmd
# 导入数据库
docker compose exec -T postgres psql -U langfuse langfuse < backup_postgres.sql
```

### 9.2 备份 MinIO 数据

MinIO 数据存储在 Docker 卷中，可以通过以下方式备份：

```cmd
# 查看卷位置
docker volume inspect langfuse_minio_data
```

或者使用 MinIO 客户端工具（mc）备份。

### 9.3 备份配置文件

定期备份这些文件：
- `.env` - 环境变量配置
- `docker-compose.yml` - Docker Compose 配置

将它们复制到安全的位置，例如：
```
C:\backups\langfuse\
```

---

## 10. 性能优化建议

### 10.1 调整 Docker 资源

如果系统资源充足，可以在 Docker Desktop Settings 中：
- 增加 CPU 核心数到 6-8 个
- 增加内存到 12-16GB
- 增加磁盘空间

### 10.2 调整环境变量

在 `.env` 中添加性能优化配置：

```bash
# PostgreSQL 连接池
DATABASE_URL="postgresql://langfuse:langfuse123@postgres:5432/langfuse?connection_limit=20"

# Worker 并发
# LANGFUSE_WORKER_CONCURRENCY=4
```

---

## 11. 卸载

如果你想完全卸载 Langfuse：

### 11.1 停止并删除容器和数据

```cmd
cd C:\Users\你的用户名\langfuse
docker compose down -v
```

### 11.2 删除镜像

```cmd
docker rmi langfuse/langfuse:3
docker rmi langfuse/langfuse-worker:3
docker rmi postgres:17
docker rmi clickhouse/clickhouse-server
docker rmi redis:7
docker rmi cgr.dev/chainguard/minio
```

### 11.3 删除文件夹

删除 `C:\Users\你的用户名\langfuse` 文件夹

---

## 附录 A：完整的 .env 配置示例（本地开发）

```bash
# 数据库
DATABASE_URL="postgresql://langfuse:langfuse123@postgres:5432/langfuse"
POSTGRES_USER=langfuse
POSTGRES_PASSWORD=langfuse123
POSTGRES_DB=langfuse

# 应用 URL
NEXTAUTH_URL="http://localhost:3000"

# 安全密钥（替换为你生成的值）
NEXTAUTH_SECRET="your-generated-nextauth-secret-here"
SALT="your-generated-salt-here"
ENCRYPTION_KEY="your-generated-encryption-key-here"

# ClickHouse
CLICKHOUSE_URL="http://clickhouse:8123"
CLICKHOUSE_MIGRATION_URL="clickhouse://clickhouse:9000"
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=clickhouse123
CLICKHOUSE_DB=default
CLICKHOUSE_CLUSTER_ENABLED=false

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_AUTH=redis123
REDIS_TLS_ENABLED=false

# MinIO
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=minio123

LANGFUSE_S3_EVENT_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_EVENT_UPLOAD_REGION=auto
LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=minio123
LANGFUSE_S3_EVENT_UPLOAD_ENDPOINT=http://minio:9000
LANGFUSE_S3_EVENT_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_EVENT_UPLOAD_PREFIX=events/

LANGFUSE_S3_MEDIA_UPLOAD_BUCKET=langfuse
LANGFUSE_S3_MEDIA_UPLOAD_REGION=auto
LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY=minio123
LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT=http://localhost:9090
LANGFUSE_S3_MEDIA_UPLOAD_FORCE_PATH_STYLE=true
LANGFUSE_S3_MEDIA_UPLOAD_PREFIX=media/

# 初始化
LANGFUSE_INIT_ORG_ID=my-org
LANGFUSE_INIT_ORG_NAME=我的组织
LANGFUSE_INIT_PROJECT_ID=my-project
LANGFUSE_INIT_PROJECT_NAME=我的项目
LANGFUSE_INIT_USER_EMAIL=admin@example.com
LANGFUSE_INIT_USER_NAME=管理员
LANGFUSE_INIT_USER_PASSWORD=admin123

# 功能
LANGFUSE_ENABLE_EXPERIMENTAL_FEATURES=true
TELEMETRY_ENABLED=true
LANGFUSE_LOG_LEVEL=info
```

---

## 附录 B：常用命令速查表

| 操作 | 命令 |
|------|------|
| 启动所有服务 | `docker compose up -d` |
| 停止所有服务 | `docker compose down` |
| 查看服务状态 | `docker compose ps` |
| 查看日志 | `docker compose logs -f` |
| 重启服务 | `docker compose restart` |
| 拉取最新镜像 | `docker compose pull` |
| 查看资源使用 | `docker stats` |
| 完全清理（删除数据） | `docker compose down -v` |
| 进入 Web 容器 | `docker compose exec langfuse-web sh` |
| 备份数据库 | `docker compose exec -T postgres pg_dump -U langfuse langfuse > backup.sql` |

---

## 附录 C：PowerShell 密钥生成脚本

将以下内容保存为 `generate-keys.ps1`，然后右键运行：

```powershell
# Langfuse 密钥生成脚本

Write-Host "正在生成 Langfuse 安全密钥..." -ForegroundColor Green
Write-Host ""

# NEXTAUTH_SECRET
$nextauthBytes = New-Object byte[] 32
$rng = [System.Security.Cryptography.RNGCryptoServiceProvider]::new()
$rng.GetBytes($nextauthBytes)
$nextauthSecret = [Convert]::ToBase64String($nextauthBytes)
Write-Host "NEXTAUTH_SECRET=`"$nextauthSecret`"" -ForegroundColor Yellow

# SALT
$saltBytes = New-Object byte[] 32
$rng.GetBytes($saltBytes)
$salt = [Convert]::ToBase64String($saltBytes)
Write-Host "SALT=`"$salt`"" -ForegroundColor Yellow

# ENCRYPTION_KEY
$encryptionBytes = New-Object byte[] 32
$rng.GetBytes($encryptionBytes)
$encryptionKey = ($encryptionBytes | ForEach-Object { $_.ToString("x2") }) -join ''
Write-Host "ENCRYPTION_KEY=`"$encryptionKey`"" -ForegroundColor Yellow

Write-Host ""
Write-Host "请复制上面的值到 .env 文件中！" -ForegroundColor Green
Write-Host ""
Pause
```

---

## 附录 D：图形化管理工具（可选）

### Portainer（Docker 可视化管理）

如果你不喜欢命令行，可以安装 Portainer：

```cmd
docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
```

然后访问 `http://localhost:9000` 进行图形化管理。

---

## 结语

现在你已经在 Windows 上成功部署了 Langfuse！

**快速启动步骤回顾**：
1. ✅ 安装 Docker Desktop
2. ✅ 下载配置文件
3. ✅ 配置 `.env`
4. ✅ 运行 `docker compose up -d`
5. ✅ 访问 `http://localhost:3000`

**需要帮助？**
- 官方文档：https://langfuse.com/docs
- GitHub Issues：https://github.com/langfuse/langfuse/issues
- Discord 社区：https://langfuse.com/discord

**遇到问题？**
查看本指南的"故障排查"章节，或在社区寻求帮助。

祝你使用愉快！
