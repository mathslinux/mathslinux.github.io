---
layout: post
tag: OpenStack
date: '\[2013-12-21 六 23:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 从错误状态恢复虚拟机
---

我的 openstack 整个环境被我不小心重启了,
重启计算节点出了问题没有启动起来, 我于是想 stop 这个实例, 但是 stop 后,
虚拟机异常了, 如下:

``` bash
# nova list
+--------------------------------------+--------------+--------+--------------+-------------+-----------------------+
| ID                                   | Name         | Status | Task State   | Power State | Networks              |
+--------------------------------------+--------------+--------+--------------+-------------+-----------------------+
| 015ac60a-1902-4d39-b4ea-11376838872b | myvm         | ACTIVE | None         | Running     | private=192.168.22.34 |
| 72824bed-4ce6-440c-996f-12d725e3fa71 | myvm-on-ceph | ACTIVE | None         | Running     | private=192.168.22.36 |
| a8bdeb86-981f-449e-a529-e6f43314f7dc | myvm2        | ACTIVE | powering-off | Running     | private=192.168.22.35 |
+--------------------------------------+--------------+--------+--------------+-------------+-----------------------+

# nova stop myvm2
ERROR: Instance a8bdeb86-981f-449e-a529-e6f43314f7dc in task_state powering-off. 
Cannot stop while the instance is in this state. (HTTP 400) (Request-ID: req-f91e6cfd-bc05-43c8-87e5-5f00bc25f713)
```

我把计算节点修复之后, 该虚拟机也一直无法恢复, 这时候只有通过 nova
reset-state 恢复状态了, 该指令会把 instance 的 Task State 置为 None,
然后就可以 stop 或者 delete 该虚拟机了.

``` bash
# nova reset-state myvm2
# nova list
+--------------------------------------+--------------+--------+------------+-------------+-----------------------+
| ID                                   | Name         | Status | Task State | Power State | Networks              |
+--------------------------------------+--------------+--------+------------+-------------+-----------------------+
| 015ac60a-1902-4d39-b4ea-11376838872b | myvm         | ACTIVE | None       | Running     | private=192.168.22.34 |
| 72824bed-4ce6-440c-996f-12d725e3fa71 | myvm-on-ceph | ACTIVE | None       | Running     | private=192.168.22.36 |
| a8bdeb86-981f-449e-a529-e6f43314f7dc | myvm2        | ERROR  | None       | Running     | private=192.168.22.35 |   <== 状态已经更新
+--------------------------------------+--------------+--------+------------+-------------+-----------------------+
```

状态已经修复, 可以 stop && start 了

``` bash
# nova stop myvm2
# nova list
+--------------------------------------+--------------+---------+------------+-------------+-----------------------+
| ID                                   | Name         | Status  | Task State | Power State | Networks              |
+--------------------------------------+--------------+---------+------------+-------------+-----------------------+
| 015ac60a-1902-4d39-b4ea-11376838872b | myvm         | ACTIVE  | None       | Running     | private=192.168.22.34 |
| 72824bed-4ce6-440c-996f-12d725e3fa71 | myvm-on-ceph | ACTIVE  | None       | Running     | private=192.168.22.36 |
| a8bdeb86-981f-449e-a529-e6f43314f7dc | myvm2        | SHUTOFF | None       | Shutdown    | private=192.168.22.35 |
+--------------------------------------+--------------+---------+------------+-------------+-----------------------+


# nova list
+--------------------------------------+--------------+--------+------------+-------------+-----------------------+
| ID                                   | Name         | Status | Task State | Power State | Networks              |
+--------------------------------------+--------------+--------+------------+-------------+-----------------------+
| 015ac60a-1902-4d39-b4ea-11376838872b | myvm         | ACTIVE | None       | Running     | private=192.168.22.34 |
| 72824bed-4ce6-440c-996f-12d725e3fa71 | myvm-on-ceph | ACTIVE | None       | Running     | private=192.168.22.36 |
| a8bdeb86-981f-449e-a529-e6f43314f7dc | myvm2        | ACTIVE | None       | Running     | private=192.168.22.35 |
+--------------------------------------+--------------+--------+------------+-------------+-----------------------+
```
