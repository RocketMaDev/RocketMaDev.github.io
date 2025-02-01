---
title: 古老的基础设施：使用邮件提交补丁
date: 2024/11/23 09:22:00
updated: 2024/11/23 23:10:00
tags:
    - non-ctf
    - debuginfod
thumbnail: /assets/trueblog/debuginfod.png
excerpt: 最近Arch上的debuginfod更新了，也带来了bug...还好只是shell脚本的错误，还在我的能力范畴之内。花了一段时间如何通过邮件向上游提交补丁，最终解决了这个bug。
---

<!-- 在网鼎杯上，手机被收了，离比赛开始还有好久，无聊到写博客了 -->

## 修复

最近Arch上的debuginfod更新了，也带来了bug...在启动shell时会显示无匹配错误：
`/etc/profile.d/debuginfod.sh:14: no matches found: /etc/debuginfod/*.certpath`，
于是我就找这个文件看

```sh
# $HOME/.profile* or similar files may first set $DEBUGINFOD_URLS.
# If $DEBUGINFOD_URLS is not set there, we set it from system *.url files.
# $HOME/.*rc or similar files may then amend $DEBUGINFOD_URLS.
# See also [man debuginfod-client-config] for other environment variables
# such as $DEBUGINFOD_MAXSIZE, $DEBUGINFOD_MAXTIME, $DEBUGINFOD_PROGRESS.

prefix="/usr"
if [ -z "$DEBUGINFOD_URLS" ]; then
    DEBUGINFOD_URLS=$(cat /dev/null "/etc/debuginfod"/*.urls 2>/dev/null | tr '\n' ' ' || :)
    [ -n "$DEBUGINFOD_URLS" ] && export DEBUGINFOD_URLS || unset DEBUGINFOD_URLS
fi

if [ -z "$DEBUGINFOD_IMA_CERT_PATH" ]; then
    DEBUGINFOD_IMA_CERT_PATH=$(cat /dev/null "/etc/debuginfod"/*.certpath 2>/dev/null | tr '\n' ':' || :)
    [ -n "$DEBUGINFOD_IMA_CERT_PATH" ] && export DEBUGINFOD_IMA_CERT_PATH || unset DEBUGINFOD_IMA_CERT_PATH
fi
unset prefix
```

{% notel blue fa-book bash与zsh对于glob的处理 %}
对于bash来说当匹配不到字符串时，会保留原字符串，例如`*.nothing -> '*.nothing'`，
而这个字符串接着传到`cat`中，由于不存在这个文件，会输出错误，紧接着错误被丢弃，
因此对于bash来说，这段代码能够正常运行。

然而，zsh视无法匹配的glob为错误，并终止当前命令，因此会打印出zsh的错误，
这里不是cat的错误。
{% endnotel %}

一开始想着这不是加个shebang的事？加上以后发现无济于事。`profile.d`...？
经过查找，这个文件是被`source`加载的，并不是直接运行，因此只能是由当前shell运行脚本。
继续调查，`/etc/debuginfod/`下没有其他文件夹，因此可以使用`find`来匹配文件并运行。
把`cat /dev/null "/etc/debuginfod"/*.urls 2>/dev/null | tr '\n' ' ' || :`修改为：
`find "/etc/debuginfod" -name "*.urls" -print0 2>/dev/null | xargs -0 cat 2>/dev/null | tr '\n' ' ' || :`

{% note blue fa-info %}
`|| :`中的`:`不做任何事。`find`缺省使用空格分隔文件名，`-print0`可以让`xargs`分清带有空格的文件名。
{% endnote %}

## PATCH IT

知道怎么修之后，就可以向上游发送补丁了。但是 **elfutils** 属于一个
*project run by Unix bros who want the 90s back*，他们不使用GitHub或是Gitlab等现代化设施接受PR，
而是使用`git send-email`，通过邮件的方式发送补丁。类似的，Linux同样使用这个方式来接受补丁。

