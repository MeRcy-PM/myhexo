---
title: 'Webrtc delay-base-bwe代码分析(1): RateStatistics模块'
date: 2017-05-22 19:18:59
tags: [webrtc, 拥塞控制]
---

RateStatistics这个类的作用为记录一个时间窗口内的速率值，并返回当前时间区域内的码率值。单独开一个文章主要是用来描述其用来记录速率值的桶，一开始看的比较迷糊。

```cpp
class RateStatistics {
  // Counters are kept in buckets (circular buffer), with one bucket
  // per millisecond.
  // 每毫秒一个记录值。
  struct Bucket {
    size_t sum;      // Sum of all samples in this bucket.
    size_t samples;  // Number of samples in this bucket.
  };
  std::unique_ptr<Bucket[]> buckets_;

  // 统计总值，一个为总的传输字节数，一个为这个单位时间内，总的采样点数。
  // Total count recorded in buckets.
  size_t accumulated_count_;

  // The total number of samples in the buckets.
  size_t num_samples_;

  // 某个采样时间内的时间起点。包括时间值和索引值。
  // Oldest time recorded in buckets.
  int64_t oldest_time_;

  // Bucket index of oldest counter recorded in buckets.
  uint32_t oldest_index_;

  // 类的参数值
  // To convert counts/ms to desired units
  const float scale_;

  // The window sizes, in ms, over which the rate is calculated.
  // 最大的时间窗口，单位为毫秒，每一毫秒都有一个独立的Bucket，形成一个桶，桶的索引以时间计算。
  const int64_t max_window_size_ms_;
  int64_t current_window_size_ms_;
};
```

数据结构主要分为三部分：

- 总的统计值，包括当前时间窗口内总的传输字节数和总的采样点个数。
- 当前时间窗口内的索引的初值，类似array[0]一样的存在，记录这个时间窗口内的第一个时间点及其对应的索引。
- 设置参数。

简单示例:

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/rate_statistics_ring.png)

如上图中，假设原来基准点为t1, 窗口值为1000ms，此时的时间窗口范围为[t1, t1 + 999ms]，此时新来的包将时间窗口的范围更新了，更新为[t1 + 3ms, t1 + 1002ms]，此时需要从t1及对应的index开始遍历，将所有超过时间窗口下限的元素抛弃，此时新的时间窗口为[t1 + 3ms - 1000, t1 + 3ms]，每一个bucket对应的时间点值也发生变化了(物理含义)。而对于插入一个节点则容易的多，只需要将该点的时间减去当前时间下限，其差值即为Bucket对应的下标。

原理弄明白了，代码也比较简单。

```cpp
void RateStatistics::Update(size_t count, int64_t now_ms) {
  if (now_ms < oldest_time_) {
    // Too old data is ignored.
    return;
  }

  // 每次更新值之前，需要更新整个窗口的时间区间。
  // 保证索引有效
  EraseOld(now_ms);

  // First ever sample, reset window to start now.
  if (!IsInitialized())
    oldest_time_ = now_ms;

  // now_ms在桶中的位置由基准时间(oldest_time)和自身时间的毫秒差作为索引。
  uint32_t now_offset = static_cast<uint32_t>(now_ms - oldest_time_);
  RTC_DCHECK_LT(now_offset, max_window_size_ms_);
  uint32_t index = oldest_index_ + now_offset;
  if (index >= max_window_size_ms_)
    index -= max_window_size_ms_;
  // 更新统计值
  buckets_[index].sum += count;
  ++buckets_[index].samples;
  accumulated_count_ += count;
  ++num_samples_;
}

rtc::Optional<uint32_t> RateStatistics::Rate(int64_t now_ms) const {
  // Yeah, this const_cast ain't pretty, but the alternative is to declare most
  // of the members as mutable...
  const_cast<RateStatistics*>(this)->EraseOld(now_ms);

  // If window is a single bucket or there is only one sample in a data set that
  // has not grown to the full window size, treat this as rate unavailable.
  // 真正有效的时间区间。
  int64_t active_window_size = now_ms - oldest_time_ + 1;
  // 采样点数不足时候返回无效值。
  if (num_samples_ == 0 || active_window_size <= 1 ||
      (num_samples_ <= 1 && active_window_size < current_window_size_ms_)) {
	// 返回一个无效值
    return rtc::Optional<uint32_t>();
  }

  // 码率的因子，最后的值为（总大小 * scale_ / 总的有效时间）
  float scale = scale_ / active_window_size;
  return rtc::Optional<uint32_t>(
      static_cast<uint32_t>(accumulated_count_ * scale + 0.5f));
}

void RateStatistics::EraseOld(int64_t now_ms) {
  if (!IsInitialized())
    return;

  // New oldest time that is included in data set.
  // 根据窗口大小更新当前时间窗口，更新为[now_ms - 窗口， now_ms]
  int64_t new_oldest_time = now_ms - current_window_size_ms_ + 1;

  // New oldest time is older than the current one, no need to cull data.
  if (new_oldest_time <= oldest_time_)
    return;

  // Loop over buckets and remove too old data points.
  // 从当前时间窗口下限开始逐毫秒遍历
  while (num_samples_ > 0 && oldest_time_ < new_oldest_time) {
    const Bucket& oldest_bucket = buckets_[oldest_index_];
    RTC_DCHECK_GE(accumulated_count_, oldest_bucket.sum);
    RTC_DCHECK_GE(num_samples_, oldest_bucket.samples);
	// 从统计数据中将超时的Bucket中的数据扣除，并清空Bucket。
    accumulated_count_ -= oldest_bucket.sum;
    num_samples_ -= oldest_bucket.samples;
    buckets_[oldest_index_] = Bucket();
	// 环形
    if (++oldest_index_ >= max_window_size_ms_)
      oldest_index_ = 0;
    ++oldest_time_;
  }
  oldest_time_ = new_oldest_time;
}
```


