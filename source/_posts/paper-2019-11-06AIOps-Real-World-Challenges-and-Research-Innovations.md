---
title: '[paper]2019/11/06AIOps: Real-World Challenges and Research Innovations'
date: 2019-11-06 20:56:04
tags:
- AIOps
- 论文综述
categories:
- AIOps
---
### 论文信息
论文名字：AIOps: Real-World Challenges and Research Innovations
引用：Dang Y, Lin Q, Huang P. AIOps: real-world challenges and research innovations[C]//Proceedings of the 41st International Conference on Software Engineering: Companion Proceedings. IEEE Press, 2019: 4-5.

### AIOps定义 
智能运维的定义：通过AI与ML有效构建运维应用 AIOps is about empowering software and service engineers (e.g., developers, program managers, support engineers, site reliability engineers) to efficiently and effectively build and operate online services and applications at scale with artificial intelligence (AI) and machine learning (ML) techniques. 

DevOps 连续开发部署应用（来源于 G. Kim, P. Debois, et al, “The DevOps Handbook: How to Create World- Class Agility, Reliability, and Security in Technology Organizations”, IT Revolution Press, Oct. 2016）

### AIOps的三个目标
1，服务智能化
及时观察多方面变化，质量下降，成本增加，工作量增加等，基于AIOps的服务还可以根据其历史行为，工作量模式和基础来预测其未来状态。根据状态自我调整，trigger self-adaption or auto-healing behaviors of a service, with low human intervention.

思考：要监控性能，监控反应时间，问题调整策略（自动化调整）

2，较高的客户满意度
具有内置智能的服务可以了解客户的使用行为，并采取积极的行动来提高客户满意度。 例如，服务可以自动向客户推荐调整建议，以使其获得最佳性能（例如，调整配置，冗余级别，资源分配）

思考：网络不好的话如何自动调整？

3，高工程生产率
 工程师和操作员免于繁琐的工作，例如（1）从各种来源手动收集信息以调查问题； （2）解决重复出现的问题。 工程师和操作人员还可以使用AI / ML技术来学习系统行为的模式，预测服务行为和客户活动的未来，以进行必要的体系结构更改和服务适应策略更改等。
 
### challenges
整体思考，充足理解系统
工程架构转变 the AIOps engineering principles should include data/label quality monitoring and assurances, continuous model-quality validation, and actionability of insights.
缺乏label，极端失衡，数量太少，噪声程度高等，监督或半监督模型
组件服务之间的复杂依存关系


思考：还有服务变更带来的问题，新学习吗？
实时数据大量产生，怎么利用?
