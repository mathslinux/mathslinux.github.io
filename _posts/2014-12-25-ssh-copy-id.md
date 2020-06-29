---
layout: post
tag: Mac
date: '\[2014-12-25 四 11:15\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: '给 Mac 一个 ssh-copy-id'
---

Mac 下默认是没有这个非常好用的工具的, 最开始我每次手动把
**\~/.ssh/id~rsa~.pub** 的 内容追加到服务器的
**\~/.ssh/authorized~keys~** 文件中.

后来我自己写了一个脚本, 放到可执行目录中, 爽了很多:

``` bash
#!/bin/bash

# file: ssh-copy-id

if [ $# -lt 1 ]; then
    echo 'Usage: ssh-copy-id [user1@]hostname1 [user2@]hostname2]'
fi

KEY_FIEL="$HOME/.ssh/id_rsa.pub"
if [ -f $KEY_FIEL ]; then
    key=`cat $KEY_FIEL`
    for server in $@; do
        ssh $server "echo $key >> ~/.ssh/authorized_keys"
    done
else
    echo 'ssh key(id_rsa.pub) does not exist, run ssh-keygen to generate it'
    exit 1
fi
```

在后来, 我偶然发现 brew 中直接有一个包就叫 **ssh-copy-id**, 我靠, 一直被
linux 系统给误导了, 在 linux 系统中, **ssh-copy-id** 是包含在名为
**openssh-clients** 的包中的. 废话不说, 用这个吧, 毕竟人家写的 300 多行,
考虑了各种兼容性等等

``` bash
$ brew install ssh-copy-id
```
