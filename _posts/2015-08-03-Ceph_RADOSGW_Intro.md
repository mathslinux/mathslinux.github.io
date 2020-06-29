---
layout: post
tag: Storage
date: '\[2015-08-03 一 18:20\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: RadosGW 初体验
---

介绍
====

Ceph 的最底层模块是对象存储，本质上就是一个 Rados,
如果不考虑用户体验，利用 rados 就可以使用 Ceph 对象存储. 例如:

``` bash
$ rados -p test_pool get SI7W1FUI43H3I3ED9CWX test.data
```

-   `test_pool`: pool 名
-   SI7W1FUI43H3I3ED9CWX: object 名
-   test.data 保存到这里

和 RBD 以及 CephMDS 一样，RadosGW 提供了对 Radow
对象存储访问的更友好支持. 它允许用户通过 Restful API 的方式进行使用
RadowGW 。

从技术架构上讲, 和 RBD 一样, 它位于 Librados 之上,
而且正如它的名字所暗示的, 它是 Rados 的一个网关，通过 HTTP RestFul API,
对外提供对象存储的服务. 为了用户使用的方便性和兼容性，还提供了兼容 S3 和
Swfit 的接口.

以上，使得可以基于 RadosGW 构建云存储.

环境描述
========

假设已经有一个安装好的 Ceph 集群. 在一台新的节点上配置 RadowGW

节点信息如下:

主机名字
:   ceph.test.com

IP 地址
:   192.168.3.140

NOTE: 所有节点的操作系统都是 centos 7.

安装部署
========

安装配置 Apache
---------------

1.  安装

```{=html}
<!-- -->
```
``` bash
$ sudo yum install -y httpd
```

1.  配置: 编辑 **/etc/httpd/conf/httpd.conf**

```{=html}
<!-- -->
```
``` bash
# 设置 ServerName
ServerName ceph.test.com

# 添加下面内容
<IfModule !proxy_fcgi_module>
    LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
</IfModule>

# 修改监听参数: 192.168.3.140 为本机 IP
Listen 192.168.3.140:80
```

1.  重启 Apache

```{=html}
<!-- -->
```
``` bash
$ sudo systemctl restart httpd
```

安装配置 RadosGW
----------------

1.  安装 RadosGW

```{=html}
<!-- -->
```
``` bash
$ sudo yum install -y radosgw-agent
```

1.  创建用于 RadosGW 的用户等信息(在 ceph admin 节点执行)

```{=html}
<!-- -->
```
``` bash
$ sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring
$ sudo chmod +r /etc/ceph/ceph.client.radosgw.keyring
$ sudo ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.gateway --gen-key
$ sudo ceph-authtool -n client.radosgw.gateway --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
$ sudo ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.gateway -i /etc/ceph/ceph.client.radosgw.keyring
```

将这个 keyring 和 admin.keyring 复制到 ceph.test.com

``` bash
$ scp /etc/ceph/ceph.client.radosgw.keyring root@ceph.test.com:/etc/ceph
$ scp /etc/ceph/ceph.client.admin.keyring root@ceph.test.com:/etc/ceph
```

1.  创建 RadosGW 需要的 pool(在 ceph admin 节点)

```{=html}
<!-- -->
```
``` bash
$ ceph osd pool create .rgw 64 64
$ ceph osd pool create .rgw.root 64 64 
$ ceph osd pool create .rgw.control 64 64 
$ ceph osd pool create .rgw.gc 64 64
$ ceph osd pool create .rgw.buckets 64 64 
$ ceph osd pool create .rgw.buckets.index 64 64 
$ ceph osd pool create .log 64 64 
$ ceph osd pool create .intent-log 64 64 
$ ceph osd pool create .usage 64 64 
$ ceph osd pool create .users 64 64 
$ ceph osd pool create .users.email 64 64
$ ceph osd pool create .users.swift 64 64
$ ceph osd pool create .users.uid 64 64
```

1.  修改并 gateway.client 的配置到各个节点.

将下列内容添加到 ceph 节点的各个配置文件（包括 RadosGW 节点）

``` python
[client.radosgw.gateway]
host = ceph
keyring = /etc/ceph/ceph.client.radosgw.keyring
rgw socket path = /var/run/ceph/ceph.radosgw.gateway.fastcgi.sock
log file = /var/log/radosgw/client.radosgw.gateway.log
rgw print continue = false
```

1.  创建 RadosGW 运行目录

```{=html}
<!-- -->
```
``` bash
$ sudo mkdir -p /var/lib/ceph/radosgw/ceph-radosgw.gateway
$ sudo chown apache:apache /var/log/radosgw/client.radosgw.gateway.log
```

