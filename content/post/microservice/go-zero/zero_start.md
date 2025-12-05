---
title: "Go-Zero 快速入门指南"
date: 2025-12-04
tags:
  - Go-Zero
  - 微服务框架
categories:
  - 微服务
---

Go-Zero 是一个集成了各种工程实践的 Web 和 RPC 框架，本文记录从零开始搭建 Go-Zero 开发环境的完整流程。

<!--more-->

## 1. 初始化项目

首先创建一个新的 Go 模块：

```bash
go mod init your-project-name
```

## 2. 安装 goctl 工具

goctl 是 go-zero 的代码生成工具，可以通过以下两种方式安装：

```bash
# 方式一：直接安装（推荐）
go install github.com/zeromicro/go-zero/tools/goctl@latest

# 方式二：作为依赖引入
go get -u github.com/zeromicro/go-zero/tools/goctl
```

安装完成后验证：

```bash
$ goctl --version
goctl version 1.7.3 linux/amd64
```

## 3. 安装依赖环境

goctl 提供了一键安装所需依赖的命令：

```bash
$ goctl env check --install --verbose --force
```

这个命令会检查并安装：
- protoc：Protocol Buffers 编译器
- protoc-gen-go：Go 语言的 protobuf 插件
- protoc-gen-go-grpc：gRPC 的 Go 插件

## 4. 安装 Go-Zero 框架

```bash
go get -u github.com/zeromicro/go-zero@latest
```

## 5. 常用命令

### API 服务生成

根据 `.api` 文件生成 HTTP 服务代码：

```bash
goctl api go -api user.api -dir ./user  
```

根据 `api` 文件生成 Swagger 文档

```bash
goctl api swagger -api ./api/http/ping.api -dir ./swaggerdoc
```

参数说明：
- `-api`：指定 api 定义文件
- `-dir`：指定输出目录

### RPC 服务生成

根据 `.proto` 文件生成 gRPC 服务代码：

```bash
goctl rpc protoc user.proto --go_out=./types --go-grpc_out=./types --zrpc_out=.
```

### Model 生成

根据数据库表生成 CRUD 代码：

```bash
# 从 MySQL 连接生成
goctl model mysql datasource -url="user:password@tcp(127.0.0.1:3306)/dbname" -table="user" -dir="./model"

# 从 DDL 文件生成
goctl model mysql ddl -src="user.sql" -dir="./model"
```

## 6. 项目结构示例

使用 goctl 生成的典型项目结构：

```
.
├── etc
│   └── user-api.yaml    # 配置文件
├── internal
│   ├── config
│   │   └── config.go    # 配置结构
│   ├── handler          # HTTP handlers
│   ├── logic            # 业务逻辑
│   ├── svc
│   │   └── servicecontext.go
│   └── types
│       └── types.go     # 请求/响应结构
├── user.api             # API 定义文件
└── user.go              # 入口文件
```

## 参考资料

- [Go-Zero 官方文档](https://go-zero.dev/)
- [Go-Zero GitHub](https://github.com/zeromicro/go-zero)
