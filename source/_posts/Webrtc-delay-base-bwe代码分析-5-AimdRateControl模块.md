---
title: 'Webrtc delay-base-bwe代码分析(5): AimdRateControl模块'
date: 2017-05-22 19:38:38
tags: [webrtc, 拥塞控制]
---

# 0. 简介

这个模块是根据OveruseDetector模块计算出来的状态来维护码率控制模块的自动状态机，并更新估算出来的对端发送速率，提供给REMB进行反馈。

# 1. 原理

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/remote_rate_finite_state_machine.png)

一共维持三个状态，增长、保持、衰减，状态转换根据OveruseDetector的三个状态(Normal, Overuse, Underuse)来进行判断。

- 当Overuse发生时，无论什么状态都进入衰减。
- 当Underuse发生时，无论什么状态都进入保持状态。
- 在保持和增长阶段，Normal状态将保持继续增长。
- 在衰减阶段，Normal状态会将状态拉回保持状态。

# 2. 代码

核心函数为ChangeBitrate，其他部分代码比较简单这里不贴了。

```cpp
uint32_t AimdRateControl::ChangeBitrate(uint32_t current_bitrate_bps,
                                        uint32_t incoming_bitrate_bps,
                                        int64_t now_ms) {
  // 在调用函数update更新对应的链路状态估计，累积码率，噪声值后
  // 会将updated置位，如果没置位则不会去更新码率。
  if (!updated_) {
    return current_bitrate_bps_;
  }
  // An over-use should always trigger us to reduce the bitrate, even though
  // we have not yet established our first estimate. By acting on the over-use,
  // we will end up with a valid estimate.
  // 初始化未完成，如果不是一开始就Overuse，直接返回初始的码率即可。
  if (!bitrate_is_initialized_ && current_input_.bw_state != kBwOverusing)
    return current_bitrate_bps_;
  updated_ = false;
  // 这里对状态进行转换，这个函数是状态机状态转换函数
  // 1. Underuse总是进入Hold状态。
  // 2. Overuse总是进入Dec状态。
  // 3. Normal状态维持，除非当前在Hold状态，此时会进入Inc状态。
  ChangeState(current_input_, now_ms);
  // Calculated here because it's used in multiple places.
  const float incoming_bitrate_kbps = incoming_bitrate_bps / 1000.0f;
  // Calculate the max bit rate std dev given the normalized
  // variance and the current incoming bit rate.
  const float std_max_bit_rate = sqrt(var_max_bitrate_kbps_ *
                                      avg_max_bitrate_kbps_);
  switch (rate_control_state_) {
	// 保持状态不更新码率
    case kRcHold:
      break;

    case kRcIncrease:
	  // 三个状态，在最大值附近，超过最大值，比最大值高到不知道哪里去
	  // 最大均值已初始化，且当前码率高于最大值加上三倍方差，此时进入
	  // 比最大值高到不知道哪里去的状态，同时认为这个均值并不是很好使，复位。
	  // Above声明了，但是没有找到相应调用点。
      if (avg_max_bitrate_kbps_ >= 0 &&
          incoming_bitrate_kbps >
              avg_max_bitrate_kbps_ + 3 * std_max_bit_rate) {
        ChangeRegion(kRcMaxUnknown);
        avg_max_bitrate_kbps_ = -1.0;
      }
	  // 
      if (rate_control_region_ == kRcNearMax) {
        // Approximate the over-use estimator delay to 100 ms.
		// 已经接近最大值了，此时增长需谨慎，加性增加。
        const int64_t response_time = rtt_ + 100;
        uint32_t additive_increase_bps = AdditiveRateIncrease(
            now_ms, time_last_bitrate_change_, response_time);
        current_bitrate_bps += additive_increase_bps;

      } else {
		// 由于没有Above状态的使用，因此认为比最大值高到不知道哪里去的状态属于
		// 上界未定，放开手倍增码率。
        uint32_t multiplicative_increase_bps = MultiplicativeRateIncrease(
            now_ms, time_last_bitrate_change_, current_bitrate_bps);
        current_bitrate_bps += multiplicative_increase_bps;
      }

      time_last_bitrate_change_ = now_ms;
      break;

    case kRcDecrease:
      bitrate_is_initialized_ = true;
      if (incoming_bitrate_bps < min_configured_bitrate_bps_) {
		// 真的不能再低了....
        current_bitrate_bps = min_configured_bitrate_bps_;
      } else {
        // Set bit rate to something slightly lower than max
        // to get rid of any self-induced delay.
        current_bitrate_bps = static_cast<uint32_t>(beta_ *
                                                    incoming_bitrate_bps + 0.5);
        if (current_bitrate_bps > current_bitrate_bps_) {
		  // 本次速率仍然在增长
          // Avoid increasing the rate when over-using.
          if (rate_control_region_ != kRcMaxUnknown) {
			// 如果上界可靠，则将码率设置在最大均值的beta_倍处，
			// 默认的beta_为0.85，同paper。
            current_bitrate_bps = static_cast<uint32_t>(
                beta_ * avg_max_bitrate_kbps_ * 1000 + 0.5f);
          }
		  // 进行修正，和上一轮迭代的码率取小，如果上界不定
		  // 则取上一次迭代的码率值。
          current_bitrate_bps = std::min(current_bitrate_bps,
                                         current_bitrate_bps_);
        }
		// 更新过新的码率值后，认为现在已经在最大均值附近。
		// 注意，每次认为上界无效时，总会把最大均值复位
		// 这里设置完对应状态后，即使上界无效，下面总会更新一个最大均值。
        ChangeRegion(kRcNearMax);

        if (incoming_bitrate_kbps < avg_max_bitrate_kbps_ -
            3 * std_max_bit_rate) {
		  // 当前速率小于均值较多，认为均值不可靠，复位
          avg_max_bitrate_kbps_ = -1.0f;
        }

		// 衰减状态下需要更新最大均值
        UpdateMaxBitRateEstimate(incoming_bitrate_kbps);
      }
      // Stay on hold until the pipes are cleared.
	  // 降低码率后回到HOLD状态，如果网络状态仍然不好，在Overuse仍然会进入Dec状态。
	  // 如果恢复，则不会是Overuse，会保持或增长。
      ChangeState(kRcHold);
      time_last_bitrate_change_ = now_ms;
      break;

    default:
      assert(false);
  }
  if ((incoming_bitrate_bps > 100000 || current_bitrate_bps > 150000) &&
      current_bitrate_bps > 1.5 * incoming_bitrate_bps) {
    // Allow changing the bit rate if we are operating at very low rates
    // Don't change the bit rate if the send side is too far off
    current_bitrate_bps = current_bitrate_bps_;
    time_last_bitrate_change_ = now_ms;
  }
  return current_bitrate_bps;
}
```

