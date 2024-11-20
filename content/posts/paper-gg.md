+++
title = 'Paper Reading: From Laptop to Lambda: Outsourcing Everyday Jobs to Thousands of Transient Functional Containers'
date = 2024-11-20T01:09:31-05:00
draft = false
tags = ['Paper Reading']
+++

## From Laptop to Lambda: Outsourcing Everyday Jobs to Thousands of Transient Functional Containers

同样是 standford DBOS 组关于 serverless 的论文，作者还有一篇 R2E2 很厉害，用 serverless 做 ray tracing 实现很高的加速，但 cost 估计控制不住

## Abstract

gg framework

users might push a button that spawns 10,000 parallel cloud functions to execute a large job in a few seconds from start.

通过 gg，应用程序将任务表示为轻量级操作系统容器的组合，这些容器是单独瞬态的（生命周期为 1-60 秒）和功能性的（每个容器都是密封且确定性的）。gg 负责在云函数上实例化这些容器，加载依赖项，最小化数据移动，在容器之间移动数据，并处理故障和滞后者。

我们移植了几个对延迟敏感的应用程序以在 gg 上运行，并评估了其性能。在最佳情况下，基于 gg 构建的分布式编译器比传统工具（icecc）快 2-5 倍，而无需持续运行的 warm 集群。在最差情况下，gg 的性能与现有视频编码工具（ExCamera）的手动调优性能相差 20% 以内。

## Introduction

云函数，也称为无服务器计算。Amazon 的 Lambda 服务将以每 100 毫秒的最小间隔出租一个 Linux 容器来运行任意的 x86-64 可执行文件，启动时间不到一秒，空闲时不收费。

云函数原本是为异步调用的微服务设计的，但其粒度和规模使得研究人员探索了一种不同的用途：作为按需的突发超级计算机。

最近的工作验证了这一愿景。ExCamera [15] 和 Sprocket [3] 启动了数千个云函数，通过 TCP 进行线程间通信，以快速编码、搜索和转换视频文件。PyWren [23] 提供了一个 Python API，并使用 AWS Lambda 函数进行线性代数和机器学习。Serverless MapReduce [35] 和 Spark-on-Lambda [36] 展示了类似的方法。

不幸的是，在云函数的集群上构建应用程序是困难的。每个应用程序都必须克服这个环境中固有的一系列挑战：
（1）工作者是无状态的，可能需要在启动时下载大量代码和数据，
（2）工作者在运行时有限，之后会被终止，
（3）工作者上的存储是有限的，但比外部存储快得多，
（4）可用云工作者的数量取决于提供商的整体负载，无法提前精确知道，
（5）在大规模运行时会发生工作者故障，
（6）云函数与本地机器上的库和依赖项不同，
（7）云的延迟使得往返成本高昂。过去的应用程序只以特定的方式解决了这些挑战的一部分。

gg 与其他并行执行系统。在目标和方法上，gg 与容器编排系统如 Kubernetes [5] 和 Docker Swarm [10]、外包工具如 Utility Coprocessor [12] 和 icecc [20]、以及集群计算工具如 Hadoop [38]、Dryad [22]、Spark [40] 和 CIEL [27] 有相似之处。

但 gg 也与这些系统在关注点上有所不同，它专注于新的计算基底（云函数）、执行模式（从零开始的突发并行、对延迟敏感的程序）和目标应用领域（日常的“本地”程序，例如软件编译，依赖于从用户自己的笔记本电脑捕获的环境）。

例如，云函数的“无状态”特性（它们启动时没有任何可靠的瞬态状态）使得 gg 特别关注高效的容器化和依赖管理：在启动时将正确文件的最小集合加载到每个容器中。

gg 在启动速度上比 Google Kubernetes Engine 快 45 倍，比 Spark-on-Lambda 快 13 倍（图 7）。

### Summary of Results

我们将四个应用程序移植到 gg 的格式中，以表达它们的工作：每个容器的描述以及它如何依赖于其他容器，我们称之为 intermediate representation (IR)（§3）。其中一个应用程序通过从现有的软件构建系统（例如 make 或 ninja）推断 IR 来自动完成此操作。其余的应用程序则明确地写出描述：一个单元测试框架（Google Test [17]）、带有线程间通信的并行视频编码（ExCamera [15]），以及使用 Scanner [30]和 TensorFlow [1]的对象识别。

> 这里的 IR 是 LLVM 类似的概念吗

