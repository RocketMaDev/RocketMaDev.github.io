---
title: Ubuntu 24.04 到 26.04 堆攻击变化总结
date: 2026/03/31 10:54:00
updated: 2026/04/02 16:44:00
tags:
    - heap - tcache
    - heap - fastbin
    - heap - largebin
    - glibc2.39
    - glibc2.42
    - glibc2.43
    - tricks
excerpt: >-
    书接上文，自 glibc 2.39 (Ubuntu 24.04) 后，glibc 陆陆续续已给 malloc 打了 85 个补丁，
    在接下来 Ubuntu 26.04 LTS 即将发布的时间节点，我来梳理一下近期堆攻击可用的向量...
thumbnail: /assets/trueblog/glibc2.42/pwndbg-demo.png
---

[书接上文]，自 glibc 2.39 (Ubuntu 24.04) 后，glibc 陆陆续续已给 malloc 打了 85 个补丁，
在接下来 Ubuntu 26.04 LTS 即将发布的时间节点，我来梳理一下近期堆攻击可用的向量。

[书接上文]: https://roderickchan.github.io/zh-cn/2023-03-01-analysis-of-glibc-heap-exploitation-in-high-version/

# glibc 2.38-2.39 (Ubuntu 24.04)

在上面 *roderick* 师傅整理的笔记中，讲了 2.35-2.37 之间的新增的检查，在 `git diff`
glibc 2.37-2.39 后，没有发现变更 ptmalloc 堆管理器的部分，想看 Ubuntu 22.04-24.04
之间堆利用变更的部分直接看上面的文章就可以。

# glibc 2.41

{% callout default fa-folder-open %}
为什么没有 glibc 2.40？因为这个版本没有对 malloc 模块做变更...
{% endcallout %}

## tcache 满后释放小堆块会直接进入 small bin

于 `e2436d6f` 引入，触发条件为释放堆块，尺寸满足 small bin，
并且 tcache 已满。原先为先进入 unsorted bin，稍后可能会 consolidate，
现在直接进入 small bin。

可能影响堆利用的情况：会影响 unsorted bin 切堆块的行为？
不过一般要切大家都会选尺寸比较大的堆块吧。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=e2436d6f5aa47ce8da80c2ba0f59dfb9ffde08f3

```diff
@@ -4760,23 +4760,39 @@ _int_free_create_chunk (mstate av, mchunkptr p, INTERNAL_SIZE_T size,
       } else
        clear_inuse_bit_at_offset(nextchunk, 0);
 
-      /*
-       Place the chunk in unsorted chunk list. Chunks are
-       not placed into regular bins until after they have
-       been given one chance to be used in malloc.
-      */
+      mchunkptr bck, fwd;
+
+      if (!in_smallbin_range (size))
+        {
+          /* Place large chunks in unsorted chunk list.  Large chunks are
+             not placed into regular bins until after they have
+             been given one chance to be used in malloc.
+
+             This branch is first in the if-statement to help branch
+             prediction on consecutive adjacent frees. */
+          bck = unsorted_chunks (av);
+          fwd = bck->fd;
+          if (__glibc_unlikely (fwd->bk != bck))
+            malloc_printerr ("free(): corrupted unsorted chunks");
+          p->fd_nextsize = NULL;
+          p->bk_nextsize = NULL;
+        }
+      else
+        {
+          /* Place small chunks directly in their smallbin, so they
+             don't pollute the unsorted bin. */
+          int chunk_index = smallbin_index (size);
+          bck = bin_at (av, chunk_index);
+          fwd = bck->fd;
+
+          if (__glibc_unlikely (fwd->bk != bck))
+            malloc_printerr ("free(): chunks in smallbin corrupted");
+
+          mark_bin (av, chunk_index);
+        }
 
-      mchunkptr bck = unsorted_chunks (av);
-      mchunkptr fwd = bck->fd;
-      if (__glibc_unlikely (fwd->bk != bck))
-       malloc_printerr ("free(): corrupted unsorted chunks");
-      p->fd = fwd;
       p->bk = bck;
-      if (!in_smallbin_range(size))
-       {
-         p->fd_nextsize = NULL;
-         p->bk_nextsize = NULL;
-       }
+      p->fd = fwd;
       bck->fd = p;
       fwd->bk = p;
```

