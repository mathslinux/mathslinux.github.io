---
layout: post
tag: QEMU_KVM
date: '\[2013-09-23 一 16:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Get the guest IP address of Virtual Machine
---

These days, I am developing a [small
project](https://github.com/mathslinux/vman). One of the convenient
functions it provides is that user can access QEMU virtual machine
directly via ssh as long as user knows the name of VM.

There are two methods to archive this if user knows the guest's MAC
address:

Using ARP
=========

[ARP](http://en.wikipedia.org/wiki/Address_Resolution_Protocol) is a
protocol which is used to convert an IP address to [MAC
Address](http://en.wikipedia.org/wiki/MAC_address).

With the ARP, we can get the MAC Address of a machine by its IP address,
but what we have known is guest's MAC address and what we want to know
is IP address.

A hack method is to query all IP\<-\>MAC address, then filter them by
guest's MAC.

``` python
#!/usr/bin/python

import commands
import sys

def find_ipaddr(mac):
    outtext = commands.getoutput('arp -n')
    for l in outtext.split('\n'):
        if l.split()[2] == mac:
            return l.split()[0]
    return None

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print 'please input MAC Address'
        exit(1)
    ip = find_ipaddr(sys.argv[1])
    if ip:
        print '%s <-> %s' % (sys.argv[1], ip)
    else:
        print '%s <-> None' % (sys.argv[1])
```

Using GuestAgent
================

[GuestAgent](http://wiki.qemu.org/Features/QAPI/GuestAgent) is a tool
provided by QEMU official which aims to provide access to a system-level
agent via standard QMP commands.

There are many commands available, one of them named
"guest-network-get-interfaces", outputs guest network information we
need.

This way is more useful and graceful than using ARP, we can also get the
guest's IP address even if guest is on another subnet.

The following code shows how it works:

Start guest, append following arguments:

``` bash
-chardev socket,id=qga0,path=/tmp/qga.sock,server,nowait \
-device virtio-serial -device virtserialport,chardev=qga0,name=org.mathslinux.qemu
```

Or if you want to use libvirt to start guest, append following xml under
\<device\> tag

``` xml
<channel type='unix'>
  <source mode='bind' path='/tmp/qga.sock'/>
  <target type='virtio' name='org.mathslinux.qemu'/>
</channel>
```

Then in the guest, run qemu-ga

``` bash
./qemu-ga -p /dev/virtio-ports/org.mathslinux.qemu
```

After guest starts, we can fetch its IP address by connecting and
sending "guest-network-get-interface" command to unix socket defined
above.

``` python
#!/usr/bin/python

import socket
import sys

path = '/tmp/qga.sock'

def find_ipaddr(mac):
    sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    sock.connect(path)

    sock.send('{"execute": "guest-network-get-interfaces"}')
    data = sock.recv(4096)
    for l in eval(data)['return']:
        if l['hardware-address'] == mac:
            ip_list = l['ip-addresses']
            for ip in ip_list:
                if ip['ip-address-type'] == 'ipv4':
                    return ip['ip-address']

    return None

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print 'please input MAC Address'
        exit(1)
    ip = find_ipaddr(sys.argv[1])
    if ip:
        print '%s <-> %s' % (sys.argv[1], ip)
    else:
        print '%s <-> None' % (sys.argv[1])
```
