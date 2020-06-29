---
layout: post
tag: QT
date: '\[2015-05-25 一 22:25\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QT 中控件关闭的几个 issue
---

最近在写一个小应用，需要一个持久化的窗口，即用户在关闭窗口的时候需要捕获这个事件，让窗口暂时隐藏.
总结了 QT 中的控件关闭时的一些小 issue：

lastWindowClosed() and setQuitOnLastWindowClosed()
==================================================

如果一个控件是一个可见的窗口类型，并且没有父容器，那么关闭该窗口导致的
结果是整个应用程序会退出. 连控件的析构函数都来不及调用。不过，QT 会在这
种情况发生的时候发送一个 QGuiApplication::lastWindowClosed() 的信号,
连接这个信号就可以获取这个事件，用来做一些清理的操作，例如：

    connect(QApplication::instance(), SIGNAL(lastWindowClosed()), this, SLOT((cleanup())));

如果想忽略这种行为，在 QGuiApplication 这个类里，提供了一个静态方法，
setQuitOnLastWindowClosed(bool) 来控制这种行为, 比如:

    QApplication::setQuitOnLastWindowClosed(false);

这样， 当最后一个可视窗口关闭，整个程序就不会退出。

closeEvent()
============

QWidget 基类提供了一个 protect closeEvent() 方法，在用户点击关闭窗口
的时候，会调用改方法，使得 QWidget
子类的实现可以控制改关闭事件，比如下面的代码， 调用
QCloseEvent::ignore() 来忽略该事件，只是隐藏窗口显示.

    void Window::closeEvent(QCloseEvent *event)
    {
        event->ignore();
        this->hide();
    }
