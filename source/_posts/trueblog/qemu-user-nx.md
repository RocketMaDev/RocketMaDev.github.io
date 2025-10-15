---
title: QEMU 引入 NX 到 qemu-user 中（栈不可执行）
date: 2025/10/15 19:00:00
updated: 2025/10/15 19:00:00
tags:
    - tricks
    - qemu-user
    - shellcode
thumbnail: /assets/trueblog/qemu-user-nx.png
excerpt: >-
    在网鼎杯决赛，我还利用过 qemu-user 没有 NX 的特性，在栈上执行 shellcode，
    然而在最近的湾区杯，有师傅惊讶地发现本地的 qemu-user 不能在栈上执行 shellcode
    了。是我记错了？还是 qemu 发生了变更，增加了 NX？二分定位版本号后，
    我最终定位到添加 NX 特性的版本是...
---

# qemu-user的nx出现了？

在网鼎杯决赛，我还利用过qemu-user没有nx的特性，在栈上执行shellcode，
然而在最近的湾区杯，有师傅惊讶地发现本地的qemu-user不能在栈上执行shellcode了。
我写了一个简单的示例，用来复现这个问题（使用`-z execstack`来给栈加可执行权限）。

```c
int main(int argc, char **argv) {
    // mov eax, 60; xor edi, edi; syscall;
    // exit(0);
    unsigned char buf[64] = "\xb8<\0\0\0\x31\xff\x0f\x05";
    void (*myexit)(void) = (void (*)(void))buf;
    myexit();
    return 0;
}
```

编译后运行，没开nx会正常退出，开了会 *SIGSEGV* ，这很正常。接下来使用`qemu-x86_64`
来运行，和直接运行没有区别，同样会 *SIGSEGV* 。可是去年我明明用这个特性做出过题啊，
这是怎么回事？

群里师傅发现，远程环境确实没有nx，而且qemu-user是没有ASLR的，因此甚至不用泄露信息。
然而，这些“优良的特性”在新版qemu上荡然无存。接下来的任务就是找到哪个版本引入了变更，
在目标靶机上安装的qemu版本是多少时，需要考虑更加复杂的方案。

# 拉取源码定位差异

从[QEMU的仓库](https://gitlab.com/qemu-project/qemu)拉取最新的源码仓库，然后编译测试特性，
按大版本标签做二分查找，最后定位到`v7.2.0`的
[b34b42](https://gitlab.com/qemu-project/qemu/-/commit/b34b42f1b6a33c455dccce6ceb49962dddbb7a8a)
commit加入了对ELF中`exec_stack` flag的判断。也就是说，从 **qemu-user v7.2.0** 开始，就不能直接栈溢出，
跳到栈上运行了。那么对应到Ubuntu版本，我做了下面这张图，简而言之，从 **Ubuntu 24.04** 开始，
自动开启NX。

<img src="/assets/trueblog/qemu-user-nx.png" width="50%">

其他系统的qemu版本可以在[Repology](https://repology.org/project/qemu/versions)上找到。

# ASLR也有影响？

在最近的变更中，加入了“ASLR”，照理来说原来的版本，程序基地址、库基地址都是固定的，
然而，在x86_64上是这样，aarch64上却不是，在手机上使用qemu-user，虽然是老版本，
但是基地址都不同。因此这里就不罗列信息了。

# 参考

1. [QEMU/QEMU - GitLab](https://gitlab.com/qemu-project/qemu)
2. [Merge tag of rth7680/qemu into staging](https://gitlab.com/qemu-project/qemu/-/commit/b34b42f1b6a33c455dccce6ceb49962dddbb7a8a)
3. [qemu package versions - Repology](https://repology.org/project/qemu/versions)
