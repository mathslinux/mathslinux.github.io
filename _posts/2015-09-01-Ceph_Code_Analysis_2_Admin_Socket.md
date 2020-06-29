---
layout: post
tag: Storage
date: '\[2015-09-01 二 16:20\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Ceph 深入解析(2) – Ceph Admin Socket
---

介绍
====

在 Ceph 每个组件(e.g. Monitor, OSD, CephFS) 运行的时候，会创建一个基于
UNIX socket 的服务，该服务通过这个 socket 提供一个和该组件交互的接口.
用户可以 通过 **ceph daemon {socket-file} command**
查询/配置一些常用的参数.

使用
====

使用下列格式访问 admin socket:

``` bash
$ ceph daemon [socket-file] [command]
```

其中， socket-file 是该组件的 unix socket 的路径, command
是具体的查询指令

一般可以通过 **help** 得到改组件的 admin socket 支持的命令

``` bash
$ ceph daemon [socket-file] help
```

下面举几个例子:

1.  查询 monitor 的 quorum 状态:

```{=html}
<!-- -->
```
``` bash
$ ceph daemon /var/run/ceph/ceph-mon.ceph1.asok quorum_status
{ "election_epoch": 304,
  "quorum": [
        0,
        1,
        2],
  "quorum_names": [
        "ceph1",
        "ceph2",
        "ceph3"],
  "quorum_leader_name": "ceph1",
  "monmap": { "epoch": 3,
      "fsid": "8e92b691-a67a-4e26-969a-4e4f18ed3fa0",
      "modified": "2015-06-17 19:35:00.946699",
      "created": "0.000000",
      "mons": [
            { "rank": 0,
              "name": "ceph1",
              "addr": "192.168.3.137:6789\/0"},
            { "rank": 1,
              "name": "ceph2",
              "addr": "192.168.3.138:6789\/0"},
            { "rank": 2,
              "name": "ceph3",
              "addr": "192.168.3.139:6789\/0"}]}}
```

1.  查询本节点的 OSD 状态

```{=html}
<!-- -->
```
``` bash
$ ceph daemon /var/run/ceph/ceph-osd.1.asok status
{ "cluster_fsid": "8e92b691-a67a-4e26-969a-4e4f18ed3fa0",
  "osd_fsid": "05ee7ec8-2309-47b1-81bf-f6ef49d0f87f",
  "whoami": 1,
  "state": "active",
  "oldest_map": 1,
  "newest_map": 280,
  "num_pgs": 699}
```

实现代码解析
============

![](/images/posts/Storage/mon_00.png)

``` c
bool AdminSocket::init(const std::string &path)
{
    // 创建一个管道，该管道用来让父进程关闭该子进程，看下面 entry 的处理
    err = create_shutdown_pipe(&pipe_rd, &pipe_wr);

    // 创建 Unix socket, 在该 socket 上监听客户端的连接
    err = bind_and_listen(path, &sock_fd);

    // 注册通用的命令 hook
    m_version_hook = new VersionHook;
    register_command("0", "0", m_version_hook, "");
    register_command("version", "version", m_version_hook, "get ceph version");
    register_command("git_version", "git_version", m_version_hook, "get git sha1");
    m_help_hook = new HelpHook(this);
    register_command("help", "help", m_help_hook, "list available commands");
    m_getdescs_hook = new GetdescsHook(this);
    register_command("get_command_descriptions", "get_command_descriptions",
                     m_getdescs_hook, "list available commands");

    // 进入新线程中执行 AdminSocket::entry()
    create();
}

void* AdminSocket::entry()
{
    while (true) {
        // 使用 poll 来轮训 socket, 注意，该线程轮训 Unix socket 和与父进程通信的
        // 管道
        struct pollfd fds[2];
        memset(fds, 0, sizeof(fds));
        fds[0].fd = m_sock_fd;
        fds[0].events = POLLIN | POLLRDBAND;
        fds[1].fd = m_shutdown_rd_fd;
        fds[1].events = POLLIN | POLLRDBAND;

        int ret = poll(fds, 2, -1);
        if (fds[0].revents & POLLIN) {
            // 处理这个请求
            do_accept();
        }
        // 如果进程推出，会通过 pipe 的写端发送一个字节
        if (fds[1].revents & POLLIN) {
            // Parent wants us to shut down
            return PFL_SUCCESS;
        }
    }
}

bool AdminSocket::do_accept()
{
    int connection_fd = accept(m_sock_fd, (struct sockaddr*) &address,
                               &address_length);
    char cmd[1024];
    int pos = 0;

    // 读取客户端发送的请求到 cmd
    while (1) {
        int ret = safe_read(connection_fd, &cmd[pos], 1);
        // new protocol: null or \n terminated string
        if (cmd[pos] == '\n' || cmd[pos] == '\0') {
            cmd[pos] = '\0';
            c = cmd;
            break;
        }
        pos++;
    }

    // 通过得到的 cmd 从所有注册的 hooks(AdminSocket::m_hooks)中找到匹配的
    // hook
    p = m_hooks.find(match);

    bufferlist out;
    // 调用 hook 的 AdminSocketHook::call 方法处理改请求，将结果填充到 out
    // 中
    bool success = p->second->call(match, cmdmap, format, out);

    // 发送请求结果
    uint32_t len = htonl(out.length());
    int ret = safe_write(connection_fd, &len, sizeof(len));
    if (out.write_fd(connection_fd) >= 0)
        rval = true;
}
```
