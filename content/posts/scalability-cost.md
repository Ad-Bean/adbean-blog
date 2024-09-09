+++
title = 'Paper Reading: Scalability! But at what COST'
date = 2024-09-08T20:25:59-04:00
draft = false
tags = ['Paper Reading']
+++

## Scalability! But at what COST?

比较短的 paper，之前也略有看过，主要讲主流分布式框架在多机器上存在 COST(不是成本) 问题，并不一定能比单机更好。是很有意思的主题。

## Abstract

new metric for big data platforms: COST Configuration that Outperforms a Single Thread, 超越单线程的配置

许多系统的 COST 出乎意料地大，通常需要数百个核心，或者在所有报告的配置中，其性能始终无法超越单线程。

## Introduction

可扩展性视为分布式数据处理平台最重要的特性，但很少有研究直接评估其系统在合理基准测试下的**绝对性能**。这些系统在多大程度上真正提升了性能，而不是仅仅并行化了它们自身引入的开销？

any system can scale arbitrarily well with a sufficient lack of care in its implementation.

![](https://s2.loli.net/2024/09/09/YSpbq8L3UJecfxo.png)

消除了并行带来的开销，损害了可扩展性，但是提高了性能。（多核情况下 speed up 降低了，但是延迟却低了）

### Methodology

graph processing, many published systems have **unbounded COST**—i.e., no configuration outperforms the best single-threaded implementation—for all of the problems to which they have been applied

In some cases the **singlethreaded** implementations are more than an order of mag- nitude faster than published results for systems using **hundreds of cores**.

> 许多老的系统很难超越单机的表现，这是为什么呢？作者是否消除了并行的开销，比如共识算法？再者说拿旧的系统现在来比，是不是有些不公平呢，优化什么的也不太一样。
>
> 不过比较了 pagerank 在不同系统的表现

## Basic Graph Computations

> 作者用的是单线程 C# 代码，有点神秘

### PageRank

![](https://s2.loli.net/2024/09/09/MviSFXzaOVbGs3T.png)

可以看到单机的性能反而优秀很多

> 论文是使用 SSD + RAM，有没有机械硬盘的比较呢

### Connected Components

![](https://s2.loli.net/2024/09/09/Hi8nKxq76NrsoLD.png)

单线程还是快

> 他的单机性能到底如何呢？之前我也做了多 ec2 和单 ec2 的比较，核心较少的时候确实是有性能提升的，但随着核数增多，确实存在由于分片算法精度丢失导致的不均匀导致提升不是线性

## Better Baselines

### Improving graph layout

论文优化了其他的实现，比如图实现

### Improving algorithms

优化算法，图联通，应该并不是适合并行的把

> 怎么还有并查集？

## Applying COST to prior work

## PageRank

## Graph connectivity

## Lessons learned

scalable systems design and implementation contribute to **overheads** and increased **COST**

> 其实到底做了什么导致 overhead 和 COST？是之前的研究基准测试不对吗，还是说这篇文章在图处理上有问题？
>
> 那为什么大家还用 mapreduce, spark 呢？甚至现在越来越多分布式数据库呢
>
> 作者提到 MapReduce 存在许多磁盘写入，是持久化的，也就是说他的实验很可能都是内存的？

## Future directions (for the area)

许多高性能的可扩展系统实例存在。Galois [17] 和 Ligra [23] 都是共享内存系统，当在单台机器上运行时，它们的表现显著优于其分布式同类系统。Naiad [16] 引入了一种新的通用数据流模型，甚至超越了专用系统。理解这些系统做得正确的地方以及如何改进它们，比在新的领域中重复现有想法并与先前工作中最差的部分进行比较更为重要。

<!--
1. the problem the paper addresses (no more than 2-3 sentences each)
2. the paper’s solution (no more than 2-3 sentences each)
3. the evidence the paper provides that the solution was correct. (no more than 2-3 sentences each)

1. whether the problem the paper solved is an important one
2. whether the solution was a good one
3. whether the paper was fun or enjoyable to read.

5. The COST paper complains that distributed systems papers are sometimes guilty of NOT evaluating against a simple single-threaded approach. However, this is not always possible -- some applications (such as building OS services) need to provide scalability and inherently cannot be single threaded. Given this, are there any lessons we can take from the COST paper when thinking about the multikernel?


------------------ 2 ---------------------

The paper COST, is trying to illustrate that some previous big data frameworks do not necessarily improve performance by propsoing a new metric COST, measuring the hardware configuration required for a system to outperform a competent single-threaded implementation. Many of them introduce parallelizable overheads and scalaility without considering performance such as GraphLab and Spark.
The authors write simple single-threaded implementations that solve the same problem as those system designed for multi-core and distributed system, and run it on high-end laptop using the same datasets like twitter with PageRank and compare to multicore systems.
The paper presents the measurements from various data-parallel and graph processing system, showing that many systems have increasing COST.

The problem addressed by the paper is important in the field of big data, especially when the spark is getting more and more popular in distributed systems. When researchers or developers are trying to build big data platforms, they must think of the tradeoffs between COST and scalability.
In my opinion, the solution doesnot seem right to me. Though it provides some meaningful metrics, but considering the practicalities of hardware costs, fault tolerance and configurations in real-world scenarios, performance is not the most important factor, we still need to think about the tradeoff among many factors.
The paper is enjoyable and thought-provking, I had implemented a MapReduce-based search engine on multiple cheap EC2 instances in another distributed system course, and it did achieve scalability and speedup. The paper’s unique perspective on evaluating big data systems gives me some new insights into performance and scalability.

We can still learn some lessons from COST that can be applied to MultiKernel, for examples, when evaluating the MultiKernel, the paper does not give the latency metrics
when conducting experiments but only the throughput. It could be some tradeoffs between scalability and performance.
 -->
