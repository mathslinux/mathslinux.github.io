---
layout: post
tag: OpenStack
date: '\[2014-10-25 六 23:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Glance 源码分析(2) – 配置文件
---

这里我们会分析 glance-api 读取以下两个配置文件

-   glance-api.conf: glance-api 的用户配置文件
-   glance-api-paste.ini: glance-api 的 WSGI 配置文件

glance-api.conf
===============

该配置文件的读取是利用 **oslo** 模块来实现的, oslo 提供了 **.ini**
格式的配置 文件的解析, 被所有 OpenStack 模块用来解析配置文件.

oslo 的用法很简单, 下面举个简单的例子:

``` python
from oslo.config import cfg

default_opts = [
    cfg.StrOpt('bind_host',
               default='0.0.0.0',
               help='IP address to listen on'),
    cfg.IntOpt('bind_port',
               default=9292,
               help='Port number to listen on')
]

app_opt = cfg.StrOpt('name',
                     default='blog',
                     help='name of this app')

# cfg.CONF 是在 oslo.cfg 模块中的一个全局变量, 首先我们需要得到一个它的引用
# 然后调用 register_opt() 注册我们需要解析的配置项, 或者使用 register_opts()
# 同时注册多个配置项
# 如果配置文件中可以找到配置项, 那么使用配置项中的值, 不然使用注册该配置项时指定
# 的默认值
CONF = cfg.CONF
CONF.register_opt(app_opt, group='app')
CONF.register_opts(default_opts)
CONF(default_config_files=['app.conf'])

# 使用的时候可以用 CONF.cfgname 来使用, 如果该 cfgname 不在 [DEFAULT] 段下,
# 那么使用 CONF.section.cfgname 来引用
print CONF.items()
print CONF.app.name
print CONF.bind_host
print CONF.bind_port
```

``` python
# file: app.conf
[DEFAULT]
bind_port = 8080
[app]
name = test
```

``` bash
# python test.py 
[('bind_port', 8080), ('config_dir', None), ('config_file', ['app.conf']), ('bind_host', '0.0.0.0'), ('app', <oslo.config.cfg.GroupAttr object at 0x7fa4a3d75c50>)]
0.0.0.0
8080
test
```

那么问题来了… 前面讲过, glance 中的配置文件通过 `config.parse_args()`
来调用的, 也就是所有的配置都是在 `glance.common.config:parse_args()`
中完成的, 通过上面的分析, 这个函数其实很简单, 这里就不深入探讨了. 总之,
以后, glance 的各个模块在想访问用户提供的配置的时候,
只需要使用类似以下的代码就可以了:

``` python
from oslo.config import cfg
CONF = cfg.CONF
print CONF.enable_v2_api
```

glance-api-paste.ini
====================

该内容和 glance 无关, 只要搞懂了 python paste 模块的用法, 就很简单了.
关于 paste 模块的使用说明, 请参考 [python paste.deploy
探索](http://mathslinux.org/?p%3D596)

下面把之前在框架那里的内容再贴一份:

![](/images/posts/OpenStack/glance_source_1.png)

``` python
[pipeline:glance-api-keystone]
pipeline = versionnegotiation authtoken context rootapp
[filter:versionnegotiation]
paste.filter_factory = glance.api.middleware.version_negotiation:VersionNegotiationFilter.factory
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
delay_auth_decision = true
[filter:context]
paste.filter_factory = glance.api.middleware.context:ContextMiddleware.factory
[composite:rootapp]
paste.composite_factory = glance.api:root_app_factory
/: apiversions
/v1: apiv1app
/v2: apiv2app
```
