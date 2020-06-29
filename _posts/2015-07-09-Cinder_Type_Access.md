---
layout: post
tag: OpenStack
date: '\[2015-07-09 四 14:15\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Cinder 的私有卷类型功能的使用
---

数据库升级
==========

如果是在之前部署的 Juno 上升级的，通过 backport
改功能来实现的，就使用下列指令 升级一下数据库。

``` bash
[root@localhost ~]# cinder-manage db sync
```

使用
====

下面以一个例子来说明一下怎么使用该功能. 使用的后端存储是 lvm 和 ceph
集群里面的 pool。

基本配置
--------

| 用户  | 租户    | 卷类型        | 后端卷类型 |     |
|-------|---------|---------------|------------|-----|
| demo1 | tenant1 | volume~type1~ | rbd1       |     |
| demo2 | tenant2 | volume~type2~ | rbd2       |     |

### 创建租户

为了比较，这里创建两个租户 demo1 和 demo2 和两个用户 demo1(属于
tenant1), demo2(属于 tenant2):

创建两个租户 tenant1 和 tenant2:

``` bash
[root@localhost ~]# keystone tenant-create --name tenant1
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | 38132b7bf32b434398862971c567bca3 |
|     name    |             tenant1              |
+-------------+----------------------------------+
[root@localhost ~]# keystone tenant-create --name tenant2
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | 3fb70cdab54c4322a277f9acaa360e71 |
|     name    |             tenant2              |
+-------------+----------------------------------+
```

再分别创建两个用户 demo1 和 demo2:

``` bash
[root@localhost ~]# keystone user-create --name demo1 --tenant tenant1 --pass demo1
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 2e69d91577e949509ce20069786526c5 |
|   name   |              demo1               |
| tenantId | 38132b7bf32b434398862971c567bca3 |
| username |              demo1               |
+----------+----------------------------------+
[root@localhost ~]# keystone user-create --name demo2 --tenant tenant2 --pass demo2
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 7687d7a33e49464cb7cedfbf20e5781c |
|   name   |              demo2               |
| tenantId | 3fb70cdab54c4322a277f9acaa360e71 |
| username |              demo2               |
+----------+----------------------------------+
```

### 创建 rbd pool

创建 pool rbd1, rbd2

``` bash
[root@localhost ~]# ceph osd pool create rbd1 128
pool 'rbd1' created
[root@localhost ~]# ceph osd pool create rbd2 128
pool 'rbd2' created
```

查看创建的 pool

``` bash
[root@localhost ~]# rados lspools
data
metadata
rbd
images
rbd1
rbd2
```

### 增加 cinder 的卷后端

编辑 cinder 的配置文件(一般为 /etc/cinder/cinder.conf):

``` python
enabled_backends=lvm,rbd2,rbd1
[rbd1]
volume_backend_name=rbd1
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_pool=rbd1
rbd_max_clone_depth=5
rbd_user=admin
rbd_flatten_volume_from_snapshot=False
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_secret_uuid=7d65f135-d7ba-4d87-a083-910ff8cf4eb2

[rbd2]
volume_backend_name=rbd2
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_pool=rbd2
rbd_max_clone_depth=5
rbd_user=admin
rbd_flatten_volume_from_snapshot=False
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_secret_uuid=7d65f135-d7ba-4d87-a083-910ff8cf4eb2
```

重启 cinder 服务

``` bash
[root@localhost ~]# systemctl restart openstack-cinder-scheduler
[root@localhost ~]# systemctl restart openstack-cinder-volume
[root@localhost ~]# systemctl restart openstack-cinder-api
```

创建私有卷类型
--------------

注意使用的 cinderclient 必须是 V2 版本，如果是 RestAPI，也必须访问 V2
的cinder，可以使用下列指令设置命令行的版本号：

``` bash
export OS_VOLUME_API_VERSION=2
```

创建两种私有的卷类型: volume~type1~, volume~type~

``` bash
[root@localhost ~]# cinder type-create volume_type1 --is-public false
+--------------------------------------+--------------+-------------+-----------+
|                  ID                  |     Name     | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| 71fa330c-5401-47ea-9e2d-96466ec7d3bd | volume_type1 |             |   False   |
+--------------------------------------+--------------+-------------+-----------+
[root@localhost ~]# cinder type-create volume_type2 --is-public false
+--------------------------------------+--------------+-------------+-----------+
|                  ID                  |     Name     | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| f4030afb-3300-47c0-97e4-debfcacbcf82 | volume_type2 |             |   False   |
+--------------------------------------+--------------+-------------+-----------+
[root@localhost ~]# cinder type-list --all
+--------------------------------------+--------------+-------------+-----------+
|                  ID                  |     Name     | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| 71fa330c-5401-47ea-9e2d-96466ec7d3bd | volume_type1 |             |   False   |
| f4030afb-3300-47c0-97e4-debfcacbcf82 | volume_type2 |             |   False   |
+--------------------------------------+--------------+-------------+-----------+
```

