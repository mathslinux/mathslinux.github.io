---
layout: post
tag: Mac
blog: 'my-blog'
category: Mac
date: '\[2013-08-24 六 20:54\]'
postid: 325
title: Linuxer 的 Mac 叛逃之路
---

本周入手了一台 Macbook air，正式开始了从 Liunx 到 Mac 的叛逃之路.

概况
====

我始终相信 [\*unix](http://en.wikipedia.org/wiki/Unix-like)
是世界上最好的操作系统, 由于 OSX ∈ \*nix, 所以 Mac OS 也是
世界上最好的操作系统之一.

由于之前是一个 linuxer, 且我对游戏没有偏好, 所以用 Mac
之后感觉没有太大的变化, 反而它华丽而不失稳定的 UI, 多达 12 小时的续航,
各种对得起业界良心的硬件设计, 无一不深深的震撼着我.

需要克服的困难
==============

除了 Mac OS 不支持 [KVM](http://www.linux-kvm.org/page/Main_Page)
之外，我暂时想不到有什么困难需要我去克服的，难道是没有
[gnome3](http://www.gnome.org/gnome-3/) (LOL)。

由于不支持 KVM，意味着我以后得去服务器上去开发调试 KVM
相关的东西了。而对于我， 一个 emacser，变化只不过是从 X 环境下的
[emacs](http://en.wikipedia.org/wiki/Emacs) 换到 terminal 下的 emacs，
某些冲突的快捷键需要重新定义而已。

Mac 的优点
==========

下面是我用了几天之后感受到：

-   硬件设计无懈可击，这个不需要解释，业界公认
-   华丽的操作系统，还记得前几年各大 linux 论坛争相模仿 mac
    主题的盛事吗， 还记得 [ubuntu](http://www.ubuntu.com)
    的全局菜单吗，还记得 gnome3 画虎不成反类犬 吗？
-   系统出奇的稳定，从买来到现在我从来没有关过一次机，没有感觉到到卡顿，
    虽然 我的 air 开机只要 10S 不到，但是我就是没有关过机。
-   超常的续航，12小时！不是盖的。(air)
-   小巧轻便，随时打开随时工作，这篇 blog
    的部分内容就是堵车的时候在同事的车里 完成的(air)。
-   软件的兼容性。这点对中国用户来说可能很重要，我随手到 [App
    Store](http://www.apple.com/cn/osx/apps/app-store.html) 上扫了一眼，
    什么迅雷，搜狗输入法，Microsoft
    office，有道词典，QQ，酷狗音乐，PPVT，PPS， 太多说不过来。
-   与 大部分 [GNU](http://www.gnu.org)
    系统完全兼容，这点对我来说太重要了，感觉除了 linux kernel，几乎 所有
    Linux 下能跑的软件跑得很欢。甚至有“丧心病狂”的项目想把 linux
    发行版除了 内核之外的所有东西移植到 OSX 中，比如 [Gentoo
    Prefix](http://www.gentoo.org/proj/en/gentoo-alt/prefix/)
    这货。这货据说可以在 Windows 下也跑得很欢。//其实我从前是一个
    gentooer

常用软件
========

开发类
------

-   bash 这个太有用了，系统自带
-   xcode 开发基础包，linuxer 必装
-   emacs 没有你我怎么办
-   pip 很奇怪 Mac 居然默认没有安装这个包
-   homebrew 包管理器 // 下面的工具都是通过 homebrew 安装
-   wget/axel 下载工具
-   automake, libtool 软件编译工具
-   proxychains-ng 翻墙工具

其他
----

-   Safari
    自带的浏览器，通过一些伪装，播放视频的时候能达到最好的效果和性能，以后介绍
-   google-chrome 不解释 //发现解决了翻墙问题后，我都不怎么用
    google-chrome 了
-   dropbox 跨平台的 “网盘”
-   搜狗拼音 输入法，不解释
-   有道词典 其实用的不多，自带的已经很强大了
-   QQ 和非 geek 交流用
-   iTunes 音乐相关，还没有玩转
-   App Store Mac
    自带的软件包管理器，由于上面的软件都是经过审查的，所以质量很高

包管理器
========

相比于 Windows 上软件管理的混乱，这是所有 ＊nix 的亮点之一， Mac
上也有同样的机制。 我考察了以下的包管理器，对比了一些各自的优缺点：

-   App Store 这个是 mac
    官方自带的，血统最纯的，严格来讲和我相说的包管理器没有
    太大关系，只是列出来做个参考
-   [Fink](http://www.finkproject.org) 据说是基于 Debian 的 packaging
    tools 开发的，安装的软件都是预编译好的。 里面的软件都太旧。
-   [Macports](http://www.macports.org) BSD 的 port
    的变种，所有东西源码编译，而且这货完全不考虑系统已有的库，
    什么依赖都自己重新来编译一遍，大哥，咱是 Air, 硬盘小啊.
-   Gentoo prefix 基于 gentoo，特点就是恨不得把 gentoo 给移植到 osx
    上，和上面的 一路货色，据说连 gcc 都要重编。
-   [Homwbrew](http://brew.sh) 新秀级选手，一出现就抢占了无数 macports
    的地盘，纯 ruby 打造，纯 git
    管理，小巧灵活，能用系统的库尽量用系统的，不求最大，但求最好。

经过一番比较，我做了一个艰难的决定，暂时使用
homebrew，根据这两天的使用来看，完全 满足我的需求。

未来如果在 mac 上要经常编译东西的话，我可能会基于 homebrew
重写一个管理器，把 [gentoo](http://www.gentoo.org) portage
好的东西整合近来，因为以前用惯了 gentoo，UES
标记这些东西是在是太好用了。

Emacs 配置
==========

默认情况下 Mac 的按键对 Emacs 有一点不友好，主要在于多了一个奇葩的
'command' 键，在终端模式下 'alt' 键被映射到 'option' 键上。

分别解决如下：

alt 的问题
----------

在 Bash 的偏好设置 -\> 设置 -\> 键盘 -\> 使用 option 键作为 meta
键(勾选)

command 键的问题
----------------

把以下 lisp code 加入 emacs 的配置文件中，把 command 作为 alt 键。

    (when (eq system-type 'darwin) ;; mac specific settings
      (setq mac-option-modifier 'alt)
      (setq mac-command-modifier 'meta)
      (global-set-key [kp-delete] 'delete-char) ;; sets fn-delete to be right-delete
      )

常用快捷键/多点触摸操作
=======================

这几天我最常用到的快捷键/多点触摸操作：

键盘快捷键
----------

### 全局

-   command + tab: 切换程序，类似于 gnome 的 ctrl + tab
-   command + \`: 在同一个程序的多个窗口间切换，比如开了多个终端窗口
-   command + m: 最小化窗口
-   command + w: 关闭窗口/标签
-   command + q: 退出程序
-   command + +/-: 放大/缩小 图片/文字
-   command + t: 新建标签
-   command + r: 刷新
-   ctrl + space: 在 Finder 中搜索，太好用了, 类似于命令行的
    locate，但不是一个级别的
-   ctrl + F3: 平铺一个程序的多个窗口，类似于 按下 gnome3 的 win 键
-   ctrl + 左/右键: 切换一般程序和全屏程序，类似 android 的多窗口切换
-   ctrl + 上/下键: 四指上推: 平铺桌面中的程序，或者按 F3
-   command + spice: 输入法切换
-   command + ctrl + f: 全屏
-   emacs keyshort: 另外 emacs
    的很多快捷键都可以在任何程序下使用，比如在浏览 器里可以 ctrl + a/e

### Safari 或者 bash 等

-   fn + 上/下键: up/down
-   fn + 左右键: home/end

多点触摸
--------

### 全局

-   四指分开: 显示桌面
-   四指上推: 平铺桌面中的程序，或者按 F3
-   四指并拢: 打开 Launchpad，或者按 F4
-   四指左右滑动: 切换一般程序和全屏程序，类似 android 的多窗口切换
-   三指拖动: 拖动窗口
-   两个指头同时轻点: 触发右键
-   三个指头同时轻点: 选中文字并唤醒词典，太好用了，结合这个加上
    startdict

的词典使我弃用有道词典

-   双指拖移: 上下翻滚
-   三指拖动文字: 选中文字
-   从右方边缘向左轻扫: 查看通知中心

### Safari 或者其他浏览器中

-   双指左/右划: 前进/后退
-   双指张开/闭合: 放大/缩小网页
-   双指双击: 智能缩放
-   双指开合: 查过所有标签页，并双指左右拖动来切换

Tips
====

解决看视频发热过大的问题
------------------------

我的 air 在播放在线视频时非常发热量非常大, 原因是 Mac 对 flash
的支持不是 很好(好像 windows, linux 系统也一样),
目前有三种办法可以解决这个问题.

三种方法本质都是一样, 都是利用 html5 来播放, 从而避免 flash 的调用,
实践中我 一直使用第三种方法.

### 使用支持 html5 的视频网站

比如看优酷的时候, 可以使用以下网址:
[猛击我](http://labs.3g.youku.com/ipad/), 这个方法的缺陷是并不是所
有的网站都能找到对应的 html5 的页面。

### 使用插件把转换当前格式

插件地址在: [猛击我](http://zythum.free.bg/youkuhtml5playerbookmark/)，
我还没有试过，不过貌似比较靠谱。

### 伪装成 iPad 播放

我现在用的就是这种方式，我现在看的视频都是这种方式，经实践，在我的 air
上， 看视频的时候完全没有发烫的感觉，cpu 温度在 10% 左右。

以下是方法：

打开 Safari 偏好设置 -\> 高级 -\> 在菜单栏中显示“开发菜单”(勾选), 然后在
开发中选择用户代理 -\> iPad.

苹果字符的输入
--------------

 这个纯属好玩，方法是：

option + shift + k
