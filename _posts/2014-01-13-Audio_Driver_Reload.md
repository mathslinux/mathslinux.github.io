---
layout: post
tag: Mac
date: '\[2014-01-13 一 23:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 重启 Mac OSX 的声卡驱动
---

最近不知道怎么回事, 我的 air
总是碰到声卡突然不能工作的情况(虽然概率很小, 但是已经碰到两次了),
每次都需要重启才能工作. 之前用 linux 的时候, 碰到各种 硬件不工作的时候,
都是 "卸载模块 && 加载模块" 来 work around 的, 按理说 OSX
是混合内核的系统, 是比 linux 还要接近微内核架构的牛逼内核,
至少这种方式也能在 OSX 上工作才对, 放狗搜索了一下 OSX 怎么重载驱动模块,
果然找到了:

``` bash
sudo kextunload /System/Library/Extensions/AppleHDA.kext
sudo kextload /System/Library/Extensions/AppleHDA.kext
```
