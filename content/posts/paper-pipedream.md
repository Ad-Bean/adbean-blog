+++
title = 'Paper Reading: PipeDream: Generalized Pipeline Parallelism for DNN Training [SOSP2019]'
date = 2025-02-24T10:49:51-05:00
draft = false
tags = ['Paper Reading', 'Deep Learning', 'Training']
+++

## PipeDream: Generalized Pipeline Parallelism for DNN Training

第一次看 ML/DL (distributed) training 框架相关的论文，有很多地方不理解。

对 Evaluation 与数学证明更是浅尝辄止，许多指标和算法难以理解，关于大模型、分布式训练等等有许多总结得非常好的博客和分享，参照着多看看加深记忆。

> 流水线并行分布式训练一篇经典的文章，结合 DP, MP 以及 Pipeline
>
> 这篇和 Gpipe 非常相似： [GPipe: Efficient Training of Giant Neural Networks using Pipeline Parallelism](https://arxiv.org/abs/1811.06965)
>
> 类似还有 MegatronLM 团队的论文 [Megatron-LM](https://arxiv.org/pdf/2104.04473)
>
> [Gpipe 知乎讲解](https://zhuanlan.zhihu.com/p/613196255)
>
> 关于 Gpipe 和 PipeDream 源码详解可以参考柳浩“罗西的思考”大神的博客 [Gpipe 源码解析](https://www.cnblogs.com/rossiXYZ/p/15172665.html)
>
> PipeDream 一个很有意思的 [笔记 by jasperzhong](https://github.com/jasperzhong/read-papers-and-code/issues/167) 详细解释了 pipedream 做 partition 的时候为什么可以用动态规划

## ABSTRACT

深度神经网络（DNN）的训练极为耗时，需高效的多加速器并行化。

当前的训练并行化方法主要使用 intra-batch parallelization, 但在较高工人数目时会遇到边际收益递减

PipeDream: adds inter-batch pipelining to intra-batch parallelism to further improve parallel training throughput

与传统流水线不同，DNN 训练是双向的，计算图的前向传播之后是反向传播，后者使用前向传播期间计算的状态和中间数据

PipeDream 对模型参数进行版本控制以确保梯度计算的数值正确性，并在不同工作节点上同时调度不同小批次的前向和反向传播，以最小化流水线停顿。

> intra-batch parallelism techniques 比如传统的数据并行 DP 和模型并行 MP
>
> https://leimao.github.io/blog/Data-Parallelism-vs-Model-Paralelism/ 毛磊大神的博客带有 Data Parallelism 的证明
>
> 他的观点也很有意思，Model Parallelism 并不是真的并行，而是 Model Serialization

## INTRODUCTION

Deep Neural Networks (DNNs) 训练所需的计算成本显著增加，需要在多个 accelerators（e.g., GPUs）上进行并行执行。

DNN 训练通过前向传播和反向传播的迭代计算进行，在每次迭代中，训练循环处理一个小批次（minibatch）的输入数据，并更新模型参数。当前的方法主要集中在将优化算法的每次迭代在一组工作节点上并行化。

- 数据并行将输入数据分配到不同工作节点上
- 模型并行将算子分配到不同工作节点上
- 而混合方案则同时分配数据和模型

然而，批内并行化在大规模训练中可能会面临高通信开销的问题

![Communication overhead of data-parallel training](https://s2.loli.net/2025/02/25/SlJxHCBUX4Q8osM.png)

在 32 个 GPU 上，某些模型的通信开销（以通信停顿时间占总时间的百分比计算）高达 90%，这是由于跨服务器的 `all_reduce` 通信成本高昂。即使在使用专用互连（如 NVLink [4]）的服务器上，通信开销仍然很高。此外，GPU 计算能力的快速提升将进一步使训练的瓶颈转向**通信**。

PipeDream: combining intra-batch parallelism with inter-batch parallelization

> 怎么减少通信开销？看之前 Gpipe 的实验大部分时间都花费在通信上，较小的批次会带来更多开销吗

PipeDream 将模型分配到可用工作节点上，为每个工作节点分配一组连续的算子 consecutive operators（在 DNN 术语中称为层 layers），然后以流水线方式重叠不同输入的计算和通信。这一过程可以显著减少工作节点间的通信，因为它将通信限制在分配到不同工作节点的连续层之间的层输入和输出（前向传播中的激活值和反向传播中的梯度），对于许多模型来说，这些通信量远小于整个模型的大小。此外，这种通信是点对点的，而不是全对全的。

PipeDream 采用了一种更精细的流水线方法：给定一个由分配到不同工作节点的连续层组（称为阶段 stage）组成的流水线，PipeDream 使用一种称为 **1F1B** 的调度算法来保持硬件的高利用率，同时实现与数据并行相似的语义。

在 1 Forward 1 Backward 的稳态下，每个工作节点严格交替执行其阶段的前向传播和反向传播，确保高资源利用率（negligible pipeline stalls, no pipeline flushes）

> PipeDream 在多 GPU 上效果很好

## BACKGROUND AND RELATED WORK

深度神经网络（DNN）模型由许多算子组成，这些算子被组织成层。

> 没正经学过 ML DL，这里的算子可以理解为各种操作吗？比如卷积、矩阵乘法或者激活函数？然后组织成一个层，卷积层等等

### Intra-batch Parallelism

在并行化 DNN 训练时，这些层可以以不同的方式分配到可用工作节点上。

Data Parallelism: 输入数据被分配到不同工作节点上，同时定期通过集体通信原语（如 `all_reduce` [24]）或参数服务器 [38] 与其他工作节点同步权重。最常用的数据并行形式称为 bulk synchronous parallel or BSP

Model Parallelism: intra-batch 将 DNN 模型中的算子分配到可用工作节点上，每个工作节点仅评估和更新模型参数的一个子集（针对所有输入）。通信的数据量是需要跨工作节点传输的中间输出（及相应的梯度）的大小。

Hybrid Intra-batch Parallelism: OWT, FlexFlow ...

> 怎么理解 intra-batch 和 inter-batch parallelism
>
> 一次训练的同一批次，比如一些数据/模型拆分，data/model 分配到不同 worker 并行，加以流水线
>
> https://blog.csdn.net/weixin_36378508/article/details/129838193

### Inter-batch Parallelism

GPipe（与 PipeDream 的早期预印本 [25] 同时期的工作）在模型并行训练的背景下使用流水线技术来训练非常大的模型

GPipe 进一步将一个小批次拆分为 m 个微批次（microbatches） 并对这些微批次执行前向传播，然后进行反向传播（见图，m=4）

![Gpipe inter-batch](https://s2.loli.net/2025/02/26/DTQjfF5cxgXmRHl.png)

> Gpipe 还使用了 权重梯度聚合（Weight Gradient Aggregation） 和 通过丢弃激活值存储来以计算换内存（Trading Computation for Memory by Discarding Activation Stashes）
>
> 前者是否是 All-Reduce？

### DNN Model and Hardware Diversity

DNN 模型具有多样性，常见的模型包括卷积层、LSTM [55]、注意力层 [53] 和全连接层。这些不同类型的模型在使用不同的并行化策略时表现出截然不同的性能特征，因此最优的并行化策略高度依赖于模型本身。

> 不同模型不同的并行表现，Pipedream 是更通用的选择还是？

## PIPELINE PARALLELISM

PipeDream 使用了一种新的并行化策略——流水线并行化 Pipeline，结合了批内并行化和批间并行化。流水线并行化将 DNN 模型的层划分为多个阶段（stage），每个阶段由模型中一组连续的层组成。每个阶段被映射到一个单独的 GPU 上，该 GPU 负责执行该阶段中所有层的前向传播和反向传播。

> pipedream 是把一个 batch 切分的更细吗，这样一小段就可以立刻发给下一个 gpu 开始 forward？
>
> 这样利用率更高， gpu bubble 减少

![Model Parallel](https://s2.loli.net/2025/02/26/o37HUOfA4ixWy1B.png)

流水线并行化优于批内并行化的原因：

- 减少了通信量（小于 DP），DP 需要聚合，PP 只需要发给另一个节点
- 重叠通信和计算：异步通信使得前向传播的激活值和反向传播的梯度可以在不同阶段之间重叠计算和通信

![pipeline-parallel](https://s2.loli.net/2025/02/26/ueol8xsTbOdiynt.png)

> 这种训练方式一般如何保证正确性？如何切分才能保证负载均衡

### Challenge 1: Work Partitioning

与任何流水线一样，最终流水线的稳态吞吐量取决于最慢阶段的吞吐量

工作节点之间的过多通信也会降低训练流水线的吞吐量

Solution: PipeDream Optimizer 输出一个平衡的流水线

PipeDream 在初始分析步骤中记录每个层的前向和反向传播的计算时间、层输出的激活值大小以及相关参数的大小；这些分析数据作为优化器划分算法的输入（图 6）。划分算法还考虑了其他约束条件，如硬件拓扑和带宽、工作节点数量以及计算设备的内存容量。

Profiler:

Partitioning Algorithm:

PipeDream 的优化器从最低级别到最高级别逐步解决动态规划问题

> 省略公式，转化成了一个 DP 问题

### Challenge2: Work Scheduling

流水线中的每个活跃小批量数据可能处于不同的阶段，要么在前向传播中，要么在反向传播中。

Solution: 在启动阶段，输入阶段会接收足够多的小批量数据，以使流水线在稳态下保持满负荷

一旦进入稳态，每个阶段会交替执行一个小批量数据的前向传播和另一个较早小批量数据的反向传播。我们称之为“一前一后”（1F1B）调度。

“一前一后轮询” 1F1B-RR：当一个阶段以数据并行配置运行（在多个 GPU 上复制）时，我们使用基于小批量数据标识符的确定性轮询负载均衡，将工作分配到各个副本上

观察到在实践中反向传播总是比前向传播耗时更长。1F1B-RR 仍然是一种有效的调度机制

### Challenge3: Effective Learning

在一个简单的流水线系统中，每个阶段的前向传播使用一个版本的参数，而其反向传播则使用另一个版本的参数

因此，在阶段 1 中，小批量数据 5 的反向传播中使用的权重与对应前向传播中使用的权重不同；这种权重版本的不一致会导致无效的梯度，并可能阻止模型收敛。

![Pipe Dream pipeline with 4 workers](https://s2.loli.net/2025/02/26/KXGzxgZojLueYsJ.png)

Solution: PipeDream 使用一种称为权重暂存（weight stashing）的技术来避免前向传播和反向传播中使用的权重版本之间的根本性不匹配。

Memory Overhead：流水线并不会显著增加每个工作节点的内存使用量，即使使用权重暂存也是如此。

## Implementation

PipeDream 的接口实现为一个独立的 Python 库，约 3000 行代码，负责管理设备内存、调度工作并处理通信。

PipeDream 使用 PyTorch 进行自动微分和执行算子；然而，PipeDream 是可扩展的，并且可以与其他机器学习框架（如 TensorFlow、MXNet 和 CNTK）一起工作。

Parameter State: PipeDream 将所有与阶段相关的参数直接保存在 GPU 内存中，只有在使用更新参数的向后传播完成后，参数才会被丢弃。

Intermediate State: 每个阶段的输入和输出数据被分配一个唯一的 blob ID。当从前一个阶段（或输入阶段从磁盘）接收到中间数据时，PipeDream 将中间数据复制到 GPU 内存中，并将指向相关缓冲区的指针放入工作队列中

Stage Replication: PipeDream 使用 PyTorch 的 DistributedDataParallel 库 [6] 来同步数据并行阶段的层的参数。

Checkpointing: PipeDream 支持定期检查模型参数以实现容错，默认在每个 epoch 结束时跨阶段创建检查点。

## EVALUATION

1. PipeDream 在不同硬件部署上的多种学习任务中显著加速了达到目标精度的时间；
2. PipeDream 比其他最近提出的跨批次方法更高效；
3. PipeDream 大大减少了通信开销，并且与数据并行训练相比，并未显著增加内存占用；
4. 结合流水线并行、模型并行和数据并行优于单独的模型并行、数据并行或混合并行。

> 减少通信开销，并未显著增加内存占用
>
> 批次如何设置？

Batch Sizes and Training Methodology: 我们使用适合单个 GPU 内存的最大每 GPU 小批量数据——更大的批量会导致内存不足异常。这确保我们在单个设备上达到峰值 FLOPs。

### Comparison to Data Parallelism

![Summary of results comparing PipeDream with data parallelism](https://s2.loli.net/2025/02/26/CPFrV7aWgSeqsKd.png)

表 1 总结了 PipeDream 与数据并行训练（DP）的比较结果。表中显示了 PipeDream 自动生成的配置及其在达到目标精度训练时间上相对于相应数据并行配置的加速比。

### Comparison to Other Intra-batch Parallelism Schemes

Model Parallelism: For VGG-16 and AlexNet 14.9x and 6.5x

### Comparison to Inter-batch Parallelism

Gpipe:
These throughput slow downs are largely due to more frequent pipeline flushes compared to PipeDream

### Microbenchmarks

优化器：PipeDream 的优化器非常高效，对于所有评估的模型和硬件部署，生成最佳训练配置的时间不到 8 秒。

内存占用：图 16 显示了 PipeDream 在 4 阶段配置下三个不同模型的每阶段内存占用。PipeDream 的最坏情况内存占用与数据并行相当，尽管 PipeDream 存储了多个权重和激活版本。

通信开销：

流水线深度的影响：最佳非 DP 配置的通信开销远低于 DP 配置的通信开销

> optimizer 如何实现？

## CONCLUSION

流水线并行的 DNN 训练有助于减少可能成为批次内并行瓶颈的通信开销。PipeDream 自动将 DNN 训练划分到多个工作节点上，结合跨批次流水线与批次内并行，以更好地重叠计算与通信，同时最小化通信数据量。

> pipeline 的实现？很好奇 optimizer 如何实现，profiling 是提前采样吗？
>
> 基于 pytorch 的简单实现 [PipeDream 实现](https://zhuanlan.zhihu.com/p/626202865)
>
> 在 gpipe 和 pipedream 有个很有意思的 tradeoff：
>
> 由于 GPU 显存反而更加珍贵，会使用 checkpoint 时间换空间
>
> pipedream 更加偏向异步、gpipe 是同步
