---
title: 初探Linux内核调试
date: 2025/04/21 12:20:00
updated: 2025/04/22 00:38:00
tags:
    - kernel
excerpt: |-
    这学期有操作系统实验课，第一课就是要求在Linux内核中实现一个系统调用，显示学号。
    由于系统调用不能通过加内核模块来实现，因此必须要拉取内核源码并编译。
    我想着正好我没有做过内核题，也没有配环境，于是决定借此机会配置一个可调试的内核。
    正好遇到能源比赛的题中发现当`write`的size很大时，写的字节数是0x800的倍数，
    而非对齐到内存页边界上，直接启动内核源码调试分析！最后发现...
thumbnail: /assets/trueblog/kernelHacking.png
---

这学期有操作系统实验课，第一课就是要求在Linux内核中实现一个系统调用，显示学号。
由于系统调用不能通过加内核模块来实现，因此必须要拉取内核源码并编译。
我想着正好我没有做过内核题，也没有配环境，于是决定借此机会配置一个可调试的内核。
作为Arch用户，直接到 kernel.org 下载当时最新的Linux源码包(6.14)。

## 编译

课程要求是要在华为云上，用aarch64的鲲鹏处理器，更改openEuler的源码并重新编译，
然后安装到云服务器中并在开机时选择对应的选项。这实在是太低效了！要给整个机器编译，
就意味着要加很多的驱动，而且云服务器配置不高，导致编译时间会很长。与其上云，
不如在本地调试。我直接编译x86_64的Linux，然后用qemu启动，那不是快多了？

{% notel purple fa-screwdriver-wrench 调整内核的特性 %}
为了缩短编译的时间，我希望尽可能减少要编译的特性，于是，我一开始选择了使用
**tinyconfig**，然后在这基础上用 **menuconfig** 添加一些看起来重要的选项加上。
然而，qemu并不能正常启动。没办法，我只能使用 **defconfig**，然后再减掉一些特性，
比如图形界面（直接使用`-nographic`启动内核）。别忘了把 *kernel hacking*
调试符号打开（使用`/`可以搜索）。

> 后面我又尝试了一次，调了几个小时，根据成功的配置，加上AI的帮助，从
> **tinyconfig** 开始配，还是没成功，没有经验千万不要从 **tinyconfig**
> 开始！
{% endnotel %}

{% note green fa-cubes %}
busybox和qemu等工具Arch直接包管理提供，而且特性基本是全开的，不需要自己编译，特别爽。
{% endnote %}

## write: 3种不一样的行为

在 *dbgbgtf* 师傅打刚刚过去的能源比赛时，他把`strlen` hijack到`malloc`，
之后`strlen`的结果会作为`write`的参数，作为size打印一个`.rodata`段的字符串。
由于此时size是非常非常大的，他期望pwntools中能看到很多很多的字符；
然而，现实结果却是无论size多大，始终打印0x1800个字节。这让我完全不能理解：
那个字符串离相邻最高的页边界还差不到0x2000字节，而不是刚好0x1800个，
为什么不把所有的页边界之前的字节全输出出来呢？于是写了点代码测试：

```c test.c
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>
extern long sys_write(int, void *, unsigned long);

int main(int argc, char **argv) {
    int fd;
    if (argc == 1)
        fd = 2;
    else if (!strcmp(argv[1], "/dev/null"))
        fd = open("/dev/null", O_WRONLY);
    else if (!strcmp(argv[1], "file"))
        fd = open("garbage", O_WRONLY | O_CREAT);
    else {
        printf("Usage: %1$s | %1$s /dev/null | %1$s file\n", argv[0]);
        return 1;
    }

    printf("fd = %d\n", fd);
    int cnt = sys_write(fd, "abc", 0x5000);
    printf("written: %#x\n", cnt);
    return 0;
}
```

```x86asm write.s
    .file   "write.s"
    .intel_syntax noprefix
    .text
    .p2align 4
    .globl  sys_write
    .type   sys_write, @function
sys_write:
.LFB0:
    mov eax, 1
    syscall
    ret
.LFE0:
    .size   sys_write, .-sys_write
    .ident  "Rocket (Arch Linux 20250418)"
    .section    .note.GNU-stack,"",@progbits
```

1. `fd = STDERR_FILENO`，写入0x2800字节
2. `fd -> /dev/null`，写入0x5000字节
3. `fd -> normal file`，写入0x2fdc字节
4. `close(fd)`，写入-9(*EBADF*)字节

除去关闭fd后输出负数的case，剩下三个fd返回的数字都不一样，这令我十分困惑。
配置好内核调试符号，启动内核调试！

## 用Makefile加速内核调试启动

如果不考虑把文件映射到虚拟机中，那么每当我们修改了要运行的elf或者init脚本后，
就需要手动打包`initramfs.cpio`。用脚本来自动打包固然方便，但是如果源码未发生改变，
那便无需使用脚本来重复打包。正好， **GNU Make** 就是做这个事情的。
Make有依赖项一说，会根据产物及其依赖的修改时间决定是否需要重新编译，
正好符合“Lazy”的要求；而且写Makefile就像写函数，还能很方便的交给zsh分析并补全，
没有不用的道理。

{% note yellow fa-circle-exclamation %}
`Makefile`的缩进只能用`\t`来区分层级，不能使用空格！vim在编辑Makefile时，
即使在`.vimrc`中指定了`expandtab`，也会自动关闭，请不要尝试自行打开，
否则在make的时候会显示没有分隔符而无法运行。
{% endnote %}

