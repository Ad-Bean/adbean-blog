+++
title = 'Paper Reading: An Inquiry into Machine Learning-based Automatic Configuration Tuning Services on Real-World Database Management Systems'
date = 2024-04-07T14:22:23-04:00
draft = false
tags = ['Paper Reading']
+++

## An Inquiry into Machine Learning-based Automatic Configuration Tuning Services on Real-World Database Management Systems

CMU 的对于 DBMS 自动调优的论文，采用了 ML 机器学习方法，是 Ottertune 的论文。

## ABSTRACT

Modern database management systems (DBMS) expose dozens of **configurable knobs** that control their runtime behavior

与专家 DBA 相比，使用机器学习（ML）进行自动调整方法，的最新工作已显示出更好的性能。然而，这些基于 ML 的方法是在具有**有限调优机会**的**合成**工作负载上进行评估的，因此尚不清楚它们是否在**生产环境**中提供相同的好处

To better understand ML-based tuning, we conducted a thorough evaluation of ML-based DBMS knob tuning methods on an enterprise database application. We use the OtterTune tuning service to compare three state-of-the-art ML algorithms on an Oracle installation with a real workload trace. Our results with OtterTune show that these algorithms generate knob configurations that improve performance by 45% over enterprise-grade configurations. We also identify deployment and measurement issues that were overlooked by previous research in automated DBMS tuning services

> 在企业数据库上实现了 OtterTune，提高了 45% 的性能，并且发现了部署和检测问题。

## INTRODUCTION

automate the tuning of database management systems (DBMSs)

1. “self-adaptive” DBMSs that used recommendation tools with physical database design (e.g., indexes, partitioning).
2. In the early 2000s, “self-tuning” systems expanded the scope of the problem to include automatic knob configuration. These **knobs** are tunable options that control nearly all aspects of the DBMS’s **runtime** behavior.

DBMSs have hundreds of knobs, and a database administrator (DBA) cannot reason about how to tune all of them for each application.

Unlike physical database design tools [11], **knob configuration tools** cannot use the **built-in cost models** of the DBMS’s query optimizers. There are two ways that these tools automatically tune knobs. The first is to use **heuristics** (i.e., static rules) that the tool developers manually create [1, 5, 14, 27, 42]. Previous evaluations with open-source DBMSs (Postgres, MySQL) showed that these tools improve their throughput by 2–5×over their default configuration for OLTP workloads [38]. These are welcomed improvements, but there are additional optimizations that the tools failed to achieve. This is partly because they only target the 10–15 knobs that are thought to have the most impact. It is also because the **rules do not capture the nuances of each workload that are difficult to codify**

> 为什么无法用内置的 cost model 调整 knobs？启发式规则没法捕捉到 workload 之间细微的差别？

tune a DBMS s **knobs** is to use machine learning (ML) methods that devise strategies to configure knobs without using hardcoded rules

> 在 OLTP 任务中 ML 方法比静态工具吞吐量提高 15 35%，原因是 ML 比静态规则考虑了更多 knobs，而且还会考虑 knobs 之间的依赖关系，是非线性的，人类难以模拟

ML 方法比人类 DBA 更好，性能好。(1) opensource DBMS with limited tuning potential (e.g., Postgres, MySQL,
MongoDB) and (2) synthetic benchmarks with uniform workload patterns

而且用了本地存储（SSD），显示 DBMS 都用的非本地共享存储，比如 on-premise SANs and cloud-based block stores (S3)? 后者有更高的读写延迟， incur more variance in their performance than local storage。

> 目前并不清楚这些差异怎么影响 ML 调优方法？为什么呢，如果云存储导致读写延迟高，那 ML 调优 建模的时候应该也能考虑到这些情况？

之前的研究也没有提到自动化调优，they do not specify how they select the **bounds of the knobs** they are tuning. This means that the quality of the configurations may still depend on a human initializing it with the right parameters

> 还是需要 DBA 初始化

this paper presents a field study of **automatic knob configuration** tuning on a commercial DBMS with a real-world workload in a **production environment**. We provide an evaluation of state-of-the-art ML-based methods for tuning an enterprise Oracle DBMS (v12) instance running on virtualized computing infrastructure **with non-local storage**. We extended the OtterTune [3] tuning service to support three ML tuning algorithms: (1) Gaussian Process Regression (GPR) from OtterTune [38], (2) Deep Neural Networks (DNN) [4, 39], and (3) Deep Deterministic Policy Gradient (DDPG) from CDBTune [40]. We also present optimizations for OtterTune and its ML algorithms that were needed to support this study.

