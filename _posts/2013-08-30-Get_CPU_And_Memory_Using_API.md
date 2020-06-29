---
layout: post
tag: Linux_System_Program
date: '\[2013-08-30 5 15:08\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 利用 API 获取 CPU 和内存信息
---

今天在研究 CPU 的热插拔时，看到了获取系统 CPU 和内存信息的 API,
才发现以前在实现 这些功能的时候去 /proc 获取数据是多么的 ugly.

这些 API 主要是利用了 `sysconf` 这个 POSIX 的接口,
这个接口可以获取系统运行 时信息, 包括 CPU 信息, 内存信息,
进程可以打开的最大文件句柄数等. 它的声明如下:

``` bash
long sysconf(int name);
```

-   `_SC_NPROCESSORS_CONF`: 获取系统中总的 CPU 数量,
    注意这里获取的是所有的 CPU 线程的数量
-   `_SC_NPROCESSORS_ONLN`: 获取系统中可用的 CPU 数量, 没有被激活的 CPU
    则不统计 在内, 例如热添加后还没有激活的.
-   `_SC_PHYS_PAGES`: 总的物理内存页大小.
-   `_SC_AVPHYS_PAGES`: 可用的物理内存页大小.

下面是我写的一些 demo 代码(在下面的代码里把 windows
获取这新信息的方法也写出来做参考):

``` bash
#include<stdio.h>

#if defined(_WIN32)
#define _WIN32_WINNT 0x0500
#include <windows.h>
void sysinfo_print()
{
    int cpu_num;
    SYSTEM_INFO si;
    MEMORYSTATUSEX memory;

    // 大部分 Windows 系统不支持热添加功能, 所以 online number 没有什么意义.
    GetSystemInfo(&si);
    cpu_num = si.dwNumberOfProcessors;
    printf("The number of processors: %d\n", cpu_num);

    memory.dwLength = sizeof(memory);
    GlobalMemoryStatusEx(&memory);
    printf("The memory size: %I64uK\n", memory.ullTotalPhys/1024);
    printf("The free memory size: %I64uK\n", memory.ullAvailPhys/1024);
}
#else
#include<unistd.h>  
#include<errno.h>
#include <string.h>
void sysinfo_print()
{
    int cpu_num, cpu_online_num;
    int mem_size, mem_free_size;

    cpu_num = sysconf(_SC_NPROCESSORS_CONF);
    if (cpu_num != -1) {
        printf("The number of processors: %d\n", cpu_num);
    } else {
        printf("Failed to get number of processors: %s\n", strerror(errno));
    }

    cpu_online_num = sysconf(_SC_NPROCESSORS_ONLN);
    if (cpu_online_num) {
        printf("The number of online processors: %d\n", cpu_num);
    } else {
        printf("Failed to get number of online processors: %s\n",
               strerror(errno));
    }

    // 注意: OSX 不支持下面两个宏.
    mem_size = sysconf(_SC_PHYS_PAGES);
    if (mem_size) {
        printf("The memory size: %dK\n", mem_size * 4);
    } else {
        printf("Failed to get memory size: %s\n", strerror(errno));
    }

    mem_free_size = sysconf(_SC_AVPHYS_PAGES);
    if (mem_free_size) {
        printf("The free memory size: %dK\n", mem_free_size * 4);
    } else {
        printf("Failed to get free memory size: %s\n", strerror(errno));
    }

}
#endif

int main(int argc, char *argv[])
{
    sysinfo_print();

    return 0;
}
```
