---
title: webrtc中rtcp码率控制分析
date: 2017-05-09 11:22:45
tags: [webrt]
---

# 0. 参考文档
[1] [google congestion control](http://c3lab.poliba.it/images/6/65/Gcc-analysis.pdf)

# 1. 简介

webrtc的带宽估计分为两部分，一部分为发送端根据rtcp反馈信息进行反馈，另一部分为接收端根据收到的rtp数据进行相应的码率估计[[1]]。
本文先分析发送端根据rtcp反馈信息进行码率调整的部分代码。

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/google_congestion_control_architecture.png)

具体计算公式:
![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/rtcp_feedback.png)

# 2. 代码结构
## 2.1 类关系

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/rtcp_feedback_class.png)

rtp\_stream\_receiver中有一个继承自抽象类RtpRtcp的ModuleRtpRtcpImpl，ModuleRtpRtcpImpl中有一个rtcp\_receiver。当有RTCP包到来时，逐层处理至rtcp\_receiver，当包是rtcp receiver report包，则会将包解析，然后在ModuleRtpRtcpImpl中再次调用rtcp\_receiver中的TriggerCallbacksFromRTCPPacket函数，触发对应rtcp的一些事件，反馈触发的主要是\_cbRtcpBandwidthObserver的观察者(RtcpBandwidthObserverImpl)，这个观察者收到对应的report block之后会计算成带宽估计所需要的参数，并调用属主bitratecontrolImpl类对带宽进行估计，这里会调用SendSideBandwidthEstimation中的UpdateReceiverBlock进行实际的带宽评估。

## 2.2 调用关系图

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/rtcp_feedback_flow.png)

# 3. 代码分析
## 3.1 HandleReportBlock
这个函数中最主要的部分就是RTT的计算，webrtc中对于RTT平滑的因子是一个线性增长的因子。