为了发送补丁，首先需要拉取上游：`git clone git://sourceware.org/git/elfutils.git`，
然后对文件做出修改后commit。再用`git format-patch origin/main`生成一个邮件patch，
最后使用`git send-email 0001-${COMMIT}.patch`发送邮件。

还可以提前配置发送对象，这样就不用每次都在发送时输入一遍。

```ini .git/config
[sendemail]
    to = elfutils-devel@sourceware.org
    suppresscc = self # prevent cc to myself
    confirm = always
```

## SEND IT

一切准备就绪，接下去就是发送邮件了。我平时使用outlook邮箱，但是微软不再允许使用简单认证登录，
换言之，不能使用密码或是应用密码登录，必须使用OAuth2的方式登录。然而，
git并没有内建OAuth2支持。因此，无法使用outlook邮箱。奇怪的是，用gmail同样不行。
看起来git自带的send-email写的一坨，可用性存在问题。于是我看到[可以用msmtp](https://jade.fyi/blog/oh-no-git-send-email/)。

{% note purple fa-circle-arrow-right %}
要调试git-send-email需要使用参数`--smtp-debug=1`，只有`--smtp-debug`不起效
{% endnote %}

我看到博客里一般用gmail，因为gmail的应用密码没有微软那样的校验，可以直接登录，
于是配置好msmtp后，再把git配置一下就可以发送邮件了。

```properties .msmtprc
account default
host smtp.gmail.com
port 587
auth on
tls on
tls_starttls on
logfile ~/.msmtp.log
```

```ini .git/config
[sendemail]
    smtpserver = /usr/bin/msmtp
    smtpserveroption = -a
    smtpserveroption = default
```

{% note green fa-info %}
可以在gmail网页端中找到应用密码的申请入口
{% endnote %}

## ACCEPTED

过了几天后，我的patch顺利被合入到主线了。

![accepted](/assets/trueblog/patchAccepted.png)

又过了几天，我看到debuginfod的包更新了，于是我在arch包的gitlab帖子中看到有人反映这个bug，
然后打包者发现了我的patch，做了一个cherry-pick，把bug修了。
在[帖子](https://gitlab.archlinux.org/archlinux/packaging/packages/elfutils/-/issues/2)里我还看到了fedora的bugzilla引用，
感觉这个patch帮助了不少人呢。

![update](/assets/trueblog/pkgupdate.png)

<img src="/assets/trueblog/cherry-pick.png" height="80%" width="80%">

## outlook: more than OAuth2

原本我还是想使用outlook邮箱的，因为我看到了`oama`等工具能使用OAuth2认证。
然而，看了一系列类似的工具，无不需要`client_id`和`client_secret`这样的客户端认证信息
（是认证客户端，不是认证用户）。但这个认证信息，需要自己到微软企业端申请，
虽然账户可以免费注册，但是需要登记一张visa信用卡，之后才能申请邮箱客户端认证。
然后我并没有visa信用卡，只好作罢。

这些工具还说可以找开源客户端的认证信息，但是我并没有找到...在自己的电脑上装了KMail，
也成功发出了邮件，看来KMail是配置了认证信息，不过我并不知道从哪里获取它的认证信息，
并且也KMail不能作为邮件代理，用来发送git的邮件。

## 参考

1. [[PATCH] config: fix globing error for zsh](https://sourceware.org/pipermail/elfutils-devel/2024q4/007580.html)
2. [/etc/profile.d/debuginfod.sh causes spurious output on interactive login for non-bash shells](https://gitlab.archlinux.org/archlinux/packaging/packages/elfutils/-/issues/2)
3. [Bug 32314 - Profile script in elfutils-debuginfod-client throws error on login](https://sourceware.org/bugzilla/show_bug.cgi?id=32314)
4. [Bug 2321818 - Profile script in elfutils-debuginfod-client throws error on login](https://bugzilla.redhat.com/show_bug.cgi?id=2321818)
