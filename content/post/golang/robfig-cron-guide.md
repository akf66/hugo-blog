---
title: "Go 定时任务完全指南：robfig/cron 库详解"
date: 2026-01-08
tags:
  - Golang后端踩坑
  - 定时任务
  - 任务调度
categories:
  - Golang后端踩坑
---

在后端开发中，定时任务是常见的需求：定期清理过期数据、发送定时通知、定时同步数据等。Go 的 `robfig/cron` 库提供了一个简洁而强大的解决方案。

这篇文章通过实际例子，详细讲解如何使用 cron 库来实现各种定时任务。

<!--more-->

## 基础概念

`robfig/cron` 是一个 Go 的定时任务库，模仿 Unix cron 的工作方式。它允许你用 cron 表达式来定义任务的执行时间。

### 安装

```bash
go get github.com/robfig/cron/v3
```

### 基本用法

```go
package main

import (
    "fmt"
    "github.com/robfig/cron/v3"
)

func main() {
    // 创建一个新的 cron 实例
    c := cron.New()
    
    // 添加定时任务
    c.AddFunc("0 */20 * * * *", func() {
        fmt.Println("每 20 分钟执行一次")
    })
    
    // 启动 cron
    c.Start()
    
    // 保持程序运行
    select {}
}
```

## Cron 表达式详解

Cron 表达式由 6 个字段组成（包括秒），从左到右分别是：

```
秒  分  时  日  月  周
*   *   *   *   *   *
```

### 字段说明

| 字段 | 范围 | 说明 |
|-----|------|------|
| 秒 | 0-59 | 秒数 |
| 分 | 0-59 | 分钟 |
| 时 | 0-23 | 小时（24小时制） |
| 日 | 1-31 | 月份中的日期 |
| 月 | 1-12 | 月份 |
| 周 | 0-6 | 星期（0=周日，6=周六） |

### 特殊符号

| 符号 | 说明 | 例子 |
|-----|------|------|
| `*` | 任意值 | `* * * * * *` 每秒执行一次 |
| `,` | 列表 | `0,30 * * * * *` 每分钟的 0 秒和 30 秒 |
| `-` | 范围 | `0 9-17 * * * *` 9 点到 17 点每分钟 |
| `/` | 步长 | `0 */20 * * * *` 每 20 分钟 |
| `?` | 不指定 | 用于日期或周 |

## 常见表达式示例

### 按时间间隔

```go
// 每秒执行一次
c.AddFunc("* * * * * *", func() {})

// 每 5 秒执行一次
c.AddFunc("*/5 * * * * *", func() {})

// 每分钟执行一次
c.AddFunc("0 * * * * *", func() {})

// 每 20 分钟执行一次（你的例子）
c.AddFunc("0 */20 * * * *", func() {})

// 每小时执行一次
c.AddFunc("0 0 * * * *", func() {})

// 每天执行一次
c.AddFunc("0 0 0 * * *", func() {})
```

### 按具体时间

```go
// 每天早上 8 点执行
c.AddFunc("0 0 8 * * *", func() {})

// 每周一早上 9 点执行
c.AddFunc("0 0 9 * * 1", func() {})

// 每月 1 号执行
c.AddFunc("0 0 0 1 * *", func() {})

// 每年 1 月 1 号执行
c.AddFunc("0 0 0 1 1 *", func() {})
```

### 复杂表达式

```go
// 工作日（周一到周五）每天 9 点
c.AddFunc("0 0 9 * * 1-5", func() {})

// 每个月的 1 号和 15 号
c.AddFunc("0 0 0 1,15 * *", func() {})

// 每小时的 0、15、30、45 分钟
c.AddFunc("0 0,15,30,45 * * * *", func() {})
```

## 实战例子

### 例子 1：定期清理过期数据

