+++
title = 'Paper Reading: Caladan'
date = 2024-10-09T10:35:26-04:00
draft = false
tags = ['Paper Reading']
+++

## Caladan: Mitigating Interference at Microsecond Timescales

Shenango 同一批人做的，在其基础上提出了性能更好、吞吐量更大的 Caladan，将 IOKernel 移出了 datapath，使用的是 Dirctpath

## Abstract

传统观点认为，CPU 资源如核心、缓存和内存带宽必须进行分区，以实现任务之间的性能隔离

> 什么是 performance isolation
>
> 多任务或多用户环境中，确保一个任务或用户的性能不受其他任务或用户的影响

在本文中，我们证明资源分区 resource partitioning 既不是必要的也不是充分的。

Caladan 是一种新的 **CPU 调度器**，通过一组依赖于快速核心分配而非资源分区的控制信号和策略，可以显著提高**quality of service (tail latency, throughput, etc.)**。Caladan 包括一个**集中式调度核心**，主动管理内存层次结构和超线程之间的资源争用，以及一个内核模块，该模块**绕过标准 Linux 内核调度器**，以支持微秒级的任务监控和放置。当将 memcached 与一个尽力而为的垃圾收集工作负载共存时，Caladan 的表现优于最先进的资源分区系统 Parties，性能提升了 11,000 倍，在资源使用变化期间将尾部延迟从 580 毫秒降低到 52 微秒，同时保持高 CPU 利用率。

> 绕过 Linux Kernel scheduler，和 eBPF 实现调度有什么区别呢
>
> 这性能提升太夸张了，到底是 baseline 选的好还是什么

## Introduction

像网络搜索、社交网络和在线零售这样的交互式、数据密集型网络服务通常会将请求分布在数千台服务器上。最小化尾部延迟对这些服务至关重要，

然而，减少尾部延迟的努力必须与最大化数据中心效率的需求仔细平衡；

大规模数据中心运营商通常 pack several tasks 在同一台机器上，以在可变负载的情况下提高 CPU 利用率[22, 57, 66, 71]。在这些条件下，任务必须竞争共享资源，如核心、内存带宽、缓存和执行单元。当共享资源争用高时，延迟会显著增加；这种由于资源争用导致的任务减速称为干扰 interference。

> 争用高会导致延迟好理解，也就是 CPU 利用率高，为什么会导致尾部延迟呢，感觉不是直接原因

我们的目标是保持高 CPU 利用率和严格的性能隔离（对于吞吐量和尾部延迟），在资源使用频繁变化的真实条件下，从而干扰也频繁变化。

实现微秒级反应时间有两个挑战。首先，共享 CPU 中存在多种干扰（超线程、内存带宽、LLC 等），在微秒级时间尺度上准确检测每种干扰的正确控制信号是困难的。其次，现有系统在收集控制信号或快速调整资源分配方面面临过多的软件开销。

> 资源监控应该可以用 eBPF 做？

为了克服这些挑战，我们提出了一种 interferenceaware CPU scheduler, Caladan。Caladan 由一个集中式的专用调度核心组成，该核心收集控制信号并做出资源分配决策，以及一个名为 `KSCHED` 的 Linux 内核模块，该模块高效地调整资源分配。我们的调度核心区分 high-priority, latency-critical (LC) tasks and low-priority, besteffort (BE) tasks。为了避免硬件分区带来的反应时间限制（§3），Caladan 完全依赖核心分配来管理干扰。

Caladan 使用精心挑选的一组 control signals 和相应的动作，以快速准确地检测和响应微秒级时间尺度上的干扰。我们观察到干扰有两个相互关联的影响：首先，干扰会减慢核心的执行速度（更多的缓存未命中、更高的内存延迟等），影响请求的服务时间；其次，随着核心速度减慢，计算能力下降；当它低于提供的负载时，排队延迟会急剧增加。

Caladan 的调度器针对这些影响进行了优化。它收集内存带宽使用和请求处理时间的细粒度测量数据，分别用于检测内存带宽和超线程干扰。

KSCHED 内核模块加速了调度操作，如唤醒任务和收集干扰指标。它通过分摊发送中断的成本，将调度工作从调度核心 offloading 到任务的核心，并提供一个非阻塞 API，

