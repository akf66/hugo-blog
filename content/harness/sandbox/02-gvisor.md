---
title: "沙箱设计 02 · gVisor"
date: 2026-10-02T20:01:00+08:00
description: "Google 的用户态内核：拦截系统调用，在应用和宿主内核之间再加一层。"
tags:
  - Harness
  - 沙箱
  - gVisor
---

> 🚧 **占位文章**：本篇正在撰写中，内容即将更新。

gVisor 是 Google 开源的应用内核，用 Go 在用户态重新实现了 Linux 系统调用接口（Sentry），把容器内的进程和宿主内核隔开，以 OCI 运行时 runsc 的形式接入容器生态。

<!--more-->

## 计划覆盖的内容

- gVisor 的定位：介于容器与虚拟机之间
- Sentry 与 Gofer：系统调用和文件访问各走哪条路
- 平台模式：systrap、KVM 与 ptrace
- 兼容性与性能开销
- 用作 Agent 代码执行沙箱时的取舍
