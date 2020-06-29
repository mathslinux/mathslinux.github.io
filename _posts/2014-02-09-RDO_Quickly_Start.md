---
layout: post
tag: OpenStack
date: '\[2014-02-09 日 22:25\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: RDO 初体验
---

无数的实践证明, Openstack 的搭建不是一个技术活, 纯粹是体力活,
所以有了各种 各样的搭建工具, e.g. puppet, chef. 理论上在 ubuntu 等平台用
puppet 搭建 相对已经 很简单了, 但是还有其他的问题, 需要去简单熟悉 puppet
的语法, 修改类似于 ruby 的 配置文件. 但是 [RDO](http://rdo.org) 这货,
虽然是基于 puppet 来构建 openstack 的, 但是用户
在使用的过程中完全感觉不到 puppet 的存在, 甚至只会简单的一些 linux
命令的新手 也能很快的上手.

环境设置
========

| Hostname       | IpAddress                            | Role                 |
|----------------|--------------------------------------|----------------------|
| rdo.cloud.com  | 192.168.176.36(eth0) 10.0.0.15(eth1) | master(compute node) |
| rdo2.cloud.com | 192.168.176.37(eth0) 10.0.0.16(eth1) | compute node         |

安装配置
========

登陆到 rdo.cloud.com 这台机器上. 执行下列指令:

``` bash
# yum install -y http://rdo.fedorapeople.org/openstack/openstack-havana/rdo-release-havana.rpm
# yum install -y openstack-packstack
# yum -y update # 为了方便, 把系统升级到最新
# reboot
```

``` bash
# packstack --gen-answer-file my_answers.txt # 生成一个应答文件
# emacsclient my_answers.txt # 修改稍许设置, 主要是以下几项
CONFIG_NTP_SERVERS=pool.ntp.org
CONFIG_KEYSTONE_ADMIN_PW=admin # 为了方便, 把 admin 的密码修改为 admin, 不然登陆 UI 的时候太麻烦了
CONFIG_NOVA_COMPUTE_HOSTS=192.168.176.36,192.168.176.37
CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=gre
CONFIG_NEUTRON_OVS_TUNNEL_RANGES=1:1000
CONFIG_NEUTRON_OVS_TUNNEL_IF=eth1
```

最后一步, 在所有上面配置文件上定义的节点上安装配置 openstack

``` bash
# packstack --answer-file my_answers.txt # 会询问密码
```

So easy, have fun!
