---
layout: post
tag: Spice
date: '\[2013-06-28 5 16:06\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: spice 客户端实现
---

目前上游社区支持的客户端是
[spice-gtk](http://cgit.freedesktop.org/spice/spice-gtk/),
它其实是一个库的项目, 编译后能得到两个库:

-   libspice-client-glib: spice client 的协议处理部分
-   libspice-client-gtk: 整合了 gtk 的更完善的实现

[virt-manager](http://virt-manager.org/) 通过 spice-gtk 实现了 spice
client 的功能, 可执行的程序称是 remote-view.

下面对现在市面上存在的各种客户端做一下分类和探讨, 其中对 spice-gtk
重点做 一下探讨, 毕竟这是东宫太子.

使用 spice-gtk
==============

如果要实现 spice 客户端, 最省事最简单的办法莫过于在 libspice-client-gtk
这个库的基础上进行开发了. 比如我曾经介绍过的 [编写简单的 Spice
Client](http://mathslinux.org/%3Fp%3D228)

但是由于以下种种原因, 基于 spice-gtk 的客户端可能不会是最佳的选择:

gtk3 不支持 X11 的共享内存
--------------------------

如果使用 gtk3, 那么渲染的后端只能是 cairo, 无法选择 X11, 同时意味者 X11
和应用程序的共享内存无法使用, 即每次做 bitmap 渲染的时候, 会在 spice
client 和 X11 server 之间多做一次内存拷贝, 损失不必要的性能.

某些平台没有 gtk
----------------

在其它平台并没有 gtk 实现, 比如 android, IOS 等, 要移植 gtk 到这些 平台,
不亚于重写 spice client.

性能的考虑
----------

同上面, 如果渲染后端使用 cairo, 在某些平台达不到最好的效果, 实际上, 在
Linux 下, 应该使用 X11 的共享内存技术, 在 windows 或者其它平台也应该使用
相应的技术.

以下几个客户端即是基于 libspice-client-gtk 实现的.

-   remote-view
-   spice-osx
-   spicy(just to be used to test spice-gtk)

使用 spice-glib
===============

libspice-client-glib 完整的实现了 spice 协议的客户端, 只不过用的 C 库是
glib 而已, 它完全和 gtk 无关. 理论上, 每个平台都应该调用
libspice-client-glib, 而图形的渲染 部分, 则由平台相关的 API 来实现,
也即实现 spice-widget.c 部分.

PS: 在 Bitmap 的处理部分, 应该使用平台相关的 API,
这样能达到最佳的性能(最大化的 使用显卡的加速功能), 比如在 linux(X86/ARM)
平台, 使用硬件加速(opengl?) 来处理 Bitmap, 在 windows 平台, 使用 GDI
来处理, 在 macos 平台(?). 遗憾的是,
虽然开发者留下了这些平台独立的接口和文件, 但是并没有完全实现,
这些接口都没有 用到, 目前的实现都是用统一的 pixman 库(纯 CPU 运算,
虽然对各个平台也会做一些优化) 操作, 具体的实现在 common/sw~canvas~.c.

言归正传, 下面描述怎么使用 libspice-client-glib, 大部分代码参照以前我在
android 上的 实现的 spice-client.

首先需要实现一个 display 的类(python/c++/c)用来做渲染,
在这个类的构造函数里面, 注册相应的信号处理函数(这些信号由 SpiceSession
在有事件到来的时候通知你), 例如:

``` c
AndroidDisplay *display;
AndroidDisplayPrivate *d;

// ......

if (!d->session) {
    g_error("AndroidDisplay constructed without a session");
}

spice_g_signal_connect_object(d->session, "channel-new",
                              G_CALLBACK(channel_new), display, 0);
spice_g_signal_connect_object(d->session, "channel-destroy",
                              G_CALLBACK(channel_destroy), display, 0);
```

其中 channel-new 这个信号, 表示 SpiceSession 监听到有新的
channel(display, input) 在的时候, 主动通知我们的 AndroidDisplay 对象,
我们使用 `channel_new()` 来处理这个 信号:

``` c
static void channel_new(SpiceSession *s, SpiceChannel *channel, gpointer data)
{
    AndroidDisplay *display = data;
    AndroidDisplayPrivate *d = ANDROID_DISPLAY_GET_PRIVATE(display);
    int id;

    g_object_get(channel, "channel-id", &id, NULL);
    if (SPICE_IS_MAIN_CHANNEL(channel)) {
        // ......
    }

    if (SPICE_IS_DISPLAY_CHANNEL(channel)) {
        SpiceDisplayPrimary primary;
        if (id != d->channel_id)
            return;
        d->display = channel;
        // 处理 primary 信号, 下面会讲到
        spice_g_signal_connect_object(channel, "display-primary-create",
                                      G_CALLBACK(primary_create), display, 0);
        spice_g_signal_connect_object(channel, "display-primary-destroy",
                                      G_CALLBACK(primary_destroy), display, 0);
        // 这个信号比较重要, 每次 guest 做任何 UI 的操作, blit, copy 等, 必然会
        // 重绘某些区域, 等到 SpiceDisplay 处理完 bitmap 的变换之后, 会给注册
        // 了这个信号的组件发送 display-invalidate 信号, 告诉这些组件哪些区域需要
        // 重绘.
        spice_g_signal_connect_object(channel, "display-invalidate",
                                      G_CALLBACK(invalidate), display, 0);
        spice_g_signal_connect_object(channel, "display-mark",
                                      G_CALLBACK(mark), display, G_CONNECT_AFTER | G_CONNECT_SWAPPED);
        if (spice_display_get_primary(channel, 0, &primary)) {
            primary_create(channel, primary.format, primary.width, primary.height,
                           primary.stride, primary.shmid, primary.data, display);
            mark(display, primary.marked);
        }
        spice_channel_connect(channel);
        spice_main_set_display_enabled(d->main, get_display_id(display), TRUE);
        return;
    }

    // input 通道, 处理鼠标键盘事件
    if (SPICE_IS_INPUTS_CHANNEL(channel)) {
        d->inputs = SPICE_INPUTS_CHANNEL(channel);
        spice_channel_connect(channel);
        return;
    }

    // handle other channels, e.g. curse, usb-redir
    return;
}
```

下面是 invalidate 的实现.

``` c
static void invalidate(SpiceChannel *channel,
    gint x, gint y, gint w, gint h, gpointer data)
{
    AndroidDisplay *display = data;
    AndroidDisplayPrivate *d = ANDROID_DISPLAY_GET_PRIVATE(display);
    android_pd = d;
    /* 正如上面讲到的, 这个是重绘的处理函数, 不同的平台这里可能会不同.
       如果是 gtk: 应该调用相应的 gtk_widget_queue_draw_area() 等函数通知上层
       UI 更新相应的显示区域. 因为这里 android 没有 gtk, 所以这里使用了 java
       的 JNI 技术通知 Android UI 需要重绘某些区域, 把控制权交给 Android UI. */
    spice_callback("OnGraphicsUpdate", "(IIII)V", x, y, w, h);

    /* 另外提醒一点, 所有的 bitmap 数据都存放在 d->data 里, 所以真正重绘的时候,
       可以从这里取数据, 如果后端的渲染是 cairo 的话, 每次 "display-primary-create
       发生的时候,  spice-gtk 会使用 cairo_image_surface_create_for_data()
       把 d->data 映射到它的画板(canvas)上去. */

    return ;
}
```

如果要写 IOS 或者 Android 客户端的话, 应该采用这种方式.

不依赖于其它模块
================

如果没有 glib 用, 或者其它的什么原因, 那么只能从头自己实现. spice-html5
就是 这么做的, 因为这是浏览器上运行的客户端, 编程语言是 javascript,
也没有 glib.

小玩具
======

如果对 spice-client 的渲染了如指掌的话, 可以做一些小 hack,
比如我喜欢用酷狗 来听音乐, 因为它的歌词显示得比较好, 但是我用的是 linux
操作系统, 酷狗音乐没有 linux 的版本.
所以如果能只显示虚拟机里面的酷狗歌词到我的 linux 桌面, 那也蛮帅的.

下面是相关的实现代码(trick and ugly):

``` c
// file: spice-widget.c
static void invalidate(SpiceChannel *channel,
                       gint x, gint y, gint w, gint h, gpointer data)
{
    // ......
    if (d->ad_drawing_area) {
        // 我在 SpiceDisplayPrivate 定义了一个 GtkWidget *ad_drawing_area 用来
        // 画歌词的窗口区域, hard code
        gtk_widget_queue_draw_area(d->ad_drawing_area, 0, 0, 1024, 113);
    }
}

// file: spice-widget-cairo.c
int spicex_image_create(SpiceDisplay *display)
{
    // other code
    if (d->ad_drawing_area) {
        // 我用歌词窗口的信息来创建在 linux 上的窗口的 bitmap 数据.
        d->ad_ximage = cairo_image_surface_create_for_data
                ((guint8 *)d->data + 1024 * 446 * 4, CAIRO_FORMAT_RGB24, 1024, 113, 1024 * 4);
    }
    return 0;
}

// file: spicy.c
static gboolean ad_draw_event(GtkWidget *widget, cairo_t *cr, void *opaque)
{
    int ww, wh;
    SpiceDisplayPrivate *d = opaque;

    ww = gdk_window_get_width(gtk_widget_get_window(widget));
    wh = gdk_window_get_height(gtk_widget_get_window(widget));

    cairo_rectangle(cr, 0, 0, ww, wh);

    cairo_set_source_surface(cr, d->ad_ximage, 0, 0);
    cairo_paint(cr);

    return TRUE;
}

static SpiceWindow *create_spice_window(spice_connection *conn, SpiceChannel *channel, int id, gint monitor_id)
{
    // other codes
    d->ad_drawing_area = gtk_drawing_area_new();
    // 注册 draw 信号, 当 gtk_widget_queue_draw_area() 被调用的时候,
    // 会触发 draw 信号, 表示某些区域需要被更新.
    g_signal_connect(d->ad_drawing_area, "draw",
                     G_CALLBACK(ad_draw_event), d);
    // other codes
}
```

附上截图一张

```{=html}
<a href="http://mathslinux.org/wp-content/uploads/2013/06/kugou-geci-redir.png"><img src="http://mathslinux.org/wp-content/uploads/2013/06/kugou-geci-redir.png" alt="kugou-geci-redir" title$
```
