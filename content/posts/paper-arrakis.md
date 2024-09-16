+++
title = 'Paper Reading: Arrakis'
date = 2024-09-16T00:22:05-04:00
draft = false
tags = ['Paper Reading']
+++

## Arrakis: The Operating System is the Control Plane

和 IX 同样是 OSDI 14 的文章，Arrakis 应该也是 Dune 沙丘里的名字吧，但好像不是同一批人 Adam Belay Standford 的人做的，应该很类似。

同样是用虚拟化技术，

## Abstract

应用程序可以直接访问虚拟化的 I/O 设备，允许大多数 I/O 操作完全跳过内核，而内核被重新设计以提供网络和磁盘保护，

## Introduction

高速以太网和低延迟持久内存的结合显著提高了 I/O 密集型软件的效率标准

许多服务器的大部分时间都用于执行操作系统代码：传递中断、demultiplexing 多路分解、和复制网络数据包，以及维护文件系统元数据。服务器应用程序通常执行非常简单的功能，如键值表的查找和存储，但在每次客户端请求时都会多次穿越操作系统内核。

这些趋势导致了一系列针对各种用例优化内核代码路径的研究：消除内核中的冗余副本[45]，减少大量连接的开销[27]，协议专业化[44]，资源容器[8, 39]，磁盘和网络缓冲区之间的直接传输[45]，中断转向[46]，系统调用批处理[49]，硬件 TCP 加速等。其中许多技术已被主流商业操作系统采用，但这是一场失败的战斗：我们展示了 Linux 网络和文件系统堆栈的延迟和吞吐量远不如硬件原始性能。

> I/O 密集型，比如网络服务器？

二十年前，研究人员提出通过将网络硬件直接映射到用户空间来简化并行计算中的数据包处理，以实现工作站网络的并行计算[19, 22, 54]。尽管在当时商业上并不成功，但虚拟化市场的兴起促使硬件供应商重新考虑这一想法[6, 38, 48]，并将其扩展到磁盘[52, 53]。

> 磁盘是什么做法？

本文探讨了在几乎所有 I/O 操作中将内核从数据路径中移除的操作系统含义。我们认为，这样做必须为应用程序提供与传统设计相同的安全模型；通过扩展可信计算基础以包括应用程序代码，例如允许应用程序未经滤直接访问网络/磁盘，很容易获得良好的性能。我们证明了操作系统的保护与高性能并不矛盾。在我们的原型实现中，对 Redis 持久化 NoSQL 存储的客户端请求在读取延迟方面提高了 2 倍，写入延迟提高了 5 倍，写入吞吐量提高了 9 倍，相比 Linux。

> 和 IX 或者说内核态网络栈一样的思路？但是优化了 Redis 看着很有意思，一个内存 NoSQL

我们做出了三个具体的贡献：

• 我们提出了一个架构，用于划分设备硬件、内核和运行时在非特权进程直接进行网络和磁盘 I/O 时的分工，并展示了如何高效地模拟我们的模型，以适应不完全支持虚拟化的 I/O 设备（§3）。

• 我们将我们的模型实现为一个开源 Barrelfish 操作系统的修改集，运行在商业可用的多核计算机和 I/O 设备硬件上（§3.8）。

• 我们使用我们的原型来量化用户级 I/O 对几种广泛使用的网络服务的潜在好处，包括分布式对象缓存、Redis、IP 层中间盒和 HTTP 负载均衡器（§4）。我们展示了在许多情况下，相对于 Linux，在延迟和可扩展性方面可以获得显著的提升，而无需修改应用程序编程接口；通过改变 POSIX API 可以获得额外的收益（§4.3）。

> 基于 Barrelfish 做的，也就是 exokernel，是 Multikernel，利用信息传递，也是区分了 user space 和 kernel space，每个核心有自己的 kernel
>
> 和 IX 区别是不是基于 Dune，不过同样的都是做了分布式对象缓存、Redis、IP 层中间盒和 HTTP 负载均衡器，延迟优化的同时不需要修改程序接口？为什么？

