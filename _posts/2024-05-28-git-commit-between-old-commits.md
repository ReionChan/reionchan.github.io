---
layout: post
title: Git 在历史提交记录中插入新提交
categories: Tools
excerpt: Git 在历史提交记录中插入新提交
image: https://git-scm.com/images/logo@2x.png
description: Git CLI commit between old
keywords: git GIT CLI 版本控制
licences: cc
---

![git](https://git-scm.com/images/logo@2x.png)

### 需求

```sh
# 原始的 Commit Log
AA-BB-CC-DD		(master)

# 预想的变更后 Commit Log
AA-BB-VV-CC-DD	(master)
```

### 操作

```sh
$ git checkout master
# 检出以插入点的上一次提交，即 BB 的代码创建临时分支 tmp
$ git checkout -b tmp BB
# 将变更添加到本地暂存
$ git add -A
# 提交记录 VV
```

现在流程分支结构类似：

```sh
AA-BB-VV	(tmp)
    \
     CC-DD	(master)
```

在当前 `tmp` 分支执行 **`rebase`** 命令：

```sh
$ git status
  位于分支 tmp
  无文件要提交，干净的工作区

# 执行完成后，将把当前 tmp 分支 VV 后接上 master CC-DD，并将其当做更新后的 master
$ git rebase tmp master
  成功变基并更新 refs/heads/master。
```

### 副作用

VV 之后接的 Common Log 的 SHA 被更新，在原 SHA 上的 **`tag`** 标签未能同步到新的 SHA 上。

解决办法是获得原由 *原标签* 的**名称**及**提交日期时间**，手动映射到相同日期时间的新 SHA 上。

```sh
# 在当前更新后的 master 流
$ git status
  位于分支 tmp
  无文件要提交，干净的工作区
  
# 查看历史标签列表
$ git tag -l

# 获得每个标签的详细信息，名称、日期时间（重要）
$ git show TAG_NAME
commit 5dfab871d402931467d639dd8e217ef238705cf6 (tag: TAG_NAME)
Author: Reion <reion78@gmail.com>
Date:   Tue May 28 20:15:28 2024 +0800

# 查看相应日期时间对应的新 SHA 值，此处为：eb1a65e37632539a7000d8e5bf3f9e25bf16f202
$ git log
commit eb1a65e37632539a7000d8e5bf3f9e25bf16f202
Author: Reion <reion78@gmail.com>
Date:   Tue May 28 20:15:28 2024 +0800

# 将原标签名打在此 eb1a65e37632539a7000d8e5bf3f9e25bf16f202 上
$ git tag --force TAG_NAME eb1a65e37632539a7000d8e5bf3f9e25bf16f202
```

### 


