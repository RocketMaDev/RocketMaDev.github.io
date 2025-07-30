---
title: 永远在“服务器故障”的gpg与解决方案
date: 2025/07/30 23:48:00
updated: 2025/07/31 01:32:00
tags:
    - non-ctf
excerpt: >-
    自从配好了 `*ray` 的 DNS 接管并自动分流后，我在使用 gpg 接收公钥的时候就一直显示
    “gpg: 从公钥服务器接收失败：服务器故障”。这令我非常头疼，明明 keyserver
    的网页都可以正常打开，但是没法使用 gpg 下载公钥。关闭 `*ray` 后，有时候可以，有时候又不行。
    背后的原因到底是什么？ hkp 和 http 协议导致发送的 DNS 请求又有什么区别？
    这次，我终于抓住了破解了其背后的秘密...
thumbnail: /assets/trueblog/strace_gpg.png
---

{% note yellow fa-exclamation %}
本文无不良引导，仅讨论网络工具的使用，均在合法合规的范围内使用
{% endnote %}

{% note purple fa-bullseye %}
本解决方案针对使用`*ray`代理工具并使用内置DNS的用户！
{% endnote %}

## 问题表象

自从我配置好`*ray`的dns模块后（DNS分流），没出过任何问题，除了每次yay的时候，
一旦导入公钥，就会显示 *“gpg: 从公钥服务器接收失败：服务器故障”* ，
即使我指定了keyserver是keyserver.ubuntu.com，并且在浏览器中确认可以访问后也不行。
这令我非常苦恼，每次导入公钥都需要去keyserver的网页上下载公钥并手动导入。
如果关掉`*ray`，那么有时候可以，有时候又不行，取决于所处的网络。

## 追根溯源

回想起上次调试[ipv6](/2025/02/15/pendingIpv6/)的时候，就使用了`strace`检查请求的成功情况，
这次也可以效仿。当keyserver以 **hkp(s)** 协议访问时，就会显示服务器故障，如果使用 **http(s)**
协议访问，则显示不包含有效公钥。很显然，使用http可以保证正常访问，只是没有公钥罢了。
那么两者有什么不同呢？`strace`结果似乎看不出啥，请求不是由gpg直接发的，而是转发到了一个
socket。

![strace](/assets/trueblog/strace_gpg.png)

那么谁持有这个socket呢？一开始以为是`gpg-agent`，结果`strace`挂上去并没有，用`lsof`
看了一下，是`dirmngr`，gpg套件中的一部分。那我们接着跟踪一下它：

![strace](/assets/trueblog/strace_dirmngr.png)

可以看到，当启动gpg请求后，`dirmngr`会解析`/etc/resolv.conf`，并向其中的nameserver
发送DNS请求，如果是`hkp`协议，那么得不到回复，但是`http`协议就能正常得到回复。

通过wireshark抓包也可以发现，使用hkp协议的DNS包没有回应，并且类型是我从来没见过的
**SRV**。

![wireshark](/assets/trueblog/wireshark.png)

## 水落石出

不难发现，是由于DNS请求无回应，导致gpg显示服务器故障，而DNS请求是被`*ray`全部接管的，
那么`*ray`是不是在DNS实现上有什么问题呢？查找官方文档，果然，均写着：

> 只支持最基本的 IP 查询（A 和 AAAA 记录），CNAME 记录将会重复查询直至返回
> A/AAAA 记录为止。其他查询不会进入内置 DNS 服务器。

针对gpg要求的 **SRV** 请求，`*ray`会直接丢弃而不处理，这也就是为什么没有应答了。

## 解决方案

最简单的就是暂时停用`*ray`了。下载公钥的情况并不多，只要临时停用，就可以正常处理DNS请求，
当然，为了避免污染，可以设置nameserver为 *1.1.1.1*，写入到`/etc/resolv.conf`中。
下载完公钥后，再启用`*ray`即可。

复杂一点的话，可以使用`SmartDNS`接管dns请求，这样就可以覆盖所有dns请求情况。

## 参考

1. [没有应答的 IPv6 长连接](/2025/02/15/pendingIpv6/)
