---
layout: post
tag: Android
date: '\[2013-04-01 一 17:59\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Port glib to android
---

[glib](https://developer.gnome.org/glib/) 是 linux 下非常基础的库,
大部分 linux 下的软件 都依赖于它, 比如 gstreamer, gtk 等等.
由于最近我在准备 hack [spice](http://mathslinux.org/?cat%3D39), 准备把它
port 到 android 上, 而 libspice 又依赖于 glib, 所以需要把 glib 移植到
android 上.

所幸几年前就 hack 过大量程序到 ARM 和
[blackfin](http://en.wikipedia.org/wiki/Blackfin) 平台上,
所以过程还算顺利.

下载相应的文件
==============

需要用到的源代码有以下几个:

libiconv
:   1.14, 从
    [这里](http://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz)
    下载

gettext
:   0.18.2, 从
    [这里](http://ftp.gnu.org/pub/gnu/gettext/gettext-0.18.2.tar.gz)
    下载

libffi
:   3.0.12, 从
    [这里](ftp://sourceware.org/pub/libffi/libffi-3.0.12.tar.gz) 下载

glib
:   2.34.3, 从
    [这里](http://ftp.gnome.org/pub/gnome/sources/glib/2.34/glib-2.34.3.tar.xz)
    下载

由于有的版本(比如 libiconv-1.14, 旧的 glib)暂时不支持自动探测 android
host, 所以需要对探测的脚本做一些修改, 这里使用最简单的方法, 使用 gnu
网站上最新的 探测脚本替换不支持的脚本, 先把脚本下载下来, 需要是替换

``` bash
$  wget -O /tmp/config.sub "git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub;hb=HEAD"
$  wget -O /tmp/config.guess "git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD"
```

编译环境
========

从 [这里](http://developer.android.com/tools/sdk/ndk/index.html)
根据操作系统的版本下载相应的 toolchain版本, 我下载的是
android-ndk-r8d-linux-x86.tar.bz2, 解压到合适的目录, 然后进入该目录,
安装 toolchain, (在这里我将 toolchain 安装在
\${HOME}/Develop/android-toolchain, 使用 gcc-4.7, android-14 的 platform

``` bash
$ tar xf android-ndk-r8d-linux-x86.tar.bz2
$ cd android-ndk-r8d
$ build/tools/make-standalone-toolchain.sh --platform=android-14 \
--toolchain=arm-linux-androideabi-4.7 \
--install-dir=${HOME}/Develop/android-toolchain
```

在 bash 的配置文件里面配置 toolchain 的相关路径

``` bash
$ echo "export SYSROOT=${HOME}/Develop/android-toolchain/sysroot" >> ${HOME}/.bashrc
$ echo 'export PATH=${PATH}:${HOME}/Develop/android-toolchain/bin' >> ${HOME}/.bashrc
$ source ${HOME}/.bashrc
```

compile libiconv
================

libiconv 目前的最新版本是 1.14, 还不支持 android, 需要做一些 hack

将上面下载的 config.sub 和 config.guest 文件替换原有的文件,
因为目前还不支持 自动探测 android host.

``` bash
$ cp /tmp/config.sub build-aux/config.sub 
$ cp /tmp/config.sub libcharset/build-aux/config.sub 
$ cp /tmp/config.guess build-aux/config.guess 
$ cp /tmp/config.guess libcharset/build-aux/config.guess 
```

另外, 由于 libiconv 自带的 stdint.h 和 android toolchain 的冲突,
导致使用 `time_t` 等几个结构体的时候会出现类型未申明的情况, 比如:

``` example
In file included from /root/Develop/android-toolchain/sysroot/usr/include/sys/time.h:33:0,
                 from /root/Develop/android-toolchain/sysroot/usr/include/time.h:32,
                 from ./time.h:40,
                 from ./stdint.h:518,
                 from /root/Develop/android-toolchain/bin/../lib/gcc/arm-linux-androideabi/4.7/include-fixed/sys/types.h:43,
                 from ./fcntl.h:46,
                 from careadlinkat.h:23,
                 from areadlink.c:27:
/root/Develop/android-toolchain/sysroot/usr/include/linux/time.h:20:2: error: unknown type name 'time_t'
/root/Develop/android-toolchain/sysroot/usr/include/linux/time.h:26:2: error: unknown type name 'time_t'
/root/Develop/android-toolchain/sysroot/usr/include/linux/time.h:27:2: error: unknown type name 'suseconds_t'
```

在 configure 中预定义 gl~cvheaderworkingstdinth~=yes 可以避开这个问题.

以下是编译的具体指令:

``` bash
$ gl_cv_header_working_stdint_h=yes ./configure --prefix="${SYSROOT}/usr" --host=arm-linux-androideabi CFLAGS="--sysroot $SYSROOT" --enable-static
$ make -j5
$ make install
```

现在, 应该可以在 \${SYSROOT}/usr/lib/ 下看到相关的库已经安装了

``` bash
$ ls ${SYSROOT}/usr/lib/libiconv*
```

compile gettext
===============

android toolchai 的 passwd 结构体没有 `pw_gecos` 成员. 会报以下错误:

``` example
msginit.c: In function 'get_user_fullname':
msginit.c:1084:21: error: 'struct passwd' has no member named 'pw_gecos'
```

以下的小 patch 简单的避免这个问题(其实就是简单粗暴地给 fullname 赋值,
避免访问 `pwd->pw_gecos`):

``` example
--- msginit.c   2012-12-04 14:28:58.000000000 +0800
+++ msginit.c.new   2013-04-01 11:59:54.054980294 +0800
@@ -1081,7 +1081,11 @@
       char *result;

       /* Return the pw_gecos field, up to the first comma (if any).  */
+#ifndef __ANDROID__
       fullname = pwd->pw_gecos;
+#else
+fullname = "android";
+#endif
       fullname_end = strchr (fullname, ',');
       if (fullname_end == NULL)
         fullname_end = fullname + strlen (fullname);
```

以下是编译的具体指令:

``` bash
$ ./configure --prefix="${SYSROOT}/usr" --host=arm-linux-androideabi CFLAGS="--sysroot $SYSROOT" --enable-static --disable-java --disable-native-java
$ make -j5
$ make install
```

libffi
======

编译没什么问题, 一次通过

``` bash
$ ./configure --prefix="${SYSROOT}/usr" --host=arm-linux-androideabi CFLAGS="--sysroot $SYSROOT" --enable-static
$ make -j5
$ make install 
```

compile glib
============

glib 的编译稍微麻烦一点.

这里我选择相对新一点的 2.34.3 版本, 这个版本的探测脚本已经支持 android
host 了.

另外, 在编译前的配置时, ARM 上的编译器会在检查一些特性时失败,
采用下面的方法 避免这类失败: 详情请参考源码目录下的
docs/reference/glib/html/glib-cross-compiling.html

新建一个文件 android.cache, 写入以下内容.

``` bash
# file android.cache
ac_cv_type_long_long=yes
glib_cv_stack_grows=no
glib_cv_uscore=no
ac_cv_func_posix_getpwuid_r=no
ac_cv_func_posix_getgrgid_r=no
```

配置编译的指令(test 模块编译不过去, 没关系, 我永不到, disable 之):

``` bash
$ ./configure --prefix="${SYSROOT}/usr" --host=arm-linux-androideabi CFLAGS="--sysroot $SYSROOT" --enable-static --cache-file=android.cache --disable-modular-tests
```

编译的时候会出现各种各样的错误:

``` example
gstrfuncs.c: In function 'g_ascii_strtod':
gstrfuncs.c:718:30: error: 'struct lconv' has no member named 'decimal_point'
gstrfuncs.c: In function 'g_ascii_formatd':
gstrfuncs.c:942:30: error: 'struct lconv' has no member named 'decimal_point'

gutils.c:840:8: error: 'struct passwd' has no member named 'pw_gecos'
gutils.c:840:25: error: 'struct passwd' has no member named 'pw_gecos'
gutils.c:846:35: error: 'struct passwd' has no member named 'pw_gecos'
gutils.c:749:12: warning: unused variable 'logname' [-Wunused-variable]
gutils.c:748:10: warning: unused variable 'error' [-Wunused-variable]

glocalfileinfo.c:1097:21: error: 'struct passwd' has no member named 'pw_gecos'

gresolver.c:1133:14: error: 'T_TXT' undeclared (first use in this function)
gresolver.c:1133:14: note: each undeclared identifier is reported only once for each function it appears in
gresolver.c:1135:14: error: 'T_SOA' undeclared (first use in this function)
gresolver.c:1137:14: error: 'T_NS' undeclared (first use in this function)
gresolver.c:1139:14: error: 'T_MX' undeclared (first use in this function)
```

同样, 简单粗暴的 hack.(由于 patch 比较打, 我放在后面的 gist 上).

不同版本可能会遇到各种各样不同的编译错误, 这里需要具体问题具体分析, 比如
某些结构体, 宏没有申明的, 可以手动添加, 一些测试模块编译不过去,
可以把测试 模块去掉, 例如上面的 configure 时的–disable-modular-tests
参数.

Others
======

glib 的 patch 我贴在 gist 上 请访问: [我的
gist](https://gist.github.com/mathslinux/5283294)
