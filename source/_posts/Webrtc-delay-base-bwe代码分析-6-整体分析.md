---
title: 'Webrtc delay-base-bwe代码分析(6): 整体分析'
date: 2017-05-22 19:44:23
tags: [webrtc, 拥塞控制]
---

当收到RTP数据包时，会触发RemoteBitrateEstimatorSingleStream::IncomingPacket函数进行处理。
这里面使用到了之前几篇文章分析的模块，各自进行各自的处理。

```cpp
void RemoteBitrateEstimatorSingleStream::IncomingPacket(
    int64_t arrival_time_ms,
    size_t payload_size,
    const RTPHeader& header) {
  uint32_t ssrc = header.ssrc;
  uint32_t rtp_timestamp = header.timestamp +
      header.extension.transmissionTimeOffset;
  int64_t now_ms = clock_->TimeInMilliseconds();
  CriticalSectionScoped cs(crit_sect_.get());
  SsrcOveruseEstimatorMap::iterator it = overuse_detectors_.find(ssrc);
  // 根据SSRC查找对应的检测器
  if (it == overuse_detectors_.end()) {
    // This is a new SSRC. Adding to map.
    // TODO(holmer): If the channel changes SSRC the old SSRC will still be
    // around in this map until the channel is deleted. This is OK since the
    // callback will no longer be called for the old SSRC. This will be
    // automatically cleaned up when we have one RemoteBitrateEstimator per REMB
    // group.
    std::pair<SsrcOveruseEstimatorMap::iterator, bool> insert_result =
        overuse_detectors_.insert(std::make_pair(
            ssrc, new Detector(now_ms, OverUseDetectorOptions(), true)));
    it = insert_result.first;
  }
  Detector* estimator = it->second;
  estimator->last_packet_time_ms = now_ms;

  // Check if incoming bitrate estimate is valid, and if it needs to be reset.
  // 会先以当前时间为基准，从历史的传输数据中尝试获取未更新当前时间的旧码率值。
  rtc::Optional<uint32_t> incoming_bitrate = incoming_bitrate_.Rate(now_ms);
  if (incoming_bitrate) {
    last_valid_incoming_bitrate_ = *incoming_bitrate;
  } else if (last_valid_incoming_bitrate_ > 0) {
    // Incoming bitrate had a previous valid value, but now not enough data
    // point are left within the current window. Reset incoming bitrate
    // estimator so that the window size will only contain new data points.
    incoming_bitrate_.Reset();
    last_valid_incoming_bitrate_ = 0;
  }
  // 在历史传输数据中更新当前数据。
  incoming_bitrate_.Update(payload_size, now_ms);

  const BandwidthUsage prior_state = estimator->detector.State();
  uint32_t timestamp_delta = 0;
  int64_t time_delta = 0;
  int size_delta = 0;
  // 到达时间统计已经有足够样本构成一个新的group时，
  if (estimator->inter_arrival.ComputeDeltas(
          rtp_timestamp, arrival_time_ms, now_ms, payload_size,
          &timestamp_delta, &time_delta, &size_delta)) {
    double timestamp_delta_ms = timestamp_delta * kTimestampToMs;
	// 根据到达时间间隔，更新卡尔曼滤波器
    estimator->estimator.Update(time_delta, timestamp_delta_ms, size_delta,
                                estimator->detector.State());
	// 根据卡尔曼滤波器校准后的值，更新当前链路使用状态。
    estimator->detector.Detect(estimator->estimator.offset(),
                               timestamp_delta_ms,
                               estimator->estimator.num_of_deltas(), now_ms);
  }
  // 当处于连续过载状态时候需要立刻对码率进行更新，
  // 如果是其他状态，会由定时器在Process函数中更新对应状态。
  // UpdateEstimate会对状态机AimdRateControl进行处理。
  if (estimator->detector.State() == kBwOverusing) {
    rtc::Optional<uint32_t> incoming_bitrate_bps =
        incoming_bitrate_.Rate(now_ms);
    if (incoming_bitrate_bps &&
        (prior_state != kBwOverusing ||
         remote_rate_->TimeToReduceFurther(now_ms, *incoming_bitrate_bps))) {
      // The first overuse should immediately trigger a new estimate.
      // We also have to update the estimate immediately if we are overusing
      // and the target bitrate is too high compared to what we are receiving.
      UpdateEstimate(now_ms);
    }
  }
}
```

简单流程图：
![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/webrtc_bwe.png)
