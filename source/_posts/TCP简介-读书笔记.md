---
title: TCP简介-读书笔记
date: 2017-03-19 20:00:38
tags: [TCP, linux, 读书笔记, 拥塞控制]
---

@(Network)[tcp, congestion control]
# 读书笔记-TCP简介
本文主要记录阅读[linuxtcp][1]文章，其第二章中主要介绍了TCP拥塞控制的基础和一些发展历程，这里作为整理。

个人理解，TCP的拥塞分为两部分，一部分是窗口值变化，慢启动和各种拥塞避免算法，这部分只尝试控制发往网络中的包的数量(拥塞窗口)，但是他并不处理是否丢包，是否应该重传，或者快速重传等；另一部分是TCP逐渐支持的一些机制，可以使tcp更好的进行网络状态的估计，不同的网络状态会对第一部分进行反馈，这部分属于框架级别(其实丢包重传和拥塞控制本身并没有完全相关性(比如固定丢包的链路，无线网络等)，但是由于长久以来的拥塞控制是基于丢包的，因此丢包作为网络拥塞状态判断，耦合在TCP的拥塞控制中，而由于丢包对拥塞控制的影响是毁灭性的，因此也在重传上做了较多的处理，来避免因为一些意外的丢包造成的影响，或者是提早避免因为网络拥塞导致的丢包)因此对应的tcp内核中，一部分对应的拥塞控制状态机，这里处理了所有tcp所支持的机制，如SACK,DSACK,FRTO等网络估计的基础，而拥塞避免则会根据猜测的网络状态进行合理控制。

![Alt text](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP简介/简介1.png)


[TOC]