```make Makefile
GREEN := $(shell printf '\033[92m')
RESET := $(shell printf '\033[0m')
ifdef DEBUG
	QEMUOPT := -gdb tcp::1337
endif
all: cpio run

cpio: rootfs.cpio.gz
	@echo '$(GREEN)rootfs generated$(RESET)'

rootfs.cpio.gz: test initramfs/init
	cp $< initramfs
	cd initramfs && find . | cpio -H newc -ov --owner=0:0 | gzip > ../$@

test: test.c write.s
	gcc -o $@ -g -O2 $^ -Wl,--image-base 0x1000000 -no-pie
	@echo '$(GREEN)test updated$(RESET)'

run:
	qemu-system-x86_64 -kernel bzImage                 \
		-initrd rootfs.cpio.gz                         \
		-append "console=ttyS0 init=/init nokaslr"     \
		-nographic $(QEMUOPT)

debug: cpio
	qemu-system-x86_64 -kernel bzImage                 \
		-initrd rootfs.cpio.gz                         \
		-append "console=ttyS0 init=/init nokaslr"     \
		-gdb tcp::1337 -S                              \
		-nographic
	
gdb:
	read yama < /proc/sys/kernel/yama/ptrace_scope;     \
		if [[ $$yama -gt 0 ]] \
		then \
		    echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope; \
		fi
	gdb vmlinux -x startup.gdb
.PHONY: all cpio run gdb debug
```

{% notel green fa-circle-info Makefile要点解析 %}
1. `cpio`的参数`--owner=0:0`将设置cpio中文件的所有者为root，而非打包者。
当时有一道内核题就因为cpio权限没设好，`poweroff`是可写的，就可以把`poweroff`
换成打印flag的脚本来非预期。
2. 在内核中设置用户态程序断点需要知晓用户态程序断点的地址，
为了防止我们的elf和其他程序冲突，可以关闭PIE并设置程序基址，方便下固定断点。
3. 即使gdb访问的是gdbserver，要想使用vmmap仍然需要将`ptrace_scope`设为0。
使用以上命令在每次启动gdb调试时测试`ptrace_scope`的值，仅在必要时将其设置为0，
保证平时运行时host的安全。
4. `qemu-system-x86_64`后面跟的`-S`参数指示qemu在虚拟机启动时挂起，
等待调试器连接。
{% endnotel %}

最后使用tmux切出两个窗口，左边`make debug`，右边`make gdb`就可以开始调试了。

## 调试内核syscall

使用`objdump --disassemble=sys_write -M intel test`取得syscall指令的地址，
复制下来以后就可以放进gdb里下断点，接着让虚拟机跑起来，并运行`/test`，
此时在syscall的地方`si`就可以进入内核系统调用函数了。

![attach kernel](/assets/trueblog/attachKernel.png)

这三种情况都从`ksys_write`函数出发，由内核file结构体的函数指针开始分叉，
总结下来三种情况的执行链大致如下：

```plaintext
/proc/self/fd/2: console.write_iter -> redirected_tty_write -> file_tty_write -> iterate_tty_write
/dev/null:       null.write -> write_null
./garbage:       shmem.write_iter -> shmem_file_write_iter -> generic_perform_write -> copy_folio_from_iter_atomic
```

对于null设备，`write_null`直接返回请求写入的大小，没有其他任何处理，
因此请求写入多少就“写入多少”。对于普通文件，会尝试输出所有可写的字节，
即写入文件直到遇到内存边界。唯一比较特殊的就是tty，看以下函数实现，
可以看到当遇到内存边界时，直接退出循环了，没有写剩余字节：

```c linux/v6.14/source/drivers/tty/tty_io.c#L961
static ssize_t iterate_tty_write(struct tty_ldisc *ld, struct tty_struct *tty,
                                 struct file *file, struct iov_iter *from)
{
        ...
        chunk = 2048;
        ...
        /* Do the write .. */
        for (;;) {
                size_t size = min(chunk, count); // 如果剩余字节数大于0x800，那么每次写0x800

                ret = -EFAULT;
                if (copy_from_iter(tty->write_buf, size, from) != size)
                        break;

                ret = ld->ops->write(tty, file, tty->write_buf, size);
                if (ret <= 0)
                        break;
                written += ret;
                ...
        }
        if (written) {
                tty_update_time(tty, true);
                ret = written;
        }
out:
        ...
        return ret;
}
```

注意看第12行，`copy_from_iter`将用户空间缓冲区复制到内核缓冲区中，
返回复制的字节数。当没遇到内存边界时，`copy_from_iter`的返回值和`size`相等，
执行下面的`ld->ops->write`；但是遇到内存边界时，对于我们的情况来说，
`copy_from_iter`的返回值是0x7dc，而`size`是0x800，两者不等，break跳出去，
结果就是页边界最后的0x7dc个字节复制了，但是没输出，最后只输出了0x2800字节。

## 参考

1. [The Linux Kernel Archives](https://www.kernel.org/)
2. [通过gdb调试内核和模块](https://www.kernel.org/doc/html/next/translations/zh_CN/dev-tools/gdb-kernel-debugging.html)
3. [tty_io.c - drivers/tty/tty_io.c](https://elixir.bootlin.com/linux/v6.14/source/drivers/tty/tty_io.c#L961)
