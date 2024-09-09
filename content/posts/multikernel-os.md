+++
title = 'Paper Reading: The Multikernel: A New OS Architecture for Scalable Multicore Systems'
date = 2024-09-08T16:25:59-04:00
draft = false
tags = ['Paper Reading']
+++

## The Multikernel: A New OS Architecture for Scalable Multicore Systems

SOSP 09 的文章，提出了 MultiKernel 分布式系统，网络架构是如何做通信、消息传递的。

## ABSTRACT

商用计算机越来越多的 processor cores 和多样化的架构，内存、互联 、指令集、IO 配置等等，以前的高性能计算机已经可以 scaled，但是现代 server workloads 是动态的，OS 是静态的？优化很难做？

提出了 multikernel 架构，将机器看作网络/独立核心，假设底层没有 inter-core sharing，将传统 os 功能放到了分布式系统进程中，使用 message-passing 进行消息传递。

> 没看懂 motivation，难点到底是什么？解决了什么？

## INTRODUCTION

硬件发展快于软件，scalability and correctness 带来了挑战。多核系统的 workloads 更加难以预测，更偏向于 os-intensive。

针对特定硬件调优通用 OS 不再能接受，硬件区别太大，当有新硬件时，优化会过时。

优化往往调参（内存一致性模型、缓存层次、成本），可移植性不高。

shared-memory kernel，使用锁来包含共享数据结构，本文将 OS 看作分布式系统的一个功能单元（用消息进行通信），遵循三个设计原则，所有内核间通信是显式的；OS 结构与硬件无关；状态是复制而不是共享。

