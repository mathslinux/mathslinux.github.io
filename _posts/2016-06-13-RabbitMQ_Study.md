---
layout: post
tag: AMQP
date: '\[2016-06-13 一 10:45\]'
title: RabbitMQ 学习
---

Basic
=====

RabbitMQ 是
[AMQP](https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol)
的一种实现.

基本概念:

-   **Producer** 发送消息方
-   **Consumer** 消费者
-   **Queue** 消息队列
-   **Message**
-   **Connection**
-   **Channel**
-   **Exchange**
-   **Binding**
-   **Routing key** 消息的一个属性(消息的地址), exchange
    使用该属性来决定将该消息发往哪个 queue
-   **AMQP**
-   **Users**
-   **Vhost, virtual host**

Exchanges
---------

### Types of exchanges

-   **Direct** 将 message 发送到和 message 的 routing key 相同的 queue
-   **Fanout** 将 message 发送到所有绑定到该 exchange 的 queue
-   **Topic** 在做 routing 的时候使用通配符规则
-   **Headers** routing 的时候, 比较 message 的 headers 的属性和 queue
    的参数, 将 message 发送到符合条件的 queue. message 有一个默认值为
    all 的 **x-match** 属性, 决定比较属性的时候是否要求满足所有的属性.
-   **Dead Letter** 如果没有任何一个 queue 满足条件, message 会被丢弃,
    RabbitMQ 有一个 名为 **Dead Letter Exchange** 的扩展,
    用来捕获所有丢弃的 message

### Exchanges 的属性

-   **durable** 永久存在直到被明确的删除. 即使服务重启, 消息也不会丢
-   **temporary** 永久存在直到服务停止
-   **auto delete** 最后一个绑定到改 exchanges 的对象解绑之后, 该
    exchanges 被删除.

queue 和 message
----------------

### ACK 机制

### 持久化

RPC
===

Installation
============

``` bash
# yum install -y rabbitmq-server
# systemctl restart rabbitmq-server
# rabbitmqctl status
```

一般使用流程
============

1.  建立连接
2.  打开 channel
3.  创建 queue
4.  在消费者/订阅者端: 创建 exchange, 绑定 queue 到 exchange
5.  对生产者: 传递消息到 exchange, 对消费者/订阅者: 从队列获取数据
6.  关闭连接

示例代码
========

Management Interface
====================

Usage
=====

插件管理
========

UI 管理插件
-----------

``` bash
# rabbitmq-plugins -h
# rabbitmq-plugins enable rabbitmq_management
# systemctl restart rabbitmq-server
# http://server-name:15672/
```

HeartBeats
==========

<https://www.rabbitmq.com/heartbeats.html>

OpenStack Bug
=============

<http://way4ever.com/?p=3121>
<https://bugs.launchpad.net/nova/+bug/856764>

Resources
=========

[RabbitMQ for
beginners](https://www.cloudamqp.com/blog/2015-05-18-part1-rabbitmq-for-beginners-what-is-rabbitmq.html)
<http://blog.ftofficer.com/2010/03/translation-rabbitmq-python-rabbits-and-warrens/>
<http://liuvblog.com/2015/12/10/kombu-library-study-1/>
<http://bingotree.cn/?cat=4>
