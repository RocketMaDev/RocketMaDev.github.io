---
title: 任意访问 Android data 的研究
date: 2026/03/25 17:42:00
updated: 2026/01/13 21:02:00
tags:
    - Android
    - kernel
thumbnail: /assets/trueblog/arbdata/message.png
---

# 一条帖子

前些天 *int* 给我发了一条消息，里面有个简短的poc，可以直接进入受限的路径。

<img src="/assets/trueblog/arbdata/message.png" width="60%">

我那小米13是Android 13，因此正常访问过去的话应该是受限的。

<img src="/assets/trueblog/arbdata/open-fail.jpg" width="60%">

然而使用`cd /storage/emulated/0/Android/$'\u200d'data`后可以直接进入受限的目录，
并列出目录，甚至能进入别的应用的目录并查看文件，这就很危险了。
*int* 在使用Pixel 9，能接收到最新的Android安全更新，然而并没有修复这个漏洞，
这令我非常好奇，决定探索一番。

<img src="/assets/trueblog/arbdata/open-success.jpg" width="60%">

于是我跟着给的帖子看了一眼，是一个[24年的漏洞]，但是国内厂商并不一定会修复。
帖子里提到的[CVE-2024-43093]我找到[google修复的代码]了，但是 *int* 那里仍然能复现，
这是怎么回事？

[24年的漏洞]: https://x.com/Timfurry233/status/2009249955307749443
[CVE-2024-43093]: https://nvd.nist.gov/vuln/detail/CVE-2024-43093
[google修复的代码]: https://android.googlesource.com/platform/frameworks/base/+/7f83c671626f9bf993581f4598c22482d87cba10

# 漏洞严重性与修复

如果你并不想知道漏洞原理的话，只看这个章节就够了，毕竟整个攻击的原理还是非常复杂的...

## 严重性

攻击者可以访问任意`/storage/emulated/0/Android/data`下的目录，这里包括了各种应用的临时缓存，
可以用来探测安装了哪些应用，以及提取其他应用的缓存数据，如微信等。

漏洞利用条件是安装一个应用，不需要给予任何权限。

## 官方修复方案

被评为 `Won't fix (infeasible)`，不会修复。

## 修复

**所有的修复方案都需要root权限！** 需要按照以上帖子里给出的方案，选择一个。
最优方案是用Xposed模块打一个补丁，影响最小。需要[从Telegram下载]这个应用。

[从Telegram下载]: https://t.me/real5ec1cff/271

# 详细分析

## 不完整的修复

要想知道这个漏洞是怎么发生的，可以先看[上次的修复]：

