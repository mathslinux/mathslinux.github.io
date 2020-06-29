---
layout: post
tag: Virtualization
date: '\[2014-06-13 五 04:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'ovirt-shell 使用'
---

Introduction
============

[oVirt](http://ovirt.org) 项目的用户接口有 GUI 方式的, rest API 方式的,
SDK 方式, 甚至也有 命令行方式的, 也就是本文要介绍的 ovirt-shell.

本文介绍在 OSX(这么牛逼)环境下, 从头建立一个数据中心, 集群, 存储域,
并且基于 此创建一个虚拟机并对其完成生命周期管理.

Install
=======

从功能来讲, ovirt-shell 只是一个控制 oVirt 的命令行客户端,
它的实现是基于 python SDK 的, 所以几乎在所有 POSIX
平台都能很容易的安装(包管理器或者 pip 方式), 我的主要 开发平台是 OSX,
正是使用 pip 来安装 ovirt-shell 的.

使用发行版提供的包管理器
------------------------

比如在 RPM 系的发行版中, 可以

``` bash
# yum install -y ovirt-engine-cli
```

使用 python 的包管理工具
------------------------

如果发行版的软件仓库中没有提供软件, 那么可以使用 python 的包管理器 pip
安装, 比如在我的 mac 上, 使用:

``` bash
# pip install ovirt-shell
```

Usage
=====

接下来演示从头使用 oVirt, 包括创建数据中心, 数据域, 虚拟机等等.

连接到 oVirt engine
-------------------

首先需要连接到 oVirt engine, 使用下列的指令:

``` bash
# wget http://192.168.3.252/ca.crt -O ~/ca.crt # 下载 engine 的 CA 证书
# ovirt-shell -c -A ~/ca.crt -l "https://192.168.3.252:443/api" -u admin@internal
```

上述命令中的参数解释:

``` bash
-c: 自动连接到服务器, 不要使用交互模式
-A: 指定 oVirt engine 的 CA 证书, 下载地址为 http://[ovirt-engine-url]/ca.crt
-l: ovirt-enging 的 url, 格式为 http[s]://server[:port]/api
-u: 连接 ovirt-engine 的用户名
```

如果出现以下错误, 是由于证书中的 hostname 和 engine 的 hostname 不匹配,
engine 的配置不一致导致的, 连接的时候用 "-I" 告诉 engine 不要验证证书的
hostname 即可.

``` bash
The host name "192.168.3.252" contained in the URL doesn't match any of the names in the server certificate
```

之后, ovirt-shell 会提示用户输入密码, 然后就进入 ovirt-shell
的交互模式了, 可以使用 help 查看使用帮助, 任何命令都可以使用 TAB 补全.

创建一个完整的数据中心
----------------------

我这里创建一个完整的 NFS 类型的数据中心

### 创建一个数据中心

先创建一个名为 test 空的 NFS 数据中心

``` bash
[oVirt shell (connected)]# add datacenter --name test --comment "test" --description "test" --storage_type nfs --version-major 3 --version-minor 3

id                              : 67ef255f-29ff-4474-8008-085d8066bb04
name                            : test
description                     : test
comment                         : test
status-state                    : uninitialized
storage_type                    : nfs
supported_versions-version-major: 3
supported_versions-version-minor: 3
version-major                   : 3
version-minor                   : 3
```

上述命令中的参数解释:

``` bash
--storage_type: 数据中心的类型.
--version-major: 和 --version-minor 一起确定数据中心的兼容版本.
```

### 创建一个集群

创建一个名为 test 的集群

``` bash
[oVirt shell (connected)]# add cluster --data_center-name test --name test --comment "test" \
> --description "test" --cpu-id "Intel SandyBridge Family" --version-major 3 --version-minor 3 \
> --virt_service true --gluster_service false

id                                         : f57b9ecf-c99d-4285-8f88-8ce4b3348f4f
name                                       : test
description                                : test
ballooning_enabled                         : False
comment                                    : test
cpu-id                                     : Intel SandyBridge Family
data_center-id                             : 67ef255f-29ff-4474-8008-085d8066bb04
error_handling-on_error                    : migrate
gluster_service                            : False
memory_policy-overcommit-percent           : 200
memory_policy-transparent_hugepages-enabled: True
scheduling_policy-policy                   : none
threads_as_cores                           : False
trusted_service                            : False
tunnel_migration                           : False
version-major                              : 3
version-minor                              : 3
virt_service                               : True
```

上述命令中的参数解释:

``` bash
--data_center-name: 对应的数据中心.
--cpu-id: 兼容的主机的 CPU 类型.
--version-minor: 和 --version-minor 对应集群的兼容版本.
--virt_service: 是否用来作为虚拟机集群.
--gluster_service: 是否作为 glusterfs 集群.
```

### 添加主机到数据中心

由于我的演示环境已经有很多主机注册到系统中了, 我使用一个空闲的.

``` bash
[oVirt shell (connected)]# list hosts --query "status=maintenance"

id         : 34905bed-ebf5-45b6-a30e-65abe149ff24
name       : node-123
```

把它加入创建的集群中, 并激活之.

``` bash
[oVirt shell (connected)]# update host node-123 --cluster-name test

[oVirt shell (connected)]# action host node-123 activate
host-id     : 34905bed-ebf5-45b6-a30e-65abe149ff24
status-state: complete
```

### 添加存储

创建一个 NFS 的存储域:

``` bash
[oVirt shell (connected)]# add storagedomain --name test --storage-type nfs --storage_format v3 --type data --host-name node-123 --storage-address 192.168.3.157 --storage-path /home/ovirt/nfs-test

```

上述命令中的参数解释:

``` bash
--storage-type: 存储类型.
--storage_format: NFS 存储的版本号.
--type: 存储域的类型, 这里作为数据存储域.
--host-name: 使用主机 node-123.
--storage-address: NFS 的地址
--storage-path: 挂载路径
```

然后把该存储域加入到数据中心中

``` bash
[oVirt shell (connected)]# add storagedomain --name test --datacenter-identifier test
```

### 添加 ISO

添加一个 ISO 存储域加入到数据中心中

``` bash
[oVirt shell (connected)]# add storagedomain --name iso --datacenter-identifier test
```

创建虚拟机
----------

我们的 oVirt 环境已经准备好了, 可以开始使用该平台进行各种操作了.
下面我们创建虚拟机:

### 创建一个虚拟机

``` bash
[oVirt shell (connected)]# add vm --name test --template-name Blank --cluster-name test \
> --os-type rhel_6x64 --memory 2147483648 --cpu-topology-cores 2
```

上述命令中的参数解释:

``` bash
--template-name: 该虚拟机基于的模板, 如果模板为空, 则填 Blank
--cluster-name: 虚拟机处于的集群
--os-type: 虚拟机的操作系统类型, 有效值为:
           windows_2008x64, unassigned, other, other_linux, windows_2008r2, 
           ubuntu_12_04, windows_2008, windows_7x64, windows_2003x64, windows_8x64, 
           windows_7, rhel_6x64, ubuntu_12_10, rhel_6, rhel_5, rhel_4x64, rhel_4, 
           rhel_3, windows_8, windows_2008R2x64, windows_2012x64, windows_xp, rhel_3x64, 
           windows_2003, ubuntu_13_10, rhel_5x64, sles_11, ubuntu_13_04
--memory: 分配的内存, 以字节为单位
--cpu-topology-cores: 虚拟机的 CPU 核心数
```

### 给虚拟机新分配一个磁盘

``` bash
[oVirt shell (connected)]# add disk --name test_disk --size 10737418240 --interface virtio \
> --format cow --bootable true --storagedomain-identifier test
```

上述命令中的参数解释:

``` bash
--size: 磁盘大小, 以字节为单位.
--interface: 磁盘的接口类型, 可选的值为: ide, virtio_scsi, virtio
-- format: 磁盘镜像的类型, 可选的值为 cow, raw
```

将磁盘添加到虚拟机上, 并激活, 注意这里只能使用磁盘的 id 来操作.

``` bash
[oVirt shell (connected)]# list disks --query "alias=test_disk" | grep id
id         : 4357a838-2a99-4a05-a0b9-be6ee6ec5eaa

[oVirt shell (connected)]# add disk --id 4357a838-2a99-4a05-a0b9-be6ee6ec5eaa --vm-identifier test

[oVirt shell (connected)]# action disk test_disk activate --vm-identifier test
```

### 给虚拟机新分配一个网卡

``` bash
[oVirt shell (connected)]# add nic --vm-identifier test --name test --network-name ovirtmgmt \
> --interface virtio --linked true --plugged true
```

上述命令中的参数解释:

``` bash
--network-name: 逻辑网络的名称.
--interface: 网卡接口类型, 可选的值为: e1000, virtio, rtl8139, rtl8139_virtio
--linked: 链接状态.
--plugin: 插入状态.
```

启动虚拟机
----------

虚拟机已经准备就绪, 现在可以启动了, 由于我们的虚拟机没有操作系统,
我们需要设置虚拟机 的启动第一启动设备为 CDROM, 并用该 CDROM
给虚拟机安装一个操作系统.

先列出 ISO 域名里可用的的镜像.

``` bash
[oVirt shell (connected)]# list files --storagedomain-identifier iso
id         : CentOS-6.5-x86_64-minimal.iso
name       : CentOS-6.5-x86_64-minimal.iso
```

给虚拟机增加一个 CDROM 设备.

``` bash
[oVirt shell (connected)]# add cdrom --vm-identifier test --file-id CentOS-6.5-x86_64-minimal.iso
```

启动虚拟机, 以 CDROM 作为启动设备

``` bash
# action vm test start --vm-stateless true --vm-os-boot "boot.dev=cdrom" --vm-display-type vnc
```

上述命令中的参数解释:

``` bash
--vm-stateless: 以 stateless 模式启动虚拟机.
--vm-os-boot: 第一启动设备, 可选的值为 cdrom, hd, network.
--vm-display-type: 访问控制协议, 这里选择 vnc, 可选的有, spice, vnc
```

访问虚拟机
----------

由于虚拟机的 vnc 和 spice 密码是按需生成的, 访问该虚拟机之前, 我们需要给
vnc 设置一个临时的密码

``` bash
[oVirt shell (connected)]# action vm test ticket --ticket-value "abc123"
```

查看 vnc server 所在的地址和端口

``` bash
[oVirt shell (connected)]# show vm test | grep display
display-address           : 192.168.3.123
display-allow_override    : False
display-monitors          : 1
display-port              : 5900
display-single_qxl_pci    : False
display-smartcard_enabled : False
display-type              : vnc
```

现在, 就可以用 vnc 客户端打开 192.168.3.123:5900
加上上面设置的临时密码来 访问该虚拟机了.

虚拟机的任务(TODO)
------------------

Resources
=========

<http://www.ovirt.org/CLI#Console>
