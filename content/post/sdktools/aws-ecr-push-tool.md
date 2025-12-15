---
title: "Go 实现 Docker 镜像自动推送到 AWS ECR"
date: 2025-12-14
tags:
- Go
- AWS
- Docker
- DevOps
categories:
- 云原生
---

## 背景

在微服务架构中，我们经常需要将构建好的 Docker 镜像推送到容器镜像仓库。AWS ECR (Elastic Container Registry) 是 AWS 提供的托管式 Docker 镜像仓库服务。本文将介绍如何使用 Go 编写一个自动化工具，实现 Docker 镜像到 AWS ECR 的推送。

## 工具概述

这个工具主要完成以下几个步骤：

1. 通过 AWS SDK 获取 ECR 认证凭证
2. 为本地 Docker 镜像打标签
3. 登录到 ECR 仓库
4. 推送镜像到 ECR

## 核心实现

### 1. 配置结构定义

首先定义配置结构体，包含 ECR 和 EKS 相关配置：

```go
type Config struct {
    // ECR配置
    Region         string
    ImageName      string
    ImageTag       string
    RepositoryName string

    // EKS配置
    KubeconfigPath string
    Namespace      string
    DeploymentName string
    ContainerName  string
}
```

### 2. 命令行参数解析

使用 Go 标准库的 `flag` 包解析命令行参数：

```go
func main() {
    // 命令行参数
    region := flag.String("region", "us-east-1", "AWS region")
    imageName := flag.String("image", "", "Local image name")
    imageTag := flag.String("tag", fmt.Sprintf("%d", time.Now().Unix()), "Image tag (default: unix timestamp)")
    repoName := flag.String("repo", "your-org/your-service", "ECR repository name")

    flag.Parse()

    config := &Config{
        Region:         *region,
        ImageName:      *imageName,
        ImageTag:       *imageTag,
        RepositoryName: *repoName,
    }

    // 执行主流程
    if err := run(config); err != nil {
        log.Fatalf("Error: %v", err)
    }
}
```

### 3. 获取 ECR 认证凭证

使用 AWS SDK v2 获取 ECR 登录凭证和仓库 URI：

```go
func getECRAuth(ctx context.Context, region, repoName string) (string, string, error) {
    // 使用V2 SDK配置
    optFns := []func(*config.LoadOptions) error{
        config.WithRegion(region),
    }
    
    // 配置认证凭证
    creds := credentials.NewStaticCredentialsProvider(
        "YOUR_ACCESS_KEY", 
        "YOUR_SECRET_KEY", 
        "",
    )
    optFns = append(optFns, config.WithCredentialsProvider(creds))
    
    cfg, err := config.LoadDefaultConfig(ctx, optFns...)
    if err != nil {
        return "", "", fmt.Errorf("unable to load SDK config: %v", err)
    }

    // 创建ECR客户端
    ecrClient := ecr.NewFromConfig(cfg)

    // 获取登录令牌
    authOutput, err := ecrClient.GetAuthorizationToken(ctx, &ecr.GetAuthorizationTokenInput{})
    if err != nil {
        return "", "", fmt.Errorf("failed to get authorization token: %v", err)
    }

    // 解码认证令牌
    authData := authOutput.AuthorizationData[0]
    authToken := *authData.AuthorizationToken
    decodedToken, err := base64.StdEncoding.DecodeString(authToken)
    if err != nil {
        return "", "", err
    }

    // 获取仓库URI
    descOutput, err := ecrClient.DescribeRepositories(ctx, &ecr.DescribeRepositoriesInput{
        RepositoryNames: []string{repoName},
    })
    if err != nil {
        return "", "", fmt.Errorf("failed to describe repository: %v", err)
    }

    repoURI := *descOutput.Repositories[0].RepositoryUri
    return repoURI, string(decodedToken), nil
}
```

### 4. Docker 镜像操作

#### 4.1 镜像打标签

```go
func tagImage(sourceImage, targetImage string) error {
    cmd := exec.Command("docker", "tag", sourceImage, targetImage)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    return cmd.Run()
}
```

#### 4.2 登录 ECR

```go
func loginToECR(authToken string, registryURL string) error {
    parts := strings.Split(authToken, ":")
    if len(parts) != 2 {
        return fmt.Errorf("invalid auth token format")
    }

    username := parts[0]
    password := parts[1]

    cmd := exec.Command("docker", "login", 
        "--username", username, 
        "--password", password, 
        registryURL)
    cmd.Stderr = os.Stderr
    return cmd.Run()
}
```