```diff ExternalStorageProvider.java
@@ -16,8 +16,6 @@
 
 package com.android.externalstorage;
 
-import static java.util.regex.Pattern.CASE_INSENSITIVE;
-
 import android.annotation.NonNull;
 import android.annotation.Nullable;
 import android.app.usage.StorageStatsManager;
@@ -61,12 +59,15 @@
 import java.io.FileNotFoundException;
 import java.io.IOException;
 import java.io.PrintWriter;
+import java.nio.file.Files;
+import java.nio.file.Paths;
+import java.util.Arrays;
 import java.util.Collections;
 import java.util.List;
 import java.util.Locale;
 import java.util.Objects;
 import java.util.UUID;
-import java.util.regex.Pattern;
+import java.util.stream.Collectors;
 
 /**
  * Presents content of the shared (a.k.a. "external") storage.
@@ -89,12 +90,9 @@
     private static final Uri BASE_URI =
             new Uri.Builder().scheme(ContentResolver.SCHEME_CONTENT).authority(AUTHORITY).build();
 
-    /**
-     * Regex for detecting {@code /Android/data/}, {@code /Android/obb/} and
-     * {@code /Android/sandbox/} along with all their subdirectories and content.
-     */
-    private static final Pattern PATTERN_RESTRICTED_ANDROID_SUBTREES =
-            Pattern.compile("^Android/(?:data|obb|sandbox)(?:/.+)?", CASE_INSENSITIVE);
+    private static final String PRIMARY_EMULATED_STORAGE_PATH = "/storage/emulated/";
+
+    private static final String STORAGE_PATH = "/storage/";
 
     private static final String[] DEFAULT_ROOT_PROJECTION = new String[] {
             Root.COLUMN_ROOT_ID, Root.COLUMN_FLAGS, Root.COLUMN_ICON, Root.COLUMN_TITLE,
@@ -309,11 +307,70 @@
             return false;
         }
 
-        final String path = getPathFromDocId(documentId);
-        return PATTERN_RESTRICTED_ANDROID_SUBTREES.matcher(path).matches();
+        try {
+            final RootInfo root = getRootFromDocId(documentId);
+            final String canonicalPath = getPathFromDocId(documentId);
+            return isRestrictedPath(root.rootId, canonicalPath);
+        } catch (Exception e) {
+            return true;
+        }
     }
 
     /**
+     * Based on the given root id and path, we restrict path access if file is Android/data or
+     * Android/obb or Android/sandbox or one of their subdirectories.
+     *
+     * @param canonicalPath of the file
+     * @return true if path is restricted
+     */
+    private boolean isRestrictedPath(String rootId, String canonicalPath) {
+        if (rootId == null || canonicalPath == null) {
+            return true;
+        }
+
+        final String rootPath;
+        if (rootId.equalsIgnoreCase(ROOT_ID_PRIMARY_EMULATED)) {
+            // Creates "/storage/emulated/<user-id>"
+            rootPath = PRIMARY_EMULATED_STORAGE_PATH + UserHandle.myUserId();
+        } else {
+            // Creates "/storage/<volume-uuid>"
+            rootPath = STORAGE_PATH + rootId;
+        }
+        List<java.nio.file.Path> restrictedPathList = Arrays.asList(
+                Paths.get(rootPath, "Android", "data"),
+                Paths.get(rootPath, "Android", "obb"),
+                Paths.get(rootPath, "Android", "sandbox"));
+        // We need to identify restricted parent paths which actually exist on the device
+        List<java.nio.file.Path> validRestrictedPathsToCheck = restrictedPathList.stream().filter(
+                Files::exists).collect(Collectors.toList());
+
+        boolean isRestricted = false;
+        java.nio.file.Path filePathToCheck = Paths.get(rootPath, canonicalPath);
+        try {
+            while (filePathToCheck != null) {
+                for (java.nio.file.Path restrictedPath : validRestrictedPathsToCheck) {
+                    if (Files.isSameFile(restrictedPath, filePathToCheck)) {
+                        isRestricted = true;
+                        Log.v(TAG, "Restricting access for path: " + filePathToCheck);
+                        break;
+                    }
+                }
+                if (isRestricted) {
+                    break;
+                }
+
+                filePathToCheck = filePathToCheck.getParent();
+            }
+        } catch (Exception e) {
+            Log.w(TAG, "Error in checking file equality check.", e);
+            isRestricted = true;
+        }
+
+        return isRestricted;
+    }
+
+
+    /**
      * Check that the directory is the root of storage or blocked file from tree.
      * <p>
      * Note, that this is different from hidden documents: blocked documents <b>WILL</b> appear
```

可以看到之前修复这个漏洞时，原先是使用regex判断路径是否匹配`/storage/emulated/0/Android/{data,obb,sandbox}`，
后面换成了成功打开目录后调用底层api，检查打开的文件是否匹配三个中的任意一个，不再使用语义化判断。
底层一般使用inode匹配的方式做检查，所以这一步基本绕不过。

但是这个漏洞理论上应该已经在 *int* 的手机上修复了，怎么还不行？我用pm从他的手机上提取出来，然后放到jadx
里看了一眼，没问题。难道说，漏洞并不在这里？

[上次的修复]: https://android.googlesource.com/platform/frameworks/base/+/7f83c671626f9bf993581f4598c22482d87cba10

## FUSE

