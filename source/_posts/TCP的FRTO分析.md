---
title: TCP的FRTO分析
date: 2017-04-11 19:04:12
tags: [TCP, linux, 拥塞控制]
---

# TCP的FRTO理解

本文主要描述内核4.9.4中的TCP丢包处理与frto相关的操作，主要覆盖使用sack的场景。

# 0. 参考文档
[1] [RFC-5682](https://tools.ietf.org/html/rfc5682#section-3.1)
[2] [linuxtcp.ps](https://www.cs.helsinki.fi/group/iwtcp/papers/linuxtcp.ps.gz)
[3] [frto.pdf](http://www.sarolahti.fi/pasi/papers/frto-ccr.pdf)

# 1. FRTO要解决的问题
FRTO主要是用来处理在DSACK生效时，突发的延迟触发RTO超时后，不必要的延迟和重传报文的ack造成了DSACK而产生非必要的快速重传[1]。
传统的基于DSACK的RTO超时会有如下问题:

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP_FRTO/conventional.png)

- 1) 在16.5s和17s中间最后一个ack到来之前一直处于慢启动阶段，指数发送数据，
- 2) 18s虚假RTO触发，其实只是延迟，但是这时候RTO工作，重传丢失的报文。
- 3) 19s的时候延迟包的ack到达，注意到此时ack的数据序列号是大于重传报文的序列号。
- 4) 延迟的ack使发送端继续重传之后延迟的ack之后的数据(这部分数据之前已经发送过))(19s到19.5s之间的重传)。
- 5) 在19.8s，之前发送的最大序列号报文被确认，接着由于4中的重传报文陆续到达，接收端发送了一系列以最大序列号报文为ack的Duplicate SACK。
- 6) 发送端收到这些重复的DSACK后，触发了快速重传，降低了传输性能。

使用FRTO后的效果:

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP_FRTO/frto.png)

- 1) 同上
- 2) 同上
- 3) 同上
- 4) 由于延迟包的ack更新了snd_una，因此这里不重传其余数据，而是发送两个新的分片。
- 5) 延迟包的ack陆续到达，此时由于未重传的包收到了对应的ack，因此可以判断当前是一个虚假的RTO，继续发送新的数据。

# 1. FRTO rfc解释
这里主要针对sack的场景，即rfc-5682的section 3。

```
   1) When the retransmission timer expires, retransmit the first
      unacknowledged segment and set SpuriousRecovery to FALSE.
      Following the recommendation in the SACK specification [MMFR96],
      reset the SACK scoreboard.  If "RecoveryPoint" is larger than or
      equal to SND.UNA, do not enter step 2 of this algorithm.  Instead,
      set variable "RecoveryPoint" to indicate the highest sequence
      number transmitted so far and continue with slow-start
      retransmissions following the conventional RTO recovery algorithm.

   2) Wait until the acknowledgment of the data retransmitted due to the
      timeout arrives at the sender.  If duplicate ACKs arrive before
      the cumulative acknowledgment for retransmitted data, adjust the
      scoreboard according to the incoming SACK information.  Stay in
      step 2 and wait for the next new acknowledgment.  If the
      retransmission timeout expires again, go to step 1 of the
      algorithm.  When a new acknowledgment arrives, set variable
      "RecoveryPoint" to indicate the highest sequence number
      transmitted so far.

      a) If the Cumulative Acknowledgment field covers "RecoveryPoint"
         but not more than "RecoveryPoint", revert to the conventional
         RTO recovery and set the congestion window to no more than 2 *
         MSS, like a regular TCP would do.  Do not enter step 3 of this
         algorithm.

      b) Else, if the Cumulative Acknowledgment field does not cover
         "RecoveryPoint" but is larger than SND.UNA, transmit up to two
         new (previously unsent) segments and proceed to step 3.  If the
         TCP sender is not able to transmit any previously unsent data
         -- either due to receiver window limitation or because it does
         not have any new data to send -- the recommended action is to
         refrain from entering step 3 of this algorithm.  Rather,
         continue with slow-start retransmissions following the
         conventional RTO recovery algorithm.

         It is also possible to apply some of the alternatives for
         handling window-limited cases discussed in Appendix A.

   3) The next acknowledgment arrives at the sender.  Either a duplicate
      ACK or a new cumulative ACK (advancing the window) applies in this
      step.  Other types of ACKs are ignored without any action.

      a) If the Cumulative Acknowledgment field or the SACK information
         covers more than "RecoveryPoint", set the congestion window to
         no more than 3 * MSS and proceed with the conventional RTO
         recovery, retransmitting unacknowledged segments.  Take this
         branch also when the acknowledgment is a duplicate ACK and it
         does not acknowledge any new, previously unacknowledged data
         below "RecoveryPoint" in the SACK information.  Leave
         SpuriousRecovery set to FALSE.

      b) If the Cumulative Acknowledgment field or a SACK information in
         the ACK does not cover more than "RecoveryPoint" AND it
         acknowledges data that was not acknowledged earlier (either
         with cumulative acknowledgment or using SACK information),
         declare the timeout spurious and set SpuriousRecovery to
         SPUR_TO.  The retransmission timeout can be declared spurious,
         because the segment acknowledged with this ACK was transmitted
         before the timeout.
```

