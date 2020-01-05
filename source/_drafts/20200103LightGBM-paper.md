---
title: 20200103LightGBM_paper
date: 2020-01-03 21:16:46
mathjax: true
tags:
- 机器学习
categories:
- 机器学习
---

### 1 摘要简介

#### 1.1 简介

GBDT的实现有XGBoost，pGBRT等。但当特征维度高，数据集size大的时候有效性还不够。主要原因在于对每一个特征，都要扫描所有实例并估计所有可能的划分节点的信息增益。

LightGBM提出的方法是：Gradient-based one-side sampling (GOSS) ，Exclusive feature bundling (EFB)。

#### 1.2 GOSS

GOSS的作用排除相当一部分小的梯度的数据实例（instance），只使用剩下的来估计信息增益。**更大的梯度的数据实例在计算信息增益上起到更重要的作用**。

GBDT中没有数据实例权重。但具有不同梯度的数据实例在信息增益的计算中起着不同的作用。根据信息增益的定义，<u>那些具有较大梯度的实例（即训练不足的实例）将为信息增益做出更大的贡献</u>。下采样那些梯度大的实例（阈值or百分比），并随机丢弃那些小的梯度的实例。

#### 1.3 EFB的作用

EFB的作用：捆绑互斥特征（即，他们很少同时采用非零值）贪心策略来找最优互斥特征。

现实中，特征空间稀疏。这为我们提供了一种设计几乎无损方法以减少有效特征数量的可能性。在稀疏特征空间中，许多特征几乎是互斥的，即他们很少同时采用非零值（文本挖掘里one hot词表征）。我们可以很安全的捆绑这些互斥特征。

设计了一个以恒定近似比率的贪心算法将最优特征捆绑问题转换为图着色问题。通过将特征作为顶点并为每两个特征（如果它们之间不是互斥的话）添加边。



### 2 GBDT回顾

GBDT是决策树的集成模型。每次迭代GBDT通过拟合负梯度（残差）学习决策树。他的主要成本是在于学习决策树。划分节点的选择是非常耗时的，之前有预排序算法，它每次枚举所有可能的划分点，基于预排序的特征值，从而找到最优划分节点。

另一种是基于直方图的算法。他将连续特征值分桶为离散的bins，利用bins来构建特征直方图。（内存消耗更少，训练速度更快）。直方图建立是 O(#data * #feature)，划分节点是 O(#bin * #feature)。

实际应用中使用的大规模数据集通常很少。 带有预排序算法的GBDT可以通过忽略零值的特征来降低训练成本[13]。 但是，具有基于直方图的算法的GBDT没有有效的稀疏优化解决方案。 原因是，无论特征值是否为零，基于直方图的算法都需要为每个数据实例检索特征仓值（请参阅算法1）。

![20200104GBDT_Algo](/Users/wangxue/gitpro/20191105MyBlog/saruagithub/source/images/20200104GBDT_Algo.jpg)

直方图算法：

![20200105Histogram](/Users/wangxue/gitpro/20191105MyBlog/saruagithub/source/images/20200105Histogram.jpg)

for node in nodeSet (for all leaf p in Tc-1(x)) 这里是对当前层的叶子节点遍历。（需要遍历所有的特征，来找到增益最大的特征及其划分值，以此来分裂该叶子节点。）

for k=1 to m (for all f in X.Features) 这里是对所有特征进行遍历。**对于每个特征，为其创建一个直方图**。





### Reference

1，论文 LightGBM: A highly efficient gradient boosting decision tree.

2，[LightGBM直方图优化算法](https://blog.csdn.net/anshuai_aw1/article/details/83040541)

3，[一些面试问题](https://juejin.im/post/5d25e1d0e51d4556da53d151)

