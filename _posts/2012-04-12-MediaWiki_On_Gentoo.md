---
layout: post
tag: Gentoo
date: '\[2012-04-12 三 15:12\]'
title: Gentoo 安装 Mediawiki
---

Install
=======

Install Apache
==============

``` bash
echo "www-server/apache ssl" >> /etc/portage/package.use
emerge apache
emerge mod_security 
```

配置
----

修改 /etc/apache2/httpd.conf

添加以下内容

``` example
#For MediaWiki
Listen 8888
<Directory /var/www/localhost>
    AddHandler php5-cgi .php
    Options +Indexes +ExecCGI +FollowSymLinks
    DirectoryIndex index.php
    AllowOverride Limit
</Directory>
```

修改 /etc/conf.d/apache2

``` example
APACHE2_OPTS="-D SSL -D PHP5" # 追加
```

修改 /etc/apache2/vhosts.d/mediawiki-vhost.conf 添加以下内容

``` example
ameVirtualHost *:8888
<VirtualHost *:8888>
    ServerName "myserver.mydomain.com"
    DocumentRoot "/var/www/localhost/htdocs"
    <Directory "/var/www/localhost/htdocs">
        AddHandler php5-cgi .php
        DirectoryIndex index.php
        AllowOverride All
        Order Allow,Deny
        Allow from All
    </Directory>
</VirtualHost>
```

Install PHP
===========

``` bash
echo "dev-lang/php suhosin apache2 mysql ssl gd jpeg truetype cgi cjk curlwrappers fpm ftp imap inifile ldap ldap-sasl mysqli mysqlnd odbc pcntl sharedmem soap sockets spell sysvipc tidy xmlreader xmlrpc xmlwriter xsl zip" >> /etc/portage/package.use
```

配置
----

修改 etc/apache2/modules.d/70~modphp5~.conf (默认的不能用)

``` example
<IfDefine PHP5>
    # Load the module first
    <IfModule !mod_php5.c>
        LoadModule php5_module    modules/libphp5.so
    </IfModule>

    # Set it to handle the files
    <FilesMatch "\.ph(p5?|tml)$">
        SetHandler application/x-httpd-php
    </FilesMatch>

    <FilesMatch "\.phps$">
        SetHandler application/x-httpd-php-source
    </FilesMatch>

    DirectoryIndex index.php index.phtml
</IfDefine>
```

Install MySQL
=============

``` bash
emerge -av mysq
mysql_install_db
/etc/init.d/mysql restart
mysqladmin -u root password 'new-password'
mysqladmin -u root -h cloud-times password 'new-password' # cloud-times 是本机名
mysql -u root -p mysql # 输入密码

mysql> create database wikidb;
mysql> grant create, select, insert, update, delete, alter, lock tables on wikidb.* to 'wikiuser'@'localhost' identified by 'password';  #'password'改为您的密码
mysql> flush privileges;
mysql> set password for 'wikiuser'@'localhost'=password('password'); #'password'改为您的密码
mysql> \q
```

Install MediaWiki
=================

``` bash
ls /usr/share/webapps/mediawiki
webapp-config -I -d mediawiki mediawiki 1.18.1 # 1.18.1 上一条指令得到
cd /var/www/localhost/htdocs/mediawiki
chmod a+w mw-config
http://192.168.0.208:8888/mediawiki
# 下载 LocalSettings.php, 放在 /var/www/localhost/htdocs/mediawiki(会有提示)
```
