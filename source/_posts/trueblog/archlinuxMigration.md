---
title: 在硬盘之间迁移 Arch Linux
date: 2025/03/05 11:46:00
updated: 2025/03/11 19:26:00
excerpt: 为了腾出一块硬盘用来装在香橙派上，也为了获得更好的数据安全性，我购买了一块致态，并迁移了我的系统。虽然当时我的确是看的wiki安装的系统，但是过了一年半载后，我发现自己已经忘得差不多了。在重读wiki，迁移了系统后，我又发现grub有些“不安全”，却不曾想为了让grub“安全”有些麻烦...在帮同学装ArchLinux的时候，又遇到了新的麻烦...
tags:
    - non-ctf
    - grub
thumbnail: /assets/trueblog/archMigration.jpg
---

## From top to buttom

原来安装Arch Linux的是一块“金士顿大号u盘”nv2，由于最近未来打算买个香橙派5 plus，
正好把这个“大号u盘”升个级，整了块致态7100。硬盘准备好了，接下来就是迁移系统。

并不像windows那样直接用diskgenius就可以迁移，在linux上需要一些额外的操作。
根据[wiki上写的迁移指南](https://wiki.archlinuxcn.org/wiki/%E8%BF%81%E7%A7%BB%E5%88%B0%E6%96%B0%E7%A1%AC%E4%BB%B6#%E8%87%AA%E4%B8%8A%E8%80%8C%E4%B8%8B)，
在安装好两块盘后，启动archiso。然后给新硬盘分区，使用`cgdisk`，
分出efi、swap和根后，初始化文件系统并挂载到`/mnt/new`下，在挂载旧硬盘到
`/mnt/old`下。正好回去看了一下安装指南，太久不看都忘了差不多了。

{% note blue fa-hard-drive %}
分区时，现代硬盘请选择GPT。固态硬盘还可以设置逻辑块大小，在一些硬盘中，4KB
性能更佳，[详见wiki](https://wiki.archlinuxcn.org/wiki/%E5%85%88%E8%BF%9B%E6%A0%BC%E5%BC%8F%E5%8C%96#NVMe_%E5%9B%BA%E6%80%81%E7%A1%AC%E7%9B%98)。
注意更改逻辑块大小会格式化硬盘！虽然我的7100没4KB这个选项就是了。
{% endnote %}

挂载好后迁移文件，使用rsync，并输出简洁的进度：

```sh
rsync --info=progress2 -qaHAXS $SOURCE_DIR $DESTINATION_DIR
```

等待20分钟后，所有文件都迁移完毕了。下一步我先`umount`所有分区，
然后在`mount`新盘到`/mnt`（也许不需要umount?），在检查了`genfstab`无误后，
就可以更新fstab并`arch-chroot`去更新grub。由于是全新盘，因此还要先install
一下grub。最后`mkinitcpio -P`后就可以重启使用了。

{% note green fa-lightbulb %}
请挂载所有分区，否则在genfstab时可能缺少一些分区挂载信息。
{% endnote %}

## 补上安全措施

由于我是使用grub启动的，因此在默认情况下，可以任意修改配置。这将意味着，
只要有恶意的使用者能接触到你的电脑，可以直接在内核启动参数后添加
`init=/bin/bash`来绕过密码认证，这是非常不安全的。为了避免这种情况发生，
grub内置了密码方案，可以设置如果要启动grub shell或者编辑启动配置，
需要认证用户和密码。具体可以参考这个
[Ask Ubuntu的问答](https://askubuntu.com/questions/656206/how-to-password-protect-grub-menu)。

但是，配置完以后，默认启动任何配置都需要密码，而想要和原来一样，不需要密码就可以启动，
只是在编辑配置时需要密码，可以参照这篇
[Arch wiki的讨论](https://wiki.archlinux.org/title/Talk:GRUB/Tips_and_tricks#c-Beckab-2019-08-12T14:54:00.000Z-Password_protection_of_non_local_system_boot_options)，
设置所有启动项在启动时无需密码。

为了防止有人插u盘来实现硬盘直接读写，在加上一个bios锁就基本万无一失了。
虽然说仍然可以通过拆机，拔盘的方式读数据，但至少获得数据的难度大了很多了。
全盘加密就算了，有点性能损失不说，windows、linux双平台的兼容性才是真正的地狱。

**然而，grub的实现似乎有点不合理...**

当submenu是unrestricted时，submenu可以随意进入，并且其中的menuentry，
尽管像其他的menuentry一样是unrestricted，仍然可以任意修改或打开grub
命令行。太不安全了！

为此，我决定用在`10_linux`规则中将submenu指定为restricted，补上这个漏洞。
（或者将submenu展开也行，但是看着有点冗余了）为了避免系统更新的时候将其覆盖，
我还写了一个pacman hook来在更新系统的时候自动加上我的限制。

```sh 04_make_OS_entries_unrestricted
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.

submenu_id_option="$menuentry_id_option"
menuentry_id_option="--unrestricted $menuentry_id_option"
```

```ini grub.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = grub
[Action]
Description = Re-restricting GRUB's submenus...
Depends = sed
When = PostTransaction
Exec = /usr/bin/sed -i "s/menuentry_id_option 'gnulinux-advanced/submenu_id_option 'gnulinux-advanced/g" /etc/grub.d/10_linux
```

这样大概就真的把漏洞补上了吧...

## Build from 0

之后还帮同学安装了Arch Linux，也是和我一样的双硬盘双系统方案。
尽管仔细阅读了安装指南，但是感觉官方文档仍然只适合有一定linux基础的用户参考，
从0开始安装，不使用`archinstall`的情况下，有一些必要的操作，比如
`systemctl enable NetworkManager`这样的操作写得并不清晰，
没有直接一条指令来警示用户，有不少操作是系统没有按预期运行，我才回到
archiso中补做的。~~我都不知道我自己装的时候是怎么一遍过的，我都忘记用
`systemctl enable sddm`开启桌面环境自启了~~

{% note yellow fa-bug %}
不要尝试在x11 + nvidia + kde的情况下，选择只在外接显示器上显示...
设置了之后无响应，只有鼠标能动，画面没反应...如果设置了的话，
需要在tty里，删除`~/.local/share/kscreen`下对应的配置文件，
然后重启才行！ ~~我们KDE用户也有自己的赛博灯泡~~
{% endnote %}

## 参考

1. [先进格式化 - NVMe固态硬盘 - Arch Linux中文维基](https://wiki.archlinuxcn.org/wiki/%E5%85%88%E8%BF%9B%E6%A0%BC%E5%BC%8F%E5%8C%96#NVMe_%E5%9B%BA%E6%80%81%E7%A1%AC%E7%9B%98)
2. [迁移到新硬件 - 自上而下 - Arch Linux中文维基](https://wiki.archlinuxcn.org/wiki/%E8%BF%81%E7%A7%BB%E5%88%B0%E6%96%B0%E7%A1%AC%E4%BB%B6#%E8%87%AA%E4%B8%8A%E8%80%8C%E4%B8%8B)
3. [How to password protect Grub menu - Ask Ubuntu](https://askubuntu.com/questions/656206/how-to-password-protect-grub-menu)
4. [Talk:GRUB/Tips_and_tricks - Password protection of non local system boot options - ArchWiki](https://wiki.archlinux.org/title/Talk:GRUB/Tips_and_tricks#c-Beckab-2019-08-12T14:54:00.000Z-Password_protection_of_non_local_system_boot_options)
