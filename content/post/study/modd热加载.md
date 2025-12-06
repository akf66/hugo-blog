---
title: "使用 modd 实现 Go 项目热加载"
date: 2025-12-06
tags:
  - 热加载
  - 开发工具
categories:
  - 开发环境
---

modd 是一个轻量级的文件监听工具，可以在代码变更时自动重新编译并重启服务，提升开发效率。

<!--more-->

## 安装 modd

```bash
go install github.com/cortesi/modd/cmd/modd@latest
```

## 配置文件

在项目根目录创建 `modd.conf`：

```conf
# usercenter rpc 服务
app/usercenter/cmd/rpc/**/*.go {
    prep: go build -o data/server/usercenter-rpc -v app/usercenter/cmd/rpc/usercenter.go
    daemon +sigkill: ./data/server/usercenter-rpc -f app/usercenter/cmd/rpc/etc/usercenter.yaml
}
```

## 配置解析

### 1. 文件监听路径

```
app/usercenter/cmd/rpc/**/*.go
```

`**` 表示递归匹配所有子目录，只要这些 `.go` 文件有任何改动，就触发本条规则。

### 2. prep 指令（准备阶段）

```bash
prep: go build -o data/server/usercenter-rpc -v app/usercenter/cmd/rpc/usercenter.go
```

- 把 `usercenter.go` 及其依赖编译成二进制文件
- `-o` 指定输出路径
- `-v` 显示详细编译过程，方便排错

### 3. daemon +sigkill（守护进程）

```bash
daemon +sigkill: ./data/server/usercenter-rpc -f app/usercenter/cmd/rpc/etc/usercenter.yaml
```

- 编译成功后，以守护进程方式启动二进制
- `-f` 指定配置文件路径
- `+sigkill`：文件再次变更时，先用 SIGKILL 强制杀掉旧进程，再重新编译启动

## 启动热加载

```bash
modd
```

## 工作流程

```
代码修改 → modd 检测到变更 → 执行 prep 编译 → 杀掉旧进程 → 启动新进程
```

## 多服务配置示例

```conf
# usercenter rpc
app/usercenter/cmd/rpc/**/*.go {
    prep: go build -o data/server/usercenter-rpc -v app/usercenter/cmd/rpc/usercenter.go
    daemon +sigkill: ./data/server/usercenter-rpc -f app/usercenter/cmd/rpc/etc/usercenter.yaml
}

# order rpc
app/order/cmd/rpc/**/*.go {
    prep: go build -o data/server/order-rpc -v app/order/cmd/rpc/order.go
    daemon +sigkill: ./data/server/order-rpc -f app/order/cmd/rpc/etc/order.yaml
}

# api gateway
app/gateway/cmd/api/**/*.go {
    prep: go build -o data/server/gateway-api -v app/gateway/cmd/api/gateway.go
    daemon +sigkill: ./data/server/gateway-api -f app/gateway/cmd/api/etc/gateway.yaml
}
```

这样每个服务独立监听，改哪个重启哪个，互不影响。
