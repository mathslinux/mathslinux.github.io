---
layout: post
tag: Spice
date: '\[2012-11-01 4 11:11\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Spice Server 架构分析
---

概述
====

Spice 是一个 [Virtual Device
Interfaces(VDI)](http://spice-space.org/vdi.html) 的库, 它以 libspice
库的形式提供给 VDI 后端, 这个后端可以是 [QEMU](http://www.qemu.org),
或者 Xorg 等.

架构模型
========

根据上面提到的 VDI 的概念, 它是一个非常复杂的模型, 包括无数的组件:
显示模块, 输入输出模块, 各种 guest agent模块, 所以需要一个精心设计
的架构来使得各个模块流畅的运行.

其实处理 VDI 的这种编程模型, 有两种比较流行的架构模型

[事件驱动架构](http://en.wikipedia.org/wiki/Event-driven_programming)

:   在一个 main loop 里面通过 select 或者 poll 各种句柄, 来对
    用户的事件进行处理. 典型的比如大部分基于 gtk 程序.

[并行处理架构](http://en.wikipedia.org/wiki/Parallel_computing)

:   开辟多个进程/线程分别处理不同的模块.

Spice server 严格说来属于上面的事件驱动架构,
即它是通过抛出一系列的文件句柄给 调用者, 然后在 mainloop(甚至 几乎所有
mainloop 都抛出给caller) 上轮询这些 句柄, 实现他的架构的. 这个架构写过
GTK 或者用过 libevent 的人应该不会陌生.

SS 通过提供接口的注册函数(`spice_server_add_interface()`), 使得 caller
可以把想要处理的模块注册到 SS 系统中, 比如想要实现声音的播放, 需要注册
SpicePlaybackInterface 实例到 SS 系统中, 当不想用这个模块了, 可以用
`spice_server_remove_interface()` 将它从 SS 中移除.

在所有的模块中, 其中有一个接口(Core interface)比较特殊, 这个接口是在 SS
初始化的时候被注册的. 这个接口主要用于监听 Spice client 的连接请求, 管理
channels 的注册和移除等.

初始化
======

SS 提供了一个 API spice~serverinit~() 来初始化 SS. 它的原型如下:

``` c
/** 
 * 初始化 SS
 * 
 * @param s SS 的实例, 用 spice_server_new() 创建
 * @param core 前面提到的 Core interface
 * 
 * @return 
 */
int spice_server_init(SpiceServer *s, SpiceCoreInterface *core);
```

其中 SpiceCoreInterface 这个 API 的核心. 下面重点介绍:

``` c
struct SpiceBaseInterface {
    /* 接口类型 */
    const char *type;
    /* 接口描述 */
    const char *description;
    /* 接口的主版本号, 初始化这个接口的时候会判断用户传递的值和 SS 允许的
     是否相等, 不相等初始化失败 */
    uint32_t major_version;
    /* 接口的次版本号, 初始化的时候比较用户传递的和 SS 允许的大小, 大于
     SS 允许的则初始化失败 */
    uint32_t minor_version;
};

struct SpiceCoreInterface {
    /* 接口信息, 见上面的分析 */
    SpiceBaseInterface base;

    /* 添加一个定时器, 旧版本的 SS 会用这个 API 自动创建一个 timer, 新版本要求
     * caller 传递 timer 给这个 API, 这个 timer 主要用来在超时的时候调用
     * callback, 超时时间是由下面的 timer_start() 设置的 */
    SpiceTimer *(*timer_add)(SpiceTimerFunc func, void *opaque);

    /* 设置超时时间, 在 ms 毫秒之后调用 callback */
    void (*timer_start)(SpiceTimer *timer, uint32_t ms);

    /* 取消上面设置的超时时间 */
    void (*timer_cancel)(SpiceTimer *timer);

    /* 移除 timer */
    void (*timer_remove)(SpiceTimer *timer);

    /* 添加一个 watch, 前面说过, SS 是事件驱动模型的, SS 会在内部创建事件
     * 源(客户端连接),把这些事件的句柄通过文件的形式做成接口, caller
     * 实现这些接口, 比如 用 select 轮询这些句柄. */
    SpiceWatch *(*watch_add)(int fd, int event_mask, SpiceWatchFunc func, void *opaque);
    /* 修改文件句柄的读写属性 */
    void (*watch_update_mask)(SpiceWatch *watch, int event_mask);
    /* 移除对该文件句柄的g监听 */
    void (*watch_remove)(SpiceWatch *watch);

    /* 当有一些特殊的事件发生时, 会调用这个 API, 比如新客户端连接, 断开等,
     * caller 可以根据自己的需求做一些操作, 比如 QEMU 会创建一个 QMP 消息
     * 等 */
    void (*channel_event)(int event, SpiceChannelEventInfo *info);
};
```

注册接口
========

上面说过, 通过注册接口的函数, 可以把想要处理的模块添加到 SS 系统中, 这个
API 就是:

``` c
int spice_server_add_interface(SpiceServer *s, SpiceBaseInstance *sin);
```

以注册键盘为例:

``` c
static void kbd_push_key(SpiceKbdInstance *sin, uint8_t frag)
{
}

static uint8_t kbd_get_leds(SpiceKbdInstance *sin)
{
}

static const SpiceKbdInterface kbd_interface = {
    .base.type          = SPICE_INTERFACE_KEYBOARD,
    .base.description   = "keyboard",
    .base.major_version = SPICE_INTERFACE_KEYBOARD_MAJOR,
    .base.minor_version = SPICE_INTERFACE_KEYBOARD_MINOR,
    /* Client的键盘输入 */
    .push_scan_freg     = kbd_push_key,
    /* Client查询 led 的状态 */
    .get_leds           = kbd_get_leds,
};

SpiceKbdInstance *kbd;
kbd = g_malloc0(sizeof(*kbd));
kbd->base.sif = &kbd_interface.base;
/* 调用 API 注册 */
spice_server_add_interface(spice_server, &kbd->base);
```
