---
layout: post
tag: QEMU_KVM
date: '\[2012-06-05 2 14:06\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QEMU Manual V0.0.1
---

Install
=======

Configure
---------

Boot Paramenters
================

Misc
----

### -hda\[b\|c\|d\] file

使用 file 作为磁盘 0, 1, 2, 3

### -cdrom file

使用 file 作为 CD 映像, 不能和 -hdc file 同时使用, 如果 file 为
/dev/cdrom 可以直接使用物理机的 CD-ROM.

### -boot \[a\|c\|d\|n\]

设置启动顺序 floppy (a) disk (c), CD-ROM (d), or Etherboot (n).
默认是硬盘.

### -snapshot

不要把临时文件写回磁盘镜像.

### -m

设置虚机的虚拟内存大小, 单位 megabytes.

### -smp

设置虚机的 CPU 个数.

### -soundhw CardName

启用并且选择声卡设备, 使用 ? 显示支持的声卡设备. CardName=ac97
工作得很好, 不需要其他驱动.

### -localtime

设置虚拟的时钟为本地时钟.

### -no-frame

不要窗口边窗

### -full-screen

全屏

-vnc
----

pass

USB
---

-usb

开启 usb 支持

-usbdevice DeviceName

DeviceName 可以是以下:

-   host:bus.addr

Network
-------

### 基本说明

默认情况下, 在 guest 建立网络的过程中, QEMU 会为每个 Guest 设置一个
VLAN(VLAN0), uuests 彼此是独立封闭的, 无法彼此连接. Guest 使用 DHCP
获得的 IP 默认都是 10.0.2.15, 存在 VLAN0 内, 使用 user mode.
默认的路由是 10.0.2.2.

在两个 guests 之间通信, 需要再设定一个 socket.

### -net nic \[,vlan=n\]\[,macaddr=addr\]\[,model=type\]

创建一个 虚拟网卡, 连到 VLAN n(默认 n 为0).

支持的网卡可以用 -net nic,model=? 列出来.

### -net user \[,vlan=n\]\[,hostname=name\]

使用用户模式的协议栈.

### -net tap\[,vlan=n\]\[,fd=h\]\[,ifname=name\]\[,script=file\]

连接 tap name 和 VLAN n, 使用 file 脚本配置此连接. script=no
表示不使用任何配置脚本.

### -net socket\[,vlan=n\]\[,fd=h\]\[,listen=\[host\]:port\]\[,connect=host:port\]

使用一个 TCP socket 连接 VLAN n 和 另外一个 guest 的 VLAN. 使两个 guests
可以通信. 'listen' 指示本 guest 接受其它 guests 通过 port 端口的连接.
'connect' 用来连接其它 guest.

例如: 这个 guest 在 1234 端口等待其它 guests 的连接

``` bash
qemu-kvm linux.img -net nic,macaddr=52:54:00:12:34:56 \
-net socket,listen=:1234
```

这个 guest 连接到本机开放 1234 端口的 qemu.

``` bash
qemu linux.img -net nic,macaddr=52:54:00:12:34:57 \
-net socket,connect=127.0.0.1:1234
```

### net socket\[,vlan=n\]\[,fd=h\]\[,mcast=maddr:port\]

创建一个VLAN n, 并使用UDP 多址通信套掊口与其他的QEMU虚拟机进行共享,
尤其是对于每一个使用多址通信地址和端口的QEMU使用同一个总线.

Linux Boot
----------

### -kernel ImageName

使用 ImageName 作为内核镜像

### -append cmdline

使用 cmdline 作为内核启动参数

### -initrd InitrdName

使用 InitrdName 作为 ram disk

Monitor
=======

如何进入控制台界面
------------------

Ctrl+Alt+2 进入, 按 Ctrl+Alt+1 切换回工作界面. 或者 QEMU 启动的时候指定
"-monitor stdio"

help command
------------

显示帮助

info sth
--------

### info network

显示网络状态

### info block

显示磁盘信息

### info snapshots

显示快照信息

q
-

退出 QEMU

eject \[-f\] device
-------------------

弹出可移动介质

screendump filename
-------------------

保存 Guest 的屏幕截图为 filename(PPM 格式)

savevm \[tag\|id\]
------------------

保存快照

loadvm tag\|id
--------------

设置快照

delvm tag\|id
-------------

删除快照

c
-

重启 QEMU

commit ImageName
----------------

在 snapshot 模式下, 把修改写回 ImageName

Display
=======

-nographic
:   禁用 Video 输出, 把输出重定向到 serial console

-curses
:   在终端输出

Disk Image
==========

create
------

创建磁盘

``` bash
qemu-img create [-e] [-6] [-b base_image ] [-f fmt ] filename [size ]
```

### -b

后面指定一个只读的磁盘镜像, 新创建的镜像基于此做快照. 注意在用 -b
创建磁盘 的时候, 不能指定 \[size\], 否则会失败 \#FIXME

### fmt

1.  raw

    速度快, 移植性好, 创建时使用全部大小

2.  qcow

    速度慢, 已被淘汰, 大小动态扩展

3.  cow

    同 qcow

4.  qcow2

    速度慢, 大小动态扩展.

5.  vmdk

    VMware 格式

6.  vdi

    VirtualBox 格式

7.  vpc

    VirtualPC 格式

8.  cloop

    pass

### -e

加密磁盘

### -c

压缩磁盘

### -6

vmdk 相关

commit
------

提交镜像更改, 这条命令需要结合 qemu-img create -b 来使用

``` bash
qemu-img commit [-f fmt ] filename
```

convert
-------

``` bash
qemu-img convert [-c] [-e] [-f fmt ] filename [-O output_fmt ] output_filename
```

转换虚拟磁盘

info
----

``` bash
qemu-img info [-f fmt ] filename
```

查看镜像信息, 会得到大小, 快照等信息.

Network
=======

USB [usb-2]
===

使用 -usbdevice 或者在 Monitor 里面用 usb~add~ 指令. 可添加的设备包括:

mouse
:   Virtual Mouse

tablet

:   

disk:file
:   Mass storage device based on file

host:bus.addr
:   直接使用 USB 设备

host:vendor~id~:product~id~
:   同上

wacom-tablet

:   

keyboard
:   Standard USB keyboard

Misc
====

运行模式
--------

### Full system emulation

模拟整个系统, 包括完整的处理器特性和各种外设.

### User mode emulation

能在 A CPU 上运行 B CPU 上编译出来的程序, 类似 Wine.
