+++
title = 'Paper Reading: Shenango'
date = 2024-09-30T00:41:37-04:00
draft = false
tags = ['Paper Reading']
+++

## Shenango: Achieving High CPU Efficiency for Latency-sensitive Datacenter Workloads

蛮知名的一篇文章，引用很高，后续论文也有 ghOSt，和 Efficient Scheduling Policies for Microsecond-scale Tasks

presentation 也很不错，看论文看到现在反而更喜欢看作者的 pre，有图就好理解，尤其是动图

> https://cloud.tencent.com/developer/article/2322227
>
> 这篇文章总结的不错
>
> 算法很好理解，但实现起来应该很难，不得不说 system 方向或者 network 方向的论文都非常扎实，得好好补补基础。

## Abstract

数据中心应用要求操作系统提供微秒级尾延迟和高请求率，

实现微秒级延迟的最佳可用解决方案是内核旁路网络，它将 CPU 核心专门用于应用程序进行网络卡的自旋轮询。但这种方法浪费 CPU：即使在适度的平均负载下，也必须为预期的峰值负载分配足够的核心。

Shenango achieves comparable latencies but at far greater CPU **efficiency** 它在非常细的粒度上——每 5 微秒——重新分配核心给应用程序，使得延迟敏感型应用程序未使用的周期可以被批处理应用程序有效地利用。它通过(1)一种高效的算法来检测应用程序何时会从更多核心中受益，以及(2)一个名为 IOKernel 的特权组件来实现如此快速的重新分配率，该组件在一个专用核心上运行，从 NIC 引导数据包并协调核心重新分配。在处理延迟敏感型应用程序（如 memcached）时，我们发现 Shenango 实现了与 ZygOS（一种最先进的内核旁路网络栈）相当的尾延迟和吞吐量，但可以线性地将延迟敏感型应用程序吞吐量交换为批处理应用程序吞吐量，大幅提高 CPU 效率。

> efficiency 指的是？
>
> zygOS https://dl.acm.org/doi/10.1145/3132747.3132780
>
> ZygOS 在 IX 基础上构建（让网卡把任务分配到多个 CPU 核对应的不同队列），为了负载均衡，空闲的 CPU 核从其他核的队列里“窃取”任务（work stealing），并利用核间中断（IPI）降低响应包的处理延迟。

## Introduction

CPU 效率变得至关重要

像 ZygOS 这样的内核旁路网络栈能够通过绕过内核调度器支持更高吞吐量的微秒级延迟

然而，这些系统仍然浪费了大量的 CPU 周期；它们依赖于自旋轮询网络接口卡（NIC）来检测数据包到达，因此即使没有数据包要处理，CPU 也总是在使用中

低尾延迟和高 CPU 效率之间的紧张关系

为什么今天的系统迫使我们浪费核心来维持微秒级延迟？谷歌最近的一篇论文认为，糟糕的尾延迟和效率是由于系统软件已经针对毫秒级 I/O（例如，磁盘）进行了调整 [15]。事实上，今天的调度器只在粗粒度上做出线程平衡和核心分配决策（Linux 每四毫秒一次，Arachne [63]和 IX [62]每 50-100 毫秒一次），阻止了对负载不平衡的快速反应。

本文介绍了 Shenango，一个专注于实现三个目标的系统：（1）为数据中心应用程序提供微秒级端到端尾延迟和高吞吐量；（2）CPU-efficient packing of applications on **multi-core** machines；（3）由于同步 I/O 和标准的编程抽象（如轻量级线程和阻塞 TCP 网络套接字），提高了应用程序开发者的生产力。

> 同步 IO 为什么可以提高开发者生产力？容易理解？那支持异步 IO 吗

为了实现其目标，Shenango 解决了在非常细的时间尺度上重新分配核心到应用程序的难题；它**每 5 微秒重新分配一次核心**，比我们所知的任何系统都快几个数量级。Shenango 提出了两个关键思想。首先，Shenango 引入了一个高效算法，**根据可运行的线程和传入的数据包**准确地确定应用程序何时会从额外的核心中受益。其次，Shenango 每台机器分配一个忙碌的 busy-spinning 自旋核心给一个名为 **IOKernel** 的集中式软件实体，该实体将数据包 steers 引导到应用程序并跨它们分配核心。应用程序在**用户级**运行时中运行，这些运行时提供了高效的高级编程抽象，并与 IOKernel 通信以促进核心分配。