#### 4.3 推送镜像

```go
func pushImage(imageURI string) error {
    cmd := exec.Command("docker", "push", imageURI)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    return cmd.Run()
}
```

### 5. 主流程编排

```go
func run(config *Config) error {
    ctx := context.Background()

    log.Println("Starting ECR push and EKS update process...")

    // 获取ECR登录凭证和仓库URI
    repoURI, authConfig, err := getECRAuth(ctx, config.Region, config.RepositoryName)
    if err != nil {
        return fmt.Errorf("failed to get ECR auth: %v", err)
    }

    // 打标签
    fullImageURI := fmt.Sprintf("%s:%s", repoURI, config.ImageTag)
    if err := tagImage(config.ImageName, fullImageURI); err != nil {
        return fmt.Errorf("failed to tag image: %v", err)
    }

    // 登录到ECR
    if err := loginToECR(authConfig); err != nil {
        return fmt.Errorf("failed to login to ECR: %v", err)
    }

    // 推送镜像
    if err := pushImage(fullImageURI); err != nil {
        return fmt.Errorf("failed to push image: %v", err)
    }

    log.Printf("Successfully pushed image: %s", fullImageURI)
    return nil
}
```

## 使用方式

### 编译

由于代码使用了 build tag，需要在编译时指定：

```bash
go build -tags push -o ecr-push app/auth/cmd/push.go
```

### 运行

```bash
# 基本用法
./ecr-push -image my-app:latest

# 指定完整参数
./ecr-push \
  -region us-east-1 \
  -image my-app:latest \
  -tag v1.0.0 \
  -repo your-org/your-service
```

### 参数说明

- `-region`: AWS 区域，默认 `us-east-1`
- `-image`: 本地镜像名称（必填）
- `-tag`: 镜像标签，默认使用当前 Unix 时间戳
- `-repo`: ECR 仓库名称，格式为 `组织名/服务名`

## 安全建议

⚠️ **重要提示**：代码中硬编码 AWS 凭证是非常不安全的做法。建议采用以下方式：

### 1. 使用环境变量

```go
accessKey := os.Getenv("AWS_ACCESS_KEY_ID")
secretKey := os.Getenv("AWS_SECRET_ACCESS_KEY")

creds := credentials.NewStaticCredentialsProvider(accessKey, secretKey, "")
```

### 2. 使用 IAM Role

如果在 EC2 或 ECS 上运行，直接使用默认凭证链：

```go
cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(region))
```

### 3. 使用 AWS Secrets Manager

```go
// 从 Secrets Manager 获取凭证
secretValue := getSecretFromSecretsManager("my-aws-credentials")
```

## 集成到 CI/CD

这个工具可以很方便地集成到 CI/CD 流程中：

### Jenkins Pipeline 示例

```groovy
stage('Push to ECR') {
    steps {
        script {
            sh """
                go build -tags push -o ecr-push app/auth/cmd/push.go
                ./ecr-push -image ${IMAGE_NAME}:${BUILD_NUMBER}
            """
        }
    }
}
```

### GitHub Actions 示例

```yaml
- name: Push to ECR
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  run: |
    go build -tags push -o ecr-push app/auth/cmd/push.go
    ./ecr-push -image my-app:${{ github.sha }}
```

## 优化建议

1. **并发推送**：如果需要推送多个镜像，可以使用 goroutine 并发处理
2. **重试机制**：添加网络失败重试逻辑
3. **进度显示**：集成 Docker API 显示推送进度
4. **镜像扫描**：推送后自动触发 ECR 镜像扫描
5. **通知机制**：推送成功/失败后发送通知（邮件、Slack 等）

## 总结

通过这个工具，我们实现了 Docker 镜像到 AWS ECR 的自动化推送流程。它简化了部署流程，提高了开发效率。在实际使用中，记得遵循安全最佳实践，不要在代码中硬编码敏感信息。

## 相关资源

- [AWS SDK for Go v2 文档](https://aws.github.io/aws-sdk-go-v2/)
- [AWS ECR 官方文档](https://docs.aws.amazon.com/ecr/)
- [Docker CLI 文档](https://docs.docker.com/engine/reference/commandline/cli/)
