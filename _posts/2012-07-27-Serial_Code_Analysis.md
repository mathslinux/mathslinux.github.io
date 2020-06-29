---
layout: post
tag: QEMU_KVM
date: '\[2012-07-27 5 22:07\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QEMU 中串口实现的代码分析
---

QEMU 中的串口实现分为两个部分, 一个是 QEMU 为 Guest 模拟的 `串口设备`,
称为前段, 是 Guest 操作的部分, 一个是 QEMU 怎么通知该串口与外部通信,
也就是 QEMU 怎么把串口的输入输出定向到其它设备, 称为后端.

本文档讨论 QEMU 重定向 串口输入输出到 stdio 的情况, 分析串口的模拟实现,
怎么和 stdio 通信, 其它情况类似.

注意, 此时的命令行参数为

``` bash
$ qemu-kvm debian.img -serial stdio
```

创建后端 stdio
==============

解析参数
--------

![](/images/posts/QEMU_KVM/serial_code_analysis_1.png)

从入口函数来看, 其实很简单, QEMU 解析到用户传递了 `-serial stdio`,
会调用 `add_device_config` 函数向 `device_configs` 链表 添加一个设备,
方便以后 索引使用

``` c
struct device_config {
    enum {
        DEV_USB,       /* -usbdevice     */
        DEV_BT,        /* -bt            */
        DEV_SERIAL,    /* -serial        */
        DEV_PARALLEL,  /* -parallel      */
        DEV_VIRTCON,   /* -virtioconsole */
        DEV_DEBUGCON,  /* -debugcon */
        DEV_GDB,       /* -gdb, -s */
    } type;
    const char *cmdline;
    Location loc;
    QTAILQ_ENTRY(device_config) next;
};
case QEMU_OPTION_serial:
    add_device_config(DEV_SERIAL, "stdio");
```

然后, 等到 QEMU 处理完命令行的解析之后, 它开始初始化各种模块,
比如串口后端的 初始化模块

``` c
static int serial_parse(const char *devname)
{
    /* ...... */
    serial_hds[index] = qemu_chr_new(label, devname, NULL);
    /* ...... */
}

int main(int argc, char **argv, char **envp)
{
    /* ...... */
    if (foreach_device_config(DEV_SERIAL, serial_parse) < 0)
        exit(1);
    /* ...... */
}
```

创建后端 stdio
--------------

通过上面的分析, 创建后端是由 `qemu_chr_new()` 完成的

![](/images/posts/QEMU_KVM/serial_code_analysis_2.png)

``` c
CharDriverState *qemu_chr_new(const char *label, const char *filename, void (*init)(struct CharDriverState *s))
{
    /* ...... */
    CharDriverState *chr;
    opts = qemu_chr_parse_compat(label, filename);
    chr = qemu_chr_new_from_opts(opts, init);
    return chr;
    /* ...... */
}
```

首先, `qemu_chr_parse_compat` 从 `vm_config_groups` 中找到 chardev
的链表, 然后创建一个名叫 "serial0" 的 `qemu_chardev_opts` 挂到该链表上

``` c
QemuOpts *qemu_chr_parse_compat(const char *label, const char *filename)
{
    /* ...... */
    opts = qemu_opts_create(qemu_find_opts("chardev"), label, 1, &local_err);

    /* ...... */
    return opts;
```

然后, 使用上面创建的 opts 创建真正的 stdio 后端

``` c
CharDriverState *qemu_chr_new_from_opts(QemuOpts *opts,
                                    void (*init)(struct CharDriverState *s))
{
    CharDriverState *chr;

    /**
     * 从 backend_table 中找到 CharDriverState:
     * { .name = "stdio",     .open = qemu_chr_open_win_stdio }
     */
    for (i = 0; i < ARRAY_SIZE(backend_table); i++) {
        if (strcmp(backend_table[i].name, qemu_opt_get(opts, "backend")) == 0)
            break;
    }

    /* 调用 qemu_chr_open_stdio() 初始化设备 */
    chr = backend_table[i].open(opts);
    /* 插入到 chardevs 中, 这个链表是一个全局的字符设备链表 */
    QTAILQ_INSERT_TAIL(&chardevs, chr, next);

    return chr;
}
```

初始化 stdio
------------

``` c
static CharDriverState *qemu_chr_open_stdio(QemuOpts *opts)
{
    CharDriverState *chr;

    /* 从标准输入0, 输出1创建一个字符设备, 在里面会将写函数的 callback 指向
       fd_chr_write */
    chr = qemu_chr_open_fd(0, 1);

    /* 主要是向 main loop event 注册标准输入的读函数 */
    qemu_set_fd_handler2(0, stdio_read_poll, stdio_read, NULL, chr);
    return chr;
}
```

至此, stdio 的创建就完成了