然后，我们为五个计算引擎（本地机器、a cluster of warm VMs、AWS Lambda、IBM Cloud Functions 和 Google Cloud Functions）和三个存储引擎（S3、Google Cloud Storage 和 Redis）实现了 gg 的后端，这些后端解释 IR 并执行任务

对于从冷启动编译大型程序，gg 的功能性方法和细粒度的依赖管理带来了显著的性能提升。

总之，gg 是一个实用的工具，解决了突发并行云函数应用程序面临的主要挑战。它帮助开发者和用户构建应用程序，从零突发到数千个并行线程，以实现日常任务的低延迟。gg 是开源软件，源代码在 https://snr.stanford.edu/gg

## Related Work

gg 有许多前身——集群计算系统如 Hadoop [38]、Spark [40]、Dryad [22]和 CIEL [27]；容器编排系统如 Docker Swarm 和 Kubernetes；外包工具如 distcc [8]、icecc [20]和 UCop [12]；基于规则的工作流系统如 make [13]、CMake [7]和 Bazel [4]；以及云函数工具如 ExCamera/mu [15]、PyWren [23]和 Spark-on-Lambda [36]。

Process migration and outsourcing: gg 从冷启动开始编排数千个不可靠和无状态的云函数

Container orchestration: gg 的 IR 类似于容器和环境描述语言，包括 Docker [10]和 Vagrant [34]，以及容器编排系统如 Docker Swarm 和 Kubernetes。与这些系统相比，gg 的 thunk 设计为在云函数中高效实例化，

> thunk 到底是什么，延迟计算？ haskeel 里的惰性求值？

Workflow systems.

- gg is aimed at a different kind of application
- gg uses OS abstractions
- gg is considerably lighter weight.
- gg supports dynamic data access

> 为什么说 spark 不能用于编译？
>
> 动态数据访问也不太理解，这篇论文前半部分有点莫名其妙的，讲一大堆 related work
>
> gg 支持非 DAG 数据流？？怎么实现的

Burst-parallel cloud functions: gg 的主要贡献是指定一个 IR，允许各种应用程序（用任何编程语言编写）从计算和存储平台中抽象出来，并利用常见的服务进行依赖管理、滞后者缓解和调度。

Build tools: 几个构建系统（例如 make [13]、Bazel [4]、Nix [11]和 Vesta [19]）和外包工具（如 distcc [8]、icecc [20]和 mrcc [26]）寻求增量化、并行化或将编译分布到更强大的远程机器上。基于这些系统，gg 自动将现有的构建过程转换为其自己的 IR。目标是快速编译程序——不考虑软件自身的构建系统——通过利用可以从完全休眠状态突发到数千倍并行性并返回的云函数平台。

现有的远程编译系统 remote compilation systems，包括 distcc 和 icecc，在构建过程中频繁地在主节点和工作节点之间传输数据。这些系统在局域网上表现最佳，**并且在云中更远程的服务器上构建时会增加大量延迟**。相比之下，gg 一次性上传所有构建输入，并在云中纯粹执行和交换数据，减少了网络延迟的影响。

> gg 的卖点好像注重分布式编译

## Design and Implementation

gg 被设计为一个通用系统，帮助应用程序开发者管理创建突发并行云函数应用程序的挑战。预期用户将把通常在本地或小型集群上长时间运行的计算（例如测试套件、机器学习、数据探索和分析、软件编译、视频编码和处理）外包给云中数千个短暂的并行线程，以实现接近交互式的完成时间。

### gg’s Intermediate Representation

gg 使用的格式——**一组描述容器及其对其他容器依赖关系的文档**——旨在从应用程序中提取足够的信息（细粒度的依赖关系和数据流），以便能够在受限和无状态的云函数上高效执行任务。它包括：

1. 内容寻址 cloud thunk 的原语：应用于命名输入数据的代码片段或可执行文件。

2. 中间表示（IR），将任务表示为**相互依赖**的 thunk 的**惰性求值** lambda 表达式。

3. 一种策略，用于以语言无关和可记忆化的方式表示动态计算图和数据访问模式，使用尾递归。

我们讨论这些元素中的每一个。

### Thunk: A Lightweight Container

在函数式编程文献中，thunk 是一个无参数的闭包（函数），它捕获其参数和环境的快照以供以后求值。求值 thunk 的过程——将函数应用于其参数并保存结果——称为强制求值[2]。

