---
layout: post
tag: OpenStack
date: '\[2013-12-08 日 19:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'OpenStack Step By Step(5) - setup cinder'
---

介绍
====

Cinder 是 OpenStack 的块设备存储模块, 是从 nova 的 nova-volume
组件演化而来的. 它提供块设备存储服务, 为 nova
提供可被虚拟机使用的块设备, nova 可以通过 cinder 创建 block 设备,
为虚拟机挂载这些设备, 也可以 对块设备执行删除, 从虚拟机分离等操作.

安装部署
========

cinder 的部署分为两部分, cinder 的控制节点和存储节点. 在本次部署中,
控制节点在之前的 openstack 控制节点上, 存储节点在之前的计算节点上.
这样做只是为了方便, 实际上, 这两个节点可以和之前的模块分离.

| role                | hostname    | ip address      |
|---------------------|-------------|-----------------|
| cinder controller   | os-control  | 192.168.176.152 |
| cinder storage node | os-compute1 | 192.168.176.156 |

另外, 为了方便, 之前我部署 openstack 的时候使用的是 sqlite 数据库,
但是在搭建 cinder 多个节点的时候遇到了问题. 我将数据库从 sqlite 改为
mysql 后 该问题消失了了. 所以本次搭建开始使用 mysql 作为数据库. 将
sqlite 改为 mysql 很简单, 只需要将前文配置文件中数据库配置修改改为 mysql
即可, 其它所有一切的配置都相同.

``` python
connection = sqlite:////var/lib/nova/nova.sqlite
```

改为:

``` python
connection = mysql://nova:linux@os-control/nova
```

并创建相关的数据库, 然后同步数据库即可(根据你的配置文件修改对应的密码):

``` python
# mysql -u root -p
mysql> CREATE DATABASE nova;
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'linux';
# nova-manage db sync
# 重启服务
```

当然, 如果你用 sqlite 也能成功部署多个 storage node,
那么此处不需要做修改.

安装控制节点
------------

``` bash
# apt-get install -y cinder-api cinder-scheduler
```

修改 /etc/cinder/cinder.conf, 增加以下几项:

``` python
[DEFAULT]
...
rpc_backend = cinder.openstack.common.rpc.impl_kombu
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = RABBIT_PASS
...

[database]
connection = mysql://cinder:linux@localhost/cinder
```

调整 /etc/cinder/api-paste.ini 中的 \[filter:authtoken\], 使其变为:

``` python
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = os-control
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = cinder
admin_password = geniux
```

创建并同步数据库:

``` bash
# mysql -u root -p
mysql> CREATE DATABASE cinder;
mysql> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'linux';
# rm /var/lib/cinder/cinder.sqlite
# cinder-manage db sync
```

创建并配置 cinder user, 创建 cinder 的 endpoint, 由于我们之前在
endpoint.sh 已经创建了一个 volume 的 endpoint, 这里只需要创建一个 V2
版本的即可.

``` bash
# keystone user-create --name=cinder --pass=geniux --tenant=service --email=cinder@example.com
# keystone user-role-add --user=cinder --tenant=service --role=admin
# keystone service-create --name=volume --type=volume2 --description="Cinder Volume Service V2"
  keystone endpoint-create \
  --service-id=the_service_id_above \
  --publicurl=http://controller:8776/v2/%\(tenant_id\)s \
  --internalurl=http://controller:8776/v2/%\(tenant_id\)s \
  --adminurl=http://controller:8776/v2/%\(tenant_id\)s
```

一切就绪, 重启 cinder 服务:

``` bash
# service cinder-scheduler restart
# service cinder-api restart
```

安装存储节点
------------

### 安装 cinder 相关包

``` bash
# apt-get install -y cinder-volume lvm2 tgt
```

### 设置 hosts

在该步骤, 需要同时在 controller node 和 storage node 上设置 /etc/hosts
文件, 使得二者可以互相看到对方的 hostname. 由于我们之前已经在配置 nova
模块的时候配置过了, 这里可以省略.

### lvm setup

由于我们这里使用 lvm + iscsi 作为存储后端, 所以需要设置 lvm, 注意我的
storage 有两块硬盘: vda 和 vdb, 这里 vdb 分配给 cinder 使用.