Shenango 实现了与 ZygOS [61]相似的吞吐量和延迟，ZygOS 是一种最先进的内核旁路网络栈，但 CPU 效率更高。例如 Shenango 可以实现每秒超过五百万次的 memcached 吞吐量，同时保持 99.9 百分位延迟低于 100 微秒（比 ZygOS 多一百万）。

## The Case Against Slow Core Allocators

为什么 millisecond-scale 毫秒级核心分配器在处理微秒级请求时无法保持高 CPU 效率。我们将 CPU efficiency 定义为用于执行应用程序级工作（而不是忙等待、上下文切换、数据包处理或其他系统软件开销）的周期比例。

> 这个定义是通用的吗？如果改成每 5us 分配一次，处于用户的时间肯定更多，那么这样的 cpu 效率肯定更高，坏处是什么呢

像 IX[62]和 Arachne[63]这样的工作引入了用户级核心分配器，它们在 50-100 毫秒的间隔内调整核心分配。同样，Linux 主要在毫秒级定时器滴答响应下重新平衡核心上的任务。不幸的是，所有这些系统调整核心的速度都太慢，无法有效地处理微秒级请求。

图 1 显示了我们模拟中提供的负载与 CPU 效率（使用周期除以分配周期）之间的关系。它还显示了 Shenango 运行时在同一工作负载下本地运行的效率，通过生成一个线程来执行每个请求持续时间的合成工作。对于模拟结果，我们用模拟器分配的核心数标记了每一段线段；锯齿形模式之所以出现，是因为只能分配整数数量的核心。**即使没有网络或系统软件开销，也必须保留大部分空闲核心以吸收负载的突发**，导致 CPU 效率的损失。这种损失在一至四个核心之间尤为严重，随着负载的变化，应用程序可能会在这种低效率区域花费大量时间。理想的系统将为每个请求的持续时间精确地启动一个核心，并实现完美的效率，因为应用程序级工作将与 CPU 周期一一对应。

On the other hand, a slow core allocator is likely to perform worse than its theoretical upper bound in practice. First, CPU efficiency would be even lower if there were more service time variability or tighter tail-latency requirements.

> service time variability
>
> 所以支持动态负载吗，shenango 提出的 fast core allocation 和 TCP 拥塞控制很像，但为什么是 fast，明明是先分配很少的核心

## Challenges and Approach

Shenango 的目标是通过**尽可能少地分配核心给每个应用程序来优化 CPU 效率** avoiding a condition we call compute congestion, 在这种情况下，如果不给应用程序分配额外的核心，将导致工作延迟超过几微秒。这个目标释放了被低估的核心供其他应用程序使用，同时仍然保持尾延迟在控制之中。

> 所以就是降低分配时间，所以能够降低尾部延迟，不需要等太久，那么平均延迟会降低多少呢 5us 是精心挑选的值吗
>
> 尽可能少地分配给每个应用程序核心来优化 CPU 效率，这提升了什么？不会更加慢吗，所以要控制算法在微妙内？

现代服务经常经历非常高的请求率（单个服务器每秒数百万数据包），核心分配开销使得按请求重新分配核心变得不可行。相反，Shenango 紧密接近这个理想，每五微秒检测一次负载变化，并每秒调整核心分配超过 60,000 次。如此短的调整间隔需要新的方法来估计负载。我们现在更详细地讨论这些挑战。

> estimating load 怎么做

Core allocations impose overhead, 分配带来开销：Arachne 重新分配一个核心需要 29 微秒的延迟[63]，而 IX 需要数百微秒，因为它必须更新 NIC 规则以引导数据包到核心

Estimating required cores is difficult: 以前的系统使用应用程序级指标，如延迟、吞吐量或核心利用率，来估计长时间尺度上的核心需求。Shenango 旨在估计瞬时负载，

> 负载估计的区别在哪？怎么预估核心数？所以 shenango 就干脆不预估，直接用 congestion 来冷启动？

### Shenango’s Approach

首先，Shenango 将**线程和数据包排队延迟**都视为计算拥堵的信号，它引入了一个高效的拥堵检测算法，利用这些信号来决定如果给应用程序更多的核心是否会受益。

> congestion detection algorithm 类似 TCP 吗
>
> 怎么知道是不是受益的

因此，Shenango 的第二个关键理念是专门分配一个持续运行的核心给一个称为 IOKernel（第 4 节）的集中式软件实体