对于 gg，我们的目标是简化新应用程序的创建,允许它们针对 IR，从而利用后端引擎提供的常见服务。因此，thunk 的表示遵循几个设计目标。它应该是：（1）足够简单，以便可以移植到不同的计算和存储平台，（2）足够通用，以表达各种合理的应用程序，（3）与实现函数的编程语言无关，（4）足够高效，以捕获可以在无状态和空间受限的云函数上具体化的细粒度依赖关系，（5）能够记忆化以防止重复工作。

> 怎么记忆化？

在内容寻址方案中，对象的名称有四个组件：（1）对象是原始值（哈希以 V 开头）还是表示强制其他 thunk 的结果（哈希以 T 开头），（2）SHA-256 哈希，（3）字节长度，（4）可选标签，命名对象或 thunk 的输出。

强制 thunk 意味着实例化描述的容器并运行代码。为此，执行器必须获取代码和数据值。因为这些是内容寻址的，所以可以从任何能够生成具有正确名称的 blob 的机制中获取——持久或短暂的存储（例如 S3、Redis 或 Bigtable）、从另一个节点的网络传输，或者通过在先前执行中已经可用的 RAM 中找到对象。然后，执行器使用提供的参数和环境运行可执行文件——出于调试或安全目的，最好是在防止可执行文件访问网络或未列为依赖项的任何数据的模式下运行。执行器收集输出 blob，计算其哈希值，并记录输出可以替换对刚刚强制的 thunk 的任何引用。

### gg IR: A Lazily Evaluated Lambda Expression