## Background

我们首先详细分析了当前网络和存储操作中的操作系统和应用程序开销，随后讨论了支持用户级网络和 I/O 虚拟化的当前硬件技术。

### Networking Stack Overheads

Linux 进程实现的 UDP 回显服务器

• 网络栈成本：硬件、IP 和 UDP 层的数据包处理。
• 调度器开销：唤醒进程（如果需要），选择它运行，并切换到它。
• 内核穿越：从内核到用户空间，再返回。
• 数据包数据的复制：在接收时从内核复制到用户缓冲区，在发送时返回。

在 Linux 中，处理每个数据包总共花费 3.36 微秒（见表 1），其中近 70%的时间花在网络栈中。这项工作主要是软件多路分解、安全检查以及由于各层间接导致的开销。内核必须验证传入数据包的头部，并在应用程序发送数据包时对提供的参数进行安全检查。栈还在层边界执行检查。

调度器开销在很大程度上取决于接收进程当前是否正在运行。如果是，只有 5%的处理时间花在调度器上；如果不是，从空闲进程上下文切换到服务器进程会增加额外的 2.2 微秒，网络栈的其他部分还会进一步减慢 0.6 微秒。

多核系统上的缓存和锁争用问题增加了进一步的开销，并且由于网络卡可以将传入消息传递到不同的队列，导致它们由不同的 CPU 核心处理，这加剧了问题——这些核心可能与用户级进程调度的核心不同，如图 1 所示。高级硬件支持，如加速接收流转向[4]，旨在减轻这种成本，但这些解决方案本身会带来非同小可的设置成本[46]。

> 多核心是怎么做的，这个是第一次接触的场景，如何保持一致性？

通过利用硬件支持将内核从数据平面中移除，Arrakis 可以完全消除某些类别的开销，并最小化其他开销的影响。表 1 还显示了 Arrakis 两种变体的相应开销。Arrakis 完全消除了调度和内核穿越开销，因为数据包直接传递到用户空间。当然，网络栈处理仍然是必需的，但它大大简化了：不再需要为不同应用程序多路分解数据包，用户级网络栈也不需要像内核实现那样广泛地验证用户提供的参数。因为每个应用程序都有单独的网络栈，并且数据包传递到应用程序运行的核心，锁争用和缓存效应减少了。

> 差不多的思路

在 Arrakis 网络栈中，将数据包数据复制到用户提供的缓冲区并从中复制的时间主导了处理成本，这是 POSIX 接口（Arrakis/P）与 NIC 数据包队列之间不匹配的结果。到达的数据首先由网络硬件放入网络缓冲区，然后复制到 POSIX 读取调用指定的位置。要传输的数据被移动到可以放入网络硬件队列的缓冲区中；然后 POSIX 写入可以返回，允许在数据发送之前重用用户内存。尽管研究人员已经研究了从内核网络栈中消除这种复制的方法[45]，如表 1 所示，内核驻留网络栈的大部分开销在其他地方。一旦消除了穿越内核的开销，就有机会重新思考 POSIX API 以实现更高效的网络。除了 POSIX 兼容接口外，Arrakis 还提供了一个本地接口（Arrakis/N），支持真正的零拷贝 I/O。

> 同样使用了零拷贝技术，arrakis 自己提供了一个零拷贝，是怎么做的？

### Storage Stack Overheads

fsync 测试，实际上就是测 IO

> 延迟写
>
> 传统的 UNIX 实现的内核中都设置有缓冲区或者页面高速缓存，大多数磁盘 IO 都是通过缓冲写的。
>
> 当你想将数据 write 进文件时，内核通常会将该数据复制到其中一个缓冲区中，如果该缓冲没被写满的话，内核就不会把它放入到输出队列中。
>
> 当这个缓冲区被写满或者内核想重用这个缓冲区时，才会将其排到输出队列中。等它到达等待队列首部时才会进行实际的 IO 操作。

