---
title: 服务器cpu占用过高问题
date: 2016-12-05 15:35:50
tags: [tcp, cpu]  
categories: 总结
---

# 排查java程序
之前公司网页项目有两台服务器，出现过cpu负载过高的情况，阿里云频繁报警。一开始以为是程序代码问题，通过top命令观察，java进程的负载比较平均都是比较低的，拿了一个进程按以下步骤查看：

1.  top -H -p pid   找到cpu负载比较高的线程，将其id转换成16进制。
2. 执行jstack -l pid > /tmp/out.txt, 获取dump文件，查询id所在线程，并不是gc线程，也不是程序线程，而是tomcat线程，这种线程比较多，怀疑是大量这种线程导致cpu占用过高了。 之后在一台正常的服务器上比较了下，tomcat线程数差不多，故排除是程序代码问题，因此极有可能是服务器硬件或者网络出现了异常。


# 排查网络情况：
执行 `netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}' ` 发现大量的time_wait 端口， 可能是因为服务器频繁释放链接，导致新的链接创建不了。
time_wait 阶段是指主动关闭链接的一方在四次握手最后一个阶段，还会等待2个报文最大生存时间（msl）默认是120s，才会关闭链接.

# 修改系统的/etc/sysctl.conf配置来减少TIME_WAIT的tcp连接：
```
vi /etc/sysctl.conf

net.ipv4.tcp_syncookies = 1  （表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭）
net.ipv4.tcp_tw_reuse = 1  （表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭）
net.ipv4.tcp_tw_recycle = 1   （表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。）
net.ipv4.tcp_fin_timeout = 30 （修改系統默认的TIMEOUT时间）

```

执行/sbin/sysctl -p让参数生效. 等待一段时间查看网络情况，发现time_wait状态变少，查看cpu占用情况，也恢复正常。
猜测linux每次找到一个随机端口，还是需要遍历一遍bound ports的吧，这必然需要一些CPU时间，会导致cpu负载变高。


# 详解tcp
tcp属于osi七层模型中的第四层传输层， ip属于第三层网络层，arp属于第二层数据链路层。第二层上的数据叫做frame，第三层叫做package，第四层叫做segment。 

# tcp  三次握手, 四次握手

