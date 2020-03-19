---
title: 静态程序构建过程
key: 10007
tags: 操作系统
layout: article
category: blog
comment: true
date: 2018-07-24 17:34:00 +08:00
modify_date: 2018-07-24 17:34:00 +08:00
---

### 静态程序生命周期

以K&R <<C程序设计语言>> 中 hello.c 为例
```
#inclue <stdio.h>

int main()
{
   printf("Hello World\n");
   return 0;
}

gcc hello.c -o hello
./hello
```
如上实现了 hello.c C语言源文件到机器可执行的二进制目标文件 hello的过程。这个过程大致经历了如下步骤：
1. 源文件 (.c/.cpp/.h)
2. 预处理 (.c/.cpp/.h => ./i)
```
gcc -E hello.c -o hello.i
```
3. 编译器 (./i => ./s)：把预处理完的文件进行一系列的词法分析、语法分析、语义分析及优化后生成相应的汇编代码
```
gcc -S hello.i -o hello.s
```
4. 汇编器 (./s => .out/.o)：汇编器将汇编语言翻译成机器语言指令，并打包成可重定位目标程序 hello.o
```
gcc -c hello.s -o hello.o
```
5. 链接器 (.o => .exe/.out)：链接可重定位的二进制文件，形成完整的逻辑地址，生成可执行目标文件
6. 装载：装载进内存，MMU把逻辑地址映射成物理地址

### 多文件构建系统

- make (Makefile)
项目变大之后，源文件一个一个去手动去编译显然不现实，make就是为了解决这个问题，它是一个自动化编译的工具，可以实现一条命令编译所有源文件。
但是前提它需要知道源文件之间的依赖关系，记录这些依赖关系规则的文件就是Makefile文件。

- configure
配置程序，最终生成Makefile文件

- cmake
更高级的构建系统，需要编写CMakeLists.txt文件(Clion IDE)，cmake根据该文件生成最终的Makefile文件

- make install
将make编译生成的可执行文件安装到执行目录

### 引用
- [深入理解计算机系统](https://book.douban.com/subject/5333562/)
- [程序员的自我修养](https://book.douban.com/subject/3652388/)