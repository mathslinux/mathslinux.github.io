---
layout: post
tag: Storage
date: '\[2013-12-18 三 15:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 整合 Openstack 和 Ceph
---

环境
====

| Hostname   | IP Address      | Roles                               |
|------------|-----------------|-------------------------------------|
| ceph1      | 192.168.176.30  | ceph-monitor/admin                  |
| ceph2      | 192.168.176.31  | ceph-osd                            |
| ceph3      | 192.168.176.32  | ceph-osd                            |
| os-control | 192.168.176.152 | controller/compute/cinder-scheduler |
| os-compute | 192.168.176.156 | compute                             |
| cinder1    | 192.168.176.156 | cinder-node                         |

为 openstack 创建 osd pool
==========================

在 ceph-admin 节点上:

``` bash
root@ceph1:~# ceph osd pool create volumes 128
root@ceph1:~# ceph osd pool create images 128
root@ceph1:~# ceph osd pool stats
pool data id 0
  nothing is going on

pool metadata id 1
  nothing is going on

pool rbd id 2
  nothing is going on

pool volumes id 3
  nothing is going on

pool images id 4
  nothing is going on
```

为 openstack 节点配置 ceph
==========================

安装软件包
----------

在 glance-node(os-control) 上:

``` bash
root@os-control:~# apt-get install -y python-ceph
```

在 cinder-node(cinder1) 上:

``` bash
root@os-compute:~# apt-get install -y python-ceph ceph-common
```

配置
----

### 配置文件

在 ceph-admin 节点把 ceph 的配置文件复制到
{glance,cinder-node}:/etc/ceph, 如果这两个节点不存在该文件夹, 先创建

``` bash
root@ceph1:~# ceph-deploy config push os-control
root@ceph1:~# ceph-deploy config push cinder1
```

### 设置认证

1.  因为我们之前的 Ceph 配置开启了 cephx 认证, 创建相应的用户来使用 Ceph
    服务.

```{=html}
<!-- -->
```
``` bash
root@ceph1:~# ceph auth get-or-create client.volumes mon 'allow r' \
osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rx pool=images'

root@ceph1:~# ceph auth get-or-create client.images mon 'allow r' \ 
osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
```

1.  然后把 client.{volumes/images} 的 keyring
    复制到相应的节点(glance-node/cinder-node)

```{=html}
<!-- -->
```
``` bash
root@ceph1:~# ceph auth get-or-create client.images | ssh os-control tee /etc/ceph/ceph.client.images.keyring
root@ceph1:~# ssh os-control sudo chown glance:glance /etc/ceph/ceph.client.images.keyring

root@ceph1:~# ceph auth get-or-create client.volumes | ssh cinder1 tee /etc/ceph/ceph.client.volumes.keyring
root@ceph1:~# ssh cinder1 chown cinder:cinder /etc/ceph/ceph.client.volumes.keyring
```

1.  配置 compute-node(os-control), 在 compute 节点上, 认证是由 libvirt
    来完成的.

```{=html}
<!-- -->
```
``` bash
# 1) 把 client.volumes 的 keyring 复制到 compute-node
root@ceph1:~# ceph auth get-key client.volumes | ssh os-control tee client.volumes.key

# 2) 在 compute-node 中, 创建 secret.xml 文件, 内容如下
<secret ephemeral='no' private='no'>
  <usage type='ceph'>
    <name>client.volumes secret</name>
  </usage>
</secret>

# 3) 把这个文件导入到 libvirt 中
# virsh secret-define --file secret.xml
3888d137-bcda-45fe-93e3-4fdd0112d0df

# 4) 把 client.volumes 的密钥导入 libvirt 中
# virsh secret-set-value --secret 3888d137-bcda-45fe-93e3-4fdd0112d0df \
--base64 $(cat client.volumes.key) && rm client.volumes.key secret.xml
```

### 设置 openstack

1.  在 glance-node(os-control) 中, 修改配置文件
    /etc/glance/glance-api.conf 的以下值

```{=html}
<!-- -->
```
``` python
default_store=rbd
rbd_store_user=images
rbd_store_pool=images
show_image_direct_url=True
```

1.  在 cinder-node(cinder1) 中, 修改 /etc/cinder/cinder.conf 的以下值

```{=html}
<!-- -->
```
``` python
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_pool=volumes
glance_api_version=2
rbd_user=volumes
rbd_secret_uuid=3888d137-bcda-45fe-93e3-4fdd0112d0df
```

1.  在各自节点重启 openstack 相关服务

```{=html}
<!-- -->
```
``` bash
# service glance-api restart
# service nova-compute restart
# service cinder-volume restart
```

使用
====

创建 images
-----------

