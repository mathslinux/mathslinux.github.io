---
layout: post
tag: QEMU_KVM
date: '\[2012-11-07 3 17:11\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 使用虚拟串口进行开发
---

开发框架
========

![](/images/posts/QEMU_KVM/develop_with_virtserialport_0.png)

一些程序分析
============

spice vdagent
-------------

ovirt guest agent
-----------------

一个简单的 demo
===============

APP 端的代码

``` python
#!/usr/bin/python
# encoding: utf-8

import socket
import time

class VSPApp(object):
    def __init__(self, path):
        self.sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.sock.connect(path)

    def run(self):
        count = 0
        while True:
            time.sleep(1)
            count += 1
            self.sock.send("free_mem")
            data = self.sock.recv(1024)
            if data:
                print data
            else:
                # peer close the socket
                print "peer close the socket, exit!!!"
                break
            if count >= 10:
                self.sock.send("shutdown")

if __name__ == "__main__":
    VSPApp("/tmp/vspagent").run()
```

Guest Agent 代码

``` python
#!/usr/bin/python
# encoding: utf-8

import select
import os
import time

class VSPAgent(object):
    def __init__(self, path):
        self.port = os.open(path, os.O_RDWR)

    def get_free_mem(self):
        free = 0
        for line in open('/proc/meminfo'):
            var, value = line.strip().split()[0:2]
            if var in ('MemFree:', 'Buffers:', 'Cached:'):
                free += long(value)
        return free

    def run(self):
        rlist = [self.port]
        while True:
            readable, writable, exceptional = select.select(rlist, [], [], 2)
            if readable:
                data = os.read(self.port, 1024)
                if data == "free_mem":
                    os.write(self.port, "mem:%d" %(self.get_free_mem()))
                elif data == "shutdown":
                    os.write(self.port, "shutdown!")
                    time.sleep(3);
                    os.system("shutdown -h now")

if __name__ == "__main__":
    VSPAgent("/dev/virtio-ports/vspagent").run()
```
