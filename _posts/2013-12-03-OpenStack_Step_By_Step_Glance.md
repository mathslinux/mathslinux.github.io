---
layout: post
tag: OpenStack
date: '\[2013-12-03 二 11:25\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'OpenStack Step By Step(2) - setup glance'
---

介绍
====

Glance 是 OpenStack 中处理虚拟机镜像的模块, 对外提供镜像查询, 上传 下载,
删除等服务. Nova 启动虚拟机的时候需要从 glance 里获取虚拟机的镜像/模板.

安装配置
========

安装
----

``` bash
# apt-get install -y glance glance-api python-glanceclient glance-common glance-registry python-glance
```

配置
----

### 修改配置文件

修改 glance 的配置文件: glance-api.conf 和 glance-registry.conf 把
\[keystone~authtoken~\] 部分从:

``` bash
[keystone_authtoken]
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = %SERVICE_TENANT_NAME%
admin_user = %SERVICE_USER%
admin_password = %SERVICE_PASSWORD%
```

修改为:

``` bash
[keystone_authtoken]
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = geniux
```

设置认证方式, 设置为通过 keystone 认证, 将配置文件中:

``` bash
[paste_deploy]
#flavor=
```

修改为:

``` bash
[paste_deploy]
flavor = keystone
```

### 创建数据库

清空 apt-get 安装时默认创建的数据库, 然后手动创建一个

``` bash
# rm -f /var/lib/glance/glance.sqlite
# glance-manage version_control 0
# glance-manage db_sync
# chown glance:glance /var/lib/glance/glance.sqlite
# service glance-api restart && service glance-registry restart
```

测试
----

``` bash
glance --os-username admin --os-password geniux --os-tenant-name admin --os-auth-url http://localhost:5000/v2.0/ index
ID                                   Name                           Disk Format          Container Format     Size          
------------------------------------ ------------------------------ -------------------- -------------------- --------------
```

上传镜像到 glance
-----------------

为了方便, 把用户名, 租户信息, api url 信息放到用户配置文件下, 每次登陆
自动导入这些信息:

``` bash
# tail -n 4 ~/.bashrc
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=geniux
export OS_AUTH_URL="http://localhost:5000/v2.0/"
```

首先获取一个需要导入的镜像(为了省事, 这里直接使用 ubuntu 提供的"云"镜像)

``` bash
# axel -n 8 http://uec-images.ubuntu.com/releases/13.10/release/ubuntu-13.10-server-cloudimg-amd64-disk1.img
```

用 glance image-create 上传该镜像

``` bash
# image-create --name="Ubuntu 13.10 cloudimg" --is-public True --container-format bare \
--disk-format qcow2 --file ubuntu-13.10-server-cloudimg-amd64-disk1.img --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | eef6cffa7f1c680afb7bc0405d932f06     |
| container_format | bare                                 |
| created_at       | 2013-12-03T06:54:34.157369           |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 836911d3-6e65-48d9-abf4-2e6051f3925d |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | Ubuntu 13.10 cloudimg                |
| owner            | 5c5dc9b3e96b4dc886ab9c9d78535618     |
| protected        | False                                |
| size             | 242941952                            |
| status           | active                               |
| updated_at       | 2013-12-03T06:54:35.371760           |
+------------------+--------------------------------------+
```

以后 Nova 创建虚拟机的时候, 就可以使用这个镜像了, 下节探讨.
