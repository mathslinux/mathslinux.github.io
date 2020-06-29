---
layout: post
tag: OpenStack
date: '\[2014-10-14 二 16:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 搭建一个最简单的 glance 服务
---

OpenStack 的各个模块是高度独立的, glance, neutron,
并且可以给其他的程序使用, e.g. ovirt,
下面的文档描述使用最简单的方式搭建一个 glance 服务.(以 ubuntu 为例)

安装 glance

``` bash
# apt-get install glance
```

That's all!

上面搭建的 glance 是使用 sqlite 作为后端数据库, 没有使用认证的服务.

issue
=====

在 icehouce 版本中, 由于配置文件解析和兼容的关系, 有一个 bug, 需要手动
在指定默认的 sqlite 数据库位置.

在 **/etc/glance/glance-api.conf** 和
**/etc/glance/glance-registry.conf** 的 **\[default\]** 下, 添加一个配置

``` python
connection = sqlite:////var/lib/glance/glance.sqlite
```

然后重新同步一下:

``` bash
# glance-manage db_sync
# service glance-api restart
# service glance-registry restart
```

使用
====

由于使用 glanceclient 的话, 默认需要加上认证的信息, 所以只能通过 API
的方式 使用, 这里我使用 curl 作为测试工具:

获取 images 列表
----------------

``` bash
# curl http://192.168.3.33:9292/v2/images | python -mjson.tool
```

下载 images
-----------

``` bash
# curl -o test.img http://192.168.3.33:9292/v2/images/ce252e1a-131a-4ebd-a9b0-0cf462f066e6/file
```

上传 image
----------

``` bash
# curl -i -X POST -H 'Content-Type: application/octet-stream' -H 'x-image-meta-disk_forma2' \
  -H 'x-image-meta-container_format: bare' -H 'Transfer-Encoding: chunked' \
  -H 'User-Agent: python-glanceclient' -H 'x-image-meta-is_public: False' \
  -H 'x-image-meta-name: test' -H 'x-image-meta-size: 197120' \
  --data-binary @test.qcow2 http://192.168.3.33:9292/v1/images
```

删除 image
----------

``` bash
# curl -X DELETE http://192.168.3.33:9292/v2/images/d7aa01fc-5999-4720-8a05-325f7ffb9332
```
