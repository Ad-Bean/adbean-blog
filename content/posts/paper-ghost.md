+++
title = 'Paper Reading: ghOSt: Fast & Flexible User-Space Delegation of Linux Scheduling'
date = 2024-10-07T00:51:29-04:00
draft = false
tags = ['Paper Reading']
+++

## ghOSt: Fast & Flexible User-Space Delegation of Linux Scheduling

Google 的 ghOSt，在这之前还有 Snap MicroKernel Approach to Host Networking，是一个用户网络协议栈，支持 RDMA 等等。

ghOSt 更进一步，可以兼容 Linux，更加容易使用，更灵活，灵感来自微内核，OS 委托 delegate 给用户空间。

而且 Shenango, Caladan, Shinjuku 等提出了可以使用 centralized model 来单独调度，可以达到微秒级别的 work-conservation

也有 IX, ZygOS, Shinjuku, Shenango, Caladan, Snap 提出了 container-like data plane os

> 是 LibOS 那一套吗，用于 kernel-bypass
>
> ghOSt 的想法还是没有太天马行空，是很实际的，扩展 Linux 而不是从头搞，可能考虑到 google 内部需要用于生产环境？

## Abstract

我们推出了 ghOSt，这是我们的基础设施，用于将**内核调度决策委托给用户空间代码**

在定制的**数据平面操作系统**中采用**专门的调度策略**可以在数据中心环境中提供令人信服的性能结果

ghOSt 在 Linux 环境中提供了将调度策略普遍委托给用户空间进程的功能。ghOSt 提供了 state encapsulation, communication, and action mechanisms 允许在用户空间代理内部复杂地表达调度策略，同时协助同步。

程序员可以使用任何语言来开发和优化策略，这些策略可以在不重启主机的情况下进行修改。

ghOSt 支持广泛的调度模型，from per-CPU to **centralized**run-to-completion to preemptive, and incurs low overheads for scheduling actions。我们在包括 Google Snap 和 Google Search 在内的学术和现实世界工作负载上展示了 ghOSt 的性能。我们证明，通过使用 ghOSt 而不是内核调度器，我们可以迅速达到相当的吞吐量和延迟，同时使 policy optimization, non-disruptive upgrades, and fault isolation 成为可能，从而为我们的数据中心工作负载服务。我们开源了我们的实现，以促进基于 ghOSt 的未来研究和开发

> schedule policy，不需要重启，怎么做到的
>
> ghost 本身不是 policy，而是 process 支持 bpf
>
> ghost 使用 txn 来进行调度提交

## Introduction

CPU scheduling plays an important role in application performance and security.

Tailoring policies for specific workload types can substantially improve key metrics such as latency, throughput, hard/soft real-time characteristics, energy efficiency, cache interference, and security.

尽管 Linux 支持多种调度实现，但为每个应用程序量身定制、维护和部署不同的机制在大型集群中是难以管理的。

在这篇论文中，我们介绍了 ghOSt 的设计及其在生产环境和学术用例中的评估。ghOSt 是一个用于原生 OS 线程的调度器，它将策略决策委托给用户空间。ghOSt 的目标是根本改变调度策略的设计、实现和部署方式。ghOSt 提供了用户空间开发的灵活性和易于部署的优势，同时仍然能够进行微秒级调度。ghOSt 为用户空间软件定义复杂的调度策略提供了抽象和接口，从每 CPU 模型到系统范围内的（集中式）模型。重要的是，ghOSt 将内核调度机制与策略定义解耦。机制位于内核中，很少改变。策略定义位于用户空间，并迅速改变。

> ghOSt 给我的感觉更像是把 snap 之类的或者微内核落地了，并且兼容 Linux

通过调度原生线程，ghOSt 无需任何更改即可支持现有应用程序。在 ghOSt（图 1）中，调度策略逻辑在正常 Linux 进程中运行，称为代理，这些代理通过 ghOSt API 与内核交互。

内核通过**消息队列**（第 3.1 节）在异步路径中通知代理所有托管线程的状态变化——例如，线程创建/阻塞/唤醒。

然后代理使用基于**事务**的 API 在同步路径中提交线程的调度决策（第 3.2 节）。

