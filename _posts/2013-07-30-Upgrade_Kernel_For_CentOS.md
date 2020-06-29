---
layout: post
tag: Others
date: '\[2013-07-30 2 20:07\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 升级 CentOS 内核到 3.x
---

一直以来, 服务器系统一直是的 CentOS-6.4, 内核的版本是 N 年以前 的
2.6.32(虽然 RH 加了很多 patch), QEMU 的版本我很早就升级到 1.5 了, 由于
KVM 在内核中的关系, 很多非常新非常 cool 的特性根本不能用, 比如: *Nested
KVM*. 加上最近在玩 openstack, 如果 hypervise 不支持 nested 的话,
会有点麻烦.

以下是一些记录:

``` bash
$ wget http://pkgs.fedoraproject.org/repo/pkgs/kernel/linux-3.10.tar.xz/4f25cd5bec5f8d5a7d935b3f2ccb8481/linux-3.10.tar.xz
$ tar xf linux-3.10.tar.xz
$ cd linux-3.10
$ cp /boot/config-`uname -r` .config
$ make oldconfig
```

kernel 的 Makefile 提供了很好的工具, 可以很方便的构建 rpm 包.

-   rpm-pkg 同时打包二进制和源码 rpm
-   binrpm-pkg 只打二进制的 rpm包

我们只需要 binrpm-pkg, so

``` bash
$ make binrpm-pkg
```

最后安装, 大功告成

``` bash
$ rpm -i kernel-3.10.0-2.x86_64.rpm  --force
```

启动 guest 的时候使用 -cpu host, 让 guest 具有几乎所有 host 的指令集

``` bash
$ qemu-system-x86_64 -enable-kvm -cpu host -smp 16 -m 8192 openstack.qcow2
```

在 guest 里面可以看到, guest 已经有 vmx(amd 的则为 svm) 指令了.

``` bash
$ grep "vmx\|svm" /proc/cpuinfo
```
