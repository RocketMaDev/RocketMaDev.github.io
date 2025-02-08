---
title: 在手机上向根证书区注入用户证书
date: 2025/02/06 22:35:00
updated: 2025/02/06 22:35:00
tags:
    - non-ctf
    - KernelSU
thumbnail: /assets/trueblog/rootca.jpg
excerpt: 在手机上抓包，可以使用`PCAPdroid`，但是为了解包https流量，需要做中间人攻击，安装用户证书，并在通知栏中显示“网络流量可能受到监控”。为了把这些字样去除，可以使用KSU模块将准备好的用户证书复制到根证书区，随后就可以删除用户证书了。然而，模块却引入了新的问题...
---

{% note blue fa-newspaper %}
在高考结束后，买了小米13，没想到成了最后一部方便解锁的小米手机。看看现在的解锁政策，
根本不是人能成功的。
{% endnote %}

既然买了小米13，自然是要狠狠地刷机！刷上KernelSU之后，自然需要找些用武之地。LSPosed
自不必说，root还可以做一些影响系统根目录的操作，比如写入根证书。为了在手机上抓包，
我们可以使用PCAPdroid，然而，为了做中间人攻击解密https流量，还需要安装证书。
安装证书后，下滑系统通知就会显示网络流量可能受到监控，这多少让强迫症看了有点难受。
到网上搜索相关模块，都是将用户证书复制到根证书，那么就会出现如下问题：

安装模块后，为了使用户证书复制到根证书，需要安装用户证书然后重启；此时将用户证书删除，
通知栏里的警告就会消失；然而，只要一重启手机，由于用户证书已经被删除了，
模块便不会复制证书，导致无法解密。这看起来像是个死循环，无解了。

于是我打算自己写模块，将自己的证书放到模块中，然后在系统启动时加载到根证书中就可以了。

## Coding time

参照[KernelSU的模块开发指南](https://kernelsu.org/zh_CN/guide/module.html)，
加上[之前的方案](https://github.com/NVISOsecurity/MagiskTrustUserCerts)做参考，
我开始自己写模块。主要代码如下：

```sh post-fs-data.sh
#!/system/bin/sh
MODDIR=${0%/*}

# should change all file time to a single one!
cadir=$MODDIR/cacerts
mkdir -p $cadir
rm $cadir/*
cp -f $MODDIR/*.0 $cadir
cp /system/etc/security/cacerts/* $cadir
set_perm_recursive $cadir root root 755 644 u:object_r:system_security_cacerts_file:s0
mount --bind $cadir /system/etc/security/cacerts
```

## 一些其他的尝试

我尝试过只复制单个证书；只复制证书后修改所有文件权限；使用`service`执行，
都不能在系统中找到证书，即便在终端中直接列出文件夹的内容中是有的。因此以上已是最简操作，
不能更短了。看来想让系统承认我们这个“外来”的证书，还是比较复杂的。

但是这个方案同样不是完美的，因为由于系统根证书目录被重新挂载了，里面所有的文件时间戳都会变成
0，这也为后面埋下伏笔。

## 损坏的B站

在正常工作了几周后，b站首页突然无法更新了，反倒是动态还能正常工作。其他有些功能能工作，
有些则不行，非常奇怪，看起来也不像是没网了。我试着抓了一下包，发现显示客户端不信任证书。
好嘛，我加的证书被b站认出来了。果然，我只要卸载模块，b站就恢复正常了。

就在我一筹莫展时，我留意到我每天在b站上花的时间实在是太多了。

> 为什么不开启模块，以此来削减我使用b站的时间呢？卸载b站然后再装回来所花的成本太高了，
> 毕竟数据会全部丢失；如果只是在scene里冻结b站，解冻的成本又太低，不足以阻拦我使用b站。

于是，这个方案就歪打正着地成为了我限制自己b站使用的方案。重启是一个成本不高不低，
在应急时方便操作，在平时又嫌麻烦的操作。一开始，只能看动态中up主更新的视频其实挺合适，
可是随着时间推移，所有的功能都变得不可用了，整个b站就废掉了。自此，我也没怎么拿手机看b
站了，虽然手机上的b站形同虚设，但至少目的达到了。

## 参考

1. [模块开发指南](https://kernelsu.org/zh_CN/guide/module.html)
2. [Magisk Trust User Certs](https://github.com/NVISOsecurity/MagiskTrustUserCerts)
