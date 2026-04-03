# Sea-RideTheWind 项目入门指南

> 本文档面向初次接触本项目的开发者，帮助你快速理解项目背景、技术栈、目录结构、开发流程与部署方式。

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 技术栈一览](#2-技术栈一览)
- [3. 目录结构](#3-目录结构)
- [4. 核心概念](#4-核心概念)
  - [4.1 go-zero 框架](#41-go-zero-框架)
  - [4.2 API 层与 RPC 层的分离](#42-api-层与-rpc-层的分离)
  - [4.3 服务发现与 etcd](#43-服务发现与-etcd)
  - [4.4 消息队列（Kafka）](#44-消息队列kafka)
- [5. 环境准备](#5-环境准备)
- [6. 快速启动](#6-快速启动)
  - [6.1 启动基础设施](#61-启动基础设施)
  - [6.2 本地运行单个服务](#62-本地运行单个服务)
  - [6.3 使用 Docker 容器运行服务](#63-使用-docker-容器运行服务)
- [7. 以 article 服务为例理解代码结构](#7-以-article-服务为例理解代码结构)
  - [7.1 API 层（HTTP 网关）](#71-api-层http-网关)
  - [7.2 RPC 层（gRPC 服务）](#72-rpc-层grpc-服务)
- [8. 请求流转全链路](#8-请求流转全链路)
- [9. 代码生成（goctl）](#9-代码生成goctl)
  - [9.1 从 .api 文件生成 HTTP 服务](#91-从-api-文件生成-http-服务)
  - [9.2 从 .proto 文件生成 gRPC 服务](#92-从-proto-文件生成-grpc-服务)
- [10. 配置说明](#10-配置说明)
  - [10.1 API 层配置示例](#101-api-层配置示例)
  - [10.2 RPC 层配置示例](#102-rpc-层配置示例)
  - [10.3 关键配置项说明](#103-关键配置项说明)
- [11. 部署指南（manage.sh）](#11-部署指南managesh)
- [12. 可观测性](#12-可观测性)
- [13. 测试](#13-测试)
- [14. 各服务职责速查](#14-各服务职责速查)
- [15. 端口速查](#15-端口速查)
- [16. 常见问题](#16-常见问题)
- [17. 进一步阅读](#17-进一步阅读)

---

## 1. 项目概述

**Sea-RideTheWind**（乘风）是一个基于 Go 和 [go-zero](https://go-zero.dev/) 微服务框架构建的社区/内容平台后端。它与推荐系统 **Sea-BreakTheWaves**（破浪）配合，共同构成"识海"社区的完整后端体系——寓意"乘风破浪"。

项目涵盖以下业务能力：

- **用户体系**：注册、登录、查询、更新、注销
- **管理员体系**：管理员登录、用户管理、封禁/解禁
- **内容体系**：文章的创建、编辑、删除、列表、图片上传
- **互动体系**：评论、点赞/点踩、收藏、关注、消息
- **运营体系**：任务查询、积分、热点内容
- **安全体系**：风控相关能力预留

---

## 2. 技术栈一览

| 分类 | 技术 | 用途 |
|------|------|------|
| 编程语言 | Go 1.25+ | 全部服务端代码 |
| 微服务框架 | [go-zero](https://github.com/zeromicro/go-zero) | HTTP（rest）+ gRPC（zrpc） |
| RPC / 序列化 | gRPC + Protocol Buffers | 服务间通信 |
| 代码生成 | goctl 1.9.2 | 从 `.api` / `.proto` 生成脚手架代码 |
| 数据库 | PostgreSQL | 结构化业务数据存储 |
| ORM | GORM | 数据访问层 |
| 缓存 | Redis | 缓存、会话、分布式锁 |
| 消息队列 | Kafka | 事件总线、异步解耦 |
| 延时队列 | Beanstalkd | 延迟任务 |
| 对象存储 | MinIO（S3 兼容） | 图片、文件存储 |
| 向量数据库 | Milvus | Embedding / 相似度检索 |
| 图数据库 | Neo4j | 关系网络、推荐 |
| 搜索引擎 | Elasticsearch + Kibana | 日志检索与全文搜索 |
| 服务发现 | etcd | 微服务注册与发现 |
| 可观测性 | OpenTelemetry → Jaeger、Prometheus、Grafana | 链路追踪、指标监控 |
| 容器化 | Docker + Docker Compose | 基础设施编排与服务部署 |

---

## 3. 目录结构

```
/
├── README.md                          # 项目总览
├── docker-compose.yaml                # 基础设施编排
├── dockerfile                         # 服务构建镜像
├── manage.sh                          # 多机部署管理脚本
├── go.sum                             # Go 依赖校验文件
│
├── api/                               # HTTP 接口定义（go-zero .api 格式）
│   ├── article.api
│   ├── comment.api
│   ├── user_center.api
│   ├── admin_center.api
│   └── ...
│
├── proto/                             # gRPC 接口定义（.proto 格式）
│   ├── user.proto
│   ├── article.proto
│   ├── comment.proto
│   └── ...
│
├── doc/                               # 文档
│   ├── docker.md                      # Docker 组件说明
│   └── getting-started.md             # 本文档
│
├── prometheus/                        # Prometheus 配置
│   └── settings.yml
│
└── service/                           # 所有微服务代码
    ├── common/                        # 公共工具库
    │   ├── logger/                    #   结构化日志工具
    │   ├── response/                  #   统一响应封装
    │   └── snowflake/                 #   分布式 ID 生成
    │
    ├── article/                       # 文章服务
    │   ├── api/                       #   HTTP 网关层
    │   │   ├── article.go             #     入口 main
    │   │   ├── etc/                   #     配置文件
    │   │   └── internal/              #     业务代码
    │   │       ├── config/            #       配置结构体
    │   │       ├── handler/           #       路由处理器
    │   │       ├── logic/             #       业务逻辑
    │   │       ├── svc/               #       服务上下文（依赖注入）
    │   │       └── types/             #       请求/响应类型
    │   ├── rpc/                       #   gRPC 服务层
    │   │   ├── article.go             #     入口 main
    │   │   ├── etc/                   #     配置文件
    │   │   ├── pb/                    #     protobuf 生成代码
    │   │   └── internal/              #     业务代码
    │   │       ├── config/
    │   │       ├── logic/
    │   │       ├── model/             #       数据模型 / ORM
    │   │       ├── mqs/               #       消息队列消费者
    │   │       ├── server/
    │   │       └── svc/
    │   └── common/                    #   文章服务内部共享代码
    │
    ├── comment/                       # 评论服务（结构同 article）
    ├── like/                          # 点赞服务
    ├── follow/                        # 关注服务
    ├── favorite/                      # 收藏服务
    ├── message/                       # 消息服务
    ├── task/                          # 任务服务
    ├── hot/                           # 热点服务（含 heavykeeper 算法）
    ├── points/                        # 积分服务（仅 RPC）
    ├── security/                      # 安全服务（仅 RPC）
    └── user/                          # 用户域
        ├── user/                      #   用户中心（API + RPC）
        └── admin/                     #   管理员中心（API + RPC）
```

---

## 4. 核心概念

### 4.1 go-zero 框架

[go-zero](https://go-zero.dev/) 是一个集成了服务治理、限流、熔断、超时控制、链路追踪等能力的 Go 微服务框架。它提供两种服务模式：

- **`rest`**：HTTP RESTful 服务，用于对外暴露 API
- **`zrpc`**：基于 gRPC 的 RPC 服务，用于服务间内部通信

go-zero 还附带代码生成工具 **goctl**，可以从 `.api` 和 `.proto` 定义文件自动生成路由、Handler、Logic、Config 等脚手架代码。

### 4.2 API 层与 RPC 层的分离

本项目中，每个业务域通常拆分为两层：

```
客户端（浏览器/App）
    │
    ▼
┌──────────┐   HTTP/REST    ┌──────────────┐
│  API 层  │ ◄────────────► │    前端/客户端  │
│ (网关)   │                └──────────────┘
└────┬─────┘
     │ gRPC
     ▼
┌──────────┐
│  RPC 层  │ ◄──► PostgreSQL / Redis / Kafka / ...
│ (核心逻辑)│
└──────────┘
```

- **API 层**：处理 HTTP 请求，做参数校验、JWT 鉴权、请求转发。不包含核心业务逻辑，通过 gRPC 调用 RPC 层。
- **RPC 层**：承载核心业务逻辑、数据库访问、消息队列交互。通过 etcd 注册自身地址，供 API 层发现并调用。

这种分离使得 API 层可以聚合多个 RPC 服务的能力（如文章 API 同时调用 article RPC、user RPC、security RPC），同时 RPC 层可以被多个 API 或其他 RPC 复用。

### 4.3 服务发现与 etcd

服务间通过 **etcd** 实现注册与发现：

1. RPC 服务启动时，将自身地址注册到 etcd（Key 如 `article.rpc`）
2. API 层启动时，根据配置中的 etcd 地址和 Key 查找目标 RPC 服务
3. go-zero 内置负载均衡，自动在多实例间分配请求

### 4.4 消息队列（Kafka）

部分服务使用 Kafka 实现异步处理：

- **article RPC**：文章审核消费者、文章同步结果消费者、Outbox 发送
- **like RPC**：有独立的消息队列消费配置
- **task**：日志通过 Filebeat 发送到 Kafka topic `task.logs.raw`

---

## 5. 环境准备

开发本项目需要以下工具：

| 工具 | 版本要求 | 用途 |
|------|---------|------|
| **Go** | 1.25+ | 编译运行服务 |
| **Docker** | 最新稳定版 | 运行基础设施 |
| **Docker Compose** | v2 (`docker compose`) | 编排基础设施容器 |
| **goctl** | 1.9.2+ | 代码生成 |
| **protoc** | 最新稳定版 | Protocol Buffers 编译器（goctl 内部调用） |
| **protoc-gen-go / protoc-gen-go-grpc** | 最新稳定版 | Go gRPC 代码生成插件 |

### 安装 goctl

```bash
go install github.com/zeromicro/go-zero/tools/goctl@latest
```

### 安装 protoc 插件

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

### 验证环境

```bash
go version          # 确认 Go 版本 >= 1.25
goctl --version     # 确认 goctl 可用
docker compose version  # 确认 Docker Compose v2 可用
```

---

## 6. 快速启动

### 6.1 启动基础设施

项目所有外部依赖（PostgreSQL、Redis、Kafka、etcd 等）都可以通过 Docker Compose 一键启动：

```bash
# 在项目根目录
docker compose up -d etcd postgres redis kafka minio
```

启动后，关键服务的访问地址：

| 服务 | 地址 | 备注 |
|------|------|------|
| etcd | `127.0.0.1:32379` | 服务发现 |
| PostgreSQL | `127.0.0.1:35432` | 用户 `admin`，密码 `Sea-TryGo`，数据库 `first_db` |
| Redis | `127.0.0.1:36379` | 无密码 |
| Kafka | `127.0.0.1:39092` | PLAINTEXT |
| MinIO | `127.0.0.1:39000` (API) / `127.0.0.1:39001` (Console) | 用户 `minioadmin` / `minioadmin` |

如果需要完整的可观测性栈（Prometheus、Grafana、Jaeger 等），可以启动全部服务：

```bash
docker compose up -d
```

> 更多基础设施组件的端口和账号密码信息，请参阅 [`doc/docker.md`](./docker.md)。

### 6.2 本地运行单个服务

以 article 服务为例，先启动 RPC 层，再启动 API 层：

```bash
# 1. 启动 article RPC（需要基础设施已运行）
cd service/article/rpc
go run article.go -f etc/article.yaml

# 2. 在新终端，启动 article API
cd service/article/api
go run article.go -f etc/article-api.yaml
```

> **注意**：API 层通常依赖一个或多个 RPC 服务，请确保相关 RPC 已启动。例如 article API 依赖 article RPC、user RPC 和 security RPC。

### 6.3 使用 Docker 容器运行服务

项目提供了 `manage.sh` 脚本进行容器化部署：

```bash
# 1. 首先在 manage.sh 顶部填写 INFRA_HOST（基础设施机器 IP）
# 2. 构建单个服务镜像
NODE_ROLE=app ./manage.sh build article

# 3. 启动单个服务
NODE_ROLE=app ./manage.sh start article

# 4. 查看服务状态
NODE_ROLE=app ./manage.sh status article

# 5. 查看服务日志
NODE_ROLE=app ./manage.sh logs article

# 6. 构建并启动全部服务
NODE_ROLE=app ./manage.sh build all
NODE_ROLE=app ./manage.sh start all
```

---

## 7. 以 article 服务为例理解代码结构

### 7.1 API 层（HTTP 网关）

位置：`service/article/api/`

```
service/article/api/
├── article.go                         # main 入口
├── etc/
│   └── article-api.yaml               # 配置文件
└── internal/
    ├── config/
    │   └── config.go                  # 配置结构体定义
    ├── handler/
    │   ├── routes.go                  # 路由注册（goctl 生成，勿手动修改）
    │   └── article/
    │       ├── create_article_handler.go
    │       ├── get_article_handler.go
    │       └── ...                    # 每个 API 一个 handler 文件
    ├── logic/
    │   └── article/
    │       ├── create_article_logic.go
    │       ├── get_article_logic.go
    │       └── ...                    # 核心业务逻辑在这里编写
    ├── svc/
    │   └── service_context.go         # 服务上下文（注入 RPC 客户端等依赖）
    └── types/
        └── types.go                   # 请求/响应结构体（goctl 生成）
```

**关键代码说明：**

**入口 `article.go`**：加载配置 → 初始化日志 → 创建 HTTP Server → 注册路由 → 启动服务。

**路由注册 `handler/routes.go`**：由 goctl 根据 `.api` 文件生成。它将 HTTP 路径映射到对应的 Handler 函数，并配置 JWT 鉴权、超时、请求大小限制等中间件。

**服务上下文 `svc/service_context.go`**：在这里初始化并注入 RPC 客户端等依赖。所有 Logic 通过 ServiceContext 获取依赖，这是 go-zero 的依赖注入方式：

```go
type ServiceContext struct {
    Config      config.Config
    ArticleRpc  articleservice.ArticleService    // article RPC 客户端
    SecurityRpc imagesecurityservice.ImageSecurityService  // security RPC 客户端
    UserRpc     userservice.UserService          // user RPC 客户端
}
```

**业务逻辑 `logic/article/create_article_logic.go`**：这是你最常编写的代码。每个 Logic 文件对应一个 API 接口。Logic 负责：
1. 从 Context 提取用户信息（JWT 解码后注入）
2. 组装 RPC 请求参数
3. 调用 RPC 层
4. 处理错误与返回

### 7.2 RPC 层（gRPC 服务）

位置：`service/article/rpc/`

```
service/article/rpc/
├── article.go                         # main 入口
├── etc/
│   └── article.yaml                   # 配置文件
├── pb/                                # protobuf 生成的 Go 代码
│   ├── article.pb.go
│   └── article_grpc.pb.go
├── articleservice/                     # RPC 客户端封装（供其他服务调用）
│   └── article_service.go
└── internal/
    ├── config/
    │   └── config.go                  # 配置（含数据库、Kafka、Outbox 等）
    ├── logic/                         # gRPC 方法实现
    │   ├── create_article_logic.go
    │   └── ...
    ├── model/                         # 数据模型 / ORM 层
    ├── mqs/                           # 消息队列消费者
    ├── server/
    │   └── article_service_server.go  # gRPC Server 实现（调度 Logic）
    └── svc/
        └── service_context.go         # 服务上下文（注入 DB、Redis 等）
```

RPC 层的 main 入口更为丰富：除了启动 gRPC Server 外，还可能启动 Kafka 消费者和定时任务（如 Outbox 中继器）。

---

## 8. 请求流转全链路

以"创建文章"为例，一个请求的完整链路：

```
1. 客户端发起 POST /v1/article（携带 JWT Token）
         │
         ▼
2. go-zero rest Server 接收请求
   - 解析路由 → 匹配 CreateArticleHandler
   - JWT 中间件校验 Token，将用户 ID 写入 Context
         │
         ▼
3. CreateArticleHandler
   - 解析请求体（JSON → types.CreateArticleReq）
   - 调用 CreateArticleLogic.CreateArticle()
         │
         ▼
4. CreateArticleLogic（API 层）
   - 从 Context 提取用户 ID
   - 构造 gRPC 请求
   - 通过 svcCtx.ArticleRpc.CreateArticle() 调用 RPC
         │
         ▼  gRPC（经 etcd 服务发现）
5. ArticleServiceServer（RPC 层）
   - 调用 CreateArticleLogic.CreateArticle()
         │
         ▼
6. CreateArticleLogic（RPC 层）
   - 业务校验
   - 写入 PostgreSQL
   - 发送 Kafka 消息（触发审核等异步流程）
   - 返回 gRPC Response
         │
         ▼  原路返回
7. API 层将 gRPC Response 转为 HTTP JSON Response
   → 返回客户端
```

---

## 9. 代码生成（goctl）

### 9.1 从 .api 文件生成 HTTP 服务

`.api` 文件（位于 `api/` 目录）定义了 HTTP 接口的请求/响应类型和路由规则。示例：

```
// api/article.api 片段
type CreateArticleReq {
    Title   string `json:"title"`
    Content string `json:"content"`
}

@server (
    prefix: /v1
    jwt:    Auth
)
service article-api {
    @handler CreateArticle
    post /article (CreateArticleReq) returns (CreateArticleResp)
}
```

使用 goctl 生成代码（暂不需要手动执行，仅作了解）：

```bash
goctl api go -api api/article.api -dir service/article/api -style gozero
```

生成的文件中，`handler/` 和 `types/` 一般不需要手动修改，**你的主要工作在 `logic/` 目录下**。

### 9.2 从 .proto 文件生成 gRPC 服务

`.proto` 文件（位于 `proto/` 目录）定义了 gRPC 服务接口。使用 goctl 生成：

```bash
goctl rpc protoc ./proto/article.proto \
  --go_out=./service/article/rpc \
  --go-grpc_out=./service/article/rpc \
  --zrpc_out=./service/article/rpc \
  --style=go_zero
```

> 各服务的完整生成命令可参考 [`help.md`](../help.md) 和 [`proto/readme.md`](../proto/readme.md)。

---

## 10. 配置说明

每个服务在 `etc/` 目录下有一个或多个 YAML 配置文件。

### 10.1 API 层配置示例

以 `service/article/api/etc/article-api.yaml` 为例：

```yaml
Name: article-api          # 服务名
Host: 0.0.0.0              # 监听地址
Port: 8889                 # 监听端口
Timeout: 70000             # 请求超时（毫秒）
MaxBytes: 10485760         # 最大请求体（10MB）

ArticleRpcConf:            # article RPC 连接配置
  Etcd:
    Hosts:
      - 127.0.0.1:32379    # etcd 地址
    Key: article.rpc        # RPC 在 etcd 中的注册 Key
  Timeout: 70000

Auth:                      # JWT 配置
  AccessSecret: ae0536f9-6450-4606-8e13-5a19ed505da0
  AccessExpire: 31536000   # Token 有效期（秒）

Log:                       # 日志配置
  ServiceName: article-api
  Mode: file
  Path: ../../../log/article-api

Telemetry:                 # OpenTelemetry 配置
  Name: article-api
  Endpoint: localhost:34317
  Sampler: 1.0
  Batcher: otlpgrpc
```

### 10.2 RPC 层配置示例

RPC 层配置通常还包含数据库、Redis、Kafka 等连接信息：

```yaml
Name: article.rpc
ListenOn: 0.0.0.0:9001
Etcd:
  Hosts:
    - 127.0.0.1:32379
  Key: article.rpc

Postgres:
  Host: "127.0.0.1"
  Port: "35432"
  User: admin
  Password: "Sea-TryGo"
  DBName: first_db

Log:
  Mode: file
  Path: ../../../log/article-rpc
```

### 10.3 关键配置项说明

| 配置项 | 说明 |
|--------|------|
| `Etcd.Hosts` | etcd 集群地址列表 |
| `Etcd.Key` | RPC 服务在 etcd 中的注册键 |
| `Auth.AccessSecret` | JWT 签名密钥（API 层和 User RPC 须一致） |
| `Postgres.*` | PostgreSQL 连接参数 |
| `KqConsumerConf` / `KqPusherConf` | Kafka 消费者/生产者配置 |
| `Telemetry.Endpoint` | OpenTelemetry Collector（Jaeger）地址 |

---

## 11. 部署指南（manage.sh）

`manage.sh` 是项目提供的多机部署管理脚本，支持两种角色：

| 角色 | 说明 |
|------|------|
| `NODE_ROLE=infra` | 基础设施机器：通过 Docker Compose 启动 etcd、PostgreSQL、Redis、Kafka、MinIO |
| `NODE_ROLE=app` | 业务服务机器：构建并启动各个 Go 微服务容器 |

### 使用步骤

**1. 配置基础设施地址**

编辑 `manage.sh` 顶部，填写 `INFRA_HOST`（基础设施机器的 IP）：

```bash
INFRA_HOST="10.0.0.12"  # 替换为你的实际 IP
```

**2. 在基础设施机器上启动 infra**

```bash
NODE_ROLE=infra ./manage.sh start
```

**3. 在业务机器上构建和启动服务**

```bash
# 构建全部服务镜像
NODE_ROLE=app ./manage.sh build all

# 启动全部服务
NODE_ROLE=app ./manage.sh start all

# 或只启动特定服务
NODE_ROLE=app ./manage.sh start article
```

**4. 常用运维命令**

```bash
NODE_ROLE=app ./manage.sh status all     # 查看所有服务状态
NODE_ROLE=app ./manage.sh logs article    # 查看 article 服务日志
NODE_ROLE=app ./manage.sh restart article # 重启 article 服务
NODE_ROLE=app ./manage.sh stop all        # 停止所有服务
NODE_ROLE=app ./manage.sh doctor article  # 诊断 article 服务
```

### 服务端口映射

| 服务 | API 端口（宿主机:容器） | RPC 端口（宿主机:容器） |
|------|------------------------|------------------------|
| article | 18889:8889 | 19001:9001 |
| comment | 18888:8888 | 19002:9001 |
| like | 18887:8887 | 18082:8082 |
| follow | 18891:8891 | 18086:8082 |
| favorite | 18890:8890 | 18088:8082 |
| message | 18892:8892 | 18087:8082 |
| task | 18886:8888 | 19005:9005 |
| user | 18885:8888 | 19004:9004 |
| admin | 18884:8889 | 18081:8081 |
| hot | 18893:8893 | 18083:8083 |
| points | - | 18084:8082 |
| security | - | 18085:8081 |

---

## 12. 可观测性

项目集成了完整的可观测性栈：

### 链路追踪（Jaeger）

- 所有服务通过 OpenTelemetry SDK 上报 Trace 到 Jaeger
- 配置项：`Telemetry.Endpoint`（默认 `localhost:34317`，OTLP gRPC）
- Jaeger UI：http://localhost:16686

### 指标监控（Prometheus + Grafana）

- Prometheus 自动抓取各组件 Exporter 的指标
- 包含 PostgreSQL、Redis、Kafka、Elasticsearch、node、cAdvisor 等 Exporter
- Prometheus UI：http://localhost:39090
- Grafana UI：http://localhost:33000（`admin / admin`）

### 日志

- 服务日志通过 go-zero 的 `logx` 写入文件（`log/` 目录下按服务分目录）
- 项目封装了 `service/common/logger`，提供结构化日志（含 TraceID、EventID、调用链路）
- task 服务的日志通过 Filebeat 发送到 Kafka，可接入 Elasticsearch + Kibana 查看

---

## 13. 测试

### 运行全部测试

```bash
go test ./...
```

### 特定服务测试

```bash
# 运行 hot 服务的 heavykeeper 测试
go test ./service/hot/heavykeeper/...

# 运行 logger 工具测试
go test ./service/common/logger/...
```

### 遗留 User RPC 测试

User RPC 的部分测试需要通过 build tag 启用，并且依赖运行中的 PostgreSQL：

```bash
# 确保 PostgreSQL 在 127.0.0.1:35432 可访问
go test -tags legacy_user_rpc_tests ./service/user/user/rpc/internal/logic/...
```

---

## 14. 各服务职责速查

| 服务 | API 定义 | RPC 定义 | 职责 |
|------|----------|----------|------|
| user | `user_center.api` | `user.proto` | 用户注册、登录、查询、更新、注销 |
| admin | `admin_center.api` | `admin.proto` | 管理员登录、用户管理、封禁/解禁 |
| article | `article.api` | `article.proto` | 文章 CRUD、图片上传、Kafka 审核与同步 |
| comment | `comment.api` | `comment.proto` | 评论发布、查询、点赞、删除、审核 |
| like | `like.api` | `like.proto` | 点赞/点踩、状态查询、点赞记录与获赞统计 |
| favorite | `favorite.api` | `favorite.proto` | 收藏夹与收藏内容管理 |
| follow | `follow.api` | `follow.proto` | 关注/取关、关注列表 |
| message | `message.api` | `message.proto` | 站内消息 |
| task | `task.api` | `task.proto` | 任务查询 |
| hot | `hot.api` | `hot.proto` | 热点内容（含 HeavyKeeper 算法） |
| points | - | `points.proto` | 积分计算（仅 RPC） |
| security | - | `security.proto` | 图片安全检测等风控能力（仅 RPC） |

---

## 15. 端口速查

### 基础设施端口

| 宿主机端口 | 组件 | 用途 |
|-----------|------|------|
| 32379 | etcd | Client API |
| 35432 | PostgreSQL | 数据库连接 |
| 36379 | Redis | 缓存连接 |
| 39092 | Kafka | PLAINTEXT 连接 |
| 39000 / 39001 | MinIO | S3 API / Console |
| 37474 / 37687 | Neo4j | Browser / Bolt |
| 19530 | Milvus | gRPC |
| 39200 | Elasticsearch | HTTP API |
| 35601 | Kibana | Web UI |
| 38080 | Kafka UI | Web |

### 可观测性端口

| 宿主机端口 | 组件 | 用途 |
|-----------|------|------|
| 39090 | Prometheus | Web UI |
| 33000 | Grafana | Web UI（`admin/admin`） |
| 16686 | Jaeger | Web UI |
| 34317 / 34318 | Jaeger | OTLP gRPC / HTTP |
| 39100 | node-exporter | 主机指标 |
| 38082 | cAdvisor | 容器指标 |

---

## 16. 常见问题

### Q: 缺少 `go.mod` 文件，无法编译？

当前仓库可能未包含 `go.mod` 文件。如果缺失，需要在项目根目录创建：

```bash
go mod init sea-try-go
```

然后根据 `go.sum` 恢复依赖或执行 `go mod tidy`。

### Q: `manage.sh` 构建失败，提示找不到 Dockerfile？

脚本中引用的是 `Dockerfile`（大写 D），而仓库中实际文件名为 `dockerfile`（小写）。在 Linux 上需要重命名：

```bash
mv dockerfile Dockerfile
```

### Q: API 层启动报错，连接 RPC 失败？

确保：
1. etcd 已启动并可访问
2. 对应的 RPC 服务已启动并注册到 etcd
3. API 配置中的 `Etcd.Hosts` 和 `Etcd.Key` 与 RPC 配置一致

### Q: 数据库连接失败？

确认 PostgreSQL 容器正在运行，且连接参数正确：
- Host: `127.0.0.1`（本地）或基础设施机器 IP
- Port: `35432`（宿主机映射端口）
- User: `admin`
- Password: `Sea-TryGo`
- DBName: `first_db`

### Q: 如何新增一个微服务？

1. 在 `proto/` 下编写 `.proto` 文件定义 RPC 接口
2. 在 `api/` 下编写 `.api` 文件定义 HTTP 接口
3. 使用 goctl 生成脚手架代码（参考[第 9 节](#9-代码生成goctl)）
4. 在生成的 `logic/` 目录中编写业务逻辑
5. 配置 `etc/` 下的 YAML 文件
6. 在 `manage.sh` 的 `ALL_SERVICES`、`resolve_build_paths`、`start_one` 中注册新服务

---

## 17. 进一步阅读

| 资源 | 链接 |
|------|------|
| go-zero 官方文档 | https://go-zero.dev/ |
| go-zero GitHub | https://github.com/zeromicro/go-zero |
| goctl 使用指南 | https://go-zero.dev/docs/tasks/cli/api-format |
| gRPC 官方文档 | https://grpc.io/docs/languages/go/ |
| 项目飞书文档 | [项目文档](https://my.feishu.cn/wiki/FWSkwcTKwiuCcGkzqwPcfx0Inuc?from=from_copylink)（密码：174w667#） |
| 前端仓库 | https://github.com/Sea-Go/Sea-RideTheWind-Fronted |
| 推荐算法仓库 | https://github.com/Sea-Go/Sea-BreakTheWaves |
| Docker 组件说明 | [`doc/docker.md`](./docker.md) |
| 代码生成命令参考 | [`help.md`](../help.md)、[`proto/readme.md`](../proto/readme.md) |
| QQ 交流群 | 750807478 |
