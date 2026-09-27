+++
title = 'Paper Reading: MB2: Decomposed Behavior Modeling for Self-Driving Database Management Systems'
date = 2024-04-05T09:50:55-04:00
draft = false
categories = ['Learned Data Systems']
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

multi-core environments with concurrent threads, MB2 also estimates the interference between OUs by defining the OUmodels’ outputs as a set of measurable performance metrics that summarizes each OU’s behavior

> 拆成更小单元，训练每个小的，联合预测。评测结果支持 OLTP OLAP 工作负载，with a minimal loss of accuracy

## BACKGROUND AND MOTIVATION

类比 self-driving DBMS 和无人驾驶，同样有 1. forecasting 2. behavior model 3. planning system

The **forecasting system** is how the DBMS observes and predicts the application’s future workload

The DBMS then uses these **forecasts** with its **behavior models** to predict its **runtime behavior** relative to the target **objective function** (e.g., latency, throughput). The DBMS’s planning system selects actions that **improve this objective function**

> 预测 + action + 目标函数
>
> Behavior models 是基础

### Behavior Modeling

Given an **action**, a self-driving **DBMS’s behavior models estimate** (1) **how long** the action takes, (2) **how much resource** the action consumes, (3) how applying the action **impacts the system performance**, and (4) how the action impacts the system once it is deployed.

> 对比 Analytical models 和 Behavir Model，前者最近可以用 ML 分析，用 query plan information (cardinality estimates) 估计性能参数 (latency)

尽管 ML 方法可扩展，但还需要调整或重新训练，而且不支持事务 workload，也没考虑 DBMS 维护操作（GC）。一些方法支持并发查询，但对完全自治的 DBMS 还不够，因为未来工作量未知。

举了个例子，如果删除二级索引，DBMS 会重新加回来，启动之前会用 planning component 和 behavior model 预测什么索引比较好，还会选择创建索引的线程数。

### Challenges

ML 方法建造一个自治 DBMS 会遇到的问题：

1. High Dimensionality 高纬度，一个大模型捕捉 DBMS 所有方面，workload, configuration, actions. 自治 DBMS 还需要考虑 runtime state (e.g., database contents, knob configurations), interactions with other components (e.g., garbage collection), and other autonomous actions (e.g., building indexes), which further increases dimensionality
2. Concurrent Operations 并行操作，动态环境 query 之间的干涉 interference，资源竞争。
3. Training, Generalizability, and Explainability，资源收集比较困难，构建索引耗时很久。

## OVERVIEW

MB2, embedde behavior modeling framework for self-driving DBMS.

1. MB2 generates model offline in a dataset and workload independent manner
2. MB2's models are debuggable, explainable, and adaptable.

