---
title: "微服务日志采集链路：从文件到 Kibana"
date: 2025-12-05
tags:
  - 日志采集
  - 微服务
categories:
  - 微服务
---

理解微服务架构中日志从产生到可搜索的完整流转过程。

<!--more-->

## 整体链路

```
微服务 → 本地日志文件 → Filebeat → Kafka → go-stash → Elasticsearch → Kibana
```

## 各环节职责

### 1. 微服务写日志

各个微服务将日志写入本地文件：

```
/var/log/looklook/user.log
/var/log/looklook/order.log
/var/log/looklook/payment.log
...
```

为什么不直接发到 ES？
- 解耦：服务只管写文件，不关心日志去哪
- 可靠：即使下游挂了，日志也不丢

### 2. Filebeat 采集

Filebeat 是轻量级的日志采集器，守着日志文件：

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/looklook/*.log

output.kafka:
  hosts: ["kafka:9092"]
  topic: "looklook-log"
```

工作方式：
- 监听文件变化，有新行立即读取
- 记录读取位置（offset），重启不重复
- 批量发送，减少网络开销

### 3. Kafka 缓冲

Kafka 作为中间缓冲层：

```
Producer(Filebeat) → Topic(looklook-log) → Consumer(go-stash)
```

为什么加这层？
- 削峰：日志量突增时不会压垮 ES
- 解耦：采集和处理可以独立扩展
- 可靠：消息持久化，消费失败可重试

### 4. go-stash 处理

go-stash（或 Logstash）从 Kafka 拉取日志，做清洗转换：

```yaml
# 典型处理
Input:  {"time":"2025-12-06","level":"info","msg":"user login","user_id":123}

处理:
  - 解析 JSON 字段
  - 时间格式转换
  - 过滤无用字段
  - 添加索引名

Output: 写入 ES 的 looklook-log-2025.12.06 索引
```

### 5. Elasticsearch 存储

ES 负责存储和索引，支持全文检索：

```
索引: looklook-log-2025.12.06
文档: { time, level, msg, user_id, service, ... }
```

### 6. Kibana 查询

最终在 Kibana 上：
- 搜索：`level:error AND service:order`
- 可视化：错误趋势图、服务调用量
- 告警：错误数超阈值通知

## 一图总结

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌──────────┐    ┌────┐    ┌────────┐
│ 微服务  │───▶│ 日志文件 │───▶│Filebeat │───▶│  Kafka   │───▶│Stash│───▶│   ES   │
└─────────┘    └─────────┘    └─────────┘    └──────────┘    └────┘    └────────┘
   写入           存储           采集           缓冲          处理       存储/索引
                                                                           │
                                                                           ▼
                                                                      ┌────────┐
                                                                      │ Kibana │
                                                                      └────────┘
                                                                        查询
```

这套架构的核心思想：**每个组件只做一件事，通过消息队列解耦**，任何一环出问题都不会导致日志丢失。