加性码率增长代码如下：

```cpp
uint32_t AimdRateControl::AdditiveRateIncrease(
    int64_t now_ms, int64_t last_ms, int64_t response_time_ms) const {
  assert(response_time_ms > 0);
  double beta = 0.0;
  if (last_ms > 0) {
	// 时间间隔和RTT之比作为系数。
	// 疑问，这里的时间点是经过采样的，可能会大于rtt？
    beta = std::min((now_ms - last_ms) / static_cast<double>(response_time_ms),
                    1.0);
    if (in_experiment_)
      beta /= 2.0;
  }
  // 默认30fps，由于每个包不超过mtu，一般也就1100+，用这两个值估计每帧码率和每帧包数。
  // 并计算平均每个包的大小，最终增加的比特数不超过1000。
  double bits_per_frame = static_cast<double>(current_bitrate_bps_) / 30.0;
  double packets_per_frame = std::ceil(bits_per_frame / (8.0 * 1200.0));
  double avg_packet_size_bits = bits_per_frame / packets_per_frame;
  uint32_t additive_increase_bps = std::max(
      1000.0, beta * avg_packet_size_bits);
  return additive_increase_bps;
}
```

乘性部分比较简单，也是根据时间差来调整系数。

```cpp
uint32_t AimdRateControl::MultiplicativeRateIncrease(
    int64_t now_ms, int64_t last_ms, uint32_t current_bitrate_bps) const {
  double alpha = 1.08;
  if (last_ms > -1) {
	// 系数计算与文档中的1.05略有不同，使用时间差作为系数，1.08作为底数。
    int time_since_last_update_ms = std::min(static_cast<int>(now_ms - last_ms),
                                             1000);
    alpha = pow(alpha,  time_since_last_update_ms / 1000.0);
  }
  uint32_t multiplicative_increase_bps = std::max(
      current_bitrate_bps * (alpha - 1.0), 1000.0);
  return multiplicative_increase_bps;
}
```

最后一个是最大均值和方差的更新，主要在衰减状态时候进行估计。

```
void AimdRateControl::UpdateMaxBitRateEstimate(float incoming_bitrate_kbps) {
  const float alpha = 0.05f;
  // 当前没有初始值，先设为当前码率，如果有的话，就用当前的值和均值做平滑。
  if (avg_max_bitrate_kbps_ == -1.0f) {
    avg_max_bitrate_kbps_ = incoming_bitrate_kbps;
  } else {
    avg_max_bitrate_kbps_ = (1 - alpha) * avg_max_bitrate_kbps_ +
        alpha * incoming_bitrate_kbps;
  }
  // Estimate the max bit rate variance and normalize the variance
  // with the average max bit rate.
  const float norm = std::max(avg_max_bitrate_kbps_, 1.0f);
  // 方差的平滑
  var_max_bitrate_kbps_ = (1 - alpha) * var_max_bitrate_kbps_ +
      alpha * (avg_max_bitrate_kbps_ - incoming_bitrate_kbps) *
          (avg_max_bitrate_kbps_ - incoming_bitrate_kbps) / norm;
  // 0.4 ~= 14 kbit/s at 500 kbit/s
  if (var_max_bitrate_kbps_ < 0.4f) {
    var_max_bitrate_kbps_ = 0.4f;
  }
  // 2.5f ~= 35 kbit/s at 500 kbit/s
  if (var_max_bitrate_kbps_ > 2.5f) {
    var_max_bitrate_kbps_ = 2.5f;
  }
}
```


