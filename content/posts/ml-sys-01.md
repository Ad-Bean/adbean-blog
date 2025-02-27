+++
title = 'CMU 10-414/714: Deep Learning Systems (2020) - 深度学习系统 01 Softmax'
date = 2025-02-05T11:04:50-05:00
draft = false
tags = ['Learning Notes', 'Deep Learning']
+++

## Deep Learning Systems

https://dlsyscourse.org/

10-414/714: Deep Learning Systems

> 这学期选了 ML systems，一些 DL 的概念不太熟悉，补一下 Tianqi Chen 大神的课

## Introduction

AlexNet: Image Classification

AlphaGo, StyleGAN, GPT-3, stable diffusion ...

> DL 是深度神经网络，越深越好吗？
>
> transformer 和 GAN 也都是深度学习
>
> CNN：卷积层、池化、全连接
>
> RNN, LSTM, GAN, Transformer...

Pytorch, Tensorflow, ...

> 深度学习和传统机器学习的区别是自动微分吗？

## ML Refresher / Softmax Regression

> 回顾 ML 里的图像识别任务
>
> 其实不知道要不要先看看 ML 的基础课，很多 ML 的基础也不太记得了

Machine Learning as data-driven programming

classify handwritten drawing of digits

> computers dont see pics like us

(supervised) ML approach: train with known labels -> ML algo -> Model h

### 3 ingredients

1. hypothesis class: program structure, a set of parameters, how we map inputs (images of digits) to ouputs (labels, probabilities)
2. loss function: a function specifies how well a given hypothesis performs
3. optimization method: a procedure for determining a set of params that minimize the sum of losses over the training set

> 框架都是一样的

### softmax regression

$$ \begin{align} x^{(i)} & \in \mathbb{R}^n \\ y^{(i)} & \in \{1, \ldots, k\} \quad \text{for} \quad i = 1, \ldots, m \end{align} $$

> $\mathbb{R}$ 表示实数集（Real numbers）。因此，公式中的 $x^{(i)} \in \mathbb{R}^n $ 意思是每个数据点 $x^{(i)}$ 是一个 $n$ 维的实数向量。也就是说，$x^{(i)}$ 是一个包含 $n$ 个实数的向量，每个数据点的特征向量。
>
> 比如 $x^{(i)} = \begin{pmatrix} 2.5 \\ -1.0 \\ 3.7 \end{pmatrix} \in \mathbb{R}^3$
>
> $y^{(i)}$ 是每个数据点对应的标签
>
> $n$ 输入数据的维度（特征数量）
>
> $k$ 类别的数量，也就是 0 到 9 的数字
>
> $m$ 训练集中的数据点数量

Our hypothesis function maps inputs $x^{(i)} \in \mathbb{R}^n$ to $k$-dimensional vectors

$$ h: \mathbb{R}^𝑛 \rightarrow \mathbb{R}^k $$

$h_i(x)$ indicates some measure of “belief” in how much likely the label is to be class $i$

> h_i 表示是 class i 的“可能性”，但目前不一定是概率？

A linear hypothesis function uses a linear operator (i.e. matrix multiplication) for this transformation

> 线性假设函数，将输入 n 维输入 x 映射到 k 维
>
> 线性假设函数使用线性算子（即矩阵乘法）进行这种变换

$$
h_\theta(x) = \theta^T x
$$

> 其中 $\theta$ 是 $n\times k$ 维度的，一个权重矩阵，最后输出一个 k 维向量，每个元素对应每个类别的置信度

### Matrix batch notation

> 矩阵运算更加高效，GPU/CPU

### Loss Function

#### classification error

$$
\ell_{\text{err}}(h(x), y) =
\begin{cases}
0 & \text{if } \arg\max_i h_i(x) = y \\
1 & \text{otherwise}
\end{cases}
$$

> 分类错误损失函数，只检查分类是否失败
>
> argmax 就是取最大的可能性，是否等于标签 y
>
> 分类错误损失函数不适合作为优化过程中选择最佳参数的损失函数，因为它不可微分
>
> 在优化算法中，我们通常需要用到梯度信息，而分类错误损失函数由于其离散的性质，无法提供连续的梯度。

#### softmax / cross-entropy loss

$$
z_i = p(\text{label} = i | x) = \frac{\exp(h_i(x))}{\sum_{j=1}^k \exp(h_j(x))}
$$

- $h_i(x)$ 是假设函数的第 $i$ 个输出。
- $k$ 是类别的总数。
- $p(\text{label} = i | x)$ 是输入 $x$ 属于类别 $i$ 的概率。