我尝试用`ps`匹配`ExternalStorageProvider`的进程，结果并没有，或许，根本就不是这个组件？
使用`mount`查看挂载信息，发现了这几条密切相关的信息：

```plaintext
/dev/fuse on /storage/emulated type fuse (rw,nosuid,nodev,noexec,noatime,lazytime,user_id=0,group_id=0,allow_other)
/dev/block/dm-44 on /storage/emulated/0/Android/data type f2fs (rw,nosuid,nodev,noatime,lazytime,seclabel,background_gc=on,gc_merge,discard,no_heap,user_xattr,inline_xattr,acl,inline_data,inline_dentry,flush_merge,extent_cache,mode=adaptive,active_logs=6,reserve_root=32768,resuid=0,resgid=1065,inlinecrypt,alloc_mode=default,checkpoint_merge,fsync_mode=nobarrier,discard_unit=block,memory=normal)
/dev/block/dm-44 on /storage/emulated/0/Android/obb type f2fs (rw,nosuid,nodev,noatime,lazytime,seclabel,background_gc=on,gc_merge,discard,no_heap,user_xattr,inline_xattr,acl,inline_data,inline_dentry,flush_merge,extent_cache,mode=adaptive,active_logs=6,reserve_root=32768,resuid=0,resgid=1065,inlinecrypt,alloc_mode=default,checkpoint_merge,fsync_mode=nobarrier,discard_unit=block,memory=normal)
```

可以看到要访问的“根”是挂载到一个fuse上的，而我们正在hack的关键路径是挂载到真实的块设备上的。

{% notel blue fa-book FUSE 是什么 %}
**FUSE**: Filesystem in Userspace，用户态文件系统，当一个目录以FUSE的方式挂载时，
对其任何操作都会由内核打包后送往FUSE daemon，有daemon负责处理，并返回结果。
daemon就像中间人，负责决定实际访问的目录、允不允许访问等。
{% endnotel %}

由于termux做的都是底层syscall，而不是弹出文件选择器询问文件，因此完全绕过了
`ExternalStorageProvider`部分的检查。

## 不同应用，不同用户

如果我们查看`/storage/emulated/0`下的目录，可以发现它们的用户和组为`u0_a220`和`media_rw`，
但是`.../Android/data`就不一样，它下面的用户和组都是每个用户对应的uid，如`u0_a351`和`media_rw`。
`u0_a220`到底是何方神圣？稍后揭晓。在我的系统上，termux的用户id是`u0_a311`，可以看到
`.../Android/data/com.termux`的持有者也确实是`u0_a311`，说明在这个app私有的目录下，
每个目录的持有者都是app对应的用户。

{% note green fa-user-tag %}
在Android这套体系中，为了分割每个app的权限，每个app都有一个独立的用户，用`u${USERID}_a${APPID}`
来标识。例如termux是我手机上的第311个应用，同时我没有启用工作空间，那么此时
`USERID=0`，`APPID=311`。
{% endnote %}

如果我们cd到`.../Android/data/com.termux`，那当然一点问题都没有，但是我们又不能cd到
`.../Android/data/org.tasks`这种别的应用目录里。但是当我们利用这个漏洞的时候，
又确实能进入任意的目录中，这怎么可能？！权限管理怎么又失效了？

事已至此，先研究一下FUSE吧。既然有root权限，直接用`lsof`看一下`/dev/fuse`是谁持有的就可以，
于是找到了`com.android.providers.media.module`这个包，查看这个进程的`status`文件，
可以发现它的uid正是`u0_a220`。当然，这还没完。它的附加组里还有1023，即`media_rw`组。
也就是说，这个进程就是FUSE daemon，所有打到`/storage/emulated/0`上的请求，
都会被转发到它这来处理。

## 文件系统fallback

有root权限，我们直接用strace attach到这个进程上看系统调用，这块我测试了一下，
监控`newfstatat`看得比较清楚。

正常情况下，我们在列出`/storage/emulated/0/Pictures`等目录时，是能看到目录下的文件被stat
的：

