---
title: "沙箱设计 01 · Firecracker"
date: 2026-10-02T20:00:00+08:00
description: "AWS 开源的 microVM：用 KVM 做硬件级隔离，又把启动时间压到百毫秒级。"
tags:
  - Harness
  - 沙箱
  - Firecracker
---

> 🚧 **占位文章**：本篇正在撰写中，内容即将更新。

Firecracker 是 AWS 为 Lambda 和 Fargate 打造的轻量虚拟机监视器（VMM），用 KVM 提供硬件级隔离，同时把设备模型精简到极致，让一台 microVM 能在百毫秒内启动。

<!--more-->

## 计划覆盖的内容

- Firecracker 的定位：为什么不是 QEMU，也不是容器
- microVM 的极简设备模型：virtio-net、virtio-block、vsock
- Jailer：VMM 进程自身的二次隔离
- 快照与恢复：冷启动之外的另一条路
- 用作 Agent 代码执行沙箱时的取舍
