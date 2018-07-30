---
title: C／C++ 编译相关必知
key: 10004
tags: 编译
layout: post
category: blog
comment: true
date: 2018-07-24 17:34:00 +08:00
modify_date: 2018-07-24 17:34:00 +08:00
---



单文件程序编译阶段
---------

以hello.cpp为例
```
g++ hello.cpp -o hello
./hello
```
静态程序的生命周期大致历经如下步骤
1. 源文件 (.c/.cpp/.h)
2. 预处理 (.c/.cpp/.h => ./i)
3. 编译器 (./i => ./s)
4. 汇编器 (./s => .out/.o)
5. 链接器
6. 可执行文件 (.o => .exe/.out)


多文件构建系统
-----------

- make (Makefile)
项目变大之后，源文件一个一个去手动去编译显然不现实，make就是为了解决这个问题，它是一个自动化编译的工具，可以实现一条命令编译所有源文件。
但是前提它需要知道源文件之间的依赖关系，记录这些依赖关系规则的文件就是Makefile文件。

- configure
配置程序，最终生成Makefile文件

- cmake
更高级的构建系统，需要编写CMakeLists.txt文件(Clion IDE)，cmake根据该文件生成最终的Makefile文件

- make install
将make编译生成的可执行文件安装到执行目录