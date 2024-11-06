+++
title = 'Paper Reading: AIFM: High-Performance, Application-Integrated Far Memory (OSDI 20)'
date = 2024-11-06T11:28:34-05:00
draft = false
tags = ['Paper Reading']
+++

## AIFM: High-Performance, Application-Integrated Far Memory

Disaggregated Memory 领域的论文，之前稍微看了一点，挺有意思的从应用层面用远端内存

[Zhenyuan Ruan](https://zainryan.github.io/), [Malte Schwarzkopf](https://cs.brown.edu/people/malte/) 和 [Adam Belay](http://www.abelay.me/)，zhenyuan 在 osdi 20 中了不少论文，很多都和解耦内存有关，非常有意思。adam 此外还有 caladan 等，太恐怖了。

## Abstract

application-integrated far memory (AIFM)

它通过一个简单的 API 将**远程的“远端”内存提供给应用程序**，并具有高性能。AIFM 实现了与本地 RAM 相同的常见情况访问延迟；它避免了**基于分页的方法所遭受的读写放大**；它允许数据结构工程师构建可远程访问的、混合近/远端内存数据结构；并且它使远端内存对应用程序开发者透明且易于使用。

> 读写放大

将 application-level semantics 暴露给高性能运行时，使得高效的远程内存成为可能。开发者使用 AIFM 的 API 来使分配可远程访问，AIFM 的运行时负责对象的交换进出、预取和内存疏散。

> 程序语义是什么意思

我们通过一个原型 Web 应用程序前端、一个纽约市出租车数据分析工作负载、一个类似 memcached 的键值缓存和 Snappy 压缩来评估 AIFM。将 AIFM 的远程内存添加到这些应用程序中，增加了它们的可用内存而没有性能损失。AIFM 的性能优于 **Fastswap**，这是一个最先进的内核集成、基于分页的远端内存系统，性能提升高达 61 倍。

> 实际上我觉得要对这些应用的类型做个评估，到底是 IO 密集型还是内存密集型
>
> 再就是对于云平台内存超卖，远端内存有这方面的讨论吗？

## Introduction

Google [73] 和阿里巴巴 [46] 的服务器平均内存利用率为 60%，服务器之间的差异很大，而平均 CPU 利用率约为 40%。

当今的操作系统主要通过交换机制支持内存弹性，通过将未使用的物理内存页推送到较慢的内存层（如磁盘或远程内存）来释放 RAM。但操作系统的交换机制在固定且粗粒度上操作，并产生大量开销。为了交换一个页面，操作系统必须处理页面错误，这需要进入内核并等待数据到达

此外，Linux 内核在等待交换数据时会旋转，以避免上下文切换和中断处理的开销。这意味着等待时间（使用 Fastswap 的 RDMA 后端大约为 15-20k 个周期）被浪费了。

我们描述了一种根本不同的方法：应用程序集成的远端内存（AIFM），它将交换绑定到**单个应用程序级别的内存对象**，而不是虚拟内存（VM）页面的抽象。开发者编写可远程访问的数据结构，其支持的内存可以是本地的和“远端”的——即在远程服务器上——而不影响常见情况下的延迟或应用程序吞吐量。当 AIFM 检测到内存压力时，**其运行时会交换出对象**，并将所有指向该对象的指针转换为远程指针。当应用程序解引用远程指针时，**一个轻量级的绿色线程运行时会将对象恢复到本地内存**。运行时的低上下文切换成本允许其他绿色线程在等待周期内进行有效利用，这隐藏了远程访问延迟并保持了高吞吐量。由于这些快速的上下文切换，AIFM 在访问 4KB 对象时比基于页面的方法高出 81%的吞吐量，并且由于 AIFM 避免了放大，它在访问小对象时实现了 6.8 倍的高吞吐量（图 1）。

> user level 所以还是比较好的？不过还得看看具体实现
>
> 至于这里提到的读写放大，应该是说读写小对象但仍然操作 4KB 页面的事情？

AIFM 的编程接口基于四个关键思想：一个快速、低开销的可远程访问指针抽象，一个**无暂停**的内存 evacuator，允许数据结构向运行时传达语义信息的运行时 API，以及一个帮助将轻量计算卸载到远程内存的远程设备接口。这些 AIFM API 允许数据结构工程师**轻松构建混合本地/远程数据结构**，并提供类似于 **C++ 标准库**数据结构的开发者体验。

无暂停的内存疏散器确保应用程序线程永远不会因交换而经历延迟峰值。因为数据结构向运行时传达了它们的语义，AIFM 支持自定义预取和缓存策略——例如，在可远程访问的列表中预取远程数据，并避免污染本地内存缓存的远程数据流。最后，AIFM 的卸载减少了数据移动，并缓解了大多数远端内存系统经历的网络瓶颈。

> 无暂停是怎么实现的，减少数据移动也就是减少读写放大？

Our prototype is limited to unshared far memory objects on a single memory server. Future work may add multi-server support, devise strategies for dynamic sizing of remote memory, or investigate sharing.

> 这限制看着很奇怪，所以没有网络通信？

## Background and Related Work

当今操作系统主要通过将物理内存页交换到二级存储来实现内存弹性，比如磁盘

最近的研究考虑将**交换到更快的内存层或远端内存**，例如主机的**远程内存**或**压缩缓存**。由于交换与内核虚拟内存子系统集成，因此对用户空间应用程序是透明的。但这种透明性 transparency 也迫使交换粒度为最小的虚拟内存原语，即 4KB 页面。**结合小于 4KB 的内存对象，这导致 I/O 放大**：当访问一个对象时，内核必须独立于对象的实际内存大小交换一个完整的 4KB 页面。此外，提供应用程序语义信息，如预期的内存访问模式、适当的预取策略或内存热度，仅限于粗略和不灵活的接口，如 madvise。

> 透明性是什么意思？系统隐藏了具体原理，直接用？

AIFM 以不同于交换的方式使用远端内存，通过在对象粒度而不是页面粒度上操作——这一想法借鉴了**分布式共享内存**（见下文）、内存压缩[75]和 SSD 存储[1]的先前工作。这些研究都指出页面级 I/O 放大是一个关键动机。AIFM 使用智能指针和受 **C++ 弱指针**[69]和 **Folly RCU 保护**[26]启发的解引用范围，提供对远端内存的透明访问。

> 分布式共享内存 有意思，这一篇论文还是蛮偏向存储设计的
>
> 哈哈 RCU 还是没看

Disaggregated and distributed shared memory: 分解内存[58]指的是一种硬件架构，其中快速结构将主机连接到内存池[29, 33]，这可能由集群范围的操作系统管理[33, 66]。分解内存需要尚未投入生产的新硬件。AIFM 专注于当今硬件的软件解决方案。

    分布式共享内存（DSM）通过消息传递提供共享内存的抽象。因为DSM需要一个缓存一致性协议，这会损害性能。

Technologies to access remote data: TCP/IP 是访问远程数据的主导协议，AIFM 目前使用 TCP/IP。TCP/IP 存在更快的替代方案，可以进一步改进 AIFM，但这些技术与 AIFM 的关键思想正交或互补。AIFM 不需要专门的硬件，RDMA 是一种旧技术，最近在以太网上实现了商品化[32]，引起了新的兴趣。许多工作致力于在一般[39, 51, 76]或特定应用程序（如键值存储[38, 49]或数据库系统[11]）中高效使用 RDMA

> 另一个好奇的点是，既然提升到 user-level，为什么不做更深入的用户网络栈呢
>
> 但 AIFM 到底用不用 RDMA？

Abstractions for remote data: 远程过程调用（RPC）广泛用于访问远程数据，包括通过 RDMA[19, 71]或 TCP/IP[37]。而远程数据的数据结构库[4, 15]为开发者提供了映射、集合、多集合、列表和其他熟悉的构造。这与持久内存的数据结构库[59, 62]的精神相似。AIFM 提供了一个更低级别的服务，帮助程序员开发这些数据结构。

I/O amplification:

Garbage collection and memory evacuation: 在 AIFM 中将对象移动到远程内存（“疏散”）与托管语言中的标记-压缩垃圾收集（GC）密切相关。AIFM 旨在通过将**冷但活跃的对象 cold, but live objects**移动到远程内存来增加内存容量，而 GC 则专注于释放死对象的内存

    AIFM没有发明新的疏散算法，而是从GC文献中借鉴了思想，并将其适应于远端内存系统。与GC类似，AIFM利用读/写屏障来维护对象热度

> 读写屏障也是个很有趣的技术

但 AIFM 使用一个字节的热度计数器而不是一个比特标志，允许更细粒度的替换策略。与 AIFM 类似，一些复制收集器通过在 GC 期间分离热数据和冷数据来优化数据局部性，但目标不同的内存层次结构；例如，缓存-DRAM 层次结构[34]，DRAM-NVM 层次结构[5, 79, 80]和 DRAM-磁盘层次结构[14]。最后，内存疏散会干扰用户任务并影响其性能。为了减少干扰，AIFM 采用了与托管语言中的无暂停 GC 算法[20]类似的方法，而不是停止世界的 GC 算法[36]。

> pauseless GC algorithms 不知道是不是 Java 现在用的，但好像也需要一些小暂停，比如 JDK 12 Shenandoah GC 和 ZGC
>
> Cliff Click, Gil Tene, and Michael Wolf. “The pauseless GC algorithm”. In: ACM/USENIX international conference on Virtual execution environments (VEE). 2005.

## Motivation

内核分页机制在访问远端内存的基本成本上引入了大量开销。考虑图 2，它分解了 Linux（v5.0.0）从 SSD 检索交换出的页面的成本。设备的硬件延迟约为 6µs，但由于与锁定（P1，P5）、虚拟内存管理（P2，P3，P5）、记账（P4）和读取 I/O 放大（P3）相关的开销，Linux 需要超过 15µs（2.5 倍）。此外，由于上下文切换的高成本，Linux 在等待数据时会旋转（P3），浪费了 11.7µs 的可能计算时间。

![](https://s2.loli.net/2024/11/07/ztLW3CPFTB2khEU.png)

> 这里的几个 phase 可以详细讲讲
>
> Linux Kernel swapping
>
> P1 page fault, trap
>
> P2 Lock -> PTE 页表获取页面磁盘位置 -> allocate page frame 页面帧 -> swap cache entry
>
> P3 read IO 请求 -> spin 检查 IO 是否完成 -> 页面进入 LRU
>
> P4 cgroup 计数，更新内存使用情况 -> 回收内存如果超过了限制
>
> P5 页面映射 PTE -> 解锁
>
> 但是 AIFM 直接请求 IO，进入 context switch 然后用协程等待，最后上下文切换回去

![](https://s2.loli.net/2024/11/07/SmRIxezWC4w7i9U.png)

## AIFM Design

### Overview

AIFM 面向两个群体：应用程序开发者和数据结构开发者。

对于应用程序开发者来说，使用远端内存编程应用程序应该感觉几乎与使用纯本地数据结构编程相同。特别是，开发者不应该需要知道一个对象当前是本地的还是远程的（即远端内存是透明的）

> 对开发者来说用法和 std 应该一样？

```CPP
RemHashtable<key_t, int> hashtable;
RemArray<data_t> arr;
```

可远程访问的内存数据结构本身（上面的 RemHashtable 和 RemArray）由数据结构工程师编写，他们使用 AIFM 的运行时 API **将可远程访问的内存对象包含在他们的数据结构中**

当**内存变得紧张**时，AIFM 的运行时会将**其中一些内存对象移动到远程内存**；当数据结构需要访问远程对象时，AIFM 运行时会获取它们。数据结构工程师有很大的设计自由度：他们可以完全依赖 AIFM 来获取远程对象，或者他们可以在远程端部署自定义逻辑。

> 这里的移动是怎么发生的？消耗是什么，怎么监控内存紧张呢？是否会被别的修改呢？什么时候回收呢

### Remoteable Memory Abstractions

We designed the abstractions such that they **impose minimal overheads** (as low as three micro-ops) on “hot path” access to local objects, and try to ensure that the “cold path” remote access incurs little latency above hardware limits

#### Remoteable Pointers

Memory representation. 唯一可远程访问指针（对应于 C++的 std::unique_ptr）的大小与普通 64 位指针相同，而共享指针（对应于 C++的 std::shared_ptr）的宽度为 128 位。图 4 显示了可远程访问唯一指针的内存布局。根据可远程访问指针是本地还是远程，我们采用不同的格式。如果内存是本地的（图 4a），指针在其低 47 位中包含一个虚拟内存地址（足以表示用户空间地址），并在高 17 位中包含控制位，包括标准的脏位（D）和存在位（P）（参考页表）。它还包含用于跟踪指针是否热（H）以及是否正在并发疏散（E）的位。对于唯一指针，共享位（S）设置为 0。我们将 D、E 和 H 位按字节对齐，允许每个位由变异器和运行时疏散器并发且原子地访问，因为字节是最小的读/写单位。

API: 清单 1 显示了可远程访问唯一指针的 API（共享指针的 API 大致相同）。RemUniquePtr 有两个构造函数：一个用于已经本地的对象，另一个用于当前远程的对象。第二个构造函数允许数据结构形成指向当前远程对象的可远程访问指针。这有助于数据结构工程师从其数据结构中引用远程对象，而无需获取这些对象。

要将远程指针转换为本地指针，程序员通过 deref 和 deref_mut API 方法进行解引用。

![](https://s2.loli.net/2024/11/07/Um6EhjyKFpD59IS.png)

Dereferencing 运行时会检查可远程访问指针的存在位。如果对象是本地的，它将设置热位并返回存储在指针中的地址。否则，运行时会从远程服务器获取对象，设置热位和脏位（在 deref_mut 中），并返回指向数据的本地指针。

#### Dereference Scopes

未来，AIFM 可能会利用静态分析来捕捉生命周期违规，如 Rust 编译器

#### Evacuation Handlers

数据结构开发者通过调用 RegisterEvacHandler 注册他们的疏散处理器。疏散处理器与唯一的数据结构 ID 绑定，每个数据结构在其构造函数中分配该 ID，数据结构工程师必须一致使用该 ID。这样，不同的数据结构或同一数据结构的实例可以在同一个应用程序中共存，而运行时会调用适当的处理器

#### Remote Devices

#### Semantic Hints

Hotness tracking 内存疏散器使用此热度信息来确保频繁访问的对象是本地的

Prefetching AIFM 包含一个库，数据结构可以使用该库来维护每个线程的解引用位置历史窗口，并使用有限状态机（FSM）预测未来的访问

Nontemporal Access 对于没有时间局部性的对象的可远程访问指针，限制用于存储其对象数据的本地内存是有意义的

> 没有时间局部性，就是未来可能不再访问？可以立即回收

> 虽然很详细，但我觉得这部分讲的还是不够清楚不够直观

## AIFM Runtime

AIFM 的运行时建立在“绿色”线程（轻量级用户级线程）、内核旁路 TCP/IP 网络栈和无暂停内存疏散器之上。应用程序将运行时链接到其用户空间进程中。这使我们能够与 AIFM 的抽象共同设计运行时，并提供高性能的远端内存，而不依赖于任何操作系统内核抽象。

### Hiding Remote Access Latency

现有的操作系统内核线程支付高上下文切换成本：例如，在 Linux 上，重新调度任务大约需要 500ns。这些成本是远程内存延迟的非平凡部分，因此 Linux 和 Fastswap 采用了一种设计，即在等待网络响应时忙等待[6]。这避免了上下文切换开销，但也浪费了几个微秒的处理时间。这种方法还对网络提供商施加了巨大压力，要求支持更低的延迟，以减少浪费的周期[9, 28]。AIFM 采用了不同的方法：它依赖于低开销的绿色线程，在等待远程数据获取时执行应用程序工作。

> 其实是很简单的思路，用协程，异步 IO
>
> 但是缺点呢，这些协程怎么调度呢

### Remoteable Memory Layout

对于 AIFM 管理的本地内存，其运行时采用了日志结构内存[63]的思想，将本地可远程访问内存按日志粒度分割和管理

### Pauseless Memory Evacuator

运行时的内存疏散器将冷对象移动到远程服务器

1. Log Selection Phase.
2. Concurrent Marking Phase
3. Evacuator Waiting Phase.

> 好奇这种阶段要不要锁，还是说使用读写屏障 + RCU + CAS 就够了

mutator: 如果变异器线程随后解引用指向运行时正在疏散的对象的指针，变异器会看到疏散位已设置。一个简单的方法是此时阻塞变异器线程

> “变异器”（mutator）是指应用程序线程?
>
> Consistent with literature on garbage collection (GC), we refer to normal application threads as mutator threads i

1. Concurrent Evacuation Phase.

### Co-design with the Thread Scheduler

当运行时面临内存压力时，疏散是一项紧急任务。使用简单的线程调度器，疏散可能会被变异器线程饿死，导致内存不足错误和应用程序崩溃。

我们需要解决两个挑战。首先，**大量变异器线程可能比疏散更快地分配内存**。其次，疏散有时会在变异器线程的解引用范围内阻塞，这造成了一个困境。一方面，调度器需要执行变异器线程，以便它们可以解除疏散的阻塞。另一方面，执行变异器线程可能会消耗更多内存。

## Remoteable Data Structure Examples

> 略

## Implementation

AIFM 的实现包括核心运行时库（§5）和数据结构库（§6）。核心运行时建立在 Shenango[55]之上，以利用其快速的用户级线程运行时和 I/O 栈。

> 怪不得说这么多线程，用了 Shenango

我们将两个远端内存后端集成到 AIFM 中：一个基于 DPDK 的 TCP 栈的远程内存服务器，以及一个使用 SPDK 存储栈的 NVMe SSD。与远程内存后端不同，SSD 后端不支持活动的远程组件（因为存储驱动器没有通用计算单元），并且由于其固定块大小的限制，存在固有的 I/O 放大。我们的评估主要集中在远程内存后端。

当前实现有一些限制。首先，我们不支持 TCP 卸载或 RDMA，这会减少我们运行时的 CPU 开销。

## Evaluation

### End-to-end Performance

Web 服务

其次，我们还移植了一个开源的 C++ DataFrame 库[16]，其接口类似于 Pandas[56]，以了解所需的移植工作量和 AIFM 对现有应用程序的性能。

我们将两种 AIFM 设置（带有和不带有数组元素的非临时解引用）与 Fastswap[6]和理想化的基线（所有 26GB 都在本地内存中）进行比较。AIFM 的良好结果将显示其性能优于 Fastswap，非临时数组访问的好处，**并且性能不比将所有数据保存在本地内存中低很多。**

![](https://s2.loli.net/2024/11/07/q7TDgtKA3xMIvnE.png)

在图左侧（5%本地内存），AIFM 在哈希表（52%）和数组（89%）中看到**高缺失率**，因为 AIFM 的**非临时数组解引用**确保大部分本地内存专用于哈希表条目。相应地，数组缺失率下降得更慢，并与可用本地内存成比例。相比之下，Fastswap（此处未显示）在两个数据结构中都有高缺失率，因为其基于页面的方法管理本地内存效率低下。

> 为什么这里 web 应用都是 8K 对象呢？

### DataFrame Application

我们将一个流行的开源 C++ DataFrame 库[16]移植到 AIFM 的 API 中。该库中使用的主要数据结构是一个存储 DataFrame 列和索引的 std::vector，我们将其替换为启用了 AIFM 的等效数据结构。此外，我们还增加了对卸载关键操作的支持，这些操作计算强度低但内存访问频率高。我们通过使用 AIFM 的远程设备 API（§4.2.4）卸载三个操作来实现这一点。Copy 和 Shuffle 操作复制一个向量（即 DataFrame 列），Shuffle 还通过另一个列中的索引位置重新排序行；Aggregate 计算聚合值（总和、平均值等）。这三种操作用于五个 DataFrame API 调用，包括过滤器、列创建、排序和聚合（表 1）。为了实现足以运行纽约市出租车行程分析工作负载[53]的覆盖范围，我们在 DataFrame 库（24.3k 行代码）中修改了 1,192 行代码，并编写了 233 行远程设备代码。这些修改由一位作者花费了大约五天时间。

> 这修改挺累的

图 7 显示了结果。即使只有 1GB 的本地内存（3.2%），AIFM 也能达到内存吞吐量的 78%，并且从大约 20%（6GB）的本地内存开始超过理想性能的 95%。相比之下，Fastswap 在 1GB 时仅达到内存性能的 20%，并且只有在超过 90%的工作集在本地内存中时才接近它。AIFM 的高性能来自于避免 Fastswap 的页面错误开销，并通过卸载计算强度低的操作来减少昂贵的网络数据移动。在没有卸载的情况下，AIFM 优于 Fastswap，直到 60%的工作集在本地，因为 Fastswap 会频繁发生次要错误。超过 60%后，Fastswap 的错误率下降到足以使大多数内存访问优于 AIFM 对计算强度低的操作的解引用时间开销（例如，内存复制）。将这些操作卸载到远程侧有助于 AIFM 避免这种成本，而高计算强度的操作摊销了解引用成本并在本地发生。我们还为 AIFM 原型化了一个批处理 API，当无法卸载时，该 API 在向量元素组之间摊销了解引用开销，并发现它将没有卸载的 AIFM 吞吐量提高到内存吞吐量的 60-80%。我们相信这可能是 AIFM API 的一个很好的未来补充，以加速必须在本地执行的计算强度低的操作。

图 8 分解了卸载的影响。卸载 Copy 贡献了最大的吞吐量增益（18%-38%）；卸载 Shuffle 贡献了 2.9%-13%；卸载 Aggregate 贡献了 4.5%-12%。这些结果表明，AIFM 在本地内存较小的情况下为实际工作负载实现了高性能，并且当工作负载包括计算强度低的操作时，AIFM 的操作卸载对于良好的性能至关重要。

> AIFM 的高性能来自于避免 Fastswap 的页面错误开销，并通过卸载计算强度低的操作来减少昂贵的网络数据移动。
>
> 怎么理解？
>
> 卸载 copy, shuffle, agg 到远端内存感觉不错
>
> 但没有启用 RDMA 还是很可惜的