LVM 的相关概念及操作见我之前的 [LVM
杂记](http://mathslinux.org/?p%3D322).

``` bash
# pvcreate /dev/vdb
# vgcreate cinder-volumes /dev/vdb
```

1.  setup cinder

cp controller/etc/cinder/cinder.conf\|api-paste.ini to /etc/cinder
change .ini host service cinder-volume restart service tgt restart

### 设置 cinder

将 controller 的 /etc/cinder 下的 cinder.conf 和 api-paste.ini
文件复制过来, 覆盖即可. 还要注意 cinder.conf 的数据库连接路径不要写
localhost, 而要写 os-control

重启 cinder 和 iscsi 服务即可:

``` bash
# service cinder-volume restart
# service tgt restart
```

测试
----

在 controller 节点上查看 cinder 的节点(可以看到 os-compute1
被加入控制了):

``` bash
# cinder-manage host list
host                        zone           
os-control                  nova           
os-compute1                 nova   
```

使用
----

前面说过, nova 利用 cinder 为虚拟机提供块设备服务. 下列操作展示了怎么给
虚拟机新增一个磁盘设备.

### 创建 volume

``` bash
# nova volume-create --display_name "volume1" 1
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| status              | creating                             |
| display_name        | volume1                              |
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| created_at          | 2013-12-07T20:25:27.648067           |
| display_description | None                                 |
| volume_type         | None                                 |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| size                | 1                                    |
| id                  | 6f01529e-1d42-4c53-b4c7-e50d466571cc |
| metadata            | {}                                   |
+---------------------+--------------------------------------+
root@os-control:/var/log/nova# nova volume-list
+--------------------------------------+-----------+--------------+------+-------------+-------------+
| ID                                   | Status    | Display Name | Size | Volume Type | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+-------------+
| 6f01529e-1d42-4c53-b4c7-e50d466571cc | available | volume1      | 1    | None        |             |
+--------------------------------------+-----------+--------------+------+-------------+-------------+
```

PS: volume 也可以用 cinder 直接操作

### 分配给虚拟机

将 id 为 6f01529e-1d42-4c53-b4c7-e50d466571cc 的磁盘分配给 myvm,
设备名称为 /dev/vdb

``` bash
# nova volume-attach myvm 6f01529e-1d42-4c53-b4c7-e50d466571cc /dev/vdb
```

去我们的虚拟机里面看看实际的情况:

``` bash
# ssh ubuntu@192.168.22.34
ubuntu@myvm:~$ cat /proc/partitions 
major minor  #blocks  name

 253        0   20971520 vda
 253        1   20970496 vda1
ubuntu@myvm:~$ sudo fdisk -l /dev/vdb

Disk /dev/vdb: 1073 MB, 1073741824 bytes
16 heads, 63 sectors/track, 2080 cylinders, total 2097152 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/vdb doesn't contain a valid partition table
```

在虚拟机里面, 就可以像操作普通磁盘一样操作了.

进一步, 我们看看这个 volume 在 host 上的细节.

这个 volume 是在 os-compute1 上的 一个 LV, 由 ISCSI 协议被连接到本地:

在 controller node 上看看 ISCSI target 的信息:

``` bash
root@os-control:~# iscsiadm -m node -o show | grep "node.name\|node.conn\[0\].address"
node.name = iqn.2010-10.org.openstack:volume-6f01529e-1d42-4c53-b4c7-e50d466571cc
node.conn[0].address = 192.168.176.156
```

在 storage node 上看看 LVM 和 ISCSI 的相关信息:

``` bash
root@os-compute1:/etc/cinder# lvdisplay 
  --- Logical volume ---
  LV Path                /dev/cinder-volumes/volume-6f01529e-1d42-4c53-b4c7-e50d466571cc
  LV Name                volume-6f01529e-1d42-4c53-b4c7-e50d466571cc
  VG Name                cinder-volumes
  LV UUID                yj3I0y-DPMg-AZxh-pcUx-A1jy-aJQa-PcKiMg
  LV Write Access        read/write
  LV Creation host, time os-compute1, 2013-12-08 04:25:27 +0800
  LV Status              available
  # open                 1
  LV Size                1.00 GiB
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:2

# cat /var/lib/cinder/volumes/volume-6f01529e-1d42-4c53-b4c7-e50d466571cc 
<target iqn.2010-10.org.openstack:volume-6f01529e-1d42-4c53-b4c7-e50d466571cc>
    backing-store /dev/cinder-volumes/volume-6f01529e-1d42-4c53-b4c7-e50d466571cc
</target>
```

Resource
========

[Add the Block Storage
Service](http://docs.openstack.org/havana/install-guide/install/apt/content/ch_cinder.html)