![TCP头格式](http://yangege.b0.upaiyun.com/product/d7812d64205.jpg)

需要注意的几点
* sequence number 报文的序号，解决乱序的问题
* acknowledgement Number 解决可靠传输的问题，不丢包
* sliding window 解决用于流量控制的
* tcp flag 操作tcp的状态机，如ACK，PUSH，SYN，FIN, RESET

![状态机变化](http://yangege.b0.upaiyun.com/product/2e51096e3886b.png)

* 三次握手连接：

    客户端 CLOSED -> SYN_SEND -> ESTABLISHED 

    服务端 CLOSED -> SYN_RECEIVED -> ESTABLISHED

* 四次握手断开： 

    客户端 ESTABLISHED -> FIN_CLOSED1 -> FINCLOSED2 -> TIME_WAIT -> CLOSED 

    服务端 ESTABLISHED -> SYN_RECEIVED -> CLOSE_WAIT -> ACK_WAIT -> CLOSED

# 为什么会有连接需要三次握手和断开需要四次握手？

* 三次握手：双方同步sequence number的初始值，所以叫做syn （synchronize sequence number）， 以保证数据的传输顺序的一致。

* 四次握手：可以看做两次断开过程。 
 
需要主要的几个点:

  1. 客户端发出SYN后挂掉。试想一下，如果server端接到了clien发的SYN后回了SYN-ACK后client掉线了，server端没有收到client回来的ACK，那么，这个连接处于一个中间状态，即没成功，也没失败。于是，server端如果在一定时间内没有收到的TCP会重发SYN-ACK。在Linux下，默认重试次数为5次，重试的间隔时间从1s开始每次都翻售，5次的重试时间间隔为1s, 2s, 4s, 8s, 16s，总共31s，第5次发出后还要等32s都知道第5次也超时了，所以，总共需要 1s + 2s + 4s+ 8s+ 16s + 32s = 2^6 -1 = 63s，TCP才会把断开这个连接。
  
  2. syn flood 攻击
     恶意攻击者可以发出SYN后马上挂掉，会导致大量的端口资源，会导致syn连接队列耗尽。linux提供了一个tcp_syncookies参数应对，当syn队列满了之后，linux会根据源，目标端口构，时间戳打造出一个特别的Sequence Number发回去（又叫cookie），如果是攻击者则不会有响应，如果是正常连接，则会把这个 SYN Cookie发回来，然后服务端可以通过cookie建连接（即使你不在SYN队列中）,对于正常的请求，你应该调整三个TCP参数可供你选择，第一个是：tcp_synack_retries 可以用他来减少重试次数；第二个是：tcp_max_syn_backlog，可以增大SYN连接数；第三个是：tcp_abort_on_overflow 处理不过来干脆就直接拒绝连接了。

  3. time_wait 
     我们注意到，在TCP的状态图中，从TIME_WAIT状态到CLOSED状态，有一个超时设置，这个超时设置是 2*MSL（RFC793定义了MSL为2分钟，Linux设置成了30s）为什么要这有TIME_WAIT？为什么不直接给转成CLOSED状态呢？主要有两个原因：1）TIME_WAIT确保有足够的时间让对端收到了ACK，如果被动关闭的那方没有收到Ack，就会触发被动端重发Fin，一来一去正好2个MSL，2）有足够的时间让这个连接不会跟后面的连接混在一起（你要知道，有些自做主张的路由器会缓存IP数据包，如果连接被重用了，那么这些延迟收到的包就有可能会跟新连接混在一起）。


  4. time_wait 数量太多
     可以配置两个参数tcp_tw_reuse、tcp_tw_recycle这两个参数默认值都是被关闭的，后者recyle比前者resue更为激进，resue要温柔一些。另外，如果使用tcp_tw_reuse，必需设置tcp_timestamps=1，否则无效.
* 关于tcp_tw_reuse。官方文档上说tcp_tw_reuse 加上tcp_timestamps（又叫PAWS, for Protection Against Wrapped Sequence Numbers）可以保证协议的角度上的安全，但是你需要tcp_timestamps在两边都被打开（你可以读一下tcp_twsk_unique的源码 ）。我个人估计还是有一些场景会有问题。

* 关于tcp_tw_recycle。如果是tcp_tw_recycle被打开了话，会假设对端开启了tcp_timestamps，然后会去比较时间戳，如果时间戳变大了，就可以重用。但是，如果对端是一个NAT网络的话（如：一个公司只用一个IP出公网）或是对端的IP被另一台重用了，这个事就复杂了。建链接的SYN可能就被直接丢掉了（你可能会看到connection time out的错误）（如果你想观摩一下Linux的内核代码，请参看源码 tcp_timewait_state_process）。

*  关于tcp_max_tw_buckets。这个是控制并发的TIME_WAIT的数量，默认值是180000，如果超限，那么，系统会把多的给destory掉，然后在日志里打一个警告（如：time wait bucket table overflow），官网文档说这个参数是用来对抗DDoS攻击的。也说的默认值180000并不小。这个还是需要根据实际情况考虑。


# TCP重传机制

TCP要保证所有的数据包都可以到达，所以，必需要有重传机制。

注意，接收端给发送端的Ack确认只会确认最后一个连续的包，比如，发送端发了1,2,3,4,5一共五份数据，接收端收到了1，2于是回ack 3，然后收到了4（注意此时3没收到），此时的TCP会怎么办？我们要知道，因为正如前面所说的，SeqNum和Ack是以字节数为单位，所以ack的时候，不能跳着确认，只能确认最大的连续收到的包，不然，发送端就以为之前的都收到了。

## 超时重传机制

一种是不回ack，死等3，当发送方发现收不到3的ack超时后，会重传3。一旦接收方收到3后，会ack 回 4——意味着3和4都收到了。

但是，这种方式会有比较严重的问题，那就是因为要死等3，所以会导致4和5即便已经收到了，而发送方也完全不知道发生了什么事，因为没有收到Ack，所以，发送方可能会悲观地认为也丢了，所以有可能也会导致4和5的重传。

对此有两种选择：

* 一种是仅重传timeout的包。也就是第3份数据。
* 另一种是重传timeout后所有的数据，也就是第3，4，5这三份数据。
这两种方式有好也有不好。第一种会节省带宽，但是慢，第二种会快一点，但是会浪费带宽，也可能会有无用功。但总体来说都不好。因为都在等timeout，timeout可能会很长（在下篇会说TCP是怎么动态地计算出timeout的）

## 快速重传机制

于是，TCP引入了一种叫Fast Retransmit 的算法，不以时间驱动，而以数据驱动重传。也就是说，如果，包没有连续到达，就ack最后那个可能被丢了的包，如果发送方连续收到3次相同的ack，就重传。Fast Retransmit的好处是不用等timeout了再重传。

比如：如果发送方发出了1，2，3，4，5份数据，第一份先到送了，于是就ack回2，结果2因为某些原因没收到，3到达了，于是还是ack回2，后面的4和5都到了，但是还是ack回2，因为2还是没有收到，于是发送端收到了三个ack=2的确认，知道了2还没有到，于是就马上重转2。然后，接收端收到了2，此时因为3，4，5都收到了，于是ack回6。示意图如下：

Fast Retransmit只解决了一个问题，就是timeout的问题，它依然面临一个艰难的选择，就是，是重传之前的一个还是重传所有的问题。对于上面的示例来说，是重传#2呢还是重传#2，#3，#4，#5呢？因为发送端并不清楚这连续的3个ack(2)是谁传回来的？也许发送端发了20份数据，是#6，#10，#20传来的呢。这样，发送端很有可能要重传从2到20的这堆数据（这就是某些TCP的实际的实现）。可见，这是一把双刃剑。


[参考](http://coolshell.cn/articles/11564.html)



返回的这个结构里再加一个tname
用户点了导航条的时候, 本地可以记一条记录， 
点击结构：
{
    target: tname (str)     
    action: 2（str）
    time: 记录时间 （str）
    gpm：str （来源信息导航条暂时没有啊吧）
}

到了一定记录数或者时间可以发送过来

/data/track 这个接口 params中是 trackList：[点击结构]