+++
title = 'Paper Reading: Apiary: A DBMS-Integrated Transactional Function-as-a-Service Framework'
date = 2024-11-16T16:08:33-05:00
draft = false
tags = ['Paper Reading']
+++

## Apiary: A DBMS-Backed Transactional Function-as-a-Service Framework

来自 DBOS 的一个很有意思的项目，DBOS 这个项目去年关注过，txn + serverless 很有趣的思路

> 传统 FaaS 客户端调用函数去连接 DB 查询
>
> DBOS 直接 DB as OS，提出万物都是表的概念
>
> https://juejin.cn/post/7345644573652582440 InfoQ 和 MeetDBOS 的一些介绍
> https://www.infoq.cn/article/r9eev0oqqs8qip31eutg > https://www.infoq.com/news/2024/03/dbos-cloud-serverless/
>
> 这一篇 Apiary 应该是开源版本的 DBOS，DBOS 本身是不开源的，DBOS 论文讨论了不少实现和 IPC 与 scheduler
>
> https://zhuanlan.zhihu.com/p/463722216 丁凯大佬的 DBOS 解读

## Abstract

开发人员越来越多地使用函数即服务（FaaS）平台来构建数据密集型应用程序，这些应用程序需要对数据进行低延迟和事务性操作，例如用于微服务或 Web 服务。然而，现有的 FaaS 平台在这些应用程序的支持上表现不佳，因为它们在物理上和逻辑上将应用程序逻辑（在云函数中执行）与数据管理（通过访问远程存储的交互式事务完成）分离开来。物理上的分离损害了性能，而逻辑上的分离则使得高效提供事务保证和容错性变得复杂。

本文介绍了一种名为 Apiary 的新型 DBMS 集成 FaaS 平台，用于部署和组合具有容错性的事务性函数。Apiary 通过将分布式 DBMS 引擎封装并将其用作函数执行、数据管理和操作日志记录的统一运行时，从而在物理上共置和逻辑上集成函数执行和数据管理，从而提供与同类系统相当或更强的事务保证，同时显著提高性能和可观察性。

结果显示，在微服务工作负载上，Apiary 的表现优于它们 2 到 68 倍，通过减少通信开销实现了这一优势。

## Introduction

