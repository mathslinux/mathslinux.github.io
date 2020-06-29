---
layout: post
tag: Storage
date: '\[2014-05-25 日 19:55\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: oVirt 使用 glance
---

oVirt 导入支持第三方的模块, 比如 forman, OpenStack 中的模块(e.g. glance,
neutron),

添加 glance
===========

首先, 需要搭建好 OpenStack 环境, 网上有很多文档, 自行 google.

注意: 由于该功能还在早期的试验阶段, 少许配置稍微有点麻烦, 用户需要手动用
engine-config 配置 keystone 的认证地址:

``` bash
# engine-config --set KeystoneAuthUrl=http://192.168.3.38:5000/v2.0/
```

然后重启 ovirt-engine

``` bash
# service ovirt-engine restart
```

点击 `系统` -\> `外部供应商` -\> `添加`, 打开 `添加供应商` 窗口:
![](/images/posts/Storage/ovirt_glance_add.png)

如图, 选择 `类型` 为 glance, 填写实际的供应商 URL, 勾选 `要求验证`,
根据实际情况填写 `用户名`, `密码`, `Tenant名称`.

然后点击 `测试`, 如果测试通过, 点击确认添加该 glance 服务. 可以看到
glance 已经加入到 oVirt 中了, 点击映像, 可以看到 glance 中的所有 image:
![](/images/posts/Storage/ovirt_glance_done.png)

导入 glance 镜像到 oVirt
========================

选中想要导入的镜像, 点击 `导入`, 打开 `导入映像` 窗口可以将 glance
镜像导入其他存储域中: ![](/images/posts/Storage/ovirt_glance_import.png)

![](/images/posts/Storage/ovirt_glance_import_window.png) 在 `导入窗口`
选择实际的数据中心.

导出 oVirt 磁盘到 glance
========================

点击 `磁盘`, 选中磁盘, 点击 `导出`: [选中磁盘](ovirt_glance_export)

![](/images/posts/Storage/ovirt_glance_export_window.png)

登陆到 glance 主机, 可以看到我们导入的磁盘已经在 glance 中了.

``` bash
# glance index
ID                                   Name                           Disk Format          Container Format     Size          
------------------------------------ ------------------------------ -------------------- -------------------- --------------
f3fe43d2-dac0-4bd5-9f62-a04c29ba46b3 centos-tiny_Disk1              qcow2                bare                              0
cfbe1c2e-999f-4f48-84da-39f7e8b5d36b Ubuntu 13.10 cloudimg          qcow2                bare                      243728384
```
