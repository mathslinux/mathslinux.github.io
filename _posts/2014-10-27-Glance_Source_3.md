---
layout: post
tag: OpenStack
date: '\[2014-10-27 一 18:15\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Glance 源码分析(3) – WSGI 框架
---

Server
======

OpenStack 大多数模块封装了 eventlet 以及 eventlet.wsgi.server 组成一个
class Server 来作为一个 HTTP RestFul API Server.
咋一看他的封装好像很复杂, 但是如果了解 eventlet 的基本概念,
这部分代码其实非常简单.

eventlet 在它内部定义的 **greenthread(绿色线程)** 里,
它利用协程来实现并发. 协程的介绍可以看我以前写的文章: [Coroutine(协程)
介绍](http://mathslinux.org/?p%3D234)

为了彻底实现高性能的 I/O 并发, eventlet 甚至把底层的一些 API 都做了修改,
.e.g socket, listen. 更多的 eventlet 的内容不在这讨论, 可以去
[官网](http://www.eventlet.net) 查看相关的文档

下面是一个简单的小例子利用 eventlet 构建一个 HTTP Server, 基本上就是
glance 的 class Server 的核心骨架:

``` python
class APP(object):
    @webob.dec.wsgify
    def __call__(self, req):
        return 'hello world'

# 创建一个绑定在本机 8000 端口 的 socket.
sock = eventlet.listen(('0.0.0.0', 8000))
# 启动 wsgi server
eventlet.wsgi.server(sock, APP())
```

that's it, so easy. 把他保存到文件并运行, 就是一个简单的 Http server,
并且 我可以付责任的告诉你, openstack 中的 WSGI 服务就是这么运行的.

所以我们回过头看 **glance.common.wsgi.py:Server** 这个类,
它只不过包装了一下 eventlet 的参数, 并且 WSGI 实例是在子进程里面运行的,
支持多进程运行该服务(多核考虑), 对外提供了 **Server:start()** 和
**Server:wait()** 两个 API, 其他的, 就真没什么了.

paste 的工厂函数
================

请先复习一下 [paste](http://mathslinux.org/?p%3D596) 的相关内容.
然后看下面的图:

![](/images/posts/OpenStack/glance_source_1.png)

图中每一个 filter 都对应着一个工厂函数(可调用的实例), 比如:

``` python
[filter:versionnegotiation]
paste.filter_factory = glance.api.middleware.version_negotiation:VersionNegotiationFilter.factory
```

``` python
# file: glance/api/middleware/version_negotiation.py
class VersionNegotiationFilter(wsgi.Middleware):
```

对应的实例都是 **Middleware** 其实是的子类, 前面我们讲过,
paste配置里面的工厂函数 所创造的是一个能调用的实例(包含 `__call__`
方法), 当收到用户请求的时候就会自动调用该 实例, 也就是调用 `__call__`
方法. 这里 openstack(所有模块) 抽象除了一个类 `Middleware`, 封装了
`process_request` 方法. 每个 filter 子类只需要继承 该类, 如果需要做处理,
就覆盖 `process_request` 方法, 然后 `Middleware` 里面 的 `__call__`
方法会根据 `process_request` 的返回值来判断是否交给下一个 filter 处理.

``` python
class Middleware(object):
    def __init__(self, application):
        self.application = application

    @classmethod
    def factory(cls, global_conf, **local_conf):
        def filter(app):
            return cls(app)
        return filter

    def process_request(self, req):
        return None

    def process_response(self, response):
        return response

    @webob.dec.wsgify
    def __call__(self, req):
        # 首先调用子类的 process_request 方法, 如果子类没有实现这个方法或者
        # 返回值不为空, 那么直接将子类的返回回复给用户, 否则进行下一个 filter
        # 的处理. 这其实是一个递归的过程, 最后返回从下游(filter或app)的到的返回
        # 给用户
        response = self.process_request(req)
        if response:
            return response
        response = req.get_response(self.application)
        response.request = req
        try:
            return self.process_response(response)
        except webob.exc.HTTPException as e:
            return e
```
