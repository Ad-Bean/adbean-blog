+++
title = 'Paper Reading: IX: A Protected Dataplane Operating System'
date = 2024-09-15T14:47:50-04:00
draft = false
tags = ['Paper Reading']
+++

## IX: A Protected Dataplane Operating System for High Throughput and Low Latency

OSDI 2014, Dune 的衍生，同样利用硬件虚拟化 VT-x 技术。IX 最重要的是 Data Plane 和 Control Plane 分离了。

不得不说这类型的论文阅读起来太费劲了。

## Abstract

传统观点认为，激进的网络需求，比如小消息的高包速率和微秒级的尾部延迟，最好在**内核之外的用户级网络堆栈**中进行处理。我们介绍了 IX，一个数据平面操作系统，它提供了高 I/O 性能，同时保持了现有核心提供的强保护的关键优势。IX 使用**硬件虚拟化将内核**(控制平面)的管理和调度功能与网络处理(数据平面)分离开来。数据平面体系结构建立在原生的**零拷贝 API** 之上，通过将硬件线程和网络队列专用于数据平面实例，处理有界批量的数据包以完成，并通过消除一致性流量和多核同步来优化带宽和延迟。我们展示了 IX 在吞吐量和端到端延迟方面都明显优于 Linux 和最先进的用户空间网络堆栈。此外，IX 提高了一个广泛部署的键值存储器的吞吐量达到 3.6 倍，并减少了超过 2 倍的尾延迟。

> IX，利用 VTX 将控制平面（内核）和数据平面（网络处理）区分开，后者利用零拷贝处理？
>
> 延迟和吞吐都有提升

## Introduction

need for networking stacks that provide more than high streaming performance. The new requirements include high packet rates for short messages, microsecond-level responses to remote requests with tight tail latency guarantees, and support for high connection counts and churn

> 目前对高性能网络的需求

some systems **bypass the kernel** and implement the networking stack in **user-space** . While kernel bypass eliminates context switch overheads, on its own it does not eliminate the difficult **tradeoffs** between high packet rates and low latency (see §5.2). Moreover, user-level networking suffers from lack of **protection**. Application bugs and crashes can corrupt the networking stack and impact other workloads. Other systems go a step further by also replacing TCP/IP with RDMA in order to offload network processing to specialized adapters [17, 31, 44, 47]. However, such adapters must be present at both ends of the connection and can only be used within the datacenter.

> bypass 比如 dpdk, 在用户态处理了，消除了上下文切换开销
>
> 并没有消除在 高数据包速率 和 低延迟之间 的艰难权衡，这种 tradeoff 怎么处理？高速率和低延迟看着并不是矛盾的
>
> 另一种解决就是 RDMA 设备

We propose **IX**, an **operating system** designed to break the **4-way tradeoff** between high throughput, low latency, strong protection, and resource efficiency. Its architecture builds upon the lessons from high performance middleboxes, such as firewalls, load-balancers, and software routers [16, 34]. IX **separates the control plane**, which is responsible for system configuration and coarse-grain resource provisioning between applications, **from the dataplanes**, which run the networking stack and application logic. IX leverages Dune and virtualization hardware to run the dataplane kernel and the application at distinct protection levels and to isolate the control plane from the dataplane [7]. In our implementation, the control plane is the full Linux kernel and the dataplanes run as protected, **library-based operating systems** on dedicated hardware threads

> IX 是 OS，使用了 Dune 和 VT-x 虚拟化技术
>
> 打破 4way tradeoff 听上去也太顶级了。
>
> 控制平面用于系统配置和资源供应
>
> 数据平面用于网络协议栈
>
> IX 实现的控制平面是完整的 Linux 内核，数据平面作为受保护的、基于库的操作系统在专用的硬件线程上运行

The IX dataplane allows for networking stacks that optimize for both **bandwidth** and **latency**. It is designed around a native, **zero-copy** API that supports processing of bounded batches of packets to completion

> 一些细节，数据平面 optimizes for multi-core，应该是解决了 Dune 存在的一些问题，比如抛弃了 POSIX，使用别的比如事件驱动。

IX 对比 Linux 和 mTCP（用户级 TCP/IP），IX 的吞吐量分别比 Linux 和 mTCP 高出 10 倍和 1.9 倍