``` bash
# 1) 用 glance 上传一个镜像, 命令和普通 glance 操作一样 ==>
# du -h /mnt/Images/ubuntu-13.10-server-cloudimg-amd64-disk1.img 
232M    /mnt/Images/ubuntu-13.10-server-cloudimg-amd64-disk1.img
# glance image-create \
--name="ubuntu-ceph-image" \
--is-public True \
--container-format bare \
--disk-format qcow2 \
--file /mnt/Images/ubuntu-13.10-server-cloudimg-amd64-disk1.img \
--progress

# 2) 查看上传的镜像
# nova image-list
+--------------------------------------+-----------------------+--------+--------+
| ID                                   | Name                  | Status | Server |
+--------------------------------------+-----------------------+--------+--------+
| 8aa5b765-1a4d-4b98-b5ad-9b6c7b58d57b | Ubuntu 13.10 cloudimg | ACTIVE |        |
| 7849c15e-9c28-49c6-97ab-d278bc27b525 | ubuntu-ceph-image     | ACTIVE |        |
+--------------------------------------+-----------------------+--------+--------+

# 3) 在 ceph 节点上查看 rados 的情况
# rbd ls images
7849c15e-9c28-49c6-97ab-d278bc27b525
# rbd info images/7849c15e-9c28-49c6-97ab-d278bc27b525
rbd image '7849c15e-9c28-49c6-97ab-d278bc27b525':
    size 231 MB in 29 objects                         <== 注意这个大小
    order 23 (8192 kB objects)
    block_name_prefix: rbd_data.10c2375b6
    format: 2
    features: layering
```

创建可启动 volumes
------------------

``` bash
# 1) 创建一个 10G 的可启动盘
# cinder create --image-id 7849c15e-9c28-49c6-97ab-d278bc27b525 --display-name boot-vol-on-ceph 10

# 2) 查看创建的磁盘信息
# cinder list
+--------------------------------------+-----------+------------------+------+-------------+----------+--------------------------------------+
|                  ID                  |   Status  |   Display Name   | Size | Volume Type | Bootable |             Attached to              |
+--------------------------------------+-----------+------------------+------+-------------+----------+--------------------------------------+
| 6f01529e-1d42-4c53-b4c7-e50d466571cc |   in-use  |     volume1      |  1   |     None    |  false   | 015ac60a-1902-4d39-b4ea-11376838872b |
| 7604e894-231b-4d86-b613-9659c7583936 |   in-use  |     volume2      |  1   |     None    |  false   | 015ac60a-1902-4d39-b4ea-11376838872b |
| bd3a9b3a-64c8-4865-8f55-32dfc683ff67 | available | boot-vol-on-ceph |  10  |     None    |   true   |                                      |
+--------------------------------------+-----------+------------------+------+-------------+----------+--------------------------------------+

# 3) 查看 rbd 的信息
# rbd ls volumes
volume-bd3a9b3a-64c8-4865-8f55-32dfc683ff67
# rbd info volumes/volume-bd3a9b3a-64c8-4865-8f55-32dfc683ff67
rbd image 'volume-bd3a9b3a-64c8-4865-8f55-32dfc683ff67':
    size 10240 MB in 2560 objects                           <== 创建的大小 10G
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.10e02ae8944a
    format: 2
    features: layering
```

启动
----

``` bash
# nova boot --flavor 2 --key_name mykey --block_device_mapping vda=bd3a9b3a-64c8-4865-8f55-32dfc683ff67:::0 myvm-on-ceph
# nova list
+--------------------------------------+--------------+--------+--------------+-------------+-----------------------+
| ID                                   | Name         | Status | Task State   | Power State | Networks              |
+--------------------------------------+--------------+--------+--------------+-------------+-----------------------+
| 015ac60a-1902-4d39-b4ea-11376838872b | myvm         | ACTIVE | None         | Running     | private=192.168.22.34 |
| 10309940-f56a-4c64-9910-561e29fb7b81 | myvm-on-ceph | ACTIVE | None         | Running     | private=192.168.22.36 |
+--------------------------------------+--------------+--------+--------------+-------------+-----------------------+
```

查看 QEMU 的命令行信息, 在 compute-node 节点

``` bash
# ps aux | grep '[1]9537' | awk -F " -" '{ i=1;while(i<NF) {print NF,$i;i++}}' | grep 'drive file'
32 drive file=rbd:volumes/volume-bd3a9b3a-64c8-4865-8f55-32dfc683ff67:id=volumes:key=AQBkJK9SOA1UDBAAUqjfU6SvWzEv/w7ZH8nBmg==:auth_supported=cephx\;none:mon_host=192.168.176.30\:6789\;192.168.176.31\:6789\;192.168.176.32\:6789,if=none,id=drive-virtio-disk0,format=raw,serial=bd3a9b3a-64c8-4865-8f55-32dfc683ff67,cache=none
```

可以看到, QEMU 已经使用了 rbd 的 url 来读写虚拟机镜像了.

其它
====

如果配置多个 {compute,cinder}-node,
按照上面各自节点的配置做相应的配置即可.
