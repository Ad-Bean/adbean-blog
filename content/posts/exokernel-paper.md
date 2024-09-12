+++
title = 'Paper Reading: Exokernel'
date = 2024-09-11T01:28:36-04:00
draft = false
tags = ['Paper Reading']
+++

## Exokernel: An Operating System Architecture for Application-Level Resource Management

微内核？搜了一下发现上交的操作系统课甚至会讲这篇论文，但这篇论文也太早了 1995 年的，MIT 6828 (现 6.1810) 应该也会讲

## Abstract

operating system abstractions such as interprocess com- munication and virtual memory

传统 OS 会通过固定这些接口来限制程序的性能、灵活性、功能。Exokernel 操作系统架构通过提供应用程序级别的物理资源管理来解决这个问题。在 Exokernel 架构中，一个小的内核通过低级接口将所有硬件资源**安全地**导出给不受信任的库操作系统。库操作系统使用这个接口来实现系统对象和策略。这种资源保护与管理的分离允许对传统操作系统抽象进行应用程序特定的定制，通过扩展、专门化甚至替换库来实现。

The exokernel operating system architecture addresses this problem by **providing application-level management of physical resources**.

例如，虚拟内存和进程间通信抽象完全在应用程序级别的库中实现

## Introduction

Operating systems define the interface between applications and physical resources

> 比如内存管理，文件系统？进程/线程管理？

Traditionally, operating systems hide information about machine resources behind high-level abstractions such as processes, files, address spaces and interprocess communication

We believe these problems can be solved through **application- level** (i.e., untrusted) resource management. To this end, we have designed a new operating system architecture, **exokernel**, in which traditional operating system abstractions, such as virtual memory (VM) and interprocess communication (IPC), are implemented en- tirely at application level by untrusted software

> 资源管理交给程序来解决？内核，exokernel 的 vm 和 ipc 也由 app 实现？

Application writers select libraries or implement their own. New implementations of library operating systems are incorporated by simply relinking application executables

应用程序级别的文件缓存控制可以将应用程序运行时间减少 45%，应用程序特定的虚拟内存策略提高应用程序性能，不适当的文件系统实现决策会对数据库的性能产生巨大影响，通过将信号处理推迟到应用程序，异常处理可以快一个数量级。

> 这里的论文也都很老，文件系统决策会影响数据库性能是什么意思呢，缓存替换策略？难道这些不是数据库来实现吗，数据库可以有自己的缓存挂哪里，还是需要定制文件系统呢，比如直接 IO，InnoDB 也有自己的缓存管理系统，文件系统的 IO 接口应该也是优化过的

The exokernel architecture is founded on and motivated by a single, simple, and old observation: the **lower the level** of a primitive, the **more efficiently** it can be implemented, and the more latitude it grants to implementors of higher-level abstractions.

> 还是很有道理的，应用程序自己做优化就好了

Therefore, an exokernel uses a different approach: it **exports hardware resources** rather than emulating them, which allows an efficient and simple implementation.

secure bindings, visible re-source revocation, abort protocol (exokernel can break
secure bindings )

(1) exokernels can be made efficient due to the **limited number of simple primitives** they must provide;

(2) low-level secure multiplexing of hardware resources can be provided with **low overhead**;

(3) **traditional abstractions**, such as VM and IPC, can be **implemented efficiently at application level**, where they can be easily extended, specialized, or replaced;

(4) applications can create **special-purpose** implementations of abstractions, tailored to their functionality and performance needs.

Aegis also gives ExOS (and other application-level software) flexibility that is not available in **microkernel-based** systems 例如，**虚拟内存**在应用程序级别实现，可以与**分布式共享内存系统**和**垃圾收集器**紧密集成。Aegis 的高效保护控制转移允许应用程序通过牺牲性能来换取额外功能，构建一系列高效的 IPC 原语

> 微内核又是什么，分布式共享内存又是什么

## Motivation for Exokernels

Typically, the abstractions include processes, files, address spaces, and interprocess communication.

### The cost of Fixed High-Level Abstractions

**abstractions** in traditional operating systems -> **general**

