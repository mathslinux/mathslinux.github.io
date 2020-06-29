---
layout: post
tag: Storage
date: '\[2013-10-30 三 16:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: iSCSI 杂记
---

[iSCSI](http://en.wikipedia.org/wiki/ISCSI) 是一个基于 IP
的网络存储技术, 该技术通过网络把两台计算机之间交换 传统的
[SCSI](http://en.wikipedia.org/wiki/SCSI) 命令. iSCSI 可以看作是 [Fibre
Channel](http://en.wikipedia.org/wiki/Fibre_Channel) 存储的一个替代,
由于 iSCSI 是 基于网络的, 成本上更低廉.

iSCSI 架构是 C-S 的, 分为 initiator(client) 和 target(server), initiator
会向 target 发送 SCSI 命令.

下面简单的记录一下搭建过程:

-   target: 192.168.0.166, 有一块硬盘 /dev/vdb, 不过任何处理直接 export
    给 initiator
-   initiator: 192.168.0.47

搭建 target
===========

在一些 KISS 的发行版(gentoo, archlinux) 上,
确保内核编译并加载了相应的模块

然后安装 target 用户空间的程序, 有两种程序可以选择: iscsitarget 和
tgt(scsi-target-utils). 其实两个程序都差不多, 这里我选择 tgt 来作为
例子.

``` bash
# debian 系
# apt-get install tgt

# Redhat 系
# yum install scsi-target-utils
```

配置很简单, tgt 的安装目录下也自带了一个 example 文件:
targets.conf.example.gz.

下面是我的配置:

``` bash
# cat /etc/tgt/targets.conf
<target iqn.2013-10.com.example:server.target0>
    direct-store /dev/vdb
</target>
# service tgt restart
```

查看 target 信息

``` bash
# tgtadm --mode target --op show
Target 1: iqn.2013-10.com.example:server.target0
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Readonly: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1                       <== 我们添加的 target, 可以使用的 LUN
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 10737 MB, Block size: 512
            Online: Yes
            Removable media: No
            Readonly: No
            Backing store type: rdwr
            Backing store path: /dev/vdb
            Backing store flags: 
    Account information:
    ACL information:
        ALL
```

搭建 initiator
==============

``` bash
# apt-get install open-iscsi open-iscsi-utils
```

"探测" target

``` bash
# iscsiadm -m discovery -t st -p 192.168.0.166
192.168.0.166:3260,1 iqn.2013-10.com.example:server.target0
```

显示 target 信息

``` bash
# iscsiadm -m node -o show
# BEGIN RECORD 2.0-871
node.name = iqn.2013-10.com.example:server.target0
node.tpgt = 1
node.startup = manual
......                    <== 省略
node.discovery_address = 192.168.0.166
node.discovery_port = 3260
node.discovery_type = send_targets
......
node.session.auth.authmethod = None
node.session.auth.username = <empty>
node.session.auth.password = <empty>
node.session.auth.username_in = <empty>
node.session.auth.password_in = <empty>
......
node.conn[0].address = 192.168.0.166
node.conn[0].port = 3260
node.conn[0].startup = manual
......
# END RECORD
```

登录 target

``` bash
# iscsiadm -m node --login
Logging in to [iface: default, target: iqn.2013-10.com.example:server.target0, portal: 192.168.0.166,3260]
Login to [iface: default, target: iqn.2013-10.com.example:server.target0, portal: 192.168.0.166,3260]: successful
```

该命令执行成功后, 可以在 dmesg 信息里面看到添加的磁盘信息, 就好像插入了
一块新的 SCSI 的磁盘一样, 设备文件名 "/dev/sda"

``` bash
tailf /var/log/syslog
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.546054] scsi18 : iSCSI Initiator over TCP/IP
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.804706] scsi 18:0:0:0: RAID              IET      Controller       0001 PQ: 0 ANSI: 5
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.805022] scsi 18:0:0:0: Attached scsi generic sg0 type 12
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.806690] scsi 18:0:0:1: Direct-Access     IET      VIRTUAL-DISK     0001 PQ: 0 ANSI: 5
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.806927] sd 18:0:0:1: Attached scsi generic sg1 type 0
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.811396] sd 18:0:0:1: [sda] 20971520 512-byte logical blocks: (10.7 GB/10.0 GiB)
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.813404] sd 18:0:0:1: [sda] Write Protect is off
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.813409] sd 18:0:0:1: [sda] Mode Sense: 49 00 00 08
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.814524] sd 18:0:0:1: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.819152]  sda: sda1
Oct 30 17:20:40 dunrong-virtual-machine kernel: [21180.822697] sd 18:0:0:1: [sda] Attached SCSI disk
Oct 30 17:20:41 dunrong-virtual-machine iscsid: connection17:0 is operational now
```

然后, 就可以像使用普通硬盘一样创建分区(或者LVM), 格式化……

此时, 如果再次在 target server(192.168.0.166) 上查看 target 的信息,
可以看到 initiator 的连接信息

``` bash
# tgtadm --mode target --op show
Target 1: iqn.2013-10.com.example:server.target0    <== client 的连接信息
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
        I_T nexus: 4
            Initiator: iqn.1993-08.org.debian:01:4393697be02
            Connection: 0
                IP Address: 192.168.0.47    <== client 的 IP 地址
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Readonly: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 10737 MB, Block size: 512
            Online: Yes
            Removable media: No
            Readonly: No
            Backing store type: rdwr
            Backing store path: /dev/vdb
            Backing store flags: 
    Account information:
    ACL information:
        ALL
```

使用完之后, 可以使用 logout 命令退出这个 target

``` bash
# iscsiadm -m node --logout
Logging out of session [sid: 16, target: iqn.2013-10.com.example:server.target0, portal: 192.168.0.166,3260]
Logout of [sid: 16, target: iqn.2013-10.com.example:server.target0, portal: 192.168.0.166,3260]: successful
```
