---
title: "Go-Zero 微服务最佳实践架构"
date: 2025-12-06
tags:
  - Go-Zero
  - 微服务
  - 架构设计
categories:
  - 微服务
---

基于 go-zero 的微服务项目目录结构设计，参考 looklook 项目的最佳实践。

<!--more-->

## 整体架构

```
app/
├── usercenter/                         # 用户中心服务
│   ├── cmd/
│   │   ├── api/                        # HTTP API 接口
│   │   └── rpc/                        # gRPC 服务
│   └── model/                          # 数据模型 (user, userAuth)
│
├── travel/                             # 民宿/旅游服务
│   ├── cmd/
│   │   ├── api/                        # HTTP API 接口
│   │   └── rpc/                        # gRPC 服务
│   └── model/                          # 数据模型 (homestay, business, activity, comment)
│
├── order/                              # 订单服务
│   ├── cmd/
│   │   ├── api/                        # HTTP API 接口
│   │   ├── rpc/                        # gRPC 服务
│   │   └── mq/                         # 消息队列消费者
│   └── model/                          # 数据模型 (homestayOrder)
│
├── payment/                            # 支付服务
│   ├── cmd/
│   │   ├── api/                        # HTTP API 接口
│   │   └── rpc/                        # gRPC 服务
│   └── model/                          # 数据模型 (thirdPayment)
│
└── mqueue/                             # 消息队列服务
    └── cmd/
        ├── job/                        # 延迟任务处理 (如订单超时取消)
        └── scheduler/                  # 定时任务调度 (如定时统计)
```

## 服务内部分层

| 目录 | 作用 | 说明 |
|------|------|------|
| `cmd/api/` | HTTP API 接口 | 对外暴露的 RESTful 接口，供前端/客户端调用 |
| `cmd/rpc/` | gRPC 服务 | 内部服务间通信，供其他微服务调用 |
| `cmd/mq/` | 消息队列消费者 | 处理异步消息（仅 order 服务有） |
| `cmd/job/` | 延迟任务处理 | 消费延迟队列中的任务（仅 mqueue 服务） |
| `cmd/scheduler/` | 定时任务调度 | 执行周期性定时任务（仅 mqueue 服务） |
| `model/` | 数据模型层 | 数据库表映射、CRUD 操作封装 |

## 单个服务详细结构

以 usercenter 为例：

```
usercenter/
├── cmd/
│   ├── api/
│   │   ├── desc/                       # API 定义文件
│   │   │   └── usercenter.api
│   │   ├── etc/                        # 配置文件
│   │   │   └── usercenter.yaml
│   │   ├── internal/
│   │   │   ├── config/                 # 配置结构体
│   │   │   ├── handler/                # HTTP 处理器
│   │   │   ├── logic/                  # 业务逻辑
│   │   │   ├── svc/                    # 服务上下文（依赖注入）
│   │   │   └── types/                  # 请求/响应类型
│   │   └── usercenter.go               # 入口文件
│   │
│   └── rpc/
│       ├── etc/                        # 配置文件
│       │   └── usercenter.yaml
│       ├── internal/
│       │   ├── config/
│       │   ├── logic/
│       │   ├── server/                 # gRPC server 实现
│       │   └── svc/
│       ├── pb/                         # protobuf 生成的代码
│       ├── usercenter.proto            # proto 定义文件
│       └── usercenter.go               # 入口文件
│
└── model/
    ├── usermodel.go                    # user 表模型
    ├── userauthmodel.go                # user_auth 表模型
    └── vars.go                         # 公共变量
```

## 代码生成命令

### 生成 API 服务代码

进入微服务的 `api/desc` 目录执行：

```bash
goctl api go -api *.api -dir ../ -style=goZero
```

> 使用 goZero 风格（小驼峰命名）

### 生成 RPC 服务代码

进入微服务的 `rpc/pb` 目录执行：

```bash
goctl rpc protoc *.proto --go_out=../ --go-grpc_out=../ --zrpc_out=../
```

### 处理 omitempty 问题

如果需要返回零值字段（如 `0`、`""`），需要去掉 `omitempty` 标签：

```bash
# Linux
sed -i 's/,omitempty//g' *.pb.go

# macOS
sed -i "" 's/,omitempty//g' *.pb.go
```

### 生成 Model 代码

```bash
goctl model mysql datasource \
  -url="${username}:${passwd}@tcp(${host}:${port})/${dbname}" \
  -table="${tables}" \
  -dir="${modeldir}" \
  -cache=true \
  --style=goZero
```

完整脚本示例：

```bash
#!/usr/bin/env bash

# 使用方法：
# ./genModel.sh usercenter user
# ./genModel.sh usercenter user_auth
# 生成后将 ./genModel 下的文件移动到对应服务的 model 目录，记得改 package

tables=$2
modeldir=./genModel

# 数据库配置
host=127.0.0.1
port=33069
dbname=looklook_$1
username=root
passwd=your_password

echo "开始创建库：$dbname 的表：$tables"
goctl model mysql datasource \
  -url="${username}:${passwd}@tcp(${host}:${port})/${dbname}" \
  -table="${tables}" \
  -dir="${modeldir}" \
  -cache=true \
  --style=goZero
```

**生成后的需要去将当前生成的复制到对应的目录中 ， 感觉这块可以用 gorm去替代**

### 创建 Kafka Topic

```bash
kafka-topics.sh --create \
  --zookeeper zookeeper:2181 \
  --replication-factor 1 \
  --partitions 1 \
  --topic {topic}
```

## 服务间调用关系

```
┌─────────────┐
│   前端/APP   │
└──────┬──────┘
       │ HTTP
       ▼
┌──────────────────────────────────────────────────┐
│                    API 层                         │
│  usercenter-api   travel-api   order-api   ...   │
└──────────────────────────────────────────────────┘
       │ gRPC
       ▼
┌──────────────────────────────────────────────────┐
│                    RPC 层                         │
│  usercenter-rpc   travel-rpc   order-rpc   ...   │
└──────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│                   数据层                          │
│     MySQL          Redis          ES             │
└──────────────────────────────────────────────────┘
```

## 设计要点

1. **API 与 RPC 分离**：API 对外，RPC 对内，职责清晰
2. **按业务领域划分服务**：usercenter、order、payment 各自独立
3. **model 层复用**：同一服务的 api 和 rpc 共享 model
4. **异步任务独立**：mq、job、scheduler 单独部署，不影响主服务
5. **统一代码风格**：使用 `--style=goZero` 保持小驼峰命名一致性
