+++
title = 'Paper Reading: Ownership: A Distributed Futures System for Fine-Grained Tasks'
date = 2024-11-10T16:27:22-05:00
draft = false
tags = ['Paper Reading']
+++

## Ownership: A Distributed Futures System for Fine-Grained Tasks

分布式系统中的一些任务调度的论文，本来是要看 Ray 的，同样都是 UCB 的研究（分布式机器学习框架）

Ownership 我一开始还以为是 Rust 里的所有权概念，弄混了，论文也提到许多 Futures, distributed futures interface 之类的第一次接触，实际上是 RPC 的扩展

> 最重要的概念是 Distributed Memory + Ref + Future
>
> 不再需要经典的 RPC 值传递，
>
> A call worker1 得到 o1 就返回引用而不是 o1
>
> A call worker2 得到 o2 返回引用
>
> A call worker2 o1+o2，worker2 解引用去 w1 找 o1
>
> future 就是异步，但是 解引用的时候就会有很多问题，所以有了这篇论文，但看下来感觉主题不太清晰，效果也不太优秀，不太令人信服。

## Abstract

分布式 Futures 接口 distributed futures interface

**Distributed futures** are an extension of **RPC** that combines futures and distributed memory

分布式 futures 是一个引用，其最终值可能存储在远程节点上。
然后，应用程序可以在**不指定执行时间**或**数据移动位置**的情况下表达分布式计算。

最近的分布式 Futures 应用程序需要执行**细粒度计算**的能力，即在**毫秒级**运行的任务。与**粗粒度任务**相比，细粒度任务难以以可接受的系统开销执行。

在本文中，我们提出了一种用于细粒度任务的分布式 Futures 系统，该系统在**不牺牲性能**的情况下提供**容错能力**。我们的解决方案基于一种称为 Ownership 的全新概念，该概念为每个对象分配一个系统操作的**leader**。我们表明，这种**去中心化**的架构可以实现**水平扩展**、每任务 1 毫秒的延迟和快速故障处理。

## Introduction

RPC 是构建分布式应用程序的标准，最初的提案使用**同步调用 synchronous**，将返回值复制回调用者
最近的一些系统 [4, 34, 37, 45] 扩展了 RPC，使得系统不仅可以管理分布式通信，还可以代表应用程序管理数据移动和并行性。

> PyTorch - Remote Reference Protocol.
>
> Ray
>
> 这些分布式训练框架使用 rpc，A 节点调用 B 节点的方法，传入参数，返回结果是一个远程引用 rref，可以通过移到本地而不是返回值

Data movement: **按值传递语**义要求所有 RPC 参数通过直接复制到请求体中发送给执行者

为了减少数据复制，一些 RPC 系统使用分布式内存，这允许通过引用**传递大参数**，而小参数仍然可以通过值传递。在最理想的情况下，如果参数已经位于执行者所在的同一节点上，则通过引用传递的 RPC 参数不需要复制。请注意，与传统的 RPC 一样，我们使所有值不可变，以简化一致性模型和实现。

> 值不可以变？类似函数式编程？缺点是内存使用增加

Parallelism: 传统的 RPC 是阻塞的，因此只有在收到回复后才会将控制权返回给调用者（图 2a）。Futures（Futures）是一种流行的方法，用于通过异步性扩展 RPC [8, 29]，允许系统在彼此之间以及与调用者并行执行函数。通过组合 [29, 37]，即将 Futures 作为参数传递给另一个 RPC，应用程序还可以表达 Futures RPC 的并行性和依赖关系。例如，在图 2c 中，add 在程序开始时被调用，但只有在计算出 a 和 b 之后，系统才会执行。

Distributed futures: RPC 的扩展，结合了 Futures 和分布式内存：分布式 Futures 是一个引用，其最终值可能存储在远程节点上（图 2d）。然后，应用程序可以在**不指定执行时间**或**数据移动位置**的情况下表达分布式计算。这对于开发处理大量数据的分布式应用程序来说是一个越来越受欢迎的接口。

> 这个 Future 太迷惑了，我总把 Rust 里的 Future 异步编程弄混
>
> 可能这种异步计算都可以叫 Future / Promises
>
> Futures are a popular method for extending RPC with asynchrony

不幸的是，现有的分布式 Futures 系统仅限于粗粒度任务

