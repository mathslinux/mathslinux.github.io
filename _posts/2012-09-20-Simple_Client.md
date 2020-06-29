---
layout: post
tag: Spice
date: '\[2012-09-20 4 14:09\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 编写 Spice Client
---

在 [spice-gtk](http://spice-space.org/page/Spice-Gtk) 的帮助下, spice
client 的编写非常简单. 以致于我在做 Spice Server 的测试的时候,
顺手写了一个简单的 spice client.

把下面的一些核心部分做一个剖析:

``` c
static void channel_new(SpiceSession *s, SpiceChannel *c, gpointer *data);

/* 创建一个 Spice session */
spice_session = spice_session_new();

/* 设置 Spice 的地址和端口 */
g_object_set(spice_session, "host", host, NULL);
g_object_set(spice_session, "port", port, NULL);

/* 设置当 Spice Channel 建立之后的 callback, 也就是说这个时候可以
 * 获取 Spice Gtk 建立的 Spice Widget, 包括 Spice Window 等 */
g_signal_connect(spice_session, "channel-new",
                 G_CALLBACK(channel_new), NULL);

/* 最后调用这个 API, 连接 Server 就可以了 */
spice_session_connect(spice_session);

static void channel_new(SpiceSession *s, SpiceChannel *c, gpointer *data)
{
    int id = 0;

    /* 获取通道 ID */
    g_object_get(c, "channel-id", &id, NULL);

    if (SPICE_IS_DISPLAY_CHANNEL(c)) {
        /* 对 Display 通道, 获取 spice window, 然后把它加入我们的容器(主窗口,
         * VBox 等), 这里的 main_window 是我用 gtk_window_new() 创建的主窗口 */
        spice_display = spice_display_new(s, id);
        gtk_container_add(GTK_CONTAINER(main_window), GTK_WIDGET(spice_display));
        gtk_widget_show_all(main_window);
    }

}
```

事实上, 核心代码不超过 10 行, 如此的简单, 当然更多的鼠标, 键盘事件, USB
重定向 等, 只要自己写 signal 的 callback 就可以了.

完整的代码在 [我的github](https://github.com/mathslinux) 上:

``` bash
$ git clone git://github.com/mathslinux/qemu-tools
$ cd qemu-tools/spice-tools/
$ make
$ ./spice-client
```
