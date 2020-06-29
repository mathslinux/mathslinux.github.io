---
layout: post
tag: Others
date: '\[2016-10-12 三 19:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: lxde 桌面屏幕截图配置
---

在 Mac 或者 Windows 下使用微信截图工具很方便，hack 一个到 lxde.

这里选择 scrot 作为后端截图的引擎

修改 **\$HOME/.config/openbox/lxde-rc.xml** 文件, 增加以下一个配置项:

``` xml
<keybind key="Print">
  <action name="Execute">
    <execute>scrot -s '%F-%H%M%S.png' -e 'mv $f ~/Desktop/'</execute>
  </action>
</keybind>
```

表示当按下 PrtScn 的时候，调用 scrot 选择区域截图，
截图完成之后将截图按照 **year-month-day-时分秒.png** 的格式保存到桌面上

稍后，让 openbox 重载配置文件即可

``` bash
$ openbox --reconfigure
```

如果习惯 QQ 或者 Wechat 快捷键截图，只需修改按键绑定到 Ctrl-Shift-A
即可:

``` bash
<keybind key="C-A-A">
```
