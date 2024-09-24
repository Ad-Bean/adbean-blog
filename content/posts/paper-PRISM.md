+++
title = 'Paper Reading: Rethinking the RDMA Interface for Distributed Systems'
date = 2024-09-22T16:12:51-04:00
draft = false
tags = ['Paper Reading']
+++

## Rethinking the RDMA Interface for Distributed Systems

对 RDMA 没什么了解

> Linux 高性能网络详解：从 DPDK、RDMA 到 XDP，这本书看评价很不错，有机会可以看看
>
> DMA: IO 设备比如网卡，需要从内存拿发送的数据，需要告诉 CPU 从内存缓冲区复制，许多复制会让 CPU 阻塞。DMA 控制器，可以专门读写内存，绕开了 CPU，但 DMA 控制器一般和 IO 设备一起？
>
> RDMA：绕开 TCP/IP 协议栈来读取远端的内存。传统网络 A 发送消息给 B，A 内存数据复制到 B 的内存，需要 CPU 中断、协议栈处理等等，RDMA 则可以直接从内存复制数据，用硬件组装后发送到对方网卡。
>
> RDMA 协议：InfiniBand

## Abstract

RDMA + distributed systems

However, most of the distributed protocols used in these systems cannot easily be expressed in terms of the simple memory READs and WRITEs provided by RDMA

> 引入额外的协议支持 RDMA 还是放弃 RDMA，additional round trip 是什么意思

本文认为，RDMA interface 的扩展可以解决这一难题。我们介绍了 PRISM 接口，它添加了四个新的原语: indirection, allocation, enhanced compare-and-swap, and operation chaining。这些提高了 RDMA 接口的表达能力，同时仍然可以使用相同的底层硬件特性来实现。我们通过使用 PRISM 原语设计三个几乎不需要服务器端 CPU 参与的新应用程序来展示它们的实用性: (1) PRISM-KV，一个键值存储; (2) PRISM-RS，一个 replicated block store; (3) PRISM-TX，一个分布式事务协议。使用基于软件的 PRISM 原语实现，我们表明这些系统优于先前基于 RDMA 的等效系统

> 扩展 RDMA 接口
>
> RDMA 应该是分布式存储里的热门方向

## Introduction

reduce the CPU cost of packet processing

This makes RDMA, which provides a standard, accelerated interface for one host to directly access another’s memory, appealing: **hardware** RDMA implementations **bypass the host CPU** entirely [28], and even **software implementations** offer significant performance improvements by simplifying the network stack and reducing context-switching overhead

> 软件绕过协议栈减少上下文切换 + 硬件绕过 CPU

A common theme is that adapting applications to run on RDMA requires complex—and costly—contortions.

Consequently, many RDMA applications are forced to **add extra operations**, i.e., extra network round trips, to their protocols, sacrificing some of the latency benefits

> network round trips 有例子吗，引用了一篇 RDMA KV store Christopher Mitchell, Yifeng Geng, and Jinyang Li. 2013. Using One-sided RDMA Reads to Build a Fast, CPU-efficient Key-value Store. In Proceedings of the 2013 USENIX Annual Technical Conference USENIX, San Jose, CA, USA
>
> 额外的 RTT 是说需要计算吗，A->B 请求生成数据，B 生成数据存在内存，A RDMA READ

其他一些则采用混合设计，需要在某些操作中涉及应用程序 CPU[10, 31, 43]，或者在某些情况下，仅仅使用 RDMA 来实现更快的消息传递协议[15, 34]，从而抵消了 CPU 旁路的益处。

> 如果用 RDMA 做消息传递是不够高效的？多次 RTT。为什么说 negating the benefits of CPU bypass

本文认为，超越基本的 RDMA 接口对于实现构建低延迟系统的全部网络加速潜力是必要的。RDMA 接口最初是为了支持并行超级计算应用而设计的，但它不能满足当今分布式系统的需求。我们展示了通过使用一些额外的原语来扩展接口，完全使用远程操作来实现复杂的分布式应用程序(如复制存储 replicated storage)成为可能。

