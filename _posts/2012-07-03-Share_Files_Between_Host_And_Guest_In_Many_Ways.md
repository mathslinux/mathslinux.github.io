---
layout: post
tag: QEMU_KVM
date: '\[2012-07-03 二\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 在主机和虚机之间共享文件的N中方法
---

以下是我常用的在主机的虚拟机间通信(共享文件)的常用方法(未完待续):

通过内建的 Samba 服务器
=======================

很少有人知道 QEMU 内置了一个 Samba server, 按如下方式启动 QEMU, 即可启用
它.

``` bash
$ qemu-kvm -net nic -net user,smb=shared_directory ~/Image/XP.img
```

或者在旧的 QEMU 中版本中.(未来可能被失效)

``` bash
$ qemu-kvm -smb shared_directory ~/Image/XP.img
```

之后 GUEST 就可以用 10.0.2.4(默认) 上访问这个文件夹, 例如在 Windows
Explorer 中, 可以用 `\\10.0.2.4`, 在 gnome nautilus 中可以用
`smb://10.0.2.4` 访问.

更过的细节:

``` c
static int slirp_smb(SlirpState* s, const char *exported_dir,
                     struct in_addr vserver_addr)
{
    // ......
    /* s->smb_dir: "/tmp/qemu-smb.QEMUPID-0" */
    mkdir(s->smb_dir, 0700);

    /* smb_conf: "/tmp/qemu-smb.QEMUPID-0/smb.conf" */
    f = fopen(smb_conf, "w");
    fprintf(f,
            "[global]\n"
            "private dir=%s\n"
            "socket address=127.0.0.1\n"
            "pid directory=%s\n"
            "lock directory=%s\n"
            "state directory=%s\n"
            "log file=%s/log.smbd\n"
            "smb passwd file=%s/smbpasswd\n"
            "security = share\n"
            "[qemu]\n"
            "path=%s\n"
            "read only=no\n"
            "guest ok=yes\n",
            s->smb_dir,
            s->smb_dir,
            s->smb_dir,
            s->smb_dir,
            s->smb_dir,
            s->smb_dir,
            exported_dir
            );
    fclose(f);

    /* smb_cmdline: "smbd -s /tmp/qemu-smb.QEMUPID-0/smb.conf" */
    snprintf(smb_cmdline, sizeof(smb_cmdline), "%s -s %s",
             CONFIG_SMBD_COMMAND, smb_conf);

    slirp_add_exec(s->slirp, 0, smb_cmdline, &vserver_addr, 139);

    // ...... 
}
```

通过主机的 Samba Server
=======================

这种方式很简单, 把主机的 Samba server 打开, 在 Guest 里面就可以用
10.0.2.2 访问

``` bash
$ /etc/init.d/samba start
```

通过主机SSH
===========

同上, 在 Guest 中, 用 ssh 协议访问 10.0.2.2(Host IP) 这个 IP 地址

通过客户机 SSH(主机的端口转发)
==============================

``` bash
$ qemu-kvm -m 1024 -redir tcp:3456::22 ~/Image/ArchLinux.img
```

QEMU 将会在 3456 端口监听, 收到数据后把所有端口转发到 Guest 的 22 端口.

在 Guest 上启动 ssh server, 然后在主机上就可以用下列的指令访问了.

``` bash
$ ssh IP_Address_Of_Host -p 3456 
```

qemu-nbd
========

qemu-nbd 是一个能使用
[NBD](http://en.wikipedia.org/wiki/Network_block_device) 协议将 QEMU
Image 导出的工具.

加载 nbd 驱动
-------------

某些版本的 linux 不加 `max_part` 参数会导致没有没有设备节点
`/dev/nbd0p{1,2,3,4...}` 等. 用 kpartx 也不行.

``` bash
$ sudo modprobe nbd max_part=8 
```

连接 qemu-nbd
-------------

``` bash
$ sudo qemu-nbd -c /dev/nbd0 path/to/image/file # 注意要写绝对路径
$ fdisk -l /dev/nbd0 # 列出分区类型
$ mount /dev/nbd0p3 /mnt # 挂载虚拟磁盘的分区到本机
```

libgustfs
=========

[libgustfs](http://libguestfs.org/)
是一个想要一统天下的虚拟机镜像查看/修改工具, 号称支持几乎所有
类型的虚拟机镜像, 在它面前 qemu-nbd 弱爆了.
分析它显然超出了这篇文档的范围. 过后将会专门写一篇文章来分析它,
以下简单的提供一种利用 libguestfs 来访问 虚拟机镜像的方法.

首先, 安装它

``` bash
# 在 Gentoo中, 使用
$ sudo emerge libguestfs
# 在 redhat 家族的发行版中, 使用
$ yum install libguestfs libguestfs-tools*
```

然后使用以下指令启动 guestfs 的命令行,

``` bash
# --rw 参数表示挂载后对镜像具有读写权限, 可能有点慢, 需要
# 等待, 注意对运行中的虚拟机, 有必要使用 --ro 只读挂载
$ guestfish --rw -i -a path/to/image/file

# 创建一个临时挂载点, guestfs 的命令行接口可以执行命令
# 命令前面加 ! 就可以了
$ !mkdir /tmp/mnt

# 导出镜像文件到 /tmp/mnt, 导出后需要执行 mount-local-run
$ mount-local /tmp/mnt 

# 进入 mount loop, 类似于 glib 的 g_main_loop_run
# 卸载镜像后这个 loop 自动结束
$ mount-local-run
```

打开另外一个终端, 在 /tmp/mnt 里面就可以看到导出的文件了,
由于是读写方式的挂载, 可以在里面像在本地一样的读写文件, 完成后
使用下面指令卸载

``` bash
$ fusermount -u /tmp/mnt
```

执行后, 上面的 loop 就会结束

启动虚拟机后, 所有的更改都会生效.
