---
layout: post
tag: QEMU_KVM
date: '\[2012-04-24 2 11:04\]'
title: NetBSD 链表使用
---

说明
====

很好的一套链表库, 以 list 的举例说明.

-   `le_prev` 二级指针, 上一个元素的 next 成员的地址. 此处需要注意的是
    le~prev~ 是上一个元素的 next 成员的地址, \*le~prev~ 是
    上一个成员的值, 所以可以通过操作 le~prev~ 来操作上一个节点 的 next
    成员. TODO

```{=html}
<!-- -->
```
``` c
/*
 * List definitions.
 */
#define LIST_HEAD(name, type)                       \
struct name {                               \
    struct type *lh_first;  /* first element */         \
}

#define LIST_HEAD_INITIALIZER(head)                 \
    { NULL }

#define LIST_ENTRY(type)                        \
struct {                                \
    struct type *le_next;   /* next element */          \
    struct type **le_prev;  /* address of previous next element */  \
}

/*
 * List functions.
 */
#define LIST_INIT(head) do {                        \
    (head)->lh_first = NULL;                    \
} while (/*CONSTCOND*/0)

#define LIST_INSERT_AFTER(listelm, elm, field) do {         \
    if (((elm)->field.le_next = (listelm)->field.le_next) != NULL)  \
        (listelm)->field.le_next->field.le_prev =       \
            &(elm)->field.le_next;              \
    (listelm)->field.le_next = (elm);               \
    (elm)->field.le_prev = &(listelm)->field.le_next;       \
} while (/*CONSTCOND*/0)

#define LIST_INSERT_BEFORE(listelm, elm, field) do {            \
    (elm)->field.le_prev = (listelm)->field.le_prev;        \
    (elm)->field.le_next = (listelm);               \
    *(listelm)->field.le_prev = (elm);              \
    (listelm)->field.le_prev = &(elm)->field.le_next;       \
} while (/*CONSTCOND*/0)

#define LIST_INSERT_HEAD(head, elm, field) do {             \
    if (((elm)->field.le_next = (head)->lh_first) != NULL)      \
        (head)->lh_first->field.le_prev = &(elm)->field.le_next;\
    (head)->lh_first = (elm);                   \
    (elm)->field.le_prev = &(head)->lh_first;           \
} while (/*CONSTCOND*/0)

#define LIST_REMOVE(elm, field) do {                    \
    if ((elm)->field.le_next != NULL)               \
        (elm)->field.le_next->field.le_prev =           \
            (elm)->field.le_prev;               \
    *(elm)->field.le_prev = (elm)->field.le_next;           \
} while (/*CONSTCOND*/0)

#define LIST_FOREACH(var, head, field)                  \
    for ((var) = ((head)->lh_first);                \
        (var);                          \
        (var) = ((var)->field.le_next))

#define LIST_FOREACH_SAFE(var, head, field, tvar)           \
    for ((var) = LIST_FIRST((head));                \
        (var) && ((tvar) = LIST_NEXT((var), field), 1);     \
        (var) = (tvar))

/*
 * List access methods.
 */
#define LIST_EMPTY(head)        ((head)->lh_first == NULL)
#define LIST_FIRST(head)        ((head)->lh_first)
#define LIST_NEXT(elm, field)       ((elm)->field.le_next)
```

使用示例
========

``` c
#include <stdio.h>
#include <stdlib.h>
#include "queue.h"

/**
 * 双向链表
 * 
 */

struct TestListEntry {
    int index;
    LIST_ENTRY(TestListEntry) entries;
};


static LIST_HEAD(list_head, TestListEntry) list_head;

int main(int argc, char *argv[])
{
    int i = 0;
    struct TestListEntry *t;

    /* 插入元素 */
    for (i = 0; i < 5; ++i) {
        t = malloc(sizeof(*t));
        t->index = i;
        LIST_INSERT_HEAD(&list_head, t, entries);
    }

    /* 遍历链表 */
    LIST_FOREACH(t, &list_head, entries) {
        printf ("Element: %d\n", t->index);
    }

    /* 删除链表 */
    LIST_FOREACH(t, &list_head, entries) {
        printf ("Remode Element: %d\n", t->index);
        LIST_REMOVE(t, entries);
    }

    return 0;
}
```
