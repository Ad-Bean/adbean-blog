+++
title = 'Paper Reading: MB2: Decomposed Behavior Modeling for Self-Driving Database Management Systems'
date = 2024-04-05T09:50:55-04:00
draft = false
tags = ['Paper Reading']
+++

## MB2: Decomposed Behavior Modeling for Self-Driving Database Management Systems

self-driving database management systems

## ABSTRACT

Database management systems (DBMSs) are notoriously difficult to deploy and administer.**self-driving DBMS** is to remove these impediments by **managing itself automatically**

predict the DBMS’s runtime behavior and resource consumption.

ModelBot2 e2e framework for constructing and maintaining prediction models using **machine learning** (ML) in self-driving DBMSs.

1. decomposes a DBMS’s architecture into fine-grained operating units that make it easier to **estimate the system’s behavior** for configurations
2. ModelBot2 then provides an offline execution environment to exercise the system to produce the training data used to train its models.

We integrated ModelBot2 in an in-memory DBMS and measured its ability to predict its performance for OLTP and OLAP workloads running in dynamic environments. We also compare ModelBot2 against state-of-the-art ML models and show that our models are up to 25×more accurate in multiple scenarios

> 完全自治的数据库，用 ML 预测和修改配置。和同样是 Andy 组的 Ottertune 区别在哪呢？

## INTRODUCTION

A self-driving DBMS can configure, tune, and optimize itself **without human intervention**, even as the application’s workload, dataset, and operating environment evolve.

Such automation seeks to remove the complications and costs involved with **DBMS deployments**. The core component that underlies a self-driving DBMS’s decision-making is **behavior models** [51]. These models estimate and explain how the system’s performance changes due to a **potential action** (e.g., changing knobs, creating an index). This is similar to how self-driving vehicles use physical models to guide their autonomous planning [49].

Techniques for constructing database **behavior models** fall under two categories: (1) **“white-box” analytical** methods and (2) **ML methods**. Analytical models use a **human-devised formula** to describe a DBMS component’s behavior, such as the buffer pool or lock manager [42, 45, 74]. These models are customized per DBMS and version. They are difficult to migrate to a new DBMS and require redesign under system updates or reconfiguration. Recent works on using ML methods to construct models have shown that they are more adaptable and scalable than white-box approaches, but they have several **limitations**. These works mostly target isolated query execution [9, 17, 20, 34, 40]. The models that support concurrent queries focus on real-time settings where the interleaving of queries is known [16, 68, 72], but a self-driving DBMS needs to plan for future workloads without such accurate information [37]. Many ML-based models also rely on dataset or workload-dependent information [16, 40, 58]; thus, a DBMS cannot deploy these models in new environments without expensive retraining.

> white-box analytical 是什么？为什么指的是 Buffer Pool 和 Lock Manager？白盒是开发人员可以看到所有的内部组件和源码，所以能够知道 buffer pool 和 lock manager 中的内容，甚至预测吗？
>
> 使用 ML 方法来构建模型，更适合扩展，但存在局限，之前的工作主要集中在隔离的查询，而支持并行查询的模型主要集中在实时设置并且交织查询已知。完全自治的 DBMS 无法知道准确的信息，而且一些 ML 预测模型依赖于数据集/workload 信息。甚至还需要训练。

本文提出了 ModelBot2 (MB2), generates **behavior models** that estimate the performance of a self-driving DBMS’s components and their interference during concurrent execution. 使得 DBMS planning components 可以推理对 action 的影响，期望收益。比如，MB2 可以回答创建索引需要多长时间，索引创建会怎么影响系统性能，新索引会如何加速 workload's query。

MB2 主要思想是分解 DBMS 的内部结构，分成了几个小的、互相独立的操作单元 operating units (OUs) (e.g., building a hash table, flushing log records).

MB2 then uses ML methods to **train an OU-model** for each OU that **predicts its runtime** and **resource consumption** for the **current DBMS state**。 小的 OU 单元模型需要更少的训练时间，并且对每个 DBMS 组件都有性能 insight。推理时，MB2 结合所有 OU-models 预测 DBMS 对未来的 workload 的性能和系统状态（工作量也是预测的）。

multi-core environments with concurrent threads, MB2 also estimates the interference between OUs by defining the OU- models’ outputs as a set of measurable performance metrics that summarizes each OU’s behavior

> 拆成更小单元，训练每个小的，联合预测。评测结果支持 OLTP OLAP 工作负载，with a minimal loss of accuracy

## BACKGROUND AND MOTIVATION

类比 self-driving DBMS 和无人驾驶，同样有 1. forecasting 2. behavior model 3. planning system

The **forecasting system** is how the DBMS observes and predicts the application’s future workload

The DBMS then uses these **forecasts** with its **behavior models** to predict its **runtime behavior** relative to the target **objective function** (e.g., latency, throughput). The DBMS’s planning system selects actions that **improve this objective function**