ghOSt 支持多个策略的并发执行、故障隔离以及为不同应用程序分配 CPU 资源。重要的是，为了使现有系统平滑过渡到使用 ghOSt，它可以与其他在内核中运行的调度器共存，如 CFS（第 3.4 节）。

我们证明了 ghOSt 使我们能够定义各种策略，其性能与现有调度器在学术和生产工作负载上的表现相当或更好（第 4 节）。我们描述了关键 ghOSt 操作的开销（第 4.1 节）。我们展示了 ghOSt 的**开销很小**，从 265 纳秒的消息传递，到进入代理的上下文切换需要几百纳秒，再到 888 纳秒调度一个线程，使得 ghOSt 的调度开销仅略高于现有内核调度器。通过摊销，这些开销允许单个 ghOSt 代理每秒调度超过 200 万条线程（图 5）。我们通过实现**集中式和抢占式策略**来评估 ghOSt，这些策略在存在高请求分散度和对抗者的情况下实现了高吞吐量和低尾部延迟（第 4.2 节）。

我们将 ghOSt 与 Shinjuku，一个专门的现代数据平面进行比较，以展示 ghOSt 的最小开销和具有竞争力的微秒级性能（与 Shinjuku 相差不超过 5%），同时支持**更广泛的工作负载集，包括多租户**。我们还为 Snap，我们在数据中心使用的生产包交换框架实现了一个策略，导致与 MicroQuanta，我们目前使用的软实时调度器相比，在某些情况下尾部延迟降低了 5-30%（第 4.3 节）。然后，我们为运行 Google Search 的机器实现了 ghOSt 策略（第 4.4 节）。通过根据机器拓扑定制策略，总共不到 1000 行代码，我们证明了 ghOSt 匹配了现有调度器提供的吞吐量，而且通常将延迟提高了 40-50%。最后，我们为虚拟机实现了一个 ghOSt 策略，该策略对最近发现的微体系结构漏洞具有安全性，并显示 ghOSt 的性能与纯内核策略实现具有竞争力。借助 ghOSt，以前需要广泛内核修改的调度策略现在只需几十或几百行代码就能实现。

## Background & Design Goals

Scheduling in Linux: Linux supports implementing multiple policies via scheduling classes.
这些类按优先级排序：使用较高优先级类调度的线程将抢占使用较低优先级类调度的线程。为了优化特定应用的性能，开发者可以修改调度类以更好地满足应用的需求，例如，优先处理应用的线程以减少网络延迟[21]，或使用实时策略调度应用的线程以满足数据库查询的截止时间[47]。

然而，实现在内核中实现和维护这些策略的复杂性导致许多开发者选择使用现有的通用策略，如 Linux 完全公平调度器（CFS [7]）。因此，Linux 中现有的类设计旨在支持尽可能多的用例。这就使得使用这些**过于通用的类**来优化高性能应用以充分利用硬件变得具有挑战性。

> CFS 完全公平调度
>
> 感觉这方向的论文，大部分都是让通用变得专用，一小部分让几个专用变成通用

Implementing schedulers is hard: ghOSt 使调度器的更新、测试和调优成为可能，而无需更新内核和/或重启机器和应用程序。

Deploying schedulers is even harder: ghOSt 是一个运行在用户空间的内核调度器，调度原生 OS 线程

User-level threading is not enough: ghOSt enables the best of both worlds by guaranteeing control over response time while allowing flexible sharing of CPU resources.

> ghost 可以调度用户级线程吗？

