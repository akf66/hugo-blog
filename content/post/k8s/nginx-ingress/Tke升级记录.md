---
title: "TKE 集群 Nginx Ingress 升级部署全记录：从远程连接到 Auth 鉴权配置"
date: 2026-02-24
tags:
  - Kubernetes
  - TKE
  - Nginx Ingress
  - Helm
  - 鉴权
categories:
  - Kubernetes
---

记录一次在腾讯云 TKE 集群上，通过 Helm 升级部署 Nginx Ingress Controller 的完整过程，包括远程连接集群、Helm Values 配置要点，以及利用 `auth_request` 实现统一鉴权的 Ingress 配置方案。

<!--more-->

## 1. 远程连接 TKE 集群

首先需要在 TKE 控制台开启公网访问，下载集群的 kubeconfig 凭证文件，然后在本地配置：

```bash
# 将 TKE 凭证加入 KUBECONFIG
export KUBECONFIG=$KUBECONFIG:/home/akf/下载/tke/cls-g9mon6gq-config

# 查看可用的 context
kubectl config --kubeconfig=/home/akf/下载/tke/cls-g9mon6gq-config get-contexts

# 切换到目标集群 context
kubectl config --kubeconfig=/home/akf/下载/tke/cls-g9mon6gq-config use-context cls-g9mon6gq-100045431505-context-default
```

连接成功后，就可以在本地通过 `kubectl` 操作远程 TKE 集群了。

## 2. 通过 Helm 部署 Nginx Ingress Controller

### 2.1 添加 Helm 仓库

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 查看已添加的仓库
helm repo list

# 搜索可用的 chart 版本
helm search repo ingress-nginx
```

### 2.2 安装/升级 Nginx Ingress

```bash
helm upgrade --install prod-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.ingressClass=ingress-new \
  --set controller.ingressClassResource.name=ingress-new
```

### 2.3 Values 配置详解

实际部署时建议使用 values 文件进行管理，以下是一份经过验证的配置：

```yaml
controller:
  # 允许使用 snippet 注解（auth_request 等高级配置必须开启）
  allowSnippetAnnotations: true
  config:
    allow-snippet-annotations: "true"
    use-forwarded-headers: "true"
    # 将注解风险等级调整为 Critical，否则 snippet 注解会被拦截
    annotations-risk-level: "Critical"

  admissionWebhooks:
    patch:
      image:
        digest: ""
        image: k8smirror/ingress-nginx-kube-webhook-certgen
        registry: docker.io
        tag: v1.6.5

  image:
    digest: ""
    image: k8smirror/ingress-nginx-controller
    registry: docker.io
    tag: v1.14.1

  ingressClass: nginx-creaibo
  ingressClassResource:
    name: nginx-creaibo

  opentelemetry:
    enabled: false

defaultBackend:
  enabled: true
  image:
    digest: ""
    image: k8smirror/defaultbackend-amd64
    registry: docker.io
    tag: "1.5"
```

> **注意：** 如果需要使用 `configuration-snippet` 或 `server-snippet` 注解，必须同时开启 `allowSnippetAnnotations: true` 并将 `annotations-risk-level` 设置为 `"Critical"`，否则 Ingress 资源会被 admission webhook 拒绝。

## 3. Ingress 统一鉴权配置（auth_request 方案）

在微服务架构中，通常希望在网关层统一处理鉴权，而不是每个服务各自实现。Nginx Ingress 的 `auth_request` 机制可以很好地解决这个问题。

### 3.1 配置示例

以下是一个完整的 Ingress annotations 配置，通过 `auth_request` 将请求先转发到鉴权服务验证，验证通过后再将用户信息注入到上游请求头中：

```yaml
annotations:
  description: messagent-ingress
  kubernetes.io/ingress.rule-mix: "true"
  nginx.ingress.kubernetes.io/configuration-snippet: |
    auth_request /auth-internal;
    auth_request_set $user_id $upstream_http_x_user_id;
    auth_request_set $token_key $upstream_http_x_token_key;
    proxy_set_header X-User-Id $user_id;
    proxy_set_header X-Token-Key $token_key;
  nginx.ingress.kubernetes.io/server-snippet: |
    location = /auth-internal {
      internal;
      proxy_pass http://new-api.messagent.svc.cluster.local:3000/api/auth/verify;
      proxy_pass_request_body off;
      proxy_set_header Content-Length "";
      proxy_set_header X-Original-IP $realip_remote_addr;
      proxy_set_header X-Original-URI $request_uri;
      proxy_set_header X-Original-Host $host;
      proxy_set_header X-Original-Method $request_method;
      proxy_set_header Host $host;
      proxy_set_header Cookie $http_cookie;
      proxy_intercept_errors on;
    }
    error_page 401 = @auth_error;
    location @auth_error {
      return 401 'Unauthorized';
    }
```

### 3.2 工作流程

1. 客户端请求到达 Nginx Ingress
2. `configuration-snippet` 中的 `auth_request /auth-internal` 触发子请求
3. Nginx 将子请求转发到 `server-snippet` 中定义的 `/auth-internal` location
4. `/auth-internal` 将请求代理到鉴权服务 `new-api.messagent.svc.cluster.local:3000/api/auth/verify`
5. 鉴权服务返回 200 则放行，并通过响应头传回 `X-User-Id` 和 `X-Token-Key`
6. `auth_request_set` 提取这些值，`proxy_set_header` 注入到上游服务的请求头中
7. 鉴权服务返回 401 则触发 `@auth_error`，直接返回 `401 Unauthorized`

### 3.3 多服务场景下的关键注意事项

当集群中存在多个服务共用同一个 Ingress 时，鉴权服务需要知道原始请求的完整信息才能正确判断权限。以下两个 header 必须传入：

```nginx
proxy_set_header X-Original-URI $request_uri;
proxy_set_header X-Original-Host $host;
```

如果缺少这两个 header，鉴权服务将无法区分请求来自哪个域名、访问的是哪个路径，导致鉴权逻辑无法正确执行或请求转发异常。
