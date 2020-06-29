---
layout: post
tag: Storage
date: '\[2013-10-30 三 19:15\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: LVM Thin Provision
---

之前对 LVM 做过一番[简单的探索](http://mathslinux.org/?p%3D322),
但其实研究的还不够, 真正在使用的时候 LVM 不太能 满足我的需求,
比如我需要的 [Thin
Provision](http://en.wikipedia.org/wiki/Thin_provisioning)
功能就没法实现. 也就是说一般情况下, 我们使用的 LVM 是不支持 Thin
Provision 的, LV 在创建之后, 大小就固定了, 虽然之后可以手动增加和减小 PE
的数量, 但是在一些场景中会带来更复杂的问题.

比如, 考虑用 LV 作为虚拟机的镜像存储方式, 我们知道很多虚拟磁盘(比如
qcow2, qde, vdi) 是支持 copy-on-write 的,
即虚拟磁盘在创建的时候只需要分配非常小的空间, 随着 用户对该磁盘的写操作,
"按需" 的扩大实际占用空间. 这样做的好处是明显的: 可以创
建比真实设备允许的更多的虚拟磁盘, 即实现了 Thin Provision.

但是, 由于 LV 的限制, 在磁盘存在的那一刹那, 除非以后用户手动 resize
磁盘, 磁盘 的实际占用大小确定了, 也就是说 copy-on-write 其实已经失效了.

为了解决这个问题, 不同的虚拟化平台采用不同的办法, 比如创建虚拟镜像的 LV
的时候, 只分配很少的 PE, 然后监控该 LV 的使用情况(通过 guest agent
或者其他工具), 一旦该 LV "快满了", 则调用 lvm 的工具 extend 该 LV.
但这样同样存在问题:

-   监控 LV 可能消耗过多资源, 如果监控程序和 lvm
    的管理程序不在一台节点上, 那么对网络带宽的消耗是不可忽视的,
    还有监控间隔? 什么时候认为 LV "快满了"等等
-   如果 LVM 的管理程序还没有来得及对磁盘进行扩充, 而 guest 的写入速度又
    太快, 那么必然发生可用空间不够的错误 (errno=ENOSPC // not enough
    space)
-   以上讨论的整个架构过于复杂, 太多因素需要考虑. e.g. 监控间隔,
    数据包结构 …

归根结底, 上面的缺陷其实就是由于 LVM 不能 Thin Provison 导致的.

情况在 [LVM 的 2.02.89
版本](http://www.redhat.com/archives/linux-lvm/2012-January/msg00018.html)
开始变好了, 从这个版本开始, LVM 开始支持一种叫 "thin pool" 的特性,
该特性可以允许用户创建一个类型为 thin-pool 的 LV, 这个 LV 其实是一个 LV
pool, 基于这个 LV pool 之上创建的 LV 就具有了 Thin Provison 的特性,
也就是 LV 池里面的 LV 也可以 copy-on-write 了.

下面来体验一下: 一个 10G 的磁盘用来作为 LVM 的实验环境的 PV

首先像之前 [LVM 杂记](http://mathslinux.org/?p%3D322) 中提到的, 建立 PV,
VG

``` bash
# pvcreate /dev/vdb 
vgcreate test-vg /dev/vdb
# vgdisplay | grep Free
  Free  PE / Size       2559 / 10.00 GiB
```

创建一个类型 thin-pool, 名字为 test-pool 的 特殊的 LV, 大小为 8G,
其实就是 一个 LV pool

``` bash
# lvcreate --size 8G --type thin-pool --thinpool test-pool test-vg
  Logical volume "test-pool" created
```

查看这个 thin-pool 的状态, 因为 thin-pool 是一个特殊的 LV, 所以可以用
lvs 查看, 可以看到和没有使用 thin pool 时候的变化,
真实地空间占用也显示出来了, 为 0%

``` bash
# lvs
  LV        VG      Attr     LSize Pool Origin Data%  Move Log Copy%  Convert
  test-pool test-vg twi-a-tz 8.00g               0.00    
```

好, 开始重头戏, 创建一个基于 thin pool 的 LV, 名字为 test-lv, 大小是 4G

``` bash
# lvcreate -V4G -T test-vg/test-pool --name test-lv
```

看一下现在各个 LV 的情况, 可以看到我们的 test-lv 显示的是 4G, 实际占用为
0%, thin pool 仍然为 8G, 实际占用还是 0%. (因为没有数据 write,
所以也没有数据 copy)

``` bash
# lvs
  LV        VG      Attr     LSize Pool      Origin Data%  Move Log Copy%  Convert
  test-lv   test-vg Vwi-a-tz 4.00g test-pool          0.00                        
  test-pool test-vg twi-a-tz 8.00g                    0.00      
```

向 test-lv 填充一些数据, 看看 copy-on-write 怎么工作

``` bash
# dd if=/dev/zero of=/dev/test-vg/test-lv bs=1M count=1024 && sync
root@cloudtimes:~# lvs
  LV        VG      Attr     LSize Pool      Origin Data%  Move Log Copy%  Convert
  test-lv   test-vg Vwi-a-tz 4.00g test-pool         25.00                        
  test-pool test-vg twi-a-tz 8.00g                   12.50                        
```

可以看到, 当我们写了 1G 的数据后, test-lv 的真实空间占用变大了
25%(1G/4G), thin pool 的空间也相应地变为 12.5%(1G/8G),
这些数值是随着写入的内容而变化的, 并不像之前那样, 我们需要用 LVM
工具干预. 真正的 Thin Provison!

下面, 为了做对比, 创建一个正常的 LV, 来看看和基于 thin pool 的 LV 的区别

``` bash
#lvcreate -l 50 -n normal-lv test-vg
# lvs
  LV        VG      Attr     LSize   Pool      Origin Data%  Move Log Copy%  Convert
  normal-lv test-vg -wi-a--- 200.00m                                         <== 仔细看这行和下一行的区别
  test-lv   test-vg Vwi-a-tz   4.00g test-pool         25.00
  test-pool test-vg twi-a-tz   8.00g                   12.50
# dd if=/dev/zero of=/dev/test-vg/normal-lv bs=1M count=100 && sync
# lvs
  LV        VG      Attr     LSize   Pool      Origin Data%  Move Log Copy%  Convert
  normal-lv test-vg -wi-a--- 200.00m                                         <== 仔细看这行和下一行的区别
  test-lv   test-vg Vwi-a-tz   4.00g test-pool         25.00                        
  test-pool test-vg twi-a-tz   8.00g                   12.50           
```

可以看到, 正常的 LV 并不能反映出到底用户真实写了多少数据, 因为不需要嘛,
它一被创建伊始就被赋予了他要求的所有空间, 自然达不到 Thin
Provison(节省空间) 的效果.
