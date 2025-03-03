+++
title = 'CMU 10-414/714: Deep Learning Systems (2020) - 深度学习系统 02 Neural Networks'
date = 2025-02-10T14:58:18-05:00
draft = false
tags = ['Learning Notes', 'Deep Learning']
+++

## "Manual" Neural Networks / Backprop

> 还是复习 ML 的内容

- From linear to nonlinear hypothesis classes
- Neural networks
- Backpropagation (i.e., computiing gradients)

> hypothesis classes 是用于模型训练的函数类型
>
> 线性假设类、非线性假设类
>
> 多项式回归或神经网络等模型都属于非线性假设类，能捕捉更复杂的关系
>
> 以下是一个两层神经网络的简化表示：
>
> $$h_\theta(x) = \theta_2^T \sigma(W_1^T x)$$
>
> 其中，$W_1$ 和 $W_2$ 是权重矩阵，$\sigma$是非线性激活函数，如 ReLU 或 sigmoid。

## nonlinear hypothesis classes

linear hypothesis classes: $h\_\theta(x) = \theta^T x$

> 线性的分类可能可以 fit 一些情况，但如果类型是分散的圆圈等等情况，可能难以拟合？

One idea: apply a linear classifier to some (potentially higher-dimensional) features of the data: $h_\theta(x) = \theta^T \phi(x), \theta \in R^{d\times k}, \phi: R^n \rightarrow R^d$

> phi 把 n 维的输入变成 d 维的
>
> 隐藏层节点的激活函数（如 ReLU）可以视为一种非线性特征映射，这使得神经网络能够捕捉到数据中的复杂非线性关系。

### How do we create features?

1. manual engineering, the “old” way of doing machine learning
2. In a way that itself is learned from data, the “new” way of doing ML

> 传统机器学习，手动提取特征比如房价预测里的面积、房间特征

### Neural networks / deep learning

Neural Network, a particular type of hypothesis class, multiple, parameterized differentiable functions (a.k.a. “layers”) composed together

Deep Network, synonym for "neural network", composing together a lot of functions, so “deep” is typically an appropriate qualifier

> 深度学习，指的就是用神经网络的机器学习？

two layer neural network

$h_\theta(x) = W^T_2\sigma(W_1^Tx)$

where $\sigma: R \rightarrow R$ is nonlinear function like ReLU or sigmoid

> $\sigma$ 非线性函数, 比如 ReLU，sigmod
>
> $\theta$ 可以看作是参数

batch matrix form:

$$
h_\theta(X) = \sigma(X W_1) W_2
$$

> 为什么加一个 non linear 就可以表示这么多结果？
>
> 数学上：神经网络已经被证明为通用函数逼近器（Universal Function Approximators）
>
> 理论上，只要网络结构足够复杂（隐藏单元和层数足够多），它就能近似任何连续函数。
>
> 之前学过一点数值计算，当时还没接触到非线性函数可以拟合的函数，基本都是线性/指数或者分段/平滑等等，应该继续学一下非线性拟合的

## Fully-connected deep networks

𝐿-layer neural network – a.k.a. "Multi-layer perceptron" (MLP)

$$Z_{i+1} = \sigma_i(Z_i W_i), \quad i = 1, \dots, L$$

$$h_\theta(X) = Z_{L+1}$$

> 每一层的输入都是上一层的 非线性函数 (输出 x 权重矩阵)

## why deep networks?

work like the brain?

parity

> 奇偶性无法学习？

empirically it seems like they work better for a fixed parameter count

> 多层结构/深层网络能更均匀地分布参数

## Backpropagation

neural networks:

- Hypothesis Class: MLP
- Loss function: cross-entropy loss
- Optimization procedure: SGD

$$\min_\theta \frac{1}{m} \sum_{i=1}^m \ell_{ce}(h_\theta(x_i), y_i) $$

> 我知道这些组件，但具体如何运作？

### The gradient(s) of a two-layer network

$$
 \nabla_{\{W_1, W_2\}} \ell_{ce}(\sigma((XW_1)W_2), y) = \sigma(XW_1)^T \cdot (S - I_y)
$$

> 链式法则（chain rule）计算偏导数
>
> 偏导数是多变量函数对其中一个变量的变化率，而保持其他变量固定不变
>
> 梯度是多变量函数所有偏导数组成的向量，梯度的方向用于描述函数值增长最快的路径，负方向是函数值下降最快的路径
>
> 对多元函数，假设有个初始点 $(x_0, y_0)$ 可以求出梯度，更新其值 $x_1 = x_0 - \eta \cdot \frac{\partial f}{\partial x}$ 其中 $\eta$ 是学习率，一旦梯度足够小，就可以减少原函数的值，也就是损失函数的值减少

## Backpropagation “in general”

consider our fully-connected network

$$
    Z\_{i+1} = \sigma_i(Z_i W_i), i = 1, ..., L \\
     \frac{\partial \ell}{\partial W_i} = \frac{\partial \ell}{\partial Z_{L+1}} \cdot \frac{\partial Z_{L+1}}{\partial Z_L} \cdot \frac{\partial Z_L}{\partial Z_{L-1}} \cdot \ldots \cdot \frac{\partial Z_{i+2}}{\partial Z_{i+1}} \cdot G_{i+1}
$$

Then we have a simple ”backward” iteration to compute the $G_i$’s

$$
G_i = G_{i+1} \cdot \frac{\partial \ell}{\partial Z_{i+1}}
$$

### Computing the real gradients

$$
\nabla_{W_i} \ell = Z_i^T \cdot (G_{i+1} \circ \sigma_i'(Z_i W_i))
$$

> 链式法则矩阵计算是反向传播（Backpropagation）的核心思想。它利用链式法则（Chain Rule）和矩阵计算，逐层向后传递误差
>
> 其中 $Z_i$ 是可以复用的

### Backpropagation: Forward and backward passes

> 为什么这些计算是“同时”发生的？
>
> - 前向传播的过程中，我们已经缓存了每一层的输出 $Z_i$ 和激活函数的导数 $\sigma'(Z_i W_i)$
> - 后向传播时，利用这些缓存值直接进行梯度计算，避免重复计算中间值。

What is really happening with the backward iteration?

$$
\frac{\partial \ell}{\partial W_i} = \frac{\partial Z_{i+1}}{\partial W_i} \cdot \frac{\partial \ell}{\partial Z_{L+1}} \cdot \frac{\partial Z_{L+1}}{\partial Z_L} \cdot \ldots \cdot \frac{\partial Z_{i+2}}{\partial Z_{i+1}}
$$

Each layer needs to be able to multiply the “incoming backward” gradient $G_{i+1}$ by
its derivatives $\frac{\partial Z_{i+1}}{\partial W_i}$ an operation called the “vector Jacobian product”

automatic differentiation

> 每一层的梯度 $G_i$ 是从后一层的梯度 $G_{i+1}$ 递归计算得到。
>
> 向量雅可比乘积
>
> 反向传播可以推广到任意计算图，成为实现自动微分的基础