> 听上去和用户级 tcp/ip 可能差距并不大，不知道有没有用户级的 kcp/quic 能否比较一下

两个 IX 服务器之间的空载单向延迟为 5.7 微秒，比标准 Linux 内核之间的延迟好 4 倍，

## Background and Motivation

### Challenges for Datacenter Applications

Microsecond tail latency: 将一些服务请求的延迟降低到几十 μs 是至关重要的，我们还必须考虑跨数据中心的 RPC 请求的延迟分布的长尾。虽然尾部公差实际上是一个端到端的挑战，但系统软件栈在加剧问题方面起着重要作用[36]。总的来说，理想情况下，每个服务节点都必须严格限制第 99 百分位的请求延迟

> tail-tolerance is actually an end-to-end challenge 应该在分布式很重要

High packet rates: 如果系统软件不能处理**大量的连接计数**，对应用程序可能会产生重大影响

> TCP 为什么无法处理大量连接？是因为握手之类的？

Protection: isolation between applications

Resource efficiency: 每个服务节点将使用最少的资源(核心、内存或 IOPS)来满足任何时候的数据包速率和尾延迟需求

### The Hardware – OS Mismatch

硬件发展

商用操作系统

> 这些文章大部分的 motivation 都来自于硬件发展快速，比如 100Gbe 网卡，商用系统不够专用等等

### Alternative Approaches

User-space networking stacks: OpenOnload [59] ，mTCP [29]和 Sandstorm [40]等系统在用户空间中运行整个网络栈，以消除内核交叉开销并优化数据包处理，而不会引起内核修改的复杂性

更重要的是，当网络被提升到用户空间，而应用程序错误可能破坏网络堆栈时，安全性权衡就会出现

Alternatives to TCP: RDMA 可以减少延迟，但是需要在连接的两端都有专门的适配器。使用普通的以太网，Facebook 的 memcached 部署使用 UDP 来避免连接可伸缩性限制[46]。即使 UDP 在内核中运行，可靠的通信和拥塞管理仍然委托给应用程序

> 比如 KCP/QUIC 呢

Alternatives to POSIX API: MegaPipe replaces the POSIX API with lightweight **sockets** implemented with in-memory command rings [24]. This reduces some software overheads and increases packet rates, but retains all other challenges of using an existing, kernel-based networking stack.

> 都是自定义了

OS enhancements: Tuning kernel-based stacks provides incremental benefits with superior ease of deployment. Linux SOREUSEPORT allows multi-threaded applications to accept incoming connections in parallel. Affinityaccept reduces overheads by ensuring all processing for a network flow is affinitized to the same core [49]. Recent Linux Kernels support a **busy polling driver** mode that trades increased CPU utilization for reduced latency [27], but it is not yet compatible with **epoll**. When microsecond latencies are irrelevant, properly tuned stacks can maintain millions of open connections

> 不兼容 epoll 是什么意思，busy polling 是增加 CPU 利用率一直轮询，减少延迟。轮询和 epoll 事件通知有区别吧。

## IX Design Approach

microsecond latency && high packet rates

These requirements have been addressed in the design of **middleboxes** such as firewalls, load-balancers by integrating the networking stack and the application into a single dataplane

> middleboxes 集成了网络栈和应用程序到一个单一的数据平面，

Separation and protection of control and data plane:

相比之下，商用操作系统将协议处理与应用程序本身解耦，以提供调度和流量控制的灵活性。例如，内核依赖设备和软中断来从应用程序切换到协议处理。类似地，内核的网络栈会在应用程序不消耗数据的情况下生成 TCP ACK 并滑动其接收窗口，直到一定程度。

IX 扩展了数据平面架构，以支持不受信任的通用应用程序

**控制和数据平面的分离**也使我们能够考虑完全不同的 I/O API，同时允许其他操作系统功能（如文件系统支持）传递到控制平面以实现**兼容性**。类似于 Exokernel，每个数据平面在一个单一地址空间中运行一个单一应用程序。然而，我们使用**现代虚拟化硬件**在**控制平面**、**数据平面**和**不受信任的用户代码**之间提供三向隔离。数据平面在虚拟化系统中具有类似客户操作系统的功能。它们管理自己的地址转换，基于控制平面提供的地址空间，并且可以通过使用特权环来保护网络栈免受不受信任的应用程序逻辑的影响。此外，数据平面通过**内存映射 I/O** 直接访问 NIC 队列