> 存储副本，冗余？

我们在本文中的目标是确定 RDMA 接口的一组通用(即非特定于应用程序的)扩展，允许分布式系统更好地利用 RDMA 硬件的低延迟和 CPU 卸载能力 better utilize the low latency and CPU offload capabilities。我们的提议是 PRISM 接口。PRISM 扩展了 RDMA 读/写接口，增加了四个基本元素: indirection, allocation, enhanced compare-and-swap, and operation chaining. 结合起来，它们支持常见的远程访问模式，比如数据结构导航、异地更新和并发控制。我们认为 PRISM API 非常简单，可以在 RDMA 网卡中实现，因为它重用了现有的微架构机制。我们还在一个基于软件的网络堆栈原型中实现了 PRISM 接口，该原型使用专用的 CPU 核心来实现远程操作，灵感来自于 Google 的 SNAP 网络堆栈所采用的方法

> Google SNAP, Scalable Network Appliance Platform
>
> 应该也是用户网络栈

1. existing RDMA interface leads to extra protocol complexity for distributed systems.
2. PRISM interface, which extends RDMA with additional primitives to support common patterns in distributed applications
3. three sophisticated applications—key-value stores, replicated block storage, and distributed transactions can be implemented entirely using the PRISM interface.
4. software-based prototype of the PRISM interface

> 尽管相对于 NIC 存在额外的性能开销，但在 PRISM 之上构建的应用程序仍能实现延迟和吞吐量的优势
>
> 什么样的 tradeoff?

## Background and Motivation

RDMA 是一个广泛部署的[13,26]网络接口，允许远程客户端直接在远程主机上**读写内存**，完全绕过远程 CPU

### RPCs vs Memory Accesses: The RDMA Dilemma

RDMA 提供了两种类型的操作。Two-sided 双边操作具有传统的消息传递 message-passing 语义。SEND 操作将消息传输到远程应用程序，该应用程序调用 RECEIVE。One-sided 单边操作允许主机在远程主机（在预注册的区域中）上进行内存的读取或写入。系统社区中关于是否使用单边或双边操作的争论一直存在 [16, 19, 43]。One-sided 单边操作更快且更节省 CPU 资源，但仅限于简单的读写操作。双边消息传递由于允许在两端进行处理，尽管通信操作本身较慢，但总体上可能会产生更快的系统。

tradeoff: 以 Pilaf [31]为例，这是一个早期的基于 RDMA 的键值存储系统。Pilaf 在哈希表中存储键值对象的指针，实际数据存储在单独的 extents 结构中。这两个结构都通过 RDMA 暴露出来，因此客户端可以通过远程读取哈希表，然后使用指针进行远程读取到 extents 存储中来执行键值查找。这不需要服务器端的 CPU 参与，但**需要两次往返**。使用双边操作构建的传统键值存储实现只需要一次往返，但每次操作都需要 CPU 参与。

> 为什么传统 kv 只需要一次 rt 但需要 cpu
>
> pilaf 应该是一次读哈希得到指针，一次读 extents 得到数据
>
> 双边：一次请求，返回数据，但是 cpu 处理？

系统设计者的困境在于，是构建一个基于读写操作的更复杂协议，还是构建一个使用消息传递的更简单协议？换句话说，是执行两次（或更多次）单边 RDMA 操作更快，还是执行一次 RPC 更快？在 RDMA 的早期，选择是明确的：RDMA 操作比 RPC 快大约 20 倍 [31]。

> 本文的 tradeoff 到底是什么？
>
> pilaf 是单边 one-sided: 多次 rt，基于读写操作的更复杂协议
>
> two (or more) one-sided RDMA operations: 消息传递更简单协议，传统 kv，一次 rt，需要 cpu，
>
> single RPC：RDMA 比 rpc 快 20 倍