$$
\ell_{ce}(h(x), y) = -\log(p(\text{label} = y | x)) = -h_y(x) + \log \left( \sum*{j=1}^k \exp(h_j(x)) \right)
$$

> 交叉熵损失衡量的是预测概率与实际类别之间的差异。当模型对正确类别的预测概率较高时，损失较低；反之，当模型对错误类别的预测概率较高时，损失较高。通过最小化交叉熵损失，我们可以训练模型，使其对正确类别的预测概率最大化。
>
> 最小化的是**负的对数概率**，这意味着我们希望增加正确类别的预测概率。通过最小化这个损失函数，我们可以训练模型，使其对正确类别的预测概率最大化。
>
> 也就是上面将其转成了概率，输入为 x 且 label 为 i 的概率，除所有 j 将其归一化，就变成了错误概率。
>
> 交叉熵与信息论：https://zhuanlan.zhihu.com/p/149186719
>
> 在 Softmax 回归（或多分类逻辑回归）中，假设函数是线性的？那 ReLU 呢

The softmax regression optimization problem

$$
\min_{\theta} \frac{1}{m} \sum_{i=1}^{m} \ell_{\text{ce}}(h_{\theta}(x_i), y_i)
$$

- $\theta$ 是模型参数
- $m$ 是训练集中样本的数量。
- $x_i$ 和 $y_i$ 分别是第 $i$ 个样本的输入和真实标签。
- $\ell_{\text{ce}}$ 是交叉熵损失函数。

在软最大值回归中，假设函数是线性的，并且使用软最大值损失：

$$
\min*{\theta} \frac{1}{m} \sum*{i=1}^{m} \ell\_{\text{ce}}(\theta^T x_i, y_i)
$$

> 就是找到这个 theta 参数，计算梯度。在实际应用中，通常使用一种称为随机梯度下降（Stochastic Gradient Descent, SGD）

### Optimization: gradient descent

the gradient is defined as the matrix of partial derivatives

> 梯度是向量函数的一个重要概念，用于描述函数在特定点的变化率
>
> 梯度下降是一种优化算法，用于最小化目标函数 $f(\theta)$ 的值。这个算法的核心思想是通过迭代地沿着负梯度方向更新参数，使得目标函数的值逐渐减小。
>
> https://dlsyscourse.org/slides/2-softmax_regression.pdf
>
> 图解可以看出 $\alpha$ 是 step size 或 learning rate 大于 0 的时候
>
> $\theta$ 发生变化

$$
\theta \leftarrow \theta - \alpha \nabla_{\theta} f
$$

### Stochastic Gradient Descent

> 随机梯度下降
>
> 用于在机器学习模型的训练过程中最小化损失函数。与传统的梯度下降不同，SGD 不是在每次更新时使用整个训练集的数据，而是使用一个**小的随机数据子集**（即小批量 mini batch）
>
> http://zh.gluon.ai/chapter_optimization/gd-sgd.html
>
> 小批量随机梯度下降（SGD）引入了一定的随机性，使得模型可以更快跳出局部最优，朝全局最优收敛。这种随机性有助于在复杂的损失函数空间中进行更有效的搜索。

$$
\mathbf{\theta} \leftarrow \mathbf{\theta} - \frac{\alpha}{B} \sum_{i=1}^B \nabla_{\mathbf{\theta}} \ell_i(h_{\mathbf{\theta}}(\mathbf{x}_i), \mathbf{y}_i)
$$

### The gradient of the softmax objective

> auto grad?
>
> 如果手动求导，需要链式法则，会很复杂
>
> 大部分人当成了标量 scalar
>
> 正式的方法：使用矩阵微积分、雅可比矩阵、克罗内克积和向量化 vector 技术。这种方法是“正确的方式”，更加严谨和系统，但也更加复杂和费时。
>
> 大家实际使用的捷径方法：假装所有变量都是标量，使用典型的链式法则，然后在计算结果后重新排列/转置矩阵和向量，使其维度匹配。虽然这种方法不那么严谨，但在实际操作中更加快捷和方便。

### softmax regression algorithm

> 尽管推导过程相当复杂，但最终的算法非常简单。
>
> Softmax 回归是多分类问题的线性模型，通过交叉熵损失函数和梯度下降优化参数

MNIST 数据集：手写数字分类（10 类，28x28 像素）。

结果：错误率<8%，训练时间仅数秒。

> Softmax 回归是线性分类器，而神经网络通过非线性层（激活函数）和隐藏层构建更复杂的假设空间。
