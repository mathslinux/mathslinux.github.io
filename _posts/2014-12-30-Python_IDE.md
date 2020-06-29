---
layout: post
tag: Emacs
date: '\[2014-12-30 二 15:50\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Build Emacs As A Python IDE
---

众所周知在很久很久以前, 初学者要把 Emacs 搭建成比较容易上手的环境, 需要
一点精力和耐心, 但这恰恰是 Emacs 的魅力所在(超高度的可定制化),
但对初学者来说, 由于 [el-get](https://github.com/dimitri/el-get) 等
**包管理器** 的出现, 情况变得越来越好了. 比如要马上开始用 Emacs
来作为开发 Python 的工具, 现在已经很简单了.

安装 el-get
===========

将下列配置写入到 .emacs 中

``` elisp
(add-to-list 'load-path "~/.emacs.d/el-get/el-get")
(unless (require 'el-get nil 'noerror)
  (with-current-buffer
      (url-retrieve-synchronously
       "https://raw.github.com/dimitri/el-get/master/el-get-install.el")
    (let (el-get-master-branch)
      (goto-char (point-max))
      (eval-print-last-sexp))))

(el-get 'sync)
```

just it! 启动 emacs 的时候会自动初始化 el-get 需要的配置信息

jedi
====

[jedi](http://tkf.github.io/emacs-jedi/latest/)
主要是一个自动补全的插件, Emacs 已经有一个名为
[auto-complete](http://cx4a.org/software/auto-complete/) 的强大的
补全插件了, 通吃所有语言. 没错, jedi 的自动补全就是利用该插件作为后端了,
不仅如此, 他还可以在编写代码的时候实时查看函数的成员信息,
函数的参数信息和文档信息等, 总之很强大.

使用 el-get-install 回车, 输出 jedi 安装, 借助 el-get 的强大,
所有的依赖都会 自动安装(.e.g. auto-complete, epc)

``` bash
M-x el-get-install
```

在 .emacs 里加入 jedi 的配置

``` bash
add-hook 'python-mode-hook 'jedi:setup)
(setq jedi:complete-on-dot t)
```

ropemacs
========

[ropemacs](http://rope.sourceforge.net/ropemacs.html) 是借助
[rope](http://rope.sourceforge.net), pymacs 来做 python
的工程管理的东西, 在它面前, 神马 代码重构, 代码跳转, 自动模块导入,
类成员补全神马都是浮云. (注意, 所有代码 补全我都使用 jedi 来做,
不用到这里的功能)

安装也很方便, 直接 el-get-install 然后回车输入 ropemacs 就可以了

安装完成在 .emacs 里加入

``` bash
(pymacs-load "ropemacs" "rope-")
(setq ropemacs-enable-autoimport t)

(autoload 'pymacs-apply "pymacs")
(autoload 'pymacs-call "pymacs")
(autoload 'pymacs-eval "pymacs" nil t)
(autoload 'pymacs-exec "pymacs" nil t)
(autoload 'pymacs-load "pymacs" nil t)
```

flycheck
========

该插件是一个实时代码检查的工具, 也就是在编代码的过程中,
会根据改语言的编码 规范, 实时检查和提示源代码的错误, 然后给出警告,
比如语法错误, 编码规范 等. 该插件也是通吃所有语言, 这里我们只关注 python
相关的

直接 el-get-install 然后回车输入 flycheck 安装.

在 .emacs 里面加入

``` elisp
(add-hook 'after-init-hook #'global-flycheck-mode)
```

注意, flycheck 需要一个检查语法的后端程序, 如果是 python 的话, 推荐
pylink,

``` bash
# pip install pylink
```

后记
====

不考虑其他的配置, 什么强大的 ido, ibuffer, 按键绑定配置, 窗口配置等等,
一个 标准的 python IDE 就配置好了, 就这么简单.

附上所有的 .emacs 配置

``` elisp
(add-to-list 'load-path "~/.emacs.d/el-get/el-get")
(unless (require 'el-get nil 'noerror)
  (with-current-buffer
      (url-retrieve-synchronously
       "https://raw.github.com/dimitri/el-get/master/el-get-install.el")
    (let (el-get-master-branch)
      (goto-char (point-max))
      (eval-print-last-sexp))))

(el-get 'sync)
(add-hook 'python-mode-hook 'jedi:setup)
(setq jedi:complete-on-dot t)

(add-hook 'after-init-hook #'global-flycheck-mode)

(pymacs-load "ropemacs" "rope-")
(setq ropemacs-enable-autoimport t)

(autoload 'pymacs-apply "pymacs")
(autoload 'pymacs-call "pymacs")
(autoload 'pymacs-eval "pymacs" nil t)
(autoload 'pymacs-exec "pymacs" nil t)
(autoload 'pymacs-load "pymacs" nil t)
```