![](https://s2.loli.net/2024/11/21/Ms3fkG5YDnjiyLq.png)

> Figure 3 使用哈希来追踪 thunk 依赖？

相互依赖的 thunk 的结构定义了 gg 的 IR，我们使用单向 IR，一种应用程序用来表达其任务的文档格式，而不是双向 API（例如，调用函数来生成新任务并观察其结果），因为我们期望应用程序将在用户的计算机上运行，而在某个远程云函数引擎上执行：目的是通过将应用程序排除在外来避免通过高延迟路径的往返。我们还设想，如果应用程序在执行开始之前被排除在外，将更容易更好地调度和优化任务，并更容易维护不同的可互操作的后端。2 这种表示形式向后台暴露了计算图，以及需要在 thunk 之间通信的对象的身份和大小。基于这些信息，后台可以调度 thunk 的强制执行，将具有相似数据依赖性或输出-输入关系的 thunk 放置在相同的物理基础设施上，并管理中间结果的存储或传输，而无需往返用户的计算机。

IR 允许 gg 高效地调度任务，**通过在关键路径上调用多个并发 thunk 来缓解滞后者的影响**，通过第二次强制 thunk 来恢复故障，并记忆化 thunk。这是以应用程序无关、语言无关的方式实现的。

应用程序通常从强制表示交互操作最终结果的单个 thunk 开始。这个 thunk 通常依赖于其他需要首先强制的 thunk，等等，导致后台递归地惰性强制 thunk，直到获得最终结果。图 3 显示了一个计算表达式 ASSEMBLE(COMPILE(PREPROCESS(hello.c)))的示例 IR。

### Tail Recursion: Supporting Dynamic Execution

上述设计足以描述在云中执行的确定性任务的有向无环图（DAG）。然而，许多任务的数据访问模式并不是完全预先已知的。例如，在编译软件时，无法事先知道某个阶段需要读取哪些头文件和库。其他应用程序使用循环、递归和其他非 DAG 数据流。

相反，gg 通过语言无关的尾递归来处理这种情况：一个 thunk 可以将其输出写为另一个 thunk。

### Front-ends

我们开发了四个前端，它们生成 gg IR：一个 C++ SDK、一个 Python SDK、一组命令行工具和一系列模型替换原语，可以从软件构建系统推断 gg IR。

### Back-ends

gg IR 表达应用程序针对一个抽象机器，该机器需要两个组件：一个用于强制单个 thunk 的执行引擎，以及一个用于**存储 thunk 引用**或生成的命名 blob 的内容寻址存储引擎。协调程序将这两个组件结合在一起。

Storage engine 存储引擎提供了一个简单的接口，用于内容寻址存储，包括 GET 和 PUT 函数来检索和存储对象。

Execution engine 与存储引擎结合，每个执行引擎实现了一个简单的抽象：一个接收 thunk 作为输入并返回其输出对象哈希（可以是值或 thunk）的函数。

The coordinator 执行 thunk 的主要入口点是协调程序。该程序的输入是目标 thunk、可用执行引擎列表和存储引擎。该程序实现了 gg 提供的服务，如作业调度、记忆化、故障恢复和滞后者缓

启动时，该程序具体化目标 thunk 的依赖图，其中包括获取输出所需的所有其他 thunk。然后，根据其可用容量，将准备好执行的 thunk 传递给执行引擎。当 thunk 的执行完成后，程序通过替换对刚刚强制的 thunk 的引用来更新图，并添加一个缓存条目，将输出哈希与输入哈希关联。准备好执行的 thunk 被放置在队列中，并在其容量允许时传递给执行引擎。统一的接口允许用户混合和匹配不同的执行引擎，只要它们共享相同的存储引擎。

调用、执行和放置的细节留给执行引擎。例如，AWS Lambda/S3 的默认引擎为每个 thunk 调用一个新的 Lambda。Lambda 从 S3 下载所有依赖项并设置环境，执行 thunk，将输出上传回 S3 并关闭。对于具有大输入/输出对象的应用程序，往返 S3 可能会影响性能。作为此类情况的优化，用户可以选择以“长期运行”模式运行执行引擎，其中每个 Lambda 工作者保持活动状态，直到作业完成并寻找新的 thunk 来执行。执行引擎维护每个工作者本地存储中已存在的所有对象的索引。在将 thunk 放置在工作者上时，它选择具有最多可用数据的工作者，以最小化从存储后端获取依赖项的需求。

协调程序还可以对依赖图应用优化。例如，多个 thunk 可以捆绑为一个并发送给执行引擎。当一个 thunk 的输出将被下一个 thunk 消耗时，这很有用，创建了一个线性工作管道。通过在一个工作者上调度所有这些 thunk，系统减少了往返次数。

Failure recovery and straggler mitigation. 在协调程序失败的情况下，作业可以从上次中断的地方继续，因为协调程序使用磁盘缓存条目来避免重复已经完成的工作

![](https://s2.loli.net/2024/11/21/6yjuLYNJ2gc4EUd.png)

Straggler mitigation 滞后者缓解是协调程序管理的另一项服务，它会在相同或不同的执行引擎中复制挂起的执行。程序跟踪每个 thunk 的执行时间，如果执行时间超过超时（由用户或应用程序开发者设置），作业将被复制。由于函数没有任何副作用，协调程序只需选择首先准备好的输出。

> 麻了随便看看了这论文，讲解的不是很清晰

### Implementation Notes

## Applications

We used gg to implement several applications, each emitting
jobs in the gg IR. We describe these in turn

### Software Compilation

软件编译所需的时间一直是软件开发者的持续烦恼；甚至有一幅流行的卡通讽刺了这个任务的持续时间[39]。如今的开源应用程序已经变得越来越大。例如，从冷启动开始，Chromium Web 浏览器在四核笔记本电脑上编译需要四个多小时。已经开发了许多解决方案来利用本地集群或云数据中心的温暖机器（例如 distcc 或 icecc）。我们在 gg 之上开发了这样一个应用程序。

使用模型替换，我们为 C 或 C++软件构建管道的七个流行阶段实现了模型：预处理器、编译器、汇编器、链接器、归档器、索引器和 strip。这些模型允许我们自动将一些软件构建过程（例如 Makefile 或 build.ninja 文件）转换为 gg IR 中的表达式，然后可以在云函数平台上以数千倍的并行性执行，以获得与本地执行构建系统相同的结果。图 4 展示了一个示例调用生成的 IR（枚举的 thunk 在图 3 中详细说明）。这些模型足以捕获一些主要开源应用程序的构建过程，包括 OpenSSH [29]、Python 解释器 [32]、Protobuf 库 [31]、FFmpeg 视频系统 [14]、GIMP 图像编辑器 [16]、Inkscape 矢量图形编辑器 [21]和 Chromium 浏览器 [6]

Capturing dependencies of the preprocessor.

为了解决这个问题，应用程序使用 gg 在运行时动态数据流的能力。gg 的预处理器模型生成 thunk，这些 thunk 在云函数上并行进行依赖项推断。这些 thunk 只能访问用户包含目录的简化版本，仅保留带有 C 预处理器指令（如#include 和#define）的行。然后，这些 thunk 生成进一步的 thunk，通过仅列出必要的头文件来预处理给定的源代码文件。

### Unit Testing

通常，每个测试都是一个独立的程序，可以与其他测试并行运行，它们之间没有依赖关系。代码不需要更改，但有一个例外：如果测试用例需要访问文件系统上的文件，那么程序员必须在测试用例中注释出它想要访问的文件列表。这个过程可以通过在本地运行测试，然后跟踪每个测试用例调用的打开系统调用来实现自动化。该工具使用这些注释，无论是手工编写的还是自动生成的，来捕获每个测试的依赖项。为每个测试用例创建一个单独的 thunk，允许执行引擎利用可用的并行性。

### Video Encoding

### Object Recognition

### Recursive Fibonacci

## Evaluation

### Startup Overhead

为了说明 gg 轻量级抽象的重要性，我们使用四个框架实现了一个简单的任务：1,000 个并行任务，每个任务执行 sleep(2)，使用的框架是 gg、PyWren、Spark-on-Lambda 和 Kubernetes。前三个框架在 AWS Lambda 上执行，最后一个在 Google Kubernetes Engine (GKE)上执行，GKE 提供了一个温暖的集群，包含十一个 96 核虚拟机（总共 1,056 个核心），用于分配容器。图 7 显示了结果。

gg 能够快速扩展到 1,000 个并发容器，并且与其他两个在 Lambda 上运行的框架相比，完成任务的速度快 7.5-9 倍。减去 2 秒的睡眠时间后，这相当于减少了 11-13 倍的额外开销。对于 PyWren，平均每个工作者花费 70%的时间设置 Python 运行时（下载和解压 tarball）。这个运行时的大部分由我们的 sleep(2)程序未使用的包组成（参见 gg 的细粒度依赖跟踪）。Google Kubernetes Engine 不是为瞬态计算设计的，也没有针对这种用例进行优化；启动 1,000 个 Docker 容器要慢得多。

### Software Compilation

为了基准测试 gg 的性能，我们选择了四个用 C 或 C++编写的开源程序：FFmpeg、GIMP、Inkscape 和 Chromium。这些包的代码或底层构建系统没有进行任何更改。我们使用 GCC 7.2 编译所有包

图 9 显示了**包构建的中位时间**。gg 在构建中等和大型软件包时比传统工具（icecc）快约 2-5 倍。例如，gg 在 AWS Lambda 上编译 Inkscape 仅需 87 秒，而使用 icecc 外包到温暖的 384 核集群需要 7 分钟。这是 4.8 倍的加速。Chromium 是最大的可用开源项目之一，使用 gg 在 AWS Lambda 上编译仅需不到 20 分钟，比 icecc (384)快 2.2 倍。

我们认为 gg 在 AWS Lambda 上的性能提升不能简单地归因于比我们的 384 核集群更多的核心；icecc 在 48 核和 384 核情况下的改进只是适度，并且似乎没有有效利用更高的并行度。这主要是因为 icecc 为了简化依赖跟踪，在本地运行预处理器，这成为了一个主要瓶颈。gg 的细粒度依赖跟踪允许系统将这一步骤高效外包到云端，并最小化本地机器上的工作。

图 10 显示了编译 Inkscape 的执行分解。我们观察到两个重要特征。首先，大的峰值对应于失败的或比平时完成时间更长的 Lambda。gg 的滞后者缓解检测并重新启动这些任务，以防止端到端延迟增加。其次，最后几个任务主要是串行的（归档和链接），消耗了近四分之一的总任务完成时间。这些特征在其他构建任务中也有所体现。

> 真的莫名其妙的论文，看不懂

## Limitations and Discussion

仅限于 CPU 程序

gg 将代码格式指定为 x86-64 Linux ELF 可执行文件。IR 没有机制来表示对 GPU 或其他加速器的需求，并且高效调度这些资源提出了非平凡的挑战，因为从 GPU 加载和卸载配置状态比内存映射文件更昂贵的操作。我们计划研究 gg 后端将 thunk 调度到 GPU 的适当机制。

> 这是绝大部分 serverless 的问题吧

## Conclusion

在本文中，我们描述了 gg，一个帮助开发者构建和执行突发并行应用程序的框架。gg 提供了一个可移植的抽象：一个中间表示（IR），将任务的未来执行捕获为轻量级 Linux 容器的组合。这使得 gg 能够支持新旧应用程序，这些应用程序在各种语言中从计算和存储平台以及解决底层挑战的运行时特性中抽象出来：依赖管理、滞后者缓解、放置和记忆化。

![](https://s2.loli.net/2024/11/21/qDYf8621ElWrQLZ.png)
