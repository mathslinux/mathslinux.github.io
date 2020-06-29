---
layout: post
tag: Storage
date: '\[2014-04-03 四 00:25\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Ceph 中的网络配置
---

Ceph 是分布式的文件系统, 由于它是网络文件系统,
它的性能不仅仅受限于物理磁盘的 读写带宽(IOPS), 好的网络架构对发挥 Ceph
的性能也至关重要.

在这里, 我不去详细探讨 Ceph 内部的架构细节(在另外文章我会详细分析),
仅仅通过 Ceph 的配置文件和一些测试来简单研究一下 Ceph 的网络配置.

Ceph 里的数据同步
=================

Ceph 里面的数据同步包括 Monitor 之间的增量同步配置(CRUSH啊这些), osd
之间 的数据冗余. 为了让 Ceph 的性能发挥到最好, 对于数据的传输, Ceph
定义了 public network 和 cluster network, public network
是外部访问用到的 网络(fuse client, rbd client), cluster network 为了
Ceph 内部组件的 数据同步(monitor, osd).

Ceph 的网络配置
===============

前面说过, Ceph 区分外部访问的数据和内部集群使用的数据,
用两个不同的网络配置来 区分: `public addr` 和 `cluster addr`.

一般来说, 生产环境中的服务器都有两块以上的网卡, 配置的时候,
一块网卡主要来跑 Ceph 的 对外的数据传输服务(保证足够的 IOPS),
另一块网卡来做集群内部的数据同步.

简单测试
========

这里的测试仅仅是为了测试上面说过的理论, 不涉及其它 Ceph 的内容.
为了方便计, 只用了一台机器同时跑 monitor 和
osd(这在生产环境中是非常不推荐的).

你可以用我之前的文章讲到的 [Ceph 初体验](http://mathslinux.org/?p%3D441)
来搭建测试环境.

附上我的配置文件(注意我的 osd.0 的配置):

``` python
root@ceph1:~# cat /etc/ceph/ceph.conf 
[global]
fsid = 382f9965-75ad-403a-b063-4d1fa3e9fb52
mon_initial_members = ceph1
mon_host = 192.168.3.30
auth_supported = cephx
osd_journal_size = 1024
filestore_xattr_use_omap = true

[osd.0]
public_addr = 192.168.3.33
cluster_addr = 192.168.3.30
```

下面的图是该配置在 client 下用 iftop 截到的图, 可以看到数据的传输 走的是
192.168.3.33 这个网络. ![](/images/posts/Storage/ceph-iftop1.png)

改变上面的 osd.0 配置为下面的内容

``` python
[osd.0]
public_addr = 192.168.3.30
cluster_addr = 192.168.3.33
```

用 iftop 可以看到, 数据传输的接口果然变了. ![](/images/posts/Storage/ceph-iftop2.png)

总结
====

Ceph 的网络配置大体就这么两个参数, 当然使用上要比上面讲的灵活一点.
灵活配置 Ceph 的网络参数可以满足性能上的要求,
甚至在网络带宽远远小于磁盘的 I/O 带宽的 时候(在生产环境中比较少见),
可以给每一个 osd 配置一个网络出口达到最好的性能.

更多具体详细的讨论请参考 Ceph 官网上
[网络配置文档](http://ceph.com/docs/master/rados/configuration/network-config-ref/).
