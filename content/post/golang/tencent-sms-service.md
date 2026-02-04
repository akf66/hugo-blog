---
title: "Go 腾讯云短信服务封装：验证码发送完整指南"
date: 2026-02-03
tags:
  - Golang后端踩坑
  - 腾讯云
  - 短信服务
  - 验证码
categories:
  - Golang后端踩坑
---

在用户认证场景中，短信验证码是常见的身份验证方式。本文介绍如何基于腾讯云 SMS SDK 封装一个简洁易用的短信发送服务，支持多种业务场景的验证码发送。

<!--more-->

## 概述

这个短信服务封装了腾讯云 SMS SDK，主要用于发送验证码短信，支持以下场景：

- 用户注册
- 用户登录
- 重置密码
- 绑定/修改手机号
- 账号注销

## 依赖安装

```bash
go get github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/sms/v20210111
```

## YAML 配置

在配置文件中添加腾讯云短信相关配置：

```yaml
TencentSMS:
  SecretId: "your-secret-id"           # 腾讯云 API 密钥 ID
  SecretKey: "your-secret-key"         # 腾讯云 API 密钥 Key
  AppID: "1400xxxxxx"                  # 短信应用 ID
  SignName: "你的签名"                  # 短信签名 (需腾讯云审核通过)
  TemplateIdBind: "xxxxxxx"            # 绑定手机模板 ID
  TemplateIdRegister: "xxxxxxx"        # 注册验证码模板 ID
  TemplateIdLogin: "xxxxxxx"           # 登录验证码模板 ID
  TemplateIdResetPassword: "xxxxxxx"   # 重置密码模板 ID
  TemplateIdModifyPhone: "xxxxxxx"     # 修改手机号模板 ID
  TemplateIdDeletion: "xxxxxxx"        # 账号注销模板 ID
```

### 配置项说明

| 配置项 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| SecretId | string | 是 | 腾讯云 API 密钥 ID |
| SecretKey | string | 是 | 腾讯云 API 密钥 Key |
| AppID | string | 是 | 短信应用 ID |
| SignName | string | 是 | 短信签名内容，需审核通过 |
| TemplateId* | string | 否 | 各场景的模板 ID |

### 获取配置值

1. **SecretId / SecretKey**: 腾讯云控制台 -> 访问管理 -> API密钥管理
2. **AppID**: 腾讯云控制台 -> 短信 -> 应用管理
3. **SignName**: 腾讯云控制台 -> 短信 -> 签名管理
4. **TemplateId**: 腾讯云控制台 -> 短信 -> 正文模板管理

## 服务实现

### 数据结构

```go
type Service struct {
    appId    *string      // 短信应用 ID
    signName *string      // 短信签名
    client   *sms.Client  // 腾讯云 SMS 客户端
}
```

### 创建服务实例

```go
func NewService(client *sms.Client, appId string, signName string) *Service {
    return &Service{
        appId:    &appId,
        signName: &signName,
        client:   client,
    }
}
```

### 发送单条短信

```go
func (s *Service) SendOne(ctx context.Context, tplId string, args []string, numbers string) error {
    return s.Send(ctx, tplId, args, numbers)
}
```

### 批量发送短信

```go
func (s *Service) Send(ctx context.Context, tplId string, args []string, numbers ...string) error {
    request := sms.NewSendSmsRequest()
    request.SmsSdkAppId = s.appId
    request.SignName = s.signName
    request.TemplateId = &tplId
    request.TemplateParamSet = common.StringPtrs(args)
    request.PhoneNumberSet = common.StringPtrs(numbers)

    response, err := s.client.SendSms(request)
    if err != nil {
        return err
    }

    // 检查发送状态
    for _, status := range response.Response.SendStatusSet {
        if *status.Code != "Ok" {
            return fmt.Errorf("发送短信失败 %s, %s", *status.Code, *status.Message)
        }
    }
    return nil
}
```

### 生成验证码

```go
func (s *Service) GenerateCode() string {
    code := rand.Intn(1000000)
    return fmt.Sprintf("%06d", code)
}
```

### 缓存 Key 生成