随着后续工作显著降低了双边 RPC 的成本 [16, 19]，并且 RDMA 在更大规模、更高延迟的环境中部署 [12, 13]，选择使用哪种操作变得更加复杂。

我们在两台通过 40 Gb 以太网连接的服务器上测量了**单边 RDMA 操作**与使用 eRPC [16]实现的双边 RPC 的性能

> 这论文感觉很多概念写得不太清楚，多次 rt 肯定简单吧，早期的做法也是多次 rt？

### Principles for Post-RDMA Systems

大多数关于 RDMA 系统的工作假设我们仅限于当前的 RDMA 读/写接口。如果我们能够扩展 RDMA 接口呢？硬件供应商 [29] 和软件实现 [26] 已经以临时方式添加了各种新操作

Navigating data structures: RDMA 支持在**已知大小和位置**的情况下进行远程读取。大多数应用程序，如 Pilaf，使用更复杂的数据结构来构建索引、存储可变长度的对象，并帮助处理并发更新。遍历这些结构需要多次 RDMA 读取。能够对指针执行间接操作可以消除其中一些往返。

> 如果不知道位置和大小呢

Supporting out-of-place writes: 修改远程数据结构尤其具有挑战性，因为读取可能**同时发生**。为了避免随之而来的一致性问题，许多系统 [10, 31, 32] 仅从服务器 CPU 执行写入。我们的目标是构建能够使用 RDMA 操作处理读取和写入的系统。为此，我们提倡一种设计模式，其中新数据被异地写入单独的缓冲区，然后指针被原子更新以从旧值切换到新值——一种类似于并发编程中的读-复制-更新 [27] 的方法。实现这一点需要新的 RDMA 支持，以将数据写入新缓冲区，并原子更新指向其位置的指针。

> RDMA KV 是如何处理的？本文看上去用了 RCU(Read-copy update) 应该很类似 Copy On Write COW？
>
> COW 存在的问题是什么时候回收，RCU 使用 RCU-sync + CoW 来解决。
>
> https://aandds.com/blog/linux-rcu.html RCU 的读性能比“读写锁”要好，它适应满足两个条件，读多写少，没有强一致要求 。
>
> 但 KV 一般会存在写多读少的情况？

Optimistic concurrency control: 复杂数据结构的更新需要同步。虽然今天可以使用 RDMA 实现锁 [44, 45]，但性能损失可能很大。扩展 RDMA 的比较-交换功能将使我们能够实现复杂的、基于版本的**乐观并发控制** [21]，这种方法与我们的异地更新方法非常契合。

> 好奇 RDMA + 锁的性能会差多少
>
> RCU 还有个关键点应该是链表 + 内存屏障？
>
> Linux 内核也有 RCU，这能复用吗
>
> https://wangxu.me/translation/2008/06/01/what-is-rcu/index.html
>
> https://www.kernel.org/doc/html/next/RCU/whatisRCU.html
>
> 过两天学一下顺便写一下

Chaining operations: 一个常见的主题是应用程序需要执行复合操作，其中一个操作的参数依赖于前一个操作的结果——读取一个指针，然后读取它指向的值，或者写入一个对象，然后交换指针。今天，这需要将中间结果返回给客户端并执行新操作——以及另一个网络往返。如果我们有一种方法可以将操作链接起来，使一个操作依赖于另一个操作，但仍然在一次往返中执行它们，我们可以避免这种开销。

> 是没太懂这种使用场景，有具体例子吗

### The Case for an Extended Interface

扩展 RDMA 接口

另一种方法是 allow applications to provide their own code that runs on the remote host，即将自定义应用程序逻辑部署到智能网卡（Smart NICs）

