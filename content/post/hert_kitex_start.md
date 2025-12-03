---
title: "Hertz + Kitex 微服务项目搭建指南"
date: 2025-12-03

tags:
  - Hertz
  - Kitex
  - 微服务
categories:
  - 微服务框架
---

使用 Hertz 作为 HTTP 网关 + Kitex 作为 RPC 微服务的项目搭建流程。

<!--more-->

## 项目结构

```
.
├── go.mod
├── go.sum
└── server
    ├── cmd
    │   ├── api          # Hertz HTTP 网关
    │   └── user         # Kitex RPC 服务
    ├── idl
    │   ├── base
    │   ├── http         # Hertz 用的 thrift
    │   └── rpc          # Kitex 用的 thrift
    └── shared
        ├── consts
        └── kitex_gen    # 共享的 model 库
```

## 环境准备

```bash
# 安装 Hertz 脚手架
go install github.com/cloudwego/hertz/cmd/hz@latest

# 安装 Kitex 脚手架
go install github.com/cloudwego/kitex/tool/cmd/kitex@latest

# 高版本 Go 需要降级 thrift（解决脚手架生成问题）
go get github.com/apache/thrift@v0.13.0
```

## Step 1：生成共享 Model 库

在 `shared` 目录下生成 Kitex model：

```bash
cd server/shared
kitex -module haha ./../../idl/rpc/user.thrift
```

生成后会在 `kitex_gen` 目录下产生对应的 model 文件。

## Step 2：创建 Kitex RPC 服务

进入 `server/cmd/user` 目录，指定使用共享的 model 库：

```bash
cd server/cmd/user
kitex -service user \
      -module haha \
      -use haha/server/shared/kitex_gen \
      ./../../idl/rpc/user.thrift
```

生成结构：

```
user/
├── build.sh
├── handler.go       # 在这里编写业务逻辑
├── kitex_info.yaml
├── main.go
└── script/
    └── bootstrap.sh
```

> **注意**：`-service` 是增量覆盖模式，不会覆盖你修改过的代码。

## Step 3：创建 Hertz HTTP 网关

进入 `server/cmd/api` 目录：

```bash
cd server/cmd/api
hz new -idl ./../../idl/http/user.thrift -module haha/server/cmd/api
```

生成结构：

```
api/
├── biz/
│   ├── handler/
│   │   ├── ping.go
│   │   └── user/
│   │       └── user_service.go   # 业务逻辑
│   ├── model/
│   │   ├── base/
│   │   └── user/
│   └── router/
│       ├── register.go
│       └── user/
│           ├── middleware.go
│           └── user.go
├── main.go
├── router.go
├── router_gen.go
└── ...
```

在 `handler` 和 `router` 中编写业务逻辑，然后在当前目录执行服务即可。
