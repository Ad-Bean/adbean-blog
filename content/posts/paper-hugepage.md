+++
title = 'Paper Reading: Temeraire, Hugepage aware memory allocator'
date = 2024-10-27T13:30:30-04:00
draft = false
tags = ['Paper Reading']
+++

## Beyond malloc efficiency to fleet efficiency: a hugepage-aware memory allocator

tcmalloc 是 google 开发的高性能内存分配器 thread-caching malloc，而 temeraire 是本文提出的，改进 tcmalloc，改进大页分配减少内存碎片。https://github.com/google/tcmalloc

> 简单来说，内存管理如 malloc 在堆上分配内存，使用一个内存池（通常是链表数据结构）管理已分配和未分配的内存块
>
> 分配时，找一个足够大的空闲块分配给用户，否则向系统申请更多内存，malloc 一般会返回对齐的地址的指针
>
> 但 malloc 存在内存碎片的问题，内部和外部碎片，比如 malloc(1) 但其实会分配更大的内存块（对齐），所以有内部碎片
>
> 外部碎片：10k(free) | 5k (busy) | 100k (free) 这时候申请 15k 内存，第一段不连续用不了
>
> 至于 `brk()` 之类的系统调用可以看一些经典八股理解
>
> 小林 coding： https://xiaolincoding.com/os/3_memory/malloc.html
>
> 【操作系统】malloc、free 实现原理： https://imageslr.com/2020/malloc.html

malloc 存在不少问题，不仅是内存碎片的问题，malloc 在多线程环境下表现也不佳，需要用到锁来保护内存分配和释放并避免竞争。而 tcmalloc 采用了线程本地缓存 (Thread Local Cache) 来减少锁竞争，对于小内存块，直接从缓存分配。

> Go 就采用了类似 TCmalloc 的设计来管理内存 http://legendtkl.com/2015/12/11/go-memory/
>
> tcmalloc: https://goog-perftools.sourceforge.net/doc/tcmalloc.html
>
> 图解：https://zhuanlan.zhihu.com/p/29216091
>
> 【性能】tcmalloc 使用和原理：https://www.cnblogs.com/bandaoyu/p/16752421.html
>
> 至于 jemalloc, ptmalloc, tcmalloc 的区别，可以看看 https://wenfh2020.com/2021/11/14/question-design-memory-pool/

tcmalloc 使用自旋锁分配大内存，但这也有问题，大内存较多的情况下？所以本文提出 TEMERAIRE 来优化。

## Abstract

**Memory allocation** represents significant compute cost at the **warehouse scale**

一种经典的优化方法是**提高分配器的效率**，以最小化分配器代码所消耗的周期。

然而，内存分配决策还通过数据布局影响整体应用程序性能，提供了通过使用更少的硬件资源完成更多应用程序工作来提高整个 fleetwide 生产力的机会。在此，我们聚焦于**大页覆盖率 hugepage coverage**。我们提出了 TEMERAIRE，这是 TCMALLOC 的一个**大页感知增强版本**，旨在减少应用程序代码中的 CPU 开销。我们讨论了 TEMERAIRE 的设计与实现，包括大页感知内存布局的策略，**以最大化大页覆盖率并最小化碎片 fragmentation 开销**。我们展示了针对 8 个应用程序的研究，将每秒请求数（RPS）提高了 7.7%，并将 RAM 使用量减少了 2.4%。

我们还展示了在机群 fleet 规模上进行的 1% 实验结果以及在谷歌仓库 warehouse 规模计算机中的长期部署情况。这带来了 6% 的 TLB 未命中停滞减少，以及 26% 的由于碎片导致的内存浪费减少。最后，我们讨论了改进分配器开发过程的额外技术以及未来内存分配器的潜在优化策略。

## Introduction

datacenter tax: within a warehouse-scale computer (WSC) is the cumulative time spent on common service overheads, such as serialization, RPC communication, compression, copying, and memory allocation

WSC 工作负载的多样性[23]意味着我们通常无法通过优化单个应用程序来显著提高整个系统的效率，因为成本分散在许多独立的工作负载上。

