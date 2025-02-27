+++
title = 'Paper Reading: Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM'
date = 2025-02-26T22:39:12-05:00
draft = false
tags = ['Paper Reading', 'Deep Learning', 'Language Models', 'Training']
+++

## Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM

姑且算作 Megatron LM v2，因为是晚一年发表的，粗略过一下，因为都是同样的 motivation 和 background 等等

> [知乎一篇不错的总结](https://zhuanlan.zhihu.com/p/19482552307)
>
> 作者 Jared 的[视频介绍](https://youtu.be/gHaNUcS1_O4?si=EEgK2Y5_6fZtAJB8)
>
> 其中和 ZeRO 的对比很有意思，吞吐量高了很多。周末看看 Gpipe 和 ZeRO

## ABSTRACT

本文中，我们展示了如何将张量并行、流水线并行和数据并行结合起来，以扩展到数千个 GPU。

提出了一种新颖的交错流水线调度方法，可以在内存占用与现有方法相当的情况下，将吞吐量提高 10%以上。

## INTRODUCTION

数据并行扩展通常效果良好，但存在两个局限性：

1. beyond a point, the per-GPU batch size becomes too small, reducing GPU utilization and increasing communication cost
2. the maximum number of devices that can be used is the batch size, limiting the number of accelerators that can be used for training.

模型并行技术、张量（层内）模型并行化

1. 张量并行所需的 all-reduce 通信需要通过服务器间链路，这些链路比多 GPU 服务器内可用的高带宽 NVLink[9]慢；
2. 高度的模型并行化可能会产生小的矩阵乘法（GEMMs），可能会降低 GPU 利用率。

Pipeline model parallelism 流水线模型并行化，一个批次被拆分为较小的微批次，执行过程在这些微批次之间进行流水线处理。

将流水线并行、张量并行和数据并行结合起来，这种技术我们称之为 PTD-P

还与 ZeRO[36]进行了比较，发现由于跨节点通信较少，我们的方法在 1750 亿和 5300 亿参数的模型上比 ZeRO-3 高出 70%。这些模型太大，无法容纳在多 GPU 服务器上。

## MODES OF PARALLELISM

### Data Parallelism

每个工作节点都拥有完整模型的副本，输入数据集被分片，工作节点定期聚合它们的梯度，以确保所有工作节点看到一致的权重版本

### Pipeline Model Parallelism

在流水线并行化中，**模型的各层**被分配到多个设备上。当用于具有重复相同 Transformer 块的模型时，每个设备可以被分配相同数量的 Transformer 层。

一个批次被拆分为较小的微批次；然后在微批次之间进行流水线执行。

> 其实流水线和 CPU 的流水线调度很相似？

量化 GPipe 的流水线气泡大小（𝑡𝑝𝑏）。我们将批次中的微批次数量表示为 𝑚，流水线阶段的数量（用于流水线并行的设备数量）表示为 𝑝，每次迭代的理想时间表示为 𝑡𝑖𝑑（假设完美或理想的扩展），执行单个微批次的前向和后向传递的时间表示为 𝑡𝑓 和 𝑡𝑏。在此调度中，流水线气泡包括批次开始时的 𝑝−1 个前向传递和批次结束时的 𝑝−1 个后向传递。流水线气泡中花费的总时间为 𝑡𝑝𝑏 = (𝑝−1)·(𝑡𝑓 +𝑡𝑏)。批次的理想处理时间为 𝑡𝑖𝑑 = 𝑚·(𝑡𝑓 +𝑡𝑏)。因此，流水线气泡中花费的理想计算时间的比例为

$$
Bubble\ time\ fraction (pipeline\ bubble\ size) = \frac{t_{pb}}{t_{id}} = \frac{(p−1)\cdot (t_f + t_b)}{m \cdot(t_f + t_b)} = \frac{p−1}{m}
$$

为了减少流水线气泡的影响，可以**增加微批次的数量**或**减少流水线阶段的数量**。

使流水线气泡时间占比（bubble time fraction）尽可能小，我们需要满足 $m >> p$

Schedule with Interleaved Stages: 新调度将气泡时间减少了 v 倍，这种流水线气泡大小的减少并非没有代价：这种调度需要额外的通信。

### Tensor Model Parallelism

![](https://s2.loli.net/2025/02/27/XAPO3Kie1fxMUlS.png)

1. MLP parallelism
2. multi-head attention parallelism

## PERFORMANCE ANALYSIS OF PARALLELIZATI ON CONFIGURATIONS

### Tensor and Pipeline Model Parallelism

$$

\frac{p - 1}{m} = \frac{n / t - 1}{m}
$$

因此，我们可以看到，**张量模型并行化**增加了设备之间的通信量。因此，当 𝑡 大于单个节点中的 GPU 数量时，跨较慢的节点间链路执行张量模型并行化的开销可能是不切实际的。

> 𝑡 表示张量模型并行的规模，𝑝 表示流水线模型并行的规模，𝑑 表示数据并行的规模。
>
> 𝑏：微批次大小
> 𝑚：每个流水线中一个批次内的微批次数量
> 𝑛：GPU 的数量

### Data and Model Parallelism

在使用张量模型并行化时，每个微批次都需要执行 all-reduce 通信。这在跨多 GPU 服务器时可能会非常昂贵

### Microbatch Size

微批次大小 b 的选择也会影响模型训练的吞吐量。

### Activation Recomputation

激活重计算（Activation Recomputation）[12, 18, 20, 21] 是一种可选技术，通过在后向传递之前再次运行前向传递（并仅保存给定流水线阶段的输入激活值，而不是整个中间激活值集，后者占用内存更大），以增加计算操作的数量为代价来减少内存占用。

## IMPLEMENTATION

> 略

## EVALUATION

- PTD-P 的性能如何？它是否能实现实际的端到端训练时间？

- 流水线并行化在给定模型和批量大小下的扩展性如何？交错调度对性能有多大影响？

- 不同的并行化维度之间如何交互？微批次大小等超参数的影响是什么？

- scatter-gather 通信优化的影响是什么？在大规模运行训练迭代时，我们对硬件施加了哪些限制？

### End-to-End Performance

评估了系统在参数量从 10 亿到 1 万亿的 GPT 模型上的端到端性能

启用了 scatter/gather 优化 的交错流水线调度

> 如何计算 FLOPS？

### Comparison to ZeRO-3

PTD-P 在这两个模型上的性能比 ZeRO-3 高出 70%，这得益于更少的跨节点通信。

### Pipeline Parallelism

Weak Scaling. 较高的批量大小具有更好的扩展性，因为流水线气泡被分摊到更多的微批次上

Interleaved versus Non-Interleaved Schedule. 启用 scatter/gather 通信优化 的交错调度比非交错（默认）调度具有更高的计算性能。

### Comparison of Parallel Configurations

### Microbatch Size

### Activation Recomputation

### Scatter-Gather Optimization

### Fused Operators

### Inter-Node Communication Bandwidth

### Checkpoint Loading and Saving

> 各种配置 evaluation 略

## RELATED WORK
