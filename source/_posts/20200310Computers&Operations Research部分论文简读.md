---
title: Computers&Operations Research部分论文简读
date: 2020-03-10 09:58:02
tags:
- AIOps
- 故障检测
- Operation Research
categories:
- AIOps
---



## Exact and heuristic approaches to detect failures in failed k-out-of-n systems

### ABS&Intro

#### 背景

本文考虑n个系统中k个故障了（表决系统），相应的测试每个组件是有成本的。另外，我们具有某些组件是故障的原因的先验概率信息。目标是以最小的预期成本去识别导致故障的那部分组件。

#### 本文工作

提出了精确与近似的策略，在故障表决系统（k-out-of-n）中检测组件状态。我们提出两种整数规划编程公式，两种基于Markov决策过程（MDP）的新颖方法以及两种启发式算法。展示了精确式算法的限制以及启发式算法在随机产生的测试例子的有效性。尽管CPU时间更长，整数规划更灵活地整合更多约束 restriction，例如必要时进行测试优先级关系。数值结果表明，针对所提出的MDP模型进行动态编程是最有效的精确方法，在一小时内最多可解决12个组件。 针对小到中级测试实例，启发式算法的性能是对比精确式算法给的，并针对高级测试实例给出下限。

#### 简介

系统越来越复杂，组件、感应器，子系统越来越多。为了系统更加可靠，系统中总会有冗余存在。当组件故障时，整个系统也可能fail，需要尽快恢复。两个主要问题：(1) 是否系统工作和失败（序列测试问题） (2) 故障原因（failure detection故障检测问题）。在这两种问题中，可行的解决方法可以被描述为二分决策树，目标是最小化期望成本。

difference区别：故障检测问题中测试结果的概率随着测试执行而变化。但在序列测试问题中则是不变的。另一个区别是，在故障检测问题中，输出是导致故障的一组组件，而在序列测试问题中，输出是系统正在运行或发生故障的信息以及该状态的证明。

#### 研究综述

序列测试问题和故障检测问题：Chang 研究在序列测试的上下文中以最小的成本诊断电子晶体，并提供多项式时间的精确式算法。B K 提出了表决系统以在核反应堆子系统中提供冗余，以实现可靠的运行。W提出当测试不完美并且测试有优先限制时的启发式算法。Ba分析某些维护策略的长期平均成本。Gar蚁群优化算法用于计算机网络中的故障定位。

故障检测问题的灵异研究领域：离散搜索问题，旨在找到隐藏在N个盒子中的一个item，并且其预期成本最小。检查盒子会很昂贵，并且已知该item在盒子内的概率是先验的。K对搜索问题提出最优贪心算法，当仅可能出现假阳性结果时。W&D考虑一个变体，当存在简单的优先级约束并且路径依赖关系由组活动定义时。

#### 贡献

1，我们引入和研究了k-out-of-n系统的故障检测问题，将文献中研究的n-out-of-n系统归纳了下来。

2，我们提供了四种精确的两种启发式方法来解决该问题，并提供了两种下限lower bound方案，用于在较大的情况下进行基准测试。

3，首次提出整数规划建模和马尔可夫决策过程来解决此类问题。

4，我们进行数值实验以评估不同方法的有效性。



（暂时了解背景和introduction，未读完）



## A survey of models and algorithms for emergency response logistics in electric distribution systems. Part I: Reliability planning with fault considerations

### ABS&Intro

配电系统的应急响应设计一系列在可靠性和应急计划级别的决策问题。这些操作包括故障诊断，故障定位，故障隔离，恢复和修复。本文回顾了针对与配电运行相关的故障考虑的可靠性规划问题的优化模型和解决方法。本文对确定配电变电站单故障容量，重新分配超负荷，配置配电系统，将地理区域划分为服务区域以及定位物料仓库和仓库的研究进行了调查。

规划应急响应的操作涉及许多决策问题，可以使用运筹学方法论来解决。 故障情况可能会导致配电系统服务中断的“极端”状态，从而降低服务质量并给电力公司造成经济损失。eg 2008年1月在中国中东部和南部地区的暴风雪使几个省的电线和电线杆倒塌，影响了中国近三分之二的土地，估计造成了100亿美元的直接经济损失。但应急分配响应研究少。

由于网络拓扑结构，操作能力和应用的操作设备等特性的差异，规划人员面临的问题非常复杂，并且因地而异。 在过去的二十年中，文献中已经出现了越来越多的运筹学应用程序用于应急分配响应。配电系统中涉及的大量组件，配电网络的复杂性以及公用事业运营这些网络的能力不断提高，所有这些都促使人们在配电公用事业的各个层面上使用优化技术。



## Application of Optimized Machine Learning Techniques for Prediction of Occupational Accidents

### ABS&Intro

机器学习在职业安全领域中预测事故的探索几乎是新的。





### Reference

1，Yavuz T, Kundakcioglu O E, Ünlüyurt T. Exact and heuristic approaches to detect failures in failed k-out-of-n systems[J]. Computers & Operations Research, 2019, 112: 104752.

2，A survey of models and algorithms for emergency response logistics in electric distribution systems. Part I: Reliability planning with fault considerations