在本文中，我们提出了一种用于细粒度任务的分布式 Futures 系统。虽然其他人 [34, 37, 45] 之前已经实现了分布式 Futures，但我们的贡献在于识别并解决了在不牺牲性能的情况下为细粒度任务提供**容错能力**的挑战。

> 34 Ray, CIEL, Dask
>
> 这些都实现了分布式 Future

主要挑战是分布式 Futures 在进程之间引入了**共享状态**。特别是，一个对象及其元数据由其引用持有者、创建对象的 RPC 执行者以及其物理位置共享。为了确保每个引用持有者能够**解引用值**，进程必须**协调 coordinate**，这是一个在存在故障时难以解决的问题。相比之下，传统的 RPC 没有共享状态，因为数据是通过值传递的，并且自然避免了协调，这对于可扩展性和低延迟至关重要。

例如，在图 2a 中，一旦 worker 1 将 a 复制到 driver，它 (worker1) 就不需要参与下游 add 任务的执行。相比之下，worker 1 在图 2d 中存储了 a，**因此两个 worker 必须协调以确保 a 在 worker 2 读取时足够长时间可用**。此外，worker 1 必须在 worker 2 执行 add 并且没有其他引用时对 a 进行**垃圾回收**。最后，进程必须协调以检测和恢复另一个进程的故障。

