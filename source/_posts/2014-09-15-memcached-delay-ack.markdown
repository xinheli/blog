---
layout: post
title: "memcached 200ms延迟分析"
date: 2014-09-15 16:08:45 +0800
comments: true
categories: memcache tcp
---
##背景
线上memcached的使用，在Get value较大（>1400 bytes）的时候，常常出现200ms的延迟。

memcache server部署在CentOS6.5, 版本是1.4.4, client端是windows server 2008 R2

##分析
不是每次请求都会延迟，而一旦发现延迟，必然是200ms，所以很容易猜测是[经典](http://www.stuartcheshire.org/papers/NagleDelayedAck/)问题: TCP 协议中的 [Nagle‘s Algorithm](http://en.wikipedia.org/wiki/Nagle's_algorithm) 和 [TCP Delayed Acknoledgement](http://en.wikipedia.org/wiki/Nagle's_algorithm) 共同起作 用所造成的结果。

下面简单介绍一下Nagle's Algorithm和 TCP Delayed Ack
<!--more-->
####Nagle's Algorithm

Nagle’s Algorithm 是为了提高带宽利用率设计的算法，其做法是合并小的TCP 包为一个，避免了过多的小报文的 TCP 头所浪费的带宽。如果开启了这个算法 （默认），则协议栈会累积数据直到以下两个条件之一满足的时候才真正发送出 去：

1,积累的数据量到达最大的 TCP Segment Size

2, 收到了一个 Ack

#### TCP Delayed Ack

TCP Delayed Acknoledgement 也是为了类似的目的被设计出来的，它的作用就 是延迟 Ack 包的发送，使得协议栈有机会合并多个 Ack，提高网络性能。

如果一个 TCP 连接的一端启用了 Nagle‘s Algorithm，而另一端启用了 TCP Delayed Ack，而发送的数据包又比较小，则可能会出现这样的情况：发送端在等 待接收端对上一个packet 的 Ack 才发送当前的 packet，而接收端则正好延迟了 此 Ack 的发送，那么这个正要被发送的 packet 就会同样被延迟。当然 Delayed Ack 是有个超时机制的，而默认的超时正好就是 40ms (Linux) 200ms(Windows)。

现代的 TCP/IP 协议栈实现，默认几乎都启用了这两个功能，按照上面的说法，当协议报文很小的时候，岂不每次都会触发这个延迟问题？事实不 是那样的。仅当协议的交互是发送端连续发送两个 packet，然后立刻 read 的 时候才会出现问题。这就是经典的Write Write Read的错误。

#### Write-Write-Read Issue
具体Nagle’s Algorithm如下：
``` c
if there is new data to send
  if the window size >= MSS and available data is >= MSS
    send complete MSS segment now
  else
    if there is unconfirmed data still in the pipe
      enqueue data in the buffer until an acknowledge is received
    else
      send data immediately
    end if
  end if
end if
```
可以看到，当待发送的数据比 MSS 小的时候（外层的 else 分支），还要再判断 时候还有未确认的数据。只有当管道里还有未确认数据的时候才会进入缓冲区， 等待 Ack。

所以发送端发送的第一个 write 是不会被缓冲起来，而是立刻发送的（进入内层 的else 分支），这时接收端收到对应的数据，但它还期待更多数据才进行处理， 所以不会往回发送数据，因此也没机会把 Ack 给带回去，根据Delayed Ack 机制， 这个 Ack 会被 Hold 住。这时发送端发送第二个包，而队列里还有未确认的数据 包，所以进入了内层 if 的 then 分支，这个 packet 会被缓冲起来。此时，发 送端在等待接收端的 Ack；接收端则在 Delay 这个 Ack，所以都在等待，直到接 收端 Deplayed Ack 超时，此 Ack 被发送回去，发送端缓冲的这个 packet 才会被真正送到接收端，从而继续下去。

Write-Write-Read问题很经典，一般网路如果采用了这样的模式，都会有这方面的隐患，一个例子就是python 2.6里的http库的实现就是采用了send header+send body+recv的socket调用，就会触发这个问题，在python 2.7里修复了这个bug，只使用一次socket.send 

如果采用的是Write-Read-Write-Read 就不会有这个问题。因为第一个 write 不会被缓冲，会立刻到达接收端，此时接收端应该已经得到所有需要的数据以进行 下一步处理。接收端此时处理完后发送结果，同时也就可以把上一个packet 的 Ack 可以和数据一起发送回去，不需要 delay，从而不会导致任何问题。

##深入分析
前面讨论了问题产生的条件：

1，错误的使用了Write-Write-Read模式

2，通讯双方都使用了 Nagle‘s Algorithm 和 TCP Delayed Acknoledgement，只要有一方禁止掉这个算法就好。

只要有一个解决掉，就不会碰到延迟的问题。可惜memcached两方面都有BUG，具体分析如下：

####Write-Write-Read模式的BUG
对于1，实际我们并没有采用Write-Write-Read的模式，而是一个简单的memcache的Get命令。应该是server拿到数据，并没有一次发过来，而是采用了write了2次。为什么一个get命令，server会write两次？查看[memcached代码](https://github.com/memcached/memcached/blob/master/memcached.c) 可以看到：
``` c
static int add_iov(conn *c, const void *buf, int len) {
    struct msghdr *m;
    int leftover;
    bool limit_to_mtu;

    assert(c != NULL);

    do {
        m = &c->msglist[c->msgused - 1];

        /*
         * Limit UDP packets, and the first payloads of TCP replies, to
         * UDP_MAX_PAYLOAD_SIZE bytes.
         */
        limit_to_mtu = IS_UDP(c->transport) || (1 == c->msgused);

        /* We may need to start a new msghdr if this one is full. */
        if (m->msg_iovlen == IOV_MAX ||
            (limit_to_mtu && c->msgbytes >= UDP_MAX_PAYLOAD_SIZE)) {
            add_msghdr(c);
            m = &c->msglist[c->msgused - 1];
        }

        if (ensure_iov_space(c) != 0)
            return -1;

        /* If the fragment is too big to fit in the datagram, split it up */
        if (limit_to_mtu && len + c->msgbytes > UDP_MAX_PAYLOAD_SIZE) {
            leftover = len + c->msgbytes - UDP_MAX_PAYLOAD_SIZE;
            len -= leftover;
        } else {
            leftover = 0;
        }

        m = &c->msglist[c->msgused - 1];
        m->msg_iov[m->msg_iovlen].iov_base = (void *)buf;
        m->msg_iov[m->msg_iovlen].iov_len = len;

        c->msgbytes += len;
        c->iovused++;
        m->msg_iovlen++;

        buf = ((char *)buf) + len;
        len = leftover;
    } while (leftover > 0);

    return 0;
}
```
如果是UDP或者是TCP first payload,  并且需要write的数据 Size大于UDP_MAX_PAYLOAD_SIZE, memcached会做一个拆包的动作，会write两次，而不是一次write。 我认为这是一个BUG。TCP first payload 完全没有必要做这个拆分操作。（UDP因为没有ACK,需要做限制，避免堵塞网络）。

####Nagle‘s Algorithm 和 TCP Delayed Acknoledgement 的 Bug
本案例，是Server发给client导致了延迟，所以需要禁掉Server的Nagle算法，或者Client的Delayed Ack。

Client：对于Linux，可以设置`TCP_QUICKACK`，具体参考[这里](http://blog.163.com/xychenbaihu@yeah/blog/static/132229655201231214038740/).特别注意的是TCP_QUICKACK需要在每次recv都需要设定，它不是全局的。但是这个参数Windows下不支持。只能在[注册表设置](http://support.microsoft.com/kb/328890) 这样修改比较暴力，是全局，所有网络交互都去掉了Delayed Ack。

Server: grep memcached 的Changed log,可以看到
``` c
2003-08-10
554		* disable Nagel's Algorithm (TCP_NODELAY) for better performance (avva) 
error = setsockopt(sfd, IPPROTO_TCP, TCP_NODELAY, (void *)&flags, sizeof(flags)); 
```
这样看memcached在listen socket里设置了TCP NODELAY，但是经验证，在Centos上这个设置是不会继承到`accept`之后的真正和用户通信的socket的。 所以后续accept都需要在设置一下TCP_NODELAY. 这也是一个BUG

##解决
快速解决：memcache client 如果是linux环境，可以设置TCP_QUICKACK。我们恰好是windows环境，只能设置注册表来解决（需要重启）

彻底解决：修改memcached的源码，每次`accpet`都加上TCP NODELAY。
