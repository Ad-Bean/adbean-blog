+++
title = 'Reading Map'
date = 2026-09-26T00:00:00+08:00
draft = false
categories = ['Notes']
tags = ['Index']
+++

## 这是什么

这个博客的知识索引。所有文章按 **6 条轨道**归类，每条轨道标出**读到哪里、断在哪**。以后再写新的，挂到对应轨道下面即可。

**总览**：61 篇，跨 2024-01 → 2025-12。最大的两块是 **OS/Kernel（22）** 和 **Database（11）**——论文读得很多，但**动手只到 Mini-LSM 第一周**。这是目前最大的缺口。

> 图例：`●` 在读 / 未收尾   `✓` 已完成一个阶段   （无标记）单篇论文笔记

---

## 一、OS & Kernel（22 篇）

数据中心系统、虚拟化、内核旁路、远端内存。这是你读得最厚的一块。

**虚拟化 / 隔离**
- [Xen and the Art of Virtualization](/posts/paper-xen-vir/) — Xen 半虚拟化
- [Firecracker](/posts/paper-firecracker/) — 轻量级 serverless 虚拟化
- [From Laptop to Lambda](/posts/paper-gg/) — 瞬时函数容器（也见轨道五）

**OS 架构**
- [Exokernel](/posts/exokernel-paper/) — 应用级资源管理
- [The Multikernel](/posts/multikernel-os/) — 多核可扩展内核
- [The Demikernel](/posts/paper-demikernel/) — 微秒级数据面 OS
- [SigmaOS](/posts/paper-sigmaOS/) — 统一 serverless 与微服务

**内核旁路 / 数据面**
- [Arrakis](/posts/paper-arrakis/) — OS 即控制面
- [Junction](/posts/paper-junction/) — 让内核旁路在云上可用
- [IX: A Protected Dataplane OS](/posts/ix-paper/) — 高吞吐低尾延迟
- [Shenango](/posts/paper-shenango/) — 延迟敏感负载的 CPU 效率
- [Caladan](/posts/paper-caladan/) — 微秒级干扰隔离
- [Enso](/posts/paper-enso/) — NIC-应用通信的流式接口
- [Dune](/posts/dune-paper/) — 用户态访问特权 CPU 特性

**调度 / eBPF**
- [ghOSt](/posts/paper-ghost/) — 用户态接管 Linux 调度
- [XDP](/posts/paper-XDP/) — 可编程包处理
- [XRP](/posts/paper-XRP/) — 内核内 eBPF 存储函数

**内存 / 远端内存**
- [Software-Defined Far Memory](/posts/paper-far-memory/) — 仓库级计算
- [Fastswap](/posts/paper-fastswap/) — 远端内存能提吞吐吗
- [AIFM](/posts/paper-aifm/) — 应用集成的远端内存
- [Temeraire](/posts/paper-hugepage/) — hugepage 感知的分配器
- [内存知识笔记 01](/posts/memo-class-1/) — 为什么可用内存远超物理内存

**RDMA**
- [PRISM](/posts/paper-PRISM/) — 重新思考 RDMA 接口

---

## 二、Database（11 篇）

**存储引擎（动手）**
- ● [Mini-LSM Week 1 Day1](/posts/minilsm-1/) · [Day2](/posts/minilsm-2/) · [Day3](/posts/minilsm-3/) — **断点：只做完 Week 1**（Week 2 compaction、Week 3 MVCC 未做）

**书**
- [DDIA 序章 + 第一章](/posts/ddia-1/)
- [DDIA 第二章：数据模型与查询语言](/posts/ddia-2/)

**系统论文**
- [DuckDB](/posts/duckdb-2019/) — 可嵌入分析型数据库
- [Delta Lake](/posts/paper-delta-lake/) — 对象存储上的 ACID 表
- [Milvus](/posts/milvus-paper/) — 向量数据管理系统
- [AnalyticDB-V](/posts/analyticDB-paper/) — 查询融合的混合分析引擎
- [VBase](/posts/vbase-paper/) — 统一向量检索与关系查询
- [Apiary](/posts/paper-apiary/) — DBMS 集成的 FaaS

---

## 三、Learned Data Systems（9 篇）

机器学习 / LLM 进入数据系统：学习型索引、代价模型、自动调优、NL2SQL。

- [Bao](/posts/bao-learned-query-opt/) — 让学习型查询优化变得实用（SIGMOD 21）
- [The Case for a Learned Sorting Algorithm](/posts/learned-sorting/) — 学习型排序（SIGMOD 20）
- [Zero-Shot Cost Models](/posts/zero-shot-learned/) — 开箱即用的学习型代价预测（VLDB 22）
- [MB2](/posts/mb2-self-driving-db/) — 自驱动 DBMS 的行为建模
- [ML-based Automatic Configuration Tuning](/posts/ML-based-autotunning/)
- [DB-BERT](/posts/llm-capability/) — 读手册的调优工具
- [SEED](/posts/SEED-paper/) — 用 LLM 做领域数据整理
- [SPADE](/posts/spade-paper/) — 为 LLM 流水线合成断言
- [CatSQL](/posts/catsql-nlsql/) — 面向真实世界的 NL2SQL（VLDB 23）

