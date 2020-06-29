---
layout: post
tag: QEMU_KVM
date: '\[2013-10-16 3 15:10\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: QEMU CPU Hotplug
---

// this post has not be done yet.

In the world of cloud compute, Scaling the compute resources without
shutdown the compute instance is important. e.g. Add/Reduce cpu number,
memory size, and so on.

In the version 1.5, QEMU has implemented the feature: CPU Hotplug, it
allows virtual machine to add cpu core without shutdown.

Below is how to do it:

Before try this feature, please make sure your QEMU version \>= 1.5.0,
or this feature is not supported.

``` bash
# ~/usr/bin/qemu-system-x86_64 --version
QEMU emulator version 1.6.50, Copyright (c) 2003-2008 Fabrice Bellard
```

Use QEMU command directly to enable this feature
================================================

``` bash
# qemu-system-x86_64 -enable-kvm -m 512 -smp 1,maxcpus=4 ArchLinux.raw -serial \
   tcp::3333,nowait,server -qmp tcp::5555,nowait,server -nographic
```

the maxcpus means the maximum number of hotpluggable CPUs.

``` bash
[root@myhost ~]# cat /proc/cpuinfo | grep processor
processor   : 0
```

After active the CPU we just add

``` bash
# telnet localhost 5555
......
{ "execute": "qmp_capabilities" }
{"return": {}}
{ "execute": "cpu-add", "arguments": { "id": 1 }}
{"return": {}}
```

``` bash
[root@myhost ~]# cat /proc/cpuinfo | grep processor
processor   : 0
processor   : 1
```

Use libvirt to enable this feature
==================================
