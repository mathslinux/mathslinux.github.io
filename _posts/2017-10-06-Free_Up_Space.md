---
layout: post
tag: Mac
date: '\[2017-10-06 五 23:00\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 释放 Mac 上不用的硬盘空间
---

**\~/Library/Developer/Xcode/DerivedData**

:   存放 Xcode 项目编译过程中的临时文件， 可以安全的删除。

**\~/Library/Developer/Xcode/Archives**

:   编译的目标文件存放处，删除之。

**/Library/Developer/Xcode/iOS DeviceSupport**

:   Xcode 用来 debug ios 开发的文件，
    这里我用不到，删除之，如果需要保留这个目录进行 ios
    开发，仅保留需要支持的 ios 版本即可。清理该目录可以释放大量空间。

**/Library/Developer/CoreSimulator**

:   iphone 和 ipad的模拟器都在这里， 非 ios
    开发者完全用不到，删除之。如果需要保留这个目录进行 ios 开发，
    仅保留需要支持的 iphone 和 ipad
    版本即可。清理该目录可以释放大量空间。

**/Library/Caches/com.apple.dt.Xcode**

:   Xcode 存放 Cache 的地方，可以安全的删除。

**/Library/Application Support/MobileSync/Backup**

:   ios 设备和 Mac 的同步文件 会放到该文件下，通过 iTunes -\> 偏好设置
    -\> 设备查看管理备份文件，将不用 和过期的删除

**/Applications/Xcode.app/Contents/Developer/Platforms**

:   所有苹果设备
    （mac，iphone，watchtv）的sdk都在这里，根据需求删除完全不需要的。

    ``` bash
    $ cd /Applications/Xcode.app/Contents/Developer/Platforms
    $ sudo rm -rf AppleTV* Watch* iPhone*
    ```

    清理该目录可以释放大量空间。
