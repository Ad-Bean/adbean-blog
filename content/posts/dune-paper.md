+++
title = 'Paper Reading: Dune'
date = 2024-09-11T09:10:07-04:00
draft = false
tags = ['Paper Reading']
+++

## Dune: Safe User-level Access to Privileged CPU Features

是 MIT OS 课上的讲座，也是一篇很有意思的论文，很类似 exokernel，不过更像是内核上的功能，利用虚拟化使得进程可以访问一些内核态的功能，比如暴露 page table，使得 jvm 可以进行更高级的 GC。后续也有 IX，ZygOS 等相关论文。

## Abstract

Dune 是一个系统，它为应用程序提供了直接但安全的硬件特性访问，例如环保护、页表和标记 TLB，同时保留了现有操作系统对进程的接口。Dune 利用现代处理器中的虚拟化硬件为进程提供抽象，而不是机器抽象。它由一个小的内核模块组成，该模块初始化虚拟化硬件并与内核交互，以及一个用户级库，帮助应用程序管理特权硬件功能。我们介绍了 Dune 在 64 位 x86 Linux 上的实现。我们使用 Dune 实现了三个可以从访问特权硬件中受益的用户级应用程序：一个用于不受信任代码的沙箱、一个特权分离设施和一个垃圾收集器。使用 Dune 极大地简化了这些应用程序的实现，并提供了显著的**性能优势**。

> 和 exokernel 很类似

## Introduction

许多应用程序可以从访问“仅内核可用”的硬件特性中受益。例如，Azul Systems 通过使用分页硬件显著加速了**垃圾回收** [15, 36]。另一个例子是进程迁移，尽管可以在用户程序中实现，但如果能访问页错误 [40] 和系统调用 [32]，将会受益匪浅。在某些情况下，甚至可能需要完全替换内核以满足特定应用程序的需求。例如，IBOS 通过将浏览器抽象移到最低的操作系统层来提高浏览器安全性 [35]。

> 感觉最主要的好处就是帮助 GC 做得更好，能知道页错误出了什么问题

这类系统需要对内核进行修改，因为出于**安全和隔离**的原因，用户空间的硬件访问是受限的。不幸的是，在实践中修改内核并不理想，因为**内核的修改可能会相当侵入性**，如果操作不当，会影响整个系统的稳定性。此外，如果多个应用程序需要内核修改，无法保证这些修改能够兼容。另一种策略是将应用程序打包到带有专用内核的**虚拟机**镜像中 [4, 14]。许多现代 CPU 包含**虚拟化硬件**，使客户操作系统能够安全高效地访问内核硬件特性。此外，虚拟机提供了类似于进程的故障隔离——即，错误或恶意行为不应导致整个物理机器崩溃。

生产内核如 Linux 复杂且难以修改。然而，实现一个带有简单虚拟内存层的专用内核也同样具有挑战性。除了虚拟内存，还必须支持文件系统、网络栈、设备驱动程序和引导过程。

本文介绍了一种新的应用程序使用内核硬件特性的方法：使用**using virtualization hardware to provide a process**，而不是**machine abstraction**。我们在 64 位 Intel CPU 上的 Linux 系统中实现了这种方法，称为 Dune。Dune 提供了一个可加载的内核模块，该模块与未修改的 Linux 内核配合工作。该模块允许进程进入“Dune 模式”，这是一个不可逆的转换，通过虚拟化硬件，安全快速地访问特权硬件特性，包括特权模式、虚拟内存寄存器、页表和中断、异常及系统调用向量。我们提供了一个用户级库 libDune，以促进这些特性的使用。

> 基于 Linux 实现的

对于符合其范式的应用程序，Dune 比虚拟机提供了几个优势。首先，Dune 进程是一个正常的 Linux 进程，唯一的区别是它使用 **VMCALL** 指令来调用系统调用。这意味着 Dune 进程可以完全访问系统的其余部分，并且是系统的一部分，Dune 应用程序易于开发（像应用程序编程，而不是内核编程）。其次，由于 Dune 内核模块不试图提供机器抽象，模块可以更简单且更快。特别是，虚拟化硬件可以配置为避免保存和恢复虚拟机所需的几种硬件状态。

