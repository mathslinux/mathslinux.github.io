---
layout: post
tag: Others
date: '\[2020-06-30 Tue\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 使用 Jekyll 和 Github 搭建个人博客
---

晕晕乎乎了两年，准备把停更的 blog
更新一下。但是时过境迁，很多技术方法都已 经成为古董了，之前的blog是在
vps 上用 wordpress 搭建的，我在电脑上写好 emacs-org
格式的文章后，用脚本转换为 html 然后用 wordpress 的 API 发表的。
现在不想太折腾了，简单研究了一番，发现用 Jekyll 和 github
的组合最方便省心。

大体思路为，文章还是在本地用 emacs-org
的格式来写(毕竟我用了10来年已经比较熟悉了)， 然后用 pandoc 转换为基本的
markdown，最后写一个小脚本简单处理一下图片， jekyll tag 等问题。

安装 Jekyll
===========

Jekyll 基于 ruby，所以首先要安装 ruby，这里选择使用 rbenv 来安装管理
ruby。

以下操作在我的 mac 上进行:

``` bash
$ brew install rbenv ruby-build
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile  
$ rbenv install -l              # 列出所有可安装的版本
$ rbenv install 2.7.1           # 安装最新稳定版本
$ rbenv global 2.7.1            # 设置全局 ruby 版本
# 将 gem 安装的文件列入可执行文件路径中
$ echo 'export PATH="$HOME/.gem/ruby/2.7.0/bin:$PATH"' >> ~/.bash_profile 
```

安装 Jekyll:

``` bash
$ gem install --user-install bundler jekyll
$ jekyll -v   # 检查是否安装成功
jekyll 4.1.1
```

在 Github 上选择模板
====================

这里选择 [leopardpan 的 blog
模板](https://github.com/leopardpan/leopardpan.github.io): 在他的 blog
github 上 fork 到自己仓库，注意 仓库名要改为 username.github.io
这种名字。=username= 是你的 github 用户名。

克隆仓库

``` bash
$ git clone git@github.com:mathslinux/mathslinux.github.io.git # 克隆仓库
```

在本地启动 blog 查看效果, 启动后在 <http://127.0.0.1:4000/>
就可以查看调试了。

``` bash
$ jekyll serve
```

也可以在 <http://username.github.io/> 查看。其中 username 是你的 github
用户名。 之后发表新的 blog 之后 直接 `git push` 后就是更新了。

修修改改
========

jekyll 模板中的主要需要修改的文件:

-   `_config.yml`: blog 的全局配置, 包括 blog
    名字等等，按照自己的实际情况修改。
-   `_includes/footer.html`: 修改里面的版权等信息。
-   about.md: 关于自己的介绍。
-   images:
    该目录包含自己的头像，背景，支付宝，微信等的二维码图像等，按需修改。

发表文章
========

发表文章很简单，只需要以 markdown 的格式写好文章，放到 `_posts`
目录下即可。 需要注意的是文件名要以 Date-Title.md 这种格式，如
=2017-10-06-Free~UpSpace~.md=。

文件内容和 markdown 格式没有区别。但是在文件头，需要添加一些元数据。

``` example
---
layout: post
tag: Emacs
date: [2017-01-22 日 16:30]
title: 在 Emacs 内部运行 Shell
---
```

-   layout: post 这行不用变。
-   tag: 文章的标签。
-   date: 日期。
-   title: 文章的标题。