我们主张采用一组简单、通用的原语。这种简单的扩展可能对更多的应用程序有用，无论是当前的还是未来的。简单性在这里也有助于实现：我们认为我们提出的原语可以添加到**基于软件的网络堆栈**、**可重配置的智能网卡**，甚至是未来的固定功能网卡中。本文的其余部分提出了这些通用扩展，并展示了它们对于常见应用程序的实用性。

> 还是有意思的，可以用在自定义网络栈或者 NIC

## PRISM Interface

为了解决使用 RDMA 构建分布式系统所固有的挑战，我们提出了一种扩展的网络接口，称为 PRISM（用于系统内存远程交互的原语）。PRISM 在现有的 RDMA 接口中增加了四个额外的功能。这些功能旨在支持我们在实现分布式协议时观察到的常见模式。PRISM 的接口设计基于三个原则：（1）通用性——它们不应编码特定于应用程序的功能；（2）最小接口复杂性；（3）最小实现复杂性，这使得性能快速且可预测，并便于在各种平台上实现，包括未来的 NIC ASIC。

### Indirect Operations

许多 RDMA 应用程序需要遍历远程数据结构。这些结构使用间接寻址来实现许多目的：提供索引、支持可变长度数据等。目前，跟随**指针**需要额外的往返。

PRISM 允许 READ、WRITE 和比较-交换（CAS）操作接受间接参数。这些操作的目标地址可以解释为指向实际目标的指针地址。此外，WRITE 或 CAS 操作的数据可以从服务器端内存位置读取，而不是从 RDMA 请求本身读取。

对于 READ 和 WRITE 操作，目标地址可以选择性地解释为⟨ptr, bound⟩结构。在这种情况下，操作长度限制为 bound 或客户端请求长度中较小的一个。这支持可变长度对象：客户端可以执行具有较大长度的读取，但只接收实际存储的数据量。

PRISM 中的间接操作重用了现有的 RDMA 安全机制，确保远程客户端只能操作主机内存中已被授予访问权限的区域。要使用间接操作访问内存区域，客户端必须包含主机在首次向 NIC 注册区域时生成的 rkey。如果目标地址或目标地址指向的位置位于具有不同 rkey（或未注册）的内存区域中，则主机将拒绝该操作。

> 应该和 RDMA 几乎一样，多了间接参数，比较-交换（CAS）可能是新的？为什么可以从服务器端内存读取是什么。