---

## 四、ML Systems（8 篇）

**CMU 10-414/714 Deep Learning Systems（2020）** —— **断点在这里**
- ● [01 Softmax](/posts/ml-sys-01/) · [02-03 Neural Networks](/posts/ml-sys-02/) · [04 Automatic Differentiation](/posts/ml-sys-03/) · [Autograd 实现](/posts/ml-sys-04/) · [hw0](/posts/ml-sys-hw0/)
- 从零写深度学习库 *Needle*；做到 autograd，**hw1（自动微分框架）之后停**。这是整个博客最新的学习痕迹（2025-03）。

**训练并行论文**
- [Megatron-LM](/posts/paper-megatron-lm/) — 模型并行训练
- [Megatron-LM v2](/posts/paper-megatron-lm-v2/) — GPU 集群上高效大规模训练
- [PipeDream](/posts/paper-pipedream/) — 流水线并行（SOSP 19）

---

## 五、Distributed & Cloud（5 篇）

- [Sky Computing](/posts/paper-skycomputing/) — 从云计算到天空计算
- [ServiceRouter](/posts/paper-servicerouter/) — Meta 的超大规模低本服务网格
- [Scalability! But at what COST](/posts/scalability-cost/) — 单机也能赢？
- [From Laptop to Lambda](/posts/paper-gg/) — 瞬时函数容器
- [Ownership](/posts/paper-ownership/) — 细粒度任务的分布式 future

---

## 六、Notes（6 篇）

- ● [Agentic Design Patterns 第一章](/posts/adp-01/) — **读了 86 字就停**，通篇在质疑 prompt chaining 是否真有用
- [2024 的一些计划](/posts/my-first-post/)
- [微服务架构](/posts/backend-microservice/)
- [Go 并发编程实战 1-4](/posts/go-concurrent-1/)
- [Ryan Dahl: NodeJS](/posts/node-js-io/)
- [人月神话](/posts/reading-manmonth-1/)

---

## 断点与下一步

**三个断点**：
1. **Mini-LSM**：Week 1 之后（2024-07 最后一次记录）
2. **CMU 10-714 DLSys**：hw0 / autograd 之后（2025-03）
3. **Agentic Design Patterns**：第一章质疑后搁置（2025-12）

**核心问题**：读了 44 篇论文，但**动手只有 Mini-LSM 第一周**。缺的不是输入，是把论文变成能跑的东西。

**建议路线（选一条，别贪多；每条都自带判分 / 测试，学完写一篇博客）**：

| 路线 | 内容 | 为什么 |
|---|---|---|
| **A（推荐）** | [CMU 15-445 BusTub](https://15445.courses.cs.cmu.edu/fall2026/) | 把 Mini-LSM 的存储层补成完整 RDBMS：buffer pool → B+Tree → 查询执行 → 并发控制 → 恢复。C++20，与日常工作（RisingWave）直接相关 |
| B | [MIT 6.1810 xv6](https://pdos.csail.mit.edu/6.1810/2025/schedule.html) | 亲手写页表 / COW / 陷阱 / 文件系统 / mmap，对应轨道一的 22 篇 OS 论文。`make grade` 自带判分 |
| C | [MIT 6.5840](https://pdos.csail.mit.edu/6.5840/) 或 [TinyKV](https://github.com/tidb-incubator/tinykv) | 补分布式侧（Raft、分片 KV），自带测试 / GitHub Classroom 判分 |

---

## 参考资源（筛过的）

**动手实验**
- [skyzh/mini-lsm](https://skyzh.github.io/mini-lsm/) — 已有 3 周 + 给 coding agent 的引导轨
- [CMU 15-445 Fall 2025 录像](https://15445.courses.cs.cmu.edu/fall2025/)（非 CMU 看这版）
- [MIT 6.1810 xv6 labs](https://pdos.csail.mit.edu/6.1810/2025/schedule.html) · [MIT 6.5840 labs](https://pdos.csail.mit.edu/6.5840/)
- [aalhour/beachdb](https://github.com/aalhour/beachdb/) · [arjunsood2025/raft-kv-engine-project](https://github.com/arjunsood2025/raft-kv-engine-project)

**真干货博客**（有署名、读源码、可验证）
- [Jack Vanlightly](https://jack-vanlightly.com/) — Kafka/Pulsar/BookKeeper/Iceberg 内核，配 TLA+ 验证
- [Marc Brooker](https://brooker.co.za/) — 分布式系统 / AWS
- [Murat Demirbas](http://muratbuffalo.blogspot.com/) — 分布式系统研究博客，写了 15 年
- [Andrey Satarin · Testing Distributed Systems](https://asatarin.github.io/testing-distributed-systems/) — 策展清单，里面有 RisingWave、TigerBeetle、Jepsen、Elle、TLA+
- [TigerBeetle Blog](https://tigerbeetle.com/blog/) · [boringSQL](https://boringsql.com/posts/postgresql-mvcc-byte-by-byte/)

> 避雷：`reddb.io` / `scalix.world` / `semicolony.dev` 这类匿名、疑似 AI 批量生成的 SEO 站，不要采信。
