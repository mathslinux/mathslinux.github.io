---
layout: post
tag: QEMU_KVM
date: '\[2012-04-17 2 10:04\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 在 QEMU 上使用 KGDB 调试内核
---

最近在研究 Linux Kernel, 由于我看问题喜欢直接看本质,
所以直接从代码开始看起, 但是 Linux 发展到现在代码何其多, 何其复杂,
里面的流程, 逻辑, 甚至各种变量绝对不是 我以前开发的项目能比的,
比如说里面全局变量的大量使用, 各种 goto 的使用,
所以必须要有一个好的阅读方法和好的阅读手段. 阅读代码的
[Emacs](http://www.gnu.org/software/emacs/) 和
[cscope](http://cscope.sourceforge.net/) 足以. 但是对于习惯 gdb
调试的我, 还是希望可以利用 gdb 的强大优势帮助学习. 在加上 *QEMU* 来作为
kernel 的运行平台, 这样的组合不事半功倍都说不过去.

内核构建
========

下载内核
--------

经我测试 linux-2.6.24 和 linux-2.6.25 没有 KGDB 的支持. 为了能有 KGDB 的
支持, 我选择版本稍微高一点内核.

``` bash
# mkdir -p ~/Develop/Linux && cd ~/Develop/Linux
# wget http://www.kernel.org/pub/linux/kernel/v2.6/linux-2.6.34.tar.bz2
# tar xf linux-2.6.34.tar.bz2
```

配置内核
--------

``` bash
# cd ~/Develop/Linux/linux-2.6.34 
# make defconfig # 用 defconfig 生成 一个精简内核 
# make menuconfig
```

确保下面的被选中

``` bash
 General setup  ---> 
     [ * ] Prompt for development and/or incomplete code/drivers
Kernel hacking  --->
     [ * ] Compile the kernel with debug info
     [ * ] Compile the kernel with frame pointers
     [ * ] KGDB: kernel debugger  --->
           < * >   KGDB: use kgdb over the serial console
```

OK, 下面开始编译内核

``` bash
# make -j5 # 因为我是 4 核的 CPU, 所以这里使用5个线程同时并发执行
```

QEMU 构建
=========

安装 QEMU
---------

这里可以选择多种安装方式, 可以选择从源码安装或者从发行版的二进制安装.
作为一个比较喜欢追根究底的 geek, 我选择的是源码安装.

另外, 我调试的只是 kernel, 所以没有硬件虚拟话我是可以忍受的, 所以我把
kvm 从编译参数那里去掉了.

``` bash
# mkdir -p ~/Develop/QEMU
# cd ~/Develop/QEMU
# wget http://wiki.qemu.org/download/qemu-1.0.tar.gz
# tar xf qemu-1.0.tar.gz
# cd qemu-1.0
# ./configure --prefix=./ --target-list="i386-softmmu x86_64-softmmu" --disable-kvm
# make -j3 && make install
```

注意, 因为我不想把 qemu 和我系统的 qemu 冲突, 我简单的将 qemu 安装在
qemu 源码目录下.

文件系统构建
============

安装 busybox
------------

``` bash
# mkdir ~/Develop/busybox
# wget http://www.busybox.net/downloads/busybox-1.19.3.tar.bz2
# tar xf busybox-1.19.3
# cd busybox-1.19.3
# make menuconfig
```

静态编译的选择很重要, 如果不选择的话, 需要把 libc.so 和 ld.so 等
复制到文件系统里面, 稍微麻烦一些, 这里我们选择最简单的方式.

另外这个版本静态编译的时候 mount umount会出错, 方正对我来说不需要,
我暂时去掉.

``` bash
Busybox Settings  ---> 
   Build Options  --->
        [ * ] Build BusyBox as a static binary (no shared libs)
Linux System Utilities  --->
   [ ] mount 
   [ ] umount
```

执行 make install 后, 会在 busybox 的源码目录地下创建一个 ~install~
的文件夹, 这个文件夹就是需要复制到文件系统里面的东西.

``` bash
# make install
```

制作文件系统
------------

首先创建一个虚拟盘, 并挂载到当前 tmp 目录下.

``` bash
# cd ~/Develop
# dd if=/dev/zero of=initrd.img bs=1k count=8192
# mkfs.ext3 -F -v -m0 initrd.img
# mkdir tmp
# mount -o loop initrd.img  tmp
```

然后把编译好的 busybox 复制到挂在虚拟盘的目录里面.

``` bash
# cp -dpRrf ~/Develop/Linux/busybox-1.19.3//_install/* tmp
```

创建一些必须的设备文件, 其实设备号几乎是通用的,
所以我直接把本机的设备文件 复制过来了.

``` bash
# cp -dfrpa /dev/console tmp/dev
# cp -dfrpa /dev/tty* tmp/dev
# cp -dfrpa /dev/mem tmp/dev
# cp -dfrpa /dev/null tmp/dev
# cp -dfrpa /dev/random tmp/dev
# umount tmp
```

运行
====

``` bash
# ~/Develop/QEMU/qemu-1.0/bin/qemu-system-x86_64 \
# -kernel ~/Develop/Linux/linux-2.6.34/arch/x86/boot/bzImage \
# -hda ~/Develop/initrd.img -m 2048 -append "root=/dev/sda init=/bin/sh"
```

在运行的时候碰到了 init 段错误的问题. 我不知道是不是静态编译导致的.
解决这个问题有两个办法, 或者从其他发行版复制一个静态的 busybox过来,
我试过, 没有问题. 或者把内核启动参数改为 init=/bin/sh 不让 kernel 去启动
init. 我选的是后者.

开始调试
========

-s 表示用默认的 1234 端口, 开启 gdb server

``` bash
# ~/Develop/QEMU/qemu-1.0/bin/qemu-system-x86_64 \
# -s -S -kernel ~/Develop/Linux/linux-2.6.34/arch/x86/boot/bzImage \
# -hda ~/Develop/initrd.img -m 2048 -append "root=/dev/sda init=/bin/sh"
```

在我的 Emacs 里面, 使用 /tmp/gdb/bin/gdb –annotate=3
\~/Develop/Linux/linux-2.6.34/vmlinux 来启动, 进去以后, 设置断点, 然后
target remote localhost:1234 连接 gdb server

其它.
=====

我 gdb 的启动指令是 /tmp/gdb/bin/gdb 而不是默认的 gdb, 因为默认的 gdb
在我的 x86 平台上调试的时候有一个小 bug, 根据邮件列表上的说法. 需要 hack
代码, 然后重新编译. 我打了一个小 patch.

``` bash
--- gdb-7.3.1-orign/gdb/remote.c    2011-07-15 10:04:29.000000000 +0800
+++ gdb-7.3.1/gdb/remote.c  2011-12-27 18:37:34.319902796 +0800
@@ -5702,9 +5702,21 @@
   buf_len = strlen (rs->buf);

   /* Further sanity checks, with knowledge of the architecture.  */
+#if 0
   if (buf_len > 2 * rsa->sizeof_g_packet)
     error (_("Remote 'g' packet reply is too long: %s"), rs->buf);
-
+#endif
+  if (buf_len > 2 * rsa->sizeof_g_packet) {
+     rsa->sizeof_g_packet = buf_len ;
+     for (i = 0; i < gdbarch_num_regs (gdbarch); i++) {
+         if (rsa->regs[i].pnum == -1)
+             continue;
+         if (rsa->regs[i].offset >= rsa->sizeof_g_packet)
+             rsa->regs[i].in_g_packet = 0;
+         else 
+             rsa->regs[i].in_g_packet = 1;
+     }    
+  }
   /* Save the size of the packet sent to us by the target.  It is used
      as a heuristic when determining the max size of packets that the
      target can safely receive.  */

```