## `calloc` 将使用 tcache 来分配堆块

于 `226e3b0a` 引入。这个没啥好说的，意思就在字面上。

{% callout green fa-circle-question %}
原来 `calloc` 不会用 tcache 里的堆块吗？
{% endcallout %}

可能影响：题目中的 `calloc` 行为变化，现在将会影响 tcache。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=226e3b0a413673c0d6691a0ae6dd001fe05d21cd

# glibc 2.42

## 增加从 fastbin 移动堆块到 tcache 的检查

于 `d10176c0` 引入。在移动 fastbin 中多余堆块到 tcache 时，检查堆块的大小是否和目标
tcache 桶要求的大小一致。

影响：技巧 *fastbin reverse into tcache* 不再可用。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=d10176c0ffeadbc0bcd443741f53ebd85e70db44

```diff
@@ -4005,6 +4005,9 @@ _int_malloc (mstate av, size_t bytes)
                    {
                      if (__glibc_unlikely (misaligned_chunk (tc_victim)))
                        malloc_printerr ("malloc(): unaligned fastbin chunk detected 3");
+                     size_t victim_tc_idx = csize2tidx (chunksize (tc_victim));
+                     if (__glibc_unlikely (tc_idx != victim_tc_idx))
+                       malloc_printerr ("malloc(): chunk size mismatch in fastbin");
                      if (SINGLE_THREAD_P)
                        *fb = REVEAL_PTR (tc_victim->fd);
                      else
```

## 增加 largebin unlink 时的检查

于 `4cf2d869` 引入。在分配大堆块，匹配 large bin 中的堆块时，在脱链前检查下个 size
的堆块的上个 size 的堆块是否符合预期。具体见下面的 patch。

影响：技巧 *large bin attack* 不再可用。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=4cf2d869367e3813c6c8f662915dedb1f3830c53

```diff
@@ -4244,6 +4244,9 @@ _int_malloc (mstate av, size_t bytes)
                       fwd = bck;
                       bck = bck->bk;
 
+                      if (__glibc_unlikely (fwd->fd->bk_nextsize->fd_nextsize != fwd->fd))
+                        malloc_printerr ("malloc(): largebin double linked list corrupted (nextsize)");
+
                       victim->fd_nextsize = fwd->fd;
                       victim->bk_nextsize = fwd->fd->bk_nextsize;
                       fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
```

## 由 `realloc` 释放的堆块不再放入 tcache

于 `cd335350` 引入。当调用 `realloc` 需要释放堆块时，例如申请更大或更小的块，
将不再把需要释放的块放入 tcache 中，始终放进 bins 中。

影响： `realloc` 在释放堆块时可以直接操作 bins，绕过 tcache。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=cd335350021fd0b7ac533c83717ee38832fd9887

```diff
@@ -3608,7 +3605,7 @@ __libc_realloc (void *oldmem, size_t bytes)
          size_t sz = memsize (oldp);
          memcpy (newp, oldmem, sz);
          (void) tag_region (chunk2mem (oldp), sz);
-          _int_free (ar_ptr, oldp, 0);
+          _int_free_chunk (ar_ptr, oldp, chunksize (oldp), 0);
         }
     }
 
@@ -5059,7 +5056,7 @@ _int_realloc (mstate av, mchunkptr oldp, INTERNAL_SIZE_T oldsize,
              (void) tag_region (oldmem, sz);
              newmem = tag_new_usable (newmem);
              memcpy (newmem, oldmem, sz);
-             _int_free (av, oldp, 1);
+             _int_free_chunk (av, oldp, chunksize (oldp), 1);
              check_inuse_chunk (av, newp);
              return newmem;
             }
@@ -5087,7 +5084,7 @@ _int_realloc (mstate av, mchunkptr oldp, INTERNAL_SIZE_T oldsize,
                 (av != &main_arena ? NON_MAIN_ARENA : 0));
       /* Mark remainder as inuse so free() won't complain */
       set_inuse_bit_at_offset (remainder, remainder_size);
-      _int_free (av, remainder, 1);
+      _int_free_chunk (av, remainder, chunksize (remainder), 1);
     }
 
   check_inuse_chunk (av, newp);
```

