---
title: "Golang Randder线性同余的坑"
date: 2025-12-02
tags:
  - Golang后端踩坑
categories:
  - Golang后端踩坑
---

记录 Golang `math/rand` 使用线性同余算法（LCG）带来的问题。

<!--more-->

## 问题背景

Go 的 `math/rand` 包默认使用线性同余生成器（Linear Congruential Generator），这会导致几个常见的坑。

## 坑 1：默认种子固定，每次运行结果相同

```go
package main

import (
    "fmt"
    "math/rand"
)

func main() {
    fmt.Println(rand.Intn(100)) // 每次运行都输出 81
    fmt.Println(rand.Intn(100)) // 每次运行都输出 87
}
```

**原因**：默认种子是 1，不设置种子每次程序启动生成的序列完全一样。

**解决方案**：

```go
// Go 1.20 之前
rand.Seed(time.Now().UnixNano())

// Go 1.20+ 推荐使用
rand.New(rand.NewSource(time.Now().UnixNano()))
```

## 坑 2：并发不安全

```go
// 错误示例：多个 goroutine 共享全局 rand
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(rand.Intn(100)) // 可能 panic 或产生重复值
    }()
}
```

**错误的解决方案 1**：在 goroutine 内部创建 rand

```go
for i := 0; i < 10; i++ {
    go func() {
        r := rand.New(rand.NewSource(time.Now().UnixNano()))
        fmt.Println(r.Intn(100))
    }()
}
```

这样写有问题！goroutine 启动太快，`time.Now().UnixNano()` 可能返回相同的值，导致多个 goroutine 使用相同的种子，生成相同的随机数。

**错误的解决方案 2**：循环内创建但加偏移

```go
for i := 0; i < 10; i++ {
    r := rand.New(rand.NewSource(time.Now().UnixNano() + int64(i)))
    go func(rng *rand.Rand) {
        fmt.Println(rng.Intn(100))
    }(r)
}
```

虽然加了 `i` 偏移，但循环执行太快时 `time.Now().UnixNano()` 本身可能没变化，还是会有重复种子的风险。

**正确的解决方案**：在外部创建一个 rand，传入 goroutine 使用

```go
// 方案 1：外部创建单个 rand，通过锁保护（简单场景）
var (
    rng   = rand.New(rand.NewSource(time.Now().UnixNano()))
    rngMu sync.Mutex
)

for i := 0; i < 10; i++ {
    go func() {
        rngMu.Lock()
        n := rng.Intn(100)
        rngMu.Unlock()
        fmt.Println(n)
    }()
}

// 方案 2：为每个 goroutine 预先创建独立的 rand
rngs := make([]*rand.Rand, 10)
for i := 0; i < 10; i++ {
    rngs[i] = rand.New(rand.NewSource(time.Now().UnixNano() + int64(i)*1000))
}
for i := 0; i < 10; i++ {
    go func(r *rand.Rand) {
        fmt.Println(r.Intn(100))
    }(rngs[i])
}

// 方案 3：Go 1.20+ 直接用全局 rand（已自动处理并发安全）
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(rand.Intn(100))
    }()
}
```

## 坑 3：线性同余可预测，不能用于安全场景

LCG 算法是可预测的，知道几个连续输出就能推算后续值。

**错误用法**：
```go
// 千万别这样生成 token！
token := fmt.Sprintf("%d", rand.Int63())
```

**正确做法**：安全场景使用 `crypto/rand`

```go
import "crypto/rand"

func generateToken() string {
    b := make([]byte, 32)
    crypto_rand.Read(b)
    return hex.EncodeToString(b)
}
```

## 坑 4：Go 1.20 的行为变化

Go 1.20 开始，全局 `rand` 函数自动使用随机种子，不再需要手动 `rand.Seed()`。但如果你显式调用 `rand.Seed()`，行为会回退到旧模式。

```go
// Go 1.20+ 这样写反而有问题
rand.Seed(time.Now().UnixNano()) // 不要这样做了

// 直接用就行
rand.Intn(100)
```

## 总结

| 场景 | 推荐方案 |
|-----|---------|
| 普通随机数 (Go 1.20+) | 直接用 `rand.Intn()` |
| 普通随机数 (Go 1.20 前) | `rand.Seed(time.Now().UnixNano())` |
| 并发场景 | 每个 goroutine 独立 `rand.New()` |
| 安全场景 (token/密码) | `crypto/rand` |
