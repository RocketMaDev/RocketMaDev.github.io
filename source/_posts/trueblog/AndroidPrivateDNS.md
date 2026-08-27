---
title: Android 透明代理与专用 DNS 之争
date: 2026/08/27 23:28:00
updated: 2026/08/28 00:46:00
tags:
    - Android
    - proxy
thumbnail: /assets/trueblog/asterisks.png
excerpt: |-
    我用了 AsteriskNG 作为手机上的代理，并开启了 tproxy 透明代理模式，但是诡异的是，
    流量下一切正常，wifi 下 tg 能正常工作，但是浏览器无法访问 google。这怎么可能呢！
    等一下，`curl -v` 拿到的解析结果怎么是污染过的 IP...
---

# 搞技术的就应该使用透明代理

既然我手机已经 root 过了，那么自然要品鉴一下需要 root 的模块了。代理作为技术人基础工具，
常驻后台是常见的需求。已知安卓的 VPN Service 走的是 TUN 虚拟设备，并且还需要在状态栏悬挂一个告示，
那有没有办法能比传统的 VPN Service 方案更好呢？有的，兄弟，有的，原来有一个 *Asterisk4Magisk*
模块，在后台通过 tproxy 的方式运行 xray core，既有效率（tproxy 效率比虚拟设备高），又没告示。

后来模块不更了，被原作者的 *AsteriskNG* 替代，直接写了一个 APP 出来，在 APP 中用 root
来接管网络操作，使用体验比原来 KernelSU Web UI 好多了，果断跟进。

{% callout green fa-star %}
作者共开发了三款应用，xray 核对应 *AsteriskNG*，mihomo 核对应 *AsteriskMETA*，sing-box 核对应
*AsteriskBOX*，均可用 root 权限接管网络实现透明代理，欢迎尝试。**不是广告。**
{% endcallout %}

# 在家里代理怎么炸了

其他时候，代理都正常工作，分流、透明代理都正常，但是唯独在我家里时，无法在浏览器访问 Google，
也无法加载 Discord 应用里的信息。一切换到流量又好了，看着十分诡异。既然目前 AI 这么强大，
就想着让 AI 大人调调看。

# 帮帮我，Codex 大人

自 DeepSeek v4 flash 可以使用 responses 接口后，我就尝试把它塞到 Codex 里用，效果还不错。
于是就问它，调试的步骤有哪些。

第一轮，它怀疑 tproxy、UDP、DNS 有问题。

{% callout red fa-xmark %}
- tproxy: 切换到 TUN2SOCKS 后并没有解决问题。
- UDP: 怀疑 Discord 使用 UDP 通信，浏览器使用了 QUIC 在传输数据，但是检查后发现并不是 UDP 的问题。
- DNS: nslookup 均返回正常结果。
{% endcallout %}

第二轮，它怀疑是 tproxy 模式本身或者 IPv6 的问题。

{% callout red fa-xmark %}
- VPN Service: 虽然 tproxy 和 TUN2SOCKS 均不正常工作，但是如果走安卓自己的 VPN Service
  就正常工作。同时在 curl 时指定代理，不使用透明代理也正常工作。
- IPv6: 根本没开。设置都是全关的。
{% endcallout %}

透明代理不行，传统的 VPN Service 却可以？第三轮，我把完整的 xray 日志给它，它并没有发现问题。
于是它要求用 curl 连接 google 等网站测试透明代理，同时开启 tcpdump 抓包。

{% callout yellow fa-question %}
curl 连接全都超时了，如果走 socks，能正常连接，但是走 tproxy 连接就会超时。
{% endcallout %}

这一轮它敏锐地意识到，走 socks 时，发往代理服务器的是域名，而走 tproxy 就必然先解析域名，
直接拿 IP 访问，因此需要调整代理的 `routeOnly` 选项。

{% callout green fa-check %}
把 `routeOnly` 关闭后，真的解决了问题。工作原理是 xray 核会把嗅探到的域名覆盖 IP，
再发回代理，由代理解析域名，不由本地应用解析域名。
{% endcallout %}

