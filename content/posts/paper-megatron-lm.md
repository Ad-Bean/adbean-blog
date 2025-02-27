+++
title = 'Paper Reading: Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism'
date = 2025-02-26T13:19:30-05:00
draft = false
tags = ['Paper Reading', 'Deep Learning', 'Language Models', 'Training']
+++

## Megatron-LM

Nvidia 开源的 [Megatron-LM](https://github.com/NVIDIA/Megatron-LM) 大模型训练框架

结合 Model Parallelism 和 Pipeline Parallelism 实现了 Tensor Model Parallelism

基于 Transformer 和 Attention 进行切分，同样是经典的一篇分布式语言模型训练的文章

> 论文比较短，细节很少，需要结合李沐大神的精读视频阅读，关于 AI 的东西还是得多看多动手，不然一知半解
>
> 柳浩“罗西的思考” 大神的 [MegatronLM 源码分析](https://www.cnblogs.com/rossiXYZ/p/15840803.html)

## Abstract

训练大型 Transformer 模型，由于内存限制，训练非常大的模型可能会非常困难

本文介绍了训练超大型 Transformer 模型的技术，并实现了一种简单高效的层内模型并行方法，使得训练具有数十亿参数的 Transformer 模型成为可能

orthogonal and complimentary to pipeline model parallelism, 可以通过在原生 PyTorch 中插入少量通信操作来完全实现

通过使用 512 个 GPU 收敛了高达 83 亿参数的基于 Transformer 的模型来展示这种方法

## Introduction

随着这些模型变得越来越大，它们超出了现代处理器的内存限制，需要额外的内存管理技术，例如激活检查点

> activation checkpointing 减小内存占用，丢弃大部分中间激活值

可扩展性：单 GPU 到 512 GPU

准确性：SOTA

## Background and Challenges

### Neural Language Model Pretraining

### Transformer Language Models and Multi-Head Attention

当前 NLP 的研究趋势倾向于使用 Transformer 模型，最初的 Transformer 设计是一种机器翻译架构，通过编码器（Encoder）和解码器（Decoder）两部分将输入序列转换为输出序列。然而，最近利用 Transformer 进行语言建模的工作，如 BERT（Devlin 等，2018）和 GPT-2（Radford 等，2019），根据需求仅使用编码器或解码器。

值得注意的是，GPT-2 和 BERT 都使用了 GeLU（Hendrycks & Gimpel，2016）非线性和层归一化（Ba 等，2016）应用于多头注意力和前馈层的输入，而原始 Transformer（Vaswani 等，2017）使用 ReLU 非线性并将层归一化应用于输出。

### Data and Model Parallelism in Deep Learning

将深度神经网络训练扩展到多个硬件加速器时，有两种核心范式：

- data parallelism(Valiant, 1990): 将训练小批量数据拆分到多个工作节点上；
- model parallelism: 将模型的内存使用和计算分布到多个工作节点上。

然而，这些技术在可处理的问题规模上存在一个根本性限制：模型必须完全适合单个工作节点。随着 BERT 和 GPT-2 等语言模型的规模和复杂性不断增加，神经网络已接近现代硬件加速器的内存容量。

Within model parallelism, there are two further paradigms:

- layer-wise pipeline parallelism: TensorFlow GPipe 然而，这种方法需要额外的逻辑来高效处理通信和计算操作的流水线，并且存在管道气泡（pipeline bubbles）问题，降低了效率，或者需要修改优化器本身，从而影响准确性。
- distributed tensor computation: 正交且更通用的方法，FlexFlow，Mesh-TensorFlow 等等

我们利用了与 Mesh-TensorFlow 类似的思路，并通过并行计算 Transformer 的注意力头来实现 Transformer 模型的并行化，仅对现有的 PyTorch Transformer 实现进行了一些有针对性的修改。不需要任何新的编译器或代码重写，只需插入一些简单的原语即可完全实现。

## Model Parallel Transformers

利用 Transformer 网络的结构，通过添加少量同步原语，实现了一种简单的模型并行方法。

![Transformer Architecture](https://s2.loli.net/2025/02/27/FN24ADnyuprMU3g.png)

Transformer 层由一个自注意力块和一个两层的多层感知机（MLP）

### MLP 块的并行化

MLP 块的第一部分是一个 GEMM（通用矩阵乘法）操作，后接一个 GeLU 非线性激活函数

$$
    Y=GeLU(XA)
$$

并行化 GEMM 的一种选择是将权重矩阵 A 沿其行 rows 分割输入 X 沿其列 columns 分割

$$
    X = [X_1 \ X_2], \quad A = \begin{bmatrix} A_1 \\ A_2 \end{bmatrix}
$$

这种分区会导致 $Y = \text{GeLU}(X_1A_1 + X_2A_2)$，由于 GeLU 是一个非线性函数（$\text{GeLU}(X_1A_1 + X_2A_2) \neq \text{GeLU}(X_1A_1) + \text{GeLU}(X_2A_2)$ ，需要在 GeLU 函数之前添加一个同步点。

另一种选择是将 $A$ 沿其列 column 分割 $A = [A_1 \ A_2]$。这种分区允许 GeLU 非线性独立应用于每个分区 GEMM 的输出：

$$
[Y_1 \ Y_2] = [\text{GeLU}(XA_1) \ \text{GeLU}(XA_2)]
$$

这种方法的优势在于它移除了一个同步点。因此，我们以这种列并行 column parallel 的方式对第一个 GEMM 进行分区，并将第二个 GEMM 沿其行 rows 分割，使其直接接收 GeLU 层的输出，而无需任何通信，

> GEMM（General Matrix Multiply）

在前向传播中只需要一次 all-reduce 操作（g 操作符），在反向传播中也只需要一次 all-reduce 操作（f 操作符）。这两个操作符互为共轭，可以在 PyTorch 中用几行代码实现。

```python
class f(torch.autograd.Function):
    def forward(ctx, x):
        return x

    def backward(ctx, gradient):
        all_reduce(gradient)
        return gradient
```

### 自注意力块的并行化

![Blocks of Transformer with Model Parallelism](https://s2.loli.net/2025/02/27/e2Hz5xPXAqaSMtU.png)

对于自注意力块，我们利用了多头注意力操作中的固有并行性，将键（K）、查询（Q）和值（V）相关的 GEMM 以列 column parallel 并行的方式分区，使得每个注意力头对应的矩阵乘法在单个 GPU 上本地完成

The transformer language model has an **output embedding** with the dimension of hidden-size (H) times vocabulary size (v)

由于现代语言模型的词汇表大小通常在数万个词（例如，GPT-2 使用了 50,257 的词汇表大小），因此并行化输出嵌入的 GEMM 操作是有益的。然而，在 Transformer 语言模型中，输出嵌入层与输入嵌入共享权重，因此需要对两者进行修改。

输入嵌入权重矩阵 $E_{H\times v}$ 沿词汇表维度并行化，按列分割

> 这里省略 embedding 没太看懂

我们的模型并行方法主要通过减少通信并保持 GPU 的计算负载来实现优化。与其让一个 GPU 计算部分 dropout、层归一化或残差连接并将结果广播到其他 GPU，我们选择在 GPU 之间**复制计算**。

附录 B 中提供了关于混合模型和数据并行以及随机数生成处理的更多细节以供参考。总之，我们上述的方法实现简单，只需在前向和反向传播中添加少量额外的 all-reduce 操作。它不需要编译器，并且与管道模型并行是正交且互补的。

## Setup

GPT-2, BERT

### Training Dataset

聚合了几个最大的语言建模数据集

### Training Optimization and Hyperparameters

混合精度训练和动态损失缩放

> 好奇为什么论文没提到 FP16 FP32 等等

## Experiments

32 台 DGX-2H 服务器（共 512 个 Tesla V100 SXM3 32GB GPU）

### Scaling Analysis

GPT-2 models with four sets of parameters detailed in Table1

![GPT model size && Model and model + data parallel](https://s2.loli.net/2025/02/27/ROUJks4GYzfcMte.png)

### MODEL AND DATA PARALLELISM

展示模型并行和模型+数据并行情况下关于模型参数的弱扩展性

> 弱扩展性 weak scaling 代表了什么？怎么计算？
>
> 是否应该稳定或者轻微下降才表示扩展良好

### Language Modeling Results Using GPT-2

![Model configurations used for GPT-2](https://s2.loli.net/2025/02/27/TvF5tNPSKdwulEn.png)

表 2 还列出了完成一个 epoch 所需的时间，相当于 68,507 次迭代。例如，对于在 512 个 GPU 上训练的 8.3B 模型，每个 epoch 大约需要两天时间。与表 1 中用于扩展性研究的配置相比，2.5B 模型相同，8.3B 模型有 24 个注意力头而不是 32 个，而 355M 模型比之前看到的任何模型都小得多，但仍然使用 64 个 GPU 进行训练，因此每个 epoch 的时间大大减少。

验证困惑度（perplexity）随迭代次数的变化，随着模型规模的增加，验证困惑度下降，8.3B 模型的验证困惑度达到 9.27

微软的研究人员与 NVIDIA 合作，使用 Megatron 训练了一个 170 亿参数的 GPT-2 模型，称为 Turing-NLG（Microsoft，2020），并展示了随着模型规模的扩大，准确率进一步提高，凸显了大型模型的价值。

### Bi-directional Transformer Results Using BERT

略

## Conclusion and Future Work

> 比较短的一篇论文，方法很粗暴简单，但也很有效
>
> 对于 DL/LLM Training 我其实很难想象这些并行/分布式训练是几年前才热门，还需要阅读更多
>
> 尤其是 Megatron 的源码
