---
layout: post
tag: OpenStack
date: '\[2013-12-04 三 16:55\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'OpenStack Step By Step(3) - setup nova'
---

介绍
====

Nova 是 Openstack 的计算模块, 是最核心的组件.
它对上提供包括虚拟机的创建, 启动, 删除, 虚拟磁盘的挂载/卸载等服务,
对下它封装各种不同的 hypervisor 的 API, 使得整个 OpenStack 对不同的
hypervisor(qemu/kvm, xen, virtualbox) 都有统一的 API 接口.

PS: 以下配置在这台机器上同时安装 controller service 和 compute
service(all in one)

安装配置
========

安装
----

``` bash
# apt-get install -y nova-api nova-cert nova-common nova-compute \
nova-compute-kvm nova-doc nova-network nova-objectstore nova-scheduler \
nova-volume nova-consoleauth novnc python-nova python-novaclient \
nova-conductor rabbitmq-server
```

配置
----

### 修改配置文件

nova 的配置文件位于 /etc/nova, 目前我们需要修改的是: nova.conf 和
api-paste.ini

1.  nova.conf

    在 \[DEFAULT\]后面, 添加

    ``` python
    auth_strategy=keystone # 使用 keystone 认证
    rpc_backend = nova.rpc.impl_kombu
    rabbit_host = 192.168.176.152 # rabbitmq server 的地址
    flat_injected=true
    # 以下是网络设置
    network_manager=nova.network.manager.FlatDHCPManager
    fixed_range=192.168.22.32/27 # 设置虚拟机内部 IP 的地址范围
    floating_range=192.168.176.160/27 # 设置浮动IP的范围, 该IP用于和外部通信
    flat_network_dhcp_start=192.168.22.33 # 设置内部DHCP的地址段
    flat_network_bridge=br100 # 设置网桥名称
    flat_interface=eth1 # 设置内部网络走的网卡
    public_interface=eth0 # 设置外部网络走得网卡, 用于浮动 IP 的发放
    my_ip=192.168.176.152 # 本节点的 IP
    glance_host=192.168.176.152 # glance 的地址
    ```

    添加两个段: database, keystone~authtoken~

    ``` python
    [database]
    sql_connection=sqlite:////var/lib/nova/nova.sqlite

    [keystone_authtoken]
    auth_host = 192.168.176.152 
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service # 参考前面 keystone 的配置
    admin_user = nova
    admin_password = geniux
    ```

2.  api-paste.ini

    将:

    ``` python
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    auth_host = 127.0.0.1
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = %SERVICE_TENANT_NAME%
    admin_user = %SERVICE_USER%
    admin_password = %SERVICE_PASSWORD%
    ```

    修改为:

    ``` bash
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    auth_host = 127.0.0.1
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service # 参考前面 keystone 的配
    admin_user = nova
    admin_password = geniux
    ```

### 创建数据库

清空 apt-get 安装时默认创建的数据库, 然后手动创建一个

``` bash
# rm /var/lib/nova/nova.sqlite
# nova-manage db sync
# chown nova:nova /var/lib/nova/nova.sqlite
# for a in nova-compute nova-network nova-api nova-cert nova-consoleauth nova-scheduler nova-conductor; do service "$a" restart; done
```

### 网络配置

创建一个虚拟机使用的内部 IP 段和浮动IP段

``` bash
# 创建内部IP段
# nova-manage network create private --fixed_range_v4=192.168.22.32/27 \
--num_networks=1 --bridge=br100 --bridge_interface=eth1 --network_size=32
# 创建浮动 IP 段
# nova-manage floating create --ip_range=192.168.176.160/27
```

测试
----

列出可用的镜像

``` bash
# nova image-list
+--------------------------------------+-----------------------+--------+--------+
| ID                                   | Name                  | Status | Server |
+--------------------------------------+-----------------------+--------+--------+
| 836911d3-6e65-48d9-abf4-2e6051f3925d | Ubuntu 13.10 cloudimg | ACTIVE |        |
+--------------------------------------+-----------------------+--------+--------+
```

启动虚拟机
==========

接下来, 就可以启动第一台虚拟机了.

### 导入密钥

由于我上面没有配置 vnc, 我通过 ssh 的方式访问启动的虚拟机. 如果 ssh 密钥
被虚拟机验证通过的话, 就可以直接访问了. 聪明的 openstack 会为我们做这件
事情, 在虚拟机创建的时候, 它会把你给它的 ssh 密钥注入进虚拟机的相应位置.

``` bash
# ssh-keygen # 如果没有 ssh 密钥(~/.ssh/id_rsa 不存在), 创建一个
# nova keypair-add --pub_key id_rsa.pub mykey # 导入密钥, 命名为 mykey
# nova keypair-list
+-------+-------------------------------------------------+
| Name  | Fingerprint                                     |
+-------+-------------------------------------------------+
| mykey | 5d:cc:62:1d:3a:1c:18:f4:b4:a3:69:e3:bc:79:2b:2f |
```

### 设置安全组

允许 ssh 和 ping 我们的虚拟机.

``` bash
# nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
# nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```

### 选择规格

openstack 的规格(flavor), 指的是你给虚拟机指定的硬件参数, 包括内存大小,
CPU 数量, 磁盘大小等等.

首先来看看 openstack 默认给用户提供了什么样的规格

``` bash
# nova flavor-list
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

如果觉得不满意, 还可以自己创建/删除规格, 比如, 创建一个 1G 内存,
10G硬盘, 双核 CPU 的规格.

``` bash
# nova flavor-create --is-public true myflavor 6 1024 10 2
+----+----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name     | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+----------+-----------+------+-----------+------+-------+-------------+-----------+
| 6  | myflavor | 1024      | 10   | 0         |      | 2     | 1.0         | True      |
+----+----------+-----------+------+-----------+------+-------+-------------+-----------+
# nova flavor-list
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
| 6  | myflavor  | 1024      | 10   | 0         |      | 2     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

### 启动

通过上面的设置, 启动虚拟机就恨简单了, 下面的参数都是自解释的.

``` bash
# nova boot --flavor 2 --key_name mykey --image 836911d3-6e65-48d9-abf4-2e6051f3925d myvm
# nova list
+--------------------------------------+-------+--------+------------+-------------+-----------------------+
| ID                                   | Name  | Status | Task State | Power State | Networks              |
+--------------------------------------+-------+--------+------------+-------------+-----------------------+
| d5de16b1-b723-4161-b409-839e83c85bc9 | myvm  | ACTIVE | None       | Running     | private=192.168.22.34 |
+--------------------------------------+-------+--------+------------+-------------+-----------------------+
# ps aux | grep qemu # 查看 qemu 的运行参数
# ssh ubuntu@192.168.22.34 # ssh 登陆虚拟机
ubuntu@myvm:~$ df -h # 查看磁盘状态
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        20G  853M   18G   5% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
udev            997M   12K  997M   1% /dev
tmpfs           201M  336K  200M   1% /run
none            5.0M     0  5.0M   0% /run/lock
none           1002M     0 1002M   0% /run/shm
none            100M     0  100M   0% /run/user
ubuntu@myvm:~$ free -m # 查看内存状态
             total       used       free     shared    buffers     cached
Mem:          2002        429       1572          0         65        295
-/+ buffers/cache:         68       1933
Swap:            0          0          0
```

Resource
========

[Openstack 的官方文档 – Configure Compute
services](http://docs.openstack.org/havana/install-guide/install/apt/content/ch_nova.html)
