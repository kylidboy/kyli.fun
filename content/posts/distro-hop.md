---
title: 最近频繁的切换Distro，为了一个最舒服的DE
date: 2020-06-29
lastmod: 2020-06-29
description: 自从发现了i3wm之后，就爱上了tiling，尝试了xmonad|awesome|i3wm，然而，孱弱的管理功能（基本没有），导致在笔记本上很难完美舒适的姿势。在这里回顾一下Hopping的各种姿势体验。
featuredImage: /images/distro-hop/distros.jpg
featuredImagePreview: /images/distro-hop/distros.jpg
tags: 
- DE
- Linux
- Distro Hopper
categories:
- Linux Distro
---
<!--more-->

为什么会喜欢Tiling window manager呢？个人使用linux作为主要的学习和开发平台（只有一台笔记本，dell xps15 最渣配），平时主要的routine就是虚拟终端+Emacs+Firefox[Chrome]，非常的固定，每次打开应用之后默认的动作都是全屏，然后在各个窗口之间切换。通过tiling，可以省略全屏这一步，同时通过虚拟桌面，直接分离各个应用，完全手不离键盘就可以完成所有的切换，非常效率（帅）。主要是喜欢这种简洁的感觉。

Win + Tab 或者 Alt + Tab 这种非人类的操作基本上我是不用的。还有macOS那种全屏之后左划右划，特别奇怪，还有不计其数的人为了这个去买苹果的触摸板（请原谅我的穷）。

这里简单罗列一下最近试过印象比较好的的Distro+DE的组合，截图什么的，就算了，太麻烦，有兴趣的人，自然会自行闻鸡起舞。

## Manjaro i3
我装的是19.x的版本，现在最新的已经20.0.3了好像。

Manjaro i3呢，不是官方的版本，属于社区维护的性质，但它更新的非常快，基本上都是紧密贴合manjaro官方发布的节奏，基本目前不用担心失去维护。
我装的时候，那个版本有个bug，安装程序没有中文支持，这个有点傻。不过不影响安装，一路next到底，重启，一个精心配置过的i3wm(i3gap)，即开即用。整个桌面和电源管理套件，用的是xfce的。我记得一个非常大的缺点是，快捷键没用使用Vim的`h,j,k,l`。所以要自己修改一下配置。默认自带的conky也配置的很好。

[Manjaro i3](https://manjaro.org/download/#i3)

## Regolith(月岩灰)
非常漂亮，比较小众，我也忘记了我是怎么找到它的。

Regolith是一个基于Ubuntu Gnome的桌面套件，可以说只是把window manager换成了i3gap，其他都是Gnome的。所以你在gnome里的所有功能和交互，在regolith里都基本保留。Gnome的桌面我一直觉得非常反人类，一直不能get到它这么设计的点，甚至一些基本配置都需要安装Tweak和dconf才能完成，不厌其烦。这点regolith也爱莫能助。

Regolith的作者对i3做了很多细心的配置，`super + ?`会弹出一个Help窗口介绍所有的快捷键，这一点我最喜欢。

相比Manjaro i3，regolith感觉要更精致一些，因为主题配色和gnome套件加成的关系。

[Regolith](https://regolith-linux.org/)

## Manjaro KDE with Krohnkite(极力推荐)
KDE当然不是什么tiling wm啦，但是KDE的功能之强，配置之具，一直可以不断的震惊到我。

最近碰巧在油管上看到了一个关于Krohnkite的介绍，怦然心动。

Krohnkite是KDE的默认WM KWin的一个脚本，通过Krohnkite，可以直接把KWin变成一个tiling window manager，还可以自定义gap，神不神奇？我装了之后，非常惊艳，可谓是极其完美的。拥有了KDE所有的管理配置能力，同时还有完美的Tiling。只要配置一下虚拟桌面和对应的快捷键，整个DE的操作体验可以完美复刻i3。公司的小朋友看到我的自定之后，也依然决然的装上了，现在沉浸不能自拔。

[Krohnkite Github](https://github.com/esjeon/krohnkite)

## LinuxMint Cinnamon
这个是我目前在用的，虽然不能说是tiling，但是Cinnamon自带的`windows tiling snap`功能，还是能比较好的为我提供“类tiling”的体验。

从Krohnkite换到Cinnamon其实我是挺不情愿的，因为Krohnkite真的挺完美的。但是KDE本身有点让我头疼的是，xps15的屏是普通DPI的，我办公室一直用的4K的27寸显示器。不能在这两者之间设置不同的DPI scale我不怪KDE，X11和Wayland都各种残疾。但是，我只使用单显示器，切换DPI每次都要重启这个就很烦了。而且莫名的，HDMI不能60hz，只能50hz。

所以我现在在LinuxMint Cinnamon下，用Roif+Albert+Cinnamon自带的功能，基本上也可以符合自己的使用习惯了。

# 流水帐完成
