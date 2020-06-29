---
layout: post
tag: OpenStack
date: '\[2014-10-24 五 18:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Glance 源码分析(1) – 框架
---

以下主要分析 V2 版本的代码, V1 和 V2的最大区别就是:

-   V1: 包含两个服务 glance-api 和 glance-registry. glance-api
    接受客户端的 所有命令, 分发并响应, 涉及到数据库的操作由内部转发到
    glance-registry 完成
-   V2 简化了流程, 所有的处理都在内部实现, 不需要额外的服务. 因此只需要
    glance-api 一个服务

启动流程
========

![](/images/posts/OpenStack/glance_source_0.png)

``` python
# glance/cmd/api.py
def main():
    # 载入配置, 下面的函数在 glance/common/config.py 中定义, 调用此函数会初始化
    # oslo 模块中的模块变量 CONF, 把 glance-api.conf 中的值设置在 CONF 的属性中,
    # 使用的时候按照 "CONF.enable_v1_api" 此方式调用, 具体的细节不多讲
    config.parse_args()

    # 初始化后端存储,
    # 1. 将所有后端存储的类名注册到 glance/store/__init__.py:REGISTERED_STORES
    # 2. 将所有后端存储的名字和实例注册到 glance/store/location.py:SCHEME_TO_CLS_MAP
    #    以后 glance-api 可以根据用户的请求和配置文件找到具体的后端存储的实例, 调用相应的
    #    实例的函数(add/delete)来操作
    glance.store.create_stores()

    # 启动 WSGI 程序
    # load_paste_app 会在默认位置(/etc/glance/)找到 glance-api-paste.ini,
    # 然后根据用户的配置(是否启用 keystone等)调用 paste.loadapp 载入相应的 app,
    # 最后传递给 server.start 启动 WSGI 应用, 具体的细节之后的系列会讲到
    server = wsgi.Server()
    server.start(config.load_paste_app('glance-api'), default_port=9292)
    server.wait()
```

API 框架
========

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

关于 paste 模块的使用说明, 请参考 [python paste.deploy
探索](http://mathslinux.org/?p%3D596)

流程如下:

-   首先 **WSGI** 接收到用户请求, 将用户的请求信息交给 **paste**
    模块处理
-   **paste** 模块根据配置文件的规则依次经过 **versionnegotiation**
    **authtoken** **context** 这几个过滤器
-   最后交由 **rootapp**, 这是个类型为 app 的处理, **rootapp**
    内部再根据用户提供的版本 信息(v1/v2) 交由 **apiv1app** 或者
    **apiv2app** 处理, 最后把返回 HTTP Response
-   真正的业务逻辑在 **apiv1app/apiv2app** 内部实现, 见下面的处理流程

处理流程
========

apiv2app 定义的工厂方法在 glance/api/v2/router:API.factory 中, API
类继承自 wsgi.Router, Router 类利用了 python-route 模块做 url
的选择处理, 具体的流程 见下(详细的分析请参考后面的系列):

![](/images/posts/OpenStack/glance_source_2.png)

关于 route 模块的使用说明, 请参考 [python Route
简单使用笔记](http://mathslinux.org/?p%3D598)