> 说实话完全看不懂了。。不太好理解在做什么

据我们所知，Caladan 是第一个能够在频繁变化的干扰和负载下同时保持严格性能隔离和高 CPU 利用率的系统

> 前文说 resource partitioning is neither necessary nor sufficient 这里应该是说分区是不必要的，但是能保证性能隔离

为了实现这些好处，Caladan 对应用程序提出了两个新的要求：采用自定义的运行时系统进行调度和 LC 任务需要暴露其内部并发性（§8）。作为交换，Caladan 能够比最先进的资源分区系统 Parties（[12]）报告的典型速度快 500,000 倍地收敛到正确的资源配置。我们展示了这种加速在将 memcached 与依赖垃圾收集的 BE 任务共存时，带来了 11,000 倍的尾部延迟减少。此外，我们展示了 Caladan 具有高度的通用性，能够扩展到多个任务，并在共存多种工作负载（memcached、内存数据库、闪存存储服务、x264 视频编码器、垃圾收集器等）时保持相同的好处。

> 假设是：custom runtime system, LC tasks exposes their internal concurrency
>
> 低延迟任务需要暴露并行性？

## Motivation

干扰未被迅速缓解时性能如何下降

许多工作负载表现出阶段性行为，在亚秒级时间尺度上急剧改变它们使用的资源类型和数量。例如，压缩、编译、Spark 计算作业和垃圾收集器

为了更好地理解与时间变化干扰相关的挑战，我们考虑当我们将一个 LC 任务 memcached[43]与一个由于垃圾收集而表现出阶段性行为的 BE 工作负载共存时会发生什么。

在这个实验中，我们向 memcached 提供一个固定的负载，并在两个任务之间 statically partition cores

这个例子说明了固定核心分区是不够的，并且还表明了为了有效缓解干扰，核心重新分配的速度是必要的。

> 就是先将 CPU 固定给特定的任务，实现性能隔离，但是发生了干扰，无法应对突发负载，需要重新分配

## Background

共享 CPU 时可能发生的三种干扰形式：hyperthreading interference, memory bandwidth interference, and LLC interference.

超线程干扰通常在任务运行在兄弟核心时存在，因为 CPU 会划分某些物理核心资源（例如，微操作队列），但其严重程度取决于**共享资源**（L1/L2 缓存、预取器、执行单元、TLB 等）是否被争用。另一方面，内存带宽和 LLC 干扰的强度可能会有所不同，但会影响共享同一物理 CPU 的所有核心。随着内存带宽使用量的增加，由于干扰，内存访问延迟会逐渐增加，直到内存带宽饱和；**访问延迟随后会呈指数级增加**[62]。LLC 干扰由每个应用程序使用的缓存量决定：当需求超过容量时，LLC 未命中率会增加。

> LLC 干扰 (Last Level Cache Interference)
>
> LLC 干扰是指在多任务或多线程环境中，多个任务或线程共享最后一级缓存（LLC）时，由于缓存资源的争用而导致的性能下降。
>
> DDIO data direct IO 可以直接将网络数据推到 LLC

### Existing Approaches to Interference

最先进的系统如 Heracles[38]和 Parties[12]通过动态分区资源（如核心和 LLC 分区大小）来处理干扰

首先，这两个系统使用应用程序级别的尾部延迟测量来检测干扰，这些测量必须在数百毫秒内进行，以获得稳定的结果；Parties 的作者发现，较短的间隔会产生“噪声和不稳定的结果”

因此，Heracles 和 Parties 至少需要 50 倍于我们例子中 GC 周期的时间来适应干扰的变化。

除了收敛速度，现有系统还存在可扩展性限制。例如，典型的数据中心服务器必须同时处理多个 LC 和 BE 任务[66, 71]，但 Heracles 仅限于单个 LC 任务（和许多 BE 任务）

这些系统所依赖的硬件机制也施加了限制。例如，超线程缺乏对资源分区的控制，因此 Heracles 和 Parties 完全关闭了它们。

> 所以一方面是现有的处理干扰策略太慢，毫秒级别，资源分配慢，convergence speed 慢，扩展性也不行，有些仅限于单个 LC 任务，也依赖于硬件，比如不支持超线程。

