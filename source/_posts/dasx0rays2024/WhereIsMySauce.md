---
title: DASCTF 2024金秋十月 WhereIsMySauce
date: 2024/10/22 23:44:00
updated: 2024/10/23 19:03:00
tags:
    - tricks
    - challenge author
thumbnail: /assets/dasx0rays2024/debuginfod.png
excerpt: debuginfod 是二进制调试当中非常好用的工具，可以从网上自动拉取调试符号和源码。题目名叫做 WhereIsMySauce，小小玩了一个谐音梗，Sauce 谐音 source， 实际上是在暗指 flag 就放在源码里。由于 debuginfod 是可以分发源码的， 而源码是最便于显示 flag 的方式，因此这道题的终极目标就是找到藏有 flag 的源码。
---

## 前言

**debuginfod** 是二进制调试当中非常好用的工具，可以从网上自动拉取调试符号和源码，
无需预先下载好，具体请看[我早前写的博客](/2024/05/08/debuginfod/)。

## 出题思路

题目名叫做`WhereIsMySauce`，小小玩了一个谐音梗，`Sauce`谐音`source`，
实际上是在暗指flag就放在源码里。由于`debuginfod`是可以分发源码的，
而源码是最便于显示flag的方式，因此这道题的终极目标就是找到藏有flag的源码。

pwner一般都知道，pwndbg是可以在有调试符号的情况下显示源码的，并且在程序崩溃时会停在崩溃的地方，
因此这也是我设计的切入点。直接把flag放在顶层文件里有点简单了，于是我设计了让程序崩溃的机制：
只要不加酱油就崩溃，因此正确做法就是打开debuginfod，并设置服务器为靶机，然后打开gdb，
直接让程序崩溃即可打印flag（没有pwndbg插件就手动`l`一下）。

我对这道题的预期难度是中等甚至简单，最后却只有一位师傅做出来了，也不知道大家卡在哪里了。

wp中也提到了可能存在的坑点是debuginfod的缓存机制，如果先前已经设置过服务器，
那么debuginfod会在服务器中查找相关符号，未找到则会创建空文件，导致后续就算服务器设置正确，
debuginfod也不会自动拉取符号，这时候就需要将这题的缓存先清空一下再运行。

```sh
rm -rf ~/.cache/debuginfod_client/$(readelf -n cook | grep Build | awk '{print $3}')
echo 'set debuginfod enabled on' >> ~/.gdbinit
DEBUGINFOD_URLS=$REMOTE gdb cook -q
r
6
```

![flag](/assets/dasx0rays2024/cook.png)

## 做题之外

这道题的所有源码可以在[这里](https://github.com/RocketMaDev/CTFWriteup/tree/main/dasx0rays2024/sources/WhereIsMySauce)找到。
当时在起server的时候还遇到了点麻烦，要想debuginfod server能提供源代码，
不能简单将代码放在某个文件夹里，而是必须放在统一的`/usr/src`下，
只是单单把源码放在家目录下并用`debuginfod -F ~`是无效的，只能分发其中的debuginfo。
与此同时，debuginfo中的源代码路径也不能是相对的，必须是绝对路径，以`/usr/src/`开始，
想用debuginfod来分发源码的师傅可以注意一下。

## 参考

[摆脱调试符号的困扰：使用debuginfod拉取libc的调试符号](/2024/05/08/debuginfod/)