## 在所有 tcache 桶中检测 double free

于 `eff1f680` 引入。原来的检测方案是根据 key 来判断 chunk 对应的桶是否发生 double
free，更新后 free chunk 时会根据 key 在所有 tcache 桶中检测 double free。例如以下片段：

```c
void hack(void) {
    long *p1 = malloc(0xf8);
    free(p1);
    p1[-1] = 0x51;
    free(p1);
}
```

在之前的版本，上述函数可以通过修改堆块的 size，多次释放堆块，现在将不被允许。

影响：如[这个 PR] 中的利用方式将失效。

[这个 PR]: https://github.com/shellphish/how2heap/pull/234/changes/4bfcf2f7515f4092066a01b5ef2086b6f00fe9a7

```diff
@@ -3226,21 +3226,24 @@ tcache_available (size_t tc_idx)
 /* Verify if the suspicious tcache_entry is double free.
    It's not expected to execute very often, mark it as noinline.  */
 static __attribute__ ((noinline)) void
-tcache_double_free_verify (tcache_entry *e, size_t tc_idx)
+tcache_double_free_verify (tcache_entry *e)
 {
   tcache_entry *tmp;
-  size_t cnt = 0;
-  LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
-  for (tmp = tcache->entries[tc_idx];
-       tmp;
-       tmp = REVEAL_PTR (tmp->next), ++cnt)
+  for (size_t tc_idx = 0; tc_idx < TCACHE_MAX_BINS; ++tc_idx)
     {
-      if (cnt >= mp_.tcache_count)
-       malloc_printerr ("free(): too many chunks detected in tcache");
-      if (__glibc_unlikely (!aligned_OK (tmp)))
-       malloc_printerr ("free(): unaligned chunk detected in tcache 2");
-      if (tmp == e)
-       malloc_printerr ("free(): double free detected in tcache 2");
+      size_t cnt = 0;
+      LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
+      for (tmp = tcache->entries[tc_idx];
+          tmp;
+          tmp = REVEAL_PTR (tmp->next), ++cnt)
+       {
+         if (cnt >= mp_.tcache_count)
+           malloc_printerr ("free(): too many chunks detected in tcache");
+         if (__glibc_unlikely (!aligned_OK (tmp)))
+           malloc_printerr ("free(): unaligned chunk detected in tcache 2");
+         if (tmp == e)
+           malloc_printerr ("free(): double free detected in tcache 2");
+       }
     }
   /* No double free detected - it might be in a tcache of another thread,
      or user data that happens to match the key.  Since we are not sure,
```
## 新增 large tcache

这项改动稍后详细说明。

## 推迟 `tcache_perthread_struct` 结构体初始化

于 `cbfd7988` 引入。原先只要一分配堆块就会初始化 tcache 的结构体，
现在这个结构体的初始化被推迟到第一次把堆块 free 进 tcache 时。

影响：利用堆溢出现在可以修改到 tcache 结构体，在 [how2heap] 中亦有记载。

[how2heap]: https://github.com/shellphish/how2heap/blob/master/glibc_2.42/tcache_metadata_hijacking.c

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=cbfd7988107b27b9ff1d0b57fa2c8f13a932e508

```c
static void tcache_init (void);

static __always_inline void *
tcache_get_align (size_t nb, size_t alignment)
{
  if (nb < mp_.tcache_max_bytes)
    {
      if (__glibc_unlikely (tcache == NULL))
       {
         tcache_init ();
         return NULL;
       }
...
```

# glibc 2.43

## mmap chunk 的大小减小 0x10

于 `6455a1b0` 引入。在之前的 glibc 中，mmap chunk 的大小为 mmap 大小，
但实际上由于切下来给用户的空间需要额外的 0x10 字节来存放元数据，因此造成了 mmap chunk
和常规 chunk 大小的不统一，glibc 需要在遇到 mmap chunk 时做额外处理。现在 mmap chunk
的大小表达方式和常规 chunk 一致。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=614cfd0f8a2820aed54f9745077c7da0e6643bac

