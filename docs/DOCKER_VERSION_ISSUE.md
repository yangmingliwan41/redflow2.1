# Docker部署版本问题排查指南

## 问题描述

**症状**: 
- 本地部署和验证都没问题
- 服务器端Docker容器显示镜像标签是 `2.1`
- 但应用功能和内容还是 `2.0` 的

**根本原因**:
Docker构建时使用了**缓存**，导致虽然镜像标签更新了，但容器内的代码还是旧的。

## 问题原因分析

### 1. Docker构建缓存机制

Docker在构建镜像时会使用缓存机制：
- 如果 `package.json` 没有变化，Docker会使用缓存的 `npm ci` 层
- 如果源代码文件的时间戳没变，Docker会使用缓存的 `COPY . .` 层
- 即使你更新了代码，如果Docker认为某些层没变化，就会使用缓存的旧层

### 2. 为什么镜像标签是2.1但内容是2.0？

- 镜像标签（`redflow-nginx:2.1`）只是给镜像起的名字
- 如果构建时使用了缓存，镜像内的代码可能还是旧的
- 标签更新了，但内容没有更新

## 解决方案

### 方案1: 使用强制部署脚本（推荐）

```bash
# 在服务器上执行
cd /path/to/redflow
chmod +x deploy-force.sh
./deploy-force.sh
```

这个脚本会：
1. 拉取最新代码（如果有git仓库）
2. 停止并删除旧容器
3. 清理旧镜像和构建缓存
4. 强制重新构建（不使用缓存）
5. 启动新容器

### 方案2: 手动强制重新构建

```bash
# 1. 确保代码是最新的
cd /path/to/redflow
git pull  # 如果有git仓库

# 2. 停止并删除旧容器
docker-compose -f docker-compose.nginx.yml down

# 3. 删除旧镜像（可选）
docker rmi redflow-nginx:2.1

# 4. 清理构建缓存
docker builder prune -f

# 5. 强制重新构建（关键：--no-cache）
docker-compose -f docker-compose.nginx.yml build --no-cache

# 6. 启动新容器
docker-compose -f docker-compose.nginx.yml up -d
```

### 方案3: 更新deploy.sh脚本

已更新 `deploy.sh` 脚本，现在会自动：
- 拉取最新代码
- 使用 `--no-cache` 强制重新构建
- 停止旧容器后再启动新容器

```bash
chmod +x deploy.sh
./deploy.sh docker
```

## 验证部署是否成功

### 1. 检查容器状态
```bash
docker ps -a | grep redflow
```

### 2. 检查容器内的文件
```bash
# 进入容器查看文件
docker exec redflow-nginx ls -la /usr/share/nginx/html

# 查看index.html的修改时间
docker exec redflow-nginx stat /usr/share/nginx/html/index.html
```

### 3. 检查应用版本
- 在浏览器中打开应用
- 查看侧边栏底部是否显示 `v2.1`
- 检查功能是否已更新（如新增的下载功能等）

### 4. 查看构建日志
```bash
docker-compose -f docker-compose.nginx.yml logs --tail=50
```

## 常见错误操作

### ❌ 错误1: 只使用 --build 参数
```bash
# 这样可能仍使用缓存
docker-compose -f docker-compose.nginx.yml up -d --build
```

### ❌ 错误2: 没有停止旧容器
```bash
# 这样可能仍使用旧容器
docker-compose -f docker-compose.nginx.yml build
docker-compose -f docker-compose.nginx.yml up -d
```

### ❌ 错误3: 只更新代码不重新构建
```bash
# 只更新代码，不重新构建镜像
git pull
# 忘记重新构建镜像
```

## 正确的更新流程

每次更新代码后，必须执行以下步骤：

```bash
# 1. 更新代码
git pull

# 2. 强制重新构建（必须使用 --no-cache）
docker-compose -f docker-compose.nginx.yml down
docker-compose -f docker-compose.nginx.yml build --no-cache
docker-compose -f docker-compose.nginx.yml up -d

# 3. 验证
docker ps -a | grep redflow
curl http://localhost:8080
```

或者直接使用强制部署脚本：
```bash
./deploy-force.sh
```

## 预防措施

### 1. 在CI/CD中强制不使用缓存

如果使用GitHub Actions或其他CI/CD，在构建时添加：
```yaml
- name: Build Docker image
  run: |
    docker-compose build --no-cache
```

### 2. 添加构建时间戳

可以在Dockerfile中添加构建时间戳，方便验证：
```dockerfile
ARG BUILD_DATE
LABEL build-date="${BUILD_DATE}"
```

### 3. 使用版本号作为构建参数

确保每次构建都传入新的版本号：
```bash
docker-compose build --build-arg VERSION=2.1.0 --no-cache
```

## 总结

**核心要点**:
1. ✅ 更新代码后必须使用 `--no-cache` 强制重新构建
2. ✅ 构建前先停止并删除旧容器
3. ✅ 使用 `deploy-force.sh` 脚本可以一键完成所有步骤
4. ✅ 验证时检查容器内的文件时间戳和应用功能

**记住**: Docker的缓存机制是为了加速构建，但在更新代码时，缓存会导致部署旧版本。必须使用 `--no-cache` 强制重新构建！