> 这个 compability 是怎么做的，不同 IO API 怎么兼容。

Run to completion with adaptive batching:

The IX dataplane also makes extensive use of batching

有界自适应批处理和运行到完成的结合意味着传入数据包的队列只能在 NIC 边缘建立，在数据平面开始处理数据包之前。网络栈仅以应用程序处理的速度向对等方发送确认。应用程序处理速率的任何减慢都会迅速导致对等方的窗口缩小。数据平面还可以监控 NIC 边缘的队列深度，并向控制平面发出信号以分配额外的资源（更多硬件线程、增加时钟频率），显式通知对等方关于拥塞（例如，通过 ECN），并做出拥塞管理策略决策（例如，通过 RED）。

Native, zero-copy API with explicit flow control:

我们不暴露或模拟 POSIX 网络 API。相反，数据平面内核和应用程序通过存储在内存中的消息在协调的过渡点进行通信。我们的 API 设计为在两个方向上实现真正的**零拷贝操作**，从而提高延迟和数据包速率。数据平面和应用程序协作管理消息缓冲池。传入的数据包以只读方式映射到应用程序，应用程序可以在稍后点保留消息缓冲并将其返回给数据平面。应用程序向数据平面的发送传输内存位置的分散/聚集列表，但由于内容未被复制，应用程序必须保持内容不变，直到对等方确认接收。数据平面强制执行流量控制正确性，并可能修剪超过滑动窗口可用大小的传输请求，但应用程序控制传输缓冲。

> 很好奇这个零拷贝怎么做的

Flow consistent, synchronization-free processing:

multi-queue NICs provide flow-consistent hashing of incoming traffic to distinct hardware queues

## IX Implementation

### Overview

