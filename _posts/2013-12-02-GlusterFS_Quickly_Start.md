---
layout: post
tag: Storage
date: '\[2013-12-02 一 10:55\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: GlusterFS 初体验
---

简介
====

[GlusterFS](http://www.gluster.org/) 是一款开源的分布式文件系统,
具有强大的 scale-out 横向扩展能力,
通过扩展能够支持PB级别的存储容量和数千客户端. 它和
[CEPH](http://ceph.com/) 在开源界
和存储界正在发起一场基于云计算的存储风暴,
向传统的存储技术([SCSI](http://en.wikipedia.org/wiki/SCSI), [Fibre
Channel](http://en.wikipedia.org/wiki/Fibre_Channel))发起 猛烈的攻击,
同时在这场战役中竞争最激烈的也是它们俩, 所以它俩既是亲密的战友,
又是敌人. 到底谁能笑道最后, 让我们拭目以待.

环境准备
========

为了体验一下 GlusterFS, 我搭建了一个基于 KVM 虚拟机的环境, 如下:

我准备了三台虚拟机, 在之上安装了 ubuntu 13.10
操作系统(其他操作系统类似), 每台虚拟机有两块 virtio 的硬盘: vda, vdb, 用
XFS 作为 brick 的文件 系统, 其它具体的情况如下:

| hostname | ip              | role   | brick            |
|----------|-----------------|--------|------------------|
| gfs1     | 192.168.176.153 | server | /data/gv0/brick1 |
| gfs2     | 192.168.176.154 | server | /data/gv0/brick1 |
| gfs3     | 192.168.176.155 | client |                  |

准备 server(在 gfs1 和 gfs2 上都执行下列操作)
---------------------------------------------

设置三台主机的 hostname

``` bash
 # emacsclient /etc/hosts
 # 最后增加
192.168.176.153 gfs1
192.168.176.154 gfs2
192.168.176.155 gfs3
```

设置 server 的 brick

``` bash
 # modprobe xfs
 # apt-get install xfsprogs # xfsprogs 包含 mount.xfs, mkfs.xfs 等
 # fdisk /dev/vdb # 将 vdb 分为一个区 vdb1
 # mkfs.xfs -i size=512 /dev/vdb1 # 将 vdb1 格式化为 XFS 文件系统

 # 把 vdb1 挂载到我们的 brick, 即 /data/gv0/brick1 下
 # mkdir -p /data/gv0/brick1
 # emacsclient /etc/fstab
 # 在最后一行增加
/dev/vdb1 /data/gv0/brick1 xfs defaults 1 2
 # mount -a
```

准备 client
-----------

设置三台主机的 hostname(同上)

安装设置
========

server
------

首先安装和启动 daemon(在 gfs1 和 gfs2 上)

``` bash
# apt-get install glusterfs-server -y
# service glusterfs-server start
```

然后用 probe 命令把对方加到 pool 里.

``` bash
# gluster peer probe gfs2 # 在 gfs1 上执行
 Probe successful
# gluster peer probe gfs1 # 在 gfs2 上执行
# gluster peer status # gfs1 上, 检查 peer 的 status
 Number of Peers: 1

 Hostname: gfs2
 Uuid: d8d3dad0-3cac-4e98-a28d-f59fb0a6c43f
 State: Peer in Cluster (Connected)
```

最后, 创建 volume(在任意一个节点上都可以执行)

``` bash
 # 两个节点, 数据保留两个副本
 # gluster volume create gv0 replica 2 gfs1:/data/gv0/brick1 gfs2:/data/gv0/brick1
 # gluster volume info
Volume Name: gv0
Type: Replicate
Status: Created
Number of Bricks: 2
Transport-type: tcp
Bricks:
Brick1: gfs1:/data/gv0/brick1
Brick2: gfs2:/data/gv0/brick1
```

client
------

client 除了 glusterfs-client, 不需要任何其他包(xfs等)

``` bash
# apt-get install glusterfs-client -y
# mount -t glusterfs gfs1:/gv0 /mnt
```

测试
====

``` bash
# cp /var/log/dmesg /mnt/ #  在 client 执行:
# 然后到 gfs1, gfs2 上, 可以看到在每个 server 上都有一个 dmesg 文件
# 这是因为上面我们创建 volume 的时候, 设置的副本数目为 2
# ls -lA /data/gv0/brick1(both server)
```

测试删除:

``` bash
# 在 client 执行:
# rm /mnt/dmesg # 然后再 gfs1, gfs2 看可以看到这些副本都删除了
```

深入一点
========

之前我们用两个节点创建了有两个副本的 volume, 每次写到 GlusterFS
同时都会写到这两个节点中.

如果节点数多于副本数, 会发生什么情况.

我们把 gfs3, 也变为 server, 用上面 setup server 的方式. 把 gfs3 的 vdb
分为 2个区, vdb1, vdb2, 分别挂载到 /data/gv0/brick1 和 /data/gv0/brick2.

``` bash
 # fdisk /dev/vdb # 分为 vdb1, vdb2
 # mkdir -p /data/gv0/brick{1,2}
 # emacsclient /etc/fstab
 # tail -n 2 /etc/fstab
/dev/vdb1 /data/gv0/brick1 xfs defaults 1 2
/dev/vdb2 /data/gv0/brick2 xfs defaults 1 2
 # mkfs.xfs -i size=512 /dev/vdb{1,2}
 # mount -a
```

在任何一个节点, 把 上面的两个 brick 加入到 gv0 这个 volume

``` bash
 # gluster volume add-brick gv0 gfs3:/data/gv0/brick1 gfs3:/data/gv0/brick2
 # gluster volume info
Volume Name: gv0
Type: Distributed-Replicate
Status: Started
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: gfs1:/data/gv0/brick1
Brick2: gfs2:/data/gv0/brick1
Brick3: gfs3:/data/gv0/brick1
Brick4: gfs3:/data/gv0/brick2
```

现在我们有 4 个 brick了, 做一个简单地测试, 在 client 我们复制 10
个文件到 gfs中, 看看各个 node 会有什么变化

``` bash
 # for i in `seq -w 1 10`; do cp -rp /var/log/syslog /mnt/copy-test-$i; done # 在 client 执行
 # ls /data/gv0/brick1/ # gfs1
copy-test-04  copy-test-05  copy-test-09
 # ls /data/gv0/brick1/ # gfs2
copy-test-04  copy-test-05  copy-test-09
 # ls /data/gv0/brick1/ # gfs3
copy-test-01  copy-test-03  copy-test-07  copy-test-10
copy-test-02  copy-test-06  copy-test-08
 # ls /data/gv0/brick2 # gfs3
copy-test-01  copy-test-03  copy-test-07  copy-test-10
copy-test-02  copy-test-06  copy-test-08
```

可以看到, 文件被放到4个brick中了, 并且总体来看, 每个文件都有两个副本.

Resources
=========

[GlusterFS
QuickStart](http://www.gluster.org/community/documentation/index.php/QuickStart)
