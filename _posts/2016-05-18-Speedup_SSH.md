---
layout: post
tag: Others
date: '\[2016-05-18 三 19:30\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 加速 SSH 连接的 tips
---

SSH 登陆服务器的时候有很多流程，例如:

-   不必要的将所有支持的登陆方式都尝试一遍知道某一种登陆成功.
-   某些场景下不必要的 DNS 反向解析验证.

客户端指定验证方式加速连接
==========================

openssh 有基于 GSSAPI, host, 公钥/私钥, challenge-response,
密码等认证方式. 默认情况下,
客户端会使用一定的顺序尝试认证，直到某一种认证通过. 实际上,
在实际使用中，
我们一般只会使用秘钥和密码这两种认证方式，其他方式完全没有必要.
并且还会浪费时间.

可以在 ssh 客户端用户的配置文件中禁用其他配置.

``` bash
# file: ~/.ssh/config
PreferredAuthentications publickey,password
```

上面的配置表示客户端仅支持(秘钥)和password(密码)认证.
并且优先尝试秘钥认证.

关闭 UseDNS
===========

如果 UseDNS 选项被打开, 在客户端发起连接的时候, SSH 服务器会做一个 DNS
的反向查询验证, 根据客户端的 IP 地址进行 DNS PTR 反向查询,
用查询出到的客户端的主机名进行 DNS 正向 A 记录查询,
验证和原始地址是否一致. 一般来讲, 我们的客户端的 IP 都是动态的, 不存在
PTR 记录, SSH 服务器做这个验证只会浪费时间, 如下的 debug 信息:

``` bash
May 18 06:22:51 localhost sshd[4969]: debug3: mm_answer_pwnamallow
May 18 06:22:51 localhost sshd[4969]: debug3: Trying to reverse map address 25.0.0.1.  <-- 这里浪费了 15 秒, 很恼人
May 18 06:23:06 localhost sshd[4969]: debug2: parse_server_config: config reprocess config len 757
May 18 06:23:06 localhost sshd[4969]: debug3: mm_answer_pwnamallow: sending MONITOR_ANS_PWNAM: 1
```

所以一般推荐关闭这个选项, 修改 **/etc/ssh/sshd~config~** 中 UseDNS 为
no:

``` bash
UseDNS no
```

PS: 稍新的 openssh 版本改选项默认已经关闭

pam 认证相关
============

有时候登录很慢, 查询日志: 通常都会看到类似 pam 的问题:

``` bash
May 18 00:00:26 m8x CROND[14936]: pam_systemd(crond:session): Failed to release session:
```

在支持 systemd 的系统上, 一般这种情况是处理用户登录的 systemd-logind
服务造成的, 重启该服务即可.

``` bash
$ sudo systemctl restart systemd-logind.service
```