> VMCALL 是什么，类似 hyper-v 之类的吗？看上去只有 x86 支持的比较好？是不是一个问题呢。VMCALL 适用于所有支持 Intel VT-x 技术的虚拟化环境，包括 KVM、Xen 等。那么 Dune 和 KVM 的区别又是什么？后者全虚拟化，提供抽象硬件，虚拟机能运行在物理硬件上，是一个内核？结合了 QEMU，类似虚拟机而不是特权访问。

通过 Dune，我们做出了以下贡献：

- 我们提出了一种设计，利用**硬件辅助虚拟化安全高效地向用户程序暴露特权硬件特性**，同时保留标准操作系统抽象。
- 我们详细评估了三种硬件特性，并展示了它们如何使受益于用户程序：**异常、分页和特权模式**。
- 我们通过实现和评估三个用例（沙箱、特权分离和垃圾回收）展示了 Dune 的端到端**实用性**。

> 虚拟化技术感觉没什么缺点，又安全，性能也不差，可能也就开销比较大吧，再就是把负担给到开发者，需要熟悉虚拟化技术、特权访问等等，

## Virtualization and Hardware

Dune 能够暴露哪些特权硬件特性

我们以 x86 CPU 和 Intel VT-x 为例描述 Dune。然而，这并不是我们设计的基础，在第 7 节中，我们将扩展讨论，包括未来可能支持的其他架构。

> 问题之一把，不是很好跨平台

### The Intel VT-x Extension

Intel x86 (ISA) 的虚拟化，但是 AMD 的不同

虚拟内存可能是 VMM 最难安全暴露的硬件特性。

> 可能暴露是比较困难的，难度比较大，要保证安全可用

### Supported Hardware Features

Dune 使用 VT-x 为 x86 保护硬件提供用户程序的完全访问权限。这包括三个特权硬件特性：

异常、虚拟内存和特权模式。

表 1 显示了为每个特性提供的相应特权指令。Dune 还暴露了**分段**，但我们不再进一步讨论，因为它是现代 x86 CPU 上的主要遗留机制。

用户程序还可以从快速灵活的虚拟内存访问中受益 User programs can also benefit from fast and flexible
access to virtual memory

Dune 还赋予用户程序手动控制 TLB 失效的能力 Dune also gives user programs the ability to manually control TLB invalidations， 这允许单个用户程序高效地在多个页表之间切换。总的来说，我们展示了使用 Dune 在 Appel 和 Li 的用户级虚拟内存基准测试 [5] 中比 Linux 快 7 倍

最后，Dune 暴露了对特权模式的访问 Dune exposes access to privilege modes

尽管 Dune 暴露的硬件特性足以支持我们的动机用例，但其他几个硬件特性，如缓存控制、调试寄存器和访问 DMA 设备的能力，也可以通过虚拟化硬件安全地暴露。我们将这些留给未来的工作，并在第 7 节中讨论它们的潜力。

> 想知道这个缓存控制、访问 DMA 是怎么做的，如果可以虚拟化 DMA 会怎么样
>
> TLB 硬件存储最近的虚拟地址和物理地址映射，一般用内核管理 TLB 失效，如果让程序来管理，手动失效可以更快更新？比如进程切换？但是会不会比较危险呢，把别人的 TLB 弄丢了，一致性也可能会存在问题，导致内存访问错误。而 TLB 失效也是能够优化 GC 的一点吧。
>
> 特权模式比如 syscall，创建内存分配，DMA 等等，还有就是页中断、异常处理、系统调用等等

## Kernel Support for Dune

### System Overview

