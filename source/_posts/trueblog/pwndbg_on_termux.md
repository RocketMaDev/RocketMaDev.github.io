---
title: 在termux上运行pwndbg
date: 2025/08/09 15:44:00
updated: 2025/08/14 22:54:00
tags:
    - non-ctf
    - tricks
thumbnail: /assets/trueblog/termux_thumbnail.jpg
excerpt: >-
    在开发过程中，调试器是绕不开的一环，如果使用gdb的话，pwndbg是必不可少的伴侣。
    平时大家主要在 PC 上使用 pwndbg，但偶尔只带了手机时，也需要在手机上做一些探索。
    借助termux，可以实现在手机上编译并调试，不过只有原版的gdb未免过于简陋，有没有办法将
    pwndbg搬到手机上呢？有的，兄弟...
---

在开发过程中，调试器是绕不开的一环，如果使用gdb的话，pwndbg是必不可少的伴侣。
平时大家主要在 PC 上使用 pwndbg，但偶尔只带了手机时，也需要在手机上做一些探索。
借助termux，可以实现在手机上编译并调试，不过只有原版的gdb未免过于简陋，有没有办法将
pwndbg搬到手机上呢？有的，兄弟，有的。

# 获取官方二进制包

打开[pwndbg的releases页](https://github.com/pwndbg/pwndbg/releases)，
里面已经有针对arm64的包了，可以使用curl等工具下载后解压。然而，
直接解压后并不能直接运行：

<img src="/assets/trueblog/termux_fail.jpg" width="50%">

# 无法启动？

如果仔细看的话，会发现这个`pwndbg`是一个脚本，实际上会用ld去执行gdb。

```sh bin/pwndbg
#!/bin/sh
dir="$(cd -- "$(dirname "$(dirname "$(realpath "$0")")")" >/dev/null 2>&1 ; pwd -P)"
export TERMINFO_DIRS=/etc/terminfo:/lib/terminfo:/usr/share/terminfo:/run/current-system/sw/share/terminfo:$dir/share/terminfo
export PYTHONNOUSERSITE=1
export PYTHONHOME="$dir"
export PYTHONPATH=""
export PATH="$dir/bin/:$PATH"

exec "$dir/lib/ld-linux-aarch64.so.1" "$dir/exe/gdb" --quiet --early-init-eval-command="set auto-load safe-path /" --command=$dir/exe/gdbinit.py "$@"
```

继续检查lib路径，其中都是glibc一类的打包时的库，如果链接`libc.so`到`libc.so.6`的话，
就会显示缺少了符号。对啊，lib中就没有`libc.so`这个文件，它一般是安卓程序的依赖。
检查`LD_PRELOAD`，发现确实提前加载了一个`libtermux-exec-ld-preload.so`。
那就把`LD_PRELOAD` unset掉试试？这样就不会引入安卓的依赖了。然而，又出错了：

<img src="/assets/trueblog/termux_invalsys.jpg" width="50%">

什么？这次直接报 *invalid syscall* 了？难道是有seccomp？我这么猜测，并用`seccomp-tools`
扫描了各种进程的seccomp filter（有root权限），结果都没有发现。这究竟是怎么回事？

我去求助pwndbg群友(discord)，既不能设置`LD_PRELOAD`，又不能unset之，该如何是好？
*@cypis* 检查了audit日志，并最终发现pwndbg调用了`set_robust_list`系统调用，
并被seccomp杀了。他发现，`proot`会[处理这些安卓上不能用的syscall](https://github.com/termux/proot/blob/master/src/tracee/seccomp.c)，
因此使用`proot`启动就可以了。

{% note green fa-heart %}
感谢 *@cypis* 帮我找到问题所在，并找到解决方案！
{% endnote %}

# 写一个wrapper

基于以上考虑，只需要在`unset LD_PRELOAD`后，使用`proot`运行就可以了

```sh $PREFIX/bin/pwndbg
#!/bin/sh
unset LD_PRELOAD
exec proot $PREFIX/../home/pwndbg/bin/pwndbg "$@"
```

然后在命令行里直接调用就可以启动了。

<img src="/assets/trueblog/termux_success.jpg" width="50%">

# 题外话

为了查看程序的沙箱，我尝试把[ceccomp](https://github.com/dbgbgtf1/Ceccomp)
移植到termux上，一开始有问题，后面一一解决了，现在只要`LD_FLAGS`加一个`-largp`就可以了。

一般来说检查程序已经应用的沙箱是需要`CAP_SYS_ADMIN`特权的，结果普通用户也可以正常检查，
我以为是给普通用户返回空结果了呢，结果root用户也检查不出来，最后看安卓内核源码发现，
根本没机会返回，内核直接拒绝了...

```c common/include/linux/seccomp.h
static inline long seccomp_get_filter(struct task_struct *task,
				      unsigned long n, void __user *data)
{
	return -EINVAL;
}
```

具体是怎么知道的呢？可以检查手机的内核选项

```sh
sudo zgrep CONFIG_CHECKPOINT_RESTORE /proc/config.gz
# CONFIG_CHECKPOINT_RESTORE is not set
```

因此这个选项并没有被启用，也就无法调用这个接口了。

# 参考

1. [Releases - pwndbg/pwndbg](https://github.com/pwndbg/pwndbg/releases)
2. [proot/src/tracee/seccomp.c at master](https://github.com/termux/proot/blob/master/src/tracee/seccomp.c)
3. [dbgbgtf1/Ceccomp: A tool to resolve seccomp](https://github.com/dbgbgtf1/Ceccomp)
4. [seccomp.h: Android Code Search](https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/include/linux/seccomp.h;l=94;drc=354893f20269ea62e322c7bc371f12e3fd606e53)
