---
title: 国赛 x 长城杯 半决赛
date: 2025/03/19 10:16:00
updated: 2025/03/20 02:10:00
tags:
    - offline
    - awdp
    - git
excerpt: 本次国赛暨长城杯半决赛，我们0RAYS战队人人有输出，最终取得了团体第14名的成绩。以下是我在比赛中的一些题解。
thumbnail: /assets/ciscnxccb2025/half_banner.jpg
---

## 赛前风波

这次半决赛浙江赛区是我们杭电承办的，其中半决赛的竞赛QQ群真的是一言难尽...
先是电源功率限制，光是文件就来回发了两次；然后附件也发得不积极，
要学生提醒才发；片哥进群了也不踢一下，在那里广告给他们刷了几十条；
老师好像也不会用QQ似的，群公告不发，遇到重要的事就在那里@全体成员好几次。

附件也抽象，给的zip用 **ZipCrypto** 压缩，一看都知道能明文攻击，
于是就怎么做呢，套两层压缩加密？有什么必要呢？这不是换个加密算法的事？
再就是电源功率，单人超过150W要报备，全队不能超过900W，还是看电源数据，
不看实测，这150W功率不是游戏本随便超？？我参加网鼎杯也不见有这样子奇奇怪怪的规定啊...

## 上半场：AWDP

9点开始，awdp有pwn有web，我打算先patch。上通防没法过检查，只能一个一个看。

### typo

**PATCH**

在edit函数中`snprintf`存在明显误用，正确用法是`snprintf(str, maxlen, format, ...)`，
为了避免潜在的溢出，需做一个参数调换。将其修改为`snprintf((char *)cards[idx],8,"%lu",buf)`。

<img src="/assets/ciscnxccb2025/typo.bug.png" width="50%">

```diff
--- typo
+++ typo
@@ -1662,3b +1662,3b @@
-48 8d 35 09 0a 00 00       # lea rsi, [rip + 0xa09] // "%lu"
+48 8d 15 09 0a 00 00       # lea rdx, [rip + 0xa09] // "%lu"
 8b 85 e4 fe ff ff          # mov eax, dword ptr [rbp - 0x11c] // idx
 48 98                      # cdqe
-48 8d 14 c5 00 00 00 00    # lea rdx, [rax * 8]
+48 8d 3c c5 00 00 00 00    # lea rdi, [rax * 8]
 48 8d 05 e0 29 00 00       # lea rax, [rip + 0x29e0] // cards
-48 8b 04 02                # mov rax, qword ptr [rdx + rax * 1] // cards[idx]
-48 8d 95 f0 fe ff ff       # lea rdx, [rbp - 0x110] // buf
-b9 08 00 00 00             # mov ecx, 8
+48 8b 04 07                # mov rax, qword ptr [rdi + rax * 1] // cards[idx]
+48 8d 8d f0 fe ff ff       # lea rcx, [rbp - 0x110] // buf
+be 08 00 00 00             # mov esi, 8
 48 89 c7                   # mov rdi, rax
 b8 00 00 00 00             # mov eax, 0
 e8 93 fa ff ff             # call snprintf@PLT
```

赛后看了攻击方式，堆溢出以后爆破stdout，还是挺复杂的

### prompt

**PATCH** by *xinhu*

一道protobuf题， ~~但是我没准备~~，首先栈溢出修了肯定是没用的，因为上面直接是
Canary，大不了一点...最后一个回合急了，瞎修了一通就过了...反正原本的功能肯定坏了，
check脚本写的一坨。

### post_quantum

**PATCH**

一个从来没见过的加密系统，也是到了最后一个回合急了，一看在加解密的时候申请了堆块，
跳到函数外又释放掉，盲猜可能存在堆溢出，就把malloc的尺寸调大了一点，结果就过了...

现在在回看，修得莫名奇妙的，在加密函数中把`operator.new[]`的参数调大了一点，
解密函数里的`malloc`调小了也能过，不知道checker是怎么写的，既然是调小，就不贴修复了，
没啥意义...

{% note blue fa-hourglass-end %}
最后交的时候甚至交错了，浪费了一分钟，最后一分钟交了正确的patch，眼看awdp结束了，
还没出结果，结果一刷新，竟然验证通过了！太极限了，最后一回合狂揽六七百分。
{% endnote %}

## 下半场：渗透赛

### 应急响应

看到题目描述说木马会定期回连，我一开始就盯着`ps -aux`的结果看，啥也没有，
问本地的qwen，还可以使用`ss -tulnpe`来查看连接，于是我就让 *xinhu* 盯着，
结果还真盯到一个网络连接不和任何进程关联，交一下，通了。

有一个raw的附件，我们一组一开始都没有磁盘取证工具，一直没用到，直到我尝试使用
qemu启动，嘿，还真行。虽然说没什么用就是了。

{% note green fa-screwdriver-wrench %}
赛后还了解到有个工具叫做`unhide`，可以通过比较ps的结果和proc目录下的文件，
得出被隐藏的进程，看起来很有用。
{% endnote %}

最后听1RAYS说这道题有个内核模块，hook了`getdents`，使其不显示木马，
不取证可能还真想不到吧。

### web-git

*Mak4R1* 问我，用git看有一个flag分支，但是不能直接checkout过去，
我想起可以直接访问底层object的，于是从`.git`中找到所有hash文件，
打印它们的内容就可以找到flag。

{% note green fa-lightbulb %}
使用`git cat-file -p $HASH`可以输出hash文件中存放的文本。
{% endnote %}

<img src="/assets/ciscnxccb2025/gitflag.png" width="70%">

## 回顾

总体而言比赛现场还是氛围挺好的，点心发了面包、牛奶、士力架之类，真的挺多的，
午饭两荤两素加酸奶，伙食很不错。有人“监考”，也不见得有选手能作弊。

{% note green fa-heart %}
感谢 *袁神* 送来的咖啡！
{% endnote %}

最后我们取得了 **第14名** 的好成绩，成功进军国赛、长城杯双决赛。回看座位表，
坐在我们对面的，正是第3名的来自复旦的白泽！！赶紧上去social...

还加了泥巴睡觉队的队员，主动拉我进星盟，十分热情。

<img src="/assets/ciscnxccb2025/half_prize.jpg" width="80%">

~~天津赛区有玻璃奖杯，四川赛区有金杯，怎么浙江赛区只有一张纸啊~~
