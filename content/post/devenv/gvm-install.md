---
title: "GVM 安装与使用指南"
date: 2025-12-04
tags:
  - 开发环境
  - gvm
categories:
  - 开发环境
---

GVM (Go Version Manager) 是一个管理多个 Go 版本的工具，方便在不同项目间切换 Go 版本。

<!--more-->

## 1. 清理旧版本 Go（可选）

如果系统之前使用install安装过 Go，建议先清理干净，避免版本冲突：

```bash
sudo rm -rf /usr/local/go
sudo apt-get remove golang
sudo apt-get remove golang-go
sudo apt-get autoremove
```

> 如果是全新环境，可跳过此步骤。

## 2. 安装 GVM

安装go版本之前呢 , 有些unbantu版本会需要安装前置的条件
```
sudo apt-get install gcc make 
```


执行以下命令安装 GVM：

```bash
bash < <(curl -s -S -L https://raw.githubusercontent.com/moovweb/gvm/master/binscripts/gvm-installer)
```

安装完成后，重启终端，输入 `gvm` 验证是否安装成功。

如果提示命令未找到，需要手动在 `~/.bashrc` 底部添加：

```bash
[[ -s "$HOME/.gvm/scripts/gvm" ]] && source "$HOME/.gvm/scripts/gvm"
```

然后执行 `source ~/.bashrc` 使配置生效。


## 3. 安装 Go 版本

使用 GVM 安装指定版本的 Go：

```bash
# 方式一：使用预编译二进制（推荐，速度快）
gvm install go1.18 -B

# 方式二：从源码编译（指定镜像源）
gvm install go1.18 --source=https://github.com/golang/go
```

## 4. 切换 Go 版本

```bash
# 临时切换（仅当前终端生效）
gvm use go1.18

# 设为默认版本（每次打开终端自动使用）
gvm use go1.18 --default
```

验证安装：

```bash
go version
go env
```

## 常用命令速查

| 命令 | 说明 |
|------|------|
| `gvm listall` | 查看所有可安装的版本 |
| `gvm list` | 查看已安装的版本 |
| `gvm install go1.x` | 安装指定版本 |
| `gvm use go1.x` | 切换版本 |
| `gvm uninstall go1.x` | 卸载指定版本 |

## 常见问题

- [GVM 安装后 cd 命令失效](/devenv/gvm-cd-bug/)

## 参考

- [GVM GitHub](https://github.com/moovweb/gvm)