![](https://s2.loli.net/2024/09/16/2aXU59cO6Gxvj8B.png)

IX 架构，重点介绍了控制平面和多个数据平面之间的分离

IX 控制平面由完整的 Linux 内核和 IXCP（一个用户级程序）组成。

Linux 内核初始化 PCIe 设备（如网卡），并为**数据平面**提供资源分配的基本机制，包括核心、内存和网络队列。同样重要的是，Linux 提供了系统调用和服务，这些是与广泛应用程序兼容所必需的，如文件系统和信号支持。

VMX ring 0 中运行 Linux 内核，这种模式通常用于虚拟化系统中的虚拟机管理程序。我们使用 Linux 中的 **Dune** 模块，使**数据平面**能够在 VMX non root ring 0 中作为特定于应用程序的操作系统运行，这种模式通常用于虚拟化系统中的客户内核。应用程序像往常一样在 VMX 非根环 3 中运行。这种方法为数据平面提供了直接访问硬件功能（如页表和异常）和网卡的直通访问。此外，它提供了控制平面、数据平面和不受信任的应用程序代码之间的完全三向保护。

每个 IX 数据平面支持一个多线程应用程序。

> memcached 是一个高性能的分布式内存对象缓存系统，用于加速动态 Web 应用程序，通过减轻数据库负载来提高性能。
>
> httpd 是 Apache HTTP Server 的守护进程，通常简称为 Apache。它是一个开源的、跨平台的 Web 服务器软件，用于提供 HTTP 服务。httpd 可以配置为负载均衡器，将请求分发到多个后端服务器，以提高系统的性能和可靠性。

例如，图 1a 展示了一个用于多线程 memcached 服务器的数据平面和另一个用于多线程 httpd 服务器的数据平面。控制平面以粗粒度方式为每个数据平面分配资源。核心分配通过实时优先级和 cpusets 控制；内存以大页面分配；每个 NIC 硬件队列分配给单个数据平面。这种方法避免了在需求应用程序之间进行细粒度时间复用资源的额外开销和不稳定性。

> app 都跑在 ring3， IX 在 ring 0 non-root, Dune 和内核 也就是控制平面是在 ring0 vmx-root
>
> 为什么需要大页内存，减少 TLB？一般网络数据包处理都需要大页内存，减少页表遍历次数。例如，如果使用 4KB 的页面，一个 2MB 的内存区域需要 512 个 TLB 条目；而如果使用 2MB 的大页，只需要一个 TLB 条目。

每个 IX 数据平面作为一个**单一地址空间操作系统** single address-space OS 运行，并在共享的用户级地址空间中支持两种线程类型：

(i) 弹性线程 elastic threads，与 IX 数据平面交互以启动和消费网络 I/O；

(ii) 后台线程 background threads。

弹性线程和后台线程都可以发出任意 POSIX 系统调用，这些调用在转发到 Linux 内核之前由**数据平面**进行中介和安全验证。弹性线程预计**不会发出阻塞调用**，因为延迟的数据包处理会对网络行为产生不利影响。每个弹性线程独占使用分配给数据平面的核心或硬件线程，以实现高预测延迟的高性能。相比之下，多个后台线程可以分时共享分配的硬件线程。例如，如果一个应用程序分配了四个硬件线程，它可以将所有线程用作弹性线程来服务外部请求，或者它可以暂时过渡到三个弹性线程，并使用一个后台线程执行任务，如垃圾收集。当控制平面使用类似于 Exokernel 中的协议撤销或分配额外的硬件线程时，数据平面调整其弹性线程的数量。

> POSIX 旨在确保不同操作系统之间的兼容性，Portable，比如 I/O 之类的。POSIX 定义了网络编程的接口，如 socket 编程。
>
> 弹性线程，网络 IO。后台线程，垃圾回收？

### The IX Dataplane

IX 数据平面，它与典型的内核不同，因为它专门用于高性能网络 I/O，并且只运行一个应用程序，类似于库操作系统，但具有内存隔离。我们的数据平面仍然提供了许多熟悉的内核级服务。

> library OS,

数据平面还管理自己的虚拟地址转换，通过嵌套分页支持。与当代操作系统相比，它仅使用大页面（2MB）。我们倾向于使用**大页面 large pages**，因为它们的地址转换开销较小 [5, 7]，并且现代服务器中物理内存资源相对丰富。数据平面仅维护一个地址空间；内核页面通过超级用户位进行保护。我们**故意选择不支持可交换内存**，以避免增加性能变异性。

> 这个大页面，在 dpdk 也能看到，但是 not to support swappable memory 是什么意思，也就是纯内存的？应该很多都是纯内存的把

We provide a hierarchical timing wheel implementation for managing network timeouts,

我们当前的 IX 数据平面实现基于 Dune，并需要 Intel x86-64 系统上可用的 VT-x 虚拟化功能 [62]。然而，它可以移植到任何具有虚拟化支持的架构，例如 ARM、SPARC 和 Power。

IX 数据平面目前由 39K SLOC [67] 组成，并利用了一些现有代码库：41% 来自 Intel NIC 设备驱动程序的 DPDK 变体 [28]，26% 来自 lwIP TCP/IP 堆栈 [18]，15% 来自 Dune 库。

> 硬件限制还是很严重的，看着像缝合怪？lwIP 是 lightweight IP 开源 TCP/IP 协议栈，为什么不基于 UDP 呢，会不会吞吐和延迟更好看？还是说受限于 TCP，因为大部分 NIC 或者 middleboxes 都优化了 TCP
>
> 还有就是 batch 能不能变成 streaming？毕竟 TCP 底层应该是流的，字节流

### Dataplane API and Operation

应用程序的 elastic threads 弹性线程通过三种异步、非阻塞机制与 IX 数据平面进行交互

> 弹性线程 处理 IO

这些机制总结在表 1 中：它们向数据平面发出批量系统调用；它们消耗数据平面生成的事件条件；它们可以直接、安全地访问包含传入有效载荷的（mbuf）。后者允许对传入的网络流量进行零拷贝访问。应用程序可以保留 mbuf，直到它通过 recv done 批量系统调用请求数据平面释放它们。

Both batched system calls and event conditions are passed through arrays of **shared memory**

我们构建了一个用户级库，称为 libix，它抽象了我们底层 API 的复杂性。它为遗留应用程序提供了一个兼容的编程模型，并显著简化了新应用程序的开发。libix 目前包括一个与 libevent 和**非阻塞 POSIX 套接字操作非常相似的接口**。它还包括**新的零拷贝**读写操作接口，这些接口更高效，但需要对现有应用程序进行更改。libix 自动将多个写请求合并到每个批处理轮次中的单个 sendv 系统调用中。这提高了局部性，简化了错误处理，并确保了正确的行为，因为它即使在传输失败时也保留了数据流顺序。合并还促进了传输流控制，因为我们可以使用传输向量（sendv 的参数）来跟踪传出数据缓冲区，并在必要时在传输窗口有更多可用空间时重新发出写操作，如发送事件条件所通知的那样。我们目前的缓冲区大小策略非常基本；我们强制执行最大挂起发送字节限制，但我们计划在未来使这更加动态[21]。

> 这里的零拷贝是怎么实现的？用户态网络协议应该都跟零拷贝有关把，都需要共享内存，通过零拷贝直接访问内存缓冲，这样数据传输开销减少。

图 1b 说明了 IX 数据平面中弹性线程的运行到完成操作。NIC 接收缓冲区映射在服务器的内存中，NIC 的接收描述符环填充了一组缓冲区描述符，允许它使用 DMA 传输传入的数据包。**弹性线程（1）轮询接收描述符环**，并可能向 NIC 发布新的缓冲区描述符，以便用于未来的传入数据包。然后，弹性线程（2）通过 TCP/IP 网络堆栈处理有限数量的数据包，从而生成事件条件。接下来，线程（3）切换到用户空间应用程序，该应用程序消耗所有事件条件。假设传入的数据包包括远程请求，应用程序处理这些请求并以一批系统调用进行响应。在从用户空间返回控制权后，线程（4）处理所有批量系统调用，特别是那些指导传出 TCP/IP 流量的调用。线程还（5）运行所有内核定时器，以确保符合 TCP 行为。最后（6），它将传出的以太网帧放入 NIC 的传输描述符环中进行传输，并通过更新传输环的尾部寄存器通知 NIC 启动这些帧的 DMA 传输。在单独的传递中，它还根据传输环的头部位置释放任何已完成传输的缓冲区，可能会生成发送事件条件。该过程在一个循环中重复，直到没有网络活动。在这种情况下，线程进入一个静默状态，该状态涉及超线程友好的轮询或可选地进入一个节能的 C 状态，代价是一些额外的延迟。

> 完整的流程，使用的是轮询，IX 能实现更高效的 epoll 或者其他事件驱动机制吗？IX 应该是有 Multi-queue 的, IX 框架中的弹性线程负责处理网络数据包。每个弹性线程可以绑定到一个特定的 CPU 核心， 提高并行处理能力

### Multi-core Scalability

IX 数据平面针对**多核可扩展**性进行了优化，因为弹性线程在常见情况下以无同步和无一致性的方式运行。这比无锁同步的要求更强，无锁同步即使在单个线程是特定数据结构的主要消费者时，也需要昂贵的原子指令[13]。这是通过一系列有意识的设计和实现权衡实现的。

> 不知道弹性线程是怎么实现的，应该是修复了 Dune 不好支持多线程的问题

API 的实现经过了精心优化。每个弹性线程管理自己的**内存池、硬件队列、事件条件**数组和批量系统调用数组。事件条件和批量系统调用的实现直接受益于 IX 和应用程序之间的显式协作控制转移。由于生产者和消费者之间没有并发执行，事件条件和批量系统调用是基于原子操作的无同步实现。

> 弹性线程应该就是用来轮询的，应该有队列？

第三，在网卡上使用**流一致性哈希**确保每个弹性线程操作一个不相交的 TCP 流子集。因此，在处理服务器应用程序的传入请求时，不会发生同步或一致性。对于具有出站连接的客户端应用程序，我们需要确保回复被分配到发出请求的同一弹性线程。由于我们无法反转 RSS 使用的 Toeplitz 哈希[43]，我们只需探测临时端口范围以找到一个会导致所需行为的端口号。请注意，这意味着客户端中的两个弹性线程不能共享到服务器的流。

> flow-consistent hashing 是什么，这个会导致冲突吗，流一致性哈希是一种用于网络数据包处理的哈希算法，其主要目的是确保同一网络流（如 TCP 连接）的数据包始终被分配到同一个处理单元

IX 确实有一些需要同步更新的共享结构。例如，ARP 表由所有弹性线程共享，并受 RCU 锁保护[41]。因此，常见情况下的读取是无一致性的，但罕见的更新不是。RCU 对象在每个弹性线程完成一个运行到完成周期的时间段后进行垃圾收集。

> 还是需要一致性，更新很罕见？

### Security Model

IX API 及其实现采用了一种应用程序代码与网络处理栈之间的协作流控模型。与用户级栈不同，在用户级栈中，应用程序被信任以正确执行网络行为，而 IX 的保护模型对应用程序的假设很少。恶意或行为不端的应用程序只能伤害自身，而不能破坏网络栈或影响其他应用程序。IX 中的所有应用程序代码都在**用户模式**下运行，而数据平面代码则在受保护的 Ring 0 中运行。应用程序无法访问数据平面内存，除非是只读的消息缓冲区。无法通过批量系统调用或其他用户级操作来违反对 TCP 和其他网络规范的正确遵循。此外，**数据平面**可用于强制执行网络安全策略，如防火墙和访问控制列表。IX 的**安全模型**与传统的基于内核的网络栈一样强大，这是所有最近提出的用户级栈所不具备的特性。

> IX 利用了 vt-x ring3 和 ring0 吧，应该是比 dune 更安全？分离也使得更安全
>
> IO MMU 没有用到？

## Evaluation

> evaluation 表现都非常好，对比了 Linux 和用户态的 mTCP，
>
> 都使用了 TCP

### Experimental Methodology

我们启用了超线程，因为它可以提高性能

我们的基准测试的 Linux 客户端和服务器实现使用了 libevent 框架和 epoll 系统调用。我们从公共领域发布中下载并安装了 mTCP [30]，但必须使用 mTCP API 自己编写基准测试。我们使用 2.6.36 Linux 内核运行 mTCP，因为这是最新支持的内核版本。我们只报告了 mTCP 的 10GbE 结果，因为它不支持 NIC 绑定。对于 IX，我们将最大批处理大小绑定到每次迭代 B=64 个数据包，这在微基准测试中最大化吞吐量

### Latency and Single-flow Bandwidth

NetPIPE ping-pong benchmark

IX 服务器的单向延迟为 5.7μs, 并实现了 5Gbps 的吞吐量，这是最大值的一半，消息大小仅为 20KB

两个 Linux 服务器的单向延迟为 24μs, 385KB messages to achieve 5Gbps

> Linux 是因为中断，带来延迟

mTCP 使用积极的批处理来抵消上下文切换的成本，这在特定测试中以**更高的延迟**为代价，高于 IX 和 Linux。

> IX 消息小，处理很快，能达到高吞吐 5Gbps，单项延迟很低

### Throughput and Scalability

对于所有三个测试（核心扩展、消息计数扩展、消息大小扩展），IX 的扩展性都比 mTCP 和 Linux 更激进。图 3a 显示，IX 仅需要 3 个核心即可饱和 10GbE 链路，而 mTCP 需要所有 8 个核心。在图 3b 中，对于每个连接 1024 次往返，IX 每秒交付 880 万条消息，这是 mTCP 吞吐量的 1.9 倍，是 Linux 的 8.8 倍。在这个数据包速率下，IX 达到了线路速率，仅受限于 10GbE 带宽。

> IX 几乎是线性扩展，核心越多吞吐量越大，估计分离和弹性线程绑定 CPU + 批处理（减少上下文切换？）带来了很多好处

### Connection Scalability

能处理大量连接 a large number of concurrent connections

最多 250,000 个连接，这是我们可用客户端机器的上限。正如预期的那样，吞吐量随着连接并发度的增加而增加，但对于非常大的连接数，由于在打开的连接之间进行多路复用的成本越来越高，吞吐量会下降。在峰值时，IX 的性能比 Linux 好 10 倍，与图 3b 的结果一致。在 250,000 个连接和 4x10GbE 的情况下，IX 能够达到其峰值吞吐量的 47%。

> Linux 应该很难做到，但 IX 也有大量的 L3 缓存未命中？可以通过进一步优化 lwIP 和 TCP/IP 协议控制块结构大小和访问模式（这里没看懂）

### Memcached Performance

平均和第 99 百分位延迟作为实现吞吐量

第 99 百分位延迟捕捉尾部延迟问题，是数据中心应用程序最相关的指标 [14]。大多数商业 memcached 部署都为每个服务器配置，以使第 99 百分位延迟不超过 200μs 到 500μs。

我们尚未尝试调整 memcached 的内部可扩展性 [20] 或支持零拷贝 I/O 操作

IX 显著减少了未加载的延迟，大约减少了一半

> 这里很奇怪，为什么没支持零拷贝？但表现仍好于 Linux 而且好太多了

## Discussion

What makes IX fast:

a networking stack can be implemented in a **protected OS kernel** and still deliver wire-rate performance for most benchmark

IX 的好处不仅仅是最小化内核开销，零拷贝方法也有所帮助

没有中间缓冲区允许高效、特定于应用程序的 I/O 抽象实现，例如 libix 事件库

> 没有中间缓冲区是什么意思，传统网络栈应该是有的，而且需要在中间缓冲区做拷贝
>
> IX 去掉中间缓冲区，用零拷贝、共享内存来提高性能，又用 Dune/VT-X 来保护虚拟内存？

最后，我们仔细调整了 IX 以实现**多核可扩展性**，消除了引入同步或一致性流量的构造。

> Dune 对多核的支持不太好吧，不知道 IX 是如何解决的？是弹性线程吗

IX 数据平面优化——完成运行、**自适应批处理**和**零拷贝** API——也可以在用户级网络栈中实现，以获得类似的吞吐量和延迟方面的优势。虽然用户级实现会消除保护域交叉，但这不会导致比 IX 显著的性能提升。VMX 非根模式内的保护域交叉只会增加少量的额外开销，大约相当于一次 L3 缓存未命中 [7]。此外，这些开销在更高的数据包速率下很快被分摊。

> 这一段很重要， IX 比用户级网络栈快的原因到底是什么。用户级消除了什么，是进程上下文切换吗，VMX non-root 切换应该也有额外开销吧

批处理通常被理解为在低负载时以更高的延迟换取高负载时更好的吞吐量。IX 使用自适应、有界的批处理来实际改善这两个指标。图 6 比较了不同批处理大小上限 B 对 USR memcached 工作负载（图 5）的延迟与吞吐量的影响。在低负载下，B 不会影响尾部延迟，因为自适应批处理不会延迟挂起数据包的处理。在较高负载下，较大的 B 值提高了吞吐量，从 B=1 到 B=16 提高了 29%。对于这个工作负载，B≥16 最大化吞吐量。

在调整 IX 性能时，我们遇到了一个意外的硬件限制，该限制在高数据包速率和小平均批处理大小（即在数据平面饱和之前）时被触发：每次迭代发布新描述符所需的 PCIe 写入率高导致性能下降，因为我们扩展了核心数量。为了避免这个瓶颈，我们简单地在接收路径上合并 PCIe 写入，以便我们每次至少补充 32 个描述符条目。幸运的是，我们不必在发送路径上合并 PCIe 写入，因为这会影响延迟。

> 上界 B 在哪里调整的，感觉论文没有仔细讨论这一块，零拷贝好像也没怎么讨论

到目前为止，我们只测试了静态配置。在未来的工作中，我们将探索控制平面问题，包括**动态运行时**，以在保持吞吐量和延迟约束的同时，在可用弹性线程之间重新平衡网络流。

> 除了 TCP 应该还可以做 UDP？

## Related Work

与 IX 类似，Arrakis 使用硬件虚拟化将 I/O 数据平面与控制平面分离 [50]。IX 的不同之处在于它使用**完整的 Linux 内核作为控制平面**；在控制平面、网络栈和应用程序之间提供三向隔离；并提出了一种数据平面架构，该架构针对高吞吐量和低延迟进行了优化。另一方面，Arrakis 使用 **Barrelfish** 作为控制平面 [6]，并包括对 IOMMU 和 SR-IOV 的支持。

## Conclusion

memcached 移植到 IX 可以消除内核瓶颈，并将吞吐量提高多达 3.6 倍，同时将尾部延迟减少超过 2 倍。

<!--
论文自己也提到用户态的协议栈应该是很不错的，为什么测试下来比 IX 差这么多？
可能是因为 lwIP 选用的版本太老了？应该和 dpdk 比较一下吧

Your response should first include a summary of the following,  for the IX paper.

1. the problem the paper addresses (no more than 2-3 sentences)
2. the paper’s solution (no more than 2-3 sentences)
3. the evidence the paper provides that the solution was correct. (no more than 2-3)


Second, the response should answer the following, for the IX paper.
1. whether the problem the paper solved is an important one; are there any holes in the motivation?
2. particular strengths of the solution
3. particular weaknesses of the solution
4. whether the paper was fun or enjoyable to read
Finally, for this class's papers, please reflect on the following questions in your response:

 1. IX/Arrakis would be targeted a datacenter environment. How does this differ from a cloud environment, in terms of tradeoffs IX  or Arrakis can make to provide performance over things like security, or convenience (e.g., in IX, developers must rewrite applications as it is not POSIX-compliant.)


1. The paper IX addresses the challenge of achieving high throughput and low latency in high-performance networking environments while ensuring system security and isolation. It resolves the performance issues inherent in traditional operating systems like Linux and user-level network stacks.
2. IX separates the control plane from the data plane, the control plane runs in the kernel, while the data plane operates in user space, leveraging hardware virtualization (Dune and VT-x mentioned in last course) to ensure protection and resource management. IX also uese zero-copy API, shared memory, elastic threads, and batch processing to handle network packets, to reduce context switching, handle high packet rates and maintain low latency while achieveing high scalability.
3. The paper provides extensive evaluations showing that IX outperforms Linux and state-of-the-art user-space network stacks (lwIP, a lightweight TCP/IP stack) in terms of both throughput and end-to-end latency. Besides, for the memcached benchmark, IX improves the throughput up to 3.6x and reduces tail latency by 2x compared to Linux.

4. The problem solved by the IX paper is crucial, the paper mentions that it breaks the 4-way tradeoff between high throughput, low latency, strong protection, and resource efficiency. For modern datacenters like search engines or e-commerce, we not only need high throughput and low latency, but also security and efficiency since we dont want data leaks. And much like the paper says, Facebook is using UDP-based memcached for high performance and scalability, leaving the reliability and resource management to the application, so we need a more specialized operating system to handle network. But the paper doesn't mention any apps that need these protection, memcached or web servers can achieve high throughput and performance by using user-level network stack, the more important motivation should be protection and isolation.
5. Ix separates control and data planes, which ensures the data plane can operate with minimal interference from the control plane(kernel), leading to better performance and scalability while maintaining strong protection. Additionally, elastic threads and zero-copy API, batch processing improve both throughput and latency.
6. Like Dune, IX relies on specialized hardware virtualization like intel VT-x even though the paper says it is easy to migrate to ARM or other architectures. And for the elastic threads, I think using polling to handle network events can sometimes inefficient, especially during high CPU loads. even epoll is much suitable for reducing context switching, event-driven could be also good for high throughput and low latency. And IX only optimizes the TCP, I think QUIC or UDP could have better performance and latency.
7. The paper is very interesting and enjoyable, particularly due to the techniques it uses to enhance network performance. The elimination of network buffers, the shared memory, and zero-copy mechanisms are intriguing, reminds me of some user-space networking stacks like DPDK and lwIP. However, I was curious about why IX demonstrates significantly higher performance compared to lwIP or mTCP.

8.
In cloud environments, such as serverless computing, users are relieved from the burden of managing underlying hardware or virtualization techniques. We can focus solely on implementing code and application without needing to understand the underlying infrastructure and customized system API. However, in datacenter environments, operating systems like IX or Arrakis require users to have a deeper understanding of the hardware, and even the APIs these kernels provide. For examples, developers must be aware of various techniques such as zero-copy and how they modified the POSIX API, which are essential for optimizing performance.

Regarding security, I believe that Arrakis and IX offer a higher degree of customization, trading off compatibility and convenience for finer-grained resource management and isolation. While cloud environments also provide robust resource isolation and security, developers generally do not need to consider the underlying infrastructure. Cloud service providers like AWS offer a variety of environments to choose from, such as ARM or x86, allowing users to select based on cost or performance considerations. In contrast, IX requires support for virtualization technologies like Intel VT-x, which might make migration and implementation on platforms like ARM more challenging. Traditional operating systems and Linux or customized Unix offer highly abstracted interfaces that are well-suited for cloud environments.

In terms of performance, I don't necessarily think that Arrakis and IX provide a significant advantage over user-space networking stacks, especially when considering the development complexity. For instance, the performance improvements mentioned for memcached or Redis mentioned in IX and Arrakis papers have to be weighed against the development effort required. As a developer, I might prefer designs that are ready to use out of the box, considering the tradeoff between performance and the ease of development.

 -->