固定的高级抽象会损害应用程序的性能，**因为没有一种单一的方式来抽象物理资源或实现对所有应用程序都最佳的抽象**。在实现抽象时，操作系统被迫在稀疏或密集地址空间、读密集或写密集工作负载等之间进行权衡。任何这样的权衡都会惩罚某些类别的应用程序。例如，关系数据库和垃圾收集器有时具有非常可预测的数据访问模式，当操作系统强加如 LRU 这样的通用页面替换策略时，它们的性能会受到影响。这种应用程序特定策略的性能改进可能是显著的；Cao 等人 [10] 测量到，应用程序控制的文件缓存可以将应用程序运行时间减少多达 45%。

固定的高级抽象会隐藏应用程序的信息。例如，大多数当前系统不会将低级异常、定时器中断或原始设备 I/O 直接提供给应用程序级软件。不幸的是，隐藏这些信息使得应用程序难以或不可能实现自己的资源管理抽象。例如，数据库实现者必须在文件系统之上努力模拟随机访问记录存储，因为操作系统隐藏了页面错误和定时器中断

> 还是 trade off 吧，认为抽象不利于性能，隐藏了信息，限制了应用程序的功能。数据库需要随机访问存储，是怎么模拟的？文件系统一开始不是为了 数据库设计的，应该是这意思吧，比如文件系统是文件为单位的，也有通用的缓存策略，所以数据库系统都基于文件系统实现了自己的 B+Tree 结构等索引结构？
>
> 操作系统 OS 应该尽量暴露 low level 的原语

### Exokernels: An End-to-End Argument

为了提供最大的应用程序级别资源管理机会，Exokernel 架构由一个薄的 Exokernel 外壳组成，通过一组低级原语安全地复用和导出物理资源。库操作系统使用低级的 Exokernel 接口实现更高层次的抽象，并且可以定义专门用途的实现，以最好地满足应用程序的性能和功能目标.应用程序可以选择一个具有特定页表实现的库，该实现最适合其需求。据我们所知，没有其他安全的操作系统架构允许应用程序如此多的有用自由。

此外，安全复用不需要复杂的算法；它主要需要表来跟踪所有权。因此，Exokernel 的实现可以很简单。简单的内核提高了可靠性和维护的便利性，消耗较少的资源，并能够快速适应新的需求（例如，千兆位网络）。此外，正如 RISC 指令的情况一样，Exokernel 操作的简单性使它们能够高效地实现。

> 这样做的问题在哪？每个应用程序都得实现自己的页表？不安全？上下文切换更复杂 ？缓存一致性？

