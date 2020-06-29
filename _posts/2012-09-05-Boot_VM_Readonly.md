---
layout: post
tag: QEMU_KVM
date: '\[2012-09-05 3 17:09\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 以只读方式启动虚拟机
---

有时候想在虚拟机里面做一些测试, 又不想让这些测试损坏虚拟机镜像, 这时候就
需要能有一种只读的方式可以用来启动虚拟机.

其实, QEMU 支持各种各样的快照模式, Live Snapshot, Temporary Snapshot
等等. 利用这些模式, 就可以实现上面所说的功能.

创建一个快照
============

用 qemu-img 指令创建一个原始镜像的快照.

``` bash
$ qemu-img create -f qcow2 -b vm-base.img vm-append.img
```

vm-base.img 是原始的镜像, vm-append.img 是快照名字. 之后对 VM
所做的所有修改都只会改变 vm-append.img, vm-base.img 将是只读的.

启动 VM 的时候, 使用创建的这个快照文件即可.

``` bash
$ qemu-kvm -hda vm-append.img
```

据我所知, 有些基于 QEMU/KVM 的虚拟化管理平台(e.g. oVirt)有一种叫
stateless 的启动模式, 就是用的上面的方式, 启动虚拟机之前创建一个快照,
虚拟机结束之后再把快照删除.

NB: raw 不支持快照, 所以最好用 qcow2 的镜像格式(qed 也可以)

使用临时快照
============

用上面的方式来实现 Read-only 略显麻烦, 比如每次启动虚拟机的时候都要手动
创建一个快照, 感觉有点奇怪, 再比如更严重的, QEMU 异常退出怎么办?
快照文件 岂不是会占用而外的空间, 难道需要开启另外一个 daemon 程序监控
QEMU?

好在, QEMU 支持另一种快照模式 - 临时快照, 这种模式不需要经过 1)创建快照,
2)指定快照名称, 3)VM 关机后删除快照就可以实现上述功能.

使用也很简单, 在 QEMU 启动的时候, 增加一个 -snapshot 的参数就可以了.

``` bash
$ qemu-kvm -hda vm.img -snapshot
```

下面简叙一下原理: 如果传递了 -snapshot 的参数, 会在初始化 img 的时候创建
一个临时的快照. 并且在具体打开这个 img 的时候把这个文件删除. 请注意,
Linux 允许一个文件打开之后删除, 其实只是删除了文件名,
对这类文件的正常操作, 内核会有一个比较优雅的方式来处理.

但是为什么要删除呢而不是让用户visable呢? 在我看来有两点:

1.  一旦这个进程结束(正常或者是异常), 这个文件的内容马上就被系统回收,
    不会造成空间浪费. 感觉是不是比上面的方式优雅?
2.  处于安全性的考虑, 一旦文件被删除以后, 除了这个进程(VM进程), 其它
    是不可能打开和操作这个文件的, 文件的安全性得到保障. 之前在邮件列表
    上看到有人提到把这个文件名让用户来设置, 看了它的代码实现后, 发觉
    QEMU 的开发者想问题的时候还是很靠谱的.(当然, 这个提议没有通过)

之后, 如果用户想要把快照的内容写回去, QEMU 提供了方式可以写回去的方式,
例如在 nographic 模式下用 Ctrl-a s 把数据写回去, 过多的 write back
不再讨论, 毕竟这里主要研究"只读".

``` c
int get_tmp_filename(char *filename, int size)
{
    int fd;
    const char *tmpdir;
    tmpdir = getenv("TMPDIR");
    if (!tmpdir)
        tmpdir = "/tmp";
    if (snprintf(filename, size, "%s/vl.XXXXXX", tmpdir) >= size) {
        return -EOVERFLOW;
    }
    fd = mkstemp(filename);
    if (fd < 0 || close(fd)) {
        return -errno;
    }
    return 0;
}

/* 检查用户是否启用临时快照 */
if (flags & BDRV_O_SNAPSHOT) {
    ret = get_tmp_filename(tmp_filename, sizeof(tmp_filename));
}

/* 打开文件的时候检查是否是临时快照, 如果是, 把文件删除 */
static int bdrv_open_common(BlockDriverState *bs, const char *filename,
    int flags, BlockDriver *drv)
{
    if (bs->is_temporary) {
        unlink(filename);
    }
}
```