注1: RecoveryPoint为tcp中的high_seq，后称恢复点。
注2: snd_nxt为下一个将要发送的包的包号。
- 3.1:  当重传定时器超时了，首先重传第一个未确认的分片，同时设置SpuriousRecovery为false，并重置sack的计分板。
如果恢复点大于等于下一个要发的包(好像基本不可能，最多等于)，则将恢复点设置为当前发送的最大包。进入常规恢复。否则进入step 2。
- 3.2: 等待重传数据的ack到来。如果重复的ack比新的ack更早到达，则更新相应的sack计分板，同时留在第二步，等待新的ack到来。如果重传定时器再次超时，回到第一步。当新的ack到来后，更新恢复点。
	- a) 如果到来的ack中包括了恢复点，但不超过恢复点，撤销至普通恢复，同时拥塞窗口设置不大于2倍MSS。不进入第三步。
	- b) 如果到来的ack中不包含恢复点，但是收到的包超过未确认包，发送两个新的分片，并进入步骤3。如果当前发送端无法发送新报文(接收窗口限制或者没有新的应用层数据)，这里建议不要进入步骤3，而是使用恢复步骤。
-3.3: 下一个ack到来，除了新的ack或者是重复ack，其他ack在这个步骤中都被忽略。
	- a) 如果新的ack包含了恢复点，则设置拥塞窗口不超过3倍MSS大小并执行恢复操作，重传未确认分片。这个分支也处理重复ack但是没有确认任何新块的场景。
	- b) 如果新的ack或者sack信息中没有包含恢复点，且确认的数据是之前没有确认的，则认为这个超时是一个奇怪的超时，并设置SpuriousRecovery为SPUR_TO。这个重传被标记为奇怪的，因为这个分片在超时前被确认了。

其实论文中的图比rfc这段(恢复点的几个判断理解费力。。)更好理解，也更契合代码，为了识别出虚假的RTO：
- 超时后先只重发丢失的一个包(3.1)。
- 判断重传后的第一个新的ack是否更新了snd\_una，如果更新了snd\_una，就发送新的数据，在判断虚假RTO之前不重传数据(3.2.b)
- 如果没重传就已经收到数据，尝试撤销本次RTO，如果撤销成功，这就是一个虚假RTO，进入恢复流程(3.3.b)。如果还是重复ack，则认为这是一个丢包事件(3.3.a)。

# 2. 4.9.4中的代码
这里主要分析tcp\_process\_loss函数。

