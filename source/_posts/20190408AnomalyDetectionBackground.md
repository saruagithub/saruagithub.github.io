---
title: 20190408AnomalyDetectionBackground
date: 2019-04-08 12:35:30
tags:
- AIOps
- 异常检测
categories:
- AIOps
---

### 1 背景

#### 1.1 企业背景

分布式系统结构的广泛应用。具有高并发，低时延，高可靠性等特点，但同时由于需求的增长，其规模，复杂性和动态生成的数据也急剧增加，这使其可靠性降低。为了避免系统故障，因此异常检测故障预判很重要。

简单来说目前的一些应用痛点，也是我企业调研的结果：

1，测试人员时间有限，不能有效测试，全覆盖测试。系统BUG是难免的。

2，系统故障后排查困难，需要及时定位。

3，运维人员希望可以提前预测故障，越早越好，从而进行排查。

4，目前企业的监控数据是有的，如何利用起来对系统更好的运维。



#### 2.1 研究背景

流数据的异常检测难点有：

1，流数据高速实时产生  ，传统的对整个数据集离线学习很难。

2，异常行为很少发生，异常检测器训练困难，难以学习对于重要的不平衡数据集的满意模型。

3，流数据的时变特性。两类异常，空间异常和上下文异常。概念漂移问题。

4，Precision与Recall之前的权衡问题。

5，不同的时序数据有不同属性。周期性，平稳性，非平稳性等等性质，对不同的方法有要求。

6，异常数据的标记很难得。

7，提前检测很重要，也很困难。



#### 2.2 智能运维背景

于是Gartner首先推出了人工智能运算（AIOP，这个方向国内还有清华大学的裴丹老师），包括性能监视，异常检测和系统故障检测任务等。理想的智能运维具有以下能力：历史数据管理、流数据（即时序数据）管理、日志数据提取、网络数据提取、性能数据提取、文本数据提取、自动化模型的发现和预测、异常检测、根因分析、按需交付等  

> AIOps is the application of artificial intelligence for IT operations. It is the future of ITOps, combining algorithmic and human intelligence to provide full visibility into the state and performance of the IT systems that businesses rely on.

性能监控里包括了对系统的CPU、memory，storage，网络，进程等资源使用的监控信息。通过对性能监控的时间序列进行异常检测，发现故障之前的征兆。



### Reference

1, “Everything you need to know about AIOps”, from https://www.moogsoft.com/resources/aiops/guide/everything-aiops/ (retrieved as of Feb. 12, 2019)