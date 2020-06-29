---
layout: post
tag: Language_Study
date: '\[2012-05-10 4 11:35\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 建立 Lisp 开发环境
---

[Slime](http://common-lisp.net/project/slime) 是 Emacs 下的一个写
[Common Lisp](http://common-lisp.net) 的一个插件

Download
========

如果是从源代码安装的话, 需要从 Slime 的官网上下载安装包, 然后进行安装.

Install
=======

在 Gentoo 上安装很简单:

``` bash
# emerge -av slime
```

Configuration
=============

``` elisp
(setq inferior-lisp-program "/usr/bin/sbcl")
(add-to-list 'load-path "/usr/share/emacs/site-lisp/slime/")
(require 'slime)
(slime-setup)
```