```
/* Process an ACK in CA_Loss state. Move to CA_Open if lost data are
 * recovered or spurious. Otherwise retransmits more on partial ACKs.
 */
// 这个函数中如果可以恢复，则会直接进入CA_OPEN。
// 不进入CA_OPEN则会直接返回tcp_ack中进行重传(或者是发送新分片)。
static void tcp_process_loss(struct sock *sk, int flag, bool is_dupack,
			     int *rexmit)
{
	struct tcp_sock *tp = tcp_sk(sk);
	// 完全恢复
	bool recovered = !before(tp->snd_una, tp->high_seq);

	// 最大的未确认包已经发生改变，可能是之前丢包已经恢复，尝试撤销丢包状态。
	// 当frto关闭时，这是唯二可以退出LOSS状态的条件。
	if ((flag & FLAG_SND_UNA_ADVANCED) &&
	    tcp_try_undo_loss(sk, false))
		return;

	// frto在enter_loss时候置位，根据超时时的设置决定是否可能是一个虚假的RTO。
	if (tp->frto) { /* F-RTO RFC5682 sec 3.1 (sack enhanced version). */
		/* Step 3.b. A timeout is spurious if not all data are
		 * lost, i.e., never-retransmitted data are (s)acked.
		 */
		// #define FLAG_ORIG_SACK_ACKED	0x200 /* Never retransmitted data are (s)acked	*/
		// 包没有丢，因为还没重传就收到了ACK，只是延迟了，这是虚假RTO，撤销丢包状态。
		// 注: 这里的frto_undo传参为true，必定恢复。
		// 对应RTO论文图中的延迟包的acks。
		if ((flag & FLAG_ORIG_SACK_ACKED) &&
		    tcp_try_undo_loss(sk, true))
			return;

		if (after(tp->snd_nxt, tp->high_seq)) {
			// 如果是刚进入LOSS状态，会先尝试重传，这时候snd_nxt总是等于high_seq的，
			// 这个分支主要对应3.2.b之后发送了两个新分片。
			// 虽然发送了新分片，没有发重传，但是这时候收到的ack并没有更新una
			// 说明这个rtt中，之前una的包仍旧没有达到，因此这里认为他是真的超时
			// 关闭frto。对应3.3.a
			if (flag & FLAG_DATA_SACKED || is_dupack)
				tp->frto = 0; /* Step 3.a. loss was real */
		} else if (flag & FLAG_SND_UNA_ADVANCED && !recovered) {
			// 这里进入的条件为
			// 1. snd_nxt == high_seq，还没发送过新分片
			// 2. una更新过，且没有完全恢复
			// 执行3.2.b，发送新分片。
			// 对应论文图中收到了一个更新过snd_una的ack。
			tp->high_seq = tp->snd_nxt;
			/* Step 2.b. Try send new data (but deferred until cwnd
			 * is updated in tcp_ack()). Otherwise fall back to
			 * the conventional recovery.
			 */
			// 3.2.b中判断当前是否可以发送新分片。
			if (tcp_send_head(sk) &&
			    after(tcp_wnd_end(tp), tp->snd_nxt)) {
				*rexmit = REXMIT_NEW;
				return;
			}
			tp->frto = 0;
		}
	}

	// 已经完全恢复，则撤销对应的恢复操作，并进入TCP_CA_OPEN状态。后续将进入恢复状态。
	// 这里主要处理了其他几个不在FRTO可处理的场景，如3.2.a和3.3.a
	// 唯一进入这里但frto还可能生效的场景为:
	// 发送新分片后，但是收到了一个不是新的sack，且不是一个dup sack。
	// 在这种情况下的处理应该和上一个旧的sack相同。
	// 个人理解应该是3.3中被忽略的case。
	if (recovered) {
		/* F-RTO RFC5682 sec 3.1 step 2.a and 1st part of step 3.a */
		tcp_try_undo_recovery(sk);
		return;
	}
	if (tcp_is_reno(tp)) {
		/* A Reno DUPACK means new data in F-RTO step 2.b above are
		 * delivered. Lower inflight to clock out (re)tranmissions.
		 */
		if (after(tp->snd_nxt, tp->high_seq) && is_dupack)
			tcp_add_reno_sack(sk);
		else if (flag & FLAG_SND_UNA_ADVANCED)
			tcp_reset_reno_sack(tp);
	}
	*rexmit = REXMIT_LOST;
}
```

# 3. 简单流程图

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/TCP_FRTO/tcp_frto.png)

# 3.1 常规frto流程
- 1) RTO超时，发送新分片
- 2) 收到一个ack，进入tcp\_process\_loss处理，此时frto开启，如果延迟的ack更新了una，则直接恢复loss状态。如果刚好等于重传包，这时候先发送新分片。
- 3) 又一个新的ack到来，这时候由于没有重传包，延迟的ack会更新una，直接撤销丢包处理，离开LOSS状态。

# 3.2 frto恢复失败流程
- 1) RTO超时，发送新分片
- 2) 收到一个ack，进入tcp\_process\_loss处理，此时frto开启，如上发送新分片。
- 3) 又一个新的ack到来，这时候由于丢包，不更新对应的una，因此关闭frto，进入重传流程。

