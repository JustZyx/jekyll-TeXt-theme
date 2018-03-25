---
title: Git中低频操作
key: 10003
tags: Git
layout: post
category: blog
comment: true
date: 2018-03-25 11:49:00 +08:00
modify_date: 2018-03-25 11:49:00 +08:00
---

## 背景

Git高频操作天天用早就烂熟于心了，此文仅仅记录Git中低频操作。大致符合以下两个特点：

* 不常用且易忘
* 知识盲区，未曾使用过

## 操作列表

### 删除分支

* 删除本地分支

  ```shell
  git branch //列出本地所有分支
  git branch -d feature/branchName //删除其中一个分支
  git branch | grep "branchName" | xargs git branch -D //批量删除带有branchName关键词的分支
  ```

* 删除远程分支

  ```shell
  git branch -r //列出所有远程分支
  git push origin :branchName //删除一个远程分支
  git push --delete origin branchName
  ```

## 分支同步

* git rebase

