---
title: "Harness 设计专栏（测试文章）"
date: 2026-09-30T10:00:00+08:00
tags:
  - Harness
categories:
  - Harness 设计
---

这是一篇测试文章，用于验证 Harness 设计专栏页面的展示效果。

<!--more-->

## 1. 为什么单独开专栏

专栏文章放在 `content/harness/` 下，不会出现在首页和归档中，与日常文章区分开。

## 2. 功能验证

- 文章页右侧应有目录（TOC）
- 文章底部的上一篇 / 下一篇只在专栏内部跳转

```go
func main() {
    fmt.Println("hello harness")
}
```

## 3. 小结

测试完成后可以删除本文。