![](https://s2.loli.net/2024/09/23/wJDWPlX3I59CUeV.png)

### Memory Allocation

使用现有的 RDMA 接口修改数据结构尤其具有挑战性：对象必须写入固定的预分配内存区域，这使得处理可变大小的对象或异地更新变得困难。需要的是一种内存分配原语。PRISM 提供了一种内存分配原语，它分配一个缓冲区并返回指向其位置的指针。

> RDMA 都要注册内存区域吧，ibv_reg_mr 获得 rkey，PRISM 到底有什么区别，客户端可以通过一次间接读取操作获取链表中的多个节点，而不需要多次往返？
>
> RDMA 读取链表，传统需要多次往返，READ 读取和下一个指针，然后重复？
>
> PRISM 可以读多个？为什么啊

要使用 PRISM 的分配原语，服务器端进程将缓冲区队列（在 RDMA 术语中称为“发布”）注册到 NIC。当 NIC 从远程主机接收到 ALLOCATE 请求时，它从**空闲列表中弹出一个缓冲区**，将提供的数据写入缓冲区，并返回地址。这种操作在与 PRISM 的请求链接机制结合使用时特别强大，如下所述：PRISM 客户端可以分配一个缓冲区，写入其中，然后通过 CAS 将其指针安装到另一个数据结构中。

PRISM 在 NIC（或软件网络堆栈）数据平面执行内存分配。然而，内存注册由服务器 **CPU** 完成。这是必要的，因为注册内存需要与内核交互以识别相应的物理地址并固定缓冲区。由于服务器 CPU 参与其中，因此对于应用程序重用这些缓冲区的正确性至关重要，只有在并发 NIC 操作完成后，回收的缓冲区才能添加回空闲列表。虽然这简单地将 NIC 和服务器 CPU 同步的负担转移到原语的实现上，但它重要的是将这种同步从应用程序的常规路径中移除。

管理客户端分配的内存可能具有挑战性；这是通过远程访问修改状态的应用程序所面临的根本挑战。具体的内存管理策略由应用程序自行决定。本文中的应用程序使用客户端检测对象何时不再使用，例如，当先前版本已被替换时。它们将未使用的缓冲区报告给在服务器上运行的守护进程（通过传统的 RPC），该守护进程将其重新注册到 NIC 的空闲列表中；可以在客户端和服务器端使用批处理来最小化开销。另一种受垃圾收集启发的方法是服务器端应用程序代码定期扫描数据结构以识别可以回收的缓冲区。

我们的分配器设计有意保持简单，仅从特定队列中分配第一个可用的缓冲区。我们选择这种设计而不是更复杂的分配器，因为（如§4.2 所述）现有的 RDMA NIC 已经具有实现它所需的硬件支持。一个结果是，使用整个预分配缓冲区分配内存会引入空间开销。应用程序可以通过注册包含不同大小缓冲区的多个队列并选择适当的队列来最小化这种影响。例如，使用大小为 2 的幂的缓冲区可以保证最大空间开销为 2 倍。纯软件实现可能会选择使用更复杂的分配器。

> 支持 variable-sized objects or out-of-place updates.
>
> pre register a queue of buffers

### Enhanced Compare-And-Swap

原子比较-交换（CAS）是并行系统中更新数据的一个经典原语。RDMA 标准已经提供了一个原子 CAS 操作 [30]，但它受到高度限制：它进行单一的相等性比较，然后交换一个 64 位的值。根据我们的经验，这对于实现高性能应用程序（包括§6–8 中的那些）是不够的。事实上，除了实现锁 [33, 45] 之外，很少有应用程序今天使用 RDMA 原子操作。虽然可以使用锁来实现任意复杂的原子操作，但这需要多次**昂贵的往返**并增加争用。

为了解决这个问题，我们通过三种方式扩展了 CAS 原语。首先，我们采用了 Mellanox RDMA 设备当前提供的**扩展原子接口** [30]，该接口允许 CAS 操作最多 32 字节，并使用单独的位掩码来比较和交换参数，使得可以比较结构中的一个字段并交换另一个字段。其次，我们为目标地址或比较和交换值引入了间接寻址（§3.1）。我们不保证间接参数指针的解引用是原子的——只有 CAS 本身是原子的——但这允许我们从内存中加载参数值。最后，我们在比较阶段提供了对算术比较操作符（大于/小于）的支持，除了按位相等性之外。这支持了更新版本化对象的常见模式：检查新版本是否大于现有版本，如果是，则更新版本号和对象。

PRISM 的 CAS 操作相对于其他 PRISM 操作是原子的。与现有的 RDMA 原子操作一样，它们不保证与并发 CPU 操作的原子性。

### Operation Chaining

分布式应用程序通常需要执行一系列**数据依赖**的操作来读取或更新远程数据结构。例如，它们可能希望分配一个缓冲区，写入其中，然后更新一个指针以指向它。目前，每个操作必须返回到客户端，然后才能发出下一个操作。PRISM 提供了一种链接机制，允许像这样的复合操作在一次往返中执行。

Conditional operations: RDMA 通常不保证操作按顺序执行。我们添加了一个条件标志，延迟执行操作，直到来自同一客户端的前一个操作完成，并且仅在前一个操作成功时才执行。比较失败的 CAS 操作在这里被视为不成功。

Output redirection: 我们引入了另一个标志，指定操作的输出（READ 或 ALLOCATE）应写入指定的内存位置，而不是发送给客户端。该内存位置通常是每个连接的临时缓冲区。例如，可以执行 ALLOCATE，将其输出重定向到临时位置，然后使用条件 WRITE 将指向新分配缓冲区的指针存储在其他地方。

> kv 确实存在先读后写的操作，这个条件标志是什么，per-connection temporary buffer

### Discussions

PRISM 原语共同实现了§2.2 和§2.3 中的目标。间接操作减少了导航数据结构所需的往返次数。ALLOCATE 和增强的 CAS 原语，结合链接，支持异地更新：应用程序可以在一次往返中 ALLOCATE 一个新缓冲区，将数据写入其中，并使用 CAS 将指向它的指针安装到另一个结构中。最后，CAS 操作的灵活性使得实现基于版本的并发控制机制成为可能。

## PRISM Implementation

PRISM 的 API 由简单的原语组成，以便它们可以轻松地添加到各种 RDMA 实现中。为了评估它们在构建分布式应用程序中的有效性，我们构建了一个基于软件的实现（§4.1）。我们还分析了在 NIC 中实现 PRISM 的可行性（§4.2）。我们评估了软件实现的性能和硬件实现的预期收益，以及智能 NIC 方法（§4.3）。

### Software PRISM Implementation

受 Google 的 Snap [26]启发的软件实现

尽管软件实现使用服务器端 CPU，但在网络堆栈中执行的单边操作通过避免上下文切换和应用程序级线程调度的成本提供了性能优势。

Snap 已经支持（并且其应用程序使用）间接读取。

> 这篇论文看得有点难受，讲解不清楚，前后也不一致，稍微看了下性能也不是提高很多，但是对 RDMA 做分布式事务或存储应该是有蛮多贡献的

原则上，软件实现也可以用于智能 NIC；我们已经在 Mellanox BlueField 上进行了实验。然而，如下所述（§4.3），这种方法的性能低于软件实现，因此我们不主张除非减少 CPU 利用率是主要目标，否则不使用它。

> 为什么呢，一般来说 offload 到 NIC 上应该性能提升会更高吧

### Hardware NIC Feasibility

PRISM 的设计旨在可实现于未来的 RDMA NIC 中。这部分分析必然是推测性的，因为支持 RDMA 的 NIC ASIC 是专有设计，我们没有能力进行扩展。然而，我们认为实现 PRISM 原语是可行的，因为它们重用了当今 NIC 上已经存在的底层机制。

> 讨论了 NIC 怎么做，但是看样子没实现，而且基本都是猜测可行性

### Performance Analysis

PRISM 的性能不仅取决于执行成本，还取决于网络延迟，因为其性能优势来自于消除与 RDMA 相比的网络往返。图 1 通过使用直接网络链接，考虑了 PRISM 的最坏情况。图 2 通过比较 PRISM 的间接读取与执行两次 RDMA 读取的成本，评估了网络延迟的影响。我们考虑了单机架部署（一个交换机）、具有三级交换机层次结构的集群，以及来自实际数据中心 [12] 的延迟数据，这些数据也反映了网络拥塞。在每种情况下，尽管使用 CPU 增加了额外成本，PRISM 的软件实现仍然优于 RDMA 基准。

> 减少 round trip 应该能大幅增加吞吐吧，应该测吞吐要比延迟来的更明显？
>
> 但是论文的测试基本都是延迟，而且 RDMA 不知道为什么明显高很多 基本都是 10+us 而

## Applications Overview

为了展示 PRISM 的潜在优势，我们使用了三个代表性分布式应用程序的案例研究：

PRISM-KV：一个实现 RDMA 读写操作的键值存储系统。（§6）

PRISM-RS：一个实现 ABD [4] 仲裁复制协议的复制存储系统。（§7）

PRISM-TX：一个使用 PRISM 原语实现基于时间戳的乐观并发控制协议的事务存储系统。（§8）

> 比较早的其实就是 FaRM, Pilaf 这几个 RDMA kv
>
> 分布式事务也比较有意思

## PRISM-KV: Key-Value Storage

键值存储是一种广泛使用的基础设施，是利用 RDMA 加速的自然机会。我们首先考虑一个简单的远程、未复制的键值存储，类似于 memcached。它提供了一个 GET/PUT 接口，其中键和值可以是任意长度的字符串。

作为一个使用 RDMA 实现键值存储的挑战示例，再次考虑 Pilaf [31]。Pilaf 仅使用单边操作来实现 GET 操作；PUT 操作通过双边 RPC 机制发送并由服务器 CPU 执行。它存储一个固定大小的哈希表，该表仅包含一个有效位和一个指向键值存储的指针，该存储在单独的 extents 区域中。要执行 GET 操作，Pilaf 客户端计算键的哈希值，并执行单边 READ 到哈希表，然后执行第二个 READ 到它指向的数据。哈希冲突使用线性探测或布谷鸟哈希解决，这可能需要读取额外的指针。

### PRISM-KV Design

单边操作实现了 GET 和 PUT 操作

它维护一个包含指向数据项指针的哈希表索引；这些数据项现在使用 PRISM 的 ALLOCATE 原语分配的缓冲区中。

> 感觉就是加入了更多缓冲区，然后来支持一些同步原语

要执行 PUT 操作，客户端首先如上所述确定正确的哈希表槽。然后，客户端使用 PRISM 原语链写入新值并更新哈希表槽。首先，客户端使用 ALLOCATE 将值写入新缓冲区，并将地址重定向到临时位置。然后，客户端在哈希表槽执行 CAS，如果自客户端确定正确槽以来未更改，则将旧地址替换为存储在临时位置的地址。2 为此最后操作设置了适当的间接位和条件标志。如果 CAS 失败，这表明并发客户端随后用较新的值覆盖了相同的键。如果成功，客户端异步通知服务器将旧版本的缓冲区返回到空闲列表。

PRISM-KV 确保在并发更新期间的正确性，因为对象被写入**单独的缓冲区**，并且这些缓冲区的指针被原子地安装到适当的哈希表槽中。与更新同时进行的读取也不会违反安全性，因为哈希表槽的间接读取保证读取格式良好的地址，并且地址适合缓存行。请注意，刚刚完成更新的客户端可能会尝试释放缓冲区，而间接读取尝试读取它。这对正确性不是问题，因为 PRISM 在将缓冲区添加回空闲列表之前等待并发 NIC 操作完成。

### Evaluation

![](https://s2.loli.net/2024/09/23/g4FE2Z1sN7nKGoO.png)

> 这个结果就看上去好很多，吞吐量在写多的情况下还是很不错的，但是写只要稍微多一些就不行了和 pilaf 没太大差别，主要还是 cow + rcu 的组合不太能适应写多的场景？
>
> 又提了硬件实现，又是预测。。

## PRISM-RS: Replicated Block Storage

块存储 + replica

PRISM-RS 是我们的块存储设计，它保证了线性化，只要不超过 𝑓 个副本失败，就可以保持可用性，并且需要最少的副本 CPU 参与。它通过使用 PRISM 操作实现 ABD [4] 原子寄存器协议的变体来实现这一点。

> ABD 协议 Amotz Bar-Noy 非常早提出的一个分布式同步协议？
>
> 为什么要用这么早的，不能支持更现代的吗

## PRISM-TX: Distributed Transactions

我们能否构建一个分布式事务协议，使用远程操作实现其执行和提交阶段？使用 PRISM 的新原语，特别是增强的 CAS 操作，我们在一个称为 PRISM-TX 的协议中实现了乐观并发控制检查。

> 乐观并发协议，能证明可序列化（但是没写）
>
> 使用了一些简单的测试，YCSB-T low contention

## Conclusion

> 看完也不知道看了什么
>
> 减少网络往返：PRISM 通过在单边操作中引入更复杂的操作，减少了额外的网络往返次数，从而提高了性能。
