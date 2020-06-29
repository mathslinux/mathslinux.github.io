---
layout: post
tag: Emacs
date: '\[2017-01-22 日 16:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 在 Emacs 内部运行 Shell
---

在专注工作时候，上下文切换是一件非常低效的。无论是对人类还是对计算机来讲。对于
前者，不断的在不同的任务间切换，不但会造成时间的浪费，而且会造成大量的错误，
比如边打电话边发邮件。后者就更不用说了，为了解决上下文切换造成的开销，各种技术
被开发出来，携程，多核并行等等。

对我日常工作来讲，我用到最多的软件是 Emacs 和
[Shell](https://en.wikipedia.org/wiki/Shell_(computing))
。前者用来做几乎 90%的 日常工作，后者一般用来登录远程 ssh
服务器，运行程序等等。

即使这样，不断的在 Emacs 和 Shell
总切来切去总是让我觉得低效浪费，而且不仅仅 是按一下 Alt-Tab
那么简单，可能还需要把 Shell 中的输出拷贝到 Emacs，或者相反.

痛定思痛，我宁可在初期牺牲一些便利，也要找到在 Emacs 中使用 Shell
的正确姿势。

Emacs 内置的各种 Shell 方案
===========================

eshell
------

eshell 是完全使用 Emacs Lisp 编写的 shell，和 zsh, tcsh 一样，自己实现的
IO，Pipe，参数补全等。 eshell 完全用 elisp 编写，并且支持 elisp
语言，意味着他可以直接用 elisp 来扩展和使用，并且在 emacs 上下文环境中。

优点:

-   支持 elisp，用户可以在 eshell 上执行 elisp
    代码，意味着强大的自定义和扩展能力。

-   和 emacs 上下文集成，比如下面的这段代码直接创建一个 emacs buffer
    并且输入 "hello world":

        $ echo "hello world" >#<buffer results>

-   每个 eshell 本身是一个 buffer，emacs 的所有窗口管理方案都可以应用到
    eshell 中

-   由于 eshell 在 emacs
    上下文内，所有文本操作都不需要再切换上下文，比如要将终端屏幕区域某部份的
    内容写道一个文件上，需要: 选中并复制这些区域 -\> 打开一个文本编辑器
    -\> 右键粘帖或者 Ctrl-V, 实际 操作可能更复杂。在 eshell
    上完成这个操作，只需要选中区域直接调用 emacs 命令就完事。

-   在 Windows 等没有 bash 的平台上只要安装了 emacs，也能像 Linux
    一样使用常用的 Shell 命令来做系统管理

缺点:

-   不兼容 Bash. 很多应用程序的配置都是通过 bash 脚本导入的，如果这些
    bash 脚本包含复杂的 bash 的一些扩展和函数，导入就会失败。比如
    openstack 的 openrc 文件
-   由于 eshell 是
    [哑终端](https://en.wikipedia.org/wiki/Computer_terminal#Dumb_terminals),
    对 less 和 git log 这种需要翻页的输出支持不完善。

shell
-----

emacs 里面的 shell 利用 comint 模式为用户提供标准的 shell
接口，在这种模式下，输入回车之后，emacs 会把用户的指令发给后端的
bash(unix/linux) 或者 cmd(windows) 子进程，然后返回结果。在这种模式下，
所有的标准 unix shell 命令 或者 cmd 命令都可以无缝运行。

优点:

-   每个 shell 本身是一个 buffer，emacs 的所有窗口管理方案都可以应用到
    shell 中
-   由于每个 shell 也在 emacs
    上下文内，所有文本操作都不需要再切换上下文。
-   兼容 bash 或者 cmd

缺点:

-   由于 shell 也是
    [哑终端](https://en.wikipedia.org/wiki/Computer_terminal#Dumb_terminals),
    对 less 和 git log 这种需要翻页的输出支持不完善。
-   不支持跨平台

term
----

term 是一个终端模拟器，所有发送的指令直接发送给后端的 bash
子进程。用户所处的上下文就完全是一个
标准的终端了（bash或其他）。这种模式类似用 gnome-terminal putty 等使用
shell。

优点:

-   完全兼容 bash

缺点:

-   不支持跨平台
-   和 emacs 几乎没交互, 其实可以通过 term-line-mode(C-c C-j)
    切到编辑子模式， 做一些简单的复制粘帖操作，通过 term-mode-map(C-c
    C-k) 再切回来。

选择
====

综上，对我这类，对 lisp 有一些开发经验，喜欢定制的 hacker 来讲，eshell
无疑是第一选择。

配置 eshell
===========

提示符配置
----------

可以参考 [这里](https://www.emacswiki.org/emacs/EshellPrompt) 的各种方案
hack 自己的配置, 包括把当前路径，日期时间，主机名，git 仓库名状态等
在提示符显示， 以及颜色配置，等等。

补全配置
--------

我选用 AutoComplete 的方案，并参考
[这里](https://www.emacswiki.org/emacs/EshellCompletion#toc4) 的配置。

多 eshell 管理
--------------

编辑管理
--------

### 历史记录

### 别名

Known Issues
============