串口模拟实现
============

串口是通过 QOM(QEMU Object Model) 实现的, QOM 的讨论超出了本文的范围,
下面只讨论 isa-serial 相关的实现.

isa-serial class 定义在 `serial.c` 中:

在初始化 machine 的时候(vl.c 中的 machine-\>init()), 会自动调用
`serial_isa_init()` 来初始化串口设备.

类的初始化
----------

在初始化 isa-serial 的时候, 会把 `serial_isa_init` 赋值给 该类的 init
指针, 这个其实可以理解为该类的构造函数, 在创建实例的时候,
自动调用该函数.

``` c
/* 类初始化函数 */
static void serial_isa_class_initfn(ObjectClass *klass, void *data)
{
    ic->init = serial_isa_initfn;
}
```

实例的初始化
------------

实例的初始化用上面提到的 `serial_isa_initfn` 函数完成

![](/images/posts/QEMU_KVM/serial_code_analysis_3.png)

实例的初始化稍微复杂一些:

-   首先设置波特率
-   然后是中断的初始化, `isa_init_irq`
-   然后调用一个叫 `serial_init_core` 的函数, 该函数初始化定时器, 将
    它的后端(这里是stdio)添加到监听事件的loop中, 这个后端就是上面创建 的
    stdio, 通过构造函数的参数传递进来
-   最后会向 ISA Bus 注册 I/O 端口. 这个过程分两部, 首先申请一个 8 Bits
    的内存, 然后调用 `isa_register_ioport` 把这部分内存作为 I/O port,
    基地址 为 { 0x3f8, 0x2f8, 0x3e8, 0x2e8 } 中的一个, 在这个 I/O port
    上, 还会 注册两个回调函数, `serial_ioport_read` 和
    `serial_ioport_write`, 其中 read 会在 guest 请求读 I/O
    端口的时候触发, write 会在 guest 写 I/O 的时候 触发.

两个设备通信
============

Guest 能访问模拟的串口, Guest 发往该串口的任何数据都会被 QEMU 捕获到,
然后 QEMU 会把真实的数据(除去控制信息外的数据)发往后端设备(这里是
stdio).

相反的, stdio 发往串口的数据也会被 QEMU 捕获, 然后 QEMU 会引发一个中断,
通知 Guest 有新的数据, 把数据送往串口.

下面分两种情况具体讨论:

Guest 发送数据
--------------

![](/images/posts/QEMU_KVM/serial_code_analysis_4.png)

由上面的分析, 在串口初始化的时候, 在 I/O 端口注册了
`serial_ioport_write` 的函数, 当这个 I/O 端口有数据写入的时候,
会自动调用该函数.

-   首先, 判断是否是第 0 bit, 根据协议, 0 bit 代表 THR, 将数据用
    `fifo_put` 函数放入队列中缓存
-   然后, 触发一个硬件中断, 其实由于我们用的是 stdio 后端, 无用.
-   QEMU 会最后会调用 `serial_xmit` 函数来进行下一步的处理, 在该函数中,
    首先 用 `fifo_get` 把缓存中的数据取出来, 然后发给调用
    `qemu_chr_fe_write=(最终
     调用 write(1, buf, len)) 在stdio中显示出来, 关键在于 =qemu_chr_fe_write`
    中传递的参数 s-\>chr,
    这是在初始化串口实例的时候传递的后端设备的指针(stdio)

Guest 获取数据
--------------

![](/images/posts/QEMU_KVM/serial_code_analysis_5.png)

类似 Guest 发送数据, QEMU 捕获到 I/O 端口上 Guest 请求读数据, 自动调用
`serial_ioport_read()` 函数把数据从缓存中读出来发给 Guest.

-   首先, 判断是否是第 0 bit, 根据协议, 0 bit 代表 RBR, 用 `fifo_get`
    函数 把数据从缓存中读出来
-   然后, 触发一个硬件中断
-   返回从缓存中读出的数据

stdio 发送数据
--------------

![](/images/posts/QEMU_KVM/serial_code_analysis_6.png)

-   首先, main~loop~ 监听到 stdio 有数据可读, 调用 stdio 后端注册的
    `stdio_read` 函数读取数据, 最后调用 `qemu_chr_be_write`, 发送数据
-   `qemu_chr_be_write` 最终会调用 `serial_receive1` 函数来获取该数据,
    这是通过 串口初始化的时候以下代码实现的.
-   最后 `serial_receive1` 把数据保存在缓存中, 触发一个中断

```{=html}
<!-- -->
```
``` c
qemu_chr_add_handlers(s->chr, serial_can_receive1, serial_receive1,
                      serial_event, s);
```

stdio 接收数据
--------------

其实就是 Guest 发送数据后, 接受的过程, 见前面的分析
