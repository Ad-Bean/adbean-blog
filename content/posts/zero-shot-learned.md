+++
title = 'Paper Reading: Zero-Shot Cost Models for Out-of-the-box Learned Cost Prediction [VLDB 2022]'
date = 2024-03-16T09:58:30-04:00
draft = false
categories = ['Learned Data Systems']
tags = ['Paper Reading']
+++

## Zero-Shot Cost Models for Out-of-the-box Learned Cost Prediction [VLDB 2022]

## Abstract

本文介绍了 zero-shot cost model，该模型可以使学习的成本估算能够 generalizes to unseen databases。与最 state-of-the-art 的工作负载驱动的方法相反，这些方法不必在每个新数据基础上执行大量 training queries ，zero-shot cost models thus allow to instantiate a learned cost model out-of-the-box **without expensive training data collection**。为了 zero-shot cost models，本文提出 a new learning paradigm based on **pre-trained cost models**。As core contributions to support the transfer of such a **pre-trained cost model** to **unseen databases**, we introduce a new model architecture and **representation technique for encoding query workloads as input** to those models. As we will show in our evaluation, zero-shot cost estimation can **provide more accurate cost estimates** than state-of-the-art models for a wide range of (real-world) databases without requiring any query executions on **unseen databases**. Furthermore, we show that zero-shot cost models can be used in a few-shot mode that further improves their quality by retraining them just with a small number of additional training queries on the unseen database.

> 以后尽可能少地直接机翻原论文了，太多名词机翻效果很差，反而降低阅读速度。基本上读完就写一点感想。

## Introduction

Motivation: **Accurate physical cost estimation** (i.e., estimating query latencies) is crucial for **query optimization** in DBMSs. Classically, cost estimation is performed using models that make several simplifying assumptions. As a result, such models often over or underestimate runtimes, leading to **suboptimal planning decisions** that degrade the overall query performance. Recently, machine learning has thus been used for learned cost models that do not need to make such simplifying assumptions.

While it was shown that the **cost estimates** of such learned cost models are significantly **more accurate** than those of the traditional cost models, the existing approaches rely on **workload-driven learning** where models have to observe thousands of queries on the same database for which the cost prediction should be performed. This workload execution is required to gather the training data which can take hours (or days) since tens of thousands of queries need to be executed on potentially large databases

In Figure 1, we show the **cost estimation accuracy** depending on how many hours we allow for gathering the **training data for a workload-driven model**. As we can see, even for a medium-sized database such as IMDB, it takes more than 5 hours of running queries on this database to gather enough training data such that the cost estimation model can provide a decent accuracy.

Unfortunately, **collecting training data** by running queries is not a one-time effort. In fact, the training data collection has to be repeated for every new database a learned model should be deployed for. This is due to the fact that current model architectures for workload-driven learning tie a **trained model to a particular database instance**. Consequently, **for every (new) unseen database** we not only have to train a model from scratch but also gather training data in the form of queries. And even for the same database, in case of **changed data characteristics** due to updates, training data collection needs to be repeated. Overall, these repeated high costs for obtaining training data for unseen databases render workloaddriven learning unattractive for many practical deployments.

> 这里几个名词很奇怪 unseen database 和他 database 的概念也不同，实际上指的是具有特定数据特点的数据集。
>
> 对于 query optimization 需要收集很多数据集，并且精读不够高，而且对于新的数据集又要重新收集。zero-shot 希望能用一种训练模型来泛化数据集。