相比之下，专注于数据中心税的组成部分可以在**整体上实现显著的性能和效率提升**，因为这些优化可以应用于整个类别的应用程序。过去几年中，我们的团队专注于**minimizing the cost of memory allocation decisions**，取得了显著成效

通过大幅减少内存分配所花费的时间，实现了整个系统的收益。但我们不仅能够优化这些组成部分的成本。通过改变 allocator 来提高应用程序代码的效率，也能带来显著的收益。本文中，我们考虑如何通过提高内存分配器提供的 hugepage coverage 来优化应用程序性能。

Cache and Translation Lookaside Buffer (TLB) misses 是现代系统中主要的性能开销。在 WSC 中，内存墙[44]问题显著：一项分析[23]显示，50% cycles 因内存停滞。我们自己的工作负载分析发现，大约 20% 的周期因 TLB 未命中而停滞。

Hugepages are a processor feature, 可以显著减少 TLB 未命中的次数及其成本

大页的增加大小使得相同数量的 TLB 条目能够映射更大范围的内存。在所研究的系统中，大页还减少了未命中+填充的总停滞时间，因为它们的页表表示需要遍历的层级少了一层。

> 大页，如果 TLB 映射 2MB 的页面，1G 只需要 512 个页面，减少了映射数量

虽然分配器无法修改用户代码访问的内存量，甚至无法改变访问对象的模式，但它可以与操作系统合作并控制新分配的内存布局。**通过优化大页覆盖率，分配器可以减少 TLB 未命中**。在 C 和 C++等语言中，内存布局决策还必须处理其决策是最终的后果：对象一旦分配就无法移动[11]。分配布局决策只能在分配时进行优化。这种方法与我们在该领域的先前工作相反，因为我们可能会增加分配的 CPU 成本，增加数据中心税，但通过减少其他地方的处理器停滞来弥补。这提高了应用程序指标，如每秒请求数（RPS）。

- **TEMERAIRE 的设计**：我们设计了 **TEMERAIRE**，这是 **TCMALLOC** 的一个大页感知增强版本，旨在减少应用程序代码中的 CPU 开销。我们提出了大页感知内存布局的策略，以最大化大页覆盖率并最小化碎片开销。
- **在复杂真实世界应用和 WSC 规模中的评估**：我们在 WSC 中对 TEMERAIRE 进行了评估，测量了在我们的基础设施中运行的 8 个应用程序的样本，观察到每秒请求数（RPS）增加了 7.7%，RAM 使用量减少了 2.4%。将这些技术应用于谷歌 WSC 中的所有应用程序，带来了 6%的 TLB 未命中停滞减少，以及 26%的由于碎片导致的内存浪费减少。
- **优化内存分配器改进开发过程的策略**：我们使用跟踪、遥测和在仓库规模上的实验相结合的方法，提出了优化内存分配器改进开发过程的策略。

## The challenges of coordinating Hugepages

虚拟内存需要通过称为 Translation Looka- side Buffers (TLBs) 的缓存将用户空间地址转换为物理地址

**TLB 的条目数量有限**，对于许多应用程序，整个 TLB 仅覆盖使用默认页面大小的总内存占用的一小部分。现代处理器通过支持 TLB 中的大页来增加这种覆盖范围。一个对齐的大页（在 x86 上通常为 2MiB）仅占用一个 TLB 条目。大页通过增加 TLB 的有效容量并减少 TLB 未命中来减少停滞

传统的分配器 manage memory in page-sized chunks。

对于将内存**释放回操作系统**的分配器（在仓库规模中，我们拥有长时间运行的具有动态工作周期的负载，这是必要的）来说，面临更大的挑战。Transparent Huge Pages (THP) 为内核提供了机会，使其能够在页表中使用大页覆盖连续的页面。表面上看，内存分配器只需分配对齐且大小为大页的内存块即可利用这种支持。

非大页对齐的内存区域的返回需要内核使用较小的页面来表示剩余部分，从而削弱了内核提供大页的能力，并对剩余使用的页面施加了性能成本。

> 比如用户申请 3MB，其中大页可以给 2MB 剩下的非对齐，用小页

