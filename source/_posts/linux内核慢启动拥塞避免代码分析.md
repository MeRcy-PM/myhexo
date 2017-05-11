---
title: linux内核慢启动拥塞避免代码分析
date: 2017-03-27 10:27:29
tags: [TCP, linux, 拥塞控制]
---

@(Network)[tcp, 慢启动]
# TCP拥塞控制两个速率增长阶段分析
# 0. 参考文档
\[1\] [rfc-5681](https://tools.ietf.org/html/rfc5681)
\[2\] [tcp-abc-rfc](http://blog.csdn.net/dog250/article/details/51348568)
\[3\] [rfc-3465](https://tools.ietf.org/html/rfc3465)
\[4\] [rfc-3742](https://tools.ietf.org/html/rfc3742)

# 1. 拥塞控制个人理解

## 1.1 慢启动与拥塞避免
慢启动和拥塞避免，主要是用于拥塞控制中拥塞窗口增长的维护。

根据阈值，拥塞控制其实分为两部分，小于阈值的慢启动阶段，大于阈值进入拥塞避免阶段。

慢启动作为拥塞控制的一部分，我觉得其名字取的比较具有混淆性。个人理解的慢启动分为两种，一种是拥塞窗口小于阈值时候正常的一个指数增长的过程，这个过程中的拥塞窗口不会重置，会持续增长，还有一种是与快速恢复对应的慢启动重新启动，这种时候会将拥塞窗口重置为1，并重新开始指数增长。这么理解的原因如下：

在[文档][1]中描述快速恢复时，当收到三个重复ack时候，这时候可能并不是实际丢包，可能是因为链路问题，较晚到达接收端。

- 在BSD 4.3之前会进入慢启动阶段，但是理论上慢启动一般是指数上升的过程，反而是拥塞避免阶段线性上升速度较慢，且拥塞避免会更新当前的拥塞窗口和阈值，会出现小范围衰减。
- 假如tcp认为当前包丢失，会很严格的重置拥塞窗口(具体代码如rto触发tcp\_enter\_loss)，这时候速率曲线不会只是单纯减低到某个值，而是会降低到零点。

## 1.2 快速恢复和快速重传
因此，个人理解，老版本上收到三个重复ack认为丢包，进入丢包处理，重置了拥塞窗口，在非重复ack到来后，拥塞窗口仍然需要从零开始指数上升，而对于快速恢复而言，其只进入拥塞避免阶段，拥塞窗口只是进行一定修正，在非重复ack到来后，仍然能根据阈值来决定是否执行非重启的慢启动，这时候恢复速度相较于严格的丢包处理快了不少。

由于tcp对于丢包的容忍极低，一旦丢包发生，就会进入严格的拥塞处理，而RTO是丢包主要判断依据，因此快速重传也是针对tcp对于丢包容忍度低的一个修正，避免进入RTO，直接影响传输性能。

# 2. 拥塞控制代码分析
本章主要基于reno的拥塞控制。下文中的代码均基于linux kernel 2.6.32版本，直到linux kernel 4.9-rc8之前的版本，tcp整体并没有太大变化。本文不分析frto相关内容。

## 2.1 调用链
由于文档描述上是直接给出一个计算过程，如慢启动阶段的指数上升，和代码直观上看略有不同，因此这里需要先缕清楚整个的调用链，能更好的描述整个拥塞控制的过程。

tcp的拥塞主要是基于定时器(RTO)和ack的，因此主要处理函数都以tcp\_ack为起点。这里不分析整个tcp\_ack函数，仅分析常规调用链。

整体入口如下：

```cpp
// 当ack时一个可疑的ack，如sack，或者路由发送的显示拥塞控制，或者当前拥塞状态不是正常状态时。
if (tcp_ack_is_dubious(sk, flag)) {
	/* Advance CWND, if state allows this. */
	if ((flag & FLAG_DATA_ACKED) && !frto_cwnd &&
	    tcp_may_raise_cwnd(sk, flag))
	    // 当窗口仍然满足可以增长的条件时，进入拥塞控制，
	    // 这是一个钩子函数，具体实现由具体拥塞控制算法来实现，
	    // 对于reno而言可能是慢启动，可能是拥塞避免。
		tcp_cong_avoid(sk, ack, prior_in_flight);
	// 处理拥塞状态机，暂时不展开
	tcp_fastretrans_alert(sk, prior_packets - tp->packets_out,
			      flag);
} else {
    // 当这个ack是一个正常的数据确认包，进入拥塞控制
	if ((flag & FLAG_DATA_ACKED) && !frto_cwnd)
		tcp_cong_avoid(sk, ack, prior_in_flight);
}
```

## 2.2 tcp reno的拥塞控制
tcp reno注册到拥塞控制框架中的是tcp\_reno\_cong\_avoid函数。

其代码较为简单，只是其中多了一部分tcp-abc的拥塞避免算法，其慢启动实现在tcp\_slow\_start中，可以参考\[[rfc-3465][3]\]\[[tcp_abc][2]\]。大体是用已经确认的byte大小来作为拥塞控制的计算，在慢启动阶段会更加激进，但是可能会带来更大的burst。

```cpp
/*
 * TCP Reno congestion control
 * This is special case used for fallback as well.
 */
/* This is Jacobson's slow start and congestion avoidance.
 * SIGCOMM '88, p. 328.
 */
void tcp_reno_cong_avoid(struct sock *sk, u32 ack, u32 in_flight)
{
	struct tcp_sock *tp = tcp_sk(sk);

	if (!tcp_is_cwnd_limited(sk, in_flight))
		return;

	/* In "safe" area, increase. */
	// 小于阈值会进入慢启动环节，不重置窗口的慢启动。
	if (tp->snd_cwnd <= tp->snd_ssthresh)
		tcp_slow_start(tp);

	/* In dangerous area, increase slowly. */
	else if (sysctl_tcp_abc) {
		/* RFC3465: Appropriate Byte Count
		 * increase once for each full cwnd acked
		 */
		// RFC3465的拥塞避免算法，使用bytes_acked来作为修改拥塞窗口的判断条件
		if (tp->bytes_acked >= tp->snd_cwnd*tp->mss_cache) {
			tp->bytes_acked -= tp->snd_cwnd*tp->mss_cache;
			if (tp->snd_cwnd < tp->snd_cwnd_clamp)
				tp->snd_cwnd++;
		}
	} else {
	    // 拥塞避免
		tcp_cong_avoid_ai(tp, tp->snd_cwnd);
	}
}
```

## 2.3 慢启动
慢启动里面额外涉及两篇rfc，[rfc-3742][4]和[tcp_abc][3]。

其中snd\_cwnd\_cnt为线性增长器，只有当线性增长器大于一个窗口大小时，其才会将发送窗口增加，即其单位为1/snd\_cwnd，后续还会在拥塞避免代码中见到。

刚开始看代码时对下面那个循环并不是很理解，不理解为什么++是指数增长，直到放到整个调用栈上看，其具体流程如代码注释中所写，为指数增长的过程。

```cpp
/*
 * Slow start is used when congestion window is less than slow start
 * threshold. This version implements the basic RFC2581 version
 * and optionally supports:
 * 	RFC3742 Limited Slow Start  	  - growth limited to max_ssthresh
 *	RFC3465 Appropriate Byte Counting - growth limited by bytes acknowledged
 */
void tcp_slow_start(struct tcp_sock *tp)
{
	int cnt; /* increase in packets */

	/* RFC3465: ABC Slow start
	 * Increase only after a full MSS of bytes is acked
	 *
	 * TCP sender SHOULD increase cwnd by the number of
	 * previously unacknowledged bytes ACKed by each incoming
	 * acknowledgment, provided the increase is not more than L
	 */
    // 不满足tcp abc的窗口增加条件，此时确认的字节数小于mss_cache。
	if (sysctl_tcp_abc && tp->bytes_acked < tp->mss_cache)
		return;

    // RFC 3742，限制慢启动在一个RTT内的burst。
	if (sysctl_tcp_max_ssthresh > 0 && tp->snd_cwnd > sysctl_tcp_max_ssthresh)
		cnt = sysctl_tcp_max_ssthresh >> 1;	/* limited slow start */
	else
	// 加上一个窗口大小，在没有abc的情况，保证在最底下的循环中拥塞窗口大小至少增加1.
		cnt = tp->snd_cwnd;			/* exponential increase */

	/* RFC3465: ABC
	 * We MAY increase by 2 if discovered delayed ack
	 */
	// tcp-abc，慢启动阶段更激进的burst。
	if (sysctl_tcp_abc > 1 && tp->bytes_acked >= 2*tp->mss_cache)
		cnt <<= 1;
	tp->bytes_acked = 0;

    // 更新snd_cwnd_cnt(窗口线性增长器)
    tp->snd_cwnd_cnt += cnt;
    // 线性增长器是窗口的多少倍，窗口就增加多少。
    // 注意：这里的标准场景下的线性增长，每次也只增长1个窗口大小，
    // 但是其仍然是指数增长，因此每个窗口发出去的数据对应一个ack，
    // 而每一个ack都会对应触发一次增长。
    // 以下为一个简单的例子，sender为发送端，receiver为接收端
    // px为包号为x的包，ack x为对第x个包的确认
    // snd_cwnd为拥塞窗口
    // sender                                           receiver
    //  p1 (snd_cwnd 1)  --------------------------->
    //    
    //                   <---------------------------     ack 1
    //  snd_cwnd++ (2)
    //
    //  p2 (snd_cwnd 2)  --------------------------->
    //  p3 (snd_cwnd 2)  ---------------------------> 
    //
    //                   <---------------------------     ack 2
    //  snd_cwnd++ (3)
    //                   <---------------------------     ack 3
    //  snd_cwnd++ (4)
    //
    //  p4 (snd_cwnd 4)  --------------------------->
    //  p5 (snd_cwnd 4)  --------------------------->
    //  p6 (snd_cwnd 4)  --------------------------->
    //  p7 (snd_cwnd 4)  --------------------------->
    //
    //                   <---------------------------     ack 4
    //  snd_cwnd++ (5)
    //                   <---------------------------     ack 5
    //  snd_cwnd++ (6)
    //                   <---------------------------     ack 6
    //  snd_cwnd++ (7)
    //                   <---------------------------     ack 7
    //  snd_cwnd++ (8)
    //  send with snd_cwnd = 8 (p8 - p15)
    // 每一个ack对应增加一个窗口大小，不丢包的场景下相当于窗口以指数上升
    // 1 --> 2 --> 4 --> 8
	while (tp->snd_cwnd_cnt >= tp->snd_cwnd) {
		tp->snd_cwnd_cnt -= tp->snd_cwnd;
		if (tp->snd_cwnd < tp->snd_cwnd_clamp)
			tp->snd_cwnd++;
	}
}
```

## 2.4 拥塞避免
拥塞避免的代码比较简短，注意2.3中所写的，snd\_cwnd\_cnt为线性增长器，其单位为1 / w。在reno调用中，这里的w也为snd\_cwnd窗口大小。即每一个ack只增加1 / snd_\cwnd大小的窗口。

```cpp
/* In theory this is tp->snd_cwnd += 1 / tp->snd_cwnd (or alternative w) */
void tcp_cong_avoid_ai(struct tcp_sock *tp, u32 w)
{
    // 每次cnt++，直到w次后snd_cwnd++，即单位 1 / w
	if (tp->snd_cwnd_cnt >= w) {
		if (tp->snd_cwnd < tp->snd_cwnd_clamp)
			tp->snd_cwnd++;
		tp->snd_cwnd_cnt = 0;
	} else {
		tp->snd_cwnd_cnt++;
	}
}
```

# 3. kernel 4.9的改变
对tcp\_slow\_start的改动不算是4.9的，早在3.18之前就已经改变了，使用的已经不是之前的snd\_cwnd\_cnt，而是采用tcp-abc算法来进行慢启动。

慢启动仍然使用类似tcp-abc的实现机制，不过其并不以byte作为单位，而是以MSS作为单位进行处理。
```cpp
/* Slow start is used when congestion window is no greater than the slow start
 * threshold. We base on RFC2581 and also handle stretch ACKs properly.
 * We do not implement RFC3465 Appropriate Byte Counting (ABC) per se but
 * something better;) a packet is only considered (s)acked in its entirety to
 * defend the ACK attacks described in the RFC. Slow start processes a stretch
 * ACK of degree N as if N acks of degree 1 are received back to back except
 * ABC caps N to 2. Slow start exits when cwnd grows over ssthresh and
 * returns the leftover acks to adjust cwnd in congestion avoidance mode.
 */
u32 tcp_slow_start(struct tcp_sock *tp, u32 acked)
{
    // 使用确认的包数(其中可能包括sack的确认，或者重传数据的确认都加上)
    // 来更新窗口值，而不是之前的byte。
    // 在函数tcp_clean_rtx_queue中有更新对应的delivered。
    // 其更新的值貌似和MSS有关系。
	u32 cwnd = min(tp->snd_cwnd + acked, tp->snd_ssthresh);

    // 当acked仍然有值，说明超过阈值，处理完slow start后还会进行congestion avoid的处理。
	acked -= cwnd - tp->snd_cwnd;
	tp->snd_cwnd = min(cwnd, tp->snd_cwnd_clamp);

	return acked;
}
```

拥塞避免上和老版本类似，也使用到了线性增长器，但是涨幅比之前版本较大，并不是以1为计数，而是以acked，即已经确认的MSS个数据片作为单位。

```cpp
/* In theory this is tp->snd_cwnd += 1 / tp->snd_cwnd (or alternative w),
 * for every packet that was ACKed.
 */
void tcp_cong_avoid_ai(struct tcp_sock *tp, u32 w, u32 acked)
{
	/* If credits accumulated at a higher w, apply them gently now. */
	// 第一次线性增长计算。
	if (tp->snd_cwnd_cnt >= w) {
		tp->snd_cwnd_cnt = 0;
		tp->snd_cwnd++;
	}

    // 以 acked / snd_cwnd为单位增长。将循环改为除法。
	tp->snd_cwnd_cnt += acked;
	if (tp->snd_cwnd_cnt >= w) {
		u32 delta = tp->snd_cwnd_cnt / w;

		tp->snd_cwnd_cnt -= delta * w;
		tp->snd_cwnd += delta;
	}
	tp->snd_cwnd = min(tp->snd_cwnd, tp->snd_cwnd_clamp);
}
```

@小刘
```
悦
```


[1]: https://tools.ietf.org/html/rfc5681
[2]: http://blog.csdn.net/dog250/article/details/51348568
[3]: https://tools.ietf.org/html/rfc3465
[4]: https://tools.ietf.org/html/rfc3742
