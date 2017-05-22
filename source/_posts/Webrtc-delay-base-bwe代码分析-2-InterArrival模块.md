---
title: 'Webrtc delay-base-bwe代码分析(2): InterArrival模块'
date: 2017-05-22 19:28:31
tags: [webrtc, 拥塞控制]
---

# 0. 参考文档
[1] [google congestion control](http://c3lab.poliba.it/images/6/65/Gcc-analysis.pdf)
[2] [Rtp payload format for h264](https://www.ietf.org/rfc/rfc3984.txt)

# 1. 功能

该模块主要对到达的时间进行小范围内的统计、采样，并根据一定的时间间隔计算出对应的延迟、传输大小变化。

```
The arrival-time filter. The goal of this block is to produce an estimate m(ti) of the one way delay gradient. For this purpose, we employ a Kalman filter that estimatesm(ti) based on the measured one way delay gradient dm(ti) which is computed as follows:
	dm(t(i) = (t(i) - t(i-1))-(Ti - T(i-1))
where Ti is the time at which the first packet of the i-th video frame has been sent and ti is the time at which the last packet that forms the video frame has been received.
```

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/gcc_arrival_filter.png)

# 2. 流程

注：RTP的timestamp默认频率为90kHz。

以发送的时间戳为准，每5ms内发送的包划为一个时间戳范围内，组成一个TimestampGroup，同一个TimestampGroup内的包，后来的时间覆盖先到的。当某个包到来时，根据时间戳判断是一个新的TimestampGroup，此时才会执行延迟差值的计算，计算使用上一个group和当前group的时间值。

注2：这里代码不涉及使用绝对的时间，涉及sdp中的abs\_time。
注3：这里默认的一个group时间为5ms，对于rtp的timestamp默认频率，450个rtp timestamp为一个group的tick。



# 3. 代码

流程比较简单，主要就是new group的判断，以及group时间的更新和差值计算。

```cpp
bool InterArrival::ComputeDeltas(uint32_t timestamp,
                                 int64_t arrival_time_ms,
                                 int64_t system_time_ms,
                                 size_t packet_size,
                                 uint32_t* timestamp_delta,
                                 int64_t* arrival_time_delta_ms,
                                 int* packet_size_delta) {
  assert(timestamp_delta != NULL);
  assert(arrival_time_delta_ms != NULL);
  assert(packet_size_delta != NULL);
  bool calculated_deltas = false;
  if (current_timestamp_group_.IsFirstPacket()) {
    // We don't have enough data to update the filter, so we store it until we
    // have two frames of data to process.
	// 尚未完全初始化
    current_timestamp_group_.timestamp = timestamp;
    current_timestamp_group_.first_timestamp = timestamp;
  } else if (!PacketInOrder(timestamp)) {
    return false;
  } else if (NewTimestampGroup(arrival_time_ms, timestamp)) {
    // First packet of a later frame, the previous frame sample is ready.
	// 需要新开一个group，此时计算prev_group和current_group的差值。
    if (prev_timestamp_group_.complete_time_ms >= 0) {
      // 发送时间的差值，即公式中的T(i) - T(i - 1)
      *timestamp_delta = current_timestamp_group_.timestamp -
                         prev_timestamp_group_.timestamp;
	  // 接收时间的差值，即公式中的t(i) - t(i - 1)
      *arrival_time_delta_ms = current_timestamp_group_.complete_time_ms -
                               prev_timestamp_group_.complete_time_ms;
      // Check system time differences to see if we have an unproportional jump
      // in arrival time. In that case reset the inter-arrival computations.
	  // 防止一些时间戳异常处理。
      int64_t system_time_delta_ms =
          current_timestamp_group_.last_system_time_ms -
          prev_timestamp_group_.last_system_time_ms;
      if (*arrival_time_delta_ms - system_time_delta_ms >=
          kArrivalTimeOffsetThresholdMs) {
        LOG(LS_WARNING) << "The arrival time clock offset has changed (diff = "
                        << *arrival_time_delta_ms - system_time_delta_ms
                        << " ms), resetting.";
        Reset();
        return false;
      }
	  // 乱序包的异常处理
      if (*arrival_time_delta_ms < 0) {
        // The group of packets has been reordered since receiving its local
        // arrival timestamp.
        ++num_consecutive_reordered_packets_;
		// 连续乱序包的异常处理
        if (num_consecutive_reordered_packets_ >= kReorderedResetThreshold) {
          LOG(LS_WARNING) << "Packets are being reordered on the path from the "
                             "socket to the bandwidth estimator. Ignoring this "
                             "packet for bandwidth estimation, resetting.";
          Reset();
        }
        return false;
      } else {
        num_consecutive_reordered_packets_ = 0;
      }
      assert(*arrival_time_delta_ms >= 0);
      *packet_size_delta = static_cast<int>(current_timestamp_group_.size) -
          static_cast<int>(prev_timestamp_group_.size);
      calculated_deltas = true;
    }
	// 更新整组group
    prev_timestamp_group_ = current_timestamp_group_;
    // The new timestamp is now the current frame.
    current_timestamp_group_.first_timestamp = timestamp;
    current_timestamp_group_.timestamp = timestamp;
    current_timestamp_group_.size = 0;
  } else {
	// 还在之前的时间范围内，此时将rtp timestanp更新，更新为一个新值。
    current_timestamp_group_.timestamp = LatestTimestamp(
        current_timestamp_group_.timestamp, timestamp);
  }
  // Accumulate the frame size.
  // 更新group中的数据。
  current_timestamp_group_.size += packet_size;
  current_timestamp_group_.complete_time_ms = arrival_time_ms;
  current_timestamp_group_.last_system_time_ms = system_time_ms;

  return calculated_deltas;
}

// Assumes that |timestamp| is not reordered compared to
// |current_timestamp_group_|.
bool InterArrival::NewTimestampGroup(int64_t arrival_time_ms,
                                     uint32_t timestamp) const {
  if (current_timestamp_group_.IsFirstPacket()) {
    return false;
  } else if (BelongsToBurst(arrival_time_ms, timestamp)) {
    return false;
  } else {
    uint32_t timestamp_diff = timestamp -
        current_timestamp_group_.first_timestamp;
    // rtp时间差大于一个Tick，即 90 * 5 个rtp时间采样点。
    return timestamp_diff > kTimestampGroupLengthTicks;
  }
}

// 允许group超出定死的group时间范围。
// 当这个函数返回false，会认为不是一个新的group。
bool InterArrival::BelongsToBurst(int64_t arrival_time_ms,
                                  uint32_t timestamp) const {
  if (!burst_grouping_) {
    return false;
  }
  assert(current_timestamp_group_.complete_time_ms >= 0);
  int64_t arrival_time_delta_ms = arrival_time_ms -
      current_timestamp_group_.complete_time_ms;
  uint32_t timestamp_diff = timestamp - current_timestamp_group_.timestamp;
  // 将rtp的时间差值转换为ms，乘上采样频率 1 / 90.
  int64_t ts_delta_ms = timestamp_to_ms_coeff_ * timestamp_diff + 0.5;
  if (ts_delta_ms == 0)
    return true;
  int propagation_delta_ms = arrival_time_delta_ms - ts_delta_ms;
  // 传播时间差，即当前包的RTT小于上一个包的RTT，且到达时间差小于爆发的阈值
  return propagation_delta_ms < 0 &&
      arrival_time_delta_ms <= kBurstDeltaThresholdMs;
}
```


