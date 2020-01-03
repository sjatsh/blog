---
layout: post
title: "关于TCP那些事（一）"
description: ""
tag: "tcp"
date: "2018-08-19 23:20:00 +0800"
categories: "net"
---

### TCP头格式

![](https://olef5l6y5.qnssl.com/tcp_head.png)
![](https://olef5l6y5.qnssl.com/tcp_options.png)

需要注意的几点：

- TCP包没有IP地址，那是IP层的事。只有源端口和目标端口。
- 一个TCP连接需要四个元组来表示是同一个连接（src_ip, src_port, dst_ip, dst_port）准确说是五元组，还有一个是协议。但因为这里只是说TCP协议，所以，这里我只说四元组。
- **Sequence Number**是包的序号，用来解决网络包乱序（reordering）问题。
- **Acknowledgement Number**就是ACK——用于确认收到，用来解决不丢包的问题。
- **Window又叫Advertised-Window**，也就是著名的滑动窗口（Sliding Window），**用于解决流控**。
- **TCP Flag**，包类型，**用于控制TCP状态机**。

<!--more-->

### TCP的状态机

TCP三次握手与四次挥手

![](https://olef5l6y5.qnssl.com/tcp_open_close.png)

为什么建立连接需要3次握手，断开需要4次挥手：

- **对于3次握手**，主要是初始化Sequence Number，双方需要互通双方的初始化Sequence Number以及MSS（最大报文长度）、WSOPT（窗口比例因子）等等，这就是叫SYN而不是CONNECT
原因，并且三次握手正好双方都对对方的报文进行了一次收发操作，说明双方都处于可用状态。四次或更多次其实也可以，只是没有必要因为接下来就要开始数据发送，只要有数据发送就说明连接
是正常的
- **四次挥手**，其实仔细看是2次挥手，因为TCP是全双工的，发送方和接受方都需要Fin和Ack。因为有一方是被动的，所以看上去就成了所谓的4次挥手，如果两边同时断开连接都会先进入
CLOSING状态，然后到TIME_WAIT。
- **连接时的SYN超时**，如果服务端发送SYN-ACK后Client掉线，那么服务端就会处于一个中间状态，连接既没有成功也没有失败。服务端会因为超时重传，Linux下默认重试次数为5次时间
间隔从1s开始每次翻倍，所以需要2^6-1=63s才会断开这个连接。
- **SYN Flood攻击**，可以通过只发送SYN就下线服务端就需要等待63s才会断开连接，攻击者可以很快把服务端SYN连接耗尽。
    - **tcp_syncookies**的参数来应对这个事——当SYN队列满了后，TCP会通过源地址端口、目标地址端口和时间戳打造出一个特别的Sequence Number发回去（又叫cookie），如果是
    攻击者则不会有响应，如果是正常连接，则会把这个 SYN Cookie发回来，然后服务端可以通过cookie建连接（即使你不在SYN队列中）。请注意，请先千万别用tcp_syncookies来处理
    正常的大负载的连接的情况。因为，synccookies是妥协版的TCP协议，并不严谨。
    - **tcp_synack_retries**，可以通过这个参数设置SYNACK重试次数。
    - **tcp_max_syn_backlog**，通过这个参数增加SYN连接数。
    - **tcp_abort_on_overflow**，设置这个参数处理不过来就直接拒绝。
- **ISN的初始化**，如果连接建好后始终用1来做ISN，如果client发了30个segment过去，但是网络断了，于是 client重连，又用了1做ISN，但是之前连接的那些包到了，于是就被当成了
新连接的包，此时，client的Sequence Number 可能是3，而Server端认为client端的这个号是30了。<a href="http://tools.ietf.org/html/rfc793" target="_blank">RFC793</a>
中说了ISN和一个假时钟绑定，每4微妙加一直到超过2^32一个ISN周期大约需要4.55小时，一个包在网络上存活时间不会超过MSL，所以只要MSL小于4.55小时就不会重用ISN。
- **MSL和TIME_WAIT**,为什么TIME_WAIT是2MSL？
    - 确保有足够的时间让对方收到ACK，如果被动关闭连接的一方没有收到ACK就会触发被动方重发FIN，一来一回最大正好是2MSL
    - 有足够的时间让这个连接不和后面新建立的连接混在一次，如果连接被重用延迟收到的包可能会和新连接混在一起。
- **TIME_WAIT过多**，在大量并发短连接的情况下就可能出现大量TIME_WAIT，修改**tcp_tw_reuse**与**tcp_tw_recycle**,这两个参数默认是关闭的。
    - **tcp_tw_reuse**加上**tcp_timestamps**可以保证协议角度上的安全，**tcp_tw_reuse**只对客户端有效对服务端无效，**tcp_timestamps**需要双方都打开，否则默写场景
    可能还是会有问题。
    - **tcp_tw_recycle**默认认为对端已经开启**tcp_timestamps**，比较时间戳变大了就可以重用。但是如果对端是NAT网或者IP被另一台电脑重用，通常服务器也是在负载后面负载会
    把 timestamp 都给清空，这个时候情况就复杂了。
- **tcp_max_tw_buckets**，控制并发TIME_WAIT数默认是180000,默认超出限制会把多的destory掉然后打`time wait bucket table overflow`，可以通过这个参数抗DDos攻击，可
以根据实际情况调整大小。  

**tcp_tw_reuse**和**tcp_tw_recycle**来解决TIME_WAIT的问题是非常非常危险的，因为这两个参数违反了TCP协议
