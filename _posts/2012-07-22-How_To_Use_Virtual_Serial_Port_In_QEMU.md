---
layout: post
tag: QEMU_KVM
date: '\[2012-07-22 日\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 如何使用 QEMU 中的虚拟串口
---

QEMU 具有模拟 [串口](http://en.wikipedia.org/wiki/Serial_port) 和
[并口](http://en.wikipedia.org/wiki/Parallel_port) 的能力, 在 QEMU
的命令行接口, 提供了 -serial 参数供用户设置把虚拟的串口重定向到哪里.

本文档主要介绍如何使用这个虚拟串口, 接下来会从代码方面描述 QEMU 如何模拟
一个串口.

不使用串口
==========

``` bash
$ qemu-kvm ArchLinux.img -serial none
```

不管是 Linxu 还是 Window, 在 QEMU 里面禁用了串口, 但是用一些硬件检测
工具还是能检测到串口的存在. 用一段简单的代码来检测是否串口可以使用

``` python
#!/usr/bin/python
import serial
try:
    s = serial.Serial('/dev/ttyS0')
    print "Find serial port on /dev/ttyS0"
except:
    print "Cant found serial port on /dev/ttyS0"
```

重定向到虚拟控制台
==================

``` bash
$ qemu-kvm ArchLinux.img -serial vc:800x600 # 或者 vc:80Cx24C
```

实际上, 默认 启动 QEMU 的时候如果不加参数的话, 会自动创建四个控制台,
分别用 Ctrl + Alt + number 来切换, number 为 1, 2 或 3, 4 其中 1 是 QEMU
的图形终端, 2 是 QEMU 的 Monitor 终端(QEMU 的 Monitor 稍候会介绍), 3 是
Serial 终端, 4 是并口终端.

重定向到一个伪终端
==================

``` bash
$ qemu-kvm ArchLinux.img -serial pty
```

然后会发生什么呢? QEMU 会自动创建一个伪终端设备(/dev/pts/3) 之类的, 然后
用 screen 之类的工具就可以操纵这个终端了.

PS. 这东西 Linux only 哟

重定向到 null
=============

``` bash
$ qemu-kvm ArchLinux.img -serial null
```

这和重定向到 none 有什么区别呢? 区别就是 -\> none QEMU 不会虚拟串口设备,
但是 -\> null 会虚拟一个串口设备, 丢弃所有的输出: 如以下的代码.
至于输入? 需要输入吗?

``` c
static int null_chr_write(CharDriverState *chr, const uint8_t *buf, int len)
{
    return len;
}
```

重定向要真实串口设备
====================

``` bash
$ qemu-kvm ArchLinux.img -serial /dev/ttyS0
```

但是虚拟串口的硬件参数需要和真实串口符合

PS. Linux only 哟

重定向到并口
============

``` bash
$ qemu-kvm ArchLinux.img -serial /dev/parportN
```

重定向到第 N 个并口

PS. Linux only 哟

重定向到一个文件
================

``` bash
$ qemu-kvm ArchLinux.img -serial file:/tmp/serial.out
```

不过从打开文件的方式看来, QEMU 只是把串口的输出写入文件,
而并不支持串口的 输入.

``` c
TFR(fd_out = qemu_open(qemu_opt_get(opts, "path"),
                       O_WRONLY | O_TRUNC | O_CREAT | O_BINARY, 0666));
```

重定向到 stdio
==============

``` bash
$ qemu-kvm ArchLinux.img -serial stdio
```

把串口重定向到标准输入输出, 这给调试 Guest OS(其实我说的是 Linux OS,
你要调试 Window OS? 你吃饱了撑的?)提供了方便.

重定向到管道
============

``` bash
$ qemu-kvm ArchLinux.img -serial pipe:/tmp/serial:
```

不过这玩意还比上面的复杂, 根据 QEMU 打开这类设备的代码, 需要手动创建
/tmp/serial.in 和 /tmp/serial.out 两个管道文件

``` c
snprintf(filename_in, 256, "%s.in", filename);
snprintf(filename_out, 256, "%s.out", filename);
TFR(fd_in = qemu_open(filename_in, O_RDWR | O_BINARY));
TFR(fd_out = qemu_open(filename_out, O_RDWR | O_BINARY));
```

``` bash
$ mkdir fifo /tmp/serial.in
$ mkdir fifo /tmp/serial.out
```

怎么使用呢? cat /tmp/serial.out 会看到 Linux 登录的一堆信息, 最后停在
`virt-debian login:` 这里等待输入, 用

``` bash
$ echo your_username >> /tmp/serial.in
$ echo your_password >> /tmp/serial.in
```

再打开 cat /tmp/serial.p.out 就可以看到内容已经变成
`root@virt-debian:~#` 这样的东西了.

所以这玩意对我用处不大

重定向到 udp 端口
=================

``` bash
$ qemu-kvm ArchLinux.img -serial udp::3333
```

将 QEMU 的串口重定向到 3333 端口, 使用 nc 访问这个端口 (当然可以自己编写
socket 访问). 这对远程管理很有帮助

``` bash
$ nc -u -l -p 3333
```

重定向到 tcp 端口
=================

``` bash
$ qemu-kvm ArchLinux.img -serial tcp::3333,server,nowait
```

可以使用 telnet 来访问该端口

``` bash
$ telnet localhost 3333
```

重定向到 telnet
===============

几乎 TCP 是一样的

``` bash
$ qemu-kvm ArchLinux.img -serial telnet::3333,server,nowait
```

重定向到 Unix socket
====================

``` bash
$ qemu-kvm ArchLinux.img -serial unix:/tmp/serial.sock,server,nowait
```

用 socat 连接

``` bash
$ socat  /tmp/serial.sock STDIO
```

同时定向到串口和 mon 控制台
===========================

``` bash
$ qemu-kvm ArchLinux.img -serial mon:telnet::3333,server,nowait
```

这将同时定向串口到 TCP 3333 端口的同时, 可以使用 Ctrl + a 然后按 c 访问
Monitor 终端

定向到 braille
==============

这个太强大了, 相关 google braille

msmouse
=======

重来没有使用过
