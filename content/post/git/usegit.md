---
title: "Git Cherry-Pick 使用指南"
date: 2025-12-15
tags:
  - git
  - 版本控制
categories:
  - git
---

## 什么是 Cherry-Pick？

Cherry-pick 是 Git 中一个强大的功能，它允许你从一个分支中选择特定的提交（commit），并将其应用到另一个分支上。这在需要将某些特定的修改从一个分支移植到另一个分支时非常有用。

## 使用场景

最常见的使用场景是：
1. 在测试分支（test/dev）上开发并测试新功能
2. 测试通过后，将特定的提交应用到发布分支（release/main）
3. 避免合并整个分支带来的不必要的提交

## 基本用法

### 1. 查看提交历史

首先，在测试分支上查看你想要 cherry-pick 的提交：

```bash
git log --oneline
```

记录下你想要应用的提交的 hash 值（例如：`abc1234`）

### 2. 切换到目标分支

切换到你想要应用提交的分支（如 release 分支）：

```bash
git checkout release
# 或者使用新的命令
git switch release
```

### 3. 执行 Cherry-Pick

将指定的提交应用到当前分支：

```bash
git cherry-pick abc1234
```

### 4. Cherry-Pick 多个提交

如果需要应用多个连续的提交：

```bash
# 应用从 commit1 到 commit2 之间的所有提交（不包括 commit1）
git cherry-pick commit1..commit2

# 应用从 commit1 到 commit2 之间的所有提交（包括 commit1）
git cherry-pick commit1^..commit2
```

应用多个不连续的提交：

```bash
git cherry-pick abc1234 def5678 ghi9012
```

## 处理冲突

如果 cherry-pick 过程中出现冲突：

```bash
# 1. 查看冲突文件
git status

# 2. 手动解决冲突后，添加文件
git add <冲突文件>

# 3. 继续 cherry-pick
git cherry-pick --continue

# 或者放弃本次 cherry-pick
git cherry-pick --abort
```

## 常用选项

```bash
# 只应用更改，不自动提交（可以修改后再提交）
git cherry-pick -n abc1234
# 或
git cherry-pick --no-commit abc1234

# 在提交信息中添加原始提交的引用
git cherry-pick -x abc1234

# 修改提交信息
git cherry-pick -e abc1234
# 或
git cherry-pick --edit abc1234
```

## 完整工作流示例

```bash
# 1. 在测试分支开发
git checkout test
# ... 进行开发和测试 ...
git add .
git commit -m "feat: 添加新功能"

# 2. 查看提交历史，找到刚才的提交 hash
git log --oneline -5
# 假设输出：abc1234 feat: 添加新功能

# 3. 切换到 release 分支
git checkout release

# 4. 确保 release 分支是最新的
git pull origin release

# 5. 应用测试分支的提交
git cherry-pick abc1234

# 6. 推送到远程仓库
git push origin release
```

## 最佳实践

1. **提交粒度要小**：保持每个提交只做一件事，这样 cherry-pick 时更容易管理
2. **及时同步**：在 cherry-pick 之前，确保目标分支是最新的
3. **测试验证**：cherry-pick 后要重新测试，确保功能在新分支上正常工作
4. **使用 -x 选项**：在团队协作中，使用 `-x` 选项可以追踪提交来源
5. **避免重复 cherry-pick**：同一个提交不要多次 cherry-pick 到同一个分支

## 注意事项

- Cherry-pick 会创建新的提交，即使内容相同，commit hash 也会不同
- 如果后续需要合并分支，可能会遇到重复的更改
- 对于大量提交，考虑使用 `git merge` 或 `git rebase` 可能更合适

## 撤销 Cherry-Pick

如果刚刚执行的 cherry-pick 有问题：

```bash
# 撤销最后一次提交（cherry-pick 产生的）
git reset --hard HEAD~1

# 或者使用 revert（保留历史记录）
git revert HEAD
```

## 总结

Cherry-pick 是一个精确控制代码合并的工具，特别适合：
- 将 bug 修复从开发分支应用到生产分支
- 将特定功能从功能分支移植到发布分支
- 在不同版本之间移植补丁

掌握 cherry-pick 可以让你的 Git 工作流更加灵活高效！