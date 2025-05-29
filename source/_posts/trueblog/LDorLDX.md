---
title: 内核加载seccomp：syscall_nr赋给了X寄存器？
date: 2025/05/28 18:38:00
updated: 2025/05/30 03:13:00
tags:
    - kernel
    - non-ctf
thumbnail: /assets/trueblog/ld_or_ldx.png
excerpt: >-
    最近 *dbgbgtf* 在编写ceccomp的代码，为了做和内核一样的bpf检查，他研究了内核对于
    `seccomp`系统调用传入的结构体做的检查。在检查过程中，他发现原本指令中的
    `LD|W|ABS`被内核替换成了`LDX|W|ABS`，而这是从`seccomp_data`中加载`nr`等关键数据结构的指令。
    难道syscall_nr其实在内核中一直是加载到X寄存器中，而非A寄存器中？
---

最近 *dbgbgtf* 在编写ceccomp的代码，为了做和内核一样的bpf检查，他研究了内核对于
`seccomp`系统调用传入的结构体做的检查。在检查过程中，他发现原本指令中的
`LD|W|ABS`被内核替换成了`LDX|W|ABS`，而这是从`seccomp_data`中加载`nr`等关键数据结构的指令，
这令他非常不解：难道syscall_nr其实在内核中一直是加载到X寄存器中，而非A寄存器中？

![strange](/assets/trueblog/ld_or_ldx.png)

为了测试是否真正加载到X中，我们准备了如下"BPF Text"喂给`ceccomp`来生成bpf用以加载到内核中：

```plaintext
$A = $syscall_nr
$A = $X
return $A
```

然后去trace内核中程序的输出结果（code path:
`do_syscall_64 -> syscall_enter_from_user_mode -> syscall_enter_from_user_mode_work -> syscall_trace_enter -> __secure_computing`）
最终的结果很遗憾，是0，也就是说，X寄存器并没有被赋值。我们甚至大费周章地检查了运行bpf的代码，
并没有任何问题。

我提出blame一下内核源码，可以看出在修改之前，老代码并没有将LD转换成LDX。

![blame](/assets/trueblog/ldx_blame.png)

既然“侧信道”不行，那还是得看加载bpf的代码。再次观察加载的过程，从`bpf_prepare_filter`
检查完用户输入的bpf，没有问题后，会调用`bpf_jit_compile`/`bpf_migrate_filter`
来将用户的bpf转换成内核自己的代码，继续跟踪，查看`bpf_convert_filter`，终于水落石出：
`LD|ABS|W`会通向`convert_bpf_extensions`该函数内的检查的abs的k都是有关 **socket buffer**
的，没有和seccomp相关的数据结构，如果是`LDX|ABS|W`，那才是真正取得seccomp的数据。

```c /v6.15/source/net/core/filter.c#L879
		/* Access seccomp_data fields. */
		case BPF_LDX | BPF_ABS | BPF_W:
			/* A = *(u32 *) (ctx + K) */
			*insn = BPF_LDX_MEM(BPF_W, BPF_REG_A, BPF_REG_CTX, fp->k);
			break;
```

{% note blue fa-info %}
BPF设计之初就是为了处理网络包的，seccomp只是借用了BPF的用法，因此内核开发者为了避免两者混淆做的
workaround无可厚非。
{% endnote %}

## 总结

看似内核将seccomp请求中的`LD|ABS|W`换成了`LDX|ABS|W`，但是在实际处理bpf的时候，
仍然是将有关数据加载到A寄存器中。有以上的代码不难发现，如果不换，那么加载的数据是
**socket buffer**而非 **seccomp**，因此为了避免干扰老代码，内核做了一个workaround，
把关于seccomp的加载绕了一道，使用`LDX`来曲线救国。这也能解释为什么bpf加载数据只能用A寄存器，
不能使用X寄存器，即使bpf标准允许这么做：`LD|W|ABS`和`LDX|W|ABS`都有自己的含义，不能混用。
