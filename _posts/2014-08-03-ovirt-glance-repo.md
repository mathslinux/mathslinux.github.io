---
layout: post
tag: Virtualization
date: '\[2014-08-03 日 21:19\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: ovirt 新版本中的 glance 仓库
---

在 oVirt 3.4 中, 内建了一个默认的外部供应商: `ovirt-image-repository`,
该仓库其实 是提供 OpenStack 的 glance 镜像服务, 目前里面有 fodora,
centos 等的虚拟机模板镜像.

但是在使用中会有一个问题, 由于该镜像存放在 ovirt.org 中,
在中国大陆(你懂得)访问速度 相当慢, 为了能更方便地使用这些镜像,
最好把这些镜像导入到本地的 glance 中.

我考察了一下该仓库, 该仓库的 endpoint 为
<http://glance.ovirt.org:9292/>, 且它没有使用 keystone 验证服务,
只是一个简单的没有权限控制的镜像服务, 所以我们
可以直接下载然后导入到我们的 glance 中:

先列出所有的镜像

``` bash
# glance --os-image-url http://glance.ovirt.org:9292 image-list
```

下载该镜像, 然后导入到 openstack glance 中, ID 为上面列出来的 ID:

``` bash
# wget -O [filename] http://glance.ovirt.org:9292/v1/images/[ID]
# glance image-create --name="what you want" --is-public True --container-format bare --disk-format qcow2 --progress --file [filename]
```