```go
package main

import (
    "fmt"
    "log"
    "time"
    "github.com/robfig/cron/v3"
)

func main() {
    c := cron.New()
    
    // 每天凌晨 2 点清理过期数据
    entryID, err := c.AddFunc("0 0 2 * * *", func() {
        log.Println("开始清理过期数据...")
        cleanExpiredData()
        log.Println("清理完成")
    })
    
    if err != nil {
        log.Fatal(err)
    }
    
    log.Printf("任务已添加，ID: %d\n", entryID)
    
    c.Start()
    select {}
}

func cleanExpiredData() {
    // 实现清理逻辑
    time.Sleep(1 * time.Second)
    fmt.Println("已清理过期数据")
}
```

### 例子 2：定时发送通知

```go
package main

import (
    "fmt"
    "log"
    "github.com/robfig/cron/v3"
)

func main() {
    c := cron.New()
    
    // 工作日每天 9 点发送早报
    c.AddFunc("0 0 9 * * 1-5", func() {
        sendDailyReport()
    })
    
    // 每周五下午 5 点发送周报
    c.AddFunc("0 0 17 * * 5", func() {
        sendWeeklyReport()
    })
    
    c.Start()
    select {}
}

func sendDailyReport() {
    fmt.Println("发送早报...")
}

func sendWeeklyReport() {
    fmt.Println("发送周报...")
}
```

### 例子 3：定时同步数据

```go
package main

import (
    "fmt"
    "log"
    "sync"
    "github.com/robfig/cron/v3"
)

type DataSyncer struct {
    cron *cron.Cron
    mu   sync.Mutex
}

func NewDataSyncer() *DataSyncer {
    return &DataSyncer{
        cron: cron.New(),
    }
}

func (ds *DataSyncer) Start() {
    // 每 20 分钟同步一次数据
    ds.cron.AddFunc("0 */20 * * * *", func() {
        ds.syncData()
    })
    
    ds.cron.Start()
    log.Println("数据同步器已启动")
}

func (ds *DataSyncer) syncData() {
    ds.mu.Lock()
    defer ds.mu.Unlock()
    
    fmt.Println("开始同步数据...")
    // 实现同步逻辑
    fmt.Println("数据同步完成")
}

func (ds *DataSyncer) Stop() {
    ds.cron.Stop()
    log.Println("数据同步器已停止")
}

func main() {
    syncer := NewDataSyncer()
    syncer.Start()
    
    select {}
}
```

## 高级用法

### 1. 获取任务 ID 并移除任务

```go
c := cron.New()

// AddFunc 返回任务 ID
entryID, err := c.AddFunc("0 */20 * * * *", func() {
    fmt.Println("执行任务")
})

if err != nil {
    log.Fatal(err)
}

c.Start()

// 后续可以移除这个任务
c.Remove(entryID)
```

### 2. 添加带名称的任务

```go
type Job struct {
    Name string
}

func (j Job) Run() {
    fmt.Printf("执行任务: %s\n", j.Name)
}

c := cron.New()

job := Job{Name: "数据同步"}
c.AddJob("0 */20 * * * *", job)

c.Start()
```

### 3. 获取所有任务

```go
c := cron.New()

c.AddFunc("0 */20 * * * *", func() {
    fmt.Println("任务 1")
})

c.AddFunc("0 0 8 * * *", func() {
    fmt.Println("任务 2")
})

c.Start()

// 获取所有任务
entries := c.Entries()
for _, entry := range entries {
    fmt.Printf("任务 ID: %d, 下次执行: %v\n", entry.ID, entry.Next)
}
```

### 4. 设置时区

```go
// 默认使用本地时区
c := cron.New()

// 指定时区
loc, err := time.LoadLocation("Asia/Shanghai")
if err != nil {
    log.Fatal(err)
}

c = cron.New(cron.WithLocation(loc))

// 每天上海时间 8 点执行
c.AddFunc("0 0 8 * * *", func() {
    fmt.Println("上海时间 8 点")
})

c.Start()
```

### 5. 错误处理

```go
// 创建自定义日志记录器
logger := cron.VerbosePrintfLogger(log.New(os.Stdout, "cron: ", log.LstdFlags))

c := cron.New(cron.WithLogger(logger))

// 添加任务时捕获错误
_, err := c.AddFunc("invalid expression", func() {})
if err != nil {
    log.Printf("添加任务失败: %v\n", err)
}

c.Start()
```

