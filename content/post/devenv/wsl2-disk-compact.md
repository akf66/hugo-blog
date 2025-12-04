---
title: "WSL2 磁盘空间释放：使用 DiskPart 压缩虚拟磁盘"
date: 2025-12-03
tags:
  - WSL
categories:
  - 开发环境
---

WSL2 删除文件后磁盘空间不释放？用 DiskPart 压缩虚拟磁盘解决。

<!--more-->

## 问题

WSL2 中删除文件或清理 Docker 镜像后，虚拟磁盘文件（ext4.vhdx）大小不会自动减少，导致宿主机磁盘空间被持续占用。

## 解决步骤

### 1. 关闭 WSL2

```powershell
wsl --shutdown
```

### 2. 打开 DiskPart

按 `Win + R`，输入 `diskpart`，以管理员身份运行。

![打开DiskPart](https://www.gscblog.com/upload/02-dispart02.png)

### 3. 找到虚拟磁盘文件

虚拟磁盘文件路径一般在：

```
C:\Users\你的用户名\AppData\Local\Packages\
```

在该目录下搜索 `ext4.vhdx` 文件，记下完整路径。

### 4. 选择并压缩磁盘

在 DiskPart 中执行：

```
select vdisk file="C:\Users\你的用户名\AppData\Local\Packages\...\ext4.vhdx"
compact vdisk
```

等待进度到 100% 完成，关闭窗口即可。

![压缩完成](https://www.gscblog.com/upload/02-dispart05.png)