将 volume~type1~ 和 volume~type2~ 的后端存储分别设置为 rbd1 和 rbd2:

注意: 查看卷的类型只能需要加 –all 参数

``` bash
[root@localhost ~]# cinder type-key 71fa330c-5401-47ea-9e2d-96466ec7d3bd set volume_backend_name=rbd1
[root@localhost ~]# cinder type-key f4030afb-3300-47c0-97e4-debfcacbcf82 set volume_backend_name=rbd2
[root@localhost ~]# . keystonerc_admin; cinder type-list --all
+--------------------------------------+--------------+-------------+-----------+
|                  ID                  |     Name     | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| 71fa330c-5401-47ea-9e2d-96466ec7d3bd | volume_type1 |             |   False   |
| f4030afb-3300-47c0-97e4-debfcacbcf82 | volume_type2 |             |   False   |
+--------------------------------------+--------------+-------------+-----------+
```

将卷类型指定给租户
------------------

将卷 volume~type1~ 和 volume~type2~ 分别指定给 tenant1 和 tenant2,
注意这里 只能使用 UUID，不能使用名字

``` bash
[root@localhost ~]# cinder type-access-add --volume-type 71fa330c-5401-47ea-9e2d-96466ec7d3bd --project-id 38132b7bf32b434398862971c567bca3
[root@localhost ~]# cinder type-access-add --volume-type f4030afb-3300-47c0-97e4-debfcacbcf82 --project-id 3fb70cdab54c4322a277f9acaa360e71
[root@localhost ~]# cinder type-access-list --volume-type 71fa330c-5401-47ea-9e2d-96466ec7d3bd
+--------------------------------------+----------------------------------+
|            Volume_type_ID            |            Project_ID            |
+--------------------------------------+----------------------------------+
| 71fa330c-5401-47ea-9e2d-96466ec7d3bd | 38132b7bf32b434398862971c567bca3 |
+--------------------------------------+----------------------------------+
[root@localhost ~]# cinder type-access-list --volume-type f4030afb-3300-47c0-97e4-debfcacbcf82
+--------------------------------------+----------------------------------+
|            Volume_type_ID            |            Project_ID            |
+--------------------------------------+----------------------------------+
| f4030afb-3300-47c0-97e4-debfcacbcf82 | 3fb70cdab54c4322a277f9acaa360e71 |
+--------------------------------------+----------------------------------+
```

创建卷
------

使用 demo1 创建类型为 volume~type1~ 的卷

``` bash
[root@localhost ~]# . keystonerc_demo1; cinder create 1 --volume-type volume_type1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|           availability_zone           |                 nova                 |
|                   id                  | 4e4fad1f-26aa-4690-9268-6628d01138e8 |
|                  name                 |                 None                 |
|      os-vol-tenant-attr:tenant_id     |   38132b7bf32b434398862971c567bca3   |
|                  size                 |                  1                   |
|                 status                |               creating               |
|                user_id                |   2e69d91577e949509ce20069786526c5   |
|              volume_type              |             volume_type1             |
+---------------------------------------+--------------------------------------+
[root@localhost ~]# rbd -p rbd1 ls
volume-4e4fad1f-26aa-4690-9268-6628d01138e8
```

使用 demo2 创建类型为 volume~type2~ 的卷

``` bash
[root@localhost ~]# . keystonerc_demo2; cinder create 1 --volume-type volume_type2
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|           availability_zone           |                 nova                 |
|                   id                  | 3366b9d9-7437-45d1-bbef-bc59127785e6 |
|                  name                 |                 None                 |
|      os-vol-tenant-attr:tenant_id     |   3fb70cdab54c4322a277f9acaa360e71   |
|                 status                |               creating               |
|                user_id                |   7687d7a33e49464cb7cedfbf20e5781c   |
|              volume_type              |             volume_type2             |
+---------------------------------------+--------------------------------------+
[root@localhost ~]# rbd -p rbd2 ls
volume-3366b9d9-7437-45d1-bbef-bc59127785e6
```

