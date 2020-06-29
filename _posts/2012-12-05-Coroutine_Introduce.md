---
layout: post
tag: Linux_System_Program
date: '\[2012-12-05 三 19:12\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Coroutine(协程) 介绍
---

概念
====

[coroutine](http://en.wikipedia.org/wiki/Coroutine) 和函数一样, 区别在于
coroutine 有多个入口点, 而一般的函数 函数只能有一个入口点.
一般的函数只能从开始的地方执行, 一旦退出, 就只能从 唯一的入口点再开始了.
但是 coroutine 不同, 当它觉得没有任务需要处理时, 它可以把 CPU
让给其他函数, 然后它在这个让出的点等待, 直到其它函数再把 CPU 给它.

考虑以下的例子(producer-consumer 的模型):

producer 创建 buf, consumer 处理 buf. 这是很常见的编程模型, 在 C/S
或者其他编程框架中很常见.

一般我们用多线程的话, 大概会这样写:

``` example
q = new queue

producer_thread():
    loop:
        create some new items
        lock queue // 操作 queue 之前需要加锁保护数据
        add the items to q
        unlock queue

consumer_thread():
    loop:
        lock queue // 同样需要加锁
        remove some items from q
        unlock queue
        consumer items

main():
    create_producer_thread()
    create_consumer_thread()
```

而如果我们使用用 coroutine 的话, 情况就会好很多:

``` example
q = new queue

producer():
    loop:
        create some new items
        add the items to q
        yield to consumer

consumer():
    loop:
        remove some items from q
        consumer items
        yield to producer

main():
    initialize coroutine
```

coroutine 的优点
================

通过上面简单的示例(真实情况可能更复杂一些), 我们已经可以看到使用
coroutine 的优点了:

-   produce 和 consume 都是在同一个线程里执行的, 不会有 race condition
    的问题 发生.
-   coroutine 是在用户空间实现的, coroutine 的堆栈是由用户自己维护的,
    在切换 的时候, 用户交换 caller 和 coroutine 的堆栈,
    这比在内核空间开执行的 thread 的切换节省了大量的开销(想象一下, 切换
    thread 的时候, 需要做很多上下文的 切换, 各种寄存器 ……).
-   coroutine 的成本很低, 可以产生大量的 coroutine, 不像线程,
    每个线程有自己的 线程堆栈, ID, 上下文等.
-   可以由用户来控制 coroutine 的执行, 虽然 thread
    也可以进行某种程序的控制, 但是 基本上 thread 的调度都是由 OS
    来完成的.

用法实例
========

c
-

下面是在 QEMU 中使用的 coroutine.

``` c
/* file: test.c
 * coroutine_ucontext.c continuation.c could be fetched from spice or QEMU
 * source, use following command to compile:
 * gcc -o test test.c coroutine_ucontext.c continuation.c -DWITH_UCONTEXT
 */

#include <stdio.h>
#include "coroutine.h"

struct coroutine *co;

static void *coroutine_fun(void *data)
{
    while (1) {
        sleep(1);
        printf ("Yield to caller context\n");
        coroutine_yield(co);
        printf("\nI am in coroutine context\n");
    }
}

void coroutine_enter(void *data)
{
    while (1) {
        printf("Yield to coroutine context\n");
        coroutine_yieldto(co, data);
        printf("\nI am in caller context\n");
    }
}

static int coroutine_setup()
{
    co = malloc(sizeof(*co));

    co->stack_size = 16 << 20; /* 16Mb */
    co->entry = coroutine_fun;
    co->release = NULL;
    coroutine_init(co);

    return 0;
}

int main(int argc, char *argv[])
{
    coroutine_setup();
    coroutine_enter("coroutine");
    return 0;
}
```

python
------

python 中用来生成迭代器的就是一个 coroutine

``` c
#!/usr/bin/python

def coroutine():
    count = 0;
    while True:
        yield count
        count += 1;


c = coroutine()
for i in range(10):
    print "count:%d" % c.next()
```

Reference
=========

-   [Coroutine at wikipedia](http://en.wikipedia.org/wiki/Coroutine)

-   [淺談coroutine與gevent](http://blog.ez2learn.com/2010/07/17/talk-about-coroutine-and-gevent/)
