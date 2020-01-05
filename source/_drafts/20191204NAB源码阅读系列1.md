---
title: NAB源码阅读系列1
date: 2019-12-04 10:04:59
mathjax: true
tags:
- 智能运维 
- 时间序列 
- 异常检测
- Python
categories:
- AIOps
---



### 1 NAB的基本框架

NAB里包括了几个在线时间序列异常检测器，包括了几组异常检测数据集，包括了阈值优化，评分算法，归一化检测器分数等内容，包括了NAB结果及可视化等内容。









### Reference

1, Ahmad S, Lavin A, Purdy S, et al. Unsupervised real-time anomaly detection for streaming data[J]. Neurocomputing, 2017, 262: 134-147.

2, Lavin A, Ahmad S. Evaluating Real-Time Anomaly Detection Algorithms--The Numenta Anomaly Benchmark[C]//2015 IEEE 14th International Conference on Machine Learning and Applications (ICMLA). IEEE, 2015: 38-44.

3, https://github.com/numenta/NAB NAB的开源库里有很多说明，以及白皮书