![](https://s2.loli.net/2024/09/11/9Iax7J1QEqXBHFT.png)

图 1 展示了 Dune 架构的高层次视图。Dune 通过一个模块扩展内核，该模块启用 VT-x，将内核置于 VMX root 模式。使用 Dune 的进程通过在 VMX non-root 模式下运行，获得对特权硬件的直接但安全的访问权限。Dune 模块拦截 VM 退出，这是 Dune 进程访问内核的唯一方式，并执行任何必要的操作，如服务页面故障、调用系统调用或在 HLT 指令后让出 CPU。Dune 还包括一个库，称为 libDune，用于协助在用户空间中管理特权硬件特性，这在第 4 节中进一步讨论。

### Threat Model

我们假设 CPU 没有缺陷，尽管我们承认在极少数情况下已经发现了可利用的硬件缺陷

> 不太懂这个假设

### Comparing to a VMM

具体来说，Dune 暴露了一个**进程环境**，而不是机器环境。因此，Dune 无法支持正常的客户操作系统，但这使得 **Dune 更轻量级和更灵活**。一些最显著的区别如下

> 所以和 KVM 等的最大区别就是 dune 是个进程，更加轻量级

### Memory Management

> 每次内存管理的实现细节就一脸懵，dune 自己做了个页表转换？
>
> EPT（Extended Page Table，扩展页表）是 Intel VT-x 虚拟化技术中的一种硬件机制，用于在虚拟机（Guest OS）和虚拟机监控程序（Hypervisor）之间提供安全的内存管理。EPT 的主要作用是增强虚拟机的内存隔离和安全性，同时提高内存访问的效率。

内存管理是 Dune 模块的最大责任之一。挑战在于**向用户程序暴露直接的页表访问**，同时**防止对物理内存的任意访问**。此外，我们的目标是默认提供一个正常的进程内存地址空间，允许用户程序仅添加他们需要的功能，而不是完全替换内核级的内存管理。

EPT format incompatibility

> 好处，直接页表访问 + 内存隔离 + 灵活+ 性能
>
> 但问题是 intel vt-x EPT 需要和 x86 页表做兼容，可能会出现一些程序不兼容？内存管理也比较复杂吧，开发人员也需要知道 ept 是什么意思

### Exposing Access to Hardware

x86 包括各种控制寄存器，

Dune 出于性能原因限制了对硬件寄存器的访问。例如，Dune 不允许修改 MSR，

> VT-x（Virtualization Technology for x86） 是 Intel 开发的一种硬件虚拟化技术，用于在 x86 架构上实现虚拟化。
>
> VMX root 模式： 用于运行虚拟机监控程序（VMM），不改变 CPU 行为，除了启用对 VT-x 管理的新指令的访问。
>
> VMX non-root 模式： 用于运行虚拟化的客户操作系统（Guest OS），限制了 CPU 行为，旨在运行虚拟化的客户操作系统。

### Preserving OS Interfaces

Dune 还保留了对操作系统系统调用的访问

### Implementation

Dune 目前支持在 Intel x86 处理器上以 64 位长模式运行的 Linux。对 AMD CPU 和 32 位模式的支持是未来的可能扩展

> 兼容性很大问题，因为是基于 VT-x 来做的，然而，高级代码不与 KVM 共享，因为 Dune 的操作方式与 VMM 不同。此外，我们的 Dune 模块比 KVM 更简单，仅由 2,509 行代码组成

## User-mode Environment

使用 Dune 的进程的执行环境与正常进程有一些差异。由于特权环是暴露的硬件特性，一个差异是用户代码在环 0 中运行。尽管改变了某些指令的行为，但这通常不会导致现有代码的不兼容性。环 3 也可用，并且可以选择用于限制不受信任的代码。另一个差异是系统调用必须作为超调用执行。为了简化支持这种更改，我们提供了一种机制，可以检测何时从环 0 执行系统调用，并自动将其重定向到内核作为超调用。这是 libDune 中包含的许多功能之一。

> 还是兼容问题吧，ring0 是最高特权级别，几乎就是内核和设备操作，ring3 是用户态了
>
> 而用户代码在环 0 中运行： 在 Dune 中，用户代码在环 0（Ring 0）中运行，而不是在传统的环 3（Ring 3）中运行。这意味着用户代码具有更高的特权级别，可以直接访问特权硬件特性
>
> 那需不需要切换到 ring3 呢？而且 syscall 变成了 hypercall

### Bootstrapping

在许多方面，将进程转换到 Dune 模式类似于启动操作系统。第一个问题是，在启用 Dune 之前，必须**提供有效的页表**。因为尽管目标是使进程地址在转换前后保持一致，但必须考虑 EPT 的压缩布局。

### Limitations

但 libDune 仍然缺少一些功能

但我们尚未完全集成对信号的支持，其次，尽管我们支持 pthreads，但 libDune 中的一些实用程序，如页表管理，尚未线程安全。这两个问题都可以通过进一步的实现来解决。

> 最大的问题还是线程不安全，可以执行较低，不支持非 x86 吧，而且要先提供页表，保证地址有效

## Applications

### Sandboxing

沙盒

我们尚未支持所有系统调用，但我们支持足够多的系统调用以运行大多数单线程 Linux 应用程序。然而，未来支持多线程程序没有任何障碍。

> 不支持多线程

### Wedge

不懂

### Garbage Collection

GC 应该是最有用的

- Fast faults
- Dirty bits
- Page table
- TLB control

我们修改了 Boehm GC [12] 以使用 Dune 来提高性能。Boehm GC 是一个健壮的标记-清除收集器，支持并行和增量收集。它被设计为要么与 C/C++ 程序一起使用作为保守收集器，要么由编译器和运行时后端使用，其中保守性可以控制。它被广泛使用，包括 Mono 项目和 GNU Objective C。

https://zhuanlan.zhihu.com/p/365571886

## Evaluation

### Overhead from Running in Dune

Dune 的性能受到两个主要开销来源的影响。首先，VT-x 增加了进入和退出内核的成本——VM 入口和 VM 退出比快速系统调用指令或异常更昂贵。因此，系统调用和其他类型的故障（例如，页面故障）在 Dune 中必须支付固定成本。其次，使用 EPT 使 TLB 未命中的成本更高，因为在某些情况下，硬件页面遍历器必须遍历两个页表而不是一个。

> getpid, page fault, page walk 开销也挺大的，看上去不仅是两三倍了
>
> 其他比如 trap, ptrace 就快得多了

### Optimizations Made Possible by Dune

ptrace 是系统调用拦截性能的度量。这是 Linux 进程使用 ptrace 拦截系统调用（getpid）、将系统调用转发到内核并返回结果的成本。在 Dune 中，这是在 VMX non-root 模式下使用环保护直接拦截系统调用、通过 VMCALL 转发系统调用并返回结果的成本。

PTRACE SYSEMU 是应用程序希望拦截系统调用但不将其转发到内核而是仅在内部实现它们时的最有效机制。由于 ptrace 需要将调用转发到内核，因此 PTRACE SYSEMU 是最有效的机制。使用 PTRACE SYSEMU 拦截系统调用的延迟为 13,592 个周期。在 Dune 中，这可以通过直接处理硬件系统调用陷阱来实现，延迟仅为 180 个周期。这表明 Dune ptrace 基准测试的大部分开销实际上是通过 VMCALL 转发 getpid 系统调用，而不是拦截系统调用。

trap 表示进程获取页面故障异常所需的时间。我们将 Linux 中 SIGSEGV 信号的延迟与 Dune 中硬件生成的页面故障进行比较。

appel1 是用户级虚拟内存管理性能的度量。它对应于 [5] 中描述的 TRAP、PROT1 和 UNPROT 测试，其中访问了 100 个受保护的页面，导致故障。然后，在故障处理程序中，故障页面被解除保护，并保护了一个新页面。

appel2 是用户级虚拟内存管理的另一个度量。它对应于 [5] 中描述的 PROTN、TRAP 和 UNPROT 测试，其中保护了 100 个页面。然后访问每个页面，故障处理程序解除故障页面的保护。

### Application Performance

... 可以看到 EPT 带来的问题还是很大的

忽略，直接看 GC

#### Garbage Collector

这些基准测试的结果如表 6 所示。直接移植显示了混合结果，由于内存保护和故障处理程序的改进，但也受到 EPT 开销的减速。一旦我们开始使用更多硬件特性，我们就会看到明显的性能改进。除了 XML Parser 之外，TLB 版本在 10.9% 到 23.8% 之间提高了性能，脏位版本在 26.4% 到 40.7% 之间提高了性能。

XML 基准测试很有趣，因为它显示了所有三个版本在 Dune 下的减速：直接版本慢 19.0%，TLB 版本慢 12.2%，脏位版本慢 0.2%。这似乎是由 EPT 开销引起的，因为基准测试没有产生足够的垃圾来受益于我们对 Boehm GC 所做的修改。这在表 6 中有所体现；总分配量几乎等于最大堆大小。我们通过修改基准测试来验证这一点，改为处理一系列 XML 文件，依次处理每个文件，以便回收内存。然后，我们看到随着文件数量的增加，Dune 版本相对于基线的线性改进。使用十个 150MB XML 文件作为输入，Boehm GC 的脏位版本显示执行时间比基线提高了 12.8%。

## Reflections on Hardware

## Related Work

## Conclusion

<!--

1. the problem the paper addresses
2. the paper’s solution
3. the evidence the paper provides that the solution was correct.
1. whether the problem the paper solved is an important one; are there any holes in the motivation?
2. particular strengths of the solution
3. particular weaknesses of the solution
4. whether the paper was fun or enjoyable to read

Compare and constrast how Dune and Exokernel approaches providing faster access to memory and memory-related optimizations, in the context of how Dune wants to provide applications with safe access to privileged hardware "in concert with the OS", while Exokernel explicitly wants to provide extensibility (aim to write at least 300-400 words, around a page).

The paper tackles the issue of how applications can safely and efficiently access privileged CPU features. Normally, these features are restricted to the operating system (OS) kernel for security and stability reasons. However, many applications could benefit from direct access to these features to improve performance and functionality.


The paper tackles the issue of how applications can safely and efficiently access privileged CPU features using Dune based on intel VT-x as a system process. Normally, these features are restricted to the operating system kernel for security and stability reasons. However, many applications such as garbage collection and sandbox could benefit from direct access to these features to improve performance and functionality.
The paper proposes a system called Dune, which leverages virtualization technique (Intel VT-x) to allow applications to access privileged CPU directly and safely. When an application enters Dune mode, it uses VT-x to run in a virtualized environment, allowing the application to execute privileged instructions directly, without compromising system security. And Dune is also using EPT to help manage memory more efficiently in virtualized environments.
The paper provides several evidence to support the effectiveness of Dune: the authors implement three applications using Dune: a sandbox for running untrusted code, a privilege separation facility called wedge, and a garbage collector, which is more efficient since it can handle page fault directly from user space.
The problem addressed by the paper is important since accessing privileged CPU from Dune can enhance performance and enable new functionalities for applications especially for the GC. I'm particularly interested in how it can improve GC and reduce the Stop The World.
By allowing applications to access privileged CPU features directly, Dune can boost performance, as demonstrated with the garbage collector, for example the dirty bit version of the Boehm GC showed a 12.8% improvement in execution time. Dune also leverages existing virtualization hardware (Intel VT-x and EPT), making it practical and feasible to implement on modern systems.
However, Dune relies on specific hardware features (Intel VT-x and EPT), which means it won’t work on all systems, particularly older or not x86 architectures. And handling signals in a Dune environment can be problematic, some utilities are not thread-safe in the libDune library.
The paper is quite engaging, it's innovative and pushes the exokernel even further, and inspires some future works like IX, ZygOS.
 -->