![](https://s2.loli.net/2024/09/11/2ESreOG9YRfbQA3.png)

### Library Operating Systems

library operating systems, 不受 Exokernel 的信任, 可以自由地信任应用程序 例如，如果应用程序向库传递错误的参数，只会影响该应用程序。最后，Exokernel 系统中的**内核交叉次数可以更少**，因为大部分操作系统运行在应用程序的地址空间中。

库操作系统可以提供所需的尽可能多的可移植性和兼容性。直接使用 Exokernel 接口的应用程序将不具备**可移植性**，因为接口将包含硬件特定的信息。使用实现标准接口（例如 POSIX）的库操作系统的应用程序将在任何提供相同接口的系统上具有可移植性。在 Exokernel 上运行的应用程序可以自由地替换这些库操作系统，而无需任何特殊权限，这简化了新标准和功能的添加和开发。

与微内核系统一样，Exokernel 可以通过三种方式提供向后兼容性：一是操作系统和其程序的二进制仿真；二是在 Exokernel 之上实现其硬件抽象层；三是在 Exokernel 之上重新实现操作系统的抽象。

> 可移植性也是一个重要的因素吧，还是得使用标准库。
>
> 微内核又是什么，微内核是将服务转移到进程上的一种内核模式。宏内核是一种传统的内核结构，它将进程管理，内存管理等各项服务功能都放到内核中去，通常用在通用式的内核上，如 unix，linux 等。微内核的代表：Minix，在 Minix 中，操作系统的内核，内存管理，系统管理都有自己的进程表，每个部分的表包含了自己需要的域。表象是精确对应的，为了保持**同步**，在进程创建或结束时，这三个部分都要更新各自的表。
>
> 微内核怎么做进程管理，信息传递，开发难度是不是很高，更安全？崩溃不会影响别的？IPC 怎么做，性能是不是更差？

## ExoKernel Design

> 引用 mit os 的课，但是很多嵌入式系统，例如 Minix，Cell，这些都是微内核设计。这两种设计都很流行，如果你从头开始写一个操作系统，你可能会从一个微内核设计开始。但是一旦你有了类似于 Linux 这样的宏内核设计，将它重写到一个微内核设计将会是巨大的工作。并且这样重构的动机也不足，因为人们总是想把时间花在实现新功能上，而不是重构他们的内核。
>
> 设计主要关注点是减少内核中的代码，它被称为 Micro Kernel Design（微内核）。在这种模式下，希望在 kernel mode 中运行尽可能少的代码。所以这种设计下还是有内核，但是内核只有非常少的几个模块，例如，内核通常会有一些 IPC 的实现或者是 Message passing；非常少的虚拟内存的支持，可能只支持了 page table；以及分时复用 CPU 的一些支持。
>
> 会很依赖 IPC？现在，对于任何文件系统的交互，都需要分别完成 2 次用户空间<->内核空间的跳转。与宏内核对比，在宏内核中如果一个应用程序需要与文件系统交互，只需要完成 1 次用户空间<->内核空间的跳转，所以微内核的的跳转是宏内核的两倍。通常微内核的挑战在于性能更差，这里有两个方面需要考虑：
>
> 在 user/kernel mode 反复跳转带来的性能损耗。
>
> 在一个类似宏内核的紧耦合系统，各个组成部分，例如文件系统和虚拟内存系统，可以很容易的共享 page cache。而在微内核中，每个部分之间都很好的隔离开了，这种共享更难实现。进而导致更难在微内核中得到更高的性能。

在将保护与管理分离的过程中，Exokernel 执行三项重要任务：

1. tracking ownership of resources
2. ensuring protection by guarding all resource usage or binding points, and
3. revoking access to resources

Exokernel 采用了三种技术。首先，使用安全绑定，库操作系统可以安全地绑定到机器资源。其次，可见撤销允许库操作系统参与资源撤销协议。第三，**中止协议**由 Exokernel 使用，通过强制手段打破不合作库操作系统的安全绑定。

### Design Principles

An exokernel specifies the details of the interface that library operating systems use to claim, release, and use machine resources.

- 安全地暴露硬件。导出的资源是底层硬件提供的资源：物理内存、CPU、磁盘内存、转换后备缓冲区（TLB）和地址上下文标识符。这一原则的动机是我们认为分布式、应用程序特定的资源管理是构建高效灵活系统的最佳方式。后续原则处理实现这一目标的细节。
- 暴露分配。
- 暴露名称。physical names
- 暴露撤销，利用可见的资源撤销协议，

> 尽可能地减少原语，暴露低级原语，同时保证安全和灵活，

exokernel hands over **resource policy** decisions to library operating systems

> 资源管理也交给库系统？让 app 决定怎么使用资源，

### Secure Bindings

Exokernel 的主要任务之一是**安全地复用资源**，为相互不信任的应用程序提供保护。为了实现保护，Exokernel 必须保护每个资源。为了高效地完成这项任务，Exokernel 允许库操作系统使用安全绑定绑定到资源

> 没太理解什么是保护应用程序

使用三种基本技术来实现安全绑定：硬件机制、软件缓存和下载应用程序代码。

安全绑定可以在 Exokernel 中缓存。例如，Exokernel 可以使用大型软件 TLB [7, 28] 缓存不适合硬件 TLB 的地址转换。软件 TLB 可以被视为经常使用的安全绑定的缓存。

安全绑定可以通过将代码下载到内核中来实现。每次资源访问或事件时都会调用此代码以确定所有权和内核应执行的操作。将代码下载到内核中允许在发生内核事件时立即执行应用程序控制线程。下载代码的优点是可以避免潜在的昂贵交叉，并且此代码可以在不要求调度应用程序本身的情况下运行。类型安全语言 [9, 42]、解释和沙箱 [52] 可以用于安全地执行不受信任的应用程序代码 [21]。

Multiplexing Physical Memory

原型 Exokernel 中实现了物理内存的安全绑定，
当库操作系统分配一个物理内存页面时，Exokernel 通过记录所有者和库操作系统指定的读写能力来为该页面创建一个安全绑定。页面的所有者有权更改与其关联的能力并将其释放。特权机器操作（如 TLB 加载和 DMA）必须由 Exokernel 保护。根据 Exokernel 暴露内核簿记结构的原则，页表应在应用程序级别可见（只读）。

Multiplexing Network

网络解复用支持可以通过软件或硬件提供。硬件机制的一个例子是使用 ATM 信元中的虚拟电路将流安全地绑定到应用程序 [19]。软件支持可以通过包过滤器 [37] 提供消息解复用。包过滤器可以被视为安全绑定的实现，其中应用程序代码——过滤器——被下载到内核中。协议知识仅限于应用程序，而确定数据包所有权的保护检查以内核理解的语言表达。通过仔细的语言设计（以限制运行时）和运行时检查（以防止野指针和危险操作）确保故障隔离。

### Downloading Code

除了实现安全绑定外，下载代码还可以用于提高性能。将代码下载到内核有两个主要的性能优势。第一个是显而易见的：消除**内核交叉**。

> 太多概念不懂了 ，周末有空看看 mit os

### Visible Resource Revocation

Exokernel 对大多数资源使用可见撤销。即使在时间片结束时，处理器也会被显式撤销；

> 撤销是什么，上下文切换？ 在可见撤销中，操作系统或内核在回收资源之前会通知应用程序或库操作系统。应用程序有机会在资源被回收之前执行一些清理或保存操作。撤销（Revocation）是指操作系统或内核回收已经分配给应用程序或进程的资源的过程。这些资源可能包括物理内存、处理器时间片、文件描述符、网络连接等。撤销的目的是确保资源能够被重新分配给其他应用程序或进程，从而提高系统的整体资源利用率。

### Revocation and Physical Naming

我们将撤销过程视为 Exokernel 和库操作系统之间的对话。库操作系统应组织资源列表，以便可以快速释放资源。例如，库操作系统可以拥有一个简单的物理页面向量：当内核指示某些页面应被释放时，库操作系统选择其页面之一，将其**写入磁盘**，并释放它。

> 写入磁盘？

### Abort Protocol

看不懂了

后面是 Aegis 的实现，和一些与 Ultrix 的性能比较

## 后记

不得不感叹自己读论文效率低下，浅显的看得慢，深入的看不懂，读起来这补补那补补，很容易就看不完

<!--
1. the problem the paper addresses (no more than 2-3 sentences)
2. the paper’s solution (no more than 2-3 sentences)
3. the evidence the paper provides that the solution was correct. (no more than 2-3)


Second, the response should answer the following, for each of the two papers:
1. whether the problem the paper solved is an important one; are there any holes in the motivation?
2. particular strengths of the solution
3. particular weaknesses of the solution
4. whether the paper was fun or enjoyable to read
Finally, for this class's papers, please reflect on the following questions in your response:

 1. Compare and constrast how Dune and Exokernel approaches providing faster access to memory and memory-related optimizations, in the context of how Dune wants to provide applications with safe access to privileged hardware "in concert with the OS", while Exokernel explicitly wants to provide extensibility (aim to write at least 300-400 words, around a page).



The paper Exokernel addresses the limitations of traditional operating systems, which restrict the performance, flexibility, and functionality of applications by enforcing fixed interfaces and implementations for operating system abstractions like interprocess communication and virtual memory. These fixed high level abstractions hide information about machine resources, limiting the ability of applications to optimize their performance and resource usage.
The paper proposes Exokernel architecture and the implementation, which allow application to manage physical resources. Exokernel is like micro-kernel, it securely exports all hardware resources through a low-level interface to untrusted library operating systems. The library operating systems are untrusted software libraries that implement traditional operating system abstractions such as virtual memory, file systems, and networking.
The paper implements Aegis, which is an prototype exokernel along with the library operating system ExOS. In the section 5 and 6,the paper presents the implementations's performance and flexibility measurements, showing that primitive kernel operations are 10 to 100 times faster than those in traditional monolithic UNIX operating system.

The paper solves an important issue, there're many micro-kernel or exokernel-like operating systems nowadays, and I think barrelfish is also insipred by exokernel. But the implementation could be very difficult, and there will be lots of context switching and IPC in exokernel, which will hurt the performance, and direct access to hardware resources raises potential security issues.
The Exokernel architecture allows for significant performance, flexibility, efficiency improvements as the paper says, for example, the author implements vm management at application level and proves that it is faster than Ultrix.
However, there might be some weakness of the solution. Implementing the applications based on Exokernel can be complex, developers need precise control over resources and robust mechanisms to ensure security and isolation. And exposing resources to applications increases the risk of security issues.
The paper is also engaging to read, though it was published in 1995, the innovation is still intriguing. The paper has had profound impacts on the way researchers and developers think about resource management, security, and the role of the operating system kernel.
 -->
