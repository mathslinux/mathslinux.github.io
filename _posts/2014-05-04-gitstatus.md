---
layout: post
tag: Tools
date: '\[2014-05-04 日 11:20\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 使用 gitstatus 生成项目的图标统计信息
---

最近老板要统计一下团队中各个开发人员的工作情况, 比如写的代码行数之类的.
由于我们 团队是用 [git](http://git-scm.com) 来做代码管理工具的,
所以很自然地我需要从 git 本身挖掘这些信息.

当然我可以使用 git log 加上各种文本处理工具(e.g. grep)来达到我的目的,
但是这种 方式太麻烦, 生成图表更不容易. 为了不重复发明轮子,
我放狗搜索了一下, 找到了 [gitstat](http://gitstats.sourceforge.net)
这个工具完美实现我的要求.

gitstat 特性
============

-   以各种维度统计代码信息, 包括代码提交次数, 代码行数, 提交持续时间,
    项目的活 跃度, 项目标签等等.
-   支持合并多个项目的统计信息
-   统计可以以图表的形式输出
-   统计输出 html 格式的文件, 可以部署到网站上
-   不支持 pdf 等其他格式的输出

怎样使用
========

gitstat 的使用非常简单:

``` bash
# gitstats [options] <repository1> <repository2> <repository3> <output dir>
```

比如, 我以 [spice](http://www.spice-space.org) 项目的统计信息为例, spice
项目有 `spice-server`, `spice-gtk`, `spice-procotol`, `vd_agent`
等子项目, 把结果保存在 `/var/www/` 目录中, 使用以下命令:

``` bash
# gitstat spice spice-gtk spice-procotol vd_agent /var/www
```

完成之后, 用浏览器打开 `/var/www/index.html` 就可以看到具体的统计信息了.

献上截图两章, 列位看官可以在里面看到我吗?

![](/images/posts/Tools/gitstatus1.png)

![](/images/posts/Tools/gitstatus2.png)
