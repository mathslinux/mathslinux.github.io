---
layout: post
tag: OpenStack
date: '\[2013-12-06 五 01:21\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'OpenStack Step By Step(4) - setup compute node'
---

介绍
====

Compute Node 负责接受 Controller Node 对于虚拟机创建, 删除等的请求,
从名字可以看出, 它仅仅是作为一个计算节点, 只需要 nove-compute 这个 nova
的子模块.

安装配置
========

安装
----

如上所说, 我们只需要 nova 的 nova-compute 模块.

``` bash
# apt-get install nova-compute-kvm python-guestfs
```

配置
----

如前所述, nova 的配置文件位于 /etc/nova, 需要修改的是: nova.conf 和
api-paste.ini, 其实 controller node 和 compute node 的 node
配置几乎相同, 在我们的例子里, 只需要修改 my~ip~ 为 node 的真实 IP 即可.

修改 nova.conf:

``` python
[DEFAULT]
...
my_ip=192.168.176.156 # 根据你自己的情况修改
```

重启 nova-compute 服务即可.

``` bash
# service nova-compute restart
```

测试
----

在 controller node 节点中, 查看我们的 compute node 是否被 controller
所认知(看最后一行):

``` bash
# nova-manage service list
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        os-control                           internal         enabled    :-)   2013-12-05 17:12:48.567536
nova-consoleauth os-control                           internal         enabled    :-)   2013-12-05 17:12:44.650398
nova-scheduler   os-control                           internal         enabled    :-)   2013-12-05 17:12:40.872264
nova-network     os-control                           internal         enabled    :-)   2013-12-05 17:12:41.040888
nova-conductor   os-control                           internal         enabled    :-)   2013-12-05 17:12:42.623085
nova-compute     os-control                           nova             enabled    :-)   2013-12-05 17:12:42.563989
nova-compute     os-compute1                          nova             enabled    :-)   2013-12-05 17:12:44.212783
```

在 compute node 上启动虚拟机
============================

启动虚拟机的时候, nova 有自己的算法在不同的 node 间调度,
这里为了测试我们新加 的 node 能正常工作, 我们指定在该 node 上启动虚拟机.

方法很简单, 就是在启动的命令后加一个参数, –availability-zone, 后面接的值
即是上面 list 列出来的名字(nova:os-compute1)

``` bash
# 启动一个名字为 myvm2 的虚拟机, 指定在 os-compute1 上运行
# nova boot --flavor 2 --key_name mykey --image 836911d3-6e65-48d9-abf4-2e6051f3925d myvm2 \
--availability-zone nova:os-compute1
# nova list
+--------------------------------------+-------+--------+------------+-------------+-----------------------+
| ID                                   | Name  | Status | Task State | Power State | Networks              |
+--------------------------------------+-------+--------+------------+-------------+-----------------------+
| d5de16b1-b723-4161-b409-839e83c85bc9 | myvm  | ACTIVE | None       | Running     | private=192.168.22.34 |
| e6036773-fdad-4593-8797-ac3c1587c724 | myvm2 | ACTIVE | None       | Running     | private=192.168.22.35 |
+--------------------------------------+-------+--------+------------+-------------+-----------------------+
# 用 ssh 登陆进去看看是否成功
# ssh ubuntu@192.168.22.35
ubuntu@myvm2:~$ ifconfig
eth0      Link encap:Ethernet  HWaddr fa:16:3e:26:ad:21  
          inet addr:192.168.22.35  Bcast:192.168.22.63  Mask:255.255.255.224
          inet6 addr: fe80::f816:3eff:fe26:ad21/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:353430 errors:0 dropped:11 overruns:0 frame:0
          TX packets:24641 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:38239227 (38.2 MB)  TX bytes:2396159 (2.3 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Yes! That's all
