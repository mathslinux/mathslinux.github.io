---
layout: post
tag: Storage
date: '\[2014-02-23 日 23:55\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 用 Puppet 部署 CEPH
---

相比其他分布式系统(e.g. glusterfs), Ceph 的复杂毋庸置疑,
甚至在部署上也稍显 麻烦, 目前主流的部署方式主要有以下几种:

-   ceph-deploy: 取代之前的 mkcephfs, 不推荐在生产环节使用
-   chef: 官方推荐的部署工具
-   puppet: 大家部署这些服务最喜欢的方式
-   XXX: 自己从底层定制, 比如我这里写的
    [ceph-aio](https://github.com/mathslinux/utility-tools/tree/master/ceph-aio)

本文主要介绍用 puppet 部署 ceph.

基本配置
========

三台机器, 每台机器都分别作为 monitor 和 osd, 并且每台机器都有两个磁盘,
把第二个磁盘(/dev/vdb)作为 osd. 我这里使用
[这个](https://github.com/enovance/puppet-ceph) puppet 模块,
但是这个模块 官方没有对同时把一台节点变为 monitor 和 osd的测试,
我这里测试中也出了一点问题, 我用了一些 work around 的方法, 需要注意,
下面的 Troubleshoot 中会提到.

我部署的基本环境如下:

| Hostname   | Ip Address     | Roles              | osd      |
|------------|----------------|--------------------|----------|
| ceph1.test | 192.168.176.30 | master + mon + osd | /dev/vdb |
| ceph2.test | 192.168.176.31 | agent + mon + osd  | /dev/vdb |
| ceph3.test | 192.168.176.32 | agent + mon + osd  | /dev/vdb |

修改每台节点的 hosts 文件, 使得可以互相用主机名 ping 通

``` bash
# cat /etc/hosts
... ...
192.168.176.30  ceph1.test
192.168.176.31  ceph2.test
192.168.176.32  ceph3.test
```

puppet master 配置
==================

1 安装 puppet 软件包

``` bash
# apt-get install -y ruby1.8 puppetmaster sqlite3 libsqlite3-ruby \
  libactiverecord-ruby git augeas-tools puppet ruby1.8-dev libruby1.8
# update-alternatives --set ruby /usr/bin/ruby1.8
```

2 配置 puppet

修改 /etc/puppet/puppet.conf 的相应配置项为:

``` python
[master]
storeconfigs = true
dbadapter = sqlite3

[agent]
pluginsync = true
server = ceph1.test
```

设置我们的节点不需要认证:

``` bash
# echo "*.test" > /etc/puppet/autosign.conf
```

3 puppet 的模块设置

安装需要的 puppet 模块:

``` bash
# puppet module install ripienaar/concat
# puppet module install puppetlabs/apt
```

安装 puppet 的 ceph 模块:

``` bash
# cd /etc/puppet/modules
# git clone git://github.com/enovance/puppet-ceph.git ceph
```

4 配置 site.pp 文件

我照着文档写了对应上面三个节点的 site.pp 文件, 可以从发
[这里](https://gist.github.com/mathslinux/9173097) 下载, 然后拷贝到
/etc/puppet/manifests 目录下

5 重启 puppetmaster

``` bash
# service puppetmaster restart
```

puppet agent 配置
=================

注意: 此操作要在三个节点上同时执行.

1 安装 puppet 软件包

``` bash
# apt-get -y install puppet
```

2 修改 /etc/puppet/puppet.conf 的相应配置项为:

``` python
[agent]
pluginsync = true
server = ceph1.test
```

3 大功告成, 开始利用 puppet 部署 ceph

``` bash
# puppet agent --enable
# export AGENT_OPTIONS="--onetime --verbose --ignorecache --no-daemonize --no-usecacheonfailure --no-splay --show_diff --debug"
# puppet agent $AGENT_OPTIONS # 运行五步这步操作, 分别可以获取 keyring, 配置磁盘等等
```

4 依次登陆每个节点同步 ceph 的配置文件

``` bash
# ssh cephx.test
# puppet agent -vt
```

Troubleshoot
============

在配置 osd 的时候会发现初始化 osd 的时候失败, 官方对同时把节点当做
monitor 和 osd 没有进行详细的测试, 我这里手动初始化 osd 来 work around
这个问题, 我这里 以 ceph1.test, 也就是 osd.0 为例来说明

``` bash
# rm -rf /var/lib/ceph/osd/ceph-0/* # 先清空之前初始化失败的数据
# ceph auth add osd.0 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-0/keyring # 初始化 osd 的目录
# service ceph start osd.0 # 启动 osd 服务
```

可以看到现在 osd 正常了

``` bash
# ceph osd tree
# id    weight  type name   up/down reweight
-1  0.04999 root default
-2  0.04999     host ceph1
0   0.04999         osd.0   up  1
```
