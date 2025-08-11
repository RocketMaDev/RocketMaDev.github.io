---
title: 修复zsh上patchelf自动补全的错误
date: 2024/5/8 22:40:00
updated: 2025/8/11 23:42:00
tags:
    - tricks
    - non-ctf
thumbnail: /assets/trueblog/patchelfFix.png
---

<!--excerpt-->

如果你使用zsh来作为你的主力shell的话，那你在使用patchelf的时候一定受到过这样的困扰：
在尝试补全`--set-interpreter`后的参数时，按下tab就变成了 *_arguments:463: command not found: dynamic*；
在尝试补全`--replace-needed`后的参数时，又变成了 *_arguments:463: command not found: LIB_ORIG*，
这个问题已经存在了一年多，也困扰了我很久

直到有人就因为这个小事，不愿尝试zsh，那不是丢了西瓜捡芝麻，zsh这么好用的说...

## 前情提要

早在21年， **@Y7n05h**[尝试为zsh增加补全支持](https://github.com/NixOS/patchelf/issues/310)，
但随着他不再使用patchelf，他也停止了补全脚本的编写

23年， **@Freed-Wu**接过了任务，完成了补全脚本并提交了PR，但是他的脚本并不完全正确，
导致了zsh无法正确补全

## 修复

首先就是针对上面两个常用arg进行修复，我先尝试问了下ChatGPT，
在[文件](https://github.com/NixOS/patchelf/blob/master/completions/zsh/_patchelf#L5)中，
`INTERPRETER`是消息，`dynamic loader`是描述，而`_files`是命令，那为什么无法运行呢？

查阅别人写的[zsh自动补全脚本入门](https://chuquan.me/2020/10/02/zsh-completion-tutorial/)，
发现多了一个参数，正确示例是`-OPT[DESCRIPTION]:MESSAGE:ACTION`，那么修复起来就很简单了，
只要把多余的参数去掉，换上`_files`，就能正常运行了

## 增强

patchelf还可以`--print-needed`，那么是不是可以把打印出来的依赖，作为`--replace-needed`的补全呢？
在ChatGPT的帮助下，这下可以在判断对象文件是elf的情况下，按tab补全它的依赖了

但是，ChatGPT给的是类似与bash与zsh脚本的混合，那既然都选了zsh了，就替换为zsh语法吧

跟着[别人的指引](https://github.com/goreliu/zshguide)，我又把补全脚本完善了一遍，
加速了条件的判断，也提高了对文件范围的判断，
详细可见这个[commit](https://github.com/RocketMaDev/patchelf/commit/61a49b905c2eb329848349dc8c0eb6c5fa873aa7)

## 安装使用

本文对应的PR: https://github.com/NixOS/patchelf/pull/552

{% note green fa-champagne-glasses %}
PR已经被合入啦，找文件在主线里找哦
{% endnote %}

1. `echo $fpath`列出补全文件搜索目录
2. 找到`_patchelf`文件所在位置（如在我这里是在`/usr/local/share/zsh/site-functions`中）
3. 从~我的仓库~官方仓库中下载`_patchelf`并替换你的`_patchelf`
```sh
curl -o /path/to/your/_patchelf https://raw.githubusercontent.com/NixOS/patchelf/master/completions/zsh/_patchelf
# 可能需要root
# 或者手动复制并覆写
```
4. 执行`unfunction _patchelf && autoload -U _patchelf`或重启shell
5. （可选）在Arch Linux上，只要更新patchelf，补全脚本就会被覆盖，因此可以写一个函数，
放在`~/.zshrc`中，方便在patchelf更新的时候再次覆盖脚本
```zsh
update_patchelf() {
    sudo curl -o /usr/share/zsh/site-functions/_patchelf "https://raw.githubusercontent.com/NixOS/patchelf/master/completions/zsh/_patchelf"
    unfunction _patchelf && autoload -U _patchelf
}
```

推荐patch对象前置，例如`patchelf $ELF --replace-needed [TAB]`，这样方便补全，
否则就需要先留个空位，然后写上要替换的so的路径，写上patch对象，再回过去补全

{% note blue fa-heart-crack %}
由于patchelf迟迟不发新包，我就想着向[Arch官方推个MR](https://gitlab.archlinux.org/archlinux/packaging/packages/patchelf/-/merge_requests/2)
解决一下补全脚本问题，直接被驳回了😭
{% endnote %}