<img src="/assets/trueblog/arbdata/trace-normal.jpg" width="60%">

如果我们尝试列出`/storage/emulated/0/Android/data`或访问其中的目录时，是看不到相应的请求的，
真的，就是一条请求都没刷新。但是如果我们利用一下漏洞的话，会发现又有大量的请求：

<img src="/assets/trueblog/arbdata/trace-zwc.jpg" width="60%">

由此我们不难得出，termux可以访问外部存储（`/storage/emulated/0` or `/sdcard`）的情况下，
当它在访问app私有目录`.../Android/data/...`时，这一步请求时在实际的f2fs文件系统上处理的，
因此会检查目录的持有者和当前用户是否匹配。然而，如果利用漏洞，加上ZWC字符后，
所有请求会过一遍fuse，fuse层面并不做用户检查（因为本来就是共享目录，检查无意义），
fuse确认完这些目录是安全的后，会“帮我们完成我们的操作”。而且fuse daemon有`media_rw`组，
因此也能任意访问所有`.../Android/data`下面的目录，换言之，我们也有了对于他们的访问权限。

{% note purple fa-circle-question %}
在查找目录过程中，VFS会从根目录不断walk。挂载点都有缓存，在尝试解析一个目录的时候，
会先`lookup_fast`，从缓存里找，挂载点就在里面。如果没匹配到，就会走`lookup_slow`，
从parent出发lookup。不过我也不是很清楚是不是这样的，有讲错的话欢迎指正。
{% endnote %}

## fuse daemon 的检查

等等，fuse怎么没检查出来这是危险目录？检查代码，发现它的错误和之前修复漏洞那一块的错误是一样的。
daemon跳到java中，尝试匹配目录是否是一个app-private的路径，代码是这样的：

```java src/com/android/providers/media/util/FileUtils.java
    public static final Pattern PATTERN_OWNED_PATH = Pattern.compile(
        "(?i)^/storage/[^/]+/(?:[0-9]+/)?"
        + PROP_CROSS_USER_ROOT_PATTERN // ""
        + "Android/(?:data|media|obb)/([^/]+)(/?.*)?");

    public static @Nullable String extractPathOwnerPackageName(@Nullable String path) {
        if (path == null) return null;
        final Matcher m = PATTERN_OWNED_PATH.matcher(path);
        if (m.matches()) {
            return m.group(1);
        }
        return null;
    }
```

这是一条处理`open`的链子（可以从`jni/FuseDaemon.cpp::pf_open`开始跟），
可以由于零宽字符，路径脱离了regex的匹配，被认为不是app-private的路径，
于是就会尝试打开底层文件系统中对应的文件。

## utf8 casefold

说了这么多，似乎有一个事被刻意地忽视了：为什么路径里包含零宽字符，
但是还能从底层文件系统中找到？这其实来源于底层文件系统——f2fs的特性。
终于，我们需要研究帖子中提到的[内核commmit]了。这个commit修复了一个漏洞，
当文件名是`CASEDFOLDED`并且没找到时，会尝试禁用hash再找一次。原先的逻辑是，
假定所有文件名都有哈希，因此只在哈希匹配成功时，才继续匹配完整文件名。
继续查看这个提到的[另一个commit]，里面是有关 *Ignorable code points* 的处理。
由这些信息，可以推断出f2fs的哈希其实是casefold哈希，是跟utf8强相关的，
并不是粗暴地根据字节流直接哈希。跟踪前一个commit的文件名判断函数`f2fs_match_name`，
其实是可以找到里面也是判断utf8 casefold后文件名的比较结果。

{% note green fa-compress %}
**UTF8 casefold** 是内核的utf8字符归一化处理，会将所有英文字符处理成小写，
还会处理一些别的逻辑。最特殊的是它会移除unicode零宽字符。
{% endnote %}

f2fs原先引入casefold的设计逻辑是大小写不敏感匹配文件名，
但是没想到这个特性还顺便把零宽字符也去除了。终于，我们抓住了这个漏洞的本质：

