---
title: "解决 GVM 安装后 cd 命令失效问题"
date: 2025-12-04
tags:
  - Golang后端踩坑
  - gvm
categories:
  - 开发环境
  - Golang后端踩坑
---

在 Ubuntu 上安装 GVM 后，可能会遇到 `cd` 命令失效的问题。本文记录问题原因及解决方案。

<!--more-->

## 问题现象

安装 GVM 后，终端中 `cd` 命令无法正常工作。

## 问题分析

安装 GVM 后，`.bashrc` 末尾会添加：

```bash
[[ -s "$HOME/.gvm/scripts/gvm" ]] && source "$HOME/.gvm/scripts/gvm"
```

这个脚本会加载 `gvm-default`：

```bash
# ~/.gvm/scripts/gvm
export GVM_ROOT=$HOME/.gvm
. $GVM_ROOT/scripts/gvm-default
```

问题出在 `gvm-default` 脚本的最后一行：

```bash
. "$GVM_ROOT/scripts/env/cd" && cd .
```

GVM 重写了 `cd` 命令（用于自动切换 Go 版本），但这会导致系统默认的 `cd` 命令被覆盖失效。

## 解决方案

在 `~/.gvm/scripts/gvm-default` 末尾添加 `unset cd`，取消 GVM 对 `cd` 的覆盖：

```bash
echo 'unset cd' >> ~/.gvm/scripts/gvm-default
```

添加后文件末尾应该是：

```bash
. "$GVM_ROOT/scripts/env/cd" && cd .
unset cd
```

重新加载配置：

```bash
source ~/.bashrc
```

验证修复：

```bash
cd /tmp && pwd
# 输出 /tmp 说明修复成功
```

## 总结

GVM 的 `cd` 覆盖功能本意是在切换目录时自动检测 `.go-version` 文件并切换 Go 版本，但实现上有 bug。通过 `unset cd` 可以恢复默认行为，同时不影响 GVM 的其他功能。