IOKernel 进程以 root 权限运行，作为应用程序与网络接口卡硬件队列之间的中介。**通过持续运行，IOKernel 可以在微秒级检查线程和数据包队列，以协调核心分配**。此外，它可以提供低延迟的网络访问，并允许在软件中将数据包导向核心，当核心被重新分配时能够快速重新配置数据包导向规则。结果是，**核心重新分配仅需 5.9 微秒即可完成**，并且需要不到两微秒的 IOKernel 计算时间来进行协调。这些开销支持足够快的核心分配速率，既能够适应负载的变化，也能迅速纠正我们的拥塞检测算法中的任何误预测。

应用程序逻辑在每个应用程序的运行时（第 5 节）中运行，通过**共享内存**与 IOKernel 通信（图 2）。每个运行时都是不可信的，并负责提供有用的编程抽象，包括线程、互斥锁、条件变量和网络套接字。应用程序以库的形式链接到 Shenango 运行时，允许类似内核的功能在其地址空间内运行。

> 5.9us 是平均还是最低？
>
> app shared memory 和 IOkernel 通信，同步问题怎么解决？尤其是多分配核心的时候

在启动时，运行时创建多个内核线程（即 pthreads），每个都有一个本地运行队列，最多达到运行时可能使用的最大核心数。应用程序逻辑在轻量级用户级线程中运行，这些线程被放入这些队列中；**通过工作窃取实现跨核心的工作平衡**。我们将运行时创建的每个每核心内核线程称为 kthread，将用户级线程称为 uthread。Shenango 被设计为能够在未经修改的 Linux 环境中共存；IOKernel 可以配置为管理一部分核心，而 Linux 调度器管理其他部分。

> steal

## IO Kernel

The IOKernel runs on a dedicated core

1. 在任何时候，它决定为每个应用程序分配多少核心（4.1.1 节）以及分配哪些核心给每个应用程序
2. 它处理所有的网络 I/O，绕过了操作系统内核。在接收路径上，它直接轮询 NIC 接收队列，并将每个传入的数据包放到应用程序核心之一的**共享内存队列**中。在传输路径上，它轮询每个运行时的数据包出口队列，并将数据包转发到 NIC

### Core Allocation

IOKernel 必须快速做出核心分配决策，因为其花费在核心分配上的任何时间都不能用于转发数据包，从而降低吞吐量。为了简化，IOKernel 分离了它的两个决策；in most cases, it first decides if an application should be granted an additional core, and then decides which core to grant.

#### Number of cores per application

每个应用程序的运行时都会被分配一定数量的 guaranteed cores 和一定数量的 burstable cores。运行时总是有权使用其保证的核心，而不会面临被抢占的风险（不允许过量订阅），但如果它没有足够的工作来占用它们，它可以使用更少（甚至是零）的核心。当有额外的核心可用时，IOKernel 可以将它们作为可突发核心分配，使繁忙的运行时能够暂时超过其保证的核心限制。 在决定为运行时分配多少核心时，IOKernel 的目标是在避免计算拥塞（第 3 节）的同时最小化分配给每个运行时的核心数量。为了确定运行时是否有多余的核心，IOKernel 依赖于运行时的 kthread 在不需要时自愿放弃核心。当一个 kthread 找不到任何工作要做，意味着它的本地运行队列为空，并且没有从其他活跃的 kthread 中找到可窃取的工作时，它会放弃其核心并通知 IOKernel（我们称之为停放）。IOKernel 也可以随时抢占可突发核心，迫使它们立即停放。 IOKernel 利用其独特的视角通过监控活跃 kthread 的队列占用情况来检测即将发生的计算拥塞。当一个数据包到达一个没有分配核心的运行时，IOKernel 立即为其分配一个核心。为了监测活跃运行时的拥塞情况，IOKernel 每 5 微秒调用一次**拥塞检测算法**（算法 1）。

拥塞检测算法根据两种负载源判断一个运行时是否超载：排队的线程和排队的入站数据包。如果在拥塞检测算法连续两次运行中发现任何项目存在于队列中，则表明一个数据包或线程至少排队了 5 微秒。因为排队的数据包或线程代表了可以在另一个核心上并行处理的工作，运行时被认为“拥塞”，IOKernel 会为其**分配一个额外的核心**。我们发现，排队的持续时间比队列长度是一个更稳健的信号，因为使用队列长度需要仔细调整不同请求持续时间的阈值参数[63, 74]。