![](https://s2.loli.net/2024/04/07/qeu2KP76GgmdZ9w.png)

decompose the DBMS into independent operating units (OUs). An OU represents a step that the DBMS performs to **complete a specific task**. These tasks include query execution steps (e.g., building join hash tables (JHTs)) and internal maintenance steps (e.g., garbage collection). DBMS developers are responsible for creating OUs and deciding their boundaries based on the system’s implementation.

MB2 **pairs each OU with an OU-runner** that exercises the OU’s corresponding DBMS component by sweeping the component’s input parameter space. MB2 then provides a lightweight **data collection layer** to transform these OU-runners’ inputs to OU features and track the OU’s behavior metrics (e.g., runtime, CPU utilization). For each OU, MB2 automatically searches, trains, and validates an OU-model using the data collected from the related OU-runner

simulate concurrent enviornments, MB2 uses concurrent runners to execute e2e workloads.

![](https://s2.loli.net/2024/04/07/EJNafQ5emIut2Lw.png)

> 将数据库操作拆成 OU 和对应的 OU runner，比如创建索引、垃圾回收、Flush Log，开发者来提供 OU 集合。OU runner 用于训练 OU 模型，需要知道有多少 tuple, column 等等，以及 output feature: CPU, memory utilization, IO 等等。

MB2 also **estimates** the effect of building the index on the regular workload by converting its queries into OUs and predicting their performance. The DBMS’s **planning system** then decides whether to build this index and provides explanations for its decision based on these detailed predictions

Assumptions and Limitations:

1. framework uses a **forecasting system** to generate estimations for future workload arrival rates in fixed intervals (e.g., a minute/hour). The workload forecasting system cannot predict **ad-hoc queries** it has never seen before. Thus, we assume the DBMS executes queries with a **cached query plan** except for the initial invocation
2. the target system is an in-memory DBMS. MB2 does not support disk-oriented DBMSs with buffer pools. This assumption **simplifies** MB2’s behavior models since it does not have to consider what pages could be in memory for each query. Estimating cache contents is difficult enough for a single query. It is more challenging when evaluating a sequence of forecasted queries.
3. MB2 supports OLTP and OLAP, and mixed workloads. We assume that the DBMS uses **MVCC** and MB2 supports **capturing lock contention**. MB2 does not, however, **model transaction aborts** due to data conflicts because it is challenging to **get precise forecasts of overlapping queries**
4. MB2’s OU-models’ input features contain the **cardinality estimation** from the **DBMS optimizer**, which is known to be error-prone Our evaluation shows that MB2’s prediction is insensitive against cardinality estimation errors within a reasonable range (30%). There are recent works that **use ML to improve an optimizer’s cardinality estimations** , which MB2 may leverage
5. Lastly, while MB2 supports **hardware context** in its models (see Sec. 4.2), we **defer the investigation** on what features to include for different hardware (e.g., CPU, disk) and environments (e.g., bare-metal, container) as future work.

> 没法预测 ad-hoc 用户自己的数据集，预测缓存非常困难。也只支持内存数据库。

## OU-MODELS

create a **DBMS’s OU-models** with MB2. The goal of these models is to **estimate the time and resources** that the DBMS will consume to execute a query or action. A self-driving DBMS can **make proper planning decisions** by combining these **estimates** from multiple queries **in a workload forecast interval**. In addition to accuracy, these models need to have three properties that are important for self-driving operations: (1) they provide **explanations** of the DBMS’s behavior, (2) they support **any dataset and workload**, and (3) they **adapt to DBMS software updates**

> 除了精确度，能够提供解释、支持任何数据集、支持动态更新

### Principles

Developers use MB2 to **decompose the DBMS into** OUs to build explainable, adaptable, and workload independent behavior models.

> 开发人员来做 decompose, 因为 NoisePage 是 CMU 自己的关系型数据库，可以拆除很多操作单元，比如 Hash Table Build, Probe 等等

![](https://s2.loli.net/2024/04/07/QzVqtXFArI7Y4HS.png)

1. Independent: The runtime behavior of an OU must be **independent of other OUs**. Thus, changes to one OU do not directly affect another unrelated OU. For example, if the DBMS changes the knob that controls the **join hash table size** then this does not change the resource consumption of the DBMS’s **WAL component** or the resource consumption of sequential scans for the same queries.

> 比较好奇怎么拆成独立的 OU，如果改 join hash table size，会不会影响 join table probe OU 呢

2. Low-dimensional: An OU is a **basic operation** in the DBMS with a small number of input features.
3. Comprehensive: Lastly, the framework must have OUs that encompass all DBMS operations which consume resources. Thus, **for any workload**, the OUs **cover the entire DBMS**, including background maintenance tasks (e.g., garbage collection) and self-driving actions that the DBMS may deploy on its own (e.g., index creation).

> OU 到底是对 DBMS 设置，还是针对数据集设置的？

### Input Features

After deciding which OUs to implement, DBMS developers then specify the OU-models’ input features based on their **degree of freedom**

1. the **amount of work** for a single OU invocation or multiple OU invocations in a batch (the number of tuples to process)
2. the parallel invocation status of an OU (e.g., number of threads to create an index), and
3. the DBMS **configuration knobs** (e.g., the execution mode).

> degree of freedom 统计学概念？

![](https://s2.loli.net/2024/04/07/jgJdQK4GrBAu9Oo.png)

Although **humans select the features for each OU-model**, the features’ **values are generated automatically** by the DBMS based on the workload and actions. Some features are generic and will be the same **across many DBMSs**, whereas others are specific to a DBMS’s implementation. We categorize OUs into three types based on their behavior pattern, which impacts what information the input features have

1. Singular OUs: The first type of OUs have input features that represent the amount of work and resource consumption for a **single invocation**. These include NoisePage’s execution category OUs in Table 1. Almost all its execution OUs have the same seven input features. The first six features are related to the relational operator that the OU belongs to: (1) **number of input tuples**, (2) number of columns of input tuples, (3) average input tuple size, (4) estimated key cardinality (e.g., sorting, joins), (5) payload size (e.g., hash table entry size for hash joins), and (6) number of loops (only for index nested loop joins). Lastly, the seventh feature is an **execution mode flag** that is specific to NoisePage; this indicates whether the DBMS executes a query with its **interpreter** or as **JIT-compiled**. NoisePage’s networking OU also belongs to this type since network communication is discrete amount of work.
2. Batch OUs: The second type of OUs have input features that represent a batch of work across multiple OU invocations. It is challenging to derive features for these OUs since a single invocation **may span multiple queries based on when those queries arrive and the invocation interval**. (log flushes) To address this, we define the log flush OU’s input features to represent the total amount of records generated by the set of queries predicted to arrive in a workload forecasting interval: (1) the total number of bytes, (2) the total number of log buffers, and (3) the log flush interval. These features are independent of what plans the DBMS chooses for each query
3. Contending OUs: The last type of OUs are for operations that may **incur contention in parallel invocations**.

> 独立的 OU，批处理 OU，竞争 OU。第一个可以迁移，因为 input 类似，比如行数、列数、行大小、基数等等。第二个可能跨 query 比如日志落盘。第三个是并行可能引起锁竞争的 OU，比如多线程构建索引，需要知道行数、key 数量、key 大小、key 基数估计、等等

A self-driving DBMS must also predict **how changes to its configuration knobs impact its OUs**. We categorize these tunable knobs into **behavior knobs** and **resource knobs**.

### Output Labels

Every OU-model produces the same output labels, (1) elapsed time, (2) CPU time, (3) CPU cycles, (4) CPU instructions, (5) CPU cache references, (6) CPU cache misses, (7) disk block reads, (8) disk block writes (for logging), and (9) memory consumption.

These **metrics explain what work an OU does independent of which OU it is**. Using the same labels enables MB2 to combine them together to **predict the interference among concurrent OUs**. They also help the self-driving DBMS estimate the **impact of knobs** for resource allocation. For example, the OU-models can predict each query’s memory consumption by predicting the memory consumption for all the OUs related to the query. A self-driving DBMS evaluates what queries can execute under the knob that limits the working memory for each query, and then sets that knob according to its memory budget. Similarly, CPU usage predictions help a self-driving DBMS evaluate whether it has assigned enough CPU resources for queries or its internal components

> OU-model 输出 label，所以有利于 debug

**Output Label Normalization**: We now discuss how MB2 normalizes OU-models’ output labels by tuple counts to improve their accuracy and efficiency. DBMSs can execute queries that range from accessing a single tuple (i.e., OLTP workloads) to scanning billions of tuples (i.e., OLAP workloads). The former is fast to replicate with OU-runners, but the latter is expensive and time-consuming. To overcome this issue, MB2 employs a normalization method inspired by a previous approach for **scaling query execution modeling** [34]. We observe that many OUs have a known complexity based on the number of tuples processed, which we denote as 𝑛. For example, if we fix all the input features except for 𝑛, the time and resources required to build a hash table for a join are O(𝑛). Likewise, the time and resources required to build buffers for sorting are O(𝑛 log 𝑛). We have found in our evaluations with NoisePage that with typically less than a million tuples, the output labels converge to the OU’s asymptotic complexity multiplied by a constant factor

> 如何处理 label？使用 normalize, divides the outputs by the related OU’s complexity **based on the number of tuples**, 但是构建哈希表时，由于 NoisePage 使用不同的哈希表处理 join 和 agg：预先分配内存给 join，哈希表会对插入的 unique key 额外增长。所以 MB2 需要用除法来归一化。

## INTERFERENCE MODEL

The OU-models capture the **internal contention on data structures or latches within an OU** (Sec. 4.2). But there can also be interference among OUs due to resource competition, such as CPU, I/O, and memory.2 MB2 builds a common interference model to capture such interference since it may impact any OU.

> 论文也观察到，OU 之间也存在资源共享，比如连续运行的 OU 会有缓存局部性。但论文说这是极端情况。

Building such an interference model has two challenges: (1) there are an **exponential number of concurrent OU combinations**, and (2) self-driving DBMSs need to **plan actions ahead of time** [50], but predicting queries’ exact future arrival times and concurrent interleavings is arguably impossible

> 需要提前预测，但不能精确预测并行发生的时间，并且 OU 组合是指数级的

### Input Features

The interference model’s inputs are the OU-model’s output labels for the OU to predict and summary statistics of the OUs forecasted to run **in the same interval** (e.g., one minute).

### Output Labels

The interference model generates the same set of outputs as the OU-models.

## DATA GENERATION AND TRAINING

MB2 is an **end-to-end solution** that enables self-driving DBMSs to **generate training data from OUs** and then **build accurate models that predict their behavior**.

> MB2 如何有利于收集数据和训练模型，并且再次强调需要在离线情况下，非生产环境，运行 MB2。推迟了如何在在线系统收集数据，而不会导致性能观察退化的问题。

### Data Collection Infrastructure

**OU Translator**: This component extracts OUs from query and action plans and then generates their corresponding input features. MB2 uses the same translator infrastructure for both **offline training** data collection and **runtime inference**.

**Resource Tracker**: Next, MB2’s tracker records the **elapsed time** and **resource consumption metrics** (i.e., output labels) during OUs execution. The framework also uses this method for the interference model data since it uses the **same output labels** to adjust the OU-models’ outputs.

**Metrics Collector**: The challenges with collecting training data are that (1) **multiple threads** produce metrics and thus require coordination, and (2) resource tracker can incur a **noticeable cost**. It is important for MB2 to support low-overhead metrics collection to reduce the cost of accumulating training data and interference with the behavior of OUs, especially in concurrent environments

MB2 uses a **decentralized metrics collector** to address the first issue. When the DBMS executes in **training mode**, a worker thread records the features and metrics for each OU that it encounters in its thread-local memory. MB2 then uses a **dedicated aggregator** to periodically gather this data from the threads and **store it in the DBMS’s training data repository**. To address the second challenge, MB2 supports resource tracking **only for a subset of queries** or DBMS components. For example, when MB2 collects training data for the OUs in the execution engine, the DBMS can turn off the tracker and metrics collector for other components

> 数据收集的架构

### OU-Runners

An OU-runner is a **specialized microbenchmark** in MB2 that exercises a single OU. The goal of each OU-runner is to reproduce situations that an OU could encounter when the DBMS executes a real application’s workload. The OU-runners sweep the input feature space (i.e., number of rows, columns, and column types) for each OU with fixed-length and exponential step sizes, which is similar to the grid search optimization

There are two ways to implement OU-runners: (1) **low-level execution code** that uses the DBMS’s **internal API** and (2) high level **SQL statements**. We chose the **latter** for NoisePage because it requires less upfront engineering effort to implement them, and has little to no maintenance costs if the DBMS’s internal API changes

> 用 SQL statement 实现 OU runner 是怎么做到的

MB2 supports modeling OLTP and OLAP workloads. To the best of our knowledge, we are the first to support both workload and **data-independent** modeling for OLTP query execution. Prior work either focused on modeling OLAP workloads [34, 40, 68] or assumes a fixed set of queries/stored procedures in the workload [42, 45]. Modeling the query execution in OLTP workloads is challenging for in-memory DBMSs: since OLTP queries access a small number of tuples in a short amount of time, spikes in hardware performance (e.g., CPU scaling), background noise (e.g., OS kernel tasks), and the precision of the resource trackers (e.g., hardware counters) can inflict large variance on query performance. Furthermore, DBMSs typically execute repeated OLTP queries as prepared statements

> 第一个支持 OLAP 和 OLTP 数据集，并且是 workload and data-independent 的，支持 OLTP。但 NoisePage 本身就是事务数据库，MB2 是怎么做到这种特性的。

1. MB2 executes the OU-runners for the OUs in the execution engine with **sufficient repetitions** (10×) and applies robust statistics to derive a reliable measurement of the OU’s behavior.
2. MB2 executes each query for **five warm-up iterations** before taking measurements for the query’s OUs, with all executions of a given query using the same query template. MB2 starts a new transaction for each execution to avoid the data residing in CPU caches. For queries that modify the DBMS state, MB2 reverts the query’s changes using transaction rollbacks. We find the labels insensitive to the trimmed mean percentage and the number of warm-up iterations.

> 多次重复和 warm up，所以这是没法在 磁盘 DBMS 使用 MB2 的原因吗，没有办法预测 buffer pool 中的内容，那 NoisePage 有缓存吗？

MB2 starts a new transaction for each execution to avoid the **data residing in CPU caches**. For queries that modify the DBMS state, MB2 reverts the query’s changes using transaction rollbacks. We find the labels insensitive to the trimmed mean percentage and the number of warm-up iterations.

> 每次都是用事务，避免了 CPU data cache，并且用 rollback 改回了 DBMS 的状态。label 和 trimmed mean percentage and warmup iterations 不敏感。那为什么要 warm up？

### Concurrent Runners

Since OU-runners invoke their SQL queries **one at a time**, MB2 also provides concurrent runners that execute end-to-end benchmarks (e.g., TPC-C, TPC-H). These concurrent runners provide MB2 with the necessary data to train its **interference model**

### Model Training

Lastly, we discuss how MB2 trains its behavior models using the **runner-generated data**. Since OUs have **different input features** and **behaviors**, they may require using **different ML algorithms** that are better at handling their unique properties and assumptions about their behavior. For example, Huber regression (a variant of linear regression) is simple enough to model the filter OUs with arithmetic operations. In contrast, sorting and hash join OUs require more complex models, such as random forests, to support their behaviors under different key number, key size, and cardinality combinations

> 使用不同的 ML 方法，那怎么选？

MB2 trains multiple models per OU and then **automatically selects the one with the best accuracy for each OU**. MB2 currently supports seven ML algorithms for its models: (1) linear regression, (2) Huber regression, (3) support vector machine, (4) kernel regression, (5) random forest, (6) gradient boosting machine, and (7) deep neural network. For each OU, MB2 first trains an ML model using each algorithm under the common 80/20% train/test data split and then uses cross-validation to determine the best ML algorithm to use. MB2 then trains a final OU-model with all the available training data using **the best ML algorithm determined previously**. Thus, MB2 utilizes all the data that the runners collect to build the models. MB2 uses this same procedure to train its interference models.

> 自动选了 ML 方法（精度最高的）

## HANDLING SOFTWARE UPDATES

讨论了 DBMS 可以更新（bug 修复，性能提升等等）的情况下，自动驾驶的 DBMS 应该怎么处理。因为 OUs 之间是独立的，MB2 只需要重新跑受影响的 OU-runners 。 This restricted retraining is possible because the **OUs are independent of each other**. The OU-runners issue SQL queries to the DBMS to exercise the OUs, which means that developers do not need to update them unless there are changes to the DBMS’s SQL syntax. Furthermore, MB2 does not need to retrain its interference models in most cases because resource competition is not specific to any OU.

In NoisePage, we currently use a heuristic for MB2 to retrain the interference models when a DBMS update affects at least five OUs

> 不需要重新训练干扰模型，只需要重新跑 OU runner。除非 DBMS 引入了新的 OU behavior 比如新的组件，MB2 就需要重新运行 OU concurrent runners 生成干扰模型。

## EXPERIMENTAL EVALUATION

NoisePage DBMS: OLTP-Bench testbed as an end-to-end workload generator for the SmallBank, TATP , TPC-C, and TPC-H benchmarks.

> OLAP 和 OLTP 都测了，使用不同的指标

We use two evaluation metrics. For OLAP workloads, we use the **average relative error** ( |𝐴𝑐𝑡𝑢𝑎𝑙−𝑃𝑟𝑒𝑑𝑖𝑐𝑡| 𝐴𝑐𝑡𝑢𝑎𝑙 ) used in similar prediction tasks in previous work . Since OLTP queries have short run-times with high variance, their relative errors are too noisy to have a meaningful interpretation. Thus, we use the **average absolute error** ( |𝐴𝑐𝑡𝑢𝑎𝑙 −𝑃𝑟𝑒𝑑𝑖𝑐𝑡|) per OLTP query template

### Data Collection and Training

MB2’s data collection and model training.

![](https://s2.loli.net/2024/04/07/R17TNgndqQ5AGaE.png)

> 看上去训练数据不是很大，训练时间也不是很久。

### OU-Model Accuracy

the OU-model test relative error averaged across all output labels. More than 80% of the OU-models have an average prediction error less than 20%, which demonstrates the effectiveness of the OU-models.

> OU 模型的预测误差小于 20%，证明了有效性。但没看懂这个结果，用不同的 ML 方法得到的误差都不同，有的很低有的很高，比如随机森林总是很低的误差，论文也提到 OU 模型维度很低，容易出现过拟合。

we show the predictive accuracy of the OU-models for each output label, averaging across all OUs. Most labels have an average error of less than 20%, where the highest error is on the cache miss label. **Accurately estimating the cache misses for OUs is challenging because the metrics depend on the real-time contents of the CPU cache**. Despite the higher cache miss error, MB2’s interference model still captures the interference among concurrent OUs (see Sec. 8.4) because the interference model extracts information from all the output labels. The results in Fig. 6 also show the OU-model errors without output label normalization optimization from Sec. 4.3. From this, we see that normalization has minimal impact on OU-model accuracy while enabling generalizability

> 看上去结果很容易受到 CPU 缓存影响。但前文又说影响不大？能够预测 query runtime 和 workload

### OU-Model Generalization

For a state-of-the-art baseline, we compare against the **QPPNet ML model** for query performance prediction

QPPNet uses a tree-structured neural network to capture a query plan’s structural information. It outperforms other recent models on predicting query runtime especially when generalizing the trained model to different workloads (e.g., changing dataset sizes)

Since NoisePage is an **in-memory DBMS** with a **fused-operator JIT query engine**, we remove any **disk-oriented features** from QPPNet’s inputs and adapt its operator-level tree structure to support pipelines. But such adaptation requires QPPNet’s training data to contain all the operator combinations in the test data pipelines to do inference with the proper tree structure. Thus, we can only train QPPNet on more complex workloads (e.g., TPC-C) and test on the same or simpler workloads (e.g., SmallBank)

> 还是强调了 NoisePage 是内存数据库，而且是 JIT 的

We evaluate MB2 and QPPNet on the (1) TPC-H OLAP workload and (2) TPC-C, TATP, and SmallBank OLTP workloads. To evaluate **generalizability**, we first train a QPPNet model with query metrics from a 1 GB TPC-H dataset and evaluate it on two other dataset sizes (i.e., 0.1 GB, 10 GB).

> 在 TPCH 上训练和评估。QPPNET 的准确性非常好，但在其他数据集上错误率很高。
>
> 而 MB2 比 QPPNET 高 25x 的精度，并且所有 workload size 上预测的精确度都很稳定。

We attribute this difference to how (1) MB2 **decomposes** the DBMS into **fine-grained OUs** and the corresponding OU-runner enumerates various input features that cover a range of workloads, and (2) MB2’s output **label normalization** technique further bolsters OU-models’ generalizability.

> 最重要的两个因素，分解成小的 OU，将结果 label normalization，所以能够泛化。

Even though the 10 GB TPC-H workload has tables up to **60m tuples**, which is 60× larger than the largest table considered by MB2’s OU-runners, MB2 is still able to **generalize with minimal loss of accuracy**. MB2 without the output normalization has much worse accuracy on large datasets.

> 再次强调 normalization 的重要性

### Interference Model Accuracy

We next measure the ability of MB2’s interference model to capture the impact of resource competition on concurrent OUs. We run the concurrent runner with the 1 GB TPC-H benchmark since it contains a diverse set of OUs. The concurrent runner enumerates three parameters: (1) **subsets of TPC-H queries**, (2) **number of concurrent threads**, and (3) **query arrival rate**.

Since the interference model is not specific to any OU or DBMS configuration, the concurrent runner does not need to exercise all OU-model inputs or knobs. For example, with the knob that controls the number of threads, we only generate training data for odd-numbered settings (i.e., 1, 3, 5, . . . 19) and then test on even-numbered settings. The concurrent runner executes each query setup for 20s. To reduce the evaluation time, we assume the average query arrival rate per query template per 10s is given to the model. In practice, this interval can be larger. Neural network performs the best for this model given its capacity to consume the summary statistics of OU-model output labels

To evaluate the model’s generalizability, the concurrent runner executes queries only in the **DBMS’s interpretive mode** (execution knob discussed in Sec. 4.2), but we test the model under **JIT compilation mode**. We also evaluate the model with thread numbers and workload sizes that are different from those used by MB2’s concurrent runners. To isolate the interference model’s estimation, we execute the queries in both single-thread and concurrent environments and compare the true adjustment factors (the concurrent interference impact) against the predicted adjustment factors

> 什么是 interpretive mode？

the **actual and interference model-estimated average query run times under concurrent environments**. The interference model has less than 20% error in all cases. It generalizes to these environments because (1) it leverages **summary statistics** of the OU-model **output labels** that are agnostic to specific OUs and (2) the **elapsed time-based input normalization** and ratio-based output labels help the model generalize across various scenarios. We also observe that generalizing the interference model to small data sizes result in the highest error (shown under TPC-H 0.1 GB in Fig. 8b) since the queries are shorter with potentially higher variance in the interference, especially since the model only has the average query arrival information in an interval as an input feature

> 还是 normalization

### Model Adaptation and Robustness

噪声影响，noisy estimation

### Hardware Context

Since MB2 **generates models offline**, their predictions may be inaccurate when the hardware used for training data generation and production differs.

> 在不同的 CPU 频率上训练，带来的误差也不同，而且看上去受影响很大？但为什么是跟主频有关？是因为训练的时候，错误估计了查询时间吗？所以可以扩展多一些参数吗？

### End-to-End Self-Driving Execution

demonstrate MB2’s **behavior models’** application for a **self-driving DBMS’s planning components** and show how it enables interpretation of its **decision-making process**. We assume that the DBMS has (1) workload forecasting information of average query arrival rate per query template in each 10s forecasting interval. and (2) an “oracle” planner that makes decisions using predictions from behavior models. We assume a perfect workload forecast to isolate MB2’s prediction error

> 创建索引时，runtime，cpu 占用率都预测得很好，是 self-driving database 的基础

This example demonstrates that MB2’s **behavior models accurately estimate the cost, impact, and benefit of actions for self-driving DBMSs** ahead of time given the workload forecasting, which is a foundational step towards building a self-driving DBMS

demonstrate MB2’s **estimations** under an alternative action plan of the self-driving DBMS in Fig. 11c. The DBMS plans the same actions under the same setup as in Fig. 11a except to build the index earlier (at 58s) with four threads to reduce the impact on the running workload. MB2 **accurately estimates the smaller query runtime** increment along with the workload being impacted for longer. Such a **trade-off** between action time and workload impact is essential information for a self-driving DBMS to plan for target objectives (e.g., SLAs). We also observe that MB2 underestimates the index build time under this scenario by 27% due to a combination of OU and interference-model errors.

> 但也存在预测错误的情况

## RELATED WORK

We broadly classify the previous work in DBMS modeling into the following categories:

1. ML models
2. analytical models.

Related to this are other methods **for handling concurrent workloads**. We also discuss other efforts on **building automated DBMSs and reinforcement learning-based approaches**

**Machine Learning Models**: Most ML-based behavior models use **query plan information** as input features for estimating system performance.

> ML 方法需要开发人员调整 feature, 收集数据，重新训练以适应新环境。并且很多 ML 模型都是在合成的小 benchmark 上进行的，所以在其他 workload 上有很大的预测错误。并且不支持事务 workloads，并且忽略了 DBMS 内部操作。而 MB2 就拆分了小的 OU 操作，并且用 SQL 所以只用少量重新训练，而且适应更新。

**Analytical Models**: Researchers have also built analytical models for DBMS components.

**Concurrent Queries**: Due to the difficulty of modeling concurrent operations as described in Sec. 2.2, the aforementioned techniques largely focus on performance prediction for one query at a time. For concurrent OLAP workloads, researchers have built sampling- based statistical models that are fixed for a known set of queries.

**Autonomous DBMSs**: There is a long history of research on automating DBMS deployments

> 这里也提到了同样是 CMU 的 Ottertune，但 Ottertune 是 Gaussian Process Regression to model how a DBMS will perform with different knob settings.

**Reinforcement Learning (RL) for DBMSs**: Recent work ex- plores using RL [55, 59] to enhance individual DBMS components [39, 48, 75], which **are independent of MB2’s goal to build behavior models for self-driving DBMSs that automate all the administrative tasks**. We use a modularized design (Sec. 2) for self-driving DBMSs, which has a different methodology than RL-based approaches. We think this approach provides better data efficiency, debuggability, and explainability, which are essential for the system’s practical application. All major organizations working on self-driving cars also use a modularized approach instead of end-to-end RL

> 没理解为什么用模块化方法而不是端到端 RL

## CONCLUSION

**Behavior modeling** is the foundation for building a self-driving DBMS. We propose a modeling framework ModelBot2 (MB2) that decomposes the DBMS into operating units (OUs) and builds models for each OU with runners that exercise OUs’ behavior. Our evalua- tion shows that MB2’s behavior models provide the essential cost, impact, and benefit estimations for actions of self-driving DBMSs and generalize to various workloads and environments.

> 主要讲了怎么建模 behavior model，怎么拆分 Operation Unit 然后用 ML 方法做预测，还有很重要的 normalization。结合起来就是 self-driving DBMS 的基础。
>
> 中间很多地方没怎么看懂，好像就是在特殊的数据库上拆出各种单元任务，用各种不同的 ML 方法预测，然后 normalize 结果。但 normalize 部分不太清楚，怎么选 ML 方法看上去也玄学。OU 选的也挺少的，而且无法处理 disk-oriented DBMS，假设还是挺强的。
>
> 让数据库自动驾驶还是非常吸引人的，尤其是优化。但数据库还是太复杂了，一个框架无法处理所有的情况，而且数据非常难收集，不同数据不同方法也差异很大，论文很有意思的一点就是用汽车的自动驾驶类比数据库的自动驾驶，是很相似的，都需要感知，知道有什么工作负载，也需要预测未来的工作负载，使其能够知道怎么优化。第二步就是优化，比如建索引、调参数，第三步是需要知道在什么时候优化，比如负载低的时候建立索引，还要知道操作带来的影响。
>
> 再一个比较重要的地方是，MB2 考虑了多线程并发的情况，这很重要，尤其是现在的数据库大部分都是支持并发的。
>
> MB2 实际上是结合了机器学习建模和分析方法建模，但大部分情况下 gradient boosting 模型表现最好，但是是根据怎么指标来确定的呢？
>
> 数据对自治数据库的实现也是非常重要的

> Summarize:
> The paper proposes a modeling framework called ModelBot2(MB2) for building a self-driving DBMS. MB2 decomposes DBMS into many fine-grained and independent operating units(OUs) such as building a hash table and interference model for concurrent operations such as log record flush. MB2 trains these models with different ML methods like gradient boosting, and use different input features to estimate system performance metrics. The experiment result shows that MB2 is able to predict DBMS performance more accurately than the state-of-the-art ML models called QPPNet, especially under dynamic different workload scenarios. MB2 performs behavioral modeling on DBMS and predicts the impact and benefit of automated operations of various datasets(OLAP and OLTP), which is foundational for establishing an self-driving database.

> Contributions:
>
> 1. MB2 breaks down the database management system's functionality into smaller, independent units called operating units (OUs), each OU can have a model trained to predict its runtime and resource consumption. Using decomposed OUs to build behavioral models not only reduces the high-dimensionality of the problem, but each OU requires less training data and gives more fine-grained predictions.
> 2. MB2 also proposes a interference model, which captures contention on locks and data structures through individual OU models. Additionally, a separate interference model is designed to predict contention on hardware resources such as memory, CPU, and I/O. This model outputs summary statistics to generalize to various concurrent OUs.
> 3. MB2 integrates the behavior model and interference model with NoisePage, which is a relational database management system from CMU. MB2 can predict the system’s performance for OLTP, OLAP and even mixed workloads in dynamic environments, while most ML-based models do not support transcational workloads. MB2 achieves this by applying output normalization techniques, and it outperforms QPPNet particularly in terms of generalizability and performance across different workloads.

> Limitations:
>
> 1. MB2 is limited to train on an offline enviornment. Though MB2 is adaptive to dynamic workload and even transcational workload, it still needs retrain when there are some DBMS updates. I'm not an expert in database tuning, but maybe some OUs in MB2 still need to be retrained when knobs and configurations are changed.
> 2. MB2 is not be directly designed for predicting disk-oriented database performance, and it uses in-memory database. Since MB2 might not be able to predict the cache content in CPU cache runs or buffer pool. And MB2 needs warm up and enough repetitions to be able to predict accurately. This may cause the CPU cache and buffer pool to cache a lot of data, resulting in inaccurate predictions.
> 3. MB2 is designed to be a self-driving DBMS, however, the initial setup of MB2 needs developers to decompose the DBMS’s architecture into operating units(OUs), which requires professional database administrators. And MB2 seems to have poor cross-platform capabilities, when the CPU frequency changes, the prediction ability changes greatly.

> Improve:
>
> The paper makes an analogy between self-driving vehicles and databases. It is a very good paper. I might try to improve it based on several limitations of MB2. First, I would try to extend MB2's capabilities to disk-oriented DBMS, it would be necessary to incorporate models that understand and predict disk I/O operations and the cached content. It might need to develop hybrid models that can handle both in-memory and disk-based operations, apply more ML methods to predict cached contents. What's more, MB2 could be improved by implementing online learning algorithms that can update the models in real-time for database updates or new workloads. Finally, I will try to enhance MB2 to adapt to different CPU frequencies, it might involve creating models that factor in CPU performance metrics and how they affect DBMS operations.