### Limitations of Hardware Extensions

英特尔的 CAT（Cache Allocation Technology）技术可以将 LLC 分区，但分区配置的变化需要相当长的时间才能生效。

内存带宽分配（Memory Bandwidth Allocation，MBA）

> 太接近硬件设计了这一篇论文，我感觉只要了解一下 interference 的原理，和现有解决方案的不足，以及 caladan 怎么做的就差不多了

## Challenges and Approach

我们的总体目标是保持性能隔离的同时最大化 CPU 利用率。

管理干扰变化需要微秒级反应时间。

因此 Caladan 的方法是通过控制核心如何分配给任务来管理干扰。

之前的系统已经将核心调整作为其管理干扰策略的一部分[12, 28, 38, 70]，但 Caladan 是第一个完全依赖核心分配来管理多种干扰形式的系统。

two key challenges:

1. Sensitivity: 为了快速和有针对性的反应，Caladan 需要能够在**微秒内识别干扰**的存在及其来源（任务和争用资源）的控制信号。常用的性能指标如 CPI[71]或尾部延迟[12, 38]（以及硬件机制如 MBM 和 CMT）在短时间内太嘈杂，无法有用。像排队延迟[8, 42, 47, 68]这样的指标可以在微秒级时间尺度上测量，但无法识别干扰的来源，只能表明任务的性能正在下降。

2. Scalability: 现有系统严重依赖 Linux 内核来收集控制信号和调整资源分配（例如，使用 `sched_setaffinity()` 来调整核心分配）。不幸的是，Linux 在这些操作中增加了开销，并且在存在干扰以及核心和任务数量增加时，这些开销会增加。

我们通过仔细选择能够快速检测干扰的控制信号，并专门分配一个核心来监控这些信号并在干扰发生时采取行动来解决敏感性挑战。我们通过一个名为 `KSCHED` 的 Linux 内核模块来解决可扩展性挑战。我们将在下面更详细地描述这些内容。

> 什么信号？
>
> 专用核心调度，解决信号问题
>
> 内核扩展、绕过

### Caladan’s Approach

Caladan 专门分配一个核心，称为调度器，以持续轮询和收集一组控制信号，时间尺度为微秒级。

> 到底是什么信号啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊
>
> 看一半才讲也太夸张了这论文

对于超线程，我们假设当两个兄弟核心都处于活动状态时，干扰总是存在（因为某些物理核心资源被分区），并专注于减少对尾部延迟有影响的请求的干扰——即运行时间最长的请求。我们测量请求处理时间以识别这些请求。

于内存带宽，我们测量全局内存带宽使用情况以检测 DRAM 饱和，并测量每个核心的 **LLC 未命中率**以将使用情况归因于特定任务。对于像 LLC 这样的情况，我们无法直接测量或推断干扰，我们仍然可以测量干扰的一个关键副作用：由于计算能力减少而增加的**排队延迟**。通过专注于干扰驱动的控制信号，Caladan 可以在服务质量下降之前检测到问题。

> the longest running requests, global Memory Bandwidth Usage
>
> Per-Core LLC Miss Rates, queueing delay

最后，Caladan 引入了一个名为 KSCHED 的 Linux 内核模块。**KSCHED 在微秒级时间尺度上跨多个核心同时执行调度功能**，即使在存在干扰的情况下也是如此。KSCHED 通过三种主要技术实现这些目标：（1）它在 Caladan 管理的所有核心上运行，**并将调度工作从调度核心转移到运行任务的核心**；（2）它利用硬件对多播处理器间中断（IPIs）的支持，以分摊同时启动多个核心操作的成本；（3）它提供了一个完全异步的调度器接口，以便调度器可以在等待远程核心完成操作的同时发起操作并执行其他工作。

## Design

### Overview

