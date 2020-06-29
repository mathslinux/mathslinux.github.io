---
layout: post
tag: QEMU_KVM
date: '\[2012-04-22 日\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QEMU 网络配置
---

网络配置
========

VDE 配置?
---------

### 内核配置:

``` example
Device drivers --->
   Network device support --->
      [M] Universal TUN/TAP device driver support

Networking support --->
   Networking options --->
      Network packet filtering framework (Netfilter) --->
         Core Netfilter Configuration --->
            <*> Netfilter connection tracking support
         IP: Netfilter Confiuration --->
            <*> IPv4 connection tracking support (required for NAT)
            <*> IP tables support (required for filtering/masq/NAT)
            <*> Full NAT
            <*>   MASQUERADE target support
```

### 用户空间配置

安装 vde(a virtual switch). NOTE!!! 如果是源代码编译的话, 记得加上 pcap
支持, 不然创建的 VM之间是 不能通信的, pkginstall 是各个平台的包管理工具.

``` bash
# pkginstall vde
```

加载 kvm 和 tun 驱动 NOTE!!! 在 gentoo 中只需加载 kvm-intel
驱动.(kvm-intel是支持intel的KVM模块).

``` bash
# modprobe kvm kvm-intel tun
```

为 VDE 创建一个 hub.

``` bash
# vde_switch --numports 4 --mod 777 --group users --tap tap0 -d
```

设置上面创建的 tap0.

``` bash
# ifconfig tap0 10.1.1.1 broadcast 10.1.1.255 netmask 255.255.255.0
# ifconfig tap0 up
```

设置 Iptable 可以转发.

``` bash
# echo "1" > /proc/sys/net/ipv4/ip_forward
```

对 VM 过来的封包, 自动获取当前IP地址来做NAT

``` bash
# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

如果 Iptable 已经做了设置, 确保 FORWARD 的包和 tap0 的包全部可以通过.

``` bash
# iptables -A FORWARD -j ACCEPT
# iptables -A INPUT -i tap0 -j ACCEPT
```

设置 dnsmasq 来做 dhcp 和 dns 服务器.

``` bash
# pkginstall dnsmasq
# dnsmasq --log-queries --dhcp-range=10.1.1.1,10.1.1.20,255.255.255.0,2h --interface=tap0 --domain=kvm.lan
```

最后启动虚拟机, 加上上面设置的参数.

``` bash
# /usr/bin/qemu-system-x86_64 --enable-kvm -net vde -net nic,vlan=0,macaddr=52:54:00:00:EE:03 -m 1024 -net user /var/lib/libvirt/images/Ubuntu-11.10.img
```

原则上, 需要用 vde~pcapplug~ eth0 使 VM 可以和外部通信, 不然 VM 只可以和
VM 通信, 但是我在 gentoo 上测试的时候, 编译 vde 的时候加上 pcap 支持后
VM 默认就可以同外部通信了.

``` bash
# vde_pcapplug eth0
```

bridge 配置
-----------

### 内核配置

``` example
Networking --->
    Networking Options ---->
    <*> 802.1d Ethernet Bridging
```

### 用户空间配置

安装用户空间工具(brctl 等包含在 bridge-utils 中)

``` bash
# pkginstall bridge-utils
```

加载 kvm 和 tun 驱动 NOTE!!! 在 gentoo 中只需加载 kvm-intel
驱动.(kvm-intel是支持intel的KVM模块).

``` bash
# modprobe kvm kvm-intel tun
# 如果 bridge 是被编译成内核模块, 则还需要 
# modprobe bridge
```

新建并配置一个桥

``` bash
# brctl addbr br0
# ifconfig br0 192.168.1.155 netmask 255.255.255.0 up
```

安装 tun 的用户空间工具

``` bash
# pkginstall usermode-utilities # tunctl 等在此.
```

新建一个 tap 设备

``` bash
# tunctl -b -u root -t tap0
```

把这个设备挂接到上面新建的 br0 桥上.

``` bash
# brctl addif br0 tap0
```

配置上面新建的 tap 设备

``` bash
# ifconfig tap0 up 0.0.0.0 promisc
```

把自己的真实网卡(eth0)也挂接到 br0 上.

``` bash
# brctl addif br0 eth0
# ifconfig eth0 0.0.0.0 
# ifconfig eth0 up
```

启动虚拟机

``` bash
# /usr/bin/qemu-system-x86_64 --enable-kvm -net nic,macaddr=00:00:00:00:00:00 -net tap,ifname=tap0,script=no,downscript=no /var/lib/libvirt/images/Fedora16.img -m 1024
```

HOWTO
=====

怎样启用共享网络?
-----------------

``` bash
# /usr/bin/qemu-system-x86_64 --enable-kvm -net nic,macaddr=00:00:00:00:00:00 -net user /var/lib/libvirt/images/Ubuntu-11.10.img
```

Debug
=====

Install
-------

``` bash
# ./configure --prefix=/tmp/qemu --target-list="i386-softmmu x86_64-softmmu"
```