## 常见问题

### Q1: 任务执行时间过长会怎样？

Cron 会等待任务完成后再执行下一个任务。如果任务执行时间超过了下一个计划执行时间，下一个任务会立即执行。

```go
c := cron.New()

// 这个任务每 5 秒执行一次，但执行需要 10 秒
c.AddFunc("*/5 * * * * *", func() {
    fmt.Println("开始执行")
    time.Sleep(10 * time.Second)
    fmt.Println("执行完成")
})

c.Start()
```

### Q2: 如何在任务中捕获错误？

```go
c := cron.New()

c.AddFunc("0 */20 * * * *", func() {
    if err := doSomething(); err != nil {
        log.Printf("任务执行失败: %v\n", err)
        // 可以选择重试或发送告警
    }
})

c.Start()
```

### Q3: 如何优雅地关闭 cron？

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "github.com/robfig/cron/v3"
)

func main() {
    c := cron.New()

    c.AddFunc("0 */20 * * * *", func() {
        fmt.Println("执行任务")
    })

    c.Start()

    // 监听关闭信号
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    <-sigChan
    fmt.Println("收到关闭信号")

    // 停止 cron（会等待正在执行的任务完成）
    ctx := c.Stop()
    <-ctx.Done()
    fmt.Println("Cron 已停止")
}
```

### Q4: 如何测试 cron 任务？

```go
func TestCronTask(t *testing.T) {
    c := cron.New()
    
    executed := false
    
    // 使用 @every 语法进行快速测试
    c.AddFunc("@every 1s", func() {
        executed = true
    })
    
    c.Start()
    defer c.Stop()
    
    time.Sleep(2 * time.Second)
    
    if !executed {
        t.Error("任务未执行")
    }
}
```

## 特殊表达式

Cron 还支持一些特殊的预定义表达式：

| 表达式 | 说明 |
|--------|------|
| `@yearly` | 每年 1 月 1 号 0 点 |
| `@annually` | 同 @yearly |
| `@monthly` | 每月 1 号 0 点 |
| `@weekly` | 每周日 0 点 |
| `@daily` | 每天 0 点 |
| `@midnight` | 同 @daily |
| `@hourly` | 每小时 0 分 |
| `@every 1h30m` | 每 1 小时 30 分钟 |

```go
c := cron.New()

// 使用特殊表达式
c.AddFunc("@daily", func() {
    fmt.Println("每天执行")
})

c.AddFunc("@every 30m", func() {
    fmt.Println("每 30 分钟执行")
})

c.Start()
```

## 性能建议

1. **避免长时间任务**：如果任务执行时间长，考虑在 goroutine 中执行
   ```go
   c.AddFunc("0 */20 * * * *", func() {
       go longRunningTask()
   })
   ```

2. **使用互斥锁防止并发**：如果任务可能重叠执行
   ```go
   var mu sync.Mutex
   
   c.AddFunc("0 */20 * * * *", func() {
       if !mu.TryLock() {
           return // 上一个任务还在执行
       }
       defer mu.Unlock()
       
       doTask()
   })
   ```

3. **监控任务执行**：记录任务执行时间和错误
   ```go
   c.AddFunc("0 */20 * * * *", func() {
       start := time.Now()
       defer func() {
           duration := time.Since(start)
           log.Printf("任务执行耗时: %v\n", duration)
       }()
       
       doTask()
   })
   ```

## 总结

`robfig/cron` 是一个简洁而强大的定时任务库，适合各种场景：

✅ 简单易用的 API  
✅ 灵活的 cron 表达式  
✅ 支持任务管理（添加、移除、查询）  
✅ 支持时区设置  
✅ 轻量级，无外部依赖  

无论是定期清理数据、发送通知，还是同步数据，cron 都能胜任。关键是理解 cron 表达式的语法，然后就可以轻松实现各种定时任务了。