![](https://s2.loli.net/2024/03/16/QqEyBkIbfGhPSmr.png)

Contributions: In this paper, we thus suggest **a new learning paradigm for cost estimation called zero-shot cost models** that reduces these high efforts. The general idea behind zero-shot cost models is motivated by recent advances in **transfer learning of models**. Similar to other approaches such as **GPT-3** which **enable zero-shot learning for NLP**, a zero-shot cost model is trained on a wide collection of different databases and workloads and can thus **generalize out-of-the-box** to a completely **unseen database** without the need to be trained particularly on that database. In fact as depicted in Figure 1, zero-shot cost models can provide a high accuracy and often even outperform existing workload-driven approaches that have been trained on large sets of training queries. Moreover, as we also show in Figure 1, zero-shot cost models can additionally be **fine-tuned on the unseen database** with just a few training queries and the resulting few-shot models further improve the accuracy.

> 本文的想法是类似语言模型训练，进行无标注的大量预先训练，所以可以对没见过的数据集直接进行代价估计，具有较高的准确率，甚至比 workload-driven 的表现好，同时也可以 fine-tuned。

One could now argue that it might be a significant effort to **collect sufficient training data across databases for pre-training a zero-shot model**. However, in contrast to workload-driven models which require training data for every unseen database, training data collection is a **one-time effort**; i.e., once trained the zero shot model can be used for any new unseen database. In fact, in our evaluation we show that zero-shot models can provide high accuracies for a **wide-variety of real-world databases**. Moreover, for historical traces can be used which eliminates the need to collect any training data. For example, cloud providers such as AWS, Microsoft, or Google, typically anyway keep logs of their customer workloads which could directly be used as training data for zero-shot learning without collecting any further training data

A key aspect to enable zero-shot learning is that **a cost model can be transferred to new (unseen) databases**, i.e., the models leverage observed query executions on a variety of different databases to **predict** runtimes on new (unseen) databases. However, state-of-the-art model architectures used for workload-driven learning do not support this training and inference mode since they are tied to a **particular database**. As a core novel contribution for zero-shot cost models we thus devise a new model architecture based on a representation of queries that generalizes across databases using a **transferable representation** with features such as the tuple width that can be derived from any database. Moreover, zero-shot models **separate concerns**; i.e., data characteristics of a new database (e.g., rows of tables) are not implicitly learned as in classical workload-driven learning (which hinders generalization), but are provided as input to the model

> separate concerns 是什么意思？新数据集的特征不同？和不够足够高的精度？

Another core question for zero-shot models is at which point a **sufficient** amount of different training databases and workloads was observed to generalize robustly to unseen databases. To answer this question, as a second contribution in this paper we derive a method to **estimate how accurate the runtime estimations of zero-shot models** will be for unseen databases. We also discuss how to address cases of workload drifts where the zero-shot models are expected to generalize less robustly. Furthermore, we also show that zero-shot models are widely applicable beyond cost models for **query optimizers** for single-node DBMSs which is the main focus of this paper. For instance, we believe that zero-shot cost models can be **naturally extended to to distributed DBMS** or even other use cases such as providing cost estimates for design advisors where the goal is to automatically find a suitable database design (e.g., a set of indexes) for a given workload.

Finally, in our extensive experimental evaluation, we verify that zero-shot cost models **generalize robustly to unseen databases and workloads** while providing **cost estimates** which are more accurate than those of workload-driven models. As part of this evaluation, we also provide a new **benchmark** (beyond JOB) which is necessary to evaluate cost estimation models more broadly on a variety of (real-world) databases. We will make this benchmark including query executions for training cost models publicly available and hope that it will benefit future research in learned cost estimation and potentially beyond.

> 没接触过这方向的工作，好像是用数据集做一些比较泛化的工作

## Overview

### Problem Statement

> 主要目标是预测在 unseen database 上的 query latency，并且没有提前知道任何 query。
>
> 文章说 zero-short cost estimation 和 stoa workload-driven cost model 有强烈的对比。并且希望能够在流数据库、图数据库做一些未来的工作。

Finally, while we believe that zero-shot learning for DBMSs is more **generally applicable**, we restrict ourselves in this paper to **cost estimations** for **relational DBMSs** (both single-node and distributed). In particular, zero-shot cost models for other types of systems such as graph-databases or streaming systems are interesting avenues of future work.

### Our Approach

A key challenge for developing zero-shot cost models is the question how to design a model that allows to **generalize across databases**.

Typically, these consist of two models: **a database-agnostic model** to estimate the runtime cost and a database dependent model (e.g., histograms) to capture data characteristics. When predicting the cost of a query, the estimated cardinalities and other characteristics (i.e., outputs of the database-dependent models) serve as input to the general database-agnostic cost model which captures the general system behavior (e.g., the costs of a sequential scan grows linearly w.r.t. the number of rows). While the classical models are lightweight, they often largely under or overestimates the true costs of a query since models are too simple to capture complex interactions in the query plan and data.

> classical cost model 包含两个 model，估计运行时 cost 和捕获 model？

这个模型分开了关注点，用一个更加丰富的具有知识的 learned 模型， similarly takes data characteristics of the unseen database as input to predict query runtimes in a database-agnostic manner。用不同的 query plan 和 data characteristic of the plan 进行训练。

to predict the runtime of a query plan on a new (unseen) database, we feed the query plan together with its **data characteristics** into a zero-shot model.

> data characteristics 是什么？tuple width, intermediate cardinalities，所以需要数据驱动学习？但这是否违反了 zero-shot？还是说只需要训练数据特征而不是 query

![](https://s2.loli.net/2024/03/17/HzPkYLps5lj1FZo.png)

> 训练然后推理，和语言模型类似
>
> 但是怎么 estimate runtime of a plan given its data characteristics?

本文提出了一种新的 query 表示方法，旧的不是 transferable 也不是 zero-shot。这种表现方式 completely relies on features that can be derived from any database to allow the model to generalize to unseen databases. For example, predicates for **filter operations** in a query are encoded by the general predicate structure (e.g., which data types and comparison operators are used in a predicate) instead of encoding the literals. In addition, data characteristics of a filter operator (e.g., input and output cardinality to express the selectivity) are provided as additional input to a zero-shot model. That way, a zero-shot model can learn the runtime overhead of a filter operation based on database-agnostic characteristics. We present details of our query representation in Section 3.

Finally, a last important aspect of zero-shot cost models is that they can easily be extended to **few-shot learning**. Hence, instead of using the zero-shot model out-of-the box (which already can provide good performance), one can **fine-tune** the model with only a few training queries on an unseen database

## Zero-Shot Cost Models

1. how we devised such a **transferable** query representation
2. how **inference and training** of a zero-shot model that uses this representation works

### Query Representation

State-of-the-art workload-driven models for cost estimation do not use **transferable** query representations and can thus only be used on the database they were trained on

1. Query Representation for Workload-Driven Models: hard-code the model against a single database, **attribute names** (e.g., those used in filter predicates) are typically encoded using a one-hot encoding assigning each attribute present in the database a specific position in a feature vector. For instance, the attribute `production_year` of the IMDB dataset might be encoded using the vector (0,1,0)(assuming that there are only three attributes in total). If the same model should now be used to predict query costs for the SSB dataset, some attributes might not even exist or even worse they might exist but have very different data distributions or even a different data type. In fact, non-transferable feature encodings are not only used for **attributes** but in various places of the query representation such as encoding table names or literals in filter predicates.

2. Query Representation for Zero-Shot Cost Models: Hence, for **zero-shot cost models** we require a new query representation that is **transferable** across databases. The main idea of the **transferable representation** we suggest in this paper is shown in Figure 3. At the core, a **query plan** and the **involved tables and attributes** are represented using a **graph** where graph nodes use **transferable features** 1 (i.e., features that provide meaningful information to **predict runtime** on different databases). This representation then serves as input for the training and inference process of zero shot cost models 2 ~ 4 that we explain in the **subsequent sections**. In the following, we discuss the **graph encoding** of the transferable featurization in detail

3. Graph Encoding of Query Plans: While **graph-based** representations have been already used to represent query operators of a query plan [28], our representation has **significant differences**. First, as shown in Figure 3 1 , our representation not only **encodes physical plan operators** as nodes (gray) in the graph as in previous work [28], but it also **covers all query plan information more holistically** using different nodes types for **attributes** (green), **tables** (blue) as well as predicate information (red). Second, as discussed before, previous approaches also covered such information, however, they used one-hot-encodings (which are non-transferable) while our representation captures the query complexity in a transferable way

For instance, to encode filter predicates, different from previous approaches we encode the predicate structure as nodes (red) without literals. In particular, we encode information such as data types of the attributes and operators used for comparisons. For example, the filter predicate company_type_id=2 for the query 0 in Figure 3, is encoded using an attribute node (𝑥5) with the comparison node = (𝑥7). As such, a zero-shot cost model provided with the transferable features (e.g., intermediate cardinalities which are given by the data-driven models) can infer the complexity of the predicates to estimate the query runtime

![](https://s2.loli.net/2024/03/17/pEd3QMsvJArb4ZO.png)

Transferable Featurization: nodes in the graph 1 (e.g., plan operators as shown in gray) are **transferable**. In particular, when used on **different databases**, features should not encode any implicit information that hinder the **transfer of the model to a new unseen database**. The set of such features used for the different node types in our query representation is depicted in Table 1. For instance, attribute nodes (green) use features such as the **data type** or the **width** in bytes. Similarly, for tables (blue nodes), we use other **transferable features** (e.g., the **number of rows** as well as the **number of pages on disk**)

transferable features can either characterize the query plan (e.g., operator types) or represent the data characteristics (e.g., intermediate cardinalities)

For transferable features that **represent data characteristics** many can be derived from the **metadata** of a database (such as the the **number of rows** of a table node). However, some other features that represent data characteristics — e.g., the **estimated output cardinality** of an operator node — require more involved techniques. In Section 3.4, we discuss alternatives of how we provide **estimated output cardinalities to zero-shot cost models.**

> 还是没理解 Transferable feature 是什么，是能代表一个数据的特征和 query plan 吗？intermediate cardinalities 也不知道是什么，是 join cardinality 吗？传统是基于直方图，之前数据库课上也见过一些，用直方图做一些统计和估计。

### Inference on Zero-Shot Models

用了每个图节点的特征向量来做 MLP 多层感知器（机器学习），加上 GNN 做消息传递，用 DAG 表示查询

We thus use a novel **bottom-up message passing scheme** through the graph (i.e., in topological ordering) to obtain an updated hidden state ℎ′𝑣 of a node 𝑣 that contains all information of the child nodes.

> 机器学习内容，没太理解

### Training Zero-Shot Models

在几个数据集和查询集上做了训练

### Deriving Data Characteristics

zero shot model 和单个数据集的分布无关，所以能预测 unseen dataset，同时为了实现该特性，需要提供数据的特征（属性宽度、页面数量、表格行数量）

> 所以需要人为 label？没有解释清楚到底怎么预测 cardinality estimation
>
> 本模型是和 traditional approach 和 data-driven model 进行比较，但 zero-shot 最好可以和后者结合，传统方法作为 fallback。

In our evaluation, we see that zero-shot models can still produce **reasonable estimates** even if only **cardinalities estimates** from **traditional models** are available

## Robustness of Zero-Shot Models

针对 unseen datasets, zero-shot model 具有优势，但在什么时候可以观察到足够多的不同训练数据库(和工作负载) ，从而可以鲁棒地推广到 unseen？

零拍模型的运行时估计的鲁棒性，We then discuss a simple method to detect cases of workload drifts (i.e., the queries at runtime have substantially different characteristics than the training queries) as well as strategies how to tackle this problem.

### Estimating the Generalization Performance

1. simply esimate the generalization errors
2. (actually use) estimate if **additional training databases** will improve the generalization performance

所以训练时用了 subsets，然后用更大的 train set 如果没有改进，则无法提高 generalization 能力

### Tackling Workload Drifts

new database and workload, we suggest a strategy to **detect cases of workload drifts** by monitoring the test error and propose to **tackle workload-drifts** using few-shot learning

> 这是一个蛮新奇的 idea，用微调过的模型来检测和解决 workload drifts

## Extensions of Zero-Shot Cost Models

zero-shot cost estimation can be **extended in various directions**.

### Distributed DBMSs

本文的研究对象还是 单机 DBMS，这一节讨论了如何支持分布式 DBMS，尤其是当查询执行时数据会被打乱 shuffle。并且讨论了一些列存/行存时，查询优化的区别。

然后本文提出了对 查询 进行扩展编码，包含 **operator nodes** for data shuffling as well as **encode data formats** (column or row) as a feature of the table node

### Physical Design Tuning

没懂

### Other Directions

hardware parameters, database knob configurations...

## A New Benchmark

本文提出了一个新的 benchmark

### Design Decisions

本文认为传统 TPC-H TPC-DS or SSB 不足够评估成本预测模型，并且因为这些数据是合成的，没法捕捉到相关性来提供 cardinality estimation，也有人基于 IMDB 数据集组了一些新的 workload 病故学习的成本和 cardinality estimation。但这也不能评估 zero-shot，因为要在不同的数据集上训练，需要一个跨越多种数据集的 benchmark。

### Datasets

correlations hardly resemble data distributions found in the real-world

本文使用了真实世界数据集，也加入了 SSB 和 TPC-H

### Workloads and Traces

20 个数据集 + 生成的查询

## Experimental Evaluation

![](https://s2.loli.net/2024/03/17/NeI2ahUsbTGYg1P.png)

1. Zero-Shot Accuracy, predict costs for unseen databases.
2. Zero-Shot vs. Workload-Driven, compare the training overhead and accuracy with state-of-the-art workload driven approaches
3. Generalization, workload drifts
4. Extensions, distributed DBMSs and different physical designs
5. Training and Inference Performance, compare training efforts to workload-driven models
6. Ablation Study, alternatives of zero-shot models, how many database are sufficient for zero-shot cost models to generalize

> 实验结果就不看了，文章没有深入讨论他们的数据集的区别，实验结果看起来和 workload-driven 看上去也差不多

## Conclusion

> The paper proposes a novel zero-shot cost model that enables learned cost estimation to unseen databases without extensive training on new datasets. The architecture of zero-shot cost model is very similar to the language model in NLP. It adopts the idea of ​​pre-training, so that it can perform well on different datasets, and can also be fine-tuned for special datasets. The paper details the methods used to enable zero-shot cost models, including the transferable query representation, training zero-shot models and deriving data characteristics.The paper also discusses the robustness and scalability of zero-shot models (distributed database). The paper proposes a new benchmark spanning multiple datasets, and gives experimental results, proving that zero-shot models can perform excellent cost estimation on unseen datasets

> 1. The paper proposes a novel zero-shot cost model, and suggests a learning paradigm based on pre-trained cost models. A new model architecture and a representation technique for encoding query workloads as input to these models are introduced, which together allow a zero shot cost model to generalize to an unseen database.
> 2. The paper derives a method to estimate how accurate the runtime estimations of zero-shot models will be for unseen databases, and provides a new benchmark necessary to evaluate cost estimation models more broadly on a variety of real-world databases.
> 3. The paper not only discusses the zero-shot cost model based on a single-node DBMS, but also discusses future work, proving that zero-shot cost estimation can also be achieved in a distributed database. It can also be combined with data-driven models or even fine-tuned zero-shot models for better cost estimations and accuracy.

> 1. The zero-shot cost model may not always accurately predict costs for queries on databases that significantly differ from those seen during the model’s training, because it relies on pre-trained models and there are often large differences between real-world datasets and query workloads.
> 2. This paper somewhat confuses the concepts of datasets and databases. And while zero-shot models can eliminate the need for extensive training on each new database, they may still require some fine-tuning or a few-shot learning approach to achieve the best performance on specific datasets and queries, which is not completely zero-shot.
> 3. Although the paper has used 20 widely different data sets for benchmarking, 19 of them were used for training and only 1 was used for testing. It should be tested with more data sets.

> First, I would try to create a larger and more diverse training and testing dataset that covers a wide range of database schemas and query workloads and extend the benchmark used for evaluating cost models to include more real-world databases and complex queries. This would provide a more comprehensive assessment of the model’s performance and its ability to generalize. Second, extending the zero-shot model to distributed databases or more databases seems very feasible. I might try to expand this part a little further, especially to implement zero-shot cost models fordistributed stream processing or graph databases, because the queries and datasets for stream processing and graph database will be very different. The other parts of the paper are very perfect, especially the architecture that draws on language models, which is very clever.
