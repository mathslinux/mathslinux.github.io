---
layout: post
tag: Others
date: '\[2013-08-12 1 09:08\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 从源码部署 seafile
---

[seafile](http://seafile.com/home/) 是一款很优秀的 开源的 dropbox
replace.

部署环境
========

[官方的
wiki](https://github.com/haiwen/seafile/wiki/Build-and-deploy-seafile-server-from-source)
推荐使用 ubuntu 12.04 作为部署的环境, 但是根据我研究的结果, seafile
用到的某些组件和 ubuntu 12.04 的软件包冲突, 比如 django 的 log
模块有问题, 这其实 django 1.3 的一个 bug,
另外的像一些依赖包也有各种各样的问题, 所以方便 起见,
最好的方式就是在稍微新一点的版本上部署, 这里我选择 ubuntu 13.04 的 64 位
server 版.

安装依赖软件包
==============

seafile 官方发行的二进制包很有意思, 他把大部分需要依赖的软件包都包含在
发布的二进制中, 在启动的时候再定制程序加载的路径. 这些依赖中,
部分并没有被 各大发行版收录, 只能通过源码手动安装.

先把 ubuntu 软件源里自带的包装上:

``` bash
$ sudo apt-get install build-essential libevent-dev libcurl4-openssl-dev \
libglib2.0-dev uuid-dev intltool libsqlite3-dev libmysqlclient-dev libarchive-dev \
flex python-django sqlite3 python-simplejson python-imaging gunicorn python-pip \
python-flup python-chardet python-django-registration python-djblets cmake unzip
```

还有以下几个包是需要我们手动从 source 安装的

-   libzdb
-   libevhtp
-   djangorestframework

安装 libzdb:

``` bash
$ wget https://seafile.googlecode.com/files/libzdb-2.11.1-mod.tar.gz
$ tar xf libzdb-2.11.1-mod.tar.gz; cd libzdb-2.11.1
$ ./configure --prefix=/usr && sudo make && sudo make install
```

安装 libevhtp

``` bash
$ wget https://github.com/ellzey/libevhtp/archive/1.1.6.zip -O libevhtp-1.1.6.zip
$ unzip libevhtp-1.1.6.zip; cd libevhtp-1.1.6
$ cmake . && sudo make && sudo make install
```

安装 djangorestframework

``` bash
$ djangorestframework
```

安装 seafile
============

seafile 由以下几大组件构成:

-   Ccnet 网络服务模块
-   Seafile server 数据服务模块
-   Seahub web 后端, 就是一个 django 实例
-   HttpServer: 为 seahub 处理文件上传下载
-   Controller:

在 github 上的项目分布很乱, seahub 是一个单独的项目, seafile 里面包含了
httpserver, controller, monitor, seafile server 还有 windows, linux, mac
的客户端, ios 和 android 客户端是单独的仓库, ccnet 也是一个单独的仓库.

但是在官方发布源码版本的时候, 在 server 这端, 是把 seahub
单独发布一个版本, seafile 单独发布一个版本(包括 ccnet 等)

编译 seafile 的过程稍微复杂一点:

为了升级和管理数据的方式, 官方建议用以下的目录布局.(foo 可以随便取名,
这里是我先用 cloudtimes)

``` example
foo/
└── ccnet # ccnet config directory
    └── ccnet.conf # ccnet config file
└── seafile-data # seafile configuration and data
    └── seafile.conf # seafile config file
└── seahub-data/ # seahub data
└── seahub.db # seahub sqlite3 database
└── seahub_settings.py # custom settings for seahub
└── seafile-server
    └── seahub/
    └── seafile-{VERSION} # seafile source code
```

首先是创建需要的目录布局, 并下载相应的源码

``` bash
# mkdir -p /data/cloudtimes/seafile-server
# cd /data/cloudtimes/seafile-server/
# wget http://seafile.googlecode.com/files/seafile-latest.tar.gz
# wget http://seafile.googlecode.com/files/seahub-latest.tar.gz
# tar xf seafile-latest.tar.gz
# mv seahub-1.7.0 seahub
```

这个版本(1.7.1 的启动脚本有点问题, 需要 hack 之): 文件在
/data/cloudtimes/seafile-server/seafile-1.7.1.1/tools

``` bash
diff --git a/seafile-admin b/seafile-admin
index 86741a0..250705d 100755
--- a/seafile-admin
+++ b/seafile-admin
@@ -561,10 +561,10 @@ def init_seahub():
     create_gunicorn_conf()

 def check_django_version():
-    '''Requires django 1.3'''
+    '''Requires django 1.4'''
     import django
-    if django.VERSION[1] != 3:
-        error('Django 1.3 is required')
+    if django.VERSION[1] != 4:
+        error('Django 1.4 is required')
     else:
         del django

@@ -584,7 +584,7 @@ def check_python_dependencies(silent=False):
     check_python_module('simplejson', 'simplejson', silent=silent)
     check_python_module('sqlite3', 'sqlite3', silent=silent)
     check_python_module('PIL', 'python imaging library(PIL)', silent=silent)
-    check_python_module('django', 'django 1.3', silent=silent)
+    check_python_module('django', 'django 1.4', silent=silent)
     check_python_module('djblets', 'djblets', silent=silent)
     check_python_module('rest_framework', 'django rest framework', silent=silent)
```

然后编译各个模块:

``` bash
# cd /data/cloudtimes/seafile-server/seafile-1.7.1.1/libsearpc
# ./configure --prefix=/usr && make && make install

# cd /data/cloudtimes/seafile-server/seafile-1.7.1.1/ccnet
# ./configure --prefix=/usr --disable-client --enable-server
# make && make install

# cd /data/cloudtimes/seafile-server/seafile-1.7.1.1
# ./configure --prefix=/usr --disable-client --enable-server
# make && make install
```

初始化配置
==========

以上一切都成功后, 使用 seafile-admin 这个工具初始化 seafile 的配置

``` bash
# cd /data/cloudtimes // 进入这个目录很重要
# seafile-admin setup // 按照提示进入, 配置端口等等
# seafile-admin start // 就这样可以启动 seafile
```

把 seafile 部署在 apache 上
===========================

``` bash
# apt-get install libapache2-mod-fastcgi
# a2enmod rewrite
```

编辑 apache 的配置文件, 端口路径根据需要自行设定

``` example
FastCGIExternalServer /var/www/seahub.fcgi -host 127.0.0.1:8000
Listen 80
<VirtualHost *:80>
  ServerName 127.0.0.1
  DocumentRoot /var/www
  Alias /media  /data/cloudtimes/seafile-server/seahub/media
  RewriteEngine On
  RewriteRule ^/(media.*)$ /$1 [QSA,L,PT]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteRule ^(.*)$ /seahub.fcgi$1 [QSA,L,E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
</virtualhost>
```

运行 fastcgi 模式

``` bash
# seafile-admin start --fastcgi
```

在 MySQL 上部署 seafile
=======================

``` bash
# apt-get install -y mysql-server mysql-client python-mysqldb
```

未完待续

Reference
=========

[Build and deploy seafile server from
source](https://github.com/haiwen/seafile/wiki/Build-and-deploy-seafile-server-from-source)

[Deploy Seafile Web with nginx
apache](https://github.com/haiwen/seafile/wiki/Deploy-Seafile-Web-with-nginx-apache)

[Deploy Seafile with
MySQL](https://github.com/haiwen/seafile/wiki/Deploy-Seafile-with-MySQL)
