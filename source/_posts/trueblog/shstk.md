---
title: 把 ROP 全都防出去：启用阴影栈
date: 2025/10/28 10:30:00
updated: 2025/10/28 10:30:00
tags:
    - tricks
    - ROP
excerpt: >-
    我们在这几年的题目里常常看见题目给的ELF里出现大量的`endbr64`指令，
    但是我们总是无视它，毕竟它从来不起作用。那么这个指令原先设计的用意是什么呢？
    直接搜索，我们发现它和Intel CET强相关，这是Intel引入的一项安全措施，
    旨在尽量减少ROP/COP/JOP攻击。具体来说，要想开启CET，需要...
thumbnail: /assets/trueblog/shstk/clear_env.png
---

# 引入

我们在这几年的题目里常常看见题目给的ELF里出现大量的`endbr64`指令，
但是我们总是无视它，毕竟它从来不起作用。那么这个指令原先设计的用意是什么呢？
直接搜索，我们发现它和Intel CET强相关，这是Intel引入的一项安全措施，
旨在尽量减少ROP/COP/JOP攻击。具体来说， **IBT** 措施会让处理器每次在执行
`call`、`jmp`等操作时，检查要跳转的地址有没有出现`endbr*`指令，否则就引发段错误。
**SHSTK** 会在`call`时向阴影栈中也压入返回地址，并在返回时检查栈上的返回地址是否和阴影栈上的一致。

{% note blue fa-book-bookmark %}
**IBT**: Indirect Branch Tracking, 间接分支追踪；
**SHSTK**: Shadow Stack, 阴影栈。
{% endnote %}

# 在 Linux 中启用 CET

现在的主线Linux已经支持CET了，但是开启需要大量的条件，具体可以看[这篇博客]。
省流：

[这篇博客]: https://h3xduck.github.io/cfi/2025/06/26/enabling-intel-cet.html

- 处理器支持：Intel 11代及以后；AMD 5000系以后 (AMD不支持IBT，只有SHSTK)
- 内核支持：Linux 5.18+ 支持IBT；Linux 6.6+ 支持SHSTK(用户态支持)
- 编译器支持：GCC 8.1+；Clang 11+
- 应用支持：需要编译时使用`-fcf-protection=full`编译
- 依赖库支持：所有依赖库必须使用以上旗标编译
- 运行时支持：通过在环境变量里设置`GLIBC_TUNABLES=glibc.cpu.hwcaps=SHSTK`通知LD启用CET

满足所有条件后，才能打开CET保护。

{% note yellow fa-triangle-exclamation %}
根据[Linux文档]，SHSTK通过`arch_prctl(ARCH_SHSTK_ENABLE, ARCH_SHSTK_SHSTK)`开启，
但是我们不能自己开启，否则系统调用返回时阴影栈开启了，却没有返回条目，就会直接崩溃。

[Linux文档]: https://docs.kernel.org/next/x86/shstk.html
{% endnote %}

# 测试 SHSTK

我的CPU是AMD CPU，只能测试SHSTK特性了。编写测试代码：

```c test.c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
static void hack() {
    puts("HIJACKED!!!");
    _exit(0);
}

int main(int argc, char **argv) {
    if (argc == 1) {
        puts("require a number");
        return 1;
    }
    register int num = atoi(argv[1]);
    long buf[1];
    for (register int i = 0; i < num; i++)
        buf[i] = (long)hack;
    return 0;
}
```

我们将其编译后，尝试运行查看结果：

![shstk](/assets/trueblog/shstk/test_shstk.png)

可以看到SHSTK开启后直接阻止了ROP的执行。继续使用gdb尝试检查，就是在返回时崩溃的。

![crash](/assets/trueblog/shstk/shstk_crash.png)

同时查看journalctl可以看到报错记录，由 *near ret* 引发。

```plaintext
10月 29 00:24:07 aRchOG kernel: a.out[22715] control protection ip:5597371d321d sp:7ffe7746dc18 ssp:7f9a21ffffe8 error:1(near ret) in a.out[121d,5597371d3000+1000]
```

# 尝试绕过限制

有没有办法能绕过shstk的限制，执行rop呢？如果我们搜索call后的返回地址，发现它出现了两次，
另一个出现在阴影栈上。阅读Linux源码，可知它对用户态是只读，内核态可读写。

```c linux/v6.17.5/source/arch/x86/kernel/shstk.c#L111
static unsigned long alloc_shstk(unsigned long addr, unsigned long size,
                                 unsigned long token_offset, bool set_res_tok)
{
        ...
        mapped_addr = do_mmap(NULL, addr, size, PROT_READ, flags,
                              VM_SHADOW_STACK | VM_WRITE, 0, &unused, NULL);
        ...
}
```

## 尝试直接写 rw 阴影栈

经过测试，这个阴影栈刚好出现在libc的低地址处，并且在`maps`文件上显示为`rw-`权限。
难道可以直接写？编写poc测试，可以看到直接崩溃了。

```c test.c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
    char buf[0x20];
    char *save = NULL;
    fgets(buf, 0x20, stdin);
    register long num = strtol(buf, &save, 0);
    if (save == buf) {
        puts("not a number");
        return 1;
    }
    *(long *)num = 0xdeadbeef;
    return 0;
}
```

![no write](/assets/trueblog/shstk/no_write.png)

## 尝试先将阴影栈权限变更为 rw

那我们先用`mprotect`将阴影栈变更为`rw-`呢？也不行。

```c test.c
#include <assert.h>
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/mman.h>

int main(int argc, char **argv) {
    char buf[0x20];
    char *save = NULL;
    fgets(buf, 0x20, stdin);
    register long num = strtol(buf, &save, 0);
    if (save == buf) {
        puts("not a number");
        return 1;
    }
    long rc = mprotect((void *)num, 0x1000, PROT_READ | PROT_WRITE);
    assert(!rc);
    *(long *)num = 0xdeadbeef;
    return 0;
}
```

![still no write](/assets/trueblog/shstk/still_no_write.png)

## 清除环境变量

最后再尝试一次，由于要想启用SHSTK，必须存在这个环境变量，那如果我们预先清除这个环境变量，
再次执行程序，会不会有变化呢？结果发现确实能关闭SHSTK。在SHSTK变成默认启用前，
恐怕这是唯一的绕过方式。（如果默认启用的话恐怕就无法关闭了）

```c test.c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
static void hack() {
    puts("HIJACKED!!!");
    _exit(0);
}

int main(int argc, char **argv) {
    if (argc == 1) {
        unsetenv("GLIBC_TUNABLES");
        execl("./a.out", "./a.out", "100");
    }
    register int num = atoi(argv[1]);
    long buf[1];
    for (register int i = 0; i < num; i++)
        buf[i] = (long)hack;
    return 0;
}
```

![bypass](/assets/trueblog/shstk/clear_env.png)

{% note blue fa-forward %}
由于篇幅原因，没有展示代码正确性验证，实际上我自己测试过是正确的（例如修改只读段的权限，
没有崩溃），也可以自己测试示例代码。
{% endnote %}

# 参考

1. [How to enable Intel CET | h3xduck blog](https://h3xduck.github.io/cfi/2025/06/26/enabling-intel-cet.html)
2. [Control-flow Enforcement Technology (CET) Shadow Stack](https://docs.kernel.org/next/x86/shstk.html)
