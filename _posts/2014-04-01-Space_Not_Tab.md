---
layout: post
tag: Emacs
date: '\[2014-04-01 二 16:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Emacs 中默认使用空格来缩进
---

我讨厌 TAB, 在 QEMU 的开发中, python 代码的编写,
在和别人共同完成的项目中, 等等, 用 TAB 缩进就是一场灾难.

每次开始接触一种新语言的时候, 都要设置类似一堆

``` elisp
(add-hook 'ruby-mode-hook (lambda ()
                            (setq indent-tabs-mode nil)))
```

今天无意中在一个 elisp 代码文件里面看到有这么一个函数:

``` elisp
; Set the default value of variable VAR to VALUE.
(setq-default indent-tabs-mode nil)
```

这下, 妈妈再也不用担心我的缩进问题了.
