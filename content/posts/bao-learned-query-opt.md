+++
title = 'Paper Reading: Bao: Making Learned Query Optimization Practical [SIGMOD 21]'
date = 2024-03-17T00:14:24-04:00
draft = false
tags = ['Paper Reading']
+++

## Bao: Making Learned Query Optimization Practical

MLDB + query optimization

## ABSTRACT

最近 ML 做 query optimization 由于需要 substantive training overhead 所以其实很少 practical gains, inability to adapt to changes, poor tail performance.

论文提出了 Bao, Bandit Optimizer, 通过利用现有查询优化器的知识，对每个查询提供优化建议。

Bao combines mordern tree Convolutional Neural Networks (CNN) with Thompson sampling (RL). Bao 可以从错误中学习，适应不同的 query workloads, data, schema.

实验结果显示 BAO 可以快速学习改善**端到端**查询执行性能（包括 tail latency）的策略，用于几种包含 long-term 查询的工作负载。在**云环境**中，我们表明，与商业系统相比，BAO 可以提供降低的成本和更好的性能。

> 长尾请求：P99, 99% 请求在一定耗时内，长尾请求就是明显高于均值的那部分比较小的请求

## INTRODUCTION

Query Optimization, cardinality estimation and cost modeling, difficult to crack

很多 Query Optimization 都是 ML 做的，但却是很少实用：

1. Long training time, impractical amount of training data, ML-powered cardinality estimators based on supervised learning 需要收集精确的 cardinalities，非常昂贵的操作。所以 Bao wish to estimate cardinalities in the first place. RL 也需要好几天的训练。
2. Inability to adjust to data and workload changes. query workload, data, schema 是会变的，cardinality estimators based on supervised learning 就需要重新训练，不然就过时了。
3. Tail catastrophe. learning techniques 比 traditional optimizers 平均表现更好，但是长尾表现很差 （100x regression in query performance）。尤其是训练数据不足时，同时统计和现实还是具有差距.
4. Black-box decisions. DL 方法是黑盒? 与传统优化器不同, 当前学到的优化方法不能给 DBA 提供方法
5. Integration cost. 大部分 learned optimizers 都是研究原型, 很少结合 DBMS.

Bao 克服了上面的问题，可以集成到 PostgreSQL，https://github.com/learnedsystems/baoforpostgresql

核心思想：避免 learning an optimizer from scratch。 take an existing optimizer (PostgreSQL’s
optimizer), and learn when to activate (or deactivate) some of its features on a query-by-query basis. In other words, Bao is a learned component that **sits on top of an existing query optimizer** in order to enhance query optimization, rather than replacing or discarding the traditional query optimizer altogether

> 在 PostgreSQL 优化器之上开始做优化，那如果之前的优化器不再更新，又或者更好的优化器出现了怎么办呢？

PostgreSQL optimizer might **underestimate the cardinality** for some joins and wrongly select a loop join when other join algorithms (e.g., merge join, hash join) would be more effective
This occurs in query 16b of the **Join Order Benchmark** (JOB)， and disabling loop-joins for this query yields a 3𝑥 performance improvement (see Figure 1). Yet, it would be wrong to always disable loop joins. For example, for query 24b, disabling loop joins causes the performance to degrade by almost 50𝑥, an arguably catastrophic regression.