将队列实现为环形缓冲区可以实现简单且高效的检测机制。检测一个项目是否在队列中存在两个连续的时间间隔只是一个比较当前头部指针与前一次迭代的尾部指针的问题。运行时通过每个 kthread 的单个缓存行的共享内存向 IOKernel 暴露这种状态。 直观上，核心分配可能会表现出振荡行为，可能每次迭代都会添加和停放一个核心。这是设计的一部分，因为较慢的调整要么牺牲尾部延迟，要么阻止我们在短时间内复用核心。事实上，现代 CPU 能够进行足够高效的上下文切换；进程上下文标识符（PCIDs）允许在不刷新 TLB 的情况下交换页表。Linux 切换进程大约需要 600 纳秒，因此足以处理由 IOKernel 产生的核心重新分配速率。在第 7.3 节中，我们评估了不同的核心分配间隔对尾部延迟和 CPU 效率的影响。

![](https://s2.loli.net/2024/10/03/SUAf3bHuCaxkjr6.png)

![](https://s2.loli.net/2024/10/03/RmFVQ2I7TWwKsJ6.png)

> 拥塞算法，先放几个线程/核心，然后怎么判断再给后续的？

#### Which cores for each application

When deciding which core to grant to an application, the IOKernel considers three factors:

1. Hyper-threading efficiency, Intel 的超线程技术允许两个硬件线程在同一物理核心上运行。这些线程共享处理器资源，如 L1 和 L2 缓存及执行单元，但被暴露为两个独立的逻辑核心，因此，IOKernel 倾向于将**同一物理核心上的超线程分配给同一应用程序**。
2. Cache locality, 如果一个应用程序的状态已经存在于新分配给它的核心的 L1/L2 缓存中，它可以避免许多耗时的缓存未命中。因此，IOKernel 跟踪运行时的当前和过去的核心分配情况。
3. Latency, 因此，如果存在空闲核心，IOKernel 总是选择一个空闲核心而不是抢占一个忙碌的核心。

> 如何跟踪？哈希表？

一旦 IOKernel 选择了要分配给应用程序的核心，它还必须选择一个已停放的 kthread 来唤醒并在该核心上运行。为了缓存局部性，它首先尝试挑选最近在这个核心上运行过的 kthread。如果没有这样的 kthread 可用，IOKernel 会选择停放时间最长的 kthread，这样可以留出其他停放的 kthread，以防它们最近运行过的某个核心变得可用。

调用检测算法的总成本与总核心数量成线性关系。

![](https://s2.loli.net/2024/10/03/LVxnRmoOFgBqD7P.png)

> shenango 长期运行会怎么样，会不会存在追踪很久的记录

### Dataplane

The IOKernel busy loops, continuously polling the incoming NIC packet queue and the outgoing application
packet queues.

Packet steering.

由于 IOKernel 跟踪属于每个运行时的核心，因此它可以将传入的数据包直接交付给运行适当运行时的核心。在 Shenango 中，每个运行时都配置了自己的 IP 和 MAC 地址。当一个新的数据包到达时，IOKernel 通过在哈希表中查找 MAC 地址来识别其运行时。然后，IOKernel **使用 RSS 哈希选择该运行时内的一个核心**，并将数据包放入该核心的入站数据包队列中。虽然 Shenango 偶尔会重新排序数据包（例如，当分配给运行时的核心数量发生变化时），但我们发现，在短时间间隔内，同一流中的数据包通常会到达同一个运行时的入站数据包队列（第 7.3 节）。我们的系统可以通过采用类似 Intel 的 Flow Director [8] 或 FlexNIC [42] 等技术进一步优化数据包转向。

> RSS hash, Receive side scaling 网卡上的跨核收发 通过哈希分发

Polling transmission queues.

为了找到要传输的数据包而轮询多个出站队列可能会带来较高的 CPU 开销，尤其是在具有许多队列的系统中 [68]。由于 IOKernel 跟踪哪些 kthread 是活跃的，它只轮询对应于活跃 kthread 的出站运行时数据包队列。这使得轮询出站队列的 CPU 开销可以根据系统中的核心数量进行扩展。

> > IOKernel 的核心会处理别的事情吗，会带来其他的缓存一致性问题吗

## Runtime

Shenango 的运行时经过优化以提高编程性，提供了高级抽象，如阻塞的 TCP 网络套接字和轻量级线程。

### Scheduling

运行时在由 IOKernel 动态分配给它的核心之间进行应用内部的调度。

我们的运行时围绕每个 kthread 的运行队列和工作窃取构建，类似于 Go [6]，与 Arachne 的工作共享模型 [63] 相反

> 应该说类似 GMP？

受到 ZygOS 的启发，我们对 uthread 进行细粒度的**工作窃取**以减少尾部延迟，这对具有服务时间变化的工作负载特别有益

> stealing 可以减少 tail latency 的原因是什么

### Networking

我们的运行时负责为应用程序提供所有网络功能，包括 UDP 和 TCP 协议处理。

On the other hand, we found that there were significant advantages to relaxing ordering requirements and
violating flow consistent hashing

> 这是为什么，不需要排序？
>
> Berkeley Sockets 是 BSD 吗，和 posix 的关系是什么
>
> http://www.openss7.org/papers/strsock/sockimp.pdf
>
> 本质上看没什么区别，为什么论文不基于 POSIX？

An earlier version of the runtime attempted to support zero-copy networking 然而，我们发现这种方法存在严重的问题。首先，它需要 API 更改，破坏了与 Berkeley Sockets 的兼容性。其次，我们惊讶地发现它对性能有负面影响。进一步调查后，我们发现 IOKernel 的吞吐量对**驻留缓冲区**的数量敏感，因为 DDIO（一种将数据包有效载荷直接推送到 LLC 的 Intel 技术）对数据包数据可以占用的**最大缓存行数**有限制。当超过这个限制时，数据包数据被推送到 RAM，大大增加了访问延迟。通过复制有效载荷，我们可以鼓励 DDIO 重用相同的缓冲区，从而保持在其缓存占用阈值内。这与“泄漏的 DMA”问题 [70] 类似。

> 没看懂，有点难理解。

## Implementation

IOKernel && runtime

Shenango 用 C 语言实现，并包括 C++ 和 Rust 的绑定

IOKernel 由 2,244 行代码（LOC）实现，运行时由 6,155 行代码实现。

IOKernel 使用 Intel 数据平面开发套件（DPDK）[2] 版本 18.11，以快速从用户空间访问 NIC 队列。整个系统在未经修改的 Linux 环境中运行。

> 同样用了 DPDK

### IOKernel Implementation

Shenango 依赖于几个 Linux 内核机制来将线程固定到核心，并在 IOKernel 和运行时之间进行通信

运行时设置了一系列描述符环队列（灵感来自 Barrelfish 的轻量级 RPC 实现 [17]）

为了将运行时的 kthread 分配到特定的核心，IOKernel 使用 sched_setaffinity。

> IOKernel 到底怎么追踪应用使用过哪些 thread?

### Runtime Implementation

我们的运行时包括对轻量级线程、互斥锁、条件变量、读-复制-更新（RCU）、高分辨率定时器以及同步的 TCP 和 UDP 套接字的支持。与 IOKernel 一样，运行时使用有限的一组现有的 Linux 原语；它通过 **mmap 分配内存**，通过调用 `pthread_create()` 创建 kthread，并通过共享内存、eventfd 文件描述符和信号与 IOKernel 交互。我们根据 RFC [36] 从头实现 TCP，我们的 TCP 堆栈与 Linux 和 ZygOS 的 TCP 堆栈兼容，包括流量控制和快速重传，**但不包括拥塞控制**。

> RCU 这个周末一定看
>
> 其他技术都是很底层的，操作系统应该要了解的
>
> shenango 还实现了 TCP，是基于 dpdk 做的吗，为什么不做更快的协议呢

为了提高内存分配性能，运行时使用了每个 kthread 的缓存 [21]，特别是在分配线程栈和网络数据包缓冲区时。运行时提供了一个 RCU 子系统，以支持对主要读取的数据结构的高效访问 [52]。运行时在每个 kthread 重新调度后检测一个空闲期，以便释放任何过期的 RCU 对象。内部，RCU 用于 ARP 表以及 TCP 和 UDP 套接字表。

Shenango 提供了 C++ 和 Rust 的绑定，具有惯用的接口（例如，像 std::thread）和分别支持 lambda 和闭包的功能。大多数绑定都是围绕底层 C 库的薄包装。然而，我们的 uthread 支持利用了一个独特的优化。我们扩展了 Shenango 的 spawn 函数，在每个 uthread 栈的底部预留空间用于跳板数据（捕获、返回值空间等），避免了额外的分配。

Preemption.

当 IOKernel 发送 SIGUSR1 信号时，Linux 内核会将 CPU 状态保存到线程栈上的 trapframe 中，并调用运行时安装的信号处理程序。信号处理程序立即将控制权转移到调度器上下文并停车，将被抢占的 uthread 重新放回运行队列。运行中的 uthread 最终可能被另一个 kthread 窃取，或者如果重新分配到核心则在同一 kthread 上恢复运行。

> GMP? cooperative

在运行时执行的某些关键部分，通过增加线程本地计数器来延迟抢占信号。这些部分包括整个调度器上下文、RCU 和自旋锁的关键部分，以及访问每个 kthread 状态的代码区域。支持活动 uthread 的抢占带来了一些挑战。如果线程上下文开始在不同的 kthread 上执行，指向 thread-local storage（TLS）的指针可能会变得陈旧。不幸的是，gcc 没有提供禁用这些地址缓存的方法。据我们所知，Microsoft 的 C++ 编译器是唯一支持这一点的编译器。作为变通方案，我们使用自己的 TLS 机制来处理在调度器上下文之外访问的每个 kthread 数据结构，并且目前要求应用程序在访问线程本地变量（包括 glibc 的 malloc 和 free）时禁用抢占。我们正在考虑扩展运行时以支持每个 uthread 的 TLS，从而减轻开发者的负担。然而，TLS 数据部分必须保持较小，以防止在创建 uthread 时出现较高的初始化开销。

> 延迟抢占，自己实现 TLS 使得 uthread 不能访问已经失效的缓存？

## Evaluation

1. latency and CPU efficiency
2. sudden bursts in load
3. What is the contribution of the individual mechanisms in Shenango to its observed performance?

> hyper-threads

我们比较了 Shenango 与 Arachne、ZygOS 和 Linux。

我们评估了 memcached

我们还用 Rust 编写了几个新的 Shenango 应用程序来测量不同的负载模式，利用了语言特性如闭包和移动语义。例如，我们实现了一个 spin-server，通过使用 CPU 一段时间来模拟计算密集型应用程序，然后再响应每个请求。

此外，我们实现了 loadgen，一个现实的负载生成器，可以为我们的 spin-server 以及 memcached 生成精确定时的请求模式。这两款应用程序共需要 1,366 行代码。为了与其它系统进行比较，我们使用了 ZygOS 仓库 [7] 中的 ZygOS 和 Linux spin-server 变体，并为 Arachne 实现了自己的 spin-server。

### CPU Efficiency and Latency

memcached, spin-server, gdnsd

Shenango 必须将一个核心（2 个超线程）专用于运行 IOKernel，因此应用程序可用的超线程减少了两个

> IOKernel 占用的核心不能处理其他事情，那这个核心会导致缓存行失效，L1 miss 率高的问题吗

Shenango 在几乎所有负载下的批处理应用吞吐量都优于其他系统。在极低负载下，Linux 因为没有为 IOKernel 或核心仲裁器预留超线程，所以批处理吞吐量最高。

> Shenango 基于 DPDK，为什么不和 DPDK 比一下呢
>
> 而且 memcached 几乎都是 99.8% GET 请求，写会导致什么呢

为了评估 Shenango 在存在批处理应用的情况下处理服务时间变化的能力，我们运行了 spin-server，Shenango 每秒可重新分配核心多达 60,000 次，能够快速适应负载突增并保持较低的尾部延迟，同时将未使用的周期授予批处理应用。

我们通过同时运行 gdnsd 和 swaptions 来评估 UDP 性能，

### Resilience to Bursts in Load

In this experiment, we generate TCP requests with 1 µs of fake work

> 虽然排除了 Linux 和 ZygOS
>
> 但我认为这个指标很重要，面对高负载可以反应迅速，尾部延迟还是很低，几乎是一条直线，怎么做到的

### Microbenchmarks

> 还给每个组件进行了 benchmark

Thread library

Network stack and core allocation overheads

Packet load balancing.

Core allocation interval.

### Conclusion

ZygOS 与 Shenango 最为相似；它在 IX 的基础上增加了工作窃取，以改进应用程序内的负载均衡。然而，这些系统都无法在细粒度上动态地在应用程序之间重新分配核心。相反，它们静态地将核心分区到应用程序，或者使用外部控制平面在大时间尺度上重新配置核心分配。
