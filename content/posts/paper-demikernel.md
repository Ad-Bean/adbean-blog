+++
title = 'Paper Reading: The Demikernel Datapath OS Architecture for Microsecond-scale Datacenter Systems'
date = 2024-09-24T23:02:58-04:00
draft = false
tags = ['Paper Reading']
+++

## The Demikernel Datapath OS Architecture for Microsecond-scale Datacenter Systems

Datacenter system 指的是 MPP 吗？

[Github DemiKernel](https://github.com/microsoft/demikernel) 好像最近还在更新

## Abstract

数据中心系统和 I/O 设备现在以单位微秒的延迟运行，需要 ns 级别的操作系统。传统的基于内核的操作系统带来了难以承受的开销，因此最近的内核绕过操作系统[73]和库[23]从 I/O 数据路径中消除了操作系统内核。然而，这些系统都没有提供一个**通用的数据路径**操作系统的替代品，以满足 μs 规模系统的需要。提出了一种适用于**异构内核旁路设备**和 μs 规模数据中心系统的柔性数据通路操作系统 Demikernel。我们构建了两个原型 Demikernel 操作系统，并展示了移植现有 μs 规模系统所需的最小努力。一旦移植，Demikernel 允许应用程序在具有 ns 级别开销且没有代码更改的异构内核旁路设备上运行

> nanoseconds 比 microseconds 还要小吧，什么样的 os 是 ns 级别的？
>
> 这篇文章和 XDP 的区别是什么？

## Overview

网络往返时间、磁盘访问以及像 Redis 这样的内存系统[80]，可以实现个位数的微秒级延迟。

为了避免成为瓶颈，datapath systems software 必须在亚微秒级——或纳秒级——延迟下运行。

为了最小化延迟，广泛部署的内核旁路设备[78, 16]将传统操作系统内核移动到 control path，并让微秒级应用程序直接执行 datapath I/O.。

内核旁路设备将操作系统的保护功能（例如，隔离、地址转换）offload 到用户空间，以安全地提供用户级 I/O，并且更强大的设备实现一些操作系统管理（例如，网络）以进一步减少 CPU 使用。现有的内核旁路库[57, 23, 44]提供了一些缺失的操作系统组件；然而，没有一个库是通用的、可移植的数据路径操作系统。

> 缺少了什么？我感觉全是不通用、难以移植的吧，除非真的合入 Linux Kernel 比如 XDP

没有标准的数据路径架构和通用的数据路径操作系统，内核旁路对于微秒级应用程序来说很难利用。程序员不希望为不同的设备重新架构应用程序，因为他们可能无法提前知道哪些设备可用。新设备功能似乎每年都在发展，

因此，微秒级应用程序需要一个具有可移植操作系统的数据路径架构，该操作系统实现了常见的操作系统管理：存储和网络堆栈、内存管理和 CPU 调度。除了支持具有纳秒级延迟的异构设备外，数据路径操作系统还必须满足微秒级应用程序的新需求。例如，零拷贝 I/O 对于减少延迟非常重要，因此微秒级系统需要一个具有清晰零拷贝 I/O 语义和内存管理的 API，以协调应用程序和操作系统之间的共享内存访问。同样，微秒级应用程序每几微秒执行一次 I/O，因此应用程序工作和操作系统任务之间的细粒度 CPU 多路复用也是至关重要的。

> 所以本文的最重要的点是可移植性？
>
> 零拷贝、更好的内存管理、CPU 调度，应该大家都是相同的

本文介绍了 Demikernel，一个为异构内核旁路设备和微秒级内核旁路数据中心系统设计的灵活数据路径操作系统和架构。Demikernel 定义了：

(1) 新的数据路径操作系统管理功能，适用于微秒级应用程序，

(2) 一个新的可移植数据路径 API（PDPIX），以及

(3) 一个灵活的数据路径架构，用于最小化异构设备之间的延迟。Demikernel 数据路径操作系统与传统的控制平面内核（例如，Linux 或 Windows）一起运行，并由具有相同 API、操作系统管理功能和架构的可互换库操作系统组成。

每个库操作系统都是设备特定的：它在可能的情况下**卸载到内核旁路设备**，并在用户空间库中实现剩余的操作系统管理。这些库操作系统旨在简化跨异构内核旁路设备的微秒级数据中心系统的开发，同时最小化操作系统开销。

> 需要设备支持吗

Demikernel 遵循从内核导向的操作系统向库导向的数据路径操作系统的趋势，这一趋势是由越来越高效的 I/O 设备导致的 CPU 瓶颈所驱动的。它不是为那些直接访问内核旁路硬件（例如，HPC [45]，软件中间件[90, 72]，RDMA 存储系统[17, 98, 7, 101]）而设计的系统，因为它施加了一个隐藏更复杂设备功能（例如，单边 RDMA）的通用 API。

本文描述了两个用于 Linux 和 Windows 的原型 Demikernel 数据路径操作系统。我们的实现主要使用 Rust，利用其在数据路径操作系统堆栈中的内存安全优势。我们还描述了**新的零拷贝**、**纳秒级 TCP 堆栈**和**内核旁路感知内存分配器**的设计。我们的评估发现，使用 Demikernel 构建和移植应用程序很容易，I/O 处理延迟约为 50 纳秒每 I/O，与直接使用内核旁路 API 相比，峰值吞吐量开销为 17-26%

> **新的零拷贝**、**纳秒级 TCP 堆栈**和**内核旁路感知内存分配器**
>
> 50ns 有比较吗，吞吐量 10-20% 是什么带来的 tradeoff 呢

## Demikernel Datapath OS Requirements

现代内核旁路设备、操作系统和库消除了**操作系统内核在 I/O 数据路径中的作用**，但没有完全替代其所有功能，从而在内核旁路操作系统架构中留下了一个空白。这个空白暴露了一个关键问题：微秒级系统的正确数据路径操作系统替代方案是什么？本节详细介绍了微秒级系统和异构内核旁路设备的需求，这些需求推动了 Demikernel 的设计。

> heterogenous 这个词其实之前看论文很少注意，有没有具体场景？

### Support Heterogenous OS Offloads

![](https://s2.loli.net/2024/09/25/mE5tCTY2NsBhlVF.png)

> 对比了 arrakis：app 绕过 kernel，通过 libOS 来做 user I/O，其他 caladan/eRPC 也很类似
>
> DemiKernel 应该和 arrakis 也差不多吧，

当今的数据路径架构是 ad-hoc 的：现有的内核旁路库[34, 73]在特定的内核旁路设备之上提供了不同的操作系统功能。由于不同的设备卸载了不同的操作系统功能，因此可移植性具有挑战性。例如，DPDK 提供了一个低级别的原始 NIC 接口，而 RDMA 实现了一个具有拥塞控制和有序、可靠传输的网络协议。因此，使用 DPDK 的系统实现了一个完整的网络堆栈，这对于使用 RDMA 的系统来说是不必要的。

> ad-hoc 实习的时候经常见到的词汇，从来没理解，到底是自己定制还是说临时方案，或者是特殊的方案，非通用？

这种异构性源于硬件设计者长期以来一直面临的根本权衡[50, 66, 10]——**卸载更多功能可以提高性能**，但会增加设备复杂性——随着最近的研究提出越来越复杂的卸载功能[52, 41, 86]，这种情况只会变得更糟。例如，DPDK 更通用且广泛可用，但 RDMA 在数据中心内的微秒级系统中实现了最低的 CPU 和延迟开销，因此任何数据路径操作系统都必须可移植地支持两者。未来的 NIC 可能会引入其他权衡，Given this complexity, Demikernel’s design must portably manage complex zero-copy I/O coordi- nation between applications and OS components.

> 所以到底是做一个更好的，还是更通用的？这才是 tradeoff 吗？
>
> complexity 是真的需要考虑的吗 ，RDMA 和 DPDK 使用起来好像并不太复杂，大部分 NIC 应该也支持 RDMA

### Multiplex and Schedule the CPU at μs-scale

微秒级数据中心系统通常每几微秒 us 执行一次 I/O；因此，数据路径操作系统必须能够在类似的速度下**多路复用**和**调度 I/O 处理**和应用程序工作。现有的内核级抽象，如进程和线程，对于微秒级调度来说过于粗粒度 coarse-grained，因为它们会占用整个核心数百微秒。因此，内核旁路系统缺乏通用的调度抽象。

> datapath OS 应该就是专指 kernel bypass 方法

最近的用户级调度器[9, 75]在每个 I/O 的基础上以微秒级分配应用程序工作者；然而，它们仍然使用**粗粒度的抽象**来处理操作系统工作（例如，整个线程[23]或核心[30]）。有些调度器更进一步，采用了微内核方法，将操作系统服务分离到另一个进程[57]或环[3]中以提高安全性。

> 没懂，但这个微内核就是 IX，使用了 Dune 虚拟化进程

微秒级的 RDMA 系统通常交错 I/O 和应用程序请求处理。这种设计使得**调度变得隐式**而不是显式：数据路径操作系统无法控制分配给应用程序与数据路径 I/O 处理的 CPU 周期的平衡。例如，FaRM [17]和 Pilaf [64]在消息到达时总是立即执行 I/O 处理，即使有更高优先级的任务（例如，为可能被丢弃的传入数据包分配新的缓冲空间）。

> 考虑调度是不是这篇论文比较有意思的地方

这些系统都不是理想的，因为它们的调度决策仍然是分布式的，要么在内核和用户级调度器之间（对于 DPDK 系统），要么在代码之间（对于 RDMA 系统）。最近的工作，如 eRPC [34]，已经表明，在单个线程上多路复用应用程序工作和数据路径操作系统任务是实现纳秒级开销所必需的。因此，Demikernel 的设计需要一个微秒级调度抽象和调度器。

> 应该是基于 arrakis 或者类似的架构，记得之前是没太看懂 arrakis 是怎么实现的，应该也是虚拟化技术吧 IOMMU SR-IOV

## Demikernel Overview and Approach

Demikernel 是首个满足微秒级应用程序和内 kernel-bypass devices 需求的“数据路径操作系统”。它引入了新的操作系统特性和新的**可移植数据路径 API**，这些对程序员是可见的，同时还有新的操作系统架构和设计，这些对程序员是不可见的。本节概述了 Demikernel 的方法，下一节将描述对程序员可见的特性和 API，而第 5 节将详细介绍 Demikernel（库）操作系统的架构和设计。

> 什么叫满足内核旁路设备？

### Design Goals

- Simplify μs-scale kernel-bypass system development
- Offer portability across heterogenous devices Demikernel 应该让应用程序无需代码更改就能在多种类型的内核旁路设备（例如，RDMA 和 DPDK）和虚拟化环境中运行。
- Achieve nanosecond-scale latency overheads

> 不太懂这三个目标和以前的有什么太大区别，同样是内核旁路和新的 API，为什么说就更加可移植呢

### System Model and Assumptions

Demikernel 依赖于流行的内核旁路设备，包括 RDMA [61] 和 DPDK [16] NIC 以及 SPDK 磁盘 [88]，同时也适应未来的可编程设备 [60, 58, 69]。我们假设 Demikernel 数据路径操作系统与**应用程序运行在同一个进程和线程中**，因此它们相互信任，任何隔离和保护都由控制路径内核或内核旁路设备提供。这些假设在数据中心中是安全的，因为在数据中心中，应用程序通常自带库和操作系统，数据中心运营商使用硬件强制执行隔离。

Demikernel 使得应用程序内存可以直接通过网络发送，因此应用程序必须仔细考虑敏感数据的位置。如果必要，这一功能可以被关闭。其他技术，如信息流控制或验证，也可以被利用来确保应用程序内存的安全性。

> 其实就是 LibOS？用户态的一个库提供运行运行时吗
>
> 发送内存，是 DPDK 还是 RDMA
>
> 没看懂这个安全的假设

To minimize datapath latency, Demikernel uses cooperative scheduling 因此应用程序必须在紧密的 I/O 处理循环中运行（

> 引入了调度的概念，协作调度应该是适合 IO 密集型的任务

我们的原型目前专注于独立调度单个 CPU 核心，依赖硬件支持进行**多核调度**

### Demikernel Approach

A portable datapath API and flexible OS architecture.

> 新的 API 称之为 PDPIX Portable datapath?

PDPIX 扩展了标准的 POSIX API，以更好地适应微秒级内核旁路 I/O。微秒级内核旁路系统是面向 I/O 的：它们花费大部分时间和内存来处理 I/O。因此，PDPIX 围绕 I/O 队列抽象展开，使 I/O 显式化：它允许微秒级应用程序提交整个 I/O 请求，消除了 POSIX 基于管道的 I/O 抽象的延迟问题。

为了最小化延迟，Demikernel 库操作系统尽可能地将操作系统功能卸载到设备上，并在软件中实现剩余的功能

> 啥意思？

A DMA-capable heap, use-after-free protection

Demikernel 提供了三个新的外部操作系统特性，以简化零拷贝内存协调

（1）具有清晰语义的 I/O 内存缓冲区所有权的可移植 API，（2）零拷贝、DMA 能力的堆，以及（3）零拷贝 I/O 缓冲区的使用后释放（UAF）保护。与 POSIX 不同，PDPIX 定义了清晰的零拷贝 I/O 语义：应用程序在调用 I/O 时将所有权传递给 Demikernel 数据路径操作系统，并且在 I/O 完成之前不会收回所有权。

> 什么叫 clear semantics for I/O memory buffer ownership
>
> 只是语义上的加强？

UAF 保护 (User after free)

> 悬垂指针的问题吧，感觉其实就是 Rust 带来的好处

Coroutines and μs-scale CPU scheduling

内核旁路调度通常以每个 I/O 为基础进行；然而，POSIX API 并不适合这种用途。epoll 和 select 有一个众所周知的“惊群”问题 [56]：当套接字共享时，不可能将事件精确地传递给一个工作者。因此，PDPIX 引入了一个新的异步 I/O API，称为 wait，它允许应用程序工作者等待特定的 I/O 请求，并且数据路径操作系统可以显式地将 I/O 请求分配给工作者。

> thundering herd 多个进程等待一个资源时候，可能会导致惊群，但是其实有别的处理方法比如 epoll_exclusive
>
> 这个 wait 是 rust Tokio? 还是 asyncio 提供的？

Demikernel 使用协程 Coroutines 来封装操作系统和应用程序的计算。协程是轻量级的，具有低成本的上下文切换，并且非常适合 I/O 堆栈通常需要的基于状态机的异步事件处理。我们选择协程而不是用户级线程（例如，Caladan 的绿色线程 [23]），后者可以同样出色地执行，因为协程封装了每个任务的状态，消除了全局状态管理的需要。例如，Demikernel 的 TCP 堆栈为每个 TCP 连接使用一个协程进行重传，这保持了相关的 TCP 状态。

> 协程不就是用户级线程吗

每个库操作系统都有一个集中化的协程调度器，针对内核旁路设备进行了优化。由于在纳秒级 [33, 15] 中断是不可承受的，Demikernel 协程是协作式的，它们通常在几微秒或更短的时间内让出。传统的协程通常通过轮询工作：调度器运行每个协程以检查进度。然而，我们发现轮询在纳秒级是不可承受的，因为大量协程被阻塞在偶发的 I/O 事件上（例如，TCP 连接的包到达）。因此，**Demikernel 协程也是可阻塞的**。调度器将可运行和阻塞的协程分开，并在事件发生时仅将阻塞的协程移动到可运行队列中。

### PDPIX: A Portable Datapath API

PDPIX 是面向队列的，而不是面向文件的

> 不面向文件是什么意思。。fd 变成 队列描述符

I/O Queues: queue() 创建一个轻量级的内存中队列，类似于 Go 通道

> 为什么要把文件变成队列？

Network and Storage I/O:

Memory: 应用程序不为传入的数据分配缓冲区；UAF protection

Scheduling: PDPIX 用异步的 wait\_\* 调用替换了 epoll

> 事件驱动、异步？

## Demikernel Datapath Library OS Design

### Design Overview

每个 Demikernel 库操作系统支持单一类型的内核旁路 I/O 设备（例如，DPDK、RDMA、SPDK），并由一个用于 I/O 设备的 I/O 处理堆栈、一个特定于库操作系统的内存分配器和一个集中化的协程调度器组成。为了同时支持网络和存储，我们将库操作系统集成到一个单一的库中，用于两种设备（例如，RDMAxSPDK）。

Rust 通过语言特性和编译器强制执行内存安全

跨平台的可移植性

Rust 对协程有出色的支持

> demikernel github 还在更新

使用 Rust 的主要缺点是需要许多跨语言绑定，因为内核旁路接口和微秒级应用程序仍然主要用 C/C++ 编写

Demikernel 库操作系统使用 Rust 的 async/await 语言特性 [85] 在协程中实现异步 I/O 处理。Rust 利用对生成器的支持将命令式代码编译成带有转换函数的状态机。Rust 编译器不直接保存寄存器和交换栈；它将协程编译成带有“在栈上”值直接存储在状态机中的常规函数调用 [55]。使用 Rust 的这个关键好处使得协程上下文切换轻量且快速（在我们的 Rust 原型中约为 12 个周期），并帮助我们的 **I/O 堆栈在关键路径上避免真正的上下文切换**。虽然 Rust 的语言接口和编写协程的编译器支持是定义良好的，但 Rust 目前没有协程运行时。因此，我们在每个库操作系统中实现了一个简单的协程运行时和调度器，该调度器针对每个内核旁路设备所需的 I/O 处理量进行了优化。

![](https://s2.loli.net/2024/09/26/V4sRQ5Eu3NnFCaW.png)

> async?
>
> DPDK/RDMA 其实是 c 写的吧，应该也有 rust-implementation
>
> 这里提到的协程运行时很有意思，Go 的应该就是是 GMP M:N 之类的？在 runtime package
>
> 图例看上去是一个更加大一统的架构，而不是很有创新自己的方法，能不能理解为，根据不同的 API 选择不同的 LibOS

### I/O Processing

Demikernel I/O 堆栈共享应用程序线程，并旨在实现运行到完成：快速路径协程处理传入数据，找到阻塞的 qtoken，调度应用程序协程并处理任何传出消息，然后再继续下一个 I/O。同样，Demikernel I/O 堆栈在 push 中内联传出 I/O 处理（在应用程序协程中），并在无错误情况下提交 I/O（到异步硬件 I/O API）。尽管在快速路径和应用程序协程之间会发生协程上下文切换，但它不会中断运行到完成，因为 Rust 将协程切换编译为函数调用。

### Memory Management

每个 Demikernel 库操作系统使用特定于设备的、经过修改的 Hoard 进行内存管理

> Hoard 资源分配器
>
> 替代 malloc？所以这篇文章基本就是缝合吗

对于使用后释放保护，所有 Demikernel 分配器提供了一个简单的引用计数接口：`inc_ref(void _)` 和 `dec_ref(void _)`。请注意，这些接口不是 PDPIX 的一部分，而是 Demikernel 库操作系统的内部接口。库操作系统 I/O 堆栈在发出 I/O 时调用 inc_ref，并在完成缓冲区时调用 dec_ref。如前所述，Demikernel 库操作系统可能需要长时间持有引用；例如，TCP 堆栈只有在接收方确认数据包后才能安全地调用 dec_ref

> 应该自己实现了一个引用计数来做 UAF
>
> 但引用计数会不会不够呢，内核态会不会存在循环引用？

### Coroutine Scheduler

A Demikernel libOS has three coroutine types, reflecting com- mon CPU consumers:
(1) a fast-path I/O processing coroutine for each I/O stack that polls for I/O and performs fast-path I/O processing,

(2) several background coroutines for other I/O stack work (e.g., managing TCP send windows), and

(3) one application coroutine per blocked qtoken, which runs an application worker to process a single reques

每个库操作系统调度器优先运行可运行的应用程序协程，然后是后台协程和始终可运行的快速路径协程，按 FIFO 方式进行

因此，每个调度器一次运行一个协程。我们期望协程设计将扩展到更多核心。然而，Demikernel 库操作系统需要仔细设计以避免跨核心的共享状态，因此我们尚不清楚这是否会成为一个主要限制。

> 跨核心的共享状态，dpdk 又怎么做呢，锁？感觉都是用多队列缓冲

### Network and Storage LibOS Integration

Demikernel 内存分配器为 DPDK 网络或 SPDK 存储 I/O 提供 DMA 能力的内存。我们修改 Hoard，从 DPDK 内存池为 SPDK 分配内存对象，并将相同的内存注册到 RDMA。我们在轮询 DPDK 设备和 SPDK 完成队列之间拆分快速路径协程，以轮询方式分配 CPU 周期的公平份额，在无待处理 I/O 的情况下。未来可以实现更复杂的 CPU 周期在网络和存储 I/O 处理之间的调度。总的来说，网络和存储数据路径库操作系统之间的可移植集成显著简化了在网络和存储内核旁路设备上运行的微秒级应用程序。

## Demikernel Library OS Implementations

We prototype two Demikernel datapath OSes: DEMILIN for Linux and DEMIWIN for Windows

> 其实做 Linux 应该不难，大家都在做。但是做 Windows 的有点厉害吧

DEMIWIN 目前仅支持通过 WSL 的 Catpaw 和 Catnap POSIX 库操作系统实现的 RDMA 内核旁路设备

DEMIWIN 实现

DEMIWIN 是 Demikernel 在 Windows 上的实现，目前仅支持 RDMA 内核旁路设备。它通过 WSL（Windows Subsystem for Linux）使用 Catpaw 和 Catnap POSIX 库操作系统。Catpaw 基于 NDSPI v2 构建

> 不太清楚这里说的支持 windows 到底是 wsl 还是 raw windows

### Catnap POSIX Library OS

### Catmint and Catpaw RDMA Library OSes

Catmint 在 `rdma_cm` [79] 接口之上构建 PDPIX 队列来管理连接，并使用 `ib_verbs` 接口高效地发送和接收消息。它使用双边 RDMA 操作来发送和接收消息，这简化了对 `wait_*` 接口的支持。我们在 NSDPI 之上的 Catpaw 使用了类似的设计。

### Catnip DPDK Library OS

### Cattree SPDK Library OS

> 命名感觉怪怪的，这里又 Cat 什么什么

## Evaluation

### Experimental Setup

### Programmability for μs-scale Datacenter Systems

TURN UDP Relay: Teams / Skype 他将 TURN 服务器移植到 Demikernel 花费了 1 天 [38]，而移植到 io_uring [12] 和 Seastar [42] 各花费了 2 天。最终，他无法使 Seastar 工作，并且遇到了 io_uring 的问题。

Redis:

TxnStore:

> 太长了这篇论文，直接根据 evaluation 做点分析
>
> 两个内核旁路应用程序（testpmd [93] 和 perftest [84]）就是 DPDK 和 RDMA
>
> 三个最新的内核旁路库（eRPC [34]、Shenango [71] 和 Caladan [23]）
>
> 首先比较的是可编程性，用的是 LoC。。第一次见，又不是说差几个数量级，同数量级下就 10% 不到的代码量减少，根本就是语言设计风格和误差吧，让不会用 Rust 的写 DemiKernel 比 C 带来的负担应该更大？不过 C 应该能调用 DemiKernel
>
> 但是让专家来移植一些经典应用来说还是有信服力的
>
> 只不过 redis 用了 epoll，论文换成了队列，解决了一些 epoll 带来的问题，metrics 在 7.5 figure 11 可以看吹的那么好，还加个了 persistent Log 指标，为什么又要比较 disk 性能了。
>
> 处于 POSIX 的性能下降了几乎 30%+，RDMA 又没有参考对比，DPDK 有提升，但也没有参考。

### Echo Application

![](https://s2.loli.net/2024/09/26/Q7hPgl6FSdUKr8s.png)

> 就结果来看，并没有太好，先不说图有点误导人，加上个 everything else 又不说是什么，
>
> 同样 DPDK / Catnip 带来了 2-3 us 的延迟，这又和 motivation 矛盾，而这只是简单的 echo，更复杂的呢？
>
> 论文还提到了跨多核心的问题

## Conclusion

> 集大成的论文，总的来说是不错的论文，恶补 MIT OS 的计划要提上日程了
>
> 但 motivatin 有点弱，通用性和性能是很经典的 tradeoff，但我不认为有很好的通用性，用户要用 dpdk 就直接用 dpdk 官方了，也不会突然转头说要来一个 rdma，同时使用两种的场景会受硬件限制吧
>
> 性能说是没有削弱太多，但给的 evaluation 都没什么说服力，echo 太简单，redis 性能只有 dpdk 不错但没有参照了，不知道是不是我看漏了什么
>
> 其他一些带来的优势更像是语言特性，UAF 之类的。唯一有意思的还是用队列替换 epoll 什么的，加入异步，但感觉也是语言特性。
>
> 零拷贝应该是很有意思的地方，论文说是实现了很不错的零拷贝，但感觉只是语义上的，能不能把 kafka 什么的拉出来做个比较呢