问题是解决了，但背后的原因是什么呢？这仍然无法解释为什么在流量下就能正常工作。

# 你这吃白饭的大肥鱼，给我继续想

新的一轮，大肥鱼给出了猜测：万一是流量情况下刚好解析到了正确的 IP 呢？
这当然是不可能的，防火墙能不拦？大肥鱼又要求重跑 `curl -v`，看解析的结果。
在流量下，DNS 能被正确解析，但是在 wifi 下，国外域名解析结果拿到的却是防火墙污染过的
IP。 但是 nslookup 测试正常啊，为什么 curl 拿到的 IP 和 nslookup 不一样呢？

下一轮，要求观察 nslookup 请求有没有进核心，再挂 tcpdump 检查 DNS 请求。

{% callout yellow fa-question %}
- nslookup: 无论是在流量条件还是 wifi 条件，查阅日志发现核心都正确处理了 DNS 请求。
- tcpdump: 使用 curl 时，流量下能正常看到 DNS 请求，但是 wifi 下一条请求都没有？
{% endcallout %}

在安卓上，调用 `getaddrinfo` 等标准函数时，会由 Bionic libc 建立 UNIX socket 
连接到系统的 *dnsproxyd*，由它代理解析 DNS 请求，而 nslookup 是直接发包的，
两者机制并不一致。大肥鱼推测是系统 DNS 缓存的问题，这当然不是原因，
就算忘记网络重连也不行。此时它让我执行 `dumpsys connectivity`，想看系统 DNS
情况，但是可惜由于忘记让我 `sudo` 了， 普通用户并不能访问这个命令，错过了真实原因。

于是，我让它查源码找原因，它定位到 DNS 缓存是根据网络 ID 来的，只要网络 ID 不刷新，
DNS 缓存就保持，因此看不到 DNS 请求出现在 tcpdump 里。这个猜想当然也是错误的，
即使忘记网络再重连，仍然看不到 DNS 流量，并且能从 `ip rule` 中确认网络 ID 确实变了。

又经过了几轮的鏖战，我从 DNS 抓包中捕获了一些电信专有的域名，这说明 wifi
下抓包是正常工作的；并确认重连网络后 **应该** 恢复 DNS 为默认的 `192.168.1.1` 了。
大肥鱼要求监听 53、853、443 三个端口，尝试获得更多信息。

{% callout green fa-check %}
这次抓包过程中，wifi 条件下，终于抓到 DNS 流量了，但是发现，并不是在 53 端口，
而是在 853 端口——也就是说，wifi 下一直在发送 DoT 查询。？！
{% endcallout %}

# 谜底揭晓

虽然有好几次都与正确答案擦肩而过，但是正确答案最终还是揭晓了。使用 `sudo dumpsys connectivity`
查看系统 DNS，发现在 wifi 条件下，DNS 是 `192.168.1.1, 223.5.5.5`，并且后者是“PrivateDNS”，
被优先使用了。

这个问题由以下三点组成，环环相扣，缺一不可：

1. 我在设置路由器时，由于担心 DNS 劫持，在配置 DHCP 时，主 DNS 设为了 `192.168.1.1`，
   副 DNS 设为了 `223.5.5.5`；
2. 新版本安卓在默认情况下，会测试 DNS 的连通性，并优先使用支持 DoT 的 DNS，不使用只支持明文的
   DNS 以提高隐私性；
3. 透明代理捕获 DNS 请求时只能捕获明文 DNS 并劫持，对加密的 DNS 请求毫无办法。

最后我把路由器里的副 DNS 删掉了，并且关闭了“PrivateDNS”优先功能，终于能在家里正常访问互联网了。

{% callout blue fa-circle-info %}
小米手机要关闭这个功能，路径是：打开 **设置**，点击 **更多连接**，选中 **专用 DNS** 后设为 **关闭**。
{% endcallout %}