## 移除 fastbin

这项改动稍后详细说明。

## tcache 槽位增加到 16

于 `0b9210bd` 引入。在之前的 glibc 中，每个 tcache size 的桶能放 7 个空闲块。
由于现在 fastbin 被移除了，作为代偿，每个 tcache size 的桶能放 **16** 个空闲块。

https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=0b9210bd760b5281f2e9f3e6640368ccb5f4a7ae

# tcache large 堆块

从 glibc 2.42 开始，除了常规的 tcache 块以外，现在 tcache 中能存放 size
不固定的大堆块。正常情况下，tcache 小堆块的范围是 `0x20-0x410`，这些尺寸的堆块会放入
tcache 中。再大的堆块要想放入 tcache，需要调整 `GLIBC_TUNABLES`，或者从 pwn 的层面，
改 `mp_` 结构体。通过设置环境变量 `GLIBC_TUNABLES=malloc.tcache.tcache_max_bytes=131072`
（或更大数字，最大可到 4M），大于 0x410 的堆块就能放进 tcache 中。

## free 时堆块的插入行为

tcache large 每个桶和 tcache small 一样，能放 16 个堆块，tcache small 总共 **64**
个桶，但是 tcache large 只有 **12** 个桶。具体堆块尺寸映射到 tcache 桶索引的公式为
`tidx = 64 + clz(0x400) - clz(size)`；换成 Python 表达式就是
`tidx = 64 - (0x400).bit_length() + size.bit_length()`。

{% callout green fa-arrows-left-right-to-line ::图片演示 %}
例如我们需要释放大小为 0x910 的堆块，那么要存入的 tcache large 的索引是 65。

![tidx](/assets/trueblog/glibc2.42/tidx-demo.png)

由此可以看出，每当堆块大小翻一倍，索引就大致增加 1。总结一下每个索引的范围，
大约是 `[64]->0x420-0x800`， `[65]->0x800-0x1000`， `[66]->0x1000-0x2000`。
以上范围左闭右开，0x800 的堆块实际会放到 65 桶中。
{% endcallout %}

由于堆块大小不同，因此在插入到 freelist 中为了提高效率，需要顺序插入。
从 tcache entry 链表头出发是最小的堆块，在遍历 `te->fd` 过程中堆块大小越来越大。
例如 `te -> 0x420 -> 0x500 -> 0x670 -> NULL` 这样。当然，tcache large
同样会对 `fd` 指针做和常规 tcache 堆块一样的加密操作。

## malloc 时堆块的取出行为

要想取出当然是先找 `tidx`。根据以上描述可以找到要求的 tcache large 桶，
接着就会尝试从桶中取出空闲堆块。如果桶中没有满足的堆块，则走非 tcache 路径，
从 unsorted 等 bin 中直接取，或者从 top chunk 划一块下来。

在 glibc 2.42，在定位到桶之后，会开始遍历每个堆块，
只要任意一个堆块大于等于所需要的尺寸，就会返回这个堆块。在 glibc 2.43，
这个行为被 [`b2b4b46a`] 改成了必须满足堆块尺寸和所需要尺寸一致才返回。

[`b2b4b46a`]: https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b2b4b46a5235d83eea6d52b44e8c18be7c65f0d9

## 对堆攻击造成的影响

由于 tcache large 的出现，原来的 [*tcache relative write*] 技巧的利用难度大大提高了。
原来的 `mp_.tcache_bins` 现在变成了 `mp_.tcache_small_bins`，正常 `free`
是碰不到这个变量的，会判断 `mp_.tcache_max_bytes`，不看 `mp_.tcache_small_bins`。

[*tcache relative write*]: https://github.com/shellphish/how2heap/blob/master/glibc_2.41/tcache_relative_write.c

只有一种情况，在 `malloc` 过了 tcache path，要求的 size 大于 small bin 范围，
**扫描 unsorted bin 时**， **发现一块堆块和要求的尺寸完全匹配**，
**并且对应过去的 `tcache->num_slots[tidx] > 0`**，才会把后续以及当前的堆块放入 tcache 中，
此时才有机会造成 relative write。并且这个行为已在 `ea4c36c3` 中被移除，
将在 glibc 2.44 中生效。

