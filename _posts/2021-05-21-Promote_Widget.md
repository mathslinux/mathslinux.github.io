---
layout: post
tag: QT
date: '\[2021-05-21 Fri 16:35\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QT Designer 使用未显示在工具箱中控件
---

在 QT 项目开发的时候，使用 QT Designer 进行 UI
设计的时候，有时候想用的控件 不在 Designer 工具箱里面，比如
**QWebEngineView** 之类的控件就没有，会感到很别扭。 在 [QT
官方文档](https://doc.qt.io/qt-5/designer-using-custom-widgets.html)
上查询了一下，可以通过 promote widget 的方式解决这个问题。

首先，拖动一个 widget 到放置控件的地方，右键这个 widget，选择提升为

![](/images/posts/QT/promote_widget_1.png)

最后在弹出的 promote 窗口上输入该控件的描述，包括控件类名和头文件，
然后点击提升就可以了。

![](/images/posts/QT/promote_widget_1.png)
