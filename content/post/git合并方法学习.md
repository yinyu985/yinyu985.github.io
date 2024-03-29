---
title: git合并方法学习
slug: Learn-git-merge-method
tags: [ Git,Shell ]
date: 2024-03-26T11:39:25+08:00
---

### 用字符画描述git的合并方法

```
1. 合并（Merge）
    main 分支       A---B---C
                         \
    dev 分支               D---E---F
    在 main 分支上执行 `git merge dev` 后：
    main 分支       A---B---C-----------G   <- (G 是合并提交)
                         \           /
    dev 分支               D---E---F
2. 变基（Rebase）
    main 分支       A---B---C
                         \
    dev 分支               D---E---F
    在 dev 分支上执行 `git rebase main` 后，再在 main 分支上执行 `git merge dev`：
    main 分支       A---B---C---D'---E'---F'
3. 压缩合并（Squash and Merge）
    main 分支       A---B---C
                         \
    dev 分支               D---E---F
    在 main 分支上执行 `git merge --squash dev` 后，再执行提交：
    main 分支       A---B---C-----------G'  <- (G' 是一个新的提交，合并来自 D、E 和 F 的变更)
4. 拣选提交（Cherry-pick）（不是标准合并所有更改的方法）
    main 分支       A---B---C
                         \
    dev 分支               D---E---F
    在 main 分支上对 dev 分支的每个提交执行 `git cherry-pick <提交哈希值>`：
    main 分支       A---B---C---D'---E'---F'  <- (每个提交被逐一复制)
```

### 合并方法的对比表格

| 方法            | 特点                               | 适用场景                                  | 结果                                |
| --------------- | ---------------------------------- | ----------------------------------------- | ----------------------------------- |
| Merge           | 保留所有分支的历史                 | 标准的合并操作，适用于大多数场景          | 创建一个新的合并提交                |
| Rebase          | 使历史线性化                       | 个人分支上的工作或清洁历史                | 不保留原始分支提交的顺序            |
| Squash and Merge | 将所有更改压缩为一个提交           | 当你想要简化复杂分支的历史时              | 一个新的单一提交                    |
| Cherry-pick     | 手动选择特定提交                   | 只合并某些特定的更改                       | 对选择的每个提交创建新的提交        |

请注意，这些方法中，Merge 和 Rebase 是最常用的策略来整合一个开发分支（如 dev）到主分支（如 main）。Squash and Merge 通常用于在合并之前压缩多个提交以保持清洁的历史。Cherry-pick 方法并不是一个真正的分支合并策略，它只是用于将选定的提交从一个分支复制到另一个分支，因此它不应用于将一个完整的功能分支合并到主分支的场景。
