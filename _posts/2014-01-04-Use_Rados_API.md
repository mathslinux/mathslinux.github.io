---
layout: post
tag: Storage
date: '\[2014-01-04 六 15:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Rados API 的使用
---

Rados 介绍
==========

Rados 是 Ceph 最底层的组件, 基于对象存储模型, 向上提供 object, file
system, block 的存储方式. 这里我主要关注 rados block storage: rbd

rbd 用两种方式提供的客户端访问, 一种方式是是把客户端操作集成在内核中,
用户 只需要像使用其他文件系统一样挂载就可以直接使用了, 我在 [CEPH
初体验](http://mathslinux.org/?p%3D441) 里面有 简单的提及.

另一种方式是通过 librados 来访问, 这个库有各种语言支持, e.g. C, python,
java 等. QEMU 中对 Ceph 的支持即是通过这种方式来实现的.

本文主要讨论 librados(librbd) 的使用.

API 说明
========

为了使用 rbd, 主要需要使用以下几个 API:

``` c
/** 
 * 创建一个 rados_t 句柄, 该句柄存储了rados 客户端的数据结构, 用来和
 * rados 通信, 第二个参数是连接 rados 的客户端 ID
 * 
 * @param cluster 存储句柄的指针
 * @param id 连接 rados 的用户名
 * @return 
 */
int rados_create(rados_t *cluster, const char * const id);

/** 
 * 设置 rados 参数, 包括认证信息, monitor 地址等, 如果设置了 cephx 认证,
 * 那么之前创建 rados 句柄的时候, 必须设置客户端 ID, 并且必须设置 key 的密码
 * 
 * @param cluster 上面创建的 rados 句柄
 * @param option 
 * @param value 
 * @return 
 */
int rados_conf_set(rados_t cluster, const char *option, const char *value);

/** 
 * 完成了上面的设置之后, 使用 rados 句柄连接 rados 服务器
 * 
 * @param cluster 
 * 
 * @return 
 */
int rados_connect(rados_t cluster);

/** 
 * 成功连接上 rados 服务器之后, 就可以用 rados_ioctx_create() 打开 rados
 * 上的 pool, 该函数需要传递一个 rados_ioctx_t, 用来保存 IO 句柄
 * 
 * @param cluster rados 句柄
 * @param pool_name 需要打开的 pool
 * @param ioctx 保存 IO 句柄
 * 
 * @return 
 */
int rados_ioctx_create(rados_t cluster, const char *pool_name, rados_ioctx_t *ioctx);

/** 
 * 利用上面创建的关联到 pool 上的 IO 句柄打开具体的镜像
 * 
 * @param io IO 句柄
 * @param name 镜像名称
 * @param image 存储 image 句柄, 以后需要这个指针读写镜像
 * @param snap_name 打开镜像上的快照, 如果不是, 传递 null
 * @return 
 */
int rbd_open(rados_ioctx_t io, const char *name, rbd_image_t *image, const char *snap_name);

/** 
 * rados 支持异步操作, 当执行大量 I/O 的时候, 不需要等待每一个操作完成,
 * 只需要传递一个回调函数给读写函数, 当操作完成后, librados 会自动条用
 * 我们的回调函数. 下面的函数创建一个 rados 的异步句柄
 * 
 * @param cb_arg 传递给回调函数的参数
 * @param complete_cb 回调函数
 * @param c 存储异步句柄
 * 
 * @return 
 */
int rbd_aio_create_completion(void *cb_arg, rbd_callback_t complete_cb, rbd_completion_t *c);

/** 
 * 异步读的 API
 * 
 * @param image 上面谈到的 image 句柄
 * @param off 读写的文件位置
 * @param len 读写大小
 * @param buf 存储读写的数据
 * @param c 异步句柄
 * 
 * @return 
 */
int rbd_aio_read(rbd_image_t image, uint64_t off, size_t len, char *buf, rbd_completion_t c);
```

测试
====

完整的测试代码可以到
[这里](https://github.com/mathslinux/stuff-code/blob/master/rados.c)
下载, 我已经详细的写了程序注释.

首先, 我们创建一个测试的镜像, 写入一些数据, 看看我们的程序能不能读到:

``` bash
# 1. 创建一个名为 test 的 pool
# rados mkpool test
# 2. 在 test 中创建镜像 hello
# rbd create -p test --size 10 hello
# 3. 在 hello 中写一些数据(为了使用这个 rbd, 先 map 到我们的文件系统, 写入数据, 再 unmap)
rbd map test/hello
echo "hello, you are reading image hello which is in test pool, have fun" > hello.txt
dd if=hello.txt of=/dev/rbd/test/hello
rbd unmap /dev/rbd/test/hello
```

然后用我们的测试程序来读取上面写入的数据

``` bash
# ./rados test hello admin AQBhjqlSKBBTCxAAwchc9GauJ4+MPHz9hkV9Iw== 192.168.176.30:6789
open rados as following setting:
poolname: test
imagename: hello
username: admin
password: AQBhjqlSKBBTCxAAwchc9GauJ4+MPHz9hkV9Iw==
monitor: 192.168.176.30:6789
buffer read:
========================================
hello, you are reading image hello which is in test pool, have fun

========================================
```

Resources
=========

[RADOS Object Store
APIs](http://ceph.com/docs/master/rados/api/librados/)