Custom scheduler/**data plane** OS per workload is impractical: Shinjuku、Shenango 内核实现不易更改，且对于其他工作负载的性能较差。 Shenango [1]有 8,399 行代码，只能为网络工作负载实现单一的线程调度策略。添加额外的策略需要大量的代码修改。此外，这两个系统都无法在没有特定 NICs 的机器上运行，且 Shenango 无法调度非网络工作负载。

Custom scheduling via BPF is insufficient: 一个吸引人的方式来定制内核调度是将 BPF [61]程序注入内核调度器，就像为其他内核子系统所做的那样[62]。确实可以实现一个 Linux 调度器类，其函数指针调用 BPF 程序来决定下一个运行的线程。不幸的是，BPF 在表达能力和可访问的内核数据结构方面有限制。例如，BPF 验证器必须能够确定循环将退出，BPF 程序不能使用浮点数。

更重要的是，BPF 程序是同步运行的，这意味着它们必须快速响应调度事件，直到完成前会阻塞 CPU。相比之下，**异步调度器**可以在稍后的时间接收调度事件并对此做出反应。因此，如第 3.3 节所述的全局调度器这样的异步模型，可以根据由多个调度事件组成的系统更广泛的视角来做出调度决策。话虽如此，BPF 在 ghOSt 中扮演了重要角色，**用于加速快速路径操作**，并在内核性能关键点提供定制化，正如第 3.2 节所解释的。

> BPF 调度应该是有前景的？

### Design Goals

Policies should be easy to implement and test: implementing systems in **userspace** rather than kernel space simplifies development and enables faster iteration as popular languages and tools can be used.

Scheduling expressiveness and efficiency.

Enabling scheduling decisions beyond the **per-CPU** model: The Linux scheduler ecosystem implements scheduling algorithms that make per-CPU scheduling decisions, 最近的工作表明，通过集中式、专用轮询调度线程[1, 21, 25]，即采用集中式模型，可以为微秒级工作负载带来显著的尾部延迟改进。

Supporting multiple concurrent policies

Non-disruptive updates and fault isolation

> 怎么做到无中断更新和故障隔离？虚拟化？

## Design

ghOSt overview:

图 1 总结了 ghOSt 的设计。**用户空间代理**做出调度决策，并指示内核如何在 CPU 上调度原生线程。ghOSt 的内核侧作为一个调度类实现，类似于常用的 CFS 类。这个调度类为用户空间代码提供了一个丰富的 API 来定义任意的调度策略。为了帮助代理做出调度决策，内核通过消息和状态字向代理暴露线程状态（第 3.1 节）。然后，代理通过事务和系统调用来指导内核的调度决策（第 3.2 节）。

本节将使用两个示例：**per-CPU 调度和集中式调度**。传统的内核调度策略，如 CFS，是每 CPU 调度器。尽管这些策略通常采用负载均衡和工作窃取来平衡系统内的负载，但它们仍然从每 CPU 的角度操作。集中式调度的例子类似于 Shinjuku [25]、Shenango [1]和 Caladan [21]中提出的模型。在这种情况下，有一个单一的全局实体不断地观察整个系统，并为其管辖下的所有线程和 CPU 做出调度决策。

Threads and CPUs:

ghOSt 调度原生线程在 CPU 上。本节提到的所有线程都是原生线程，与第 2 节提到的用户级线程相对。我们将逻辑执行单元称为 CPU。例如，我们认为一台拥有 56 个物理核心和 112 个逻辑核心（超线程）的机器有 112 个 CPU。

> 所以不能调度用户线程？

Partitioning the machine.

ghOSt supports multiple concurrent policies on a single machine using **enclaves**
系统可以按照 CPU 粒度划分为多个独立的 enclave，每个 enclave 运行自己的策略，如图 2 所示。从调度的角度来看，这些 enclave 是隔离的。当在单台机器上运行不同的工作负载时，分区尤其有意义。通常可以通过机器拓扑设置这些 enclave 的粒度，如每个 NUMA 插槽或每个 AMD CCX [40]。enclave 也有助于隔离故障，将代理崩溃的损害限制在其所属的 enclave 内

> enclaves 是什么

ghOSt userspace agents.

调度策略逻辑是在用户空间代理中实现的

为了实现容错和隔离，如果一个或几个代理崩溃，系统将回退到默认调度器，如 CFS。此时，机器仍完全功能正常，直到启动新的 ghOSt 用户空间代理——无论是最后一个已知的稳定版本还是带有修复的新修订版。得益于崩溃恢复的特性，更新调度策略相当于重新启动用户空间代理，而**无需重启机器**。这一特性使得可以针对各种硬件和工作负载进行实验和快速策略定制。开发者可以对策略进行微调并简单地重新启动代理。关于 ghOSt 策略的动态更新将在第 3.4 节中讨论。

> 崩溃后怎么回退，是全部回退到 per-CPU 还是什么，如果之前的是 centrialized 的怎么办

无论调度模型是每 CPU 还是集中式，每个由 ghOSt 管理的 CPU 都有一个**本地代理**，如图 2 所示。在每 CPU 的情况下，每个代理负责其自身 CPU 的线程调度决策。在集中式的情况下，一个单一的全局代理负责 enclave 中所有 CPU 的调度。所有其他本地代理都是不活跃的。每个代理都在一个 Linux pthread 中实现，所有代理都属于同一个用户空间进程。

> agent 到底代表什么，调度器？

### Kernel-to-Agent Communication

Exposing thread state to the agents. 为了让代理能够对其管辖下的线程做出调度决策，内核必须向代理暴露线程状态。 一种方法是将现有的内核数据结构内存映射到用户空间，例如 task_structs，这样代理可以检查它们以推断线程状态。然而，这些数据结构的可用性和格式在不同的内核和内核版本之间有所不同，紧密地将用户空间策略实现与内核版本耦合在一起。另一种方法是通过 `sysfs` 文件暴露线程状态，采用`/proc/pid/...`的形式。然而，文件系统 API 对于快速路径操作效率低下，难以支持微秒级的策略：`open/read/fseek` 最初是为块设备设计的，速度太慢且复杂（例如，需要错误处理和数据解析）。

ghOSt messages. 在 ghOSt 中，内核使用表 1 中列出的消息通知用户空间代理线程状态的变化。例如，如果一个线程被阻塞但现在准备运行，内核会发布一个 `THREAD_WAKEUP` 消息。此外，内核还通过 `TIMER_TICK` 消息通知代理定时器滴答。为了帮助代理验证它们的决策基于最新的状态，消息还包括序列号，这一点将在后面解释。

Message queues: 消息通过消息队列传递给代理。在集中式例子中，所有线程都被分配到全局队列（图 2，右）。关于 CPU 事件的消息，如 TIMER_TICK，会被路由到与 CPU 相关联的代理线程的队列。

> 这个很有意思，这些队列从哪里来的？

虽然实现队列的方法有很多，但我们选择在**共享内存中使用自定义队列**，以高效地处理代理唤醒（如下所述）。我们认为现有的队列机制对于 ghOSt 来说是不足的，因为它们仅存在于**特定的内核版本**中。例如，BPF 系统通过 BPF 环形缓冲区[BPF ring buffers][65]将 BPF 事件传递到用户空间，最近版本的 Linux 还通过 io_uring[66]将异步 I/O 消息传递到用户空间。这两种都是**快速的无锁环形缓冲**区，同步消费者/生产者访问。然而，旧版本的 Linux 内核和其他操作系统不支持它们。

> BPF ring buffer, io_uring
>
> 所以 ghost 是自己做的，而不是用 bpf, io_uring 吗

Thread-to-queue association.

在 ghOSt 的 enclave 初始化之后，enclaves 中有一个默认队列。代理进程可以使用 CREATE/DESTROY_QUEUE() API 创建/销毁队列。添加到 ghOSt 的线程默认被分配为向默认队列发送消息。代理可以通过 ASSOCIATE_QUEUE()改变这种分配。

Queue-to-agent association.

队列可以配置为当消息被生产到队列时唤醒一个或多个代理。代理可以通过 CONFIG_QUEUE_WAKEUP()配置唤醒行为。在每 CPU 的例子中，每个队列与一个 CPU 关联，并配置为唤醒相应的代理。在集中式例子中，**队列由全局代理连续轮询，因此唤醒是多余的，因此没有配置。**消息被生产到队列并在代理中观察到的延迟将在第 4.1 节中讨论。

**代理唤醒使用标准的内核机制来唤醒阻塞的线程**。这涉及识别要唤醒的代理线程，将其标记为可运行，可选地向目标 CPU 发送中断以触发重新调度，并执行上下文切换到代理线程。

> 这里有什么办法绕过中断和上下文切换吗

Moving threads between queues/CPUs.

在我们的每 CPU 例子中，为了实现 CPU 之间的负载均衡和工作窃取，代理可以通过 ASSOCIATE_QUEUE()改变线程消息到队列的路由。正确的跨队列到代理的消息路由协调取决于代理实现（在用户空间）。如果一个线程在原始队列中有待处理消息时从一个队列关联到另一个队列，关联操作将会失败。在这种情况下，代理必须排空原始队列，然后再重新发出 ASSOCIATE_QUEUE()。

> 协程、用户级线程是如何处理的？

Synchronizing agents with the kernel

代理根据通过消息观察到的系统状态做出调度决策

每个代理都有一个序列号，记为 𝐴𝑠𝑒𝑞，每当有消息被发布到与该代理关联的队列时，这个序列号就会递增。

Exposing sequence numbers via shared memory

ghOSt 允许代理通过状态字高效地轮询有关线程和 CPU 状态的辅助信息，这些状态字被映射到代理的地址空间中。为了简洁起见，我们只讨论如何使用状态字向代理暴露序列号 𝐴𝑠𝑒𝑞 和 𝑇𝑠𝑒𝑞。当内核更新线程或代理的序列号时，它也会更新相应状态字。然后，代理可以从共享映射中的状态字读取序列号。

> 轮询线程和 CPU 状态，那能用复杂结构实现 epoll 吗

### Agent-to-Kernel Communication

We now describe how the agents instruct the kernel which thread to schedule next.

Sending scheduling decisions via transactions:

Agents send scheduling decisions to the kernel by committing transactions

代理通过提交事务向内核发送调度决策。代理必须能够调度其本地 CPU（每 CPU 的情况）以及其他远程 CPU（集中式的情况）。提交机制必须足够快，以支持微秒级的策略，并且能够扩展到数百个核心。

顺便提一句，将调度接口设计为共享内存中的事务，未来可以将调度决策卸载到可以访问该内存的外部设备上。

> transactions in shared memory

受到事务内存[67]和数据库系统[68]的启发，我们设计了自己的事务 API，通过共享内存实现。这些系统支持快速的分布式提交操作，具有原子性语义，并且可以有多个提交同时针对同一个远程节点。ghOSt 代理也需要类似的属性。

代理使用 TXN_CREATE()辅助函数在共享内存中打开一个新的事务。代理写入要调度的线程的 TID 以及要调度线程的 CPU 的 ID。在每 CPU 的例子中，每个代理只调度自己的 CPU。当事务填写完毕后，代理通过 TXNS_COMMIT()系统调用将其提交给内核，这会触发提交过程并促使内核发起上下文切换。图 3 展示了一个简化的例子。

Group commits:

在集中式调度的例子中，为了让 ghOSt 能够扩展到数百个 CPU 和每秒数十万次的事务，我们必须减轻系统调用的高昂成本。我们通过引入组合提交来分摊事务的成本。组合提交还减少了发送到其他 CPU 的中断数量，类似于 Caladan [21]的做法。代理通过将所有事务传递给 TXNS_COMMIT()系统调用来提交多个事务。这个系统调用将高昂的开销分摊到多个事务上。最重要的是，它通过使用大多数处理器中存在的批量中断功能来分摊发送中断的开销。内核不会为每个事务发送多个中断，而是向远程 CPU 发送一个批量中断，从而节省大量开销。

> 这里的事务是为了什么，隔离还是并发？

Sequence numbers and transactions.

在每 CPU 的例子中，提交事务的代理正在将其 CPU 让给它正在调度的目标线程。当代理正在运行时，队列中新发布的消息不会引起唤醒，因为代理已经在运行。然而，**队列中的新消息可能是来自优先级更高的线程**，并且如果代理知道这一点的话，会影响调度决策。代理只有在下次唤醒时才有机会检查该消息，而这已经太晚了。我们现在通过序列号来解释如何解决这一挑战，适用于每 CPU 的例子。集中式调度的略有不同情况将在第 3.3 节中解释。

我们使用代理序列号 𝐴𝑠𝑒𝑞 来解决这一挑战。代理通过检查代理线程的状态字来轮询其 𝐴𝑠𝑒𝑞。回想一下，当新的消息被发布到与代理关联的队列时，𝐴𝑠𝑒𝑞 会递增。操作顺序是：1) 读取 𝐴𝑠𝑒𝑞；2) 从队列中读取消息；3) 做出调度决策；4) 将 𝐴𝑠𝑒𝑞 与事务一起发送给 TXNS_COMMIT()。如果与事务一起发送的 𝐴𝑠𝑒𝑞 比内核观察到的当前 𝐴𝑠𝑒𝑞 旧（即，有新消息被发布到代理的队列），则事务被认为是“过期”的，并将以 ESTALE 错误失败。然后，代理将清空其队列以检索较新的消息，并重复该过程。

> 这里的 challenge 没太理解，没有具体的例子

Accelerating scheduling with BPF:

ghOSt 提供的用户级灵活性并非没有代价：message delivery 和 group scheduling 最多可能耗费 5 微秒（参见第 4.1 节中的表 3）；在集中式调度模型中，一个线程可能要等到整个集中式调度循环结束才能为其提交调度决策（在第 4.4 节中为 30 微秒）。

> 5us 和 30us 看上去很低，为什么说这是代价？

### The Centralized Scheduler

One global agent with a single queue: 在集中式调度中，存在一个单一的全局代理轮询单个消息队列，并为所有受 ghOSt 管理的 CPU 做出调度决策。如果指定的 CPU 上已经运行了一个 ghOSt 线程，事务将抢占之前的线程，转而运行新的线程。

Avoiding preemption of the global agent: 为了支持微秒级调度，全局代理必须持续运行，因为任何抢占都会直接导致调度延迟。

> 怎么知道是全局代理？最高优先级？

Sequence numbers and centralized scheduling: 在某些时候，**全局代理**可能对某个线程的状态有一个不一致的视图。例如，线程 𝑇 可能发布了 `THREAD_WAKEUP` 消息。全局代理接收到这条消息并决定在 𝐶𝑃𝑈𝑓 上调度 𝑇。与此同时，系统中的某个实体调用了 `sched_setaffinity()`，导致发布了 `THREAD_AFFINITY` 消息，禁止 𝑇 在 𝐶𝑃𝑈𝑓 上运行。我们需要一种机制来确保将 𝑇 调度到 𝐶𝑃𝑈𝑓 的事务会失败。

我们通过线程序列号来解决这个问题。回想一下，每个排队的消息 𝑀𝑇 都附带了线程序列号 𝑇𝑠𝑒𝑞，形式为（𝑀𝑇, 𝑇𝑠𝑒𝑞）。当代理为线程 𝑇 提交事务时，它会将事务连同它所知的最新线程序列号 𝑇𝑠𝑒𝑞 一起发送。当内核接收到事务时，它会验证事务中的线程的 𝑇𝑠𝑒𝑞 是否是最新的。如果不是，事务将以 ESTALE 错误失败。

> 这就是用事务的原因吗，避免 centralized scheduling 导致 global agent have inconsistent view of a thread state

### Fault Isolation and Dynamic Upgrades

Interaction with other kernel scheduling classes: 我们希望避免 ghOSt 线程对其他线程造成意外后果，比如饥饿、优先级反转、死锁等。

我们通过将 ghOSt 的内核调度类分配一个**比默认调度类（通常是 CFS）更低的优先级**来实现这一目标。结果是，系统中的大多数线程会抢占 ghOSt 线程。ghOSt 线程的抢占会导致创建一个 `THREAD_PREEMPT` 消息，触发相关的代理（该代理运行在不同的高优先级调度类中）做出调度决策。代理进一步决定如何处理抢占。

> ghost 导致死锁、饥饿、优先级反转的原因是什么？抢占？

Dynamic upgrades and rollbacks. ghOSt 通过以下两种方式之一实现动态升级：

(a) 替换代理，同时保持 enclave 基础设施完好；
(b) 销毁 enclave 并重新开始。

> 没理解，这说的太简单了，怎么替换，怎么保持 enclave, enclave 也没感觉好好介绍到底是什么

ghOSt watchdog. ghOSt 或任何其他内核调度器中的调度错误可能有系统范围的影响。例如，一个 ghOSt 线程可能在持有内核互斥锁时被**抢占**，如果它长时间未被调度，可能会间接地**阻塞**其他线程，包括那些在 CFS 或其他 ghOSt 飞地中的线程。类似地，如果关键线程（如垃圾收集器和 I/O 轮询器）未被调度，机器将陷入停滞。作为一种安全机制，ghOSt 会自动销毁行为异常的代理飞地。例如，当内核检测到代理在用户可配置的毫秒数内未调度可运行线程时，会销毁飞地。

> halt 所以需要 linux 来调度？还是 ghost 怎么自动销毁，这个配置怎么调才合适呢

## Evaluation

我们的 ghOSt 评估集中在三个问题上：

(a) ghOSt 特有的操作有哪些开销，这些操作在经典调度器中不存在（第 4.1 节）；
(b) 使用 ghOSt 实现的调度策略与先前的工作（如 Shinjuku [25]）相比表现如何（第 4.2 节）；
(c) ghOSt 是否是大规模和低延迟生产负载的有效解决方案，包括 Google Snap（第 4.3 节）、Google Search（第 4.4 节）和虚拟机（第 4.5 节）？

> 特有的操作，比如消息传递
>
> 我还想看一个 evaluation 就是论文提到的死锁、优先级反转，能不能设计一个实验来复现，观察一下回退导致的问题

### Analysis of ghOSt Overheads and Scaling

LOC: ghOSt 已准备好用于生产，并且支持我们生产负载的一系列调度策略，比 CFS 少 40%的内核代码。

Experimental Setup: 实验在 Linux 4.15 上运行

> 4.15 是比较老的，19 年 5.0 发布，21 年应该 5.15 也发布了，24 年现在是 6.12，在 6.6 引入了新的 EEVDF process scheduler(Earliest eligible virtual deadline first scheduling) 替代 CFS，毕竟 CFS 是很老了
>
> https://lwn.net/Articles/925371/ An EEVDF CPU scheduler for Linux
>
> CFS 追踪了每个进程获得的时间，实现公平调度
>
> EEVDF 提出更多来解决一些约束，比如最大限度利用系统的内存缓存，最小化进程在 CPU 之间的移动
>
> 让所有 CPU 保持忙碌
>
> 但有种情况：有些进程不需要很多 CPU time 但是确实需要，是一种 latency requirements 所以需要优先级，但这是特权操作？
>
> 对于每个进程，EEVDF 计算该进程应该获得的时间与实际获得的时间之间的差异; 这种差异称为“滞后”。
>
> 是为了解决 CFS 的一些启发式补丁吗？

Message delivery overhead: 在每 CPU 的例子中，**向本地代理传递消息包括将消息添加到队列**、**上下文切换**到本地代理以及从队列中移除消息。开销（725 纳秒）主要由**上下文切换**（410 纳秒）主导。在集中式例子中，向**全局代理传递消息**（265 纳秒）包括将消息添加到队列并在全局代理中移除消息，全**局代理始终处于自旋状态**。

Local scheduling: 在每 CPU 模型中，这是提交事务并在本地 CPU 上执行上下文切换，直到目标线程运行的开销。开销（888 纳秒）略高于 CFS 上下文切换开销（599 纳秒），主要是由于**事务提交**，但仍具有竞争力。

> 事务带来了一些问题

Remote scheduling: 在集中式调度模型中，代理端提交事务并发送**跨处理器中断**（IPI）。目标 CPU 处理 IPI 并执行上下文切换。代理的开销（668 纳秒）设定了每个代理每秒 1.5M 可调度线程的理论最大吞吐量。将 10 个不同 CPU 的事务组合起来，通过分摊 IPI 开销，理论最大值提高到 10 \* 10^9 / 3964 = 2.52M 每秒可调度线程。根据这些数字，单个代理理论上可以在 100 个 CPU 的服务器上每秒调度大约 25,200 个线程。如果线程长度为 40 微秒，代理可以保持 100 个 CPU 忙碌。policy 开发者在设计 ghOSt 策略时应考虑每个代理的可扩展性限制。随着更多代理的加入，这一限制相对线性地改善。

> 这个 inter-processor interrupt (IPI) 会产生什么问题吗

Global agent scalability (Fig. 5). To show how a global agent scales, we analyze a simple round-robin policy.

### Comparison to Custom Centralized Schedulers

Systems under comparison.

We compared three implementations of the scheduling approach in Shinjuku [25], all serving a **RocksDB** workload [70]. We use one physical core for load generation with all systems. The Shinjuku system runs on Linux 4.4 as its Dune [35] driver fails to compile for newer versions. The other systems under comparison (ghOSt-Shinjuku and CFS-Shinjuku) run on Linux 4.15 with our ghOSt patches applied.

> RocksDB workload，这个有意思

![](https://s2.loli.net/2024/10/08/Aiu8GBzKFoPpg9w.png)

Single Workload Comparison:

99.5% of requests - 4 𝜇s, 0.5% of requests - 10 ms.

反映了 ghOSt 在调度每个请求的线程时的额外开销，而 Shinjuku 则在自旋线程之间传递请求描述符。由于缺乏抢占，CFS-Shinjuku 比其他两个系统早约 30%达到饱和。

> 没看懂这个数据集。实验结果还是纯 shinjuku 比较好？是因为调度和消息传递的开销吗。

Multiple Workloads Comparison:

图 6c 显示，当我们把一个批处理应用程序与由 Shinjuku 管理的 RocksDB 工作负载共置时，即使 RocksDB 负载较低，批处理应用程序也无法获得任何 CPU 资源。

为了实现低延迟和**批处理工作负载**的安全共置，可以考虑使用**面向线程的集中式调度系统**，如 Shenango [1]。Shenango 的集中式调度器监控网络应用程序的负载，当应用程序负载较轻时，调度器将多余的 CPU 周期分配给批处理应用程序。然而，Shenango 不适合处理执行时间变化较大的请求，因此 RocksDB 工作负载的尾部延迟会远高于 Shinjuku。

将多余的 CPU 周期分配给批处理应用程序

> shinjuku 的缺点使用 ghost 可以解决

### Google Snap

类似于 DPDK [72]，Snap 维护了轮询（工作）线程，负责与 NIC 硬件的交互，并代表重要服务运行自定义的网络和安全协议。Snap 可能会根据网络负载的变化决定生成或合并工作线程。

How are worker threads scheduled today?

> 和 Snap 的对比，感觉 ghost 就是用来增强 snap 的

![](https://s2.loli.net/2024/10/08/apy6oi3BG28Urxb.png)

> 在 64B 情况下

因此，当流量突发且包含大量 64B 消息时，ghOSt 的调度事件开销变得明显。对于 64KB 消息，ghOSt 在 99 百分位延迟以内表现与基线相似（在 15%范围内）。对于 99.9 百分位及以上，ghOSt 的尾延迟比基线低 5%到 30%。64KB 消息需要更多的处理（用于复制数据），因此导致较少的调度事件。ghOSt 在某些情况下表现优于 MicroQuanta 的原因是它可以在线程可用时重新定位工作线程。

> 其实看不太懂这个对比，是说 p99999 的表现更好吗
>
> 但是这也太严格了，只有 0.001% 的请求超过了延迟？p99 其实都差不多，但是 p999 好一点

### Google Search

> 主要是尾延迟，tail latency 表现很不错就不看了
>
> QPS、吞吐量和 CFS 没太差

### Protecting VMs from L1TF/MDS Attacks

> 这一章没理解，怎么突然多一个安全测试

使用三种调度策略：1) CFS，不提供对推测执行攻击的保护；2) 内核中的安全虚拟机核心调度；3) ghOSt 中的安全虚拟机核心调度。结果如表 4 所示。CFS 提供了更好的整体性能，但没有任何安全性。ghOSt 策略缓解了跨超线程攻击，并且在考虑 ghOSt 中的额外上下文切换开销的情况下，性能与内核中的核心调度版本相当。

> 主要是前文只是稍微提了一下安全、隔离，但是实现讲的比较模糊

## Future Work

Accelerating scheduling with BPF：通过将一些代理职责委托给同步 BPF 回调来加速 ghOSt 是一个开放的研究领域。在§4.4 中，全局代理的调度循环耗时 30 微秒，这可能导致调度间隙。实际上，系统中的一些线程在阻塞之前只运行 5-30 微秒，导致这些间隙期间 CPU 空闲。我们可以使用§3.2 中描述的集成 BPF 程序来缓解这些调度间隙。

BPF 程序通过共享内存与用户空间通信，使用多个多生产者、多消费者环形缓冲区。代理将可运行的线程插入这些缓冲区，BPF 尝试运行它们。代理可以在 BPF 调度线程之前撤销线程。例如，全局代理可以为每个 NUMA 节点使用一个环形缓冲区；全局代理可以跟踪每个线程的首选 NUMA 节点，并在两个环之间进行负载均衡。

> 能用 bpf 来 bypass 一些上下文切换吗？

Tick-less scheduling

当 ghOSt 处于集中模式时，可以禁用所有 CPU 上的定时器滴答，以避免在虚拟机工作负载中出现昂贵的 VM 退出。在经典的每个 CPU 调度器中，滴答每毫秒触发一次调度器，以确保所有虚拟机之间的轮转抢占。不幸的是，这些滴答会导致虚拟机退出到主机内核上下文。

> 这个完全看不懂

## Related Work

## Conclusion
