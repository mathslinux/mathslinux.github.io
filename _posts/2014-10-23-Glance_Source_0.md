---
layout: post
tag: OpenStack
date: '\[2014-10-23 四 11:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Glance 源码分析(0) – 介绍
---

这周我把 glance 的源代码通读了一遍(业务流程主要是 V2 版本),
感觉这个模块虽然是 OpenStack 里面相对较小的模块(vs Nova),
但是他的设计模式, 架构一点都不含糊, 我看完 glance 之后再去看其它项目,
发现 API 的等框架大部分地方都是一模一样. 也就是 说看完这个模块,
再去分析其它模块就省事很多了, 以下将我的分析拆分做一个记录:

-   [框架](http://mathslinux.org/?p%3D606)
-   [配置文件](http://mathslinux.org/?p%3D620)
-   [WSGI 框架](http://mathslinux.org/?p%3D626)
-   paste 具体流程分析
-   API 工作流程