```go
// 带用户ID的缓存Key
func (s *Service) KeyWithUserId(biz, userId, phone string) string {
    return fmt.Sprintf("phone_code:%s:%s:%s", biz, userId, phone)
}

// 不带用户ID的缓存Key
func (s *Service) KeyWithoutUserId(biz, phone string) string {
    return fmt.Sprintf("phone_code:%s:%s", biz, phone)
}
```

## 完整使用示例

```go
package main

import (
    "context"
    "log"
    "time"

    "github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/common"
    "github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/common/profile"
    sms "github.com/tencentcloud/tencentcloud-sdk-go/tencentcloud/sms/v20210111"
)

func main() {
    // 1. 初始化腾讯云客户端
    credential := common.NewCredential("your-secret-id", "your-secret-key")
    cpf := profile.NewClientProfile()
    client, err := sms.NewClient(credential, "ap-guangzhou", cpf)
    if err != nil {
        log.Fatal(err)
    }

    // 2. 创建短信服务
    smsService := NewService(client, "1400xxxxxx", "你的签名")

    // 3. 生成验证码
    code := smsService.GenerateCode()

    // 4. 生成缓存 Key 并存储到 Redis
    phone := "13800138000"
    key := smsService.KeyWithoutUserId("login", phone)
    // redis.Set(key, code, 5*time.Minute)

    // 5. 发送短信
    ctx := context.Background()
    err = smsService.SendOne(ctx, "模板ID", []string{code, "5"}, "+86"+phone)
    if err != nil {
        log.Printf("发送失败: %v", err)
        return
    }

    log.Println("验证码发送成功")
}
```

## 业务场景示例

### 登录验证码

```go
func SendLoginCode(ctx context.Context, svc *Service, phone string) error {
    code := svc.GenerateCode()
    key := svc.KeyWithoutUserId("login", phone)
    
    // 存储到 Redis，5分钟过期
    if err := redis.Set(ctx, key, code, 5*time.Minute).Err(); err != nil {
        return err
    }
    
    // 发送短信，模板参数: {1}验证码 {2}有效时间
    return svc.SendOne(ctx, "登录模板ID", []string{code, "5"}, "+86"+phone)
}
```

### 注册验证码

```go
func SendRegisterCode(ctx context.Context, svc *Service, phone string) error {
    code := svc.GenerateCode()
    key := svc.KeyWithoutUserId("register", phone)
    
    if err := redis.Set(ctx, key, code, 5*time.Minute).Err(); err != nil {
        return err
    }
    
    return svc.SendOne(ctx, "注册模板ID", []string{code, "5"}, "+86"+phone)
}
```

### 验证码校验

```go
func VerifyCode(ctx context.Context, biz, phone, inputCode string) bool {
    key := fmt.Sprintf("phone_code:%s:%s", biz, phone)
    
    storedCode, err := redis.Get(ctx, key).Result()
    if err != nil {
        return false
    }
    
    if storedCode == inputCode {
        // 验证成功后删除验证码
        redis.Del(ctx, key)
        return true
    }
    
    return false
}
```

## 错误处理

发送失败时返回格式化错误信息：

```
发送短信失败 {错误码}, {错误信息}
```

常见错误码：

| 错误码 | 说明 |
|--------|------|
| LimitExceeded.PhoneNumberDailyLimit | 单个手机号日发送超限 |
| LimitExceeded.PhoneNumberThirtySecondLimit | 30秒内发送超限 |
| InvalidParameterValue.TemplateParameterFormatError | 模板参数格式错误 |
| UnauthorizedOperation.SdkAppIdIsDisabled | 应用已被禁用 |

## 安全建议

1. **频率限制**: 对同一手机号设置发送频率限制（如60秒一次）
2. **验证码有效期**: 设置合理的过期时间（建议5分钟）
3. **验证码长度**: 使用6位数字验证码
4. **一次性使用**: 验证成功后立即删除验证码
5. **错误次数限制**: 限制验证码错误尝试次数

## 总结

基于腾讯云 SMS SDK 封装的短信服务具有以下特点：

- 简洁的 API 设计
- 支持单发和批量发送
- 内置验证码生成
- 灵活的缓存 Key 生成
- 完善的错误处理

通过合理的封装，可以快速在项目中集成短信验证码功能，满足用户注册、登录、密码重置等常见业务场景。
