---
title: Unsupervised Anomaly Detection via Variational Auto-Encoder for Seasonal KPIs in Web Applications (Donut model)
date: 2020-03-13 10:54:13
mathjax: true
tags:
- AIOps
- 论文
- AnomalyDetection
categories:
- AIOps
---



### ABS&Intro

#### Abstract

大的互联网公司一般会密切监控KPIs。然而这些有不同模式不同数据质量的季节性KPIs的异常检测很有挑战，特别是无标签。本文中，我们提出了Donut，基于VAE（Variational Auto-Encoder 变分自动编码器）的无监督异常检测算法，其最佳F-score达到0.75 ~ 0.9。我们为Donut的重构提出了一种新颖的KDE解释，使其成为第一种基于VAE的异常检测算法，并且具有扎实的理论解释。

#### KPIs时间序列数据

度量指标如页面浏览量，在线用户数量，订单数量等等。在所有的这些KPIs里，本文主要关心一些商业相关的KPIs，这些受用户行为计划所影响，因此大致表现出有规律的季节性模式（如按天，按周等）。

检测KPI异常的相关文献【1, 2, 5– 8, 17, 18, 21, 23–27, 29, 31, 35, 36, 40, 41】 很丰富。现有的异常检测算法遭受算法挑选/参数调整的麻烦，严重依赖标签，性能不令人满意和/或缺乏理论基础。

#### 本文

本文提出的Donut，基于VAE（代表性的深层生成模型）的无监督异常检测算法，伴有理论解释，可以无标签或偶尔提供的标签下学习。

本文贡献：

1，Donut里的三项技术：改进的ELBO，缺失数据注入进行训练以及为了检测的MCMC (imputation)借补法。使它大大超越了最新的监督类和基于VAE的异常检测算法。 对于来自顶级全球互联网公司的研究KPI，无监督Donut的最佳F-score在0.75到0.9之间。

2，在文献中，我们首次发现采用VAE（或一般而言的生成模型）进行异常检测需要对正常数据和异常数据进行训练，这与通常的直觉相反。

3，我们为Donut在z空间中提出了一种新颖的KDE解释，使其成为第一个基于VAE的具有可靠理论解释的异常检测算法。这种解释可能有益于异常检测中其他深度生成模型的设计。 我们发现了潜在z空间中的时间梯度效应，很好地说明了Donut在检测季节性KPI异常方面的出色性能。



### Problem

#### KPIs的异常检测

商业KPIs一般由季节性，另一方面在每个重复周期中，KPI曲线的形状并不完全相同，因为用户行为可能会随着天变化。这里我们命名KPIs为“具有局部变化的季节性KPIs” （如图1）

![20200313Donut_Figure1](/images/20200313Donut_Figure_KPIs.jpg)

另一类局部变化是随着日数增加的趋势，可以通过Holt-Winters [41]和时间序列分解[6]来确定。 除非正确处理了这些局部变化，否则异常检测算法可能无法正常工作。

除了季节性，局部变化variation，这些KPIs也有噪声noise。我们假设噪声独立，在每个点服从0均值高斯分布。高斯噪声的精确值是没有意义的，因此，我们仅关注这些噪声的统计数据，即噪声的方差。

现在我们可以将季节性KPI的“正常模式”形式化为两个组成部分的组合：（1）具有局部变化的季节性模式（2）高斯噪声的统计数据。

我们使用Anomaly表示不遵循正常模式的（突然的尖峰和骤降）数据点，abnormal表示Anomaly和缺失点。本文主要检测Anomalies。

由于操作员需要处理异常以进行故障排除/缓解，因此有些异常会被标上标签。 请注意，此类偶然标签对异常的覆盖范围与典型的监督学习算法所需要的相去甚远。

异常检测算法一般对$x_{t-T+1}, \ldots, x_{t}$ 计算$p\left(y_{t}=1 | x_{t-T+1}, \ldots, x_{t}\right)$ ，操作员只需选择阈值判断异常。



#### 2.3的Problem Statement

（挪到前面来了）在我们的上下文中，尽管远不完整，但偶尔还是可以使用标签，应该以某种方式加以利用。本文的问题陈述如下：

我们致力于<u>基于深度生成模型且具有可靠理论解释的无监督异常检测算法，该算法可以利用偶尔可用的标签</u>。



### Previous Work

#### 传统统计模型

许多基于传统统计模型的异常检测器（大多是时间序列模型）[6, 17, 18, 24, 26, 27, 31, 40, 41] 已经被提出来计算异常分数。由于这些算法通常对适用的KPI有基本的假设，需要涉及专家的参考才能为给定的KPI选择合适的检测器，然后基于训练数据去微调检测器参数。这些检测器的简单集合（例如多数表决[8]和归一化[35]）不能很好地起作用。



#### 监督集成方法