```cpp
/* 这个函数根据对应的report block生成了一个新的RTCPReportBlockInformation结构体，
 * 并计算出对应的RTT，多report block在调用点处执行循环。  */
void RTCPReceiver::HandleReportBlock(
    const RTCPUtility::RTCPPacket& rtcpPacket,
    RTCPPacketInformation& rtcpPacketInformation,
    uint32_t remoteSSRC)
    EXCLUSIVE_LOCKS_REQUIRED(_criticalSectionRTCPReceiver) {
  // This will be called once per report block in the RTCP packet.
  // We filter out all report blocks that are not for us.
  // Each packet has max 31 RR blocks.
  //
  // We can calc RTT if we send a send report and get a report block back.

  // |rtcpPacket.ReportBlockItem.SSRC| is the SSRC identifier of the source to
  // which the information in this reception report block pertains.

  // Filter out all report blocks that are not for us.
  if (registered_ssrcs_.find(rtcpPacket.ReportBlockItem.SSRC) ==
      registered_ssrcs_.end()) {
    // This block is not for us ignore it.
    return;
  }

  RTCPReportBlockInformation* reportBlock =
      CreateOrGetReportBlockInformation(remoteSSRC,
                                        rtcpPacket.ReportBlockItem.SSRC);
  if (reportBlock == NULL) {
    LOG(LS_WARNING) << "Failed to CreateReportBlockInformation("
                    << remoteSSRC << ")";
    return;
  }

  // 用于RTCP超时的计算。
  _lastReceivedRrMs = _clock->TimeInMilliseconds();
  // 其他字段的拷贝。
  const RTCPPacketReportBlockItem& rb = rtcpPacket.ReportBlockItem;
  reportBlock->remoteReceiveBlock.remoteSSRC = remoteSSRC;
  reportBlock->remoteReceiveBlock.sourceSSRC = rb.SSRC;
  reportBlock->remoteReceiveBlock.fractionLost = rb.FractionLost;
  reportBlock->remoteReceiveBlock.cumulativeLost =
      rb.CumulativeNumOfPacketsLost;
  if (rb.ExtendedHighestSequenceNumber >
      reportBlock->remoteReceiveBlock.extendedHighSeqNum) {
    // We have successfully delivered new RTP packets to the remote side after
    // the last RR was sent from the remote side.
    _lastIncreasedSequenceNumberMs = _lastReceivedRrMs;
  }
  reportBlock->remoteReceiveBlock.extendedHighSeqNum =
      rb.ExtendedHighestSequenceNumber;
  reportBlock->remoteReceiveBlock.jitter = rb.Jitter;
  reportBlock->remoteReceiveBlock.delaySinceLastSR = rb.DelayLastSR;
  reportBlock->remoteReceiveBlock.lastSR = rb.LastSR;

  if (rtcpPacket.ReportBlockItem.Jitter > reportBlock->remoteMaxJitter) {
    reportBlock->remoteMaxJitter = rtcpPacket.ReportBlockItem.Jitter;
  }

  int64_t rtt = 0;
  uint32_t send_time = rtcpPacket.ReportBlockItem.LastSR;
  // RFC3550, section 6.4.1, LSR field discription states:
  // If no SR has been received yet, the field is set to zero.
  // Receiver rtp_rtcp module is not expected to calculate rtt using
  // Sender Reports even if it accidentally can.
  if (!receiver_only_ && send_time != 0) {
	// 当RR在SR之前发送，send_time为0.
	// delay计算:
	// Send SR                                                       Receive RR
	//  |                          delay in RR                           |
 	//  |                        |<----------->|                         |
	//  |<---------------------->|             |<----------------------->|
	//
	// RTT = total_time - delay_in_RR
	//     = receiver_rr_time - send_sr_time - delay_in_RR
	// 即使中间几个SR丢包，但是如果RTT本身是平滑的，那么RTT不会受到这几个丢包的影响
	// 因为SR->RR之间的delay可以精确计算。
    uint32_t delay = rtcpPacket.ReportBlockItem.DelayLastSR;
    // Local NTP time.
    uint32_t receive_time = CompactNtp(NtpTime(*_clock));

    // RTT in 1/(2^16) seconds.
    uint32_t rtt_ntp = receive_time - delay - send_time;
    // Convert to 1/1000 seconds (milliseconds).
    rtt = CompactNtpRttToMs(rtt_ntp);
    if (rtt > reportBlock->maxRTT) {
      // Store max RTT.
      reportBlock->maxRTT = rtt;
    }
    if (reportBlock->minRTT == 0) {
      // First RTT.
      reportBlock->minRTT = rtt;
    } else if (rtt < reportBlock->minRTT) {
      // Store min RTT.
      reportBlock->minRTT = rtt;
    }
    // Store last RTT.
    reportBlock->RTT = rtt;

    // store average RTT
	// RTT的平滑计算。
	// 如果这个块是在CreateOrGetReportBlockInformation新生成的，
	// 则权重会从0开始随着受到的report逐渐递增。
	// srtt(i) = i/(i+1)*srtt(i-1) + 1/(i+1)*rtt + 0.5
    if (reportBlock->numAverageCalcs != 0) {
      float ac = static_cast<float>(reportBlock->numAverageCalcs);
      float newAverage =
          ((ac / (ac + 1)) * reportBlock->avgRTT) + ((1 / (ac + 1)) * rtt);
      reportBlock->avgRTT = static_cast<int64_t>(newAverage + 0.5f);
    } else {
      // First RTT.
      reportBlock->avgRTT = rtt;
    }
    reportBlock->numAverageCalcs++;
  }

  TRACE_COUNTER_ID1(TRACE_DISABLED_BY_DEFAULT("webrtc_rtp"), "RR_RTT", rb.SSRC,
                    rtt);

  // 添加回rtcpPacketInformation，在ModuleRtpRtcpImpl中会使用这个进行事件回调。
  rtcpPacketInformation.AddReportInfo(*reportBlock);
}
```

## 3.2 UpdateMinHistory

这个函数主要用于更新变量min\_bitrate\_history\_，这个变量将会作用于上升区间，用来作为基数，这里简单描述下。

```cpp
// Updates history of min bitrates.
// After this method returns min_bitrate_history_.front().second contains the
// min bitrate used during last kBweIncreaseIntervalMs.
// 主要结合这个函数解释下变量min_bitrate_history_
// 这个变量的两个维度，front记录的是离当前最远的时间，
// 每个速率都是按照时间先后顺序逐渐push到尾部。
// 因此更新的时候，需要先将超时的元素从列表头剔除。
// 后一个维度是最小速率值，
// 在相同的时间区间内，保留最小的速率值。
// |-------Interval 1---------|----------Interval 2------|
// |                          |                          |
// |--t1 < t2 < t3 < t4 < t5--|--t1 < t2 < t3 < t4 < t5--|
// 这样的操作较为简单，不用在每次插入元素时去判断对应的时间区域，再找到对应时间区间的最小值，用部分冗余的内存换取操作的快捷。
void SendSideBandwidthEstimation::UpdateMinHistory(int64_t now_ms) {
  // Remove old data points from history.
  // Since history precision is in ms, add one so it is able to increase
  // bitrate if it is off by as little as 0.5ms.
  while (!min_bitrate_history_.empty() &&
         now_ms - min_bitrate_history_.front().first + 1 >
             kBweIncreaseIntervalMs) {
    min_bitrate_history_.pop_front();
  }

  // Typical minimum sliding-window algorithm: Pop values higher than current
  // bitrate before pushing it.
  while (!min_bitrate_history_.empty() &&
         bitrate_ <= min_bitrate_history_.back().second) {
    min_bitrate_history_.pop_back();
  }

  min_bitrate_history_.push_back(std::make_pair(now_ms, bitrate_));
}
```

