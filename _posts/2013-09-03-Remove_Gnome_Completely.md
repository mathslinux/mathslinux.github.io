---
layout: post
tag: Gentoo
date: '\[2013-09-03 二 17:05\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Remove Gnome From Gentoo Completely
---

It has been a year since I didn't use gnome! The new Gnome3 disappointed
me too much. So I use LXDE as my desktop, which is more simple, more
confortable to me. But for some reasons, Gnome still get inside in my
system.

Today, When I attempt to upgrade all my installed packages to latest,
the package "systemd" drives me crazy. It conflicts with udev, which is
a very basic packages in gentoo. Since I have not been ready to use
systemd, all packages will be broken without udev installed correctlly.
I have used many ways to solve the issue, e.g. mask udev and systemd
temporarily, however, all methods I have tried failed.

As above metioned, I don't need systemd at all actually, so why I need
to install it when upgrade my system. After some research, I find it is
the gnome's upgrade which requires systemd.

**BS**, now I find the reason, the fix is very easy.

``` bash
// Add USE="-gnome" in /etc/make.conf file 
// Set system profile through "eselect profile set", i.e. dont select gnome profile
emerge -C $(grep gnome /var/lib/portage/world) # Delete all packages belong to gnome
emerge -DuNav world # Rebuild the system
emerge -Da --depclean # Clean up system
revdep-rebuild -i # Check whether something is missing
```

That's all!