![](https://s2.loli.net/2024/10/10/IxZmOSVl1wgdWFh.png)

图 2 展示了 Caladan 的关键组件及其之间的共享内存区域。Caladan 与 Shenango[47]共享一些架构和实现构建块：每个应用程序都与一个运行时系统链接，一个专用的调度核心（以 root 权限运行）忙碌轮询共享内存区域以收集控制信号并进行核心分配。这两个系统都设计为在正常 Linux 环境中互操作，可能管理可用核心的子集。

Shenango 使用排队延迟作为其唯一的控制信号来管理负载变化；

Caladan 使用多个控制信号来管理多种类

外，Shenango 的调度核心将网络处理与 CPU 调度结合在一起；Caladan 的调度核心**仅负责 CPU 调度**，消除了数据包处理瓶颈（§6）型的干扰以及负载变化。

Caladan 的运行时系统提供“绿色”线程和内核旁路 I/O，使用工作窃取来平衡负载，并在没有工作可窃取时让出核心。这使得管理干扰更容易，并能导出正确的每任务控制信号。

用户为每个任务配置保证核心和突发核心，任务被指定为 LC（延迟敏感型）或 BE（尽力而为型）。BE 任务以较低的优先级运行，仅在 LC 任务不需要时分配突发核心，并根据需要进行限制以管理干扰。

### The Caladan Scheduler

独立的控制器模块检测内存带宽和超线程干扰，分别对核心分配施加约束并在必要时撤销核心。

> 撤销？

调度器从三个来源收集控制信号。

首先，运行时系统提供请求处理时间和排队延迟的信息。

其次，DRAM 控制器提供全局内存带宽使用情况的信息。

第三，KSCHED 提供每个核心的 LLC 未命中率信息，并在任务自愿让出时通知调度器核心。

> 感觉就是加了这些指标
>
> 后面的就不看了，理解 Caladan 是什么，怎么做的（新指标、KSched）
>
> 主要是将一些 LC 和 BE 任务，不再需要资源分区
>
> 而是通过分配和调度，减少干扰，降低尾部延迟

## Implementation

## Evaluation

### Comparison to Other Systems

### Diverse Colocations

### Microbenchmarks

## Discussion

Caladan 要求应用程序使用其运行时系统，因为它依赖于运行时系统向调度器导出控制信号，并快速将线程和数据包处理工作映射到频繁变化的可用核心集上。我们的运行时系统不是完全 Linux 兼容的，但它提供了一个现实的并发编程模型（继承自 Shenango），包括线程、互斥锁、条件变量和同步 I/O[47]。

Caladan 还包括一个部分兼容层，用于系统库（例如，libpthread），可以在不修改的情况下支持 PARSEC[9]，使我们对设计在未来支持未修改的 Linux 应用程序具有一定的信心。不使用我们运行时的应用程序可以共存在同一台机器上，但它们必须运行在不受 Caladan 管理的核心上，并且如果它们引起干扰，则无法被限制。

> 为什么不完全支持 linux，为什么

Caladan 更基本的要求是需要 LC 任务向运行时系统暴露其内部并发性（例如，通过生成绿色线程），这可能需要对现有代码进行更改。如果并发性不足，任务将无法从额外核心中受益，从而阻碍 Caladan 管理负载或干扰变化的能力。通常，我们建议任务通过为每个连接或每个请求生成一个线程来暴露并发性。例如，通常 memcached 为每个线程复用多个 TCP 连接，但我们修改它以生成一个单独的线程来处理每个 TCP 连接。

> 这个很重要，因为大部分 web 应用要么用线程池要么用事件驱动，每个线程复用 TCP
>
> 但一个线程用一个 TCP，这是否就降低了 memcached 的并发量和最大用户量、吞吐量，或者是线程数量增加？
>
> 其实除去尖峰， p99.9 尾部延迟是增加了的，

Caladan 的当前实现有两个限制。首先，它无法管理跨 NUMA 节点的干扰。NUMA 引入了额外的共享资源，这些资源容易受到干扰，包括跨套接字互连和每个节点的独立内存控制器。幸运的是，这些资源的高精度性能计数器是可用的，我们计划在未来探索 NUMA 感知的干扰缓解策略，例如在节点之间撤销核心或迁移任务。其次，我们的调度策略不会最小化跨超线程兄弟的瞬态执行攻击的威胁[3, 11, 65]。理想情况下，只有相互信任的任务才允许在兄弟核心上运行。在撰写本文时，Linux 内核正在开发类似的功能[13]。