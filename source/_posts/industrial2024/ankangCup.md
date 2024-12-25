---
title: “安康杯” 青年网络安全技能大赛
date: 2024/12/25 19:01:00
updated: 2024/12/25 19:01:00
tags:
    - awdp
    - offline
thumbnail: /assets/industrial2024/ankang.jpg
excerpt: 安康杯上一道很简单的awdp pwn，虽然只要patch掉后门就可以拿防御分了，但是实际漏洞点不止1个。最后我培训的队伍成功patch了，取得了不错的成绩。
---

之前帮上海的一队培训awd，他们就是培训的下一周要参加这个竞赛了。虽然他们的pwn
能力没有提高很多，但是patch是会了，最后也是取得了一个不错的成绩。

## chall

比赛里有唯一一道awdp pwn题，程序里有后门，patch掉就过了。但是漏洞不止一个。

### 文件属性

|属性  |值    |
|------|------|
|Arch  |amd64 |
|RELRO |Full  |
|Canary|on    |
|NX    |on    |
|PIE   |on    |
|strip |no    |

### 潜在漏洞点

{% note grey fa-bug %}
漏洞点一：后门
{% endnote %}

![bug1](/assets/industrial2024/chall1.png)

在`chall_menu`中可以输入数字，只要等于`0xaaadbeef`就可以进入后门，写一段shellcode。

**参考修复方案**:

使用无条件跳转指令`jmp`跳过后门函数。

```diff
@@ -13cd,18 +13cd,18 @@
 E8 C1 FF FF FF         # call chall_menu
 89 45 FC               # mov  dword ptr [rbp - 4], eax
 81 7D FC EF BE AD AA   # cmp  dword ptr [rbp - 4], 0xaaadbeef
-75 1D                  # jne  0x2e
+EB 1D                  # jmp  0x2e
 48 8D 3D 54 02 00 00   # lea  rdi, [rip + 0x254]
```

**参考攻击方案**:

输入该数字，并输入一段shellcode拿shell。

{% note grey fa-bug %}
漏洞点二：信息泄露
{% endnote %}

![bug2](/assets/industrial2024/chall2.png)

向`NAME`上读取了0x20字节，由于紧邻`RANDBUF`且打印`NAME`时使用`puts`，
因此刚好输入0x20字节可以在打印时泄露`RANDBUF`上放的指针，泄露程序基址。

**参考修复方案**:

修改读取大小为0x1f。

```diff
@@ -12f7,16 +12f7,16 @@
-BA 20 00 00 00         # mov  edx, 0x20
+BA 1f 00 00 00         # mov  edx, 0x1f
 48 8D 35 DD 0D 20 00   # lea  rsi, [rip + 0x200ddd]
 BF 00 00 00 00         # mov  edi, 0
 E8 F3 F6 FF FF         # call read@PLT
```

**参考攻击方案**:

输入0x20字节后执行`print_username`推算出程序基址，然后使用`set_username`
设置`NAME`为`"/dev/zero"`，同时设置`RANDBUF`指向`NAME`。这样在`game`中猜数时，
就会打开`/dev/zero`，读取出4个0x00，只要猜数字是0就可以继续到`chall2`。

{% note grey fa-bug %}
漏洞点三：UAF
{% endnote %}

![bug3](/assets/industrial2024/chall3.png)

在`copy`函数中，如果指定源索引和目标索引为同一个，那么由于其对应的size是一致的，
指针会被保存下来并在free后还原回去，从而造成UAF。

**参考修复方案**:

验证源堆块和目标堆块大小时，只允许目标堆块比源堆块大，避免使用同一个索引。（偷懒了）

```diff
@@ -f12,e +f12,e @@
 8B 04 01       # mov eax, dword ptr [rcx + rax]
 39 C2          # cmp edx, eax
-7C 1B          # jl  0x22
+7E 1B          # jle 0x22
 8B 45 F4       # mov eax, dword ptr [rbp - 0xc]
 48 C1 E0 04    # shl rax, 4
```

**参考攻击方案**:

利用同一个索引造成UAF，泄露堆指针，然后如法炮制，多释放几个。接着申请堆块放上任意分配的地址，
用`copy`把任意分配地址写到bin里，就可以进行下一步利用（泄露libc/打free_hook等）