![](https://s2.loli.net/2024/11/11/oM5FCugkGBy61f4.png)

以前系统中的常见解决方案是使用集中式主节点来存储系统状态并协调这些操作，确保容错的一种简单方法是将与操作相关的元数据同步记录和复制到主节点。

> Ray 使用了主节点 Head Node https://docs.ray.io/en/latest/cluster/key-concepts.html
>
> 只要保证 a 不会在调用 add 前被垃圾回收？ray 用 head node 来任务调度、资源管理

因此，分散系统状态对于可扩展性是必要的。问题是如何在不复杂化协调的情况下做到这一点。我们工作的关键见解是利用应用程序结构：分布式 Futures 可以通过**引用传递共享**，但大多数分布式 Futures 在调用者的范围内共享。例如，在图 1 中，a_future 被创建然后传递给同一范围内的 add。

因此，我们提出了所有权，一种在 **RPC executors 之间分散系统状态**的方法。特别是，任务的调用者是返回的 Futures 及其相关元数据的拥有者。在图 2d 中，**driver 拥有 a、b 和 c**。

这个解决方案有三个优点。首先，对于 horizontal scaling，应用程序可以使用**嵌套任务**将系统状态“分片”到 worker 上。
其次，由于 Futures 的拥有者是任务的调用者，任务延迟较低，因为所需的元数据写入虽然是同步的，但却是本地的。**这与一致性哈希等应用程序无关的分片方法形成对比**。第三，每个 worker 实际上成为了它所拥有的分布式 Futures 的集中式主节点，简化了故障处理。

> 水平扩展就是加机器
>
> 和一致性哈希有什么区别？写入元数据写本地，与其他节点无关？而一致性哈希需要所有节点都知道？

系统保证如果 Futures 的拥有者存活，任何持有该 Futures 引用的任务**最终都可以解引用该值**。这是因为拥有者将协调系统操作，如引用计数（用于内存安全）和 lineage reconstruction（用于恢复）。当然，如果拥有者失败，这还不够。

在这里，我们依赖于 lineage reconstruction 和对应用程序结构的第二个关键见解：在许多情况下，对分布式未来的**引用由失败拥有者的后代任务持有**。失败的任务可以通过其拥有者的血统重建**重新创建**，后代任务也将在此过程中重新创建。因此，与 futures 的拥有者共享任何持有分布式未来引用的任务是安全的。由于我们预计故障相对较少，我们认为这种**减少系统开销和复杂性的好处超过了故障时额外重新执行的成本**。

总之，我们的贡献是：

- 一种具有 transparent recovery 和 automatic memory management 分布式未来的去中心化系统。
- 一种基于 lineage reconstruction and fate sharing 的轻量级透明恢复技术。
- 在 Ray 系统 [34] 中的实现，提供了高吞吐量、低延迟和快速恢复。

> 分布式数据库应该比较常用 checkpoint + log 来做故障恢复把？

## Distributed Futures

### API

分布式未来的主要优势在于系统可以 transparently manage parallelism 和 data movement

![](https://s2.loli.net/2024/11/11/6FtTGDWCbXx2avi.png)

> f 是调用函数，传入 x 引用
>
> get(x) 是解引用，del(x) 删除引用
>
> Actor.f(x) 带状态调用 f
>
> shared(x) 返回 shared 状态可以传 x 到其他 worker，而不用解引用
>
> f(shared x) 传送 x 当作 first-class ，会解引用 x
>
> Distributed Futures 缩写 DFut

To spawn a task, **调用者**调用一个远程函数，该函数立即返回一个 DFut（表 1）。启动的任务包括函数及其参数、资源需求等。返回的 `DFut` 引用由函数返回的对象的值。**调用者**可以通过 get 解引用 DFut，这是一个阻塞调用，返回对象的副本。**调用者**可以删除 DFut，将其从作用域中移除，并允许系统回收该值。与其他系统 [34, 37, 45] 一样，所有对象都是不可变的。

通过任务调用创建 DFut 后，**调用者**可以通过两种方式创建其他引用。首先，调用者可以将 DFut 作为参数传递给另一个任务。DFut 任务参数由**系统隐式解引用**。因此，任务只有在所有上游任务完成后才会开始，执行者只能看到 DFut 的值。

其次，DFut 可以作为 first-class 传递或返回 [21]，即在不解引用的情况下传递给另一个任务。表 1 展示了如何将 DFut 转换为 SharedDFut，以便系统可以区分何时解引用参数。我们将接收 DFut 的进程称为借用者，以区别于原始调用者。与原始调用者一样，借用者可以通过传递 DFut 或再次转换为 SharedDFut（创建更多的借用者）来创建其他引用。

与最近的系统 [4, 34, 45] 类似，我们**支持带有 actor 的有状态计算**。调用者通过调用远程构造函数来创建一个 actor。这立即返回对 actor 的引用（ARef），并在远程进程上异步执行构造函数。ARef 可以用于启动绑定到同一进程的任务。与 DFut 类似，ARef 是一等值，即调用者可以将 ARef 返回或传递给另一个任务，系统会在所有 ARef 超出作用域后自动收集 actor 进程。

> Actor 管理状态计算

### Applications

分布式未来的典型应用是那些需要 RPC 的灵活性以及数据移动和并行性优化的应用。

分布式未来之前已被探索用于数据密集型应用，这些应用无法有效地表达或执行为数据并行程序。Ciel 识别了在执行过程中动态指定任务的关键能力，例如基于先前的结果，而不是**预先指定整个图** [37]。这使得新的工作负载成为可能，例如动态规划，它本质上具有递归性

> 任务调度？

Model serving:

图 3a 展示了一个基于 GPU 的图像分类管道的示例。每个客户端将其**输入图像**传递给一个 Preprocess 任务，例如调整大小，然后将返回的 **DFut** 与一个 **Router actor** 共享。Router 实现了**调度策略**，并通过引用将 DFut 传递给选定的 Model actor。然后，Router 将结果返回给客户端。

actor 通过两种方式提高性能：(1) 每个 Model 在其本地 GPU 内存中保持权重预热 keeps weights warm，(2) Router 缓冲预处理的 DFut，直到它有一批请求传递给 Model，以利用 GPU 并行性提高吞吐量。通过动态任务，Router 还可以选择在超时时刷新其缓冲区，以减少批处理带来的延迟。

First-class distributed futures 对于减少路由开销非常重要。它们允许 Router 将预处理图像的引用传递给 Model actor，而不是复制这些图像。这避免了在 Router 处形成瓶颈，我们在图 15a 中对此进行了评估。虽然应用程序可以使用**中间存储系统来存储预处理图像**，但它必须管理额外的关注点，如垃圾回收和故障。

Online video processing:

视频处理算法通常具有复杂的数据依赖关系，这些依赖关系不受 Apache Spark 等数据并行系统的良好支持 [22, 43]。例如，视频稳定（图 3b）通过跟踪帧之间的对象（Flow），对这些轨迹进行累积求和（CumSum），然后应用移动平均（Smooth）来工作。帧与帧之间的依赖关系很常见，例如图 3b 中存储在 actor 中的视频解码状态。每个阶段每帧运行 1-10 毫秒。

> Spark 为什么不行？是 GPU 限制还是并行关系？
>
> 论文 Scanner: Efficient video analysis at scale,
>
> Lightdb: A DBMS for virtual reality video.

在这种设置中，安全和及时的垃圾回收可能具有挑战性，因为单个对象（例如视频帧）可能被多个任务引用。实时视频处理也对延迟敏感：输出必须以与输入相同的帧速率生成。低延迟依赖于帧之间的流水线并行性，因为应用程序无法等待多个输入帧出现才开始执行

![](https://s2.loli.net/2024/11/11/cUNtMJaxmLV35zK.png)

> DFut 的错误检测和错误恢复

## Overview

### Requirements

系统保证每个 DFut 都可以解引用到其值。这涉及三个问题：automatic memory management, failure detection, and failure recovery.

> 就是把值传递改成了引用传递
>
> 可以减少网络传递的大小，但是带来的问题却很多，生命周期，错误检测，错误恢复

Automatic memory management: 引用计数？

Failure detection: 系统检测到 DFut 由于 worker 故障而无法解引用的时间。在没有未来但有分布式内存的情况下，这是直接的，因为在创建引用时已知值的位置。

    未来的加入使故障检测变得复杂，因为可以在值之前创建引用。
    因此，在图 4b 中，当 worker 2 接收到 add RPC 时，可能没有 a 的位置。然后，worker 2 必须决定 f 是否仍在执行，或者是否已经失败。如果是前者，那么 worker 2 应该等待。但如果发生故障，系统必须恢复 a。为了解决这个问题，系统必须记录所有任务的位置，即待处理对象，而不仅仅是已创建的对象。

> 异步

Failure recovery: 系统还必须提供一种从失败的 DFut 中恢复的方法。最低要求是如果应用程序尝试解引用失败的 DFut，则抛出错误。我们进一步提供透明恢复的选项，即系统将恢复失败的 DFut 的值。

在没有分布式内存但有 Future 的情况下，如果一个进程失败，我们将失去该进程上任何待处理任务的回复。假设幂等性，这可以通过**重试**来恢复，这是按值传递 RPC 的常见方法。例如，在图 5a 中，driver 通过重新提交 add(a, b) 来恢复。故障恢复很简单，因为所有数据都是按值传递的。

然而，在分布式内存的情况下，任务还可以包含通过**引用传递的参数**。因此，节点故障可能导致仍然被引用的对象值丢失，如图 4b 中的 b。解决这个问题的一个常见方法是记录每个对象的 lineage，即**在运行时生成对象的子图** [17, 30, 56]。然后，**系统遍历丢失对象的血统**，并通过任务重新执行递归地重建对象及其依赖关系。**这种方法减少了日志记录的运行时开销**，因为数据本身没有记录，并且部分失败后必须重做的工作，因为分布式内存中缓存的对象不需要重新计算。尽管如此，实现低运行时开销仍然很困难，**因为血统本身必须在运行时记录和收集，并且必须能够承受故障**。

请注意，我们特别关注 object recovery，并且与之前的系统 [34, 37, 56] 一样，假设幂等性以确保正确性。因此，我们的技术直接适用于幂等函数和具有只读、 checkpointable 或瞬态状态的 actor，正如我们在图 15c 中评估的那样。虽然这不是我们的重点，但这些技术也可以与已知的 actor 状态恢复技术结合使用 [17, 34]，例如 nondeterministic execution 的恢复 [52]。

Metadata requirements: 在正常操作期间，系统必须至少记录以下内容：(1) 每个**对象值的位置**，以便引用持有者可以检索它，(2) 对象是否仍然被**引用**，以确保安全的垃圾回收。对于故障检测和恢复，系统还必须分别记录 (3) 每个 pending 对象的位置，即**任务位置**，以及 (4) **对象血统**。

关键问题是何时以及在哪里记录这些系统元数据，以确保其一致性和容错性。

在某些情况下，元数据的异步更新是安全的，即系统元数据与系统状态之间存在暂时不一致
另一方面，故障处理所需的元数据理想情况下应同步更新。

> 所以到底是异步还是同步更新？

### Existing solutions

![](https://s2.loli.net/2024/11/11/V8S6zuGCNZ7YOvn.png)

Centralized master: 使用同步更新的集中式主节点进行故障处理，但这种设计也会增加显著的运行时开销，故障检测要求主节点在**调度之前记录任务的计划位置**，这使得主节点成为可扩展性和延迟的瓶颈。主节点可以通过分片来提高可扩展性，但这会使协调多个对象的操作（如垃圾回收和血统重建）变得复杂。此外，延迟开销是根本性的。每个任务调用必须首先联系主节点，即使不复制元数据以实现容错，也会在执行的关键路径上增加至少一次 RTT。

Distributed leases: 这类似于异步更新的集中式主节点，这种设计通过分片实现**水平可扩展性**，并减少任务延迟，因为元数据是异步写入的。然而，**依赖时间来协调系统状态会减慢恢复速度**（图 14）。此外，这种去中心化方法引入了一个新问题：工作节点还必须在**谁应该恢复对象（即重新执行创建任务）上进行协调**。这在集中式方案中很简单，因为主节点协调所有恢复操作。

> 在 Kubernetes 中，租约机制用于节点心跳和组件级别的领导者选举
>
> 分布式原理介绍 https://lrita.github.io/images/posts/distribution/%E5%88%86%E5%B8%83%E5%BC%8F%E5%8E%9F%E7%90%86%E4%BB%8B%E7%BB%8D.pdf
>
> 分布式系统中的 lease（租约）机制 https://lrita.github.io/2018/10/29/lease-in-distributed-system/
>
> 基于 PAXSO 算法的，时钟漂移有上界的部分同步网络的 lease(租约)算法 https://lrita.github.io/images/posts/distribution/flease-fault-tolerant-and-decentralized-lease-coordination-for-distributed-systems.pdf
>
> 这里提到的等待租约过期，那我将租约寿命调低，不断续约能否解决呢？
>
> 至于谁来恢复，用主节点协调其实挺好的？这也是为什么 Ray 继续用 head node 吧

### Our solution: Ownership

我们工作的关键见解是“shard”集中式主节点以提高可扩展性，但基于应用程序结构来实现低运行时开销和简单的故障处理。

在所有权中，**调用任务的工作节点存储与返回的 DFut 相关的元数据**。与**集中式主节点一样，它协调操作，如任务调度，以确保它知道任务位置，并进行垃圾回收**。例如，在图 6d 中，worker 1 拥有 X 和 Y。

选择任务的**调用者作为所有者**的原因是，通常情况下，它是访问元数据最频繁的工作节点。调用者通过任务调用参与 DFut 的初始创建，并通过将 DFut 传递给其他 RPC 来创建其他引用。因此，任务调用延迟最小，因为计划位置是本地写入的。类似地，如果 DFut 保持在所有者的作用域内，垃圾回收的开销很低，因为当所有者将 DFut 传递给另一个 RPC 时，可以在本地更新 DFut 的引用计数。对于小对象，这些开销可以进一步减少，这些对象可以按值传递，就像没有分布式内存一样（参见第 4.2 节）。

当然，如果所有任务都由单个驱动程序提交，就像在 BSP 程序中那样，所有权机制将无法扩展到驱动程序的吞吐量之外。任何动态任务系统也是如此。然而，通过**所有权机制**，应用程序可以通过将控制逻辑分布在**多个嵌套任务中来水平扩展**，而不是像一致性哈希（图 12e）这样的应用程序无关的方法。此外，worker 进程持有系统元数据的大部分。这与之前的解决方案形成对比，之前的解决方案将所有元数据推送到系统的集中式或每个节点的进程中，限制了单个节点在拥有许多 worker 进程时的垂直可扩展性（图 12）。

> 单个节点，single failure？

然而，有些问题是假设性能足够的情况下，使用**完全集中式设计更容易解决**的：

First-class futures 第一类未来（第 2 节）允许非所有者进程引用 DFut。虽然许多应用程序可以在没有第一类未来的情况下编写（图 3b），但它们在某些情况下对性能至关重要。例如，图 3a 中的模型服务应用程序使用第一类未来将任务调用委托给嵌套任务，而无需解引用和复制参数。第一类 DFut 可能会离开所有者的范围，因此我们必须考虑在垃圾回收期间的情况。我们避免在所有者处集中引用计数，因为这将违背委托的目的。相反，我们使用分布式分层引用计数协议（第 4.2 节）。每个借用者代表所有者存储 DFut 的本地引用计数（表 2），并在本地引用计数达到零时通知所有者。所有者决定何时可以安全地回收对象。我们使用**引用计数方法**而不是追踪 [42] 来避免全局暂停。

Owner recovery: 如果一个 worker 失败，那么我们也会丢失其拥有的元数据。为了实现透明恢复，系统必须在新进程上恢复 worker 的状态，并重新关联与之前拥有的 DFut 相关的状态，包括任何值的副本、引用持有者和挂起的任务。我们选择了一种最小化的方法，保证进度，但可能会在失败时增加额外的重新执行成本：我们与所有者共享对象和任何引用持有者的命运，然后使用行踪重建来恢复对象和所有者的任何命运共享的子任务（第 4.3 节）。这种方法增加了最小的运行时开销，并且是正确的，即应用程序将恢复到之前的状态，并且系统保证不会发生资源泄漏。未来的扩展是持久化所有者的状态，以最小化恢复时间，但代价是增加恢复复杂性和运行时开销

> Ray 文档说目前不支持 owner recovery 但为什么论文这里说集中式设计容易解决呢

## Ownership Design

集群中的每个节点托管一个或多个 worker（通常每个核心一个），一个调度器和一个对象存储

![](https://s2.loli.net/2024/11/12/IJc29VWs8g6RGbt.png)

worker 使用一个所有权表来记录它所处理的任务和数据

The distributed memory layer (Section 4.2) consists of an immutable distributed object store (Figure 7d) with Locations stored at the owner

> 具体细节看论文

### Task scheduling

Owner 如何协调任务调度

在较高层次上，**所有者**将每个任务分派到**分布式调度器选择的位置**。这确保了所有权表中的任务位置与分派同步更新。我们假设一个抽象的调度策略，该策略接收资源请求并返回应分配资源节点的 ID。该策略还可能更新其决策，例如由于资源可用性的变化。

![](https://s2.loli.net/2024/11/12/wHitG2CMXj8ZKse.png)

图 8c 展示了分派任务的协议。在任务调用时，**调用者，即返回的 DFut 的所有者**，首先从其本地调度器请求资源。请求是一个元组，包含任务所需的资源（例如，{"CPU": 1}）和分布式内存中的参数。如果策略选择本地节点，**调度器接受请求**：它获取参数，分配资源，然后将**本地 worker 租给所有者**。否则，调度器拒绝请求并将所有者重定向到策略选择的节点。

在这两种情况下，**调度器都会向所有者返回新位置**：要么是租用的 worker 的 ID，要么是另一个节点的 ID。所有者在将任务分派到该位置之前，将其存储在其本地所有权表中。如果请求被接受，所有者直接将任务发送给租用的 worker 执行；否则，它在**下一个调度器重复该协议**。

因此，所有者总是将任务分派到其下一个位置，确保任务的挂起位置（表 2）同步更新。这也允许所有者在任务的资源需求得到满足时，通过直接将任务分派给已租用的 worker 来绕过调度器。例如，在图 8d 中，worker 1 重用了从节点 2 租用的资源来执行 C。所有者在配置的超时时间后或没有更多任务需要分派时返回租约。我们目前不重用具有不同分布式内存依赖关系的任务的资源，因为这些资源由调度器获取。我们将其他**租约撤销**和**工作重用**的策略留待未来的工作。

> 工作重用到底是什么意思

任务执行前的最坏情况 RTT 数量高于之前的解决方案，因为每个策略决策都返回给所有者（图 8e）。然而，之前解决方案的吞吐量受到限制（图 12），**因为它们无法支持直接的 worker-to-worker 调度**（图 8d）。这是因为 worker 不存储系统状态，因此所有任务必须通过主节点或每个节点的调度器来更新任务位置（图 8a 和 8b）。

> 这里很奇怪，说吞吐量是由于 worker2worker 调度限制了，那对于 lease 呢？

Actor scheduling 系统调度 actor 构造任务的方式与普通任务类似。然而，在完成之后，所有者持有 worker 的租约，直到 actor 不再被引用（第 4.2 节），并且 worker 只能执行通过相应 ARef 提交的 actor 任务。

### Memory management

Allocation 对于小对象，复制可能比通过分布式内存传递更快，后者需要更新对象目录，从远程节点获取对象等。因此，在对象创建时，系统根据大小透明地选择是按值传递还是按引用传递。

Dereferencing

Reclamation

Actors

### Failure recovery

Failure detection. 故障通知包含 worker 或节点 ID，发布给所有 worker。Worker 不交换心跳；worker 故障由其本地调度器发布。节点故障通过节点之间交换心跳来检测，所有 worker 与其节点 fate-share。

在接收到节点或 worker 故障通知后，每个 worker 扫描其本地所有权表以检测 DFut 故障。DFut 在两种情况下被认为是失败的：1）通过比较 Location 字段，丢失拥有的对象（图 10a），或 2）通过比较 Owner 字段，丢失所有者（图 11a）。我们接下来分别使用行踪重建和命运共享来讨论这两种情况的处理

请注意，非所有者不需要检测对象的丢失。例如，在图 10a 中，节点 2 在 worker 3 接收到 C 时失败。当 worker 3 在所有者处查找 X 时，可能找不到任何位置。从 worker 3 的角度来看，这意味着节点 2 的目录写入被延迟，或者节点 2 失败。Worker 3 不需要决定是哪一种；**它只需等待 X 的所有者处理故障。**

Object recovery: The owner recovers a lost value through lineage reconstruction

> ---

Owner recovery. 所有者故障可能导致“dangling pointer”：无法解引用的 DFut。如果对象同时从分布式内存中丢失，就会发生这种情况。例如，如果节点 2 也失败，图 11a 中的 C 将会挂起。

我们使用命运共享来确保系统在所有者失败时能够继续运行。首先，所有者及其任何引用持有者持有的所有资源都被回收。具体来说，在接收到所有者失败的通知后，分布式对象存储释放对象（如果存在）或调度层回收 worker 租约（如果对象正在等待），如图 11b 所示。所有引用持有者，即借用者和依赖任务，也与所有者共享命运

然后，为了恢复共享命运的状态，我们依赖于行踪重建。特别是，在失败的所有者上执行的任务或 actor 本身必须由另一个进程拥有。该进程最终将重新提交失败的任务。当新的所有者重新执行时，它将重新创建其先前的状态，无需系统干预。例如，图 11a 中 A 的所有者最终将重新提交 A（图 11b），这将再次提交 B 和 C。

为了正确性，我们证明所有先前的引用持有者都被重新创建，并带有新所有者的地址。考虑计算 DFut x 值的任务 T。T 最初在 worker W 上执行，并在恢复期间在 W0 上重新执行。API（第 2 节）提供了三种创建对 x 的另一个引用的方法：（1）将 x 作为任务参数传递，（2）将 x 转换为 SharedDFut 然后作为任务参数传递，（3）从 T 返回 x。

## Evaluation

我们与三个基线进行了比较：（1）一个没有分布式内存的未来但按值传递的模型，类似于图 2c，（2）一个基于租约的去中心化系统用于分布式未来（Ray v0.7），（3）一个集中式主节点用于分布式未来（Ray v0.7 修改为在任务执行前写入集中式主节点）。

Task submission is divided across multiple intermediate drivers, either colocated on the m5.8xlarge
head node or spread with one m5.8xlarge node per driver

### Microbenchmarks

Throughput and scalability.

We could not produce stable results for pass-by-value with large objects due to the **lack of backpressure** in our implementation

当驱动程序分散时（图 12b 和 12d），所有权和租约都线性扩展。在图 12b 中，所有权比租约扩展得更好，因为更多的工作被卸载到 worker 进程上。在图 12d 中，所有权和租约实现了类似的吞吐量，但**所有权系统还包括内存安全**（第 4.2 节）。集中式设计（2 个分片）线性扩展到 ∼60 个节点。添加更多分片会提高这个阈值，但只是常数数量。

> 是不是有点假这里的 small object 测试，pass by value 肯定不需要分布式内存，ownership 又调优小对象，另两个呢？
>
> 大对象 colocated 为什么 ownership 好一点
>
> spread 多个 driver 其实应该都差不多我觉得，因为分散了任务都是并行？这里和 lease 性能其实很接近

Scaling through borrowing

Latency 该 worker 要么与驱动程序位于同一节点（“本地”），要么位于单独的 m5.16xlarge 节点上（“远程”）
![](https://s2.loli.net/2024/11/12/jTdsJrtk6XgGSFH.png)

首先，分布式内存在所有情况下都比按值传递实现了更好的延迟，因为这些系统避免了从驱动程序到 worker 的**不必要任务参数副本。**

其次，与集中式和租约相比，所有权平均实现了 1.6 倍的更低延迟。这是因为（1）能够在所有者本地写入元数据，而不是远程进程，以及（2）能够**重用租用的资源**，在许多情况下**绕过调度层**（第 4.1 节）。

> 如果任务的资源需求得到满足，所有者还可以绕过调度器，直接将任务分派给已租用的工人。 in Figure 8d, worker 1 reuses the resources leased from node 2 in Figure 8c to execute C.

Recovery

### End-to-end applications

Model Serving
所有权和集中式实现了相同的中位数延迟（54ms），但集中式的尾部延迟高出 9 倍（1 秒 vs. 108ms）

Online video processing.
图 15b 显示了没有故障的延迟。所有系统实现了相似的中位数延迟（∼65ms），但租约和集中式的尾部延迟较长（分别为 1208ms 和 1923ms）。

基于租约的恢复很慢，因为解码器 actor 必须重放所有任务，并且每个任务从租约到期中累积开销。由于租约实现**无法安全地垃圾回收血统**，因此检查点 actor 是不可行的。

> 例如，当资源的所有权需要转移或释放时，租约机制可能无法可靠地追踪这些变化，导致未使用的资源无法被及时回收。
>
> etcd 租约撤销
>
> 分布式垃圾回收 DGC

application-level checkpoints (O+CP)

## Conclusion

> 一些问题
>
> 1. median latency 都差不多，但是 ownership 尾部延迟很好，差别在哪？是因为高负载出现了瓶颈？ O+CP(Redis) 是不是没有控制变量啊。。O+CP; WF 呢？
>
> 2. Ray does not support recovery from owner failure. 你觉得是为什么？因为 metadata 丢失了，计数也丢失了，所以也是个瓶颈
>    fate share 是不是难以实现
>
> 3. Spark / Ray / MapReduce by default is using centralized architecture, master/ head node
>
> Ray < v0.8 is using lease manager?
>
> Ownership 假设任务都是 DAG / Tree
> ObjectRef 是有 scope 的可以传递
>
> 4. 我觉得 Ownership owner recovery 太假了，所有 owner metadata 都丢了
>    w3 返回到一个失败节点？然后找到 owner 的 owner 来重建？这不是最坏的情况吗
>
>    那么有没有更好的办法呢，比方说不是重建所有，而是同步这个 DAG 里所有的 ownership 表，只重建那一个？

> https://docs.google.com/document/d/1tBw9A4j62ruI5omIJbMxly-la5w4q_TjyJgJL_jN2fI/preview?tab=t.0
>
> Some of the trade-offs that come with ownership are:
>
> 1. To resolve an `ObjectRef`, the object’s owner must be reachable. This means that an object will fate-share with its owner. See Object failures and Object spilling for more information about object recovery and persistence.
> 2. Ownership currently cannot be transferred.

1. (Individual) The first question is: What do you think of Ownership's design? Do you buy it? Or which one do you prefer, centralized head node, distributed lease, or Ownership, why?

> Personally, I prefer distributed lease design because it's simpler and widely used. For example, Kubernetes uses leases and implements resource management and error tolerance based on leases.
>
> Previous versions of Ray utilized centralized head node design and combined with leases. However, since it is difficult to handle garbage collection under lease, and ownership provides better throughput and tail latency because it facilitates worker-to-worker task scheduling and worker reuse. I prefer the Ownership model.

2. (Group Discussion) The purpose of Ownership is to support fault tolerance with distributed memory and distributed future, while also improving throughput and latency.
   Refer to some memory management papers we read before, software-defined far memory, AIFM, fastswap.
   Do you think that ownership can be application-aware or resources-aware?
   like utilizing some data structure from AIFM to allow the owner to release some cold data in advance, so we could reuse the worker?
   What will the potential problems be?

3. (Group Discussion, optional if time permits) We have learned that many distributed systems utilize logs to achieve fault tolerance. For instance, Delta Lake employs Write-Ahead Logs to support consistency and ACID properties.
   The paper mentions some future work in section 7, it says that it might support a spectrum of application recovery requirements. For example, we could extend ownership with options to recover actor state. Do you think logs or global object store can be used to implement future like this or even time travel? What metadata do we need?

4. (Whiteboard Question) According to Ray's documentation, Ray does not support Owner recovery and does not support ownership transfer even though the latest version is build on top of Ownership.

   What do you think is the reason? Personally, I believe it is because losing all ownership record makes it difficult to rebuild lineage reconstruction. If you were to design it, do you think it could be solved through ownership transfer?

   For example, use the head node design and adding the feature of ownership transfer, when the owner fails, let the head node restart the owner node and transfer the ownership copy from a certain worker node to the owner node, making the node the owner again. There might be some consistency issues here, and this is an open question, you can present your solution as well as what you think might be the issue?
