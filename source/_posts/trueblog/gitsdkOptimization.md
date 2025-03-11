---
title: 优化一些 msys2 上 git-sdk 冗余的东西
date: 2025/03/11 19:57:00
updated: 2025/03/11 19:57:00
tags:
    - non-ctf
    - Windows
excerpt: 通过编写`rm-git-sdk`函数删除`git-extra`包中引发错误的profile，解决了Git for Windows SDK启动时的快捷方式创建和zsh补全错误。
---

如果下载git for Windows sdk，那么每次启动shell都会尝试在桌面上创建快捷方式。
我的桌面上一般什么都没有，因此我把创建快捷方式的程序`create-shortcut.exe`删了。
然而，这又导致每次启动时都会显示错误，未找到这个exe。并且由于我使用zsh，
还会显示一个zsh不应该加载bash的git补全的错误。于是我把所有相关的profile全删了，
终于清净了。

直到我执行`pacman -Syu`后，这些错误突然又回来了：

```plaintext
ERROR: this script is obsolete, please see git-completion.zsh
sdk:484: command not found: create-shortcut.exe
sdk:484: command not found: create-shortcut.exe
Welcome to the Git for Windows SDK!

The common tasks are automated via the `sdk` function;
See `sdk help` for details.
```

经过查找，这些profile都来自`*git-extra`包中，我担心直接把包删了会出bug，
又为了快速清除这些错误信息，并且保持shell整洁（反正我也用不到`sdk`命令），
我写了一个函数来删除产生错误的profile：

```zsh
rm-git-sdk () {
    rm /etc/profile.d/git-prompt.sh /etc/profile.d/git-sdk.sh
}
```
