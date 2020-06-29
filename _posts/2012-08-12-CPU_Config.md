---
layout: post
tag: QEMU_KVM
date: '\[2012-08-12 7 23:08\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QEMU 的 CPU 配置
---

根据前面描述 [CPU 的基本知识](~/Dropbox/Doc/org/Others/CPU.org),
可以知道 CPU 有物理 CPU, 多核 CPU, 超线程 CPU 之分.

事实上, QEMU 支持所有这些配置, 下面一一举例来说明如何模拟这些 CPU.

基本的 CPU 模拟
===============

下面的指令模拟了一个具有 1 个物理 CPU, 两个逻辑 CPU 的系统

``` bash
$ qemu -enable-kvm -m 1024 ArchLinux.img -smp 2,sockets=1
```

在 guest 上看看 cpuinfo 的信息:

``` bash
$ cat /proc/cpuinfo
processor   : 0
physical id : 0
siblings    : 2
core id     : 0
cpu cores   : 2

processor   : 1
physical id : 0
siblings    : 2
core id     : 1
cpu cores   : 2
```

可以看到两个逻辑 CPU 是双核的, 没有使用超线程技术.

指定核心数
==========

模拟一个具有 1 个物理 CPU(双核), 四个逻辑 CPU 的系统. 此时为了满足双核
四线程的概念, 得启用超线程技术, 如下

``` bash
$ qemu -enable-kvm -m 1024 ArchLinux.img -smp 4,sockets=1,cores=2
```

``` bash
$ cat /proc/cpuinfo
processor   : 0
physical id : 0
siblings    : 4
core id     : 0
cpu cores   : 2

processor   : 1
physical id : 0
siblings    : 4
core id     : 0
cpu cores   : 2

processor   : 2
physical id : 0
siblings    : 4
core id     : 1
cpu cores   : 2

processor   : 3
physical id : 0
siblings    : 4
core id     : 1
cpu cores   : 2
```

指定 thread 数
==============

模拟一个具有 2 个物理 CPU, 四个逻辑 CPU 的系统, 启用超线程技术,
每个核心两个 线程. 不难算出, 此时每个 CPU 都是单核的(4 = 2\*2\*1).

``` bash
$ qemu -enable-kvm -m 1024 ArchLinux.img -smp 4,sockets=2,threads=2
```

``` bash
$ cat /proc/cpuinfo
processor   : 0
physical id : 0
siblings    : 2
core id     : 0
cpu cores   : 1

processor   : 1
physical id : 0
siblings    : 2
core id     : 0
cpu cores   : 1

processor   : 2
physical id : 1
siblings    : 2
core id     : 0
cpu cores   : 1

processor   : 3
physical id : 1
siblings    : 2
core id     : 0
cpu cores   : 1
```

其它
====

事实上, QEMU 还有更强大的 CPU 的配置, 比如配置 CPU 指令级, 配置
[NUMA](http://zh.wikipedia.org/wiki/%25E9%259D%259E%25E5%259D%2587%25E5%258C%2580%25E8%25AE%25BF%25E5%25AD%2598%25E6%25A8%25A1%25E5%259E%258B),
等等, 这里不一一列举.
