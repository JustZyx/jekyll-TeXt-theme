---
title: 很实用的一些配置集合
key: 10003
tags: 文档
layout: article
category: blog
comment: true
date: 2018-01-21 17:03:00 +08:00
modify_date: 2018-01-21 17:03:00 +08:00
---


换设备经常会用到的一些东西、有些比较容易忘，在此记录一下

## Vim

### 显示行号

1. 永久

```shell
echo 'set number' >> ~/.vimrc
```

2. 临时

```vimrc
: set number
```

### 批量注释&&取消注释

```shell
:起始行号,结束行号s/^/注释符/g
:起始行号,结束行号s/^注释符//g
```

### 批量替换

```shell
:%s/源字符串/目的字符串/g
```

### tab设置为4个空格

```shell
set ts=4
set expandtab
```

## Shell命令

### scp

1. 上传

```shell
scp 本地文件完整路径 username@ip:远程目录路径
```

2. 下载

```shell
scp username@ip:远程文件完整路径 本地目录
```

## VSCode

### 更换默认shell

```
https://code.visualstudio.com/docs/editor/integrated-terminal#_configuration
```

### 列编辑

`alt + shift + 鼠标滚动`


## OS

### Mac环境变量更改
```shell
cd /etc/paths.d
touch go //go环境变量
touch nginx //nginx环境变量
```
参考: [mac设置环境变量](https://www.jianshu.com/p/acb1f062a925)

## VCS

### Git免密Pull&Push
```shell
git config --global credential.helper store --file ~/.my-credentials
```
参考：[Git工具-凭证存储](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%87%AD%E8%AF%81%E5%AD%98%E5%82%A8)