1.  启动 RadosGW

```{=html}
<!-- -->
```
``` bash
$ /etc/init.d/ceph-radosgw start
```

1.  将 RadosGW 的配置写入 Apache 中 a) 创建文件
    /etc/httpd/conf.d/rgw.conf， 内容为:

```{=html}
<!-- -->
```
``` bash
<VirtualHost *:80>
ServerName localhost
DocumentRoot /var/www/html

ErrorLog /var/log/httpd/rgw_error.log
CustomLog /var/log/httpd/rgw_access.log combined

# LogLevel debug
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]
SetEnv proxy-nokeepalive 1
ProxyPass / unix:///var/run/ceph/ceph.radosgw.gateway.fastcgi.sock|fcgi://localhost:9000/
</VirtualHost>
```

b\) 重启 Apache

``` bash
$ sudo systemctl restart httpd
```

测试
====

创建用户
--------

为了完成测试，分别创建一个用于 S3 和 Swift 的用户:

1.  创建 S3 用户:

```{=html}
<!-- -->
```
``` bash
$ sudo radosgw-admin user create --uid="testuser" --display-name="First User"
{ "user_id": "testuser",
  ... ...
  "keys": [
        { "user": "testuser",
          "access_key": "EZU2MX4CJCIZILATWQSK",
          "secret_key": "4z18K+f7MZQop2Z99PxN2KKGSX9rb6KBd9ioW0D\/"},
  ... ...
  "temp_url_keys": []}
```

1.  创建 Swift 用户: a) 创建 Swift 用户

```{=html}
<!-- -->
```
``` bash
$ sudo radosgw-admin subuser create --uid=testuser --subuser=testuser:swift
{ "user_id": "testuser",
  ... ...
  "keys": [
        { "user": "testuser",
          "access_key": "EZU2MX4CJCIZILATWQSK",
          "secret_key": "4z18K+f7MZQop2Z99PxN2KKGSX9rb6KBd9ioW0D\/"},
        { "user": "testuser:swift",
          "access_key": "SI7W1FUI43H3I3ED9CWX",
          "secret_key": ""}],
  ... ...
  "temp_url_keys": []}
```

b\) 设置 Swift 用户的密钥:

``` bash
$ sudo radosgw-admin subuser create --uid=testuser --subuser=testuser:swift
{ "user_id": "testuser",
  ... ...
  "keys": [
        { "user": "testuser",
          "access_key": "EZU2MX4CJCIZILATWQSK",
          "secret_key": "4z18K+f7MZQop2Z99PxN2KKGSX9rb6KBd9ioW0D\/"},
        { "user": "testuser:swift",
          "access_key": "SI7W1FUI43H3I3ED9CWX",
          "secret_key": ""}],
  "swift_keys": [
        { "user": "testuser:swift",
          "secret_key": "yTXiN+2y1Uf6j+CXioZwhqzwCPhOgqVblm2iShj+"}],
  ... ...
  "temp_url_keys": []}
```

测试
----

下面的例子使用 s3cmd 测试用 S3 接口连接使用 RadosGW:

NOTE: s3cmd的安装配置请参考 s3cmd 的相关文档

``` bash
$ s3cmd mb s3://my-bucket # 创建一个 bucket
Bucket 's3://my-bucket/' created
$ s3cmd ls
2015-08-03 10:11  s3://my-bucket
$ cat test.sh 
source openrc
python foo.py $@
$ s3cmd put test.sh s3://my-bucket/my-dir/test.sh # 将 test.sh 上传到 my-bucket/my-dir 下
test.sh -> s3://my-bucket/my-dir/test.sh  [1 of 1]
 31 of 31   100% in    0s   196.90 B/s  done
$ s3cmd ls s3://my-bucket # 查看目录
                       DIR   s3://my-bucket/my-dir/
$ s3cmd get s3://my-bucket/my-dir/test.sh test1.sh # 下载 test.sh 这个对象
s3://my-bucket/my-dir/test.sh -> test1.sh  [1 of 1]
s3://my-bucket/my-dir/test.sh -> test1.sh  [1 of 1]
 31 of 31   100% in    0s     5.83 kB/s  done
$ cat test1.sh
source openrc
python foo.py $@
```

Resources
=========

[CEPH OBJECT GATEWAY](http://docs.ceph.com/docs/master/radosgw/)

[Amazon S3 Tools: Command Line S3 Client Software and S3
Backup](http://s3tools.org/s3cmd)