分别使用 demo1, demo2 创建默认类型的卷

``` bash
[root@localhost ~]# . keystonerc_demo1; cinder create 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|           availability_zone           |                 nova                 |
|                   id                  | 6d2811e9-3ca6-4993-801a-ff59783ef891 |
|                  name                 |                 None                 |
|      os-vol-tenant-attr:tenant_id     |   38132b7bf32b434398862971c567bca3   |
|                  size                 |                  1                   |
|                 status                |               creating               |
|                user_id                |   2e69d91577e949509ce20069786526c5   |
|              volume_type              |             volume_type1             |
+---------------------------------------+--------------------------------------+
[root@localhost ~]# rbd -p rbd1 ls
volume-4e4fad1f-26aa-4690-9268-6628d01138e8
volume-6d2811e9-3ca6-4993-801a-ff59783ef891
[root@localhost ~]# . keystonerc_demo2; cinder create 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|           availability_zone           |                 nova                 |
|                   id                  | ae3466e2-3ed3-4b6a-be9b-0dd485f99da0 |
|                  name                 |                 None                 |
|      os-vol-tenant-attr:tenant_id     |   3fb70cdab54c4322a277f9acaa360e71   |
|                  size                 |                  1                   |
|                 status                |               creating               |
|                user_id                |   7687d7a33e49464cb7cedfbf20e5781c   |
|              volume_type              |             volume_type2             |
+---------------------------------------+--------------------------------------+
[root@localhost ~]# rbd -p rbd2 ls
volume-3366b9d9-7437-45d1-bbef-bc59127785e6
volume-ae3466e2-3ed3-4b6a-be9b-0dd485f99da0
```

测试分别使用 demo1 创建 volume~type2~, 使用 demo2 创建 volume~type1~
的卷

``` bash
[root@localhost ~]# . keystonerc_demo1; cinder create 1 --volume-type volume_type2
ERROR: Not Found (HTTP 404) (Request-ID: req-c749eb4e-d4dd-4bfb-aa62-26f8b8a4f986)
[root@localhost ~]# . keystonerc_demo2; cinder create 1 --volume-type volume_type1
ERROR: Not Found (HTTP 404) (Request-ID: req-255b8249-f260-4b36-a2c0-0a5aa28e74e3)
```

查看卷类型
----------

可以看到，除了管理员用户，用户只能看到各自租户的卷类型，不能看到其他租户的卷类型

``` bash
[root@localhost ~]# . keystonerc_demo1; cinder type-list
+--------------------------------------+--------------+-------------+-----------+
|                  ID                  |     Name     | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| 71fa330c-5401-47ea-9e2d-96466ec7d3bd | volume_type1 |             |   False   |
+--------------------------------------+--------------+-------------+-----------+
[root@localhost ~]# . keystonerc_demo2; cinder type-list
+--------------------------------------+--------------+-------------+-----------+
|                  ID                  |     Name     | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| f4030afb-3300-47c0-97e4-debfcacbcf82 | volume_type2 |             |   False   |
+--------------------------------------+--------------+-------------+-----------+
```

查看卷
------

可以看到，除了管理员用户，用户只能看到各自租户的卷，不能看到其他租户的卷

``` bash
[root@localhost ~]# . keystonerc_demo1; cinder list
+--------------------------------------+-----------+------+------+--------------+----------+-------------+
|                  ID                  |   Status  | Name | Size | Volume Type  | Bootable | Attached to |
+--------------------------------------+-----------+------+------+--------------+----------+-------------+
| 4e4fad1f-26aa-4690-9268-6628d01138e8 | available | None |  1   | volume_type1 |  false   |             |
| 6d2811e9-3ca6-4993-801a-ff59783ef891 | available | None |  1   | volume_type1 |  false   |             |
+--------------------------------------+-----------+------+------+--------------+----------+-------------+
[root@localhost ~]# . keystonerc_demo2; cinder list
+--------------------------------------+-----------+------+------+--------------+----------+-------------+
|                  ID                  |   Status  | Name | Size | Volume Type  | Bootable | Attached to |
+--------------------------------------+-----------+------+------+--------------+----------+-------------+
| 3366b9d9-7437-45d1-bbef-bc59127785e6 | available | None |  1   | volume_type2 |  false   |             |
| ae3466e2-3ed3-4b6a-be9b-0dd485f99da0 | available | None |  1   | volume_type2 |  false   |             |
+--------------------------------------+-----------+------+------+--------------+----------+-------------+
```
