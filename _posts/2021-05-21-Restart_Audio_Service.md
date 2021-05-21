---
layout: post
tag: Mac
date: '\[2021-05-21 Fri 09:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 重启声卡服务
---

随着时间的变迁，我的老古董 Mac Air
又遇到新的问题，联系播放几小时电影之后，
或者频繁地进行快进等操作的时候，画面突然会暂停，以为是显卡什么的坏了，进行
一番排查之后，才发现罪魁祸首原来是名为 coreaudiod 的服务出现了 bug.

![](/images/posts/Mac/coreaudiod_bug.png)

找到问题之后解决就很简单了. 重启该服务即可。PS: 在 Mac
上重启很多系统服务 只需要杀死该服务即可，如 Finder 等。

``` bash
$ sudo killall -9 coreaudiod
```
