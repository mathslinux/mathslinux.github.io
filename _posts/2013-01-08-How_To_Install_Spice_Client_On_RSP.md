---
layout: post
tag: Spice
date: '\[2013-01-08 2 11:01\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: How to install spice client on Raspberry Pi
---

Download source
===============

First, we need to download the source of spice client from [spice
official website](http://spice-space.org), the latest stable version is
0.14.

``` bash
$ wget http://spice-space.org/download/gtk/spice-gtk-0.14.tar.bz2
```

Install dependencies packages
=============================

Spice client depends many other packages, e.g. jpeg, gtk, audio … We
must install these packages before compiling source.

Furthermore, since we install spice client from source, packages related
to compile are also needed, e.g. gcc, autoconf, libtool

The celt package in raspbian's repository(0.7.1) is newer than Spice
client requires(0.5.1), so we have to install this required version from
source.

``` bash
$ sudo apt-get install libogg-dev
$ wget https://launchpadlibrarian.net/59154526/celt_0.5.1.3.orig.tar.gz
$ tar xf celt_0.5.1.3.orig.tar.gz
$ cd celt-0.5.1.3
$ ./configure --prefix=/usr
$ make && sudo make install
```

All packages we need to install are following:

``` bash
$ sudo apt-get install build-essential autoconf libtool intltool libspice-protocol-dev libgtk2.0-dev libssl-dev libpulse-dev gobject-introspection libgirepository1.0-dev libjpeg8-dev pulseaudio
```

Compile and install source
==========================

This step, we are ready to compile and install Spice client.

[A accelerated X driver for Raspberry
Pi](http://www.raspberrypi.org/phpBB3/viewtopic.php?f%3D63&t%3D28294)
are being developped, and a test version has been released. The driver
takes advantage of hardware acceleration for display. So spice client
could run much smoother.

But by default, spice-gtk uses cairo as display backend, which does not
use hardware acceleration. In order to use hardware acceleration, we
must configure "–with-gtk=2.0".

``` bash
$ tar xf spice-gtk-0.14.tar.bz2
$ cd spice-gtk-0.14
$ ./configure --prefix=/usr --disable-maintainer-mode --disable-static --enable-introspection --without-python --without-sasl --disable-polkit --disable-vala --enable-smartcard=no --with-gtk=2.0
$ make
$ sudo make install
```

Ok, if no errors occur, we have completed the compilation and
installation, it's time to enjoy it.

Done, connect to server
=======================

Now you can connect to your VM using following command:

``` bash
$ spicy -h spice_server -p port
```

That's all, enjoy it!
