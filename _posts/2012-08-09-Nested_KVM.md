---
layout: post
tag: QEMU_KVM
date: '\[2012-08-09 四\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Nested KVM
---

最近同事抱怨开发的机器不够用, 由于是研发 KVM 相关的程序,
所以需要有带有硬件 虚拟化支持的机器. 其实公司服务器很多都是闲置的,
所以只要 Guest 能支持 KVM, 问题就迎刃而解了.
简单的对这个问题做了一下研究.

概念
====

Nested KVM 指的是在 一个运行的 KVM 虚拟机里面在运行 KVM 虚拟机.

现在 Nested KVM 已经相对稳定了(以前 Nest KVM 甚至不支持 Intel 的 CPU).

实现
====

配置 KVM 模块
-------------

首先需要加载 KVM 模块, 这个地球人都知道, 我就不多说了,
不知道的翻我以前的 BLOG

### Intel 的 Nested KVM 配置

检查是否开启了 Nested KVM,

``` bash
$ cat /sys/module/kvm_intel/parameters/nested
Y
```

如果结果是 Y, 那么说明加载 KVM 的时候已经开启了 Nested KVM, 否则使用以
下指令重载 KVM 模块

``` bash
$ sudo modprobe -r kvm-intel
$ sudo modprobe kvm-intel nested=1
```

### AMD 的 Nested KVM 配置

AMD 的和 KVM 的类似.

检查是否开启了 Nested KVM,

``` bash
$ cat /sys/module/kvm_amd/parameters/nested
1
```

如果结果是 1, 那么说明加载 KVM 的时候已经开启了 Nested KVM, 否则使用以
下指令重载 KVM 模块

``` bash
$ sudo modprobe -r kvm-amd
$ sudo modprobe kvm-amd nested=1
```

启动虚拟机
----------

首先说明一点, Nested KVM 在 QEMU 里面经过了一些变化,
[见此](http://lists.nongnu.org/archive/html/qemu-devel/2011-05/msg03198.html).

主要是之前 QEMU 有一个选项 -enable-nesting 用来开启这个功能的,
但是有一些 developper 谈到在那个时候 Intel 不支持该功能,
所以建议把这个选项去掉, 如果 是 AMD 的 CPU, 默认开启这个功能.

所以有以下的两种情况:

旧版本的 QEMU/KVM
-----------------

旧版本的 QEMU/KVM 不支持 Intel 的 CPU, 所以如果是 Intel 的 CPU,
就不用尝试了.

``` bash
$ qemu-system-x86_64 enable-kvm -enable-nesting -cpu host -m 512 debian.raw
```

请注意 -enable-nesting 参数, 没有也可能开启此功能, 但是不保证成功, 我在
16 核的 AMD 上测试的结果是没有 -enable-nesting 参数是没有此功能的.

PS, 我在 CentOS-6.3(内核版本 2.6.32-220.el6.x86~64~) 上测试时, "AMD
Opteron(tm) Processor 6128" 启动虚拟机启动失败, 换成 "amd opteron"
就没有问题.

新版的 QEMU/KVM
---------------

``` bash
$ qemu-system-x86_64 enable-kvm -cpu host -m 512 debian.raw
```

新版的 nested KVM 完全不需要 QEMU 作任何修改, 只需要指定 cpu 参数就行了,
这里我选择模拟和 host 一样的 CPU.

参考
====

[linux/Documentation/virtual/kvm/nested-vmx.txt](https://github.com/torvalds/linux/blob/master/Documentation/virtual/kvm/nested-vmx.txt)