> 本文应该是集大成的，自动调优、商业数据库、生产环境、stoa benchmark、云存储。OtterTune 使用了 GDR 、DNN 和 DDPG 方法。

the quality of the configurations from the ML algorithms (GPR, DDPG, DNN) are **about the same** on **higher knob counts**, but vary in their convergence times.

> knob 多的时候，效果差不多，但收敛速度不一样

## BACKGROUND

how an automated tuning service works using **OtterTune** as an example

limitations of previous evaluations

why a more robust assessment is needed to understand their capabilities

### OtterTune Overview

介绍 OtterTune，It maintains a repository of data collected from previous tuning sessions and uses it to build models of how the DBMS responds to different knob configurations. It uses these models to guide experimentation and recommend new settings. Each recommendation provides the service with more data in a feedback loop for refining and improving the accuracy of its models

> feedback loop 有点像 RL

OtterTune is made up of a **controller** and a **tuning manager**. The controller acts as an intermediary between the target DBMS and the tuning manager. It collects runtime data from the target DBMS and installs configurations recommended by the tuning manager. The tuning manager updates its repository and internal ML models with the information provided by the controller and then recommends a new configuration for the user to try

At the start of a tuning session, the user specifies which metric should be the target objective for the service to optimize.

> 需要用户指定优化目标

first tuning iteration, 1.executes the target workload on the DBMS. After the workload finishes, the controller 2.collects the runtime metrics and configuration knobs from the DBMS and 3.uploads them to the tuning manager. OtterTune’s tuning manager 4.receives the latest result from the controller and stores it in its repository. Next, the tuning manager 5.uses its ML models to generate the next knob configuration and returns it to the controller. The controller 6.applies the knob configuration to the DBMS and starts the next tuning iteration. This loop continues until the user is satisfied with the improvements over the original configuration