## 3.3 UpdateEstimate
函数UpdateReceiverBlock会根据当前的report block对当前带宽估计的一些变量进行相应的赋值，此外，只有当传输包的数量达到一定数量才会再次触发带宽估计的调整。函数UpdateEstimate是主要用于带宽估计的函数。

```cpp
void SendSideBandwidthEstimation::UpdateEstimate(int64_t now_ms) {
  // We trust the REMB and/or delay-based estimate during the first 2 seconds if
  // we haven't had any packet loss reported, to allow startup bitrate probing.
  if (last_fraction_loss_ == 0 && IsInStartPhase(now_ms)) {
    uint32_t prev_bitrate = bitrate_;
	// bwe_incoming_是remb更新的值，如果当前无丢包且在启动阶段，直接使用remb的值。
    if (bwe_incoming_ > bitrate_)
      bitrate_ = CapBitrateToThresholds(now_ms, bwe_incoming_);
      ...
    }
  }
  UpdateMinHistory(now_ms);
  // Only start updating bitrate when receiving receiver blocks.
  // TODO(pbos): Handle the case when no receiver report is received for a very
  // long time.
  if (time_last_receiver_block_ms_ != -1) {
    if (last_fraction_loss_ <= 5) {
      // Loss < 2%: Increase rate by 8% of the min bitrate in the last
      // kBweIncreaseIntervalMs.
      // Note that by remembering the bitrate over the last second one can
      // rampup up one second faster than if only allowed to start ramping
      // at 8% per second rate now. E.g.:
      //   If sending a constant 100kbps it can rampup immediatly to 108kbps
      //   whenever a receiver report is received with lower packet loss.
      //   If instead one would do: bitrate_ *= 1.08^(delta time), it would
      //   take over one second since the lower packet loss to achieve 108kbps.
        
        //TODO:tjl
	  // 这里与公式有一定不同：
	  // 1. 系数不同，且附带一定的修正值(向上取整加1kbps)
	  // 2. 取的是上一个时间间隔之内最小值，比较平滑。
      bitrate_ = static_cast<uint32_t>(
          min_bitrate_history_.front().second * 1.08 + 0.5);

      // Add 1 kbps extra, just to make sure that we do not get stuck
      // (gives a little extra increase at low rates, negligible at higher
      // rates).
      bitrate_ += 1000;

      event_log_->LogBwePacketLossEvent(
          bitrate_, last_fraction_loss_,
          expected_packets_since_last_loss_update_);
    } else if (last_fraction_loss_ <= 26) {
      // Loss between 2% - 10%: Do nothing.
    } else {
      // Loss > 10%: Limit the rate decreases to once a kBweDecreaseIntervalMs +
      // rtt.
      if (!has_decreased_since_last_fraction_loss_ &&
          (now_ms - time_last_decrease_ms_) >=
              (kBweDecreaseIntervalMs + last_round_trip_time_ms_)) {
        time_last_decrease_ms_ = now_ms;

        // Reduce rate:
        //   newRate = rate * (1 - 0.5*lossRate);
        //   where packetLoss = 256*lossRate;
          
          //TODO:tjl
		// 当从未开始降低窗口值，且距离上一次衰减的时间差大于衰减周期加上rtt。
		// 其实当前貌似只有这个case下会对这两个变量赋值。
		// 这里的last_fraction_loss_是一次统计间隔(一定包数)之间的总丢包率。
		// 丢包率的单位是1/256，因此这里是(1 - 丢包率/2) * 当前速率
		// 与公式相同。
        bitrate_ = static_cast<uint32_t>(
            (bitrate_ * static_cast<double>(512 - last_fraction_loss_)) /
            512.0);
        has_decreased_since_last_fraction_loss_ = true;
      }
      event_log_->LogBwePacketLossEvent(
          bitrate_, last_fraction_loss_,
          expected_packets_since_last_loss_update_);
    }
  }
  // 在有效范围内修正。
  bitrate_ = CapBitrateToThresholds(now_ms, bitrate_);
}
```

[1]: [http://c3lab.poliba.it/images/6/65/Gcc-analysis.pdf]