# 0. 参考资料
\[1\] [linux_tcp](https://www.cs.helsinki.fi/group/iwtcp/papers/linuxtcp.ps.gz)
\[2\] [SACK](https://tools.ietf.org/html/rfc2018)
\[3\] [Duplicate SACK](https://tools.ietf.org/html/rfc2883)
\[4\] [Duplicate SACK ppt](https://www.eecis.udel.edu/~amer/856/sack.04f.ppt)
\[5\] [FACK](http://conferences.sigcomm.org/sigcomm/1996/papers/mathis.pdf)
\[6\] [FRTO-4138](https://tools.ietf.org/html/rfc4138)
\[7\] [FRTO-5862](https://tools.ietf.org/html/rfc5682)
\[8\] [FRTO-细节](http://blog.csdn.net/zhangskd/article/details/7446441)
\[9\] [An Enhanced Recovery Algorithm for TCP Retransmission Timeouts
](https://pdfs.semanticscholar.org/58b0/5b3f89481d2e84e136fe98d4236fea031226.pdf)

# 1. 慢启动和拥塞避免
TCP拥塞控制算法主要由发送端通过拥塞窗口(cwnd)来控制。最开始主要有两个拥塞控制的方法，通过阈值ssthresh来作为两种窗口增长的临界标志。

- 慢启动：在小于阈值时，当每一个ack到来时，慢启动算法会将拥塞窗口增加一个segment大小。当你有cwnd个包发出，收到cwnd个ack，在慢启动阶段将会增加cwnd个窗口大小，是指数增长的过程。
- 拥塞避免：当大于阈值后，拥塞避免算法会限制拥塞窗口在一个RTT内只增加一个segment大小。当你有cwnd个包发出，在这个RTT内收到cwnd个ack，在拥塞避免阶段每个ack增加(1 / 拥塞窗口大小)，窗口增加 (1 / cwnd) * cwnd = 1个大小。

# 2. RTO
重传可能被重传定时器RTO触发，RTO一旦超时，表明某个包丢失，对于TCP而言，丢包意味着网络产生拥塞，因此此时会把拥塞窗口降低到最小值(1个segment大小)。因此丢包是非常严苛的拥塞条件，一旦丢包发生，对整个传输效率会造成极大的影响。

- In addition, when RTO occurs, the sender resets the congestion window to one segment, since the RTO may indicate that the network load has changed dramatically.

- This is done because the packet loss is taken as an indication of congestion, and the sender needs to reduce its transmission rate to alleviate the network congestion.

# 3. RTT与重传二义性
由于RTO本身对于TCP性能而言非常严苛，因此RTO使用的定时器时间是否能准确反映网络实际传输情况对于TCP而言非常重要。比如链路RTT突然增加了，但是用于RTO的RTT仍然是之前的小值，这样可能会导致数据虽然没有丢包，但是交往较慢，RTO触发过于频繁，再由于丢包对于TCP的影响，会导致TCP窗口衰减剧烈。

但是TCP对于普通包和重传包使用相同的sequence number，因此当ack到来时，无法区分这个ack对应的是普通包的还是重传包的，因此此时用这个ack的时间戳选项来计算链路RTT可能会导致各种问题。

因此当前使用的RTT计算公式一般是smooth rtt计算。

# 4. 快速重传和快速恢复
由于丢包属于一类重大事故，因此TCP中总是需要尝试提前发现，在其形成重大事故(重置cwnd)前将其提前识别。因此当接收端收到乱序包时，会发送期待的包的序号ack，当发送端收到两个重复的ack时，会发现出现乱序(状态机中的Disorder)，并尝试进行重传来恢复这个包，以免RTO超时触发。当发送端收到三个重复的ack时，会进入快速恢复(状态机中的Recovery)，认为网络可能存在一定的拥塞，会降低拥塞窗口(但是不会像RTO触发以后那样激进)。两种状态下，都会恢复认为丢失或者乱序的包，直到收到非重复的ack为止。

# 5. 显式拥塞控制
由于TCP本身并无法感知到整个网络链路的质量，因此基本是基于自己的算法和丢包反馈来进行拥塞判断，本身存在一定的局限性。

后来的一些路由器在处理数据包，可能可以感知到网络是否真正拥塞，当路由器感知到拥塞时，会通过设置TCP标志位的ECE给TCP，TCP发送端收到带有ECE的ack时会进入拥塞状态(状态机中的CWR)，同时发送一个携带标志位CWR的包给接收端，表示自己当前正在衰减拥塞窗口。

# 6. Selective acknowledgements
由于进入乱序或者快速重传后，在一个RTT之内只能处理一个异常包(不会发送新的包直到新的ack到来，但是当丢包是不连续的若干个，恢复完后仍然会进入对应状态，单独处理下一个乱序丢包，如发送端发送1-6，接收端先收到1，3，6，这时候发送先恢复包2，恢复完包2收到ack发现乱序，请求包4，发送端再恢复4，恢复后发现仍然乱序，请求包5，再次单独恢复包5)，因此在恢复阶段严重影响吞吐量。

![Alt text](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP简介/tcp_recovery_multi_loss.png)

SACK并没有打破原有的ack机制，只是在其ack机制上，在TCP option字段中附加了额外信息。<sup>[4]</sup>

例子如下图：
![Alt text](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP简介/sack.png)

- Multiple packet losses from a window of data can have a catastrophic effect on TCP throughput.

- SACK does not change the meaning of ACK field.

注: SACK附加在TCP option字段中，option字段最多只有40字节，因此SACK最多包含四个区间。

- A SACK option that specifies n blocks will have a length of 8*n+2 bytes, so the 40 bytes available for TCP options can specify a   maximum of 4 blocks.

注2: SACK必须接收和发送端都支持才可以正常使用。

# 7. Duplicate-SACK
SACK在rfc 2018定义的时候，并没有声明其收到两个相同的包以后的处理。D-SACK会使接收端发送重复块的信息给发送端。

简单流程如下：
当DSACK激活时，最后一个ACK中的SACK第一个区域为重复区域，不同于普通SACK，它是已经收到的区域的一个子区间，每个重复块只会上报一次。

![Alt text](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP简介/dsack.png)


# 8. Forward Acknowledgements
Forward Acknowledgement是基于SACK的相关的拥塞控制算法。

当拥塞发生时，此时已经发生丢包，这时候会引入新的变量fackets\_out来统计SACK中的数据量，并根据当前已经发送的数据量una和重传数据量retran来估算本条连接实际在网络中的包数量(总包数 - 已经ACK的数量 + 重传包)。并根据这个数值和拥塞窗口进行比较，如果当前网络中的包数量小于拥塞窗口，说明仍然可以往网络中发送部分数量的包。

这种算法本质上是对网络中的包数量进行更精确的估计，结合DSACK，可以更精准的进行判断，可以在恢复阶段依旧保持一定的速率，在处理乱序包的时候可以比传统TCP更加激进。

# 9. FRTO

当网络链路存在一些突发的特殊场景时，可能会触发超时定时器，由于TCP对于丢包的处理异常严格，可能会造成链路质量下降。

可能的一些场景：
- 对于移动信号的跨域处理，可能会造成突发的延迟。
- 对于低带宽链路，偶发的竞争可能也会造成整个RTT的增加。
- 稳定的链路上可能也有一些原因导致某些包及其重传老是失败。

- First, some mobile networking technologies involve sudden delay spikes on transmission because of actions taken during a hand-off.  
- Second, given a low-bandwidth link or some other change in available bandwidth, arrival of competing traffic (possibly with higher priority) can cause a sudden increase of round-trip time. This may trigger a spurious retransmission timeout. 
- A persistently reliable link layer can also cause a sudden delay when a data frame and several retransmissions of it are lost for some reason.

可能造成的一些影响：

- 虚假超时后的慢启动可能会导致向已经产生拥塞的网络中注入更多的数据包，会严重影响实际网络的拥塞状态。
- 当虚假超时触发后，可能造成虚假重传，当过多虚假重传发生后，对应的ack回来时可能会触发虚假的快速恢复，如下图，第二次虚假的快速重传是由于第一次虚假RTO超时导致重传发送了重复包导致。

- However, if the RTO occurs spuriously and there still are segments outstanding in the network, a false slow start is harmful for the potentially congested network as it injects extra segments to the network at increasing rate.

![Alt text](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP简介/frto.png)

FRTO会在RTO超时后，不会类似传统TCP的超时机制，会额外根据后续两个ACK与当前未确认的最小包进行比较，根据这个结果判断当前RTO是否在安全范围内。  
如果收到的是未确认的包之后的包，则可能是因为网络原因导致的延迟，可以进入恢复状态。
如果收到的是重复的ack，则认为这个包确实已经丢失，进入丢包状态。

# 10. 概览图

简单用图画出之间衍生的关系。  

![Alt text](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP简介/概览.png)


[1]: https://www.cs.helsinki.fi/group/iwtcp/papers/linuxtcp.ps.gz
[2]: https://tools.ietf.org/html/rfc2018
[3]: https://tools.ietf.org/html/rfc2883
[4]: https://www.eecis.udel.edu/~amer/856/sack.04f.ppt
[5]: http://conferences.sigcomm.org/sigcomm/1996/papers/mathis.pdf
[6]: https://tools.ietf.org/html/rfc4138
[7]: https://tools.ietf.org/html/rfc5682
[8]: http://blog.csdn.net/zhangskd/article/details/7446441
[9]: https://pdfs.semanticscholar.org/58b0/5b3f89481d2e84e136fe98d4236fea031226.pdf