![](https://s2.loli.net/2024/11/18/yFRaD9V4qSjHdLA.png)

FaaS 模型在数据密集型应用程序中越来越受欢迎：例如电子商务 Web 服务等需要低延迟和事务性操作的应用程序。然而，现有的 FaaS 平台在这些应用程序的支持上表现不佳，因为它们在物理上和逻辑上将函数执行与数据管理分离开来，采用了图 1a 所示的架构，并在每次数据操作时调用远程 DBMS

> 真的吗？这里说的是生产还是学术的 faas 模型。。
>
> 不一定所有 faas 都会去调用 dbms 把

逻辑上的分离使得容错性和事务保证变得复杂，因为函数可能会被任意地重新执行，而事务不能跨越多个函数。它们要么提供物理上的共置以获得良好的性能，要么提供逻辑上的集成以获得强有力的保证，但无法同时做到两者。

例如，Cloudburst[42]通过使用本地缓存来物理上共置计算和数据，但不提供事务支持。其他如 Boki[22]和 Beldi[47]通过**远程存储上的外部事务管理器提供事务性函数**，但这增加了已经很高的存储访问时间，达到了 3 倍。

本文介绍了 Apiary，一种用于**数据密集型应用程序的事务性、高性能 FaaS 平台**。与现有平台不同，Apiary 通过封装分布式 DBMS 引擎并将其用作函数执行、数据管理和操作日志记录的统一运行时，从而在物理上共置和逻辑上集成函数执行和数据管理（图 1b）。我们将函数编译为存储过程，即在非 SQL 语言中运行的本地 DBMS 事务，从而使函数成为控制流和原子性的基本单位。然后，我们利用这种集成来支持高效、容错性的函数组合和新颖的可观察性能力。我们证明，Apiary 提供了（a）与同类平台相当或更强的保证，（b）与最先进的系统相比，性能提升了 2 到 68 倍，以及（c）自动跟踪应用程序与数据库交互以实现可观察性。

设计用于数据密集型应用程序的 FaaS 平台的主要挑战是提供高效、容错性的函数组合。FaaS 开发者通过将函数组合成工作流 **workflows** 来编写复杂的程序。为了在故障面前保持稳健，工作流需要强大的执行保证：**它们必须始终运行到完成，并且每个函数必须恰好执行一次**。现有平台通过要求函数具有幂等性[6]或构建昂贵的外部事务管理器[47]来提供这些保证。相比之下，我们可以利用 Apiary 中函数和数据的集成，构建一个容错性前端，**以最小的开发者要求和低开销提供这些保**证。由于函数是存储过程，Apiary 可以对它们进行检测，以事务性地记录它们在 DBMS 中的执行情况。因此，如果工作流执行失败，前端可以通过从头开始调用其函数来安全地恢复它，跳过已经执行过的函数。然而，naively 地检测 instrumenting 所有函数是昂贵的，会降低性能高达 2.2 倍。因此，我们开发了一种**新颖的算法来识别函数何时可以安全地重新执行**，将开销降低到 <5%。

> exectly once 保证
>
> idempotent 幂等

Apiary 还解决了 FaaS 开发者面临的一个常见挑战：了解应用程序如何与数据交互的可观察性。

> tracing 是 dbos 的一个特色

总结起来，我们的贡献如下：

- 我们提出了 Apiary，一种事务性 FaaS 平台，通过封装分布式 DBMS 在物理上共置和逻辑上集成函数与数据。Apiary 在提供类似或更强保证的同时，性能优于研究和生产平台 2 到 68 倍。
- 我们利用 Apiary 的架构设计了一个 fault-tolerant frontend，用于编排函数工作流。它保证了无论是否发生故障，工作流总是能运行到完成，并且其中的每个函数都恰好执行一次。
- 我们利用 Apiary 的架构来增强**可观察性**，通过自动检测应用程序及其与数据的交互，实现了<15%的开销，相比手动日志记录的 92%开销有了显著改进。

## Apiary Overview

### System Architecture

![](https://s2.loli.net/2024/11/18/xd6GtAUayrcIniB.png)

Apiary 的设计动机源于一个关键观察：数据密集型应用程序在其运行时中，大部分时间要么在与 DBMS 通信，要么在执行 DBMS 操作。

> 这是假设还是观察？
>
> 其实我一直不明白 DBOS 是做了个 serverless DB 还是什么
>
> 看起来是非存算分离的架构？

我们将 Apiary 设计为通过封装分布式 DBMS 并将函数编译为数据库存储过程，从而在物理上共置计算和数据。由于存储过程是事务性的，这种架构也在逻辑上集成了函数执行和数据管理；我们利用这种集成来高效地提供事务保证、容错性和可观察性

Clients 客户端通过客户端库向前端发送请求，以调用工作流和函数，这些工作流和函数在后端执行。开发者使用我们的编程接口编写函数并组合工作流。

Frontend 前端服务器将客户端请求路由并认证到后端。每个服务器都有一个调度器，它通过在后端调用每个函数，传入其输入，收集其输出以发送给后续函数，并强制执行容错保证（§3，§4）来管理工作流执行。服务器还包含注册器，负责函数和工作流的注册、检测和编译。

Backend **后端执行函数并管理数据**。它封装了一个分布式 DBMS 及其存储过程。Apiary 在 DBMS 服务器上以**事务性**方式执行函数，对其进行检测以提供容错性（§3，§4），并捕获应用程序与数据库交互的信息以实现可观察性（§5）。由于函数在物理上与 DBMS 共置，我们依赖 DBMS 的本地弹性扩展能力来扩展后端。

### Non-Goals

Compute-Heavy Workloads Apiary 的设计专注于短生命周期的数据密集型应用程序，而不是长时间运行的计算密集型工作负载，如视频处理[17]或批量分析[37]

Non-Relational Data Models Apiary 目前仅支持关系型数据模型。我们相信可以将 Apiary 扩展以支持事务性的非关系型数据库，如 MongoDB，但这超出了本文的范围。

> 其实几个微服务 benchmark 也有 mongodb 把

## Apiary Semantics

### Programming Interface

酒店预订服务的例子来概述其编程接口，该服务检查房间是否可用，预订房间，然后发送确认邮件。

Function Interface:

Apiary 要求函数遵循三条规则：

1. 函数中的所有 SQL 查询必须静态定义 defined statically 为参数化预备语句 parameterized prepared statements。
2. 函数必须是确定性 deterministic 的。
3. 外部服务或 API 调用必须是幂等 idempotent 的。

第一条规则支持静态分析（§4）和数据跟踪以实现可观察性（§5）；最后两条规则支持实现恰好一次语义的实际实现（§4）。

Workflow Interface: 开发者使用图 5 中的接口将程序构造为函数的工作流。每个工作流是一个有向无环图（DAG），其中节点是函数，边是数据流，函数的输入是其父函数的输出。开发者从函数列表和将早期函数的输出映射到后期函数输入的规范中构建工作流。**不允许递归或循环依赖。**

![](https://s2.loli.net/2024/11/18/pIDX3FG6aOkB7Te.png)

### Transactional Semantics

FaaS 程序通常需要事务保证；例如，我们的酒店预订工作流需要事务来保证房间不会被重复预订。

因此，Apiary 在逻辑上集成了函数执行和数据管理：每个函数在数据库中作为可序列化的 ACID 事务执行。这一设计的一个关键含义是，函数既是控制流的基本单位，也是原子性的基本单位；我们利用这一点来实现容错性（§4），并跟踪应用程序与数据库的交互以实现可观察性（§5）。

一个重要的问题是我们为工作流提供什么样的事务语义。天真地，我们可以**将整个工作流在一个事务中执行**，但这会导致不必要的、性能较差的大型事务
但开发者通常需要跨多个函数的事务保证。例如，在酒店工作流中，前两个函数（checkAvail 和 reserve）必须在一次事务中执行，以确保房间在预订时确实可用。为了平衡这些权衡，我们提供了多函数事务 multi-function transactions: 开发者可以将工作流中的多个函数分组，以作为单个 ACID 事务执行，前提是它们形成工作流图的一个连通子图。这种设计使开发者能够灵活地事务性执行相关操作，但将不相关的操作分开，以避免过大事务的性能开销。

> 限制蛮大的

### Fault-Tolerance Guarantees

First, workflows run to completion 因此即使调度器或 DBMS 服务器的故障导致工作流执行暂停，工作流最终也会恢复。

Second, functions in workflows execute exactly once 因此即使工作流或其任何函数多次失败并重新启动，工作流对应用程序状态（在数据库中）的影响与工作流中的每个函数恰好执行一次的效果相同。例如，在酒店工作流中，Apiary 保证房间只被预订一次，并且如果预订成功，确认邮件总是会被发送。

### Comparison with Related Systems

在表 1 中，我们将 Apiary 的语义与相关系统的语义进行了比较。Apiary 提供的保证 guarantees 比商业系统如 AWS Step Functions[7]和 Azure Durable Functions[31]要强得多。这些系统支持运行到完成的工作流，但既不提供事务性函数保证，也无法看到应用程序数据库

AWS Step Functions 声称它也可以提供恰好一次的语义，但这实际上是一个最多一次的保证：它在失败时不重试，而是保证任务永远不会运行超过一次

> 比起 lambda，这类系统应该更加适合 dag 任务

Apiary 提供的保证与事务性 FaaS 系统如 Beldi[47]、Boki[22]和 Transactional Statefun[15]类似。它们都允许开发者为单个函数提供 ACID 事务保证，尽管 Transactional Statefun 实现了一个有限的“one-shot”模型，其中事务中较早查询的输出不能用作同一事务中较晚查询的输入。所有系统都为函数提供了**恰好一次**的语义，并支持运行到完成的工作流。**没有一个系统为整个工作流提供事务保证，但所有系统都支持在单个事务中运行多个函数**，类似于 Apiary 的多函数事务。在 Beldi 和 Boki 中，事务性函数可以同步调用其他函数，因此它们都在一个大事务中执行。在 Transactional Statefun 中，“协调器函数”可以通过两阶段提交协调多个其他函数，使它们作为一个事务执行。然而，尽管这些系统在远程存储上构建了昂贵的外部事务管理器，Apiary 通过 co-locating compute and data 来最小化事务开销。

一个重要的相关系统类别是围绕因果一致性构建的 FaaS 平台，如 Cloudburst[42]、Hydrocache[45]和 FaaSTCC[27]。这些系统将数据存储在远程键值存储中，并使用本地缓存来提高性能。它们提供的最强保证是整个工作流的 transactional causal consistency（TCC）。

TCC 保证工作流在**完全完成之前不能看到其他工作流的效果**，只有当整个工作流在一个多函数事务中执行时，Apiary 或其他事务性 FaaS 系统才能提供这种保证。然而，对于**单个函数**，TCC 提供了相对较弱的保证，允许严重的异常，如 stale reads 和 write-write conflicts。相比之下，**Apiary 将每个函数作为具有可序列化隔离的 ACID 事务运行**，不允许这些异常。此外，这些系统仅提供键值 API 进行数据管理，而 Apiary 支持**关系模型**。

![](https://s2.loli.net/2024/11/18/LhjwDnmuSTo5tZY.png)

## Fault-Tolerant Workflows

run-to-completion workflow execution and exactly-once function execution.

### Handling Machine Failures

Apiary 必须在调度器或 DBMS 后端服务器故障的情况下强制执行其执行保证。大多数分布式 DBMS 可以从其服务器故障中恢复，通常使用复制和日志记录。

- 单个服务器失败，DBMS 可以通过故障转移到副本而不会损失可用性，因此工作流执行不受影响。
- 多个服务器失败，DBMS 可以从持久日志中恢复而不会丢失数据，因此调度器必须等到恢复完成后再恢复执行。

如果在工作流执行期间调度器失败，具有待处理工作流调用的客户端会超时，然后重新提交其调用以在新调度器上恢复部分执行的工作流。为了唯一标识工作流以进行恢复，客户端为每次工作流调用生成一个唯一 ID，并使用数据库生成的唯一客户端 ID 作为前缀。新调度器必须从失败调度器停止的地方恢复工作流执行，完成工作流而不重新执行任何已完成的函数。由于 Apiary 将函数作为存储过程以事务性方式运行，我们可以通过检测函数在返回之前以事务性方式记录其输出（以二进制格式序列化）来实现这一点。每个记录的输出都与一个唯一函数调用 ID 相关联，该 ID 派生自工作流 ID，并且仅在工作流的生命周期内保留。在重试期间，调度器从开始恢复工作流并（重新）调度每个函数。函数首先检查早期执行的记录，如果找到一个，则返回它而不是执行。

**当前实现的局限性在于它依赖客户端检测调度器故障**。在未来的工作中，我们计划通过让调度器将工作流元数据预写到 DBMS 中，并以去中心化的方式相互 ping 以检测故障（使用 DBMS 进行发现）来解决这个问题。然后，检测到另一个调度器故障的调度器可以从 DBMS 中检索其待处理的工作流并完成其执行，而无需客户端参与。

> 比如酒店预订，如果某个函数出错，发生重试，唯一标识
>
> 客户端来检测超时，怎么设置呢？

### Optimizing Function Recording

协议在数据库中记录每个函数的输出，但这样做天真地会导致高达 2.2 倍的开销，因为它需要在每个函数中执行额外的数据库查找和更新。

然而，我们可以通过**识别一些函数**可以在**不违反恰好一次语义**的情况下安全地重新执行，从而将这种开销减少到<5%（在我们测试的所有工作负载中），因此它们的输出不需要记录。例如，如果整个工作流是**只读**的，如果其原始执行失败，它可以安全地重新执行，因此我们不需要记录其任何函数。因此，我们开发了一种新的算法，称为**选择性函数记录 selective function recording（SFR）**，使用静态分析在工作流注册时确定哪些函数必须记录，哪些可以安全地重新执行。

我们必须记录任何执行 DBMS **写操作**的函数，以确保写操作不会重新执行，此外，如果存在从它到多个不同记录函数或至少一个记录函数和汇的不相交路径，我们必须记录一个只读函数。

我们在算法 1 中概述了 SFR

Correctness: 我们不保证找到最小的记录函数集

> 没给证明，看上去是个拓扑排序，找相关联的，但是说时间复杂度很低，有点奇怪

Complexity: 我们可以记忆化工作流图搜索，因此，SFR 的时间复杂度为 O(V + E)，其中 V 是函数的数量，E 是工作流图中的边数。我们只在每次工作流注册时运行此算法一次。

![](https://s2.loli.net/2024/11/18/H5ruT4I7gCqfEnS.png)

### Handling Function Failures

Apiary 还必须在单个函数出现故障或错误的情况下强制执行其执行保证。

## Observability

开发者通常需要了解应用程序如何与数据交互的信息，以便用于调试、监控和审计用途，例如验证程序是否未不当访问私有数据

### Observability Interface

Apiary 检测工作流以跟踪工作流和函数执行的历史，检测查询以记录数据库操作，然后将这些信息结合起来，创建应用程序与数据交互的完整记录。

跟踪层自动将此信息 spool 到分析数据库（在我们的实现中，Vertica[43]）以进行长期存储和分析

### Implementing Tracing Layer

我们利用 Apiary 与数据的紧密集成，将数据库技术（如变更数据捕获和查询重写[4, 20]）适应到 FaaS 环境中，构建一个跟踪层，以高效捕获应用程序与数据的交互。

当一个函数执行时，跟踪层向 FunctionInvocations 添加一个条目。

> 实际上应该就是 log？
>
> CDC + 写入

## Implementation

### Choosing a DBMS

Apiary 需要一个具有以下四个特性的分布式 DBMS：

- 支持 ACID 事务。
- 支持在存储过程中以事务性方式运行非 SQL 语言的用户代码。
- 支持 change data capture（用于可观察性，§5.2）。
- 支持弹性 DBMS 集群调整大小。

尽管许多 DBMS 具有这些特性（例如，SingleStore[40]，Yugabyte[46]），我们选择了 VoltDB，因为它能最有效地执行我们的目标工作负载。大多数分布式 DBMS，包括 VoltDB，通过**数据分区**来扩展。我们观察到，在我们的目标工作负载中，几乎所有事务都是**单站点**[24]的，并且只访问单个分区中的数据。VoltDB 高效地执行这些事务，在内存中运行它们到完成，而不需要锁。然而，VoltDB 的一个限制是它在**多站点事务中效率较低**；一个事务必须持有全局锁才能访问多个分区上的数据。最近有一些研究试图解决这个问题[49]，但我们把高效实现 multi-sited transactions 多站点事务留到未来的工作中。

> 之前见过一些 serverless 框架也用 voltdb 实现，有机会看一下 voltdb
>
> 多站点事务：事务涉及多个分区的数据。例如，在下订单时，需要同时更新库存和订单信息，而这两个数据可能位于不同的分区。

### Compilation

当开发者在 Apiary 中注册函数和工作流时，它将每个函数编译为一个存储过程，这是一个在非 SQL 语言中运行的本地 DBMS 事务。
在我们的实现中，函数提供的保证与 VoltDB 事务相同：它们是 ACID 和可序列化的。为了实现多函数事务，Apiary 将所有涉及的函数编译为一个存储过程。编译分两步进行。首先，Apiary 检测每个函数以捕获应用程序与数据库的交互（§5），并记录其执行以实现恰好一次语义（§4）。然后，Apiary 将检测后的函数（或多函数事务）编译为存储过程，并将其注册到 DBMS 中。

Apiary 扩展了 DBMS 存储过程接口，因此它可以编译任何使用其编程接口（图 5）并遵循§3.1 中概述的规则的函数。此外，在我们的基于 VoltDB 的实现中，由于 VoltDB 可以高效执行单站点事务，我们让开发者指定一个函数（或多函数事务）是否是单站点的，如果是，哪个函数输入指定了站点。

## Evaluation

1. physically co-locating compute and data, 7–68× and research systems by 2–27×
2. By selectively instrumenting functions using the **SFR** algorithm, Apiary provides **fault tolerance** with overhead of <5% compared to 2.2× for a naive solution
3. By instrumenting database operations and functions, Apiary captures information on application-database interactions critical to **observability** with overhead of <15% as compared to 92% with manual logging

### Experimental Setup

### Baselines

![](https://s2.loli.net/2024/11/18/fF5BdpYUy7jh4mV.png)

> Microservice benchmark information 以后可以用到

![](https://s2.loli.net/2024/11/18/qWUFjcLyMufm6Ok.png)

> 为什么 Apiary 的延迟要比 rpc 更优秀一点呢？
>
> 个人认为就是少了一层通信。。直接访问数据库的意思？

### Microservice Workloads

Shop。这个基准测试改编自 Google Cloud 的演示[19]，模拟了一个服务，用户在其中浏览在线商店，更新购物车，并结账商品。

Hotel。这个基准测试来自 DeathStarBench[18]，模拟了搜索和预订酒店房间。我们的实现包含一个多函数事务，类似于图 6，其中验证和预订是事务性执行的。

Retwis。这个基准测试来自 Redis[38]

### End-to-End Benchmarks

Apiary 优于 RPC 服务器基准的原因是减少了通信开销：因为它将服务编译为在数据库服务器中运行的存储过程，所以执行数据库操作所需的往返次数更少

### Comparing with Boki and Cloudburst

Retwis 是读密集型的，因此为了评估写操作的性能影响，我们使用了一个微基准测试，该测试检索并递增与键关联的计数器。

> 没看过这两篇
>
> 但是 apiary 的性能好到离谱，认为 cloudburst 用 py 实现都本地缓存慢，voltdb 读更快，大量读的情况更好

### Fault-Tolerant Workflows Performance Analysis

### Enhancing Observability with Apiary

> 其实按理来说，我觉得 dbos/apiary 更出彩的地方是 tracing 和 fault tolerance 但论文却没大量讲，整个结构也不是很完善，没第一次看 dbos 那么有意思，比如说 time travel, OpenTelemetry traces

### Cost Analysis
