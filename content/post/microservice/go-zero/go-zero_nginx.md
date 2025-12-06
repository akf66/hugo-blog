---
title: "Go-Zero 微服务网关：Nginx 统一入口实践"
date: 2025-12-06
tags:
  - Go-Zero
  - Nginx
  - 微服务
  - 网关
categories:
  - 微服务
---

理解 go-zero 中 API 与网关的关系，以及如何使用 Nginx 作为统一流量入口。

<!--more-->

## API 不是网关

很多人把 go-zero 的 API 服务理解成网关，这其实是个误区。

如果把单个 API 当网关用，会导致：
- 一个 API 对应多个 RPC
- 改任何业务都要重新构建整个 API
- 效率低，维护麻烦

正确做法是：**API 只是聚合服务**，每个业务有自己的 API + RPC：

```
用户服务：usercenter-api + usercenter-rpc
订单服务：order-api + order-rpc
支付服务：payment-api + payment-rpc
```

这样改用户服务只需更新用户相关的代码，互不影响。

## 真正的网关

那多个 API 服务怎么统一入口？这才需要真正的网关：

```
                    ┌─────────────────┐
                    │      Nginx      │  ← 真正的网关
                    │    (8888端口)    │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ usercenter-api│  │   order-api   │  │  payment-api  │
│    :1004      │  │     :1001     │  │     :1002     │
└───────────────┘  └───────────────┘  └───────────────┘
```

可选方案：Nginx、Kong、APISIX 等，本质都是统一流量入口。

## Nginx 网关配置

```nginx
server {
    listen 8081;
    access_log /var/log/nginx/looklook.com_access.log;
    error_log /var/log/nginx/looklook.com_error.log;

    # 订单服务
    location /order/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://looklook:1001;
    }

    # 支付服务
    location /payment/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://looklook:1002;
    }

    # 旅游服务
    location /travel/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://looklook:1003;
    }

    # 用户中心
    location /usercenter/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://looklook:1004;
    }
}
```

Docker 映射：外部 8888 → 容器内 8081

## 请求流程示例

访问 `http://127.0.0.1:8888/usercenter/v1/user/detail`：

```
客户端请求 :8888
    ↓
Nginx 匹配 /usercenter/
    ↓
转发到 usercenter-api :1004
    ↓
go-zero JWT 鉴权
    ↓
业务逻辑处理
```

## JWT 鉴权

在 API 定义文件中配置需要鉴权的路由：

```go
// 需要登录的接口
@server(
    prefix: usercenter/v1
    group: user
    jwt: JwtAuth
)
service usercenter {
    @doc "get user info"
    @handler detail
    post /user/detail (UserInfoReq) returns (UserInfoResp)
}
```

在 Logic 中获取用户 ID：

```go
func (l *DetailLogic) Detail(req types.UserInfoReq) (*types.UserInfoResp, error) {
    // JWT 鉴权通过后，从 ctx 中获取 userId
    userId := ctxdata.GetUidFromCtx(l.ctx)
    // ...
}
```

## 总结

| 组件 | 职责 |
|------|------|
| Nginx | 统一入口、路由分发、日志收集 |
| API | 聚合 RPC、HTTP 协议转换、JWT 鉴权 |
| RPC | 内部业务逻辑、服务间通信 |

这套架构的好处：
- 统一入口，便于日志收集和行为分析
- 各服务独立部署，互不影响
- 鉴权在 API 层统一处理，RPC 层专注业务
