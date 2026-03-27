---
name: docker-patterns
description: Docker 与 Docker Compose 的本地开发模式、容器安全、网络、卷策略，以及多服务编排最佳实践。
origin: ECC
---

# Docker Patterns

Docker 与 Docker Compose 的本地开发最佳实践。

## 何时启用

- 为本地开发设置 Docker Compose
- 设计多容器架构
- 故障排查容器网络或卷问题
- 审核 Dockerfile 的安全性与体积
- 将本地开发迁移到容器化工作流

## 本地开发的 Docker Compose

### 标准 Web 应用栈

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: dev                     # 使用多阶段 Dockerfile 的 dev 阶段
    ports:
      - "3000:3000"
    volumes:
      - .:/app                        # 热重载绑定挂载
      - /app/node_modules             # 匿名卷 — 保留容器依赖
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db:5432/app_dev
      - REDIS_URL=redis://redis:6379/0
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

  mailpit:                            # 本地邮件测试
    image: axllent/mailpit
    ports:
      - "8025:8025"                   # Web UI
      - "1025:1025"                   # SMTP

volumes:
  pgdata:
  redisdata:
```

### 本地开发与生产 Dockerfile

```dockerfile
# Stage: dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage: dev (热加载、调试工具)
FROM node:22-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# Stage: build
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --production

# Stage: production (最小镜像)
FROM node:22-alpine AS production
WORKDIR /app
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001
USER appuser
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./
ENV NODE_ENV=production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Override 文件

```yaml
# docker-compose.override.yml (自动加载，开发环境专用设置)
services:
  app:
    environment:
      - DEBUG=app:*
      - LOG_LEVEL=debug
    ports:
      - "9229:9229"                   # Node.js 调试器

# docker-compose.prod.yml (生产环境用)
services:
  app:
    build:
      target: production
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

```bash
# 开发环境（自动加载覆盖）
docker compose up

# 生产环境
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## 网络

### 服务发现

在同一 Compose 网络中的服务按服务名解析：
```
# 来自 app 容器：
postgres://postgres:postgres@db:5432/app_dev    # db 指向 db 容器
redis://redis:6379/0                             # redis 指向 redis 容器
```

### 自定义网络

```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net              # 仅从 api 访问，不直接暴露给前端

networks:
  frontend-net:
  backend-net:
```

### 仅暴露所需部分

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5432:5432"   # 仅对主机可访问，不对网络暴露
    # 生产环境中完全不暴露端口 -- 仅在 Docker 网络内可访问
```

## 卷策略

```yaml
volumes:
  # 命名卷：跨容器重启保持数据，由 Docker 管理
  pgdata:

  # 绑定挂载：将主机目录映射到容器（开发用）
  # - ./src:/app/src

  # 匿名卷：保留绑定挂载覆盖下容器生成内容
  # - /app/node_modules
```

### 常用模式

```yaml
services:
  app:
    volumes:
      - .:/app                   # 源代码（绑定挂载便于热Reload）
      - /app/node_modules        # 保护容器的 node_modules 免受主机覆盖
      - /app/.next               # 保护构建缓存

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data          # 持久数据
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # 初始化脚本
```

## 容器安全

### Dockerfile 加固

```dockerfile
# 1. 使用特定标签，而非 latest
FROM node:22.12-alpine3.20

# 2. 以非 root 用户运行
RUN addgroup -g 1001 -S app && adduser -S app -u 1001
USER app

# 3. 在 compose 中降权
# 4. 只读根文件系统（尽可能）
# 5. 镜像层中不包含机密
```

### Compose 安全性

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /app/.cache
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE          # 仅在绑定端口 < 1024 时使用
```

### 秘密管理

```yaml
# 好：使用环境变量（运行时注入）
services:
  app:
    env_file:
      - .env                     # 不要将 .env 提交到 git
    environment:
      - API_KEY                  # 继承自主机环境

# 好：Docker secrets（Swarm 模式）
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password

# 不好：在镜像中硬编码
# ENV API_KEY=sk-proj-xxxxx      # 绝不这样做
```

## .dockerignore

```
node_modules
.git
.env
.env.*
dist
coverage
*.log
.next
.cache
docker-compose*.yml
Dockerfile*
README.md
tests/
```

## 调试

### 常用命令

```bash
# 查看日志
docker compose logs -f app
docker compose logs --tail=50 db

# 在容器中执行命令
docker compose exec app sh
docker compose exec db psql -U postgres

# 检查
docker compose ps
docker compose top
docker stats

# 重建
docker compose up --build
docker compose build --no-cache app

# 清理
docker compose down
docker compose down -v
docker system prune
```

### 调试网络问题

```bash
# 检查容器内 DNS 解析
docker compose exec app nslookup db

# 检查连通性
docker compose exec app wget -qO- http://api:3000/health

# Inspect network
docker network ls
docker network inspect <project>_default
```

## 反模式

```
# BAD: 在生产环境中直接使用 docker compose 而不进行编排
# 使用 Kubernetes、ECS、或 Docker Swarm 管理生产环境的多容器工作负载

# BAD: 将数据存放在容器中而不使用卷
# 容器是临时的——没有卷数据将在重启时丢失

# BAD: 以 root 用户运行
# 始终创建并使用非 root 用户

# BAD: 使用 latest 标签
# 为可重复构建固定版本

# BAD: 一个容器中承载所有服务
# 将职责拆分成独立的容器

# BAD: 将机密写在 docker-compose.yml 中
# 使用 .env 文件（gitignore 保护）或 Docker secrets
```