或者，分配器可以在整个大页变为空闲之前不将其返回给操作系统。这保留了大页覆盖，但相对于实际使用情况，可能会导致显著的放大效应，使内存闲置。DRAM 是部署 WSC 的显著成本[27]。**分配器在管理外部碎片（即块中太小而无法用于请求分配的未使用空间）方面在这个过程中非常重要**。例如，考虑图 1 中的分配。经过这一系列分配后，有 2 个单位的空闲空间。选择是使用小页，这会导致较低的碎片化但 TLB 条目使用效率较低，或者使用大页，这具有 TLB 效率但碎片化较高。

![](https://s2.loli.net/2024/10/29/YbB3ywOj7eKi19M.png)

> 图一，大页带来的碎片

通过密集地将**分配与大页边界对齐**，优先使用分配的大页，并（理想情况下）以相同的对齐方式返回未使用的内存，来与其结果合作。大页感知的分配器有助于在用户级别管理内存连续性。目标是最大限度地将分配打包到接近满的大页上，反之亦然，最小化在空（或较空）大页上使用的空间，以便它们可以作为完整的大页返回给操作系统。这有效地使用内存，并与内核的透明大页支持良好交互。此外，更一致地分配和释放大页形成了一个积极的反馈循环：减少内核级别的碎片化，并提高未来分配由大页支持的可能性。

## Overview of TCMALLOC

TCMALLOC 是一种在大规模应用程序中使用的内存分配器，常见于 WSC 环境中。它展示了强大的性能[21]。我们的设计直接建立在 TCMALLOC 的结构上。

![](https://s2.loli.net/2024/10/29/quvmYjVWnOKXrJs.png)

图 2 展示了 TCMALLOC 中内存的组织方式。对象按大小进行隔离。首先，TCMALLOC 将内存划分为**对齐 page size 的 spans**。

TCMALLOC 的结构由其对驱动任何内存分配器的两个相同问题的回答所定义。

1. 我们如何选择对象大小并组织元数据以最小化空间开销和碎片化？

2. 我们如何可扩展地支持并发分配？

足够大的分配由仅包含分配对象的 span 来满足。其他 span 包含多个相同大小（sizeclass）的较小对象。“小”对象大小的界限是 256 KiB。在这个“小”阈值内，分配请求被向上舍入到 100 个 sizeclass 之一。

> spans 大小怎么定义？小对象是什么
>
> 这里的 对象 object 指的是内存吗

TCMALLOC 将对象存储在一系列缓存中，如图 3 所示。span 从简单的 pageheap 中分配，pageheap 跟踪所有未使用的页面并进行最佳适应分配。

![](https://s2.loli.net/2024/10/29/GiSQ9O8Kbepn3Vs.png)

pageheap 还负责在可能的情况下将不再需要的**内存返回给操作系统**。

与其在 free()路径上执行此操作，不如定期调用一个专用的 release-memory 方法，旨在维持一个可配置的稳定释放速率（MB/s）。这是一种 heuristic 方法，TCMALLOC 希望在**稳态下尽可能少地使用内存**，避免使用先前提供的内存来省略昂贵的系统分配。

> 定期释放，GC？那需要 stop the world 吗

理想情况下，TCMALLOC 会返回用户代码将不再需要的所有内存。内存需求变化不可预测，这使得在同时保留内存以避免系统调用和页面错误的情况下，返回将未使用的内存变得具有挑战性。关于内存返回策略的更好决策具有很高的价值，并在第 7 节中进行了讨论。

TCMALLOC 首先尝试从“本地”缓存中提供分配，类似于大多数现代分配器[9,12,20,39]。最初这些是所谓的**每个线程缓存**（per-Thread Caches），为每个 sizeclass 存储一个空闲对象列表。为了减少孤立内存并提高高度线程化应用程序的重用率，TCMALLOC 现在使用每个超线程本地缓存。

当本地缓存没有适当 sizeclass 的对象来满足请求（或在尝试释放后有太多对象）时，请求会路由到该 sizeclass 的**单个中央缓存**。这有两个组成部分——一个小的快速、受**互斥锁保护的传输缓存 transfer
cache**（包含该 sizeclass 的扁平数组对象）和一个大的、受**互斥锁保护的中央空闲列表 central freelist**，包含分配给该 sizeclass 的所有 span ；可以从这些 span 中获取对象，也可以将对象返回给这些 span 。当一个 span 中的所有对象都返回到中央空闲列表中的 span 时，该 span 将返回到 pageheap。

> Sizeclass 是指内存分配器根据不同大小的内存块对内存进行分类的方法。在 TCMALLOC 中，sizeclass 用于将内存块划分为特定大小的类别，每个类别对应一定范围的内存块大小。
> 举个例子：
>
> Sizeclass 1: 管理 8 字节的内存块。
>
> Sizeclass 2: 管理 16 字节的内存块。
>
> Sizeclass 3: 管理 32 字节的内存块。
>
> Sizeclass 4: 管理 64 字节的内存块。

在我们的 WSC 中，大多数分配都很小（50%的分配空间是 ≤ 8192 字节的对象），如图 4 所示。这些对象随后被聚合到跨度中。pageheap 主要分配 1 页或 2 页的跨度，如图 5 所示。80%的跨度小于一个大页。

“stack” 缓存的设计使系统具有有用的模块化特性，TCMALLOC 的 pageheap 有一个简单的接口来管理内存，New(N)，Delete(S)，Release(N)

## TEMERAIRE’s approach

TEMERAIRE 是本文对 TCMALLOC 的贡献，它用一个旨在最大限度填充（和清空）大页的设计替换了 pageheap。

我们开发了启发式方法，将分配密集地打包到**高使用率的大页**上，并同时形成完全未使用的大页以返回给操作系统。

**Slack** 是指分配请求的大小与**下一个完整大页**之间的差距。从操作系统分配的虚拟地址空间在没有 reserving physical memory 时是 unbacked 的。在使用时，它由操作系统支持，通过物理内存映射。我们可以再次将内存释放给操作系统，使其再次 unbacked。我们主要在大页 boundaries 内打包，但使用大页 regions 来跨大页边界打包分配。

> slack 比如申请 5MB 得到 3 个 2MB 大页，gap 就是 1MB
>
> 申请完成后，物理内存还没有分配，处于 unbacked 状态，当用的时候才映射

1. Total memory demand varies unpredictably with time, but not every allocation is released.
2. Completely draining hugepages implies packing memory at hugepage granularity.
3. Draining hugepages gives us new release decision points
4. Mistakes are costly, but work is not.

我们的分配器通过委托给几个子组件来实现其接口，如图 6 所示。每个组件都牢记**上述原则**，并针对其处理的最佳分配类型进行专门化。根据原则#4，we emphasize smart placement over speed

虽然 TEMERAIRE 的具体实现与 TCMALLOC 内部结构相关，但大多数现代分配器共享类似的页面（或更高）粒度的大后备分配，如 TCMALLOC 的跨度：比较 jemalloc 的“extents”[20]，Hoard 的“superblocks”[9]，和 mimalloc 的“pages”[29]。Hoard 的 8KB superblocks 直接通过‘mmap‘分配，防止大页连续性。这些 superblocks 可以改为密集地打包到大页上。mimalloc 将其 64KiB+的“pages”放置在“segments”中，但这些是按线程维护的，这阻碍了进程段之间的密集打包。急切地将页面返回给操作系统可以最小化这里的 RAM 成本，但会分解大页。这些分配器也可以从类似 TEMERAIRE 的大页感知分配器中受益。

> jemalloc, hoard 都是很好的内存分配器
>
> jemalloc 也采用了 temeraire 的策略，采用了启发式的避免碎片，但不完全一样
>
> https://github.com/jemalloc/jemalloc/issues/2301

![](https://s2.loli.net/2024/10/29/mvWOpiuZhI46ePo.png)

### The overall algorithm

![](https://s2.loli.net/2024/10/29/J9mI4OQSArYhVo8.png)

> 图 7 分配算法，
>
> 如果大于 1G，调用 HugeCache，slack 不重要
>
> 如果小于 1MB 采用 HugeFiller，调用大页
>
> 如果 1M-2M 之间，尝试 HugeFiller 复用内存，处理小请求，打包到 hugepage
>
> 2M 之上的用 HugeRegion 来分配，如果不够就 HugeCache 分配大页

我们的目标是最大限度地减少生成的 slack，如果我们确实生成了 slack，则将其**重用于其他分配**（如同任何页面级别的碎片化一样）。

在所有组件背后是 `HugeAllocator`，它处理虚拟内存和操作系统。它为其他组件提供未支持的内存，这些组件可以支持并传递下去。我们还维护一个**backed** **完全空的大页缓存**，称为 `HugeCache`

> HugeCache 处于 backed，原因是什么？

我们保持一个**部分填充**的单个大页列表（HugeFiller），这些大页可以通过后续的**小分配密集填充**。在沿大页边界打包分配效率低下的地方，我们实现了一个**专门的分配器（HugeRegion）**。

1. 对于 exact multiple of hugepage size 或那些足够大以至于 slack 无关紧要的分配，我们直接转发到 HugeCache。

2. 中等大小的分配（介于 1MiB 和 1GiB 之间）通常也从 HugeCache 分配，最后一步是捐赠 slack。例如，从 HugeCache 分配的 4.5 MiB 会产生 1.5 MiB 的 slack，这是一个**不可接受的高开销比率**。TEMERAIRE 通过假装 the last hugepage of the request has a single “leading” allocation on it（图 8），将该 slack 捐赠给 HugeFiller。

![](https://s2.loli.net/2024/10/29/8XMyD7qirb2BsSc.png)

当这样的大跨度被释放时，分配器还将标记为虚构的前导分配为空闲。如果 slack 未被使用，它将与剩余部分一起返回到尾部大页。否则，尾部大页将留在 HugeFiller 中，只有前 N-1 个大页返回到 HugeCache。

3. 对于某些分配模式，中等大小的分配产生的 slack 比我们可以在严格的 2MiB 箱子中用较小分配填充的更多。例如，许多 1.1MiB 的分配每个大页将产生 0.9MiB 的 slack（见图 12）。当我们检测到这种模式时，HugeRegion 分配器跨大页边界放置分配，以最小化这种开销。

![](https://s2.loli.net/2024/10/29/hnM8t97AcQIOkib.png)

4. 小请求（<= 1MiB）总是从 HugeFiller 提供。对于介于 1MiB 和大页之间的分配，我们评估几个选项：
   1. HugeFiller：如果我们有可用空间，我们使用它并乐于填充一个大部分为空的大页。
   2. HugeFiller 无法满足这些请求，我们接下来考虑 HugeRegion；如果我们有可以满足请求的分配区域，我们就这样做。如果没有区域存在（或者它们都太满了），我们考虑分配一个，但只有在测量到高比例的 slack 与小分配时才这样做（如下所述）。
   3. 否则，我们从 HugeCache 分配一个完整的大页。这会产生 slack，但我们预计它将被未来的分配填充。

我们在 TEMERAIRE 中做出一个设计选择，即**关心外部碎片化到大页级别**，但基本上不关心超过这个级别（但见第 4.5 节中的例外）。例如，一个具有单个 1 GiB 空闲范围的系统和另一个具有 512 个不连续空闲大页的系统在 TEMERAIRE 中处理得同样好。在这两种情况下，分配器通常会将所有未使用的空间返回给操作系统；一个 1 GiB 的新分配将需要在任一情况下引入内存错误。在碎片化的情况下，我们将在新的虚拟内存上这样做。未被活动分配占用且不消耗物理内存的虚拟地址范围的浪费不是一个问题，因为对于 64 位地址空间，虚拟内存实际上是免费的。

### HugeAllocator

HugeAllocator 跟踪映射的虚拟内存。所有操作系统映射都在这里进行。它存储大页对齐的未支持范围（即没有关联物理内存的范围）。虚拟内存几乎是免费的，因此我们追求简单和合理的速度。我们的实现使用 treap[40]跟踪未使用的范围。我们通过其最大包含范围增强子树，这使我们能够快速选择一个近似最佳适应。

### HugeCache

HugeCache 跟踪以**完整大页粒度支持的内存范围**。HugeFiller **填充**和清空整个大页的一个结果是，我们需要决定何时将空大页返回给操作系统。我们会后悔返回我们将再次需要的内存，同样也会后悔不返回将在缓存中闲置的内存。急切地返回内存意味着我们进行系统调用以返回内存，并在重新使用时引入页面错误。仅在 TCMALLOC 的定期释放线程请求的速率下释放内存意味着内存被闲置。

![](https://s2.loli.net/2024/10/29/aVQevNFCcxG4Z5I.png)

考虑图 9 中的人工程序，没有额外的堆分配。在循环的每次迭代中，‘New‘需要一个新的大页并将其放置在 HugeFiller 中。‘Delete‘移除分配，现在大页完全空闲。急切地返回将要求每次迭代都进行系统调用，对于这个简单但病态的程序来说。

> 512K 直接走 Filler，

我们跟踪 2 秒滑动窗口内的需求周期性，并计算所见的最小值和最大值（demandmin，demandmax）。每当内存返回到 HugeCache 时，如果缓存将大于 demandmax − demandmin，我们将大页返回给操作系统。我们还尝试了其他算法，但这个算法简单且足以捕捉我们看到的经验动态。只要我们的窗口需求看到了新大小的需求，缓存就可以增长。在振荡使用中，这将（错误地）释放内存一次，然后（正确地）从那时起保持它。图 10 显示了 Tensorflow 工作负载的缓存大小，该工作负载通过大幅振荡使用；我们紧密跟踪实际需要的内存。

![](https://s2.loli.net/2024/10/29/MgsihxmyCW8916w.png)

> 没懂

### HugeFiller

HugeFiller 满足每个适合单个大页的小分配。这满足了大多数分配（78% of the pageheap is backed by the HugeFiller on average across the fleet），并且是我们系统中最重要的——也是最优化的——组件。在给定的大页内，我们使用一个**简单（且快速）**的 best-fit algorithm 来放置分配；挑战在于决定在哪个大页上放置分配。

这个组件解决了我们的 binpacking problem：我们的目标是**将大页分成一些保持最大填充**，而其他一些保持空或近空。最空的大页可以被回收（可能根据需要分解大页），同时最小化对大页覆盖的影响，因为密集填充的页面覆盖了大部分使用内存的大页。但清空大页具有挑战性，因为我们不能依赖任何特定的分配消失。

次要目标是最大限度地减少每个大页内的碎片化，以使新请求更有可能被满足。如果系统需要一个新的 K 页跨度，并且没有 ≥ K 页的空闲范围可用，我们需要从 HugeCache 获取一个大页。这会创建(2MiB − K ∗ pagesize)的 slack，浪费空间。

这给我们两个优先考虑的目标。因为我们希望最大化大页完全空闲的概率，近空的大页是宝贵的。因为我们需要最小化碎片化，具有长空闲范围的大页也是宝贵的。两个优先级都通过保留具有最长空闲范围的大页来满足，因为较长的空闲范围必须具有较少的在使用块。我们相应地将大页组织成排序列表，利用每个大页的统计数据。

在每个大页内，我们跟踪使用页的位图；为了从某个大页填充请求，我们从该位图中进行最佳适应搜索。

> 次要目标才是减少大页内部碎片？

- the longest free range (L), the number of contiguous pages **not already allocated**,
- the total number of allocations (A),
- the total number of used pages (U).

这三个统计数据决定了大页的优先顺序以放置分配。我们选择具有最低足够 L 和最高 A 的大页。对于 K 页的分配，我们首先只考虑最长空闲范围足够的大页（L ≥ K）。这决定了一个大页是否是可能的分配目标。

在 L ≥ K 的大页中，我们按 fullness 优先排序。大量的实验使我们选择了 A，而不是 U。

> 实验证明，优先分配数量？

This choice is motivated by a radioactive decay-type allocation model 其中每个分配，无论大小，都有可能变得空闲（具有某种概率 p）。在这个模型中，具有 5 个分配的大页有 `p^5 << p` 的概率变得空闲；因此，我们应该非常强烈地避免从分配非常少的大页中分配。特别是，这个模型预测 A 是一个比 U 更好的“空闲度”模型：大小为 M 的一个分配比大小为 1 的 M 个分配更有可能被释放。

> 看不懂，略过概率模型
>
> L ≥ K：首先选择具有足够空闲页的巨页。
>
> 按 A 排序：在符合条件的巨页中，优先选择已分配数较多的巨页（因为在放射性衰变模型下，这样的巨页更有可能完全变为空闲页）。

我们的策略与最佳适应不同。考虑一个大页 X 有一个 3 页的间隙和一个 10 页的间隙，另一个大页 Y 有一个 5 页的间隙。最佳适应会选择 X。**我们的策略选择 Y**。这个策略有效，因为我们正在寻找分配在**最碎片化的页面上**，因为碎片化的页面不太可能完全空闲。如果我们需要，比如 3 页，那么最多包含 3 个可用页的间隙的页面更有可能是碎片化的，因此是好的分配候选者。在放射性衰变模型下，靠近大间隙的分配与任何其他分配一样可能变得空闲，这可能导致这些间隙显著增长；然后它们可以用于大分配。我们将 10 页的间隙视为宝贵，并避免在其附近分配，除非没有其他选择，这允许它增长。

令人惊讶的是，这个简单的策略显著优于全局最佳适应算法——将请求放置在任何大页中最接近其大小的单个间隙中。最佳适应将非常昂贵——我们不能为每个请求搜索 10-100K 大页，但令人非常反直觉的是，它也会产生更高的碎片化。最佳适应对于一般碎片化问题远非最佳结果并不是新发现[36]，但看到它在这里有多糟糕是很有趣的。

> best fit 不一定是最优的，是因为局部不一定全局？反而会有更多的碎片

![](https://s2.loli.net/2024/10/29/z6hs2cXj7VPwoan.png)

> 图 11 证明了 LFR 随着需求内存降低也在降低

### HugeRegion

HugeCache（及其背后的 HugeAllocator）足以处理大分配，其中舍入到完整大页的成本很小。HugeFiller 对于可以打包到单个大页的小分配效果很好。HugeRegion 则介于两者之间。

考虑一个请求 1.1 MiB 内存的情况。我们从 HugeFiller 中提供它，留下 2 MiB 大页中未使用的 0.9 MiB 内存：slack 空间。HugeFiller 假设 slack 将被未来的小分配（<1MiB）填充，通常确实如此：我们观察到的机群范围内小分配与 slack 的字节比率为 15:1。在极限情况下，我们可以想象一个二进制文件只请求 1.1 MiB 跨度，如图 12 所示。

### Memory Release

## Evaluation of TEMERAIRE

Google’s WSC workloads.

因此，我们检查了 IPC 指标（作为用户吞吐量的代理），并在可能的情况下，我们获得了应用程序级别的性能指标，以评估工作负载在我们服务器上的生产力（例如，每核心每秒请求数）。

### Application Case Studies

- Tensorflow
- search1, search2, ads1, ads2...
- Spanner 分布式数据库
- loadbalancer
- redis

对于具有定期释放的 8 个应用程序，我们观察到平均 CPU 改进为 7.7%，平均 RAM 减少为 2.4%。其中两个工作负载没有看到内存减少。TEMERAIRE 的 **HugeCache 设计很好地处理了 Tensorflow 的分配模式**，但无法影响其突发需求。Spanner 将其缓存最大化到某个内存限制，因此减少 TCMALLOC 的开销意味着在相同的内存占用下可以缓存更多的应用程序数据。

> 这些应用看着挺有意思，不知道有没有开源
>
> tensorflow 改进是最大的
>
> 内存大部分都会减少
>
> 为什么 IPC 会增加？是每周期指令数吗，可以表示性能更好，缓存命中更高？
>
> page fault 上升了，几乎 2x 这不会有影响吗？

## Conclusion

In warehouse scale computers, TLB lookup penalties are one of the most significant compute costs facing large applications. TEMERAIRE optimizes the whole WSC by changing the memory allocator to make hugepage-conscious placement decisions while minimizing fragmentation. Application case studies of key workloads from Google’s WSCs show RPS/CPU increased by 7.7% and RAM usage decreased by 2.4%. Experiments at fleet scale and longitudinal data during the rollout at Google showed a 6% reduction in cycles spent in TLB misses, and 26% reduction in memory wasted due to fragmentation. Since the memory system is the biggest bottleneck in WSC applications, there are further opportunities to accelerate application performance by improving how the allocator organizes memory and interacts with the OS.

> 没看到 temeraire 是怎么处理高并发场景的，还是说沿用 tcmalloc 的设计？