### Application Overheads

NoSQL 测试

### Hardware I/O Virtualization

Single-Root I/O Virtualization (SR-IOV)

并通过 IOMMU（例如 Intel 的 VT-d [34]）进行访问保护

在 Arrakis 中，我们使用 SR-IOV、IOMMU 和支持适配器来提供对 I/O 设备的直接应用程序级访问。这是 20 年前使用 U-Net [54]实现的一个想法的现代实现，但已推广到闪存存储和以太网网络适配器。

> 同样使用虚拟化技术，让应用可以直接访问 IO 设备

虽然 RDMA 为并行应用程序提供了用户级网络的性能优势，但将其模型应用于更广泛的客户端-服务器应用程序[21]具有挑战性。最重要的是，RDMA 是点对点的。每个参与者接收一个认证器，授予其远程读/写特定内存区域的权限。由于客户端-服务器计算中的客户端不是相互信任的，硬件需要为每个活动连接保留单独的内存区域。因此，我们在这里不考虑 RDMA 操作。

> 不考虑 RDMA 为什么，点对点，不信任？

## Design and Implementation

Minimize kernel involvement for data-plane operations: Arrakis 旨在限制或消除内核对大多数 I/O 操作的干预。I/O 请求在应用程序的地址空间之间路由，无需内核参与，同时不牺牲安全性和隔离性。

> 基本都是消除内核态的 IO 干预

Transparency to the application programmer:

Appropriate OS/hardware abstractions:

> 其他感觉看看就行，都是虚拟化带来的好处

