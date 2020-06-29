---
layout: post
tag: Virtualization
date: '\[2014-10-09 四 11:25\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'cloud-init 探索'
---

背景
====

Amazon 的 EC2 有一个叫
[CloudFormation](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
的服务， 用来定制基于同一个模板的不 同实例. 通过一个本地能访问到的 API,
实例可以获取由 EC2 提供的 **实例独有** 的 数据和信息(比如 IP地址,
主机名, MAC 地址等等)在启动的时候初始化自己, 这个服
务在自动部署成千上万的实例的时候给用户提供了极大地方便.

[cloud-init](http://launchpad.net/cloud-init/)
就是一个运行在实例里面并用来初始化实例的一个框架. 它利用这些 cloud
provider(e.g. CloudFormation, nova metadata service) 提供的信息在实例
启动的时候初始化该实例, 例如, 修改主机名, 创建 ssh 密钥, 将用户的 ssh
公共密钥 导入实例中, 配置软件源, 配置 puppet, chef 等.

和其他配置工具(e.g. puppet)的区别
=================================

既然已经有 puppet/chef 等配置工具了, 为什么还需要 cloud-init
来做配置实例的 事情呢? 首先注意一个前提, 使用 puppet 这些工具之前,
需要首先安装配置好 puppet, 设置好网络, DNS, 主机名, puppet master
信息等.

当然用户也可以把这些配置步奏提前在实例基于的模板上完成,
对于部署少量的实例来说, 这样做事可行的, 但是对大规模的部署应用,
这样既费时有费力. 所以 cloud-init 不是 puppet 等工具可以取代的.

简单的例子
==========

下面先通过一个简单的例子来感受一下 cloud-init.

``` bash
# cat userdata6.txt 
#cloud-config
ssh_pwauth: true
disable_root: 0
user: root
password: abc123
chpasswd:
  expire: false
```

-   首先, 这个文件把一般云主机里面的 ssh 密码认证打开, 默认关闭的
-   然后, 通过 `disable_root` 允许 root 认证, 默认关闭
-   将 root 的密码设置为 abc123
-   如果不设置 chpasswd 的 expire 为 false,
    那么登陆的时候会提示马上修改密码才能进去

然后启动一个实例

``` bash
# nova boot --flavor 7 --key_name mykey --image fedora20 --nic net-id=[NIC-ID] --user-data ~/userdata6.txt vm
```

登陆

``` bash
# ip netns exec qrouter-e7985db5-f7d4-4811-892a-d8f7035f169a ssh root@10.11.0.28
Warning: Permanently added '10.11.0.28' (RSA) to the list of known hosts.
root@10.11.0.28's password: 
```

实现细节
========

在 EC2/OpensStack 有一个 metadata 的服务, 这个服务接受对于
<http://169.254.169.254> 的请求, 提供实例需要的数据:

``` bash
# curl http://169.254.169.254
1.0
2007-01-19
2007-03-01
2007-08-29
2007-10-10
2007-12-15
2008-02-01
2008-09-01
2009-04-04
latest

# curl http://169.254.169.254/latest/meta-data/
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
hostname
instance-action
instance-id
instance-type
kernel-id
local-hostname
local-ipv4
placement/
public-hostname
public-ipv4
public-keys/
ramdisk-id
reservation-id
security-groups

# curl http://169.254.169.254/latest/meta-data/local-ipv4
10.11.0.28

# curl http://169.254.169.254/latest/meta-data/public-ipv4
192.168.3.117

# curl http://169.254.169.254/latest/user-data
#cloud-config
ssh_pwauth: true
disable_root: 0
user: root
password: abc123
chpasswd:
  expire: false
```

然后 cloud-init 利用 user-data 做初始化云主机的操作.

格式
====

User data 是 cloud-init 的核心内容, 它支持多种格式: e.g. 脚本,
cloud-config

格式例子
--------

### user script

以 **\#!** 的文件或者以 **Content-Type: text/x-shellscript** 开始的 MIME
格式

``` python
#!/usr/bin/python
import os
os.system('echo "test about user data" >> /tmp/userdata')
```

``` bash
# ip netns exec qrouter-e7985db5-f7d4-4811-892a-d8f7035f169a ssh fedora@10.11.0.31
[fedora@vm3 ~]$ cat /tmp/userdata
test about user data
```

### Cloud Config Data

Cloud-config 是 cloud-init 支持的格式, 通过这种格式,
可以很方便的完成很多 任务:

-   apt 升级
-   yum 升级
-   设置 DNS, 网络文件
-   ……

该格式内部定义了很多语法, 方便用户的部署, 比如:

更新 hostname:

``` bash
#cloud-config

manage_etc_hosts: true
fqdn: eayunstack.domain
```

创建新文件:

``` bash
#cloud-config

write_files:
-   encoding: b64
    content: YWJjMTIzCg== 
    owner: root:root
    path: /var/b64_test
    permissions: '0644'
-   content: |
        EayunStack VM 
    path: /etc/issue
```

### MIME

怎样同时传递多种格式/多个文件的 user-data 呢, 可以通过 MIME
来包装多个文件:

通过下面的 python 脚本, 将多个文件封装成一个 cloud-init 支持的 MIME
文件传递 给云主机

``` python
#!/usr/bin/python

# filename: mime.py

import sys

from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

if len(sys.argv) == 1:
    print("%s input-file:type ..." % (sys.argv[0]))
    sys.exit(1)

combined_message = MIMEMultipart()
for i in sys.argv[1:]:
    (filename, format_type) = i.split(":", 1)
    with open(filename) as fh:
        contents = fh.read()
    sub_message = MIMEText(contents, format_type, sys.getdefaultencoding())
    sub_message.add_header('Content-Disposition', 'attachment; filename="%s"' % (filename))
    combined_message.attach(sub_message)

print(combined_message)
```

``` bash
# ./test.py python.txt:x-shellscript userdata5.txt:cloud-config
Content-Type: multipart/mixed; boundary="===============1752240832932743336=="
MIME-Version: 1.0

--===============1752240832932743336==
MIME-Version: 1.0
Content-Type: text/text/x-shellscript; charset="us-ascii"
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="python.txt"

#!/usr/bin/python
import os
os.system('echo "test about user data" >> /tmp/userdata')

--===============1752240832932743336==
MIME-Version: 1.0
Content-Type: text/text/cloud-config; charset="us-ascii"
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="userdata5.txt"

#cloud-config

manage_etc_hosts: true
fqdn: eayunstack.domain

--===============1752240832932743336==--
```

### Include file

``` bash
#include
http://192.168.3.30:8000/python.txt
http://192.168.3.30:8000/host.txt
```

### Upstart job

### Part Handler

这种形式的 user-data 可以让用户以 python 脚本的方式编写 cloud-init
的插件.

它其实只是一个 python 代码片段, 必须实现 `list_types()` 和
`handle_part()` 方法, `list_types()` 指定了能被该模块处理的 MIME 类型,
`handle_part()` 是具体 的处理逻辑, 他接受用户传递的 MIME 的数据,
处理这些数据, 达到用户的需求.

下面的例子展示使用用户传递的用户名创建用户:

首先根据用户提供的文件创建传递给云主机的 user-data

``` bash
# cat one 
dunrong
hunt
# cat two 
lee
coffee
# ./test.py part-handler.txt:part-handler one:plain two:plain > user-data.txt
```

``` python
# cat part-handler.txt 
#part-handler

def list_types():
    # 放回一个能被这个模块处理的 MIME 类型的列表
    return(["text/plain", "text/go-cubs-go"])

def handle_part(data,ctype,filename,payload):
    # data: the cloudinit object
    # ctype: '__begin__', '__end__', 或者是用户指定的 MIME 类型
    # filename: 用户传递的文件名
    # payload: 用户传递的文件内容
    if ctype == "__begin__":
       print "my handler is beginning"
       return
    if ctype == "__end__":
       print "my handler is ending"
       return

    print "==== received ctype=%s filename=%s ====" % (ctype,filename)
    import os
    for user in payload.splitlines():
        print " == Creating user %s" % (user)
        os.system('useradd -p linux -m %s' % (user) ) # TODO
    print "==== end ctype=%s filename=%s" % (ctype, filename)
```

``` bash
# ip netns exec qrouter-e7985db5-f7d4-4811-892a-d8f7035f169a ssh fedora@10.11.0.39
[fedora@vm3 ~]$ ls /home/
coffee  dunrong  fedora  hunt  lee

[fedora@vm3 ~]$ less /var/log/cloud-init.log
Oct  8 09:45:19 localhost cloud-init: my handler is beginning
Oct  8 09:45:19 localhost cloud-init: ==== received ctype=text/plain filename=one ====
Oct  8 09:45:19 localhost cloud-init: == Creating user dunrong
Oct  8 09:45:19 localhost cloud-init: == Creating user hunt
Oct  8 09:45:19 localhost cloud-init: ==== end ctype=text/plain filename=one
Oct  8 09:45:19 localhost cloud-init: ==== received ctype=text/plain filename=two ====
Oct  8 09:45:19 localhost cloud-init: == Creating user lee
Oct  8 09:45:19 localhost cloud-init: == Creating user coffee
Oct  8 09:45:19 localhost cloud-init: ==== end ctype=text/plain filename=two
Oct  8 09:45:19 localhost cloud-init: my handler is ending
```

目录布局
========

数据来源
========

数据来源说的是 user-data/metadata 的出处, 随着历史的发展, 有以下几种:

-   EC2 由于历史的原因, 这是最流行的数据来源, 它通过一个 http
    服务(IP:169.254.169.254) 来提供 user-data.
-   Config Drive 这是 OpenStack 使用的方式(ovirt 目前也是用这种方式)
-   OpenNebula
-   Alt cloud
-   No cloud
-   MAAS
-   CloudStack
-   OVF
-   Fallback/None

```{=html}
<!-- -->
```
``` bash
# nova boot --config-drive true --image fedora20 --key-name mykey --flavor 8 --user-data ./userdata6.txt --nic net-id=b4af9baa-be56-4958-8643-feaeb7b21dea --file /etc/network/interfaces=/root/interfaces --file known_hosts=/root/.ssh/known_hosts --meta role=webservers --meta essential=false vm
# ip netns exec qrouter-e7985db5-f7d4-4811-892a-d8f7035f169a ssh root@10.11.0.42
[root@vm ~] # mount  /dev/disk/by-label/config-2 /mnt
[root@vm ~] # ls /mnt/
ec2  openstack
[root@vm ~]# ls /mnt/openstack/
2012-08-10  2013-04-04  2013-10-17  content  latest
[root@vm ~]# ls /mnt/openstack/latest/
meta_data.json  user_data  vendor_data.json
```

建立支持 cloud-init 的镜像
==========================

Resources
=========

[cloud-init
Documentation](http://cloudinit.readthedocs.org/en/latest/index.html)

[cloud config
example](http://bazaar.launchpad.net/~cloud-init-dev/cloud-init/trunk/view/head:/doc/examples/cloud-config.txt)