为了规避传统统计异常检测器算法/参数调整的麻烦，提出了有监督的集成方法**EGADS [21]和Opprentice [25]**。 他们使用用户反馈作为标签并使用传统检测器输出的异常评分作为特征来训练异常分类器。 EGADS和Opprentice均显示出令人鼓舞的结果，但它们严重依赖于**良好的标签**（远远超过我们所积累的轶事标签），这在大规模应用中通常不可行。 此外，运行多个传统检测器以在检测期间提取特征会引入大量的计算成本，这是一个实际问题。



#### 无监督方法与深度生成模型

最近采用无监督机器学习方法是个趋势。, e.g., one-class SVM [1, 7], clustering based methods [9] like K-Means [28] and GMM [23], KDE [29], and VAE [2] and VRNN [36].  

其理念是关注正常模式而不是异常：由于KPIs通常主要由正常数据组成，因此即使没有标签也可以轻松地训练模型。 粗略地说，它们都首先识别原始或某些潜在特征空间中的“正常”区域，然后通过测量观测值与正常区域的“距离”来计算异常分数。

沿着此方向，1，学习正常模式可以看作是学习训练数据的分布，这是生成模型的主题。2，最近在利用深度学习技术训练生成模型方面取得了巨大进展（例如**GAN [13]和深度贝叶斯网络[4，39]**）。后者属于深度生成模型家族，它采用图形graphical[30]模型框架和变型variational技术[3]，以VAE [16，32]为代表。 3，尽管深度生成模型在异常检测方面具有广阔的前景，但现有的基于VAE的异常检测方法[2]并非为KPIs（时间序列）设计，在我们的设置中效果不佳（请参见§4），并且没有 为其用于异常检测的深度生成模型的设计提供支持的理论基础（请参阅§5）4，简单地采用基于VRNN的更复杂的模型[36]在我们的实验中显示出训练时间长且性能差。5，[2]假定仅对干净数据进行训练，这在我们的上下文中是不可行的。



### 变分自动编码器Variational Auto-Encoder

由于VAE是深度贝叶斯网络的基本构建块，因此我们选择开始使用VAE。

![20200313Donut_Figure2_VAE](/images/20200313Donut_Figure2_VAE.jpg)

#### VAE的背景

深度贝叶斯网络使用神经网络来表达变量之间的关系，因此它们不再局限于简单的分布族，因此可以轻松地应用于复杂的数据。 在训练和预测中经常采用变分推理技术[12]，这是由神经网络延伸出的解决后验分布的有效方法。

VAE是深度贝叶斯网络，它对两个随机变量（潜变量$z$和可见变量$x$）之间的关系进行建模。 为 $z$ 选择一个先验分布，它通常是多元单位高斯分布 $\mathcal{N}(\mathbf{0}, \mathbf{I})$ 。之后，从由参数为 $\theta$ 的神经网络提取出的 $p_{\theta}(\mathbf{x} | \mathbf{z})$ 中采样得到 $x$ 。$p_{\theta}(\mathbf{z} | \mathbf{x})$ 的准确形式由任务需求进行选择。真实的后验 $p_{\theta}(\mathbf{z} | \mathbf{x})$ 是无解析解的，但对于训练是必不可少的，并且通常对预测有用，因此变分推理技术可用于fit另外的神经网络，作为近似后验 $q_{\phi}(\mathbf{z} | \mathbf{x})$。该后验通常被假定为 $\mathcal{N}\left(\boldsymbol{\mu}_{\phi}(\mathbf{x}), \boldsymbol{\sigma}_{\phi}^{2}(\mathbf{x})\right)$ , $\boldsymbol{\mu}_{\phi}(\mathbf{x})$ 与 $\boldsymbol{\sigma}_{\phi}(\mathbf{x})$ 是由神经网络导出。（如图FIgure2）

SGVB [16, 32] 是一个伴随VAE一起用的变分推断算法，其中通过最大化变分下界evidence lower bound (ELBO Eqn1 ) 来联合训练近似后验模型和生成模型。

Equation 1:

$$\begin{aligned} \log p_{\theta}(\mathbf{x}) & \geq \log p_{\theta}(\mathbf{x})-\operatorname{KL}\left[q_{\phi}(\mathbf{z} | \mathbf{x}) \| p_{\theta}(\mathbf{z} | \mathbf{x})\right] \\ &=\mathcal{L}(\mathbf{x}) \\ &=\mathbb{E}_{\boldsymbol{q}_{\phi}(\mathbf{z} | \mathbf{x})}\left[\log p_{\theta}(\mathbf{x})+\log p_{\theta}(\mathbf{z} | \mathbf{x})-\log q_{\phi}(\mathbf{z} | \mathbf{x})\right] \\ &=\mathbb{E}_{\boldsymbol{q}_{\phi}(\mathbf{z} | \mathbf{x})}\left[\log p_{\theta}(\mathbf{x}, \mathbf{z})-\log q_{\phi}(\mathbf{z} | \mathbf{x})\right] \\ &=\mathbb{E}_{\boldsymbol{q}_{\phi}(\mathbf{z} | \mathbf{x})}\left[\log p_{\theta}(\mathbf{x} | \mathbf{z})+\log p_{\theta}(\mathbf{z})-\log q_{\phi}(\mathbf{z} | \mathbf{x})\right] \end{aligned}$$