![](https://s2.loli.net/2024/03/17/LGqXZjPCFvBbWOp.png)

Bao assumes a **finite set of hint sets** and treats each hint set as an arm in a contextual multi-armed bandit problem.

> multi-armed bandit problem 多臂赌博机问题，机器学习问题，每个老虎机赢的概率不一样，不知道概率，选择哪个能做到最大收益？

Bao learns a model that predicts **which hints will lead to good performance** for a **particular query**. When a query arrives, our system **selects a hint set**, executes the resulting query plan, and observes a reward. Over time, Bao refines its model to **more accurately** predict which hint set will most benefit an incoming query. For example, for a highly selective query, Bao can automatically steer an optimizer towards a left-deep loop join plan (by restricting the optimizer from using hash or merge joins), and to disable loop joins for less selective queries

By formulating the problem as a **contextual multi-armed bandit**, Bao can take advantage of Thompson sampling, a **well-studied sample efficient algorithm**

Because Bao uses an underlying query optimizer, Bao has **cardinality estimates** available, allowing Bao to **adapt to new data and schema changes** just as well as the underlying optimizer. 其他 learned query optimization 需要重新学习 传统优化器 已知的信息，Bao 可以直接开始学习怎么去调优底层优化器，并且相较于传统优化器可以减少 tail latency。

1. Short training time, 1 hour, by taking full advantage of existing query optimization knowledge, which was encoded by human experts into traditional optimizers available in DBMSes today
2. Robustness to schema, data, and workload changes, maintain performance even in the presence of workload, data, and schema changes, leveraging a **traditional query optimizer’s cost** and **cardinality estimates**
3. Better tail latency, Bao is capable of **improving tail performance** by orders of magnitude with as little as 30 minutes to a few hours of training
4. Interpretability and easier debugging, be inspected using standard tools,
5. Low integration cost, every SQL
6. Extensibility, adding new query hints over time, w/o retraining. Additionally, Bao’s feature representation can be **easily augmented** with **additional information** which can be taken into account during optimization, although this does require **retraining**. cache state

drawback:

query optimization time increases, Bao must run the traditional query optimizer several times for each incoming query.
for very short running queries,

- Bao is ideally suited to workloads that are **tail-dominated** (80% of query processing time is spent processing 20% of the queries) or contain many long-running queries,
- limited set of hints, Bao has a **restricted action space**, Bao is not always able to learn the best possible query plan.
- outperform **traditional optimizers** while training and adjusting to change orders-of-magnitudes faster than “unrestricted” learned query optimizers, like Neo

> Neo 也是强化学习 + DNN，也是用 PG 优化器通过训练然后做一些贪心搜索得到一些计划。可能是 query-level 的编码或者 plan-level 的编码不同，比如谓词的表示借鉴了 word2vec 捕捉 rows 之间相关性，也用了 tree DNN 来训练，value model 人为设计 cost 预测 latency，但需要不断迭代，对 ad-hoc 没有太大帮助。

summary:

1. We introduce Bao, a **learned system** for **query optimization** that is capable of **learning how to apply query hints on a case-by-case basis**
2. For the first time, we demonstrate a learned query optimization system that outperforms both open source and commercial systems in **cost and latency**, all while **adapting to changes in workload, data, and schema**.

## SYSTEM MODEL

Bao combines a **tree convolution model**, a **neural network operator** that can recognize important patterns in query plan trees, with Thompson sampling (solving contextual multi-armed bandit problems)

1. **Generating 𝑛 query plans**: When a user submits a query, Bao uses the underlying query optimizer to produce 𝑛 query plans, one for each set of hint. While some hints can be applied to a single relation or predicate, Bao focuses **only on query hints that are a boolean flag** (e.g., disable loop join, force index usage). The sets of hints available to Bao **must be specified upfront**, empty -> original optimizer
2. **Estimating the run-time** for each query plan: Afterwards, each query plan is transformed into a **vector tree** (a tree where each node is a feature vector) -> value model(tree convolutional neural network) -> which **predicts** the quality (e.g., execution time) of each plan -> parallel
3. Selecting a query plan for execution: best expected performance -> standard supervised fashion and pick the query plan with the best predicted performance. However, **value model** might be wrong, **might not always pick the optimal plan** and never try **alternative strategies** never learn when we are wrong. Thompson sampling explore a specific query offline and guarantee that only the best plan is selected. Once the query execution is complete, the **combination of the selected query plan** and the **observed performance** is added to Bao’s experience. Periodically, this experience is used to retrain the predictive model, creating a feedback loop
4. Assumptions and Limitations: assumes **hints** result in semantically equivalent **query plans**, Bao always uses the hints for the **entire query plan**. Bao cannot restrict features for **only a part of a query plan**. RL converge -> small size of action

> Bao 的 vaule model 永远不会从错误中学习？而且看上去不支持 subquery 优化，不知道是不是时间复杂度太高的原因。

## SELECTING QUERY HINTS

Bao’s learning approach, optimization goal, formalize it as a contextual **multi-armed bandit problem.**, apply Thompson sampling, a classical technique used to solve such problems

Bao models each hint set in the family of hint sets

Bao also assumes a user-defined performance metric 𝑃

Contextual multi-armed bandits (CMABs):

Thompson sampling:

> RL 的一些东西，不看了

### Predictive model

The core of Thompson sampling, Bao’s algorithm for selecting hint sets on a per-query basis, is a predictive model that estimates the performance of a particular query plan.

> Bao 是怎么转换 query plan trees: **binarizing** the query plan tree and encoding each query plan operator as a vector, optionally augmenting this representation with cache information

**Binarization**: Many queries involve non-binary operations like aggregation or sorting. However, **strictly binary query plan trees** (i.e., all nodes have either zero or two children) are convenient because they greatly **simplify tree convolution** (explained in the next section).

**Vectorization**: Each node in a query plan tree is transformed into a vector containing: (1) a one-hot encoding of the operator, (2) cardinality and cost information, and optionally (3) cache information

> Tree convolutional neural networks

**Integrating with Thompson sampling**: ?

### Training loop

> 提出了新的 optimization, 在云平台上可以热加载 GPU？

## POSTGRESQL INTEGRATION

install and use with PostgreSQL, hook system

**Per-query activation**: sits on top of a traditional optimizer, When Bao is activated, Thompson sampling is used to select query hints. When Bao is deactivated, the PostgreSQL optimizer is used. Note that even when Bao is disabled, Bao can (optionally) still learn from query executions. _off-policy reinforcement learning_ ??
**Active vs. advisor mode**: In active mode, Bao operates as described above, **automatically selecting hint sets and learning from their performance**. In advisor mode, Bao does not select hint sets (all queries are optimized by the PostgreSQL planner), but still observes the performance of executed queries and trains a predictive model.
**Triggered exploration**: query regressions, because Bao actively explores new query plans, regressions may be more **erratic**

## RELATED WORKS

One of the earliest applications of learning to query optimization was Leo [72], which used successive runs of the similar queries to adjust histogram estimators.

Neo [51] showed that **deep reinforcement learning** could be applied **directly to query latency**, and could learn optimization strategies that were competitive with commercial systems after 24 hours of training. However, none of these techniques are **capable of handling changes in schema, data, or queryworkload**, and none **demonstrate** improvement in **tail performance**. Works applying reinforcement learning to adaptive query processing have shown interesting results, but are not applicableto existing, non-adaptive systems like **PostgreSQL**.

> 我不认为 Bao 和 Neo 有太大的差别，同样都是 RL，为什么说 Neo 无法适应变化呢？

## EXPERIMENTS

real-world database **workloads**, quantifying not only query performance, but also on the **dollar-cost** of executing a workload

### Setup

IMDb dataset, new real-world datasets and workload called Stack, Corp dataset

> 实验结果只是和 PG, ComSys 比了一下 latency，效果还不错，大概 20% 的改善， P99 也改善很多，但为什么这部分不和 Neo 对比呢？

**Query optimization time**: The maximum optimization time required by PostgreSQL was 140ms, for the commercial system 165ms, and for **Bao 230ms**. For some applications, a 70ms increase in optimization time could be acceptable. Moreover, our current prototype is simple (e.g., our inference code is in Python), and thus a lot of optimization potential exists

> 230ms 比起 140ms 虽然不多，但也不算少吧，而且用了大量并行

**Prior learned optimizers**: Neo [51] and DQ [40] are two other learning based approach to query optimization. Like Bao, Neo uses **tree convolution**, but unlike Bao, Neo **does not select hint sets for specific queries**, but instead fully builds query execution plans on its own. DQ uses deep Q learning [56] with a hand-crafted featurization and a fully-connected neural network (FCNN). We compare performance for the IMDb workload in Figure 14 (average of 20 repetitions on an N1-16 machine with a cutoff of 72 hours).

For Figure 14a, we uniformly at random select a query to create a stable workload, and for Figure 14b we use our original dynamic workload. **With a stable workload, Neo is able to overtake PostgreSQL after 24 hours**, and **Bao after 65 hours**. This is because Neo has many more degrees of freedom than Bao: Neo can use any logically correct query plan for any query, **whereas Bao is limited to a small number of options**. These degrees of freedom come at a cost, as Neo takes significantly longer to converge. After 200 hours of training, **Neo’s query performance was 15% higher than Bao’s**. DQ, with a similar degrees of freedom as Neo, takes longer to outperform PostgreSQL, possibly due to FCNNs having a poor inductive bias [50] for query optimization [51]. With the **dynamic workload** (Figure 14b), Neo and DQ’s convergence is significantly hampered, as both techniques struggle to learn a policy robust to the changing workload. With a dynamic workload, neither DQ nor Neo is able to overtake Bao within 72 hours.

> 65h > 24h 为什么 Bao 会被 a small number of options 限制呢，文章也没有仔细解释 Bao 是怎么适应 dynamic workload 的

![](https://s2.loli.net/2024/03/18/dqC2A8ljWtbuI9L.png)

## CONCLUSION AND FUTURE WORK

> 实际上 Bao 团队和 Neo 团队都是一批人，应该是注重点不太一样：
>
> Neo 是第一个提出深度学习 query optimizer 的，用一个未补全的 query plan encoding 之后用树形网络做预测 cost，并用最小堆搜索最优 plan，忽略 cardinality 等等传统，为每个 predicate 用浮点数存 feature 设计 word vector 存语义，可能好于 histgram 预测。但缺点是训练时间长，只在平均上提升比较好，高质量的计划很可能在随机样本中缺失，样本量必须大，模型训练成本太高。

> 而 Bao 放弃做 Optimizer，为成熟数据库提供 query hints 如 boolean flag 表示是否使用 hash join, merge join, index scan 等等，枚举所有 hint set (access method + join method) 视作多臂老虎机，用 Thompson sampling 平衡选一个最优的。论文提到了 Bao 收敛快于 Neo，快很多，而且可以根据 workload 调整，在 JOB 数据集上表现非常好，但还是可能会去掉一些最优的，选择次优的。
>
> 去年也有一篇 Lero，采用成对方法训练分类器，也不是从头开始，很类似。尝试解决 Bao 的一些问题: 1. 在整个计划搜索过程中，通常对**整个查询**应用提示集。如果查询的不同部分（子查询）具有不同的最佳选择，则在查询级别调整标志可能会错过寻找高质量计划的机会 2. 可用的提示集是**特定于系统**的。优化器通常包含数百个标志来实现/禁用某些优化规则。在实践中**枚举各种组合是不可行**的。手动选择提示集的有效子集需要对系统进行深入的理解，并对工作负载进行综合分析。
>
> Lero 甚至实现了一个 Bao+, 用更多计划进行模型训练，但好像没有仔细说。同时拓展了更多的数据集， IMDB，JOB，STATS，TPC-H，TPC-DS，等等

## Paer Reading

> The paper proposes a more practical model called Bao, which sits on top of an existing query optimizer (PostgreSQL) and utilizes machine learning and reinforcement learning for database query optimization. Bao solves lots of the issues with previous learned optimizers, including long training times, inability to adapt, and being difficult to interpret while offers advantages like faster training, robustness to workload and data changes, and better interpretability. The core idea of Bao is to utilize tree convolutional neural networks to analyze query plans and leverage ML techniques called Multi-armed bandit and Thompson Sampling to choose the best query plan among alternatives.

> 1. Bao addresses a major limitation of previous learned optimizers – long training times. By employing Thompson Sampling to solve bandit problem, Bao significantly reduces training time compared to traditional machine learning methods used for query optimization, and achieves similar performance to PostgreSQL in 2 hours training.
> 2. Bao is more robust to changes in schema, data distribution, and query workload. By combining modern tree convolutional neural networks and Thompson sampling to provide query hints, Bao offers interpretability unlike other black-box machine learning models and aims to reduce tail latency and minimize the occurrence of slow-running queries.
> 3. The paper also proposes a new method to optimize resource usage in cloud environments, specifically for tasks involving training a neural network with GPU and then using it for query optimization with CPU, which can reduce overall costs.

> 1. Bao uses Thompson Sampling to enumerate all hint sets in the multi-armed bandit problem, but during the searching process, the hint set is applied to the entire query. If different parts of the query (subqueries) have different best choices, it will lead to the final choice is not optimal.
> 2. Enumerating all collections is very time-consuming, the query optimization time of Bao is nearly 90ms higher than PostgreSQL. And for different systems, the size of the hint sets may be different, somtimes it might include hundreds of boolean flags.
> 3. The architectures of Bao and Neo look very similar, though the authors of these papers are from the same group, the paper does not carefully compare Neo and Bao, especially the performance of training time and query latency.

> First, I would explain the difference between Neo and Bao in more depth, for example, why they all use modern tree convolutional neural networks. Besides, I will collect more data sets and testing their performance on more data benchmarks (such as TPC-H, TPC-DS, STATS). If possible, I would try some methods to limit the hint sets to reduce the current query optimization time, such as manual selection or from expert experiences.
