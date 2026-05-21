---
title: 前端面试题-Git
date: 2026-05-21 15:00:00
tags: [前端, 面试, Git]
categories: 前端面试
---

## Git

### 如何解决冲突？

- 手动解决：打开冲突文件，按照标记（<<<<<<<、=======、>>>>>>>）手动编辑，保留需要的代码，删除标记后保存。
**使用命令**
- git rebase
- git merge
  - 区别：rebase会将当前分支的提交移到目标分支的最新提交之后，保持提交历史线性；merge会创建一个新的合并提交，保留分支的历史结构。

### git pull设置为rebase
- 命令行：git config --global pull.rebase true
- 作用：避免不必要的合并提交，保持提交历史清晰。

