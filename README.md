# SynPharm Backend

智互药研（SynPharm）AI 药物研发平台 —— **业务后端仓库**。

基于 Spring Boot 3 的业务 API 服务，负责用户认证（JWT）、预测任务编排、批量任务异步调度（RabbitMQ）、结果持久化（MySQL）与缓存（Redis），并作为网关调用 FastAPI 算法引擎完成 AI 推理。

## 技术栈

| 分类 | 选型 |
|------|------|
| 框架 | Spring Boot 3.3.5（Java 17） |
| ORM | MyBatis-Plus 3.5.5 |
| 数据库 | MySQL 8.0 |
| 缓存 | Redis 7（验证码 / Token 黑名单） |
| 消息队列 | RabbitMQ 3.13（批量任务异步调度） |
| 安全 | Spring Security + JJWT 0.12 |
| 算法调用 | WebClient（WebFlux）→ FastAPI |
| API 文档 | Knife4j 4.5（OpenAPI 3） |
| 其他 | Lombok、Actuator、Spring Mail（QQ 邮箱验证码） |

## 目录结构

```
synpharm-backend/
├── src/main/java/com/synpharm/
│   ├── config/           # Security / WebClient / RabbitMQ 等配置
│   ├── controller/       # REST API
│   ├── service/          # 业务逻辑
│   ├── model/            # 实体与 DTO
│   └── utils/            # JWT 等工具
├── src/main/resources/
│   ├── application.yml          # 公共配置
│   ├── application-dev.yml      # 本地/Docker 开发（环境变量化，含默认值）
│   └── application-docker.yml   # 生产容器配置
├── sql/                  # 建表脚本（MySQL 容器首次启动自动执行）
├── Dockerfile            # 多阶段构建（Maven 打包 → 精简 JRE 非 root 运行）
└── maven-settings.xml    # 阿里云镜像加速（容器构建用）
```

## 快速开始

### 方式一：Docker 开发环境（推荐，全栈联调）

与本仓库同级的 `synpharm-deploy` 提供一键编排。后端以 **Maven 源码模式**运行，集成 `spring-boot-devtools`：IDE 编译导致 `target/classes` 变更后自动重启，另暴露 **5005 远程调试端口**供 IDE attach。

```bash
docker compose --env-file .env -f synpharm-deploy/docker-compose.dev.yml up -d backend
# API:        http://localhost:7000
# 接口文档:    http://localhost:7000/doc.html（Knife4j）
# 健康检查:    http://localhost:7000/actuator/health
# 远程调试:    localhost:5005
```

### 方式二：本地独立运行

```bash
# 需本机已有 MySQL(3306) / Redis(6379) / RabbitMQ(5672)，或自行修改 application-dev.yml
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

`application-dev.yml` 的连接信息全部支持环境变量覆盖（如 `DB_HOST`、`RABBITMQ_HOST`），默认值面向本机开发。

## 配置说明

| Profile | 用途 | 关键差异 |
|---------|------|----------|
| `dev` | 本地 / Docker 开发 | 固定开发 JWT 密钥、`CAPTCHA_DEV_MODE` 可回显验证码、debug 日志 |
| `docker` | 生产容器 | JWT 密钥必须由 `JWT_SECRET` 注入（缺失即 fail-fast）、连接 compose 服务名 |

生产环境**必须**通过环境变量注入：`JWT_SECRET`、`DB_PASSWORD`、`RABBITMQ_PASSWORD` 等，参见 `synpharm-deploy/.env.example`。

## 与算法引擎的契约

后端通过 `fastapi.base-url` + `fastapi.api-key` 调用 FastAPI 引擎（DTI/DDI/PPI 推理）。算法接口的请求/响应模型以后端 `WebClientConfig` 与 FastAPI 的 `/openapi.json` 为准，两端独立演进、按契约联调。

## 构建与部署

```bash
mvn clean package -DskipTests        # 本地打包
docker build -t synpharm/backend:1.0.0 .   # 生产镜像
```

生产部署使用 `synpharm-deploy/docker-compose.yml`（纯镜像引用）。

## 相关仓库

| 仓库 | 说明 |
|------|------|
| [synpharm-frontend](https://github.com/Ljz4777/synpharm-frontend) | Vue 3 前端（调用本服务） |
| [synpharm-fastapi](https://github.com/Ljz4777/synpharm-fastapi) | FastAPI 算法引擎（本服务调用） |
| [synpharm-deploy](https://github.com/Ljz4777/synpharm-deploy) | Docker 编排、一键脚本与项目文档 |
