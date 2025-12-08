# Docker 部署故障排除指南

## 常见问题

### 1. 镜像拉取失败（网络问题）

**错误信息：**
```
failed to resolve source metadata for docker.io/library/nginx:alpine: failed to do request: Head "https://mirror.ccs.tencentyun.com/...": EOF
```

**原因：**
- Docker镜像源配置的服务器无法访问
- 网络连接问题
- 镜像源服务暂时不可用

**解决方案：**

#### 方案A：使用Docker官方源（推荐，如果网络允许）

检查并修改Docker镜像源配置：

**Windows (Docker Desktop):**
1. 打开 Docker Desktop
2. 进入 Settings → Docker Engine
3. 修改或删除 `registry-mirrors` 配置
4. 点击 "Apply & Restart"

**Linux:**
```bash
# 编辑Docker daemon配置
sudo nano /etc/docker/daemon.json

# 删除或注释掉 registry-mirrors，使用官方源
{
  "registry-mirrors": []
}

# 或者完全删除该配置项
{}

# 重启Docker服务
sudo systemctl restart docker
```

#### 方案B：更换为其他可用的镜像源

**国内可用的Docker镜像源：**

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

**配置步骤（Linux）：**
```bash
# 创建或编辑配置文件
sudo nano /etc/docker/daemon.json

# 添加镜像源配置
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}

# 重启Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证配置
docker info | grep -A 10 "Registry Mirrors"
```

#### 方案C：手动拉取镜像后构建

```bash
# 手动拉取基础镜像
docker pull node:20-alpine
docker pull nginx:alpine

# 然后再构建
docker-compose -f docker-compose.nginx.yml up -d --build
```

#### 方案D：使用代理

如果有可用的HTTP代理：

```bash
# 设置代理环境变量
export HTTP_PROXY=http://your-proxy:port
export HTTPS_PROXY=http://your-proxy:port

# 然后构建
docker-compose -f docker-compose.nginx.yml up -d --build
```

### 2. docker-compose version 警告

**警告信息：**
```
the attribute `version` is obsolete, it will be ignored
```

**原因：**
- Docker Compose v2 不再需要 `version` 字段
- 新版本会自动检测compose文件格式

**解决方案：**
- 已自动修复：已从所有docker-compose文件中移除 `version` 字段
- 如果仍有警告，请确保使用最新版本的docker-compose

### 3. 构建缓存问题

**问题：**
构建时使用了旧的缓存，导致部署了旧版本

**解决方案：**
```bash
# 强制重新构建，不使用缓存
docker-compose -f docker-compose.nginx.yml up -d --build --no-cache

# 或者先清理构建缓存
docker builder prune
docker-compose -f docker-compose.nginx.yml up -d --build
```

### 4. 端口被占用

**错误信息：**
```
Error: bind: address already in use
```

**解决方案：**
```bash
# 检查端口占用
# Linux
sudo netstat -tulpn | grep 8080
# 或
sudo lsof -i :8080

# Windows PowerShell
netstat -ano | findstr :8080

# 停止占用端口的容器
docker ps
docker stop <container-id>

# 或修改docker-compose.yml中的端口映射
```

### 5. 权限问题（Linux）

**错误信息：**
```
permission denied while trying to connect to the Docker daemon socket
```

**解决方案：**
```bash
# 将当前用户添加到docker组
sudo usermod -aG docker $USER

# 重新登录或执行
newgrp docker

# 验证权限
docker ps
```

## 验证部署

部署成功后，验证服务：

```bash
# 检查容器状态
docker ps

# 查看容器日志
docker logs redflow-nginx
# 或
docker-compose -f docker-compose.nginx.yml logs -f

# 测试健康检查
curl http://localhost:8080/health

# 访问应用
curl http://localhost:8080
```

## 快速修复脚本

如果遇到镜像源问题，可以使用以下脚本快速修复：

**Linux:**
```bash
#!/bin/bash
# 备份原配置
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak 2>/dev/null || true

# 配置中科大镜像源
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
EOF

# 重启Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

echo "Docker镜像源已更新，请重新尝试构建"
```

## 联系支持

如果以上方案都无法解决问题，请检查：
1. Docker版本：`docker --version` 和 `docker-compose --version`
2. 网络连接：`ping docker.io` 或 `ping registry-1.docker.io`
3. 防火墙设置
4. Docker服务状态：`sudo systemctl status docker`

