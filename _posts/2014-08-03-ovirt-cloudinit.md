---
layout: post
tag: Virtualization
date: '\[2014-08-03 日 21:20\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: oVirt 中的 cloudinit 体验
---

为了解决云平台中的初始化问题, 开发者们设计了
[cloudinit](http://cloudinit.readthedocs.org/en/latest/),
它驻留在虚拟机中, 随机启动, 根据配置文件和云平台提供的 userdata
进行相关配置. oVirt 从很早 的版本就有了 cloudinit 的支持. 下面我在
oVirt-3.4 中简单的体验一下 cloudinit 的功能.

首先, 得有一个支持 cloudinit 的虚拟机, 为了方便, 我们从内置的
ovirt-image-repository 中导入一个 fedora19 的镜像作为模板创建虚拟机:
选择 \[系统\] –\> \[存储\] –\> \[ovirt-image-repository\],
在列出的镜像中选择 "Fedora 19 64-Bit", \[导入\] –\> \[作为模板导入\]

然后, 创建虚拟机: \[新建虚拟机\] –\> 选择刚才导入的 fedora19 作为模板,
设置虚拟机类型为 linux, 点击 \[显示高级选项\] –\> \[初始运行\] –\>
\[使用 Cloud-Init/Sysprep\] –\> \[验证\], 把想要设置的密码填写进去,
把本机的 ssh 公钥输入进去, 保存.

然后, 启动虚拟机. 过一会儿, 在界面上显示虚拟机的 IP 地址之后,
连接该虚拟机

``` bash
# ssh root@192.168.2.225
```

如果之前导入公钥是本机的话, 不用密码直接就可以登陆了,
如果是在另外的机器, 输入 上面设置的密码也可以登陆进去.

深入一点
========

通过查看该虚拟机的启动参数, 发现 cloudinit 的配置是以 IDE
设备的方式注入进去的.

``` bash
-drive file=/var/run/vdsm/payload/81654b36-1d6e-4093-82a0-2f730ca720d1.3944715d95e1a09bf282ede59538de1e.img
# mount -o loop /var/run/vdsm/payload/81654b36-1d6e-4093-82a0-2f730ca720d1.3944715d95e1a09bf282ede59538de1e.img  /mnt
# ls /mnt/openstack/latest/
meta_data.json  user_data
```

至于数据源的格式, 完全是借用了 openstack 的标准:

``` bash
# cat /mnt/openstack/latest/user_data 
#cloud-config
ssh_pwauth: true
disable_root: 0
output:
  all: '>> /var/log/cloud-init-output.log'
user: root
password: abc123
chpasswd:
  expire: false
runcmd:
- 'sed -i ''/^datasource_list: /d'' /etc/cloud/cloud.cfg; echo ''datasource_list:
  ["NoCloud", "ConfigDrive"]'' >> /etc/cloud/cloud.cfg'
```

在虚拟机内部, cloudinit 把数据源所在的磁盘 /dev/sr1 临时挂载,
然后把源数据复制到 cloudinit 的目录下.

``` bash
# less /var/log/cloud-init.log
... util.py[DEBUG]: Running command ['mount', '-o', 'ro,sync', '/dev/sr1', '/tmp/tmpdem7bo'] with allowed return codes [0] (shell=False, capture=True)
# cat /var/lib/cloud/instance/user-data.txt
#cloud-config
ssh_pwauth: true
disable_root: 0
output:
  all: '>> /var/log/cloud-init-output.log'
user: root
password: abc123
chpasswd:
  expire: false
runcmd:
- 'sed -i ''/^datasource_list: /d'' /etc/cloud/cloud.cfg; echo ''datasource_list:
  ["NoCloud", "ConfigDrive"]'' >> /etc/cloud/cloud.cfg'
```
