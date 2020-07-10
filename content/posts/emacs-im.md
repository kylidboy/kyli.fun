---
title: 终于可以在Emacs下调用Fcitx打拼音了
date: 2020-06-29
lastmod: 2020-06-29
description: 一直苦于没有中文输入法，在Emacs和VSCode之间来回切换，两种不同的键位mindset确实累，经常敲错。
featuredImage: 
featuredImagePreview: 
tags: 
- emacs 
categories:
- Emacs
---
<!--more-->

## LC_CTYPE=zh_CN.UTF-8 emacs

据说这是一个Emacs的千年bug。

这就是这篇Blog的全部内容了。。。。。。之前在Manjaro KDE下，这个是无效的，不知道为什么，而且我在Settings里只要装了中文支持，命令行就会变成中文的，`.pam_environment`怎么修改都没有用，而且Emacs还是不能输入中文。。。。。。这就很Fuck了，给我看中文，但是不给我输入中文的福利。

最近Linux Mint 20发布了，重新装了Cinnamon，一切都OK了。

为了不影响系统其他部分的显示，拷贝`/usr/share/applications/emacs.desktop`到`$HOME/.local/share/applications/`下，然后修改`Exec=env LC_CTYPE=zh_CN.UTF-8 emacs %F`。

大功告成。
