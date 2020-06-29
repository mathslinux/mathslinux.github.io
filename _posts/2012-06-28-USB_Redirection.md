---
layout: post
tag: QEMU_KVM
date: '\[2012-06-28 四\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: USB Redirection
---

更多的信息请参考

<http://hansdegoede.livejournal.com/>

安装必要的包
============

在我的 Gentoo 上:

``` bash
$ emerge libusbx usbredir spice-protocol spice
```

libusbx 是 libusb-1.0 的一个fork, 由于将 USB Redirection merge 到
libusb-1.0 非常缓慢(貌似两个开发者有些冲突), libusbx 的开发者把具有 USB
Redirection 的 libusb-1.0 重命名为 libusbx 并且 release 了稳定版本.

usbredir 是处理 USB Redirection 的协议

spice-protocol 是 SPICE protocol 的协议头文件

spice 是 SPICE server 和 client

QEMU 编译 USB Redirection 支持
==============================

作为 QEMU contributor, QEMU 肯定要自己编译

``` bash
$ ./configure --prefix=/home/mathslinux/usr --enable-kvm --enable-debug --enable-werror \
--target-list="x86_64-softmmu" --enable-usb-redir --enable-spice
$ make && make install
```

以上指令给 QEMU 添加了 spice 支持, usb 重定向支持, 并把 QEMU 安装到了
我的家目录的 usr 下

启动 QEMU
=========

``` bash
$ ~/usr/bin/qemu-system-x86_64 -enable-kvm -cpu core2duo -smp 4 -m 4096 \
-usb -device usb-ehci -spice port=5900,addr=0.0.0.0,disable-ticketing \
-vga qxl -global qxl-vga.vram_size=67108864 -readconfig ich9-ehci-uhci.cfg \
-chardev spicevmc,name=usbredir,id=usbredirchardev1 \
-device usb-redir,chardev=usbredirchardev1,id=usbredirdev1,debug=3 \
Ubuntu-12-04-append.img
```

启动了一个 4 核, 4G内存的虚拟机, spice 端口在 5900, 开启一个 USB
Redirection 的通道

启动 Client
===========

据我所知到目前为止, 支持 USB Redirection 重定向的客户端好像只有
spice-gtk(0.11 版本之后)

记得加上 usbredir 的支持

``` bash
$ USE="usbredir" emerge spice-gtk
```

装完启动 Spice client

``` bash
$ spicy -h qemu-ipaddr -p 5900
```
