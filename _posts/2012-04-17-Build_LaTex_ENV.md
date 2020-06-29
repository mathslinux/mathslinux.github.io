---
layout: post
tag: Gentoo
date: '\[2012-04-17 二 10:51:33\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Gentoo 搭建 LaTex 编辑环境
---

记录一下过程:

使用的是 ctex 的方案, 这样我不用再为中文和各种字体头疼了, 直接使用系统
中的字体. 由于系统是 Gentoo, 所以我装 zhspacing 这个包, 把依赖也一起
解决了.

    $ sudo emerge texlive           #不知道为什么安装 zhspacing 没有安装这个包
    $ sudo emerge zhspacing
    $ sudo emerge dvipdfmx          #需要这个包的支持

剩下的, 参考之前的文章 [用 org-mode 写
LaTeX](http://mathslinux.org/?p%3D58)