![](https://s2.loli.net/2024/04/08/uWeoQbUAY9SltMF.png)

> 控制器 1○ 执行 DBMS 上的目标工作负载。工作负载完成后，控制器 2○ 收集 DBMS 的运行时指标和配置旋钮将其上传到 Tuning Manager。Ottertune 的调整管理器 4○ 收到控制器的最新结果，并将其存储在其存储库中。接下来，Tuning Manager 5○ 使用其 ML 型号生成下一个旋钮配置并将其返回到控制器。控制器 6○ 将旋钮配置应用于 DBMS，并开始下一个调整迭代。该循环继续进行，直到用户对原始配置的改进感到满意

### Motivation

(1) workload, (2) DBMS, and (3) operating environment.

**Workload Complexity**: Gaining access to production workloads to evaluate new research ideas is non-trivial due to privacy constraints and other restrictions. Prior studies evaluate their techniques using synthetic benchmarks; the most complex benchmark used to evaluate ML-based tuning techniques to date is the TPC-C OLTP benchmark from the early 1990s. But previous studies have found that some characteristics of TPC-C are not representative of real-world database applications [23, 25]. Many of the unrealistic aspects of TPC-C are due to its simplistic database schema and query complexity. Another notable difference is the existence of temporary and **large objects** in production databases. Some DBMSs provide knobs for tuning these objects (e.g., Postgres, Oracle), which have not been considered in prior work

> 之前的研究都是 synthetic benchmarks, TPC-C OLTP benchmark 无法代表显示，很多不符合实际。而且许多生产数据库会有超大临时对象，以前的工作没有考虑这种

**System Complexity**: The simplistic nature of workloads like TPC-C means that there are fewer tuning opportunities in some DBMSs, especially for the two most common DBMSs evaluated in previous studies (i.e., MySQL, Postgres). For these two DBMSs, one can achieve a substantial portion of the performance gain from configurations generated by ML-based tuning algorithms by setting two knobs according to the DBMS’s documentation. These two knobs control the amount of RAM for the buffer pool cache and the size of the redo log file on disk

> TPC-C workload 简单，某些数据库调整的机会比较少，就能得到很大的性能调整（buffer pool 和 redo log file size）
>
> Postgres Knobs – SHARED_BUFFERS, MAX_WAL_SIZE
>
> MySQL Knobs – INNODB_BUFFER_POOL_SIZE, INNODB_LOG_FILE_SIZE

文章用 OtterTune 在 DDPG 算法下调节十个 knob，改善了 TPC-C 在 DBMS 的性能，

More importantly, however, the configurations generated by ML algorithms achieve only 5–25% higher **throughput** than the two-knob configuration across the different versions of MySQL and Postgres. That is, one can achieve 75–95% of the performance obtained by ML-generated configurations by tuning only two knobs for the TPC-C benchmark

> 这个实验表现了只用调整两个 knob 就能在 TPC-C workload 上得到很好的吞吐量表现，比标准配置高很多

**Operating Environment**: Disk speed is often the most important factor in a DBMS’s performance. Although the previous studies used virtualized environments to evaluate their methods, to our knowledge, they deploy the DBMS on ephemeral storage that is physically attached to the host machine. But many real-world DBMS deployments use durable, non-local storage for data and logs, such as on-premise SANs and cloud-based block/object stores. The problem with these non-local storage devices is that their performance can vary substantially in a multi-tenant cloud environment

> 强调了非本地存储的情况，对比了本地存储 SSD 和云存储 VM 存储，后者延迟不好预测，前者则稳定。

## AUTOMATED TUNING FIELD STUDY

> 文章的突破点在于，更加现实的自动调优 DBMS，给出了商业数据库的例子，甚至对比专业的 DBA 都有很好的表现。

### Target Database Application

TicketTracker 数据

**Database**: We created a **snapshot** of the TicketTracker database from its production server using the Oracle Recovery Manager tool. The total uncompressed size of the database on disk is ∼1.1 TB, of which 27% is table data, 19% is table indexes, and 54% is large objects (LOBs). This LOB data is notable because Oracle exposes knobs that control how it manages LOBs, and previous work has not explored this aspect of DBMS tuning.

> large objects LOB 数据是以前没有探讨过的，

**Workload**: We collected the TicketTracker workload trace using Oracle’s Real Application Testing (RAT) tool. RAT captures the queries that the application executes on the production DBMS instance starting at the snapshot. It then supports replaying those queries multiple times on a test database with the exact timing, **concurrency**, and transaction characteristics of the original workload [19]. Our trace is from a two-hour period during regular business hours and contains over 3.6m query invocations.

> 大部分查询都是 SELECT 短查询，2% 查询是多表查询，98% 都是单表。大部分查询都走了索引。

There are important differences in the TicketTracker application compared to the TPC-C benchmark used in previous ML tuning evaluations. Foremost is that the TicketTracker database has **hundreds of tables** and the TPC-C database only has nine. TPC-C also has a much **higher write ratio for queries** (46%) than the TicketTracker workload (10%). This finding is consistent with previous work that has compared TPC-C with real-world workloads [23, 25]. Prior to our study, it was unknown whether these differences affect the efficacy of ML-based tuning algorithms.

> 分析了 TPC-C 和该 workload 的区别，虽然这是现实例子，但它具有代表性吗？

### Deployment

DBA 实现根据文档也调整了一些 knobs， As such, for the TicketTracker workload, the DBA further customized some of the knobs in the pre-tuned configuration, including one that improves the performance of LOBs

We set up multiple of **OtterTune’s tuning managersand controllers** in the same data center as the Oracle DBMSs. We ran each component in a Docker container with eight vCPUs and 16 GB RAM. Each DBMS instance has a dedicated OtterTune tuning manager assigned to it. This separation prevents one session from using training data collected in another session, which will affect the convergence rate and efficacy of the algorithms

> Oracle Knob: DB_32K_CACHE_SIZE

### Tuning

At the beginning of each iteration in a tuning session, the controller first **restarts** its target DBMS instance. Restarting ensures that the knob changes that OtterTune made to the DBMS in the last iteration take effect. Some knobs in Oracle do not require restarting the DBMS, but changing them is not instantaneous and requires additional monitoring to determine when their updated values have been fully applied. To avoid issues with incomplete or inconsistent configurations, we restart the DBMS each time

> 需要重启，以更改 knob，这是不是一个限制？也有很多 knob 不需要重启 DBMS，但修改不是立刻见效的，还需要额外检测，所以还是需要重启。

Another issue is that Oracle could **refuse to start** if one of its knobs has an invalid setting.

> 启动失败的时候，没有错误日志，调优迭代就会停止然后报告给 tuning manager，找到下一个配置再迭代。但重启之后的恢复 workload trace 也很快？五分钟，利用了 snapshot 仅重置修改过的 pages 尽管 workload 有 1T

reset 之后会跑一个 microbenchmark 评测性能，但不是必须的。This step is not necessary for tuning, and none of the algorithms use this data in their models. Instead, we use these metrics to explain the DBMSs’ performance in noisy cloud environments (see Section 6).

> 没太理解这里，benchmark 的结果是什么，为什么对于 tuning 不是必须的？

OtterTune’s controller executes TicketTracker’s workload trace using Oracle RAT. We limit the replay time for two reasons. First, the segment’s timespan is based on the wall clock of when the trace was collected on the production DBMS. This means that when the trace executes on a DBMS with a **sub-optimal** configuration (which is often the case at the beginning of a tuning session), the **10-minute segment could take several hours to complete**. We halt replays that run longer than 45 minutes. The second reason is specific to Oracle: RAT is unstable on large traces for our DBMS version. Oracle’s engineers did provide SG with a fix, but only several months after we started our study, and therefore it was too late to restart our experiments

> replay time 是什么，为什么限制 replay time，一个是 10min segtime 可能需要几个小时完成，一个是 Oracle RAT 不稳定，尽管提供了解决方案但太晚了（可能也是另一个 limitation）

## TUNING ALGORITHMS

(1) Gaussian Process Regression (GPR),

(2) Deep Neural Network (DNN), and

(3) Deep Deterministic Policy Gradient (DDPG).

Although there are other algorithms that use query data to guide the search process [28], they are not usable at SG because of privacy concerns since the queries contain user-identifiable data. Methods to anonymize this data are outside the scope of this paper

> 为什么其他方法会设计隐私？那为什么 DDPG 不会？

### 4.1 GPR — OtterTune (2017)

Data Pre-Processing: Each invocation takes up to an hour, depending on the number of samples and DBMS metrics

具体算法略

### 4.2 DNN — OtterTune (2019)

Previous research has argued that **Gaussian process** models do not perform well on **larger data sets** and **high-dimensional feature vectors**

### 4.3 DDPG — CDBTune (2019)

RL 方法，OtterTune 实现了 DDPG++

We identified a few optimizations to CDBTune’s DDPG algorithm that reduce the amount of training data needed to learn the representation of the Q-value. We call this enhanced version DDPG++.

## EVALUATION

使用 Random sampling 方法评估

### Performance Variability

Because each tuning session in our experiments **takes multiple days to complete**, we deployed the Oracle DBMS on multiple VMs to run the sessions in parallel. Our VMs run on the same physical machines during this time, but the other tenants on these machines or in the same rack may change. As discussed in Section 2.2, running a DBMS in virtualized environments with shared storage can lead to unexplained changes in the system’s performance across instances with the same hardware allocations and even on the same instance

> 每次 tuning session 都需要好几天？这是为什么？甚至花了六个月来检测，每周测一次。甚至同一个 VM 上 DBMS 也有性能波动？

We believe that these **inconsistent results are due to latency spikes in the shared-disk storage**. Figure 9 shows the DBMS’s performance for one VM during a tuning session, along with its CPU busy time and I/O latency. These results show a correlation between spikes in the I/O latency (three highlighted regions) and degradation in the DBMS’s performance. In this example, the algorithm had converged at this point of the tuning session, so the

> 出现了一些不一致的结果，论文认为是共享存储的一些延迟尖峰引起的，因为他们测了 VM 的性能，CPU 占用率和 IO 延迟，认为算法已经收敛，所以应该是外部因素有关，而不是 DBMS 内部问题。
>
> 为什么不是某次的 knob 调整出问题了？这里没有提到 knob 是否更改，虽然 IO 的问题应该更大一些。Performance Variability 这部分没有很好地解释这个变化到底是为什么，也不太直白。

### Tuning Knobs Selected by DBA

让 DBA 选 40 个 knobs 来调优，

Figure 11 shows that the configurations recommended by DNN and DDPG++ that tune 10 knobs improve the DB Time by 45% and 43% over the default settings, respectively. Although LHS, GPR, and DDPG achieve over 35% better DB Time, they do not perform as well as DNN and DDPG++ because they select a sub-optimal version of the optimizer features to enable

> DNN 和 DDPG++ 改了 10 个 knobs 提高了 45% 左右的性能，改 20 个 knobs 时提高 33-40% 性能但存在至少一个 knob 错误，论文认为是复杂度太高，算法的随机性以及 VM 的噪声引起的。但没有解释哪个原因才是根本的。

We also see that DNN has the largest gap between its minimum and maximum optimized configurations.

the configurations from DNN and GPR achieve **40% better DB Time** than the default configuration. DDPG and DDPG++ only achieve 18% and 32% improvement, respectively. The reason is that neither of them can fully optimize the **40 knobs within 150 iterations**. DDPG++ outperforms DDPG because of the optimizations that help it converge more quickly (see Section 4.3). With more iterations, DDPG would likely achieve similar performance to the other ML-based algorithms. But due to computing costs and labor time, it was not practical to run a session for more than 150 iterations in our evaluation. The LHS configuration performs the worst of all, achieving only 10% improvement over the

> 150 迭代都没法优化 40 个 knobs，为什么？DDPG++ 收敛更快一些，

In summary, we find that the configurations generated by all of the algorithms that tune 10, 20, and 40 knobs can improve the DBMS’s performance over the default configuration. GPR always converges quickly, even when optimizing 40 knobs. But GPR is prone to getting stuck in local minima, and once it converges, it stops exploring and thus does not continue to improve after that point. The performance of GPR, therefore, depends on whether it explores the best-observed ranges of the impactful knobs from Table 2. We also observe that its performance is influenced by the initial samples executed at the start of the tuning session. This is consistent with findings from previous studies [26]. In contrast, DNN, DDPG, and DDPG++ require more data to converge and carry out more exploration. The configurations that tune 10 knobs perform the best overall. This is because the lower complexity of the configuration space enables DNN and DDPG++ to find good settings for the impactful knobs in Table 2.

> GPR 可以快速收敛，但容易陷入 local minima，所以选取初始 knob 就很重要，而其他的需要更多数据。

### Tuning Knobs Ranked by OtterTune

不需要 DBA 参与，使用 OtterTune Lasso 算法选择旋钮，然后用 LHS。

> 实际上 OtterTune 选择的 knobs 和 DBA 很近似，而且会选很重要的几个 knob 比如 DB_CACHE_SIZE

![](https://s2.loli.net/2024/04/08/yiqeu8EPX4W7JK3.png)

> LHS 配置无法识别？但是提升很大，10knobs 都有提升，但是 20knobs 只有 GPR 和 DNN 有提升，大部分要么没提升，要么就一点，甚至不超过默认配置。论文认为是当时共享存储的 noise 很多，导致了差异。但这差异也太大了。。。
>
> 论文也没怎么介绍 Lasso 算法，甚至在 10knobs 也不知道哪个算法是比较好的，有些小的改进还可能是 noise 引起的

### Adaptability to Different Workload

We next analyze the quality of ML-generated configurations when we train their models on one workload and then use them to tune **another workload**. The ability to reuse training data across workloads potentially reduces the number of iterations that the algorithms need for their models to converge.

> 最近看的一些调优相关的，都很注重 across workloads

用 TPC-C 训练

We examined the configurations for differences in the best-observed settings for TPC-C and TicketTracker that may explain why the algorithms were unable to achieve performance comparable to the results in Figures 11 and 13. We observed that **none of the algorithms changed the sizes of the two buffer caches** in Table 2 from their SG default settings. This is expected for the LOB buffer cache since none of TPC-C’s data is stored in the 32 KB tablespace.

> 前文提到的 LOB ，但是却没有修改 LOB buffer，因为 TPC-C 数据比较小，而他们实验是在 TicketTracker 这种很大的数据集上，实际上需要打开。

### Execution Time Breakdown

DDPG++ has **fewer canceled replays** than DDPG due to its **improved convergence rate** (see Section 4.3). LHS has the highest workload execution time and percentage of canceled replays because it is a sampling technique and never converges

## LESSONS LEARNED

(1) Handling Long-running Configurations:

Given this, the controller needs to support an **early abort mechanism** that stops long-running workload replays. Setting the early abort threshold to lower values is beneficial because it reduces the total tuning time. This threshold, however, must be set high enough to account for variability in cloud environments. We found that the 45-minute cut-off worked well, but further investigation is needed on more robust methods. For early aborted configurations, the DBMS’s metrics, especially Oracle’s DB Time, are incorrectly smaller because the workload is cut off. Thus, to correct this data, the controller calculates a completion ratio as the number of finished transactions divided by the total transactions in the workload. It then uses this ratio to scale all counter metrics to approximate what they would have been if the DBMS executed the full workload trace

> 需要一个可以早起终止的机制

(2) Handling Failed Configurations

> 需要一个机制可以知道什么时候 knob 不正确
> ，

(3) DBMS Maintenance Tasks

(4) Unexpected Cost Considerations:

## RELATED WORK

略

## CONCLUSION

In this study, we conducted a **thorough evaluation** of machine learning-based DBMS knob tuning methods with a real workload on an Oracle installation in an enterprise environment. We implemented three state-of-the-art ML algorithms in the OtterTune tuning service in order to make a head-to-head comparison. Our results showed that these algorithms could generate knob configurations that improved performance by up to 45% over ones generated by a human expert, but the performance was influenced by the number of tuning knobs and the assistance of human experts in knob selection. We also solved several deployment and measurement issues that were overlooked by previous studies

> 这篇论文读着太吃力了，而且对 ML 调优的印象更加差了。做了一堆事情，好像什么都没解决，也不知道原因。
>
> 更像是 OtterTune 在现实数据库和数据集的一个尝试，但这个新的数据集区别又很大，实验结果也很莫名其妙，10-20 knobs 效果有，但是 40 knobs 效果不好又说是共享存储的问题

> The paper investigates the effectiveness of using machine learning (ML) for automatic configuration and knobs tuning in real-world database management systems (an enterprise database called Oracle). Since previous ML-based tuning methods were tested on synthetic workloads and cannot represent real-world conditions, the paper conducted an evaluation of ML-based tuning methods using the OtterTune framework on an Oracle DBMS with a real-world workload trace SG and TicketTracker. The paper experiments with many ML algorithms (GPR, DNN, DDPG, DDPG++ and LHS) for automatic DBMS knob tuning, and achieves over 45% improvement with 10 and 20 tuning knobs chosen by OtterTune and compared the results with knobs chosen by DBA. And the paper also discusses the problems encountered in the experiment, such as I/O latency with shared storage, long tuning time and convergence speed.

> Contributions:
>
> 1. While previous ML tuning methods mainly benchmarked on synthetic workloads and open-source DBMSs, this paper uses a real-world workload trace from a bank’s application(SG and TicketTracker) and commercial DBMS(Oracle), offering insights into the practical effectiveness of ML tuning.
> 2. The paper conducts lots of experiments by using the OtterTune framework to evaluate many ML-based methods on the commercial DBMS, they tested Gaussian Process Regression (GPR) and Deep Neural Networks (DNNs), as well as Reinforcement Learning (RL) with Deep Deterministic Policy Gradient (DDPG) and DDPG++.
> 3. Prior research often uses local storage like SSDs to do experiments, the paper points out that local storage can impact the performance and the effectiveness of the tuning algorithms being evaluated because shared storage is more real-world, and the I/O latency of shared storage will have an impact on tuning.
>
> Limitations:
>
> 1. Though the paper conducts lots of experiments, there was a big difference in the experimental results when using OtterTune tuning 20 knobs. The evaluation was heavily affected by the latency of non-local storage.
> 2. When the paper introduces various ML tuning methods, it does not carefully distinguish their differences. Especially when analyzing the experimental results, DDPG++ should converge faster than DDPG, but the experiments cannot present the advantages of DDPG++.
> 3. The paper mentions that when using shared storage, I/O latency will lead to great variability in tuning results, but it does not provide a control group or a reproduction of the situation. Moreover, the TicketTracker database looks very different from TPC-C. The paper should try to analyze some other representative real-world workloads.

> The paper did a lot of experiments, but the paper also mentioned that Oracle fixed a very important issue in the later stage. I guess this will require re-experimenting, although this may be very time consuming. I may also want to try some methods to reduce the tuning time, because this will allow me to experiment with more rounds and reflect the differences between DDPG and DDPG++. In addition, I would like to conduct the experiments on other real-world workloads, which can demonstrate the ability of OtterTune and ML-based tuning methods to tune on diverse datasets and workloads. In addition, I might also want to carefully explain the impact of shared storage on ML-based tuning methods and OtterTune, especially setting up a control group to experiment, or try to reproduce the situation at that time to find out the real problem.
