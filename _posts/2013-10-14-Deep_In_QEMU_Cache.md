---
layout: post
tag: QEMU_KVM
date: '\[2013-10-14 一 15:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 深入分析 QEMU 的 cache 机制
---

Cache 的一些基本概念
====================

Cache 最先指的是[高速缓存](http://zh.wikipedia.org/wiki/Cache), 用于平衡
CPU 和内存之间的速度差异, 后来, 进一步发展为 一种技术,
用在速度相差较大的两种硬件之间, 用于协调两者数据传输速度差异的结构.

本文主要指的是协调内存和硬盘速度的 Cache.

同步在 Cache 和他的后端存储一般有以下几种方式:

-   Write through: 数据一旦在 Cache 中, 就马上同步到真实设备中,
    也就是保持同步
-   Write back: 见闻知意, writeback 的意思就是数据一旦在 Cache 中,
    这个操作就算完成了, 并不需要马上同步到真实设备中,
    除非用户手动指定(fsync), 或者此 Cache 中的内容发生了改变

QEMU 中的 Cache 模型
====================

由于虚拟化的关系, QEMU 中的 Cache 模型有一点复杂.

在虚拟化的世界中, 一个对象通常有两端, guest 端和 host 端, 或者称为
frontend 和 backend. 比如 vcpu 对象, 在 frontend 是一个 CPU, 在 backend
端, 它只是 一个线程, 对磁盘来讲, frontend 端看到 的是一个磁盘设备, 在
backend 端, 仅仅 是一个普通的文件而已.

所以 QEMU 中的 Cache, 就有两种情况, guest(frontend) 看到的 disk 的
Cache, 和 host(backend)看到的那个普通文件的 Cache. QEMU
需要对前者进行模拟, 对后者 需要管理. 后面会用代码详细解释 QEMU
是怎么实现的.

先遍历一下 QEMU 中 Cache 模式:

| Cache Mode   | Host Cache | Guest Disk Cache |
|--------------|------------|------------------|
| none         | off        | on               |
| writethrough | on         | off              |
| writeback    | on         | on               |
| unsafe       | on         | ignore           |
| directsync   | off        | off              |

实现的代码分析
==============

为了方便, 我选择使用 [IDE](http://en.wikipedia.org/wiki/Parallel_ATA) 的
device 和 raw 的磁盘格式.

host end 初始化 block 的流程
----------------------------

![](/images/posts/QEMU_KVM/deep_in_qemu_cache_1.png)

根据上面所说的,
模拟磁盘的初始化分为前端(磁盘设备)和后端(镜像文件)的初始化,
在后端的初始化中, 入口函数是 `drive_init()`, 到最后的 `qemu_open()`
终止. 上图 中红色得模块就是 Cache 初始化最重要的两个地方.

简单的分析一下流程.

``` c
int bdrv_parse_cache_flags(const char *mode, int *flags)
{
    *flags &= ~BDRV_O_CACHE_MASK;

    /*
     * - BDRV_O_NOCACHE: host end 绕过 cache
     * - BDRV_O_CACHE_WB: guest 的磁盘设备启用 writeback cache
     * - BDRV_O_NO_FLUSH: 在 host end 永远不要把 cache 里的数据同步到文件里
     * 这几个宏的具体应用后面分析到数据读写的时候会进行分析
     */
    if (!strcmp(mode, "off") || !strcmp(mode, "none")) {
        /* 由上, 这个组合表示的是 host end 不用 Cache, 数据直接在用户空间(QEMU)
         * 和真实设备之间通过 DMA 直接传输, 但是同时, 告诉 guest 模拟的磁盘设备
         * 是有 cache 的, guest 能发起 flush 的操作(fsync/fdatasync) */
        *flags |= BDRV_O_NOCACHE | BDRV_O_CACHE_WB;
    } else if (!strcmp(mode, "directsync")) {
        /* 很好理解, 完全没有 cache, host end 和 guest end 都没有 cache, guest
         * 不会发起 flush 的操作 */
        *flags |= BDRV_O_NOCACHE;
    } else if (!strcmp(mode, "writeback")) {
        /* 和上面相反, host side 和 guest side 都有 cache, 性能最好, 但是如果
         * host 掉电, 会导致数据的损失 */
        *flags |= BDRV_O_CACHE_WB;
    } else if (!strcmp(mode, "unsafe")) {
        /* 见文可知意, 最不安全的模式, guest side 有cache, 但是 host side 不理睬
         * guest 发起的 flush 操作, 完全忽略, 这种情况性能最高, snapshot 默认使用
         * 的就是这种模式 */
        *flags |= BDRV_O_CACHE_WB;
        *flags |= BDRV_O_NO_FLUSH;
    } else if (!strcmp(mode, "writethrough")) {
        /* host end 有 cache, guest 没有 cache, 其实通过后面的代码分析可以知道,
         * 这种模式其实是 writeback + flush 的组合, 也就是每次写操作同时触发
         * 一个 host flush 的操作, 会带来一定的性能损失, 尤其是非 raw(e.g. qcow2)
         * 的网络存储(e.g. ceph), 但是很遗憾, 这是 QEMU 默认的 Cache 模式 */
        /* this is the default */
    } else {
        return -1;
    }

    return 0;
}

DriveInfo *drive_init(QemuOpts *all_opts, BlockInterfaceType block_default_type)
{
    /* code snippet */

    value = qemu_opt_get(all_opts, "cache");
    if (value) {
        int flags = 0;

        /* 解析命令行 -drive 的 cache= 选项 */
        if (bdrv_parse_cache_flags(value, &flags) != 0) {
            error_report("invalid cache option");
            return NULL;
        }

        /* Specific options take precedence */
        if (!qemu_opt_get(all_opts, "cache.writeback")) {
            qemu_opt_set_bool(all_opts, "cache.writeback",
                              !!(flags & BDRV_O_CACHE_WB));
        }
        if (!qemu_opt_get(all_opts, "cache.direct")) {
            qemu_opt_set_bool(all_opts, "cache.direct",
                              !!(flags & BDRV_O_NOCACHE));
        }
        if (!qemu_opt_get(all_opts, "cache.no-flush")) {
            qemu_opt_set_bool(all_opts, "cache.no-flush",
                              !!(flags & BDRV_O_NO_FLUSH));
        }
        qemu_opt_unset(all_opts, "cache");
    }

    return blockdev_init(all_opts, block_default_type);
}
```

接下来的流程稍微复杂一点:

``` example
blockdev_init
  -> bdrv_open
    -> bdrv_file_open
      -> raw_open
        -> raw_open_common
          -> raw_parse_flags
          -> qemu_open
```

首先是从 `blockdev_init` 到 `raw_open` 的流程, 简单分析如下:

``` c
static DriveInfo *blockdev_init(QemuOpts *all_opts,
                                BlockInterfaceType block_default_type)
{
    DriveInfo *dinfo;

    /* code snippet 解析和配置各种各样的参数, e.g. 磁盘格式, 启动顺序等等,
     * 最后填充到 dinfo 对象中 */

    snapshot = qemu_opt_get_bool(opts, "snapshot", 0);

    file = qemu_opt_get(opts, "file");

    if (qemu_opt_get_bool(opts, "cache.writeback", true)) {
        bdrv_flags |= BDRV_O_CACHE_WB;
    }
    if (qemu_opt_get_bool(opts, "cache.direct", false)) {
        bdrv_flags |= BDRV_O_NOCACHE;
    }
    if (qemu_opt_get_bool(opts, "cache.no-flush", false)) {
        bdrv_flags |= BDRV_O_NO_FLUSH;
    }

    if (snapshot) {
        /* 前面讲过, snapshot 打开磁盘时候, 使用 unsafe 的 cache 模式 */
        bdrv_flags &= ~BDRV_O_CACHE_MASK;
        bdrv_flags |= (BDRV_O_SNAPSHOT|BDRV_O_CACHE_WB|BDRV_O_NO_FLUSH);
    }

    bdrv_flags |= ro ? 0 : BDRV_O_RDWR;

    /* 使用 bdrv_open 打开文件 */
    ret = bdrv_open(dinfo->bdrv, file, bs_opts, bdrv_flags, drv, &error);

    /* 返回配置好的 DriveInfo *dinfo, 这个对象在初始化模拟磁盘设备
     * 的时候被传入, 写入该磁盘设备的 PCI config space */
    return dinfo;
}

int bdrv_open(BlockDriverState *bs, const char *filename, QDict *options,
              int flags, BlockDriver *drv, Error **errp)
{
    BlockDriverState *file = NULL;
    const char *drvname;

    /* 打开文件, QEMU 支持的镜像格式都有一个后端的 format, 比如 raw 和
     * qcow2 这个 format 就是 file, 其他的还有 sheepdog, glusterfs 的
     * gluster 等. 所以这里其实是打开一个本地文件 */
    ret = bdrv_file_open(&file, filename, file_options,
                         bdrv_open_flags(bs, flags | BDRV_O_UNMAP), &local_err);

    /* 其实 bdrv_file_open() 就会调用 bdrv_open_common 函数, 只不过那个时候调用
     * bdrv_open_common() 用的是 file 这个 BlockDriver, 现在使用的是磁盘文件
     * format 的 BlockDriver(qcow2, raw 等), 所以这里的函数用来初始化特定格式的
     * 磁盘文件, 如 qcow2_open 等 */
    ret = bdrv_open_common(bs, file, options, flags, drv, &local_err);
}

int bdrv_file_open(BlockDriverState **pbs, const char *filename,
                   QDict *options, int flags, Error **errp)
{
    BlockDriverState *bs;

    /* 找到相应的格式的 BlockDriver, 由于这里是 file 的 open, 因为 file
     * 还没有被打开, 所以这里第二个指针传递的是空, 注意 drv 这个参数, 表示
     * file 的 BlockDriver */
    ret = bdrv_open_common(bs, NULL, options, flags, drv, &local_err);

    return ret;
}

static int bdrv_open_common(BlockDriverState *bs, BlockDriverState *file,
    QDict *options, int flags, BlockDriver *drv, Error **errp)
{
    bs->open_flags = flags;

    open_flags = bdrv_open_flags(bs, flags);

    /* 注意这里, flags 保存着之前 bdrv_parse_cache_flags 获取到的 flags,
     * 如果用户没有指定 none, writeback, 或者是 unsafe, 那么 guest 看到
     * 的这个磁盘设备是没有 cache 的, 后面我会以 hdparm 这个工具来验证,
     * 同时, 这个变量(bs->enable_write_cache)控制着 QEMU 怎么模拟 cache
     * 行为, 后面写文件的时候会分析到 */
    bs->enable_write_cache = !!(flags & BDRV_O_CACHE_WB);

    /* 打开文件, 因为从上面的流程可以看到, 这里作为 file 打开, 而不是
     * 作为 image format(e.g. qcow2) 打开, 所以这里调用的是 bdrv_file_open()
     * 方法, 也就是 raw-posix.c 中 format_name="file" 的 BlockDriver 里面的
     * 的 raw_open */
    if (drv->bdrv_file_open) {
        assert(file == NULL);
        assert(drv->bdrv_parse_filename || filename != NULL);
        ret = drv->bdrv_file_open(bs, options, open_flags, &local_err);
    } else {
        if (file == NULL) {
            error_setg(errp, "Can't use '%s' as a block driver for the "
                       "protocol level", drv->format_name);
            ret = -EINVAL;
            goto free_and_fail;
        }
        bs->file = file;
        ret = drv->bdrv_open(bs, options, open_flags, &local_err);
    }

    return ret;
}
```

最后剩下核心的 `raw_open` 函数, 这个函数和 host side 的 Cache 相关,
通过上面 的分析, 已经比较简单了.

``` c
/* 这个函数其实是 raw_open_common */
static int raw_open(BlockDriverState *bs, QDict *options, int flags,
                    Error **errp)
{
    BDRVRawState *s = bs->opaque;

    s->type = FTYPE_FILE;
    return raw_open_common(bs, options, flags, 0);
}

static int raw_open_common(BlockDriverState *bs, QDict *options,
                           int bdrv_flags, int open_flags)
{
    s->open_flags = open_flags;
    /* 解析 open 的参数, 把 QEMU 的 BDRV_O_* 映射成 open 的 O_*, 下面详细分析 */
    raw_parse_flags(bdrv_flags, &s->open_flags);

    s->fd = -1;

    /* 用上面解析到得参数打开文件 */
    fd = qemu_open(filename, s->open_flags, 0644);
    s->fd = fd;
}

static void raw_parse_flags(int bdrv_flags, int *open_flags)
{
    assert(open_flags != NULL);

    /* 首先清空其他标志 */
    *open_flags |= O_BINARY;
    *open_flags &= ~O_ACCMODE;
    /* 设置读写权限位 */
    if (bdrv_flags & BDRV_O_RDWR) {
        *open_flags |= O_RDWR;
    } else {
        *open_flags |= O_RDONLY;
    }

    /* 如果设置了 cache=none, 那么直接用 O_DIRECT 打开文件, 这个标志保证数据
     * 的传输将不会通过内核空间, 而是使用 DMA 直接在用户空间到存储设备之间
     * 传送, 不保证数据是否同步. 这就是 cache=none 的由来 */
    if ((bdrv_flags & BDRV_O_NOCACHE)) {
        *open_flags |= O_DIRECT;
    }
}
```

guest end cache 的设置
----------------------

在上面的的 `bdrv_open_common()` 函数中,
`会设置 BlockDriverState->enable_write_cache` 成员, 这个成员表示 guest
默认是否启用 writeback 的 Cache. 接下来会看到, guest
请求设备寄存器的时候, QEMU 会相应地用这个值填充寄存器的位, 下面以 IDE
硬盘来做例子.

guest 在初始访问 IDE 设备时, 会发送 IDENTIFY DRIVE (0xec) 指令,
设备收到这个 指令后, 需要返回 256 个字(512byte)的信息, 包括设备的状态,
扇区数目, 等等的信息.

``` c
static void ide_identify(IDEState *s)
{
    uint16_t *p;
    unsigned int oldsize;
    IDEDevice *dev = s->unit ? s->bus->slave : s->bus->master;

    memset(s->io_buffer, 0, 512);
    p = (uint16_t *)s->io_buffer;

    /* 看这里, bit 85 表示 writeback Cache 的状态, cache=none 之类的模式
     * 是会设置这个位的, 这样, guest 会发 fsync 指令过来, 否则, geust
     * 不会自动发送 fsync 同步数据, 当然, guest 可以在稍后设置是否启用
     * writeabck cache, QEMU 的新版本已经支持这个功能了. */
    /* 14 = NOP supported, 5=WCACHE enabled, 0=SMART feature set enabled */
    if (bdrv_enable_write_cache(s->bs))
        put_le16(p + 85, (1 << 14) | (1 << 5) | 1);
    else
        put_le16(p + 85, (1 << 14) | 1);

    memcpy(s->identify_data, p, sizeof(s->identify_data));
    s->identify_set = 1;
}
```

Cache 的工作模式
----------------

首先简单分析一下 guest 到 QEMU 的 I/O 执行路径:

![](/images/posts/QEMU_KVM/deep_in_qemu_cache_2.png)

下面的函数 `bdrv_co_do_writev(bdrv_co_do_rw)`, 很好的阐释了 QEMU
怎么模拟 磁盘的 cache

``` c
static int coroutine_fn bdrv_co_do_writev(BlockDriverState *bs,
    int64_t sector_num, int nb_sectors, QEMUIOVector *qiov,
    BdrvRequestFlags flags)
{
    if (ret < 0) {
        /* Do nothing, write notifier decided to fail this request */
    } else if (flags & BDRV_REQ_ZERO_WRITE) {
        ret = bdrv_co_do_write_zeroes(bs, sector_num, nb_sectors);
    } else {
        ret = drv->bdrv_co_writev(bs, sector_num, nb_sectors, qiov);
    }

    /* enable_write_cache 为 false, 即 cache=writethrough或者directsync, 那么
     * 每次写完都执行一次 flush 操作, 保证数据的同步, 当然, 这样会损失很大的性能 */
    if (ret == 0 && !bs->enable_write_cache) {
        ret = bdrv_co_flush(bs);
    }

    return ret;
}

int coroutine_fn bdrv_co_flush(BlockDriverState *bs)
{
    int ret;

    /* 对于 unsafe 的cache, 任何时候都不需要把数据真正同步到磁盘 */
    if (bs->open_flags & BDRV_O_NO_FLUSH) {
        goto flush_parent;
    }

    BLKDBG_EVENT(bs->file, BLKDBG_FLUSH_TO_DISK);
    if (bs->drv->bdrv_co_flush_to_disk) {
        ret = bs->drv->bdrv_co_flush_to_disk(bs);
    } else if (bs->drv->bdrv_aio_flush) {
        BlockDriverAIOCB *acb;
        CoroutineIOCompletion co = {
            .coroutine = qemu_coroutine_self(),
        };

        acb = bs->drv->bdrv_aio_flush(bs, bdrv_co_io_em_complete, &co);
        if (acb == NULL) {
            ret = -EIO;
        } else {
            qemu_coroutine_yield();
            ret = co.ret;
        }
    }

flush_parent:
    return bdrv_co_flush(bs->file);
}
```

那么对于 guest side, 如果磁盘有 cache, 那么 guest
是如何保证同步到磁盘上的:

guest 会根据磁盘的 cache 情况指定相应的调度策略,
来把数据从内存同步到磁盘 中, 或者用户手动指定 fsync, 这时也会发生 flush.

关于 host 刷新的机制, 没有什么好说的, 无非就是调用系统调用 fdatasync,
如果系统不支持 fdatasync, 那么调用 fsync.

``` bash
# hdparm -I /dev/sdc #cache=writethrough or directsync
Commands/features:
    Enabled Supported:
       *    SMART feature set
       *    Write cache
       *    NOP cmd
       *    48-bit Address feature set
       *    Mandatory FLUSH_CACHE
       *    FLUSH_CACHE_EXT
# hdparm -I /dev/sdc
Commands/features:
    Enabled Supported:
       *    SMART feature set
            Write cache
       *    NOP cmd
       *    48-bit Address feature set
       *    Mandatory FLUSH_CACHE
       *    FLUSH_CACHE_EXT

```

测试
====

使用 ubuntu-13.04 server 版本, 内核版本号 3.8.0-19. 24 cores, 20GB的
memory.

PS: 这里只是比较几种 cache 模式的性能, 不能作为专业的测试结果.

FIO 配置:

``` example
[test]
rw=rw
rwmixread=50
ioengine=libaio
iodepth=1
direct=1
bs=4k
filename=/dev/sdb
runtime=180
group_reporting=1
```

| Cache Mode   | Read IOPS                | Write IOPS               |
|--------------|--------------------------|--------------------------|
| none         | 293/235/218/200/200      | 292/234/215/199/200      |
| writethrough | 5/36/72/93/113           | 5/35/73/93/113           |
| writeback    | 1896/1973/1944/1627/2027 | 1894/1979/1947/1627/2023 |

writethough 的性能相当糟糕, 写的性能我还能理解, 读的性能完全不能理解,
仔细 想了一下, 应该是"混合读写"导致的, "写"拖慢了"读",
我随后单独做了一个随机读 的测试, IOPS 达到 四千多, 这次应该是正常的.

结论
====

通过以上的分析, writethrough 的性能很差, 因为它几乎相当于 writeback +
flush, writeback 的性能最好, 但是在掉电/迁移的情况下不能保证数据安全,
none 的读性能 很差, 因为他完全绕过了 kernel 的 buffer.

总的来说, 选择 cache 模式的时候, 至少要考虑以下几种情况:

-   你的存储是本地还是分布式的, 具体是那种存储, 支不支持 Direct
    IO(FUSE?), 一般来说网络文件系统使用 writethrough cache 性能很糟糕
-   是否需要迁移, 有的 cache 模式迁移会导致数据丢失.