![](https://s2.loli.net/2024/09/16/TzYM3PtUay6dnJX.png)

### Architecture Overview

### Hardware Model

我们工作的关键要素是开发一个与**硬件无关的虚拟化 I/O 层**——即提供“理想”硬件特性集的设备模型。这个设备模型捕捉了在硬件中实现传统内核数据平面操作所需的功能。我们的模型类似于一些硬件 I/O 适配器已经提供的内容；我们希望它能为支持安全的用户级网络和存储提供指导

> virtual network interface cards VIC 是怎么来的

Queues:

Transmit and receive filters:

Virtual storage areas:

Bandwidth allocators:

### VSIC Emulation

看不懂

> 但是比 IX 等更加细节，应该是能复现的

### Control Plane Interface

应用程序与 Arrakis 控制平面之间的接口用于从系统请求资源，并将 I/O 流引导到用户程序。该接口提供的关键抽象是 VICs、门铃、过滤器、VSAs 和速率说明符。

> 这里的实现很有意思，可以对比 IX 看看，IX 用了 flow consistent 哈希，
>
> 最大区别应该是 arrakis 支持 iommu，ix 没有？

### File Name Lookup

VFS + POSIX

> arrakis 支持文件存储应该很有意思，IX 支持吗？还是纯内存的，应该也是支持的吧，论文提到了 SDD 和 flash。

### Network Data Plane Interface

在 Arrakis 中，应用程序通过直接与硬件通信来发送和接收网络数据包。因此，数据平面接口在应用程序库中实现，允许它与应用程序共同设计[43]。Arrakis 库为应用程序提供了两个接口。我们描述了本地的 Arrakis 接口，它稍微偏离了 POSIX 标准以支持真正的零拷贝 I/O；Arrakis 还提供了一个支持未修改应用程序的 POSIX 兼容层。

> 怎么做到真正的零拷贝？

应用程序在队列上发送和接收数据包，这些队列之前已经分配了过滤器，如上所述。虽然过滤器可以包括 IP、TCP 和 UDP 字段谓词，但 Arrakis 并不要求硬件执行协议处理，只进行多路复用。在我们的实现中，Arrakis 在数据平面接口之上提供了一个用户空间网络栈。该栈旨在最大化延迟和吞吐量。我们保持了数据包传输和接收的三个方面的清晰分离。

首先，数据包使用传统的 DMA 技术通过数据包缓冲区描述符环在网络和主内存之间异步传输。其次，应用程序通过将缓冲区链入硬件描述符环来将传输数据包的所有权转移给网络硬件，并通过反向过程获取接收到的数据包。这是通过两个 VNIC 驱动程序函数完成的。send_packet(queue, packet_array) 在队列上发送数据包；数据包由散布/聚集数组 packet_array 指定，并且必须符合已与队列关联的过滤器。receive_packet(queue) = packet 从队列接收数据包并返回指向它的指针。这两个操作都是异步的。packet_done(packet) 将接收到的数据包的所有权返回给 VNIC。

为了获得最佳性能，Arrakis 栈将通过编译器生成的、针对 NIC 描述符格式优化的代码直接与硬件队列交互，而不是通过这些调用。然而，我们在本文中报告的实现使用函数调用驱动程序。第三，我们使用与队列关联的门铃处理异步事件通知。当应用程序运行时，门铃通过硬件虚拟化中断直接从硬件传递给用户程序，当应用程序不运行时，通过控制平面调用调度器。在后一种情况下，较高的延迟是可以容忍的。门铃通过常规事件传递机制（例如文件描述符事件）暴露给 Arrakis 程序，并完全集成到现有的 I/O 多路复用接口（例如 select）中。它们对于通知应用程序接收队列中数据包的通用可用性以及作为高优先级队列中数据包接收和 I/O 完成的轻量级通知机制都很有用。

这种设计产生了一个协议栈，通过使用描述符环作为缓冲区，尽可能地将硬件与软件解耦，在高速数据包率下最大化吞吐量并最小化开销，从而实现低延迟。在这个本地接口之上，Arrakis 提供了 POSIX 兼容的套接字。这个兼容层允许 Arrakis 支持未修改的 Linux 应用程序。然而，我们表明，通过使用异步本地接口可以实现性能提升

> 这里的协议栈 和 IX, mTCP 等有什么区别。
>
> 而且 arrakis 还有缓冲区？为什么不像 IX 一样直接去掉用共享内存做零拷贝呢。

<!--

IX/Arrakis would be targeted a datacenter environment. How does this differ from a cloud environment, in terms of tradeoffs IX  or Arrakis can make to provide performance over things like security, or convenience (e.g., in IX, developers must rewrite applications as it is not POSIX-compliant.)

In cloud environments, such as serverless computing, users are relieved from the burden of managing underlying hardware or virtualization techniques. We can focus solely on implementing code and application without needing to understand the underlying infrastructure and customized system API. However, in datacenter environments, operating systems like IX or Arrakis require users to have a deeper understanding of the hardware, and even the APIs these kernels provide. For examples, developers must be aware of various techniques such as zero-copy and how they modified the POSIX API, which are essential for optimizing performance.

Regarding security, I believe that Arrakis and IX offer a higher degree of customization, trading off compatibility and convenience for finer-grained resource management and isolation. While cloud environments also provide robust resource isolation and security, developers generally do not need to consider the underlying infrastructure. Cloud service providers like AWS offer a variety of environments to choose from, such as ARM or x86, allowing users to select based on cost or performance considerations. In contrast, IX requires support for virtualization technologies like Intel VT-x, which might make migration and implementation on platforms like ARM more challenging. Traditional operating systems and Linux or customized Unix offer highly abstracted interfaces that are well-suited for cloud environments.

In terms of performance, I don't necessarily think that Arrakis and IX provide a significant advantage over user-space networking stacks, especially when considering the development complexity. For instance, the performance improvements mentioned for memcached or Redis mentioned in IX and Arrakis papers have to be weighed against the development effort required. As a developer, I might prefer designs that are ready to use out of the box, considering the tradeoff between performance and the ease of development.


 -->
