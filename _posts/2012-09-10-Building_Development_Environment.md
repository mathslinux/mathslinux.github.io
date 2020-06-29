---
layout: post
tag: Spice
date: '\[2012-09-10 1 13:09\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 构建开发环境
---

Spice 基本的组件包括:

spice-protocol
:   Spice 协议, 全是以头文件的形式提供的, 这个头文件 expose
    给外部用到spice的相关程序, e.g. QEMU, spice-gtk

spice-common
:   提供了一些公共的模块, 比如内存分配的 API, ssl 的 API 等, 是给
    spice-server 和 spice-gtk 共同使用的.

spice-server
:   以库的形式提供接口, 一方面用 spice-protocol 和 client 通信, 一方面,
    hypervise 调用其提供的接口提供 VDI 的能力, 如果是做客户端开发,
    这个组件是不需要的.

spice-gtk
:   用 gtk 封装的一系列客户端与 spice-server 通信的 API, 包括 连接 spice
    实例, 传递鼠标键盘事件等等, 如果是做 hypervise 端的开发, 比如 只使用
    QEMU/KVM, 那么不需要这个组件.

clone 项目仓库
==============

在 git 仓库里, spice-common 作为 spice 和 spice-gtk 的一个子项目存在,
spice-protocol 又是在 spice-common 里的一个子项目, 所以只需要 clone
spice 和 spice-gtk 就可以了.

``` bash
$ git clone git://git.freedesktop.org/git/spice/spice
$ git clone git://git.freedesktop.org/git/spice/spice-gtk
```

编译安装
========

``` bash
$ cd spice && ./autogen.sh --prefix=/usr && make -j5 && sudo make install
$ cd spice-gtk && ./autogen.sh --prefix=/usr && make -j5 && sudo make install
```

当然可以在编译的时候做一些更深的定制, 比如我的:

``` bash
$ cd spice && ./autogen.sh --prefix=/usr --build=x86_64-pc-linux-gnu \
--host=x86_64-pc-linux-gnu --mandir=/usr/share/man --infodir=/usr/share/info \
--datadir=/usr/share --sysconfdir=/etc --localstatedir=/var/lib \
--libdir=/usr/lib64 --disable-dependency-tracking --disable-static \
--disable-tunnel --enable-client --enable-gui --without-sasl \
--disable-static-linkage && make -j5 && sudo make install

$ cd spice-gtk && ./autogen.sh --prefix=/usr --build=x86_64-pc-linux-gnu \
--host=x86_64-pc-linux-gnu --mandir=/usr/share/man --infodir=/usr/share/info \
--datadir=/usr/share --sysconfdir=/etc --localstatedir=/var/lib \
--libdir=/usr/lib64 --disable-dependency-tracking --disable-maintainer-mode \
--disable-static --enable-introspection --with-audio=pulse --without-python \
--without-sasl --enable-polkit --disable-vala --with-gtk=3.0 --enable-werror \
--enable-usbredir && make -j5 && sudo make install
```