![multikernel](https://s2.loli.net/2024/09/09/eyM5R4XzqHJK6vI.png)

如图是本文提出的 multikernel 架构

尽管是当前高效的 cache-coherent shared memory，使用消息传递 OS 也带来了好处，shared data structure 会受到串行、远程数据访问的影响，对远程数据进行管道 pipeline 和 batch messages encoding 可以让一个 core 加大吞吐和减少 interconnect utilization，也适用于各种异构架构。

本文的贡献：

- 设计了 multi-kernel 架构，显式消息传递，硬件无关，状态复制
- 基于架构设计了 Barrelfish OS？
- 测量了 Barrelfish 满足可伸缩性和适应性。

> 将 OS 看作分布式，在我看来是非常夸张的想法，OS 本身就是和硬件紧密相连的，如果状态都是复制，都是通信，IO 带来的延迟谁能接受呢，那 RTOS 怎么做呢。
> 此外，我一直以为 CPU 的设计是现代硬件的基础，CPU 决定了架构，决定了 OS 长什么样。因为程序需要编译成指令集运行在 CPU 上，不同指令集可能效果不一样
>
> 消息传递又怎么保持一致性呢？OS 到底是基于硬件产生的？还是先有 OS 再有硬件？这根本不是先有鸡先有蛋的问题吧。
>
> 其实文章引出了一个很好的问题，共享内存 还是 消息传递。

## MOTIVATIONS

现代计算机都有多核处理器，商业服务器已经有几百核的处理器，所以需要新的 OS 技术面对多核硬件，还是说商用 OS 只需要利用多核处理器的技术？

本文认为 OS 在面对未来硬件的问题和用于高性能计算的 ccNUMA 和 SMP 不同（前者使用非统一内存访问和缓存一致性，访问远处的内存还是需要等待；后者是 Symmetric Multi-Processor 表示每个处理器都是对称的，没有主从关系，共享所有内存，但是存在竞争，所以扩展性较差）

> 非统一内存，内存分成多个节点，每个节点属于一个核心
>
> 缓存一致性，ccNUMA 所有处理器共享一个全局内存，需要缓存一致性保证所有处理器看到的内存数据是一致的。
>
> MPP(Massive Parallel Processing) 是多个 SMP，是 share nothing 架构，扩展能力强，节点间信息交换是互联网通信
>
> SMP 可以线性扩展，NUMA 理论可以无限扩展，但不是线性的，需要等待远处内存访问

> 看完 motivation 反而觉得文章合理了许多，文章注重点在于可扩展性

### Systems are increasingly diverse

和 hpc 不同，通用 OS 必须在多种硬件/系统设计中表现良好，所以无法对特定的 硬件进行优化

比如：

Dice and Shavit show how a **reader-writer lock** can be built to exploit the shared, banked L2 cache on the Sun Niagara processor, using concurrent writes to the same cache line to track the presence of readers

> 特定处理器上，利用共享 L2 缓存实现高效读写锁，缓存行在 L2 不会频繁移动。但是这在传统多核处理器不高效，会移动缓存。

所以，OS 设计如果针对特定的同步机制，对不同的硬件可能无法适应。推出新硬件时，OS 可能却难以适应。

os 为了适应现代硬件，需要采用日渐复杂的优化方法（举例...），linux readcopy update implementation 需要大量的迭代/

> 本文针对可扩展性，OS 的可扩展性？但是这些硬件也不是一般商用，也不通用啊 Sun Niagara 是更加多核多线程，指令集也是特殊的 SPARC V9，也是商用的服务器才能用上吧，商用肯定会有厂家做专门的 OS？
>
> 再说了，是开发新的硬件难 ，还是 OS 内核开发者做适配难呢？文章也举例一些 OS 的问题，优化很复杂等等，但是这例子 真的令人信服吗，win7, linux, windows server 2003 都是普遍性的，大量商用的，是有大量维护和使用的，再说 linux 开源也有不同发行版。

### Cores are increasingly diverse

多样性。

Moreover, core heterogeneity means cores can no longer share a single OS kernel instance, either because the performance tradeoffs vary, or because the ISA is simply different

> 核心异构，不同核心不同架构，不同指令集，比如常见高通骁龙处理器，大核心小核心异构，还有图形 GPU 架构，也有 AMD 的 APU 架构也是 CPU GPU 异构。

### The interconnect matters

尽管对于现代 cache-coherent multiprocessor，消息传递也替代了单个共享，因为可扩展性。CPU 之间的缓存一致性协议确保了 OS 可以安全地使用单个共享内存，但是像 路由 routing 和 congestion 拥塞这种网络问题是总所周知的。

### Messages cost less than shared memory

共享内存之前被认为是性能更好，但消息传递现在反转了？

![](https://s2.loli.net/2024/09/09/QEntPpm3DejzkqL.png)

![](https://s2.loli.net/2024/09/09/B9abYsWcXGQM2yq.png)

实验中 4x4 核心的 AMD 随着核心数增大，shared memory 延迟也增大，线性增长，因为缓存丢失需要等待。

> 核心间延迟？

但是对于消息传递，cache line 在服务端本地缓存，所以延迟不会线性增长。但是对于有时候延迟也高于共享内存

这个例子展示了在 cache-coherent shared memory 在少量核心上的可扩展问题，本文的条件就是共享内存模型存在不可/难以扩展的问题，并且硬件创新速度很快，为 OS 内核带来问题。

> 带来什么问题？可扩展问题？ OS kernel 不应该是性能和稳定优先，可移植性是很重要的指标吗？硬件更新速度真的遵从摩尔定律吗，而且硬件更新速度和可移植性应该关系并不是很大才对。

### Cache coherence is not a panacea

核心数量增加，互连增加，硬件的 cache-coherence protocols 也会非常昂贵。所以未来的 OS 需要处理非一致性内存，或者能够绕过 cache-coherence protocol

NIC 和 GPU 等可编程外围已经可以不需要和 CPU 保持缓存一致性，许多多核处理器也可以使用非一致性共享内存。

### Messages are getting easier

消息传递，是 shared nothing 的吗。共享数据存在正确性和性能缺陷，需要各种粒度的锁、数据结构来保证 cache line 竞争问题。

> shared nothing 是一种分布式计算架构，尤其是分布式数据库中，也有数据仓库数据湖的趋势，每个节点有自己的资源，具有良好的可扩展性 ，性能优秀也灵活。但是实现难度大，资源利用率可能不足，数据格式也是个问题。

消息传递带来的另一个顾虑是，stack ripping (调用栈剥离问题)，和事件驱动带来的 control flow 混淆？但是，传统的 monolithic kernels 就是事件驱动的，尽管在多核处理器上。

最后，消息传递和事件驱动模型是编程范式。

### Discussion

未来计算机架构发展趋势：越来越多的核心数量、硬件多样化（内核之间、系统之间）

we take the opposite approach: design and reason about the OS as a **distributed, non-shared system**, and then employ **sharing** to optimize the model where appropriate

![](https://s2.loli.net/2024/09/09/trmqosgkwIuXpvZ.png)

> 应该是一个复合的，共享 + 消息架构，但这样和 NUMA 又有什么很大的差别呢 ？文章好像也没有深入讨论消息传递缓存一致性的问题，

## THE MULTIKERNEL MODEL

- Make all inter-core communication explicit
- Make OS structure hardware-neutral
- View state as **replicated** instead of shared

### Make inter-core communication explicit

不共享内存，但是不排除程序在核心之间共享内存？OS 设计不依赖这个。

explicit communication 有助于系统互连。更加符合分布式系统，有利于 pipelining 和 batching。也能提供 isolation 和资源管理。

消息传递是异步的？发送后可以休眠、做别的事。

### Make OS structure hardware-neutral

信息传递机制和硬件接口（CPU 和设备）

> 一些平台没有缓存一致性机制，比如嵌入式，RTOS，专用计算设备 GPU FPGA？

late binding + 消息传递，动态绑定，灵活，优化了 TLB shootdown 维护 TLB 一致性

### View state as replicated

复制是可伸缩的重要技术

> 文章还是 share replicas of system states 不过是对于紧密耦合的核心或者线程，使用自旋锁保护同步。

### Applying the model

缺点：

- 某些基于特定平台的性能优化可能被牺牲，比如 核心之间共享的 L2 缓存
- 复制需要保持一致性协议

Barrelfish 的目标：

- 性能和目前商用操作系统可以比较
- 在大量核心上可以扩展，尤其是在全局 OS data structure 下的 workload
- 可以使用不同的共享机制、定位到不同硬件，不需要重构
- 可以通过消息传递，通过 pipeline, batching 实现良好性能
- OS 模块化，利用硬件拓扑或负载

## IMPLEMENTATION

### Test platforms

x86, 但是 ARM 却还在 wip？

> 说好的可以扩展呢，而且实验看上去也不是异构处理器。

### System structure

OS instance on each core into a privileged-mode CPU driver and a distinguished user-mode monitor process,

![](https://s2.loli.net/2024/09/09/jSgV9C5TOQaker6.png)

kernel space: cpu driver

user space: monitor

### CPU drivers

CPU driver 是时间驱动、单线程、不可抢占的，它以 trap 的形式连续处理来自用户进程的事件或来自设备或其他核心的中断。

> 全是 X86-64 架构，想知道基于 ARM 开发 OS 的难度究竟如何。而且 LRPC 的延迟看上去蛮高的，一般 E5 应该在 us microseconds 级别，文章数据基本都是 nanoseconds。

### Monitors

每个核心，复制的数据结构，比如内存分配表和地址映射表，都全局一致。是 monitor 运行一致性协议。

> 看上去 monitor 才是实现的核心，需要考虑复制、保持一致。其实这里能不能也实现一些零拷贝呢？

### Process structure

Communication in Barrelfish is not actually between processes but between **dispatchers** (and hence cores)

### Inter-core communication

不同场景不同传输实现，但是本文**核间通信**只实现了缓存一致性？

> inter-core 和 per-cores/dispatchers 消息传递还不一样，而且实际上并不是 processes 之间通信，是调度器

核间通信很重要，使用 URPC，

> urpc 好像基本只有国外教材才有了，不太清楚用户级别的 rpc 有什么区别 ，看上去利用了缓存，轮询，阻塞等等，那能不能改进成类似 epoll 等红黑树结构呢？

urpc 性能看上去挺不错的

### Memory management

虚拟内存管理

> 本来想仔细看看这部分，但看起来作者走了弯路，不知道最后是怎么实现的。文章提到的内存管理能力 capability 就是内存管理机制，通过能力引用和操作，标识内存对象，权限等等。同步怎么做？还需要 2PC 来同步吗？

### Shared address spaces

线程可以共享地址空间，这样不需要 IPC，但会面临数据不一致问题。

TLB 用于加速虚拟地址到物理地址。

Barrelfish 支持传统的进程模型，即多个调度器（dispatcher）在多个核心上共享单一的虚拟地址空间。

可以通过 共享所有调度器的硬件页表 TLB ，或者通过消息协议**复制硬件页**表来实现。

> 本文是怎么做的？

### Knowledge and policy engine

这一段不理解

### Experiences

大部分是 RPC 调用而不是系统调用，需要更多的上下文切换

## EVALUATION

### Case study: TLB shootdown

保持 TLB 一致性，TLB shootdown 是指在页面被取消映射时，通过使 TLB 条目失效来保持 TLB 一致性的过程。

本文使用消息传递，广播来实现 TLB shootdown，延迟会高一些。

### Messaging performance

2 阶段提交、轮询、IP lookback

> 两阶段提交的问题在这里会不会更明显呢？尤其是当 monitor 阻塞的时候，很可能产生长时间的阻塞和单点故障，当然性能也难以接受

### Compute-bound workloads

这些基准测试在任一操作系统上都没有特别好的扩展性，但至少证明了尽管 Barrelfish 具有分布式结构，它仍然可以支持大规模的共享地址空间并行代码，且性能损失很小。

> 性能差距并不是很大，至少超出了我的预期，我以为性能差距会很大。但很明显还是要看 latency 而不是只看 cycles、还需要看资源利用率、上下文切换、吞吐

吞吐量看着也不错，

Web server and relational database

作为服务器系统，说是避免了内核-用户态切换，能够明显减少上下文切换，所以能每秒处理更多的请求？

> 为什么？

## SUMMARY

microbenchmarks 结果看着不错

但是评估一个 os 是非常复杂的工作

> 文章没有真的在异构硬件上测试

<!--
1. the problem the paper addresses (no more than 2-3 sentences each)
2. the paper’s solution (no more than 2-3 sentences each)
3. the evidence the paper provides that the solution was correct. (no more than 2-3 sentences each)

1. whether the problem the paper solved is an important one
2. whether the solution was a good one
3. whether the paper was fun or enjoyable to read.


4. What are the pros/cons of shared state vs. message passing? When would you use one or the other? In the context of OS design, are there any cases where shared state might actually be better than message passing?

5. The COST paper complains that distributed systems papers are sometimes guilty of NOT evaluating against a simple single-threaded approach. However, this is not always possible -- some applications (such as building OS services) need to provide scalability and inherently cannot be single threaded. Given this, are there any lessons we can take from the COST paper when thinking about the multikernel?



-------------- 1 ---------------

The paper MultiKernel is trying to solve the challenge of designing and implementing a more flexible operating system that can be scaled with the increasing number of processor cores in modern hardware. Previous OS cannot handle scalability and performance issues due to shared memory patterns, cache coherence, and contentions. Besides, optimizations may become obsolete when new hardware arrives, and there are more and more diverse and heterogeneous hardwares that general-purpose OS cannot perform well.
The paper proposes MultiKernel OS, which treats multiple processors as distributed systems and network of independent cores. The OS uses message passing for communication, state replication instead of mapping, and there is no shared memory at lowest level. The researchers develope lots of RPC for communication between kernel space and user space, and they develope a Monitor to handle inter-core communication and CPU driver for managing tasks for each core.
The authors implement the MultiKernel OS called Barrelfish and conduct some evaluation of network throughput and web server and relational databases, the results show that Barrelfish can improve network throughput, can handle higher number of cucurrent connections compared to Linux, and have better scalability.

The paper is trying to solve a very important issue, especially for distributed systems and the current trend towards shared-nothing architectures. The multikernel design is hardware-neutral, it could be easily and effectively scaled, like MPP (even though the paper says it is not related, but I think the higher idea is the same).
The solution, in my opinion, is remarkable but can be improved, the use of distributed systems principles in OS design is innovative and provides a fresh approach to handling multiprocessor hardwares. But it still needs more tests in heteroeneous hardware like ARM or GPU, and provide more metrics like latency, utilization.
The reading is both fun and educational, I learned a lot about cache coherence and how distributed system works. I'm also interested in how they implement the RPC and consensus protocol during message passing, do they need more complex protocols like Raft? And will zero-copy help the MultiKernel perform better?

Shared state is very fast when dealing with inter-process communication since OS doesnot need to copy shared data to the message channel, but it requires synchronization to prevent data race and contention, so it's not design for scalable and flexible systems.
Message passing is suitable for scalability, and it is widely used in large distributed system, but it needs to copy and serialize the data, and sometimes it can be lower than shared state due to latency or agreement protocols.
I'll use shared state when developing multi-process programs like traditional databases, shared buffer or caching system is very helpful for improving the read latency.
When I'm trying to develop distributed programs, I will use message passing like microservices or distributed systems, using RPC or message queue can improve the scalability of the system.
In OS, especially in latency-matter and not only communication scenarios, shared state will be better than message passing. For example, the ring buffer in linux, it will be complicated to design buffer using message passing.


 -->
