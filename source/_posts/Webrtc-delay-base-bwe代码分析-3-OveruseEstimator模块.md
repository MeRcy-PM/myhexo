---
title: 'Webrtc delay-base-bwe代码分析(3): OveruseEstimator模块'
date: 2017-05-22 19:32:48
tags: [webrtc, 拥塞控制]
---

该模块是一个卡尔曼滤波，根据当前到达时间差和传输大小的值，对到达时间差进行滤波，计算更精准的到达时间差。

# 0. 卡尔曼滤波基础公式

从[参考文档][1]中获得基础公式及对应变量意义。

公式：

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/Kalman_formula.png)
变量：

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/kalman_argument.png)

# 1. OveruseEstimator的卡尔曼滤波公式

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/kalman_analyze.png)

![](https://raw.githubusercontent.com/MeRcy-PM/PIC/master/webrtc/webrtc_formula.png)

# 2. 代码分析

## 2.1 update

```cpp
void OveruseEstimator::Update(int64_t t_delta,
                              double ts_delta,
                              int size_delta,
                              BandwidthUsage current_hypothesis) {
  const double min_frame_period = UpdateMinFramePeriod(ts_delta);
  const double t_ts_delta = t_delta - ts_delta;
  double fs_delta = size_delta;

  ++num_of_deltas_;
  if (num_of_deltas_ > kDeltaCounterMax) {
    num_of_deltas_ = kDeltaCounterMax;
  }

  // Update the Kalman filter.
  // 误差矩阵
  // 加上协方差，为配置初始化值。
  // process_noise_ Q
  // P(k) = AP(k - 1)AT + Q
  // 预测方程2
  // 2.3
  E_[0][0] += process_noise_[0];
  E_[1][1] += process_noise_[1];

  if ((current_hypothesis == kBwOverusing && offset_ < prev_offset_) ||
      (current_hypothesis == kBwUnderusing && offset_ > prev_offset_)) {
    E_[1][1] += 10 * process_noise_[1];
  }

  // h观测矩阵
  // 2.2
  const double h[2] = {fs_delta, 1.0};
  // P * h转置
  // 2.4
  const double Eh[2] = {E_[0][0]*h[0] + E_[0][1]*h[1],
                        E_[1][0]*h[0] + E_[1][1]*h[1]};

  // t_ts_delta 为 zk，residual为 zk - H * xk
  // 2.7 
  const double residual = t_ts_delta - slope_*h[0] - offset_;

  const bool in_stable_state = (current_hypothesis == kBwNormal);
  const double max_residual = 3.0 * sqrt(var_noise_);
  // We try to filter out very late frames. For instance periodic key
  // frames doesn't fit the Gaussian model well.
  if (fabs(residual) < max_residual) {
    UpdateNoiseEstimate(residual, min_frame_period, in_stable_state);
  } else {
    UpdateNoiseEstimate(residual < 0 ? -max_residual : max_residual,
                        min_frame_period, in_stable_state);
  }

  // denom = h * P * ht + R，var_noise为测量噪声协方差
  // 2.5
  const double denom = var_noise_ + h[0]*Eh[0] + h[1]*Eh[1];

  // K卡尔曼增益矩阵
  // K(k) = Eh / (HPkHT + R) 
  // 校正方程3
  // 2.6
  const double K[2] = {Eh[0] / denom,
                       Eh[1] / denom};

  // Ikh I-Kh
  // 2.9
  const double IKh[2][2] = {{1.0 - K[0]*h[0], -K[0]*h[1]},
                            {-K[1]*h[0], 1.0 - K[1]*h[1]}};
  const double e00 = E_[0][0];
  const double e01 = E_[0][1];

  // Update state.
  // 使用IKh更新误差矩阵
  // P(k) = (I - Kh)P(k - 1)
  // 校准方程5
  // 2.10
  E_[0][0] = e00 * IKh[0][0] + E_[1][0] * IKh[0][1];
  E_[0][1] = e01 * IKh[0][0] + E_[1][1] * IKh[0][1];
  E_[1][0] = e00 * IKh[1][0] + E_[1][0] * IKh[1][1];
  E_[1][1] = e01 * IKh[1][0] + E_[1][1] * IKh[1][1];

  // The covariance matrix must be positive semi-definite.
  bool positive_semi_definite = E_[0][0] + E_[1][1] >= 0 &&
      E_[0][0] * E_[1][1] - E_[0][1] * E_[1][0] >= 0 && E_[0][0] >= 0;
  assert(positive_semi_definite);
  if (!positive_semi_definite) {
    LOG(LS_ERROR) << "The over-use estimator's covariance matrix is no longer "
                     "semi-definite.";
  }

  // 2.8
  slope_ = slope_ + K[0] * residual;
  prev_offset_ = offset_;
  offset_ = offset_ + K[1] * residual;
}
```

## 2.2 噪声更新

```cpp
// 这个函数主要提供下面函数需要用到的ts_delta，这里不是直接使用
// 本次的ts_delta，而是根据历史时间窗口中最小时间值作为噪声估计中的ts_delta
// 这个函数就是在一个固定的时间窗口中按时间顺序存放每一个ts_delta
// 当数量超过时就从最早的时间开始pop
// 然后遍历整个时间窗口选择最小的时间差值。
double OveruseEstimator::UpdateMinFramePeriod(double ts_delta) {
  double min_frame_period = ts_delta;
  if (ts_delta_hist_.size() >= kMinFramePeriodHistoryLength) {
    ts_delta_hist_.pop_front();
  }
  std::list<double>::iterator it = ts_delta_hist_.begin();
  for (; it != ts_delta_hist_.end(); it++) {
    min_frame_period = std::min(*it, min_frame_period);
  }
  ts_delta_hist_.push_back(ts_delta);
  return min_frame_period;
}

void OveruseEstimator::UpdateNoiseEstimate(double residual,
                                           double ts_delta,
                                           bool stable_state) {
  // 仅在Normal状态下更新噪声值。
  if (!stable_state) {
    return;
  }
  // Faster filter during startup to faster adapt to the jitter level
  // of the network. |alpha| is tuned for 30 frames per second, but is scaled
  // according to |ts_delta|.
  double alpha = 0.01;
  if (num_of_deltas_ > 10*30) {
    alpha = 0.002;
  }
  // Only update the noise estimate if we're not over-using. |beta| is a
  // function of alpha and the time delta since the previous update.
  // 时间差越大，残差的比重越小
  // 由于上面函数控制一个时间窗口，因此一个时间窗口内，其权重基本固定。
  const double beta = pow(1 - alpha, ts_delta * 30.0 / 1000.0);
  // 更新均值
  avg_noise_ = beta * avg_noise_
              + (1 - beta) * residual;
  // 利用方差更新噪声值。
  var_noise_ = beta * var_noise_
              + (1 - beta) * (avg_noise_ - residual) * (avg_noise_ - residual);
  if (var_noise_ < 1) {
    var_noise_ = 1;
  }
}
```

# 参考文档
[1] [卡尔曼滤波简介](http://www.cnblogs.com/jcchen1987/p/4371439.html)

[1]: [http://www.cnblogs.com/jcchen1987/p/4371439.html]


