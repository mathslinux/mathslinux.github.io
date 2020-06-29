---
layout: post
tag: QEMU_KVM
date: '\[2014-06-17 二 09:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QEMU 使用 API 直接访问各种存储后端中的镜像
---

现在的虚拟化(云计算)环境的存储技术越来越复杂, 各种存储协议层出不穷,
从古老的 NFS, 到最近流行的 *Ceph*, *GlusterFS* 等. QEMU/KVM
作为开源虚拟机的事实标准, 发展是相当地与时俱进, 对大部分存储协议,
他都有相对应的驱动直接和这些后端存储通信.
使用这些驱动来访问存储具有很多好处:
包括绕过用户空间和内核空间的数据传输层(FUSE) 提高 I/O 性能, 使得非 root
用户运行能访问这些存储设备等. 下面来做一个总结.

Ceph(RBD)
=========

优势
----

-   和 QEMU 的磁盘工具 qemu-img 完美整合
-   由于直接利用 librbd 和 Ceph 通信, 大大增加了性能.
-   通过 librbd, 可以通过调整 RBD 的缓存来达到性能调优的目的.
-   可以使用 librbd 使得虚拟磁盘拥有 TRIM 指令.

使用
----

安装客户端, 参考官方文档, 不累述.

在 Ceph 集群中先创建一个名为 qemu, 大小为 8G 的 rbd 设备 用来测试

``` bash
# rbd create --size 8192 qemu
# rbd map qemu --pool rbd
```

在启动 QEMU 的时候, 磁盘的参数为

``` bash
# -device virtio-blk-pci,drive=drive0 \
# -drive file=rbd:rbd/qemu:id=admin:key=AQDVmFJT0JnIJxAAjXG/22KnhIC91W4Gd9iJMg==,if=none,id=drive0
```

nfs
===

优势
----

-   不需要挂载 NFS, 直接连接
-   由于不需要挂载 NFS, 普通用户也可以使用

使用
----

由于该后端使用 libnfs, 需要安装以下地址提供的 libnfs:

``` bash
# git clone git://github.com/sahlberg/libnfs.git
# cd libnfs 
# ./bootstrap && ./configure && make && make install
```

使用以下指令使用

``` bash
# -device virtio-blk-pci,drive=drive0 \
# -drive file=nfs://192.168.3.155/test/test1.qcow2,if=none,id=drive0
```

iSCSI
=====

优势
----

-   不需要先登录 iSCSI 设备到本地, 有很多虚拟机和很多 iSCSI
    设备的时候很方便
-   由于上一个原因, 一定程度上增强了安全性
-   非 root 用户也可以访问 iSCSI 设备.

使用
----

``` bash
# -device lsi -device scsi-generic,drive=drive0 \
# -drive file=iscsi://192.168.3.31/iqn.qemu.test/1,if=none,id=drive0
```

http
====

优势
----

-   对于在网络上的镜像, 不需要下载直接就可以访问

使用
----

由于是承载在 http 上镜像, 不需要下载, 因此该镜像是只读的, 需要以
readonly 参数打开, 或者以 snapshot 方式打开.

``` bash
# -device virtio-blk-pci,drive=drive0 \
# -drive file=http://192.168.3.155:8000/test1.qcow2,if=none,id=drive0,snapshot=on
```

ftp
===

优势
----

同 http

使用
----

打开方式同 http, 支持传入用户名和密码, 实际上, http 和 ftp 都是使用
libcurl 这个后端来实现的.

``` bash
# -device virtio-blk-pci,drive=drive0 \
# -drive file=ftp://hao32:hao32@192.168.3.155/test1.qcow2,if=none,id=drive0,snapshot=on
```

tftp
====

优势
----

同 http

使用
----

打开方式同 http

``` bash
# -device virtio-blk-pci,drive=drive0 \
# -drive file=tftp://192.168.3.155/test1.qcow2,if=none,id=drive0,snapshot=on
```

ssh
===

优势
----

-   对于可以使用 ssh 的镜像资源, 不需要把镜像 scp 到本地直接就可以访问

使用
----

目前只支持使用 ssh-agent 的认证方式, 所以必须先设置好 ssh-agent

``` bash
# eval `ssh-agent`
# ssh-add
```

连接到 ssh 服务器上的镜像文件:

``` bash
# -device virtio-blk-pci,drive=drive0 \
# -drive file=ssh://root@192.168.3.155:22/test/test1.qcow2,if=none,id=drive0
```

host~device~
============

优势
----

可以直接打开主机上文件作为虚拟机的设备

使用
----

``` bash
# -device virtio-blk-pci,drive=drive0 -drive file=host_device:/dev/sdb,if=none,id=drive0
```

GlusterFS
=========

TODO

NBD
===

TODO

sheepdog
========

TODO