蒙特卡洛积分常常被用于近似 Eqn 1 中的期望。如Eqn 2，$\mathbf{z}^{(l)}, l=1 \ldots L$ 是来自 $q_{\phi}(\mathbf{z} | \mathbf{x})$ 的样例samples。整个本文中，我们都坚持使用这种方法。

Equation 2：

$$\mathbb{E}_{\boldsymbol{q}_{\phi}(\mathbf{z} | \mathbf{x})}[f(\mathbf{z})] \approx \frac{1}{L} \sum_{l=1}^{L} f\left(\mathbf{z}^{(l)}\right)$$



### Architecture结构

Donut的总体架构如图Figure3，三个关键技术是Modified ELBO，训练时的Missing Data Injection， 检测时的MCMC Imputation。

![20200314Arch_Donut](/images/20200314Arch_Donut.jpg)

![20200314Network_Donut](/images/20200314Network_Donut.jpg)



#### Network Structure

VAE不是一个序列模型，因此我们在KPI上应用长度为 $w$ 的**滑动窗口**[34]。 对于每一个 $x_t$ 使用 $x_{t-W+1}, ..., x_{t}$ 作为VAE的 $X$  向量。该滑动窗口由于其简单性而首先被采用，实际上却带来了重要而有益的结果。

Donut的总体网络结构如图4，其中有双线轮廓的组件 (e.g., Sliding Window x, W Dimensional at bottom left) 是我们的新设计，其余组件来自标准VAEs。先验 $p_{\theta}(z)$ 被选择为 $\mathcal{N}(\mathrm{0}, \mathrm{I})$ 。 x和z后验均选择为对角高斯 diagonal Gaussian：

$$p_{\theta}(\mathbf{x} | \mathbf{z})=\mathcal{N}\left(\boldsymbol{\mu}_{\mathbf{x}}, \boldsymbol{\sigma}_{\mathbf{x}}^{2} \mathbf{I}\right) $$

$$q_{\phi}(\mathbf{z} | \mathbf{x})=\mathcal{N}\left(\boldsymbol{\mu}_{\mathbf{z}}, \boldsymbol{\sigma}_{\mathbf{z}}^{2} \mathbf{I}\right)$$

这里的$\mu_x,\mu_z,\sigma_x,\sigma_z$ 是每个独立高斯组件的均值和标准差。 $z$ 是 K 维的。通过分离的隐藏层$f_{\phi}(\mathbf{x}) , f_{\theta}(\mathbf{z})$ ，从 $x$ 和 $z$ 隐藏特征被提取出来。然后从隐藏特征中得出x和z的高斯参数。 

均值是从线性层导出：

$$\boldsymbol{\mu}_{\mathbf{X}}=\mathbf{W}_{\boldsymbol{\mu}_{\mathbf{X}}}^{\top} f_{\theta}(\mathbf{z})+\mathbf{b}_{\boldsymbol{\mu}_{\mathbf{X}}}$$

$$\boldsymbol\mu_{z}=\mathbf{W}_{\mu_{z}}^{\top} f_{\phi}(\mathbf{x})+\mathbf{b}_{\mu_{z}}$$

标准差是从soft-plus层导出，加上一个非负小数 $\epsilon$ ：

$$\sigma_{\mathrm{x}} = \text{SoftPlus} \left[\mathbf{W}_{\sigma_{x}}^{\top} f_{\phi}(\mathbf{x})+\mathbf{b}_{\sigma_{x}}\right]+\epsilon$$

$$\sigma_{\mathrm{z}}=\text{SoftPlus} \left[\mathbf{W}_{\boldsymbol{\sigma}_{\mathbf{z}}}^{\top} f_{\phi}(\mathbf{z})+\mathbf{b}_{\boldsymbol{\sigma}_{\mathbf{z}}}\right]+\boldsymbol{\epsilon}$$

这里的 $\text{SoftPlus}[a] = log[exp(a) + 1]$ ，这里介绍的所有W-s和b-s是相应层的参数。 注意，将标量函数 $f(x)$ 应用于向量 $x$ 时，意味着将其应用于每个部分component。







### Reference

1，Xu H, Chen W, Zhao N, et al. Unsupervised anomaly detection via variational auto-encoder for seasonal kpis in web applications[C]//Proceedings of the 2018 World Wide Web Conference. 2018: 187-196.