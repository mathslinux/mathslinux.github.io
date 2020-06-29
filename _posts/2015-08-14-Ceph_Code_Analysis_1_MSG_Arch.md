---
layout: post
tag: Storage
date: '\[2015-08-14 五 17:20\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: Ceph 深入解析(1) – Ceph 的消息处理架构
---

Ceph 的消息处理主要关联到以下几个类:

![消息处理架构](/images/posts/Storage/network_arch1.jpg)

架构上采用
[Publish/subscribe(发布/订阅)](https://zh.wikipedia.org/wiki/%25E5%258F%2591%25E5%25B8%2583/%25E8%25AE%25A2%25E9%2598%2585)
的设计模式.

模块说明
========

class Messenger
---------------

该类作为消息的发布者, 各个 Dispatcher 子类作为消息的订阅者, Messenger
收到消息之后，通过 Pipe 读取消息，然后转给 Dispatcher 处理

class SimpleMessenger
---------------------

Messenger 接口的实现

class Dispatcher
----------------

该类是订阅者的基类，具体的订阅后端继承该类，初始化的时候通过
`Messenger::add_dispatcher_tail/head` 注册到 `Messenger::dispatchers`.
收到消息后，通知改类处理

class Accepter
--------------

监听 peer 的请求, 有新请求时, 调用 `SimpleMessenger::add_accept_pipe()`
创建新的 Pipe 到 SimpleMessenger::pipes 来处理该请求

class Pipe
----------

用于消息的读取和发送，该类主要有两个组件，Pipe::Reader 和 Pipe::Writer,
分别用来处理 消息的读取和发送. 这两个类都是 class Thread
的子类，意味这每次处理消息都会有两个 线程被分别创建.

消息被 Pipe::Reader 读取后，该线程会通知注册到 Messenger::dispatchers
中的某一个 Dispatcher(如 Monitor) 处理, 处理完成之后将回复的消息放到
`SimpleMessenger::Pipe::out_q` 中，供 Pipe::Writer 来处理发送

class DispatchQueue
-------------------

该类用来缓存收到的消息, 然后唤醒 `DispatchQueue::dispatch_thread`
线程找到后端的 Dispatch 处理消息

深入解析
========

![解析](/images/posts/Storage/network_arch2.jpg)

下面的代码涉及到的订阅子类以 Monitor 为例:

初始化
------

``` c
int main(int argc, char *argv[])
{
    // 创建一个 Messenger 对象，由于 Messenger 是抽象类，不能直接实例化，提供了一个
    // ::create 的方法来创建子类，目前 Ceph 所有模块使用 SimpleMessenger
    Messenger *messenger = Messenger::create(g_ceph_context,
                                             entity_name_t::MON(rank),
                                             "mon",
                                             0);

    /**
     * 执行 socket() -> bind() -> listen() 等一系列动作, 执行流程如下:
     SimpleMessenger::bind()
         --> Accepter::bind()
             socket() -> bind() -> listen()
    */
    err = messenger->bind(ipaddr);

    // 创建一个 Dispatch 的子类对象, 这里是 Monitor
    mon = new Monitor(g_ceph_context, g_conf->name.get_id(), store, 
                      messenger, &monmap);

    // 启动 Reaper 线程
    messenger->start();

    /**
     * a). 初始化 Monitor 模块
     * b). 通过 SimpleMessenger::add_dispatcher_tail() 注册自己到
     * SimpleMessenger::dispatchers 中, 流程如下:
     * Messenger::add_dispatcher_tail()
     *      --> ready()
     *        --> dispatch_queue.start()(新 DispatchQueue 线程)
              --> Accepter::start()(启动start线程)
     *            --> accept
     *                --> SimpleMessenger::add_accept_pipe
     *                    --> Pipe::start_reader
     *                        --> Pipe::reader()
     * 在 ready() 中: 通过 Messenger::reader(),
     * 1) DispatchQueue 线程会被启动，用于缓存收到的消息消息
     * 2) Accepter 线程启动，开始监听新的连接请求.
     */
    mon->init();

    // 进入 mainloop, 等待退出
    messenger->wait();
    return 0;
}
```

消息处理
--------

### 收到连接请求

请求的监听和处理由 SimpleMessenger::ready –\> Accepter::entry 实现

``` c
void SimpleMessenger::ready()
{
    // 启动 DispatchQueue 线程
    dispatch_queue.start();

    lock.Lock();
    // 启动 Accepter 线程监听客户端连接, 见下面的 Accepter::entry
    if (did_bind)
        accepter.start();
    lock.Unlock();
}

void *Accepter::entry()
{
    struct pollfd pfd;
    // listen_sd 是 Accepter::bind() 中创建绑定的 socket
    pfd.fd = listen_sd;
    pfd.events = POLLIN | POLLERR | POLLNVAL | POLLHUP;
    while (!done) {
        int r = poll(&pfd, 1, -1);
        if (pfd.revents & (POLLERR | POLLNVAL | POLLHUP))
            break;
        if (done) break;
        entity_addr_t addr;
        socklen_t slen = sizeof(addr.ss_addr());
        int sd = ::accept(listen_sd, (sockaddr*)&addr.ss_addr(), &slen);
        if (sd >= 0) {
            // 调用 SimpleMessenger::add_accept_pipe() 处理这个连接
            msgr->add_accept_pipe(sd);
        } 
    }
    return 0;
}
```

随后创建 Pipe() 开始消息的处理

``` c
Pipe *SimpleMessenger::add_accept_pipe(int sd)
{
    lock.Lock();
    Pipe *p = new Pipe(this, Pipe::STATE_ACCEPTING, NULL);
    p->sd = sd;
    p->pipe_lock.Lock();
    // 
    /**
     * 调用 Pipe::start_reader() 开始读取消息, 将会创建一个读线程开始处理.
     * Pipe::start_reader() --> Pipe::reader
     */
    p->start_reader();
    p->pipe_lock.Unlock();
    pipes.insert(p);
    accepting_pipes.insert(p);
    lock.Unlock();
    return p;
}
```

### 创建消息读取和发送线程

处理消息由 `Pipe::start_reader()` –\> `Pipe::reader()`
开始，此时已经是在 Reader 线程中. 首先会调用 accept()
做一些简答的处理然后创建 Writer() 线程，等待发送回复 消息. 然后读取消息,
读取完成之后, 将收到的消息封装在 Message 中，交由 `dispatch_queue()`
处理.

`dispatch_queue()` 找到注册者，将消息转交给它处理，处理完成唤醒 Writer()
线程发送回复消息.

``` c
void Pipe::reader()
{
    /**
     * Pipe::accept() 会调用 Pipe::start_writer() 创建 wirter 线程, 进入 writer 线程
     * 后，会 cond.Wait() 等待被激活，激活的流程看下面的说明. Writer 线程的创建见后后面
     * Pipe::accept() 的分析
     */
    if (state == STATE_ACCEPTING) {
        accept();
    }

    while (state != STATE_CLOSED &&
           state != STATE_CONNECTING) {
        // 读取消息类型，某些消息会马上激活 writer 线程先处理
        if (tcp_read((char*)&tag, 1) < 0) {
            continue;
        }
        if (tag == CEPH_MSGR_TAG_KEEPALIVE) {
            continue;
        }
        if (tag == CEPH_MSGR_TAG_KEEPALIVE2) {
            continue;
        }
        if (tag == CEPH_MSGR_TAG_KEEPALIVE2_ACK) {
            continue;
        }
        if (tag == CEPH_MSGR_TAG_ACK) {
            continue;
        }
        else if (tag == CEPH_MSGR_TAG_MSG) {
            // 收到 MSG 消息
            Message *m = 0;
            // 将消息读取到 new 到的 Message 对象
            int r = read_message(&m, auth_handler.get());

            // 先激活 writer 线程 ACK 这个消息
            cond.Signal();  // wake up writer, to ack this

            // 如果该次请求是可以延迟处理的请求，将 msg 放到 Pipe::DelayedDelivery::delay_queue, 
            // 后面通过相关模块再处理
            // 注意，一般来讲收到的消息分为三类：
            // 1. 直接可以在 reader 线程中处理，如上面的 CEPH_MSGR_TAG_ACK
            // 2. 正常处理, 需要将消息放入 DispatchQueue 中，由后端注册的消息处理，然后唤醒发送线程发送
            // 3. 延迟发送, 下面的这种消息, 由定时时间决定什么时候发送
            if (delay_thread) {
                utime_t release;
                if (rand() % 10000 < msgr->cct->_conf->ms_inject_delay_probability * 10000.0) {
                    release = m->get_recv_stamp();
                    release += msgr->cct->_conf->ms_inject_delay_max * (double)(rand() % 10000) / 10000.0;
                    lsubdout(msgr->cct, ms, 1) << "queue_received will delay until " << release << " on " << m << " " << *m << dendl;
                }
                delay_thread->queue(release, m);
            } else {
                // 正常处理的消息，放到 Pipe::DispatchQueue *in_q 中, 以下是整个消息的流程
                // DispatchQueue::enqueue()
                //     --> mqueue.enqueue() -> cond.Signal()(激活唤醒 DispatchQueue::dispatch_thread 线程)
                //         --> DispatchQueue::dispatch_thread::entry() 该线程得到唤醒
                //             --> Messenger::ms_deliver_XXX
                //                 --> 具体的 Dispatch 实例, 如 Monitor::ms_dispatch()
                //                     --> Messenger::send_message()
                //                         --> SimpleMessenger::submit_message()
                //                             --> Pipe::_send()
                //                                 --> Pipe::out_q[].push_back(m) -> cond.Signal 激活 writer 线程
                //                                     --> ::sendmsg() // 发送到 socket
                in_q->enqueue(m, m->get_priority(), conn_id);
            }
        } 

        else if (tag == CEPH_MSGR_TAG_CLOSE) {
            cond.Signal();
            break;
        }
        else {
            ldout(msgr->cct,0) << "reader bad tag " << (int)tag << dendl;
            pipe_lock.Lock();
            fault(true);
        }
    }
}
```

Pipe::accept() 做一些简单的协议检查和认证处理，之后创建 Writer() 线程:
`Pipe::start_writer()` –\> `Pipe::Writer`

``` c
int Pipe::accept()
{
    ldout(msgr->cct,10) << "accept" << dendl;
    // 检查自己和对方的协议版本等信息是否一致等操作
    // ......

    while (1) {
        // 协议检查等操作
        // ......

        /**
         * 通知注册者有新的 accept 请求过来，如果 Dispatcher 的子类有实现
         * Dispatcher::ms_handle_accept()，则会调用该方法处理
         */
        msgr->dispatch_queue.queue_accept(connection_state.get());

        // 发送 reply 和认证相关的消息
        // ......

        if (state != STATE_CLOSED) {
            /**
             * 前面的协议检查，认证等都完成之后，开始创建 Writer() 线程等待注册者
             * 处理完消息之后发送
             * 
             */
            start_writer();
        }
        ldout(msgr->cct,20) << "accept done" << dendl;

        /**
         * 如果该消息是延迟发送的消息, 且相关的发送线程没有启动，启动之
         * Pipe::maybe_start_delay_thread()
         *     --> Pipe::DelayedDelivery::entry()
         */
        maybe_start_delay_thread();
        return 0;   // success.
    }
}
```

随后 Writer 线程等待被唤醒发送回复消息

``` c
void Pipe::writer()
{
    while (state != STATE_CLOSED) {// && state != STATE_WAIT) {
        if (state != STATE_CONNECTING && state != STATE_WAIT && state != STATE_STANDBY &&
            (is_queued() || in_seq > in_seq_acked)) {

            // 对 keepalive, keepalive2, ack 包的处理
            // ......

            // 从 Pipe::out_q 中得到一个取出包准备发送
            Message *m = _get_next_outgoing();
            if (m) {
                // 对包进行一些加密处理
                m->encode(features, !msgr->cct->_conf->ms_nocrc);

                // 包头
                ceph_msg_header& header = m->get_header();
                ceph_msg_footer& footer = m->get_footer();

                // 取出要发送的二进制数据
                bufferlist blist = m->get_payload();
                blist.append(m->get_middle());
                blist.append(m->get_data());

                // 发送包: Pipe::write_message() --> Pipe::do_sendmsg --> ::sendmsg()
                ldout(msgr->cct,20) << "writer sending " << m->get_seq() << " " << m << dendl;
                int rc = write_message(header, footer, blist);
                m->put();
            }
            continue;
        }

        // 等待被 Reader 或者 Dispatcher 唤醒
        ldout(msgr->cct,20) << "writer sleeping" << dendl;
        cond.Wait(pipe_lock);
    }
}
```

### 消息的处理

Reader 线程将消息交给 `dispatch_queue` 处理，流程如下：

``` example
Pipe::reader()
    --> Pipe::in_q->enqueue()
```

``` c
void DispatchQueue::enqueue(Message *m, int priority, uint64_t id)
{
    Mutex::Locker l(lock);
    ldout(cct,20) << "queue " << m << " prio " << priority << dendl;
    add_arrival(m);
    // 将消息按优先级放入 DispatchQueue::mqueue 中
    if (priority >= CEPH_MSG_PRIO_LOW) {
        mqueue.enqueue_strict(
            id, priority, QueueItem(m));
    } else {
        mqueue.enqueue(
            id, priority, m->get_cost(), QueueItem(m));
    }
    // 唤醒 DispatchQueue::entry() 处理消息
    cond.Signal();
}

void DispatchQueue::entry()
{
    while (true) {
        while (!mqueue.empty()) {
            QueueItem qitem = mqueue.dequeue();
            Message *m = qitem.get_message();
            /**
             * 交给 Messenger::ms_deliver_dispatch() 处理，后者会找到
             * Monitor/OSD 等的 ms_deliver_dispatch() 开始对消息的逻辑处理
             * Messenger::ms_deliver_dispatch()
             *     --> Monitor::ms_dispatch()
             */
            msgr->ms_deliver_dispatch(m);
        }
        if (stop)
            break;

        // 等待被 DispatchQueue::enqueue() 唤醒
        cond.Wait(lock);
    }
    lock.Unlock();
}
```

下面简单看一下在订阅者的模块中消息是怎样被放入 `Pipe::out_q` 中的:

``` example
Messenger::ms_deliver_dispatch()
    --> Monitor::ms_dispatch()
        --> Monitor::_ms_dispatch
            --> Monitor::dispatch
                --> Monitor::handle_mon_get_map
                    --> Monitor::send_latest_monmap
                        --> SimpleMessenger::send_message()
                            --> SimpleMessenger::_send_message()
                                --> SimpleMessenger::submit_message()
                                    --> Pipe::_send()
```

``` c
bool Monitor::_ms_dispatch(Message *m)
{
    ret = dispatch(s, m, src_is_mon);

    if (s) {
        s->put();
    }

    return ret;
}

bool Monitor::dispatch(MonSession *s, Message *m, const bool src_is_mon)
{
    switch (m->get_type()) {
    case CEPH_MSG_MON_GET_MAP:
        handle_mon_get_map(static_cast<MMonGetMap*>(m));
        break;
    // ......
    default:
        ret = false;
    }
    return ret;
}

void Monitor::handle_mon_get_map(MMonGetMap *m)
{
    send_latest_monmap(m->get_connection().get());
    m->put();
}

void Monitor::send_latest_monmap(Connection *con)
{
    bufferlist bl;
    monmap->encode(bl, con->get_features());
    /**
     * SimpleMessenger::send_message()
     *     --> SimpleMessenger::_send_message()
     *         --> SimpleMessenger::submit_message()
     *             --> Pipe::_send()
     */
    messenger->send_message(new MMonMap(bl), con);
}

void Pipe::_send(Message *m)
{
    assert(pipe_lock.is_locked());
    out_q[m->get_priority()].push_back(m);
    // 唤醒 Writer 线程
    cond.Signal();
}
```

总结
====

由上面的所有分析，除了订阅者/发布者设计模式，对网络包的处理上采用的是古老的
[生产者消费者问题](https://zh.wikipedia.org/wiki/%25E7%2594%259F%25E4%25BA%25A7%25E8%2580%2585%25E6%25B6%2588%25E8%25B4%25B9%25E8%2580%2585%25E9%2597%25AE%25E9%25A2%2598)
线程模型，每次新的请求就会有创建一对收/发线程用来处理消息的接受
发送，如果有大规模的请求，线程的上下文切换会带来大量的开销，性能可能产生瓶颈。

不过在较新的 Ceph 版本中，新增加了两种新的消息模型: AsyncMessenger 和
XioMessenger 让 Ceph 消息处理得到改善.

相关的评测以后带来
