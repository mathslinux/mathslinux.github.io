---
layout: post
tag: Mac
date: '\[2013-11-10 日 22:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 在 Mac 下配置 Linux Kernel 开发环境
---

自从我的 [主要办公环境转到 Mac 下](http://mathslinux.org/?p%3D325) 之后,
我的开发方式经历了从 native –\> remote server 的转变.
但在有时公司网络很差, 而周末我在家里又想研究一些非得在 Linux 下运行的程
序的时候, 麻烦就来了. 比如 Linux kernel, KVM 等等.

不过作为一个虚拟机爱好者(QEMU/KVM contributor),
这种事情应该难不倒我才对. 非常不幸, KVM 不支持 OSX, 或者说, OSX 不支持
QEMU 下的某一种 CPU 模拟的加速(KVM,XEN,或 kmod). 直接后果就是, 如果我用
QEMU 原生的 [TCG](http://wiki.qemu.org/Documentation/TCG) 翻译,
那么性能会差非常多, 多到不能忍受, 躲到 可以放弃节操!

万邦无奈之下, 只好把节操先丢一边, 把
[Virtualbox](https://www.virtualbox.org) 先用起来再说.

到 [VirtualBox
的官方下载页面](https://www.virtualbox.org/wiki/Downloads)
下载安装最新的版本. 再根据个人喜好安装一个 Linux 发行版,
这里为了省事我选择最新的 Ubuntu-13.10. 安装好 Ubuntu-13.10 之后,
照着我之前写的 [在 QEMU 上使用 KGDB
调试内核](http://mathslinux.org/?p%3D130) 就没有问题了.

以下策略是为了最小化资源占用所做的配置, 没办法, 我的 Macbook Air 只有 4
GB 的内存, 我还要在虚拟机里面开 QEMU 调试 kernel, 所以不得不省着点用.

策略 1:
=======

务必最小化安装 linux, 什么乱七八糟的 X, unity, gnome 都不要安装, 这样
不仅可以节省大量的空间和安装时间, 还可以节省很多内存.

策略 2:
=======

为了方便调试和开发, 采用 ssh 登陆到 geust 的方式, 由于我用的是 NAT
的网络 模式, 为了能方便地访问虚拟机, 需要做 host \<–\> guest 的端口转发:

在 设置 –\> 网络 –\> 端口转发, 添加一条名称为 ssh, 协议是 TCP, 主机端口
3456, 子系统端口 22 的规则, 其他部分留空即可.

这时, 即可以用以下指令直接连接到虚拟机

``` bash
$ ssh -p 3456 root@127.0.0.1
```

为了更方便的能连接到虚拟机, 可以把主机的 ssh 秘钥复制到虚拟机的 ssh
配置文件里. 然后在 \$HOME/.ssh/config 文件里添加一个配置条目

``` bash
Host vbox
    HostName 127.0.0.1
    User root
    Port 3456
```

之后, 就可以直接

``` bash
$ ssh vbox 
```

就直接连接进入虚拟机了

策略 3
======

为了节省运行时内存, 减少应用程序的窗口, 可以用 headless 模式打开虚拟机,
并且关闭 rdp. 这类似于 QEMU 的 nographic 模式, 就像用命令行启动 VBOX
一样.

``` bash
$ VBoxHeadless --startvm ubuntu-13.10 --vrde off
```

然后, 就可以

``` bash
# apt-get update && apt-get build-dep qemu
```

PS
==

我知道有一个叫 [vagrant](http://www.vagrantup.com)
的东西可以把上述步奏都一次性完成, but:

-   我没有时间去熟悉他的配置和用法, 编写(寻找)需要的 guest 的配置文件
-   我完成上诉的结果花了不到 10 分钟.
-   我现在开始我的开发之旅只需 ssh vbox, 而且资源占用小到不行.
