---
title: moectf2023 - Baby Calculator
date: 2023/9/23 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 通过连接服务器并自动回答100道加法题，成功获取flag。
---

## 文件分析

没有文件

## EXPLOIT

nc直连，发现要检查100道加法题的正确与否  
~~直接算不就行了，100道题罢了~~

```python
from pwn import *
sh = remote("localhost", 39679)

for _ in range(100):
    sh.recvuntil(b'nd:') # skip
    sh.recvline() # skip
    eq = sh.recvline().decode() # e.g. b'9+37=45\n'
    eq = eq.split('+')
    eq.extend(eq[1].split('='))
    if int(eq[0]) + int(eq[2]) == int(eq[3]): # int("45\n") == 45
        sh.sendline(b'BlackBird')
    else:
        sh.sendline(b'WingS')

sh.interactive()
```

直接拿到flag  
就是连接有时间限制，不能长时间挂着

Done.
