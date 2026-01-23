---
title: "在本地单机快速搭建 Kubernetes：k3s 从安装到彻底用 kubectl 命令"
date: 2026-01-22
tags:
  - 开发环境
  - k3s
  - kubernetes
  - kubectl
categories:
  - 开发环境
---

最近在本地机器上折腾轻量级 Kubernetes 时，发现 **k3s** 仍然是目前最省心、最省资源的单机/边缘/开发环境选择。

但很多人（包括我一开始）都会遇到一个很烦人的问题：

> 为什么每次都要打 `sudo k3s kubectl get pods` 这么长一串？有没有办法像正常 k8s 集群一样，直接敲 `kubectl` 就行？

本文就完整记录一次从 **零开始安装 k3s** 到 **彻底改成只用 kubectl** 的全过程，适用于 Ubuntu/Debian/CentOS 等主流 Linux 系统。

<!--more-->

## 1. 安装 k3s（最简单一行命令）

官方推荐的安装方式超级简洁：

```bash
curl -sfL https://get.k3s.io | sh -
```

这条命令会做这些事：

- 下载最新稳定版 k3s 二进制
- 使用 containerd 作为容器运行时（不会干扰已有的 Docker）
- 安装成 systemd 服务（开机自启）
- 自动创建一个单节点集群（server + agent 合一）
- kubeconfig 文件放在 `/etc/rancher/k3s/k3s.yaml`

安装大概 1~3 分钟，完成后会看到类似输出：

```text
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
```

### 推荐轻量版（关掉不必要的内置组件，省 200~400MB 内存）

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --disable=servicelb --disable=metrics-server" sh -
```

这样默认不装 Traefik Ingress 和 servicelb，后续需要可以自己用 Helm 装更灵活的版本。

## 2. 验证安装是否成功

```bash
# 查看服务状态
sudo systemctl status k3s

# 查看节点（最简单验证方式）
sudo k3s kubectl get nodes
```

正常输出示例：

```text
NAME   STATUS   ROLES                  AGE   VERSION
tmp    Ready    control-plane          2m    v1.34.3+k3s1
```

如果看到 `Ready`，恭喜，集群已经起来了！

## 3. 问题来了：为什么每次都要加 k3s 前缀 + sudo？

因为 k3s 自带的 `k3s kubectl` 是一个包装过的 kubectl，它默认读取了 `/etc/rancher/k3s/k3s.yaml` 这个配置文件。

而我们平时用的 `kubectl` 会默认读取 `~/.kube/config`，所以不配置的话当然连不上。

## 4. 彻底改成只用 kubectl（只需做一次）

### 步骤一：把 kubeconfig 复制到用户目录

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

### 步骤二：修改文件所有者（非常重要，避免权限问题）

```bash
sudo chown $(whoami):$(whoami) ~/.kube/config
chmod 600 ~/.kube/config
```

### 步骤三：设置环境变量（永久生效）

在 `~/.bashrc` 或 `~/.zshrc`（看你用哪个 shell）最后加一行：

```bash
export KUBECONFIG=~/.kube/config
```

然后立即生效：

```bash
source ~/.bashrc   # 或 source ~/.zshrc
```

### 可选：设置 kubectl 别名，更简洁

```bash
echo 'alias k="kubectl"' >> ~/.bashrc
source ~/.bashrc
```

## 5. 验证：现在可以直接用 kubectl 了！

```bash
kubectl get nodes
# 或
k get nodes
```

输出应该和之前一样：

```text
NAME   STATUS   ROLES                  AGE   VERSION
tmp    Ready    control-plane          15d   v1.34.3+k3s1
```

再看看所有系统 Pod：

```bash
kubectl get pods -A
```

如果大部分都是 `Running`，说明一切正常。

## 6. 小结：从此告别 k3s kubectl

改完之后，你后续的所有操作都可以这样写：

- `kubectl get pods -A`
- `kubectl apply -f my-app.yaml`
- `k create deployment nginx --image=nginx`
- Helm、k9s、Lens、VS Code Kubernetes 插件……全部无缝使用

## 7. 额外小技巧（可选）

| 操作 | 命令 |
|------|------|
| 重启 k3s | `sudo systemctl restart k3s` |
| 查看实时日志 | `sudo journalctl -u k3s -f` |
| 彻底卸载（干净） | `/usr/local/bin/k3s-uninstall.sh` |
| 想升级 | 重新跑一次安装命令即可（会自动升级） |

## 最后

用 k3s 的最大爽点就是：**轻、快、接近原生 k8s，却几乎零配置**。

把 kubeconfig 配置好后，它和云上的完整 Kubernetes 用起来几乎没有区别。

希望这篇记录能帮到正在本地折腾 k3s 的朋友。如果你也遇到启动失败、端口冲突、内存问题等，欢迎留言，我可以继续补充常见坑的填法。

祝大家本地 k8s 玩得开心～

## 参考

- [k3s 官方文档](https://docs.k3s.io/)
- [k3s GitHub](https://github.com/k3s-io/k3s)
