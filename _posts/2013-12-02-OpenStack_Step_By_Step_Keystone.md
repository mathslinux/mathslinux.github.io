---
layout: post
tag: OpenStack
date: '\[2013-12-02 一 10:55\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'OpenStack Step By Step(1) - setup keystone'
---

介绍
====

关于 Keystone 的介绍, 网络上很多, 比如 [What is this Keystone
anyway?](http://docs.openstack.org/trunk/openstack-identity/admin/content/what-is.html),
这里就不浪费口水了.

安装配置
========

为了方便, 在以下的安装过程中使用 sqlite3 作为数据库

安装
----

``` bash
# apt-get install -y sqlite3 keystone python-keystone python-keystoneclient
```

配置
----

### 修改 admin 权限

首先修改 admin 的令牌, 这里我修改为 "geniux"

``` bash
# emacsclient /etc/keystone/keystone.conf
# admin_token = geniux # 设置 admin_token 为 geniux
```

接下来, 创建需要的 tenant, user, roles 等 keystone 需要的元素, 这里使用
devstack 的
[keystone~data~.sh](https://github.com/openstack-dev/devstack/blob/master/files/keystone_data.sh)
脚本来创建, 我简单的把 admin 和 service 的 token 修改为上面设置的值.
具体见下表:

| Tenant             | User    | Roles                                 |
|--------------------|---------|---------------------------------------|
| admin              | admin   | admin                                 |
| service            | glance  | admin                                 |
| service            | nova    | admin, \[ResellerAdmin (swift only)\] |
| service            | quantum | admin \# if enabled                   |
| service            | swift   | admin \# if enabled                   |
| demo               | admin   | admin                                 |
| demo               | demo    | Member, anotherrole                   |
| invisible~toadmin~ | demo    | Member                                |

首先, 清空 apt-get 安装时默认创建的数据库, 然后手动创建一个

    # rm -f /var/lib/keystone/keystone.db
    # keystone-manage db_sync
    # chown keystone:keystone /var/lib/keystone/keystone.db
    # service keystone restart

然后, 用 keystone~data~.sh 脚本创建上表的元素

``` bash
# ./keystone_data.sh
```

最后一步, 是创建 endpoint(OpenStack 服务的 API 接口), 这里我使用一个由
[hastexo](http://www.hastexo.com/system/files/user/4/endpoints.sh__0.txt)
提供的脚本来实现, 因为原始的脚本是用 mysql 来搭建地, 所以我这里需要 将
mysql 的相关代码修改为 sqlite 的代码, 修改后的文件见附件.

``` bash
# ./endpoints.sh
```

测试
----

``` bash
# keystone --token geniux --endpoint http://127.0.0.1:35357/v2.0/ user-list
+----------------------------------+--------+---------+--------------------+
|                id                |  name  | enabled |       email        |
+----------------------------------+--------+---------+--------------------+
| c3ec029db5f5447c949fe36b8f1119a0 | admin  |   True  | admin@hastexo.com  |
| 2fcb4478f84a4a6d92b0aa62063fb320 |  demo  |   True  |  demo@hastexo.com  |
| 1c9c5d982c894f2ebb1edc70addebcf8 | glance |   True  | glance@hastexo.com |
| e0485d57f45744a58c3ac2d4cb9db137 |  nova  |   True  |  nova@hastexo.com  |
| ee73fbd4d3ee4c898ccb9aa5d4ec1b7c | swift  |   True  | swift@hastexo.com  |
+----------------------------------+--------+---------+--------------------+
```

附件
====

[keystone~data~.sh](https://gist.github.com/mathslinux/7748065)

[endpoints.sh](https://gist.github.com/mathslinux/7748174)