就算不打 `mp_.tcache_small_bins`，打 `mp_.tcache_max_bytes`，由于新算法对 tcache
large 的索引计算是对数级的，因此就算把堆块 size 调得非常大，超过 4 MiB 后还得翻几倍，
也没法离开 tcache 元数据堆块很远。

## pwndbg 解析 tcache large 堆块

目前 pwndbg 已基本适配 glibc 2.42-43 的堆，tcache large 的识别工作[正在编写中]，
如有需要可以试用 PR 的分支。

[正在编写中]: https://github.com/pwndbg/pwndbg/pull/3854

<img src="/assets/trueblog/glibc2.42/pwndbg-demo.png" width="60%">

# fastbin 被移除

从 glibc 2.43 开始，fastbin 由于和 tcache 定位类似，因此被移除了，
所有有关 fastbin 的技巧，如 *fastbin reverse into tcache* 等，
都将不可用。

未来小堆块释放溢出 tcache 将直接放入 small bin 中。

# 省流

Ubuntu 26.04 将于今年 4 月发布，据观察，目前其 glibc 版本已升级至 2.43。
总结一下，未来 CTF 中很大概率不会出现 glibc 2.42 这样的过渡版本，新一点就会使用
glibc 2.43。

fastbin 已被移除，未来就没法使用相关的技巧了。除此之外，tcache large
的加入，会对 tcache relative write 产生影响，同时 largebin attack 被移除也导致改
`mp_` 结构体的收益变得很低。tcache 新增的所有 size 的 double free
检测现在强制要求修改堆块的 `key` 才有机会绕过，改变堆块 size 的 trick 不再生效。
tcache 的槽位增加到 16 也使预填充变得更加麻烦。

唯一可能引入的新攻击面是 `tcache_perthread_struct` 的 lazy load，
使用堆溢出原语有机会能操控 tcache 的分配方式，剩下的改动都提高了堆利用的难度。

# 参考

1. [Glibc 高版本堆利用方法总结](https://roderickchan.github.io/zh-cn/2023-03-01-analysis-of-glibc-heap-exploitation-in-high-version/)
2. [commitdiff of e2436d6f](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=e2436d6f5aa47ce8da80c2ba0f59dfb9ffde08f3)
3. [commitdiff of 226e3b0a](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=226e3b0a413673c0d6691a0ae6dd001fe05d21cd)
4. [commitdiff of d10176c0](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=d10176c0ffeadbc0bcd443741f53ebd85e70db44)
5. [commitdiff of 4cf2d869](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=4cf2d869367e3813c6c8f662915dedb1f3830c53)
6. [commitdiff of cd335350](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=cd335350021fd0b7ac533c83717ee38832fd9887)
7. [Add tcache dup (with off-by-one or other ability) technique - GitHub](https://github.com/shellphish/how2heap/pull/234/changes/4bfcf2f7515f4092066a01b5ef2086b6f00fe9a7)
8. [how2heap/glibc_2.42/tcache_metadata_hijacking.c - GitHub](https://github.com/shellphish/how2heap/blob/master/glibc_2.42/tcache_metadata_hijacking.c)
9. [commitdiff of cbfd7988](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=cbfd7988107b27b9ff1d0b57fa2c8f13a932e508)
10. [commitdiff of 614cfd0f](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=614cfd0f8a2820aed54f9745077c7da0e6643bac)
11. [commitdiff of 0b9210bd](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=0b9210bd760b5281f2e9f3e6640368ccb5f4a7ae)
12. [commitdiff of b2b4b46a](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b2b4b46a5235d83eea6d52b44e8c18be7c65f0d9)
13. [how2heap/glibc_2.41/tcache_relative_write.c - GitHub](https://github.com/shellphish/how2heap/blob/master/glibc_2.41/tcache_relative_write.c)
14. [Add support for tcache large chunks since glibc 2.42 - GitHub](https://github.com/pwndbg/pwndbg/pull/3854)