1. 加入零宽字符的路径在内核层面没有改变挂载点，将lookup请求发送到了fuse
2. fuse在判断的时候没有考虑零宽字符，认为其实安全的，向底层f2fs请求文件
3. f2fs对路径名做casefold归一化处理，丢弃了零宽字符，打开了不含零宽字符的文件

由此实现了一个unicode零宽字符，允许攻击者直接访问任何app-private目录。

[内核commit]: https://github.com/torvalds/linux/commit/91b587ba79e1b68bb718d12b0758dbcdab4e9cb7
[另一个commit]: https://github.com/torvalds/linux/commit/5c26d2f1d3f5e4be3e196526bead29ecb139cf91

# 疑问

正当我以为我已经对底层掌握的一清二楚的时候，我想到改变大小写对内核切换挂载点的影响。
访问`/STORAGE/EMULATED/0/Android`，会返回`ENOENT`的错误，但是如果访问
`/storage/emulated/0/ANDROID/DATA`，又能正常匹配到挂载点导致无法列目录。
我对着内核VFS路径lookup的代码看了又看，始终想不明白。

正常情况下，内核是逐字节哈希、匹配的，并且f2fs是casefold的，
因此应该使用简单的大小写就能绕过限制；然而真实情况却是内核仍然匹配到了挂载点。
如果内核使用casefold的方式匹配挂载点，那就能解释为什么改变路径的大小写也能匹配到挂载点，
但是这样的话零宽字符在匹配挂载点的时候就应该被移除掉了，因此漏洞使用的路径，
应该同样能匹配到挂载点。

如果有内核✌️在看的话，希望在评论区解答一下🙏

{% note yellow fa-bug-slash %}
如果你安装termux后，发现漏洞打不通的话，是因为termux的target api很低，
需要手动允许外部文件访问，才能正常cd过去。对于最近的应用来说，是不需要设置权限，
就能直接访问`/storage/emulate/0`的。
{% endnote %}

# 上游 issue tracker

目前[这个issue]应该已经公开，可以直接访问

{% notel purple fa-star 踩的一些坑 %}
Google 也是相当有钱啊，复现一下别人的发现就发了 *250 USD*，发到 bugcrowd 账号。
如果后续有别的师傅要想从 bugcrowd 提取金额，注意 **税表尽量填真实的**，
否则就他们的处理速度，如果没填对就得等好几天；如果税表没填对，会发邮件通知你，
在重写填写后记得要回复邮件更新消息。支付信息尽量填银行转账，事比 paypal 少很多，
如果只能选 paypal 的话，可以用 Xoom 转账，可以直接转到国内的支付宝账号里，
手续费也比较少。
{% endnotel %}

[这个issue]: https://issuetracker.google.com/issues/475242046

# 时间线

- 1 月 9 日: 初次见到漏洞，根据帖子复现问题
- 1 月 13 日: writeup 编写完毕，并报送 Google
- 2 月 21 日: Google 判定完毕，认为bug不需要修复
- 3 月 25 日: writeup 公开

# 参考

1. [“立刻检查你的安卓手机是否还存在零宽字符扫描漏洞！”](https://x.com/Timfurry233/status/2009249955307749443)
2. [NVD - CVE-2024-43093 Detail](https://nvd.nist.gov/vuln/detail/CVE-2024-43093)
3. [Restrict access to directories - googlesource](https://android.googlesource.com/platform/frameworks/base/+/7f83c671626f9bf993581f4598c22482d87cba10)
4. [@5ec1cff message](https://t.me/real5ec1cff/271)
5. [f2fs: Introduce linear search for dentries](https://github.com/torvalds/linux/commit/91b587ba79e1b68bb718d12b0758dbcdab4e9cb7)
6. [unicode: Don't special case ignorable code points](https://github.com/torvalds/linux/commit/5c26d2f1d3f5e4be3e196526bead29ecb139cf91)
7. [Access /storage/emulated/0/Android/data arbitrarily via native syscall](https://issuetracker.google.com/issues/475242046)
