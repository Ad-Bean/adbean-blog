+++
title = 'Paper Reading: The eXpress Data Path'
date = 2024-09-18T00:00:33-04:00
draft = false
tags = ['Paper Reading']
+++

## The eXpress Data Path: Fast Programmable Packet Processing in the Operating System Kernel

ACM coNEXT 2018，在内核层面快速处理 packet，开源很完善，与 dpdk 之类的区别在于不是纯在用户态处理，也不是用户态协议栈。

而是利用 eBPF 在内核态执行。

> 明明不算很高强度看了两周论文，已经不适了。可能是不太懂 OS + Network 相关的吧，实在是太难了，很多概念都没接触过，只是入个门，比如这篇的 BPF 就没接触过。没写过没认真研究过的东西需要在几个小时弄明白太困难了。

> XDP 是可以在 Linux 开启的，应该是应用比较广泛的，https://www.datadoghq.com/blog/xdp-intro/，分三种，一个是加载到网卡驱动，一个是加载到网卡，一个是 Linux 协议栈入口（性能-通用 tradeoff）
>
> AF_XDP 是 XDP 技术的一种应用场景，AF_XDP 是一种高性能 Linux socket。
>
> 主要几个缺点是，没有缓存队列，可能不太适合高延迟的设备，而且没有网络栈通用吧
>
> https://github.com/facebookincubator/katran
>
> Meta 基于 BPF + XDP 实现的高性能 load balancer，尤其是在 XDP 驱动模式下表现非常快。

## ABSTRACT

Programmable packet processing is increasingly implemented using kernel bypass techniques, where a userspace application takes complete control of the networking hardware to avoid expensive context switches between kernel and userspace

application isolation and security mechanisms and well-tested configuration, deployment and management tools cease to function

> 绕过操作系统会带来什么，隔离、安全、配置？有具体例子吗，比如防火墙、ACL、还是权限、进程隔离？

we present the design of a novel approach to programmable packet processing, called the **eXpress Data Path** (XDP)

XDP is part of the **mainline Linux kernel** and provides a fully **integrated** solution working in concert with the kernel’s networking stack. Applications are written in higher level languages such as C and compiled into custom byte code which the kernel statically analyses for safety, and translates into native instructions

## INTRODUCTION

DPDK bypass the operating sys- tem completely, instead passing control of the network hardware directly to the network application and dedicating one, or several, CPU cores exclusively to packet processing.

内核旁路方法可以显著提高性能，但是有一个缺点，那就是更难与现有系统集成，应用程序必须重新实现操作系统网络栈提供的功能，比如路由表和更高级别的协议。

在最坏的情况下，这将导致一种场景，即数据包处理应用程序在一个完全独立的环境中运行，由于需要直接的硬件访问，操作系统提供的熟悉的工具和部署机制无法使用。这会增加系统的复杂性，模糊操作系统内核强制执行的安全边界。

基础设施正在转向基于容器的工作负载和编排系统(如 Docker 或 Kubernetes) ，在这些系统中，内核在资源抽象和隔离方面发挥着主导作用

> 容器化和虚拟化的区别是什么。

XDP 以运行 eBPF 代码的虚拟机的形式定义一个有限的执行环境，eBPF 是原始 BSD 包过滤器 (BPF) 字节码格式的扩展版本。这个环境在内核本身接触数据包数据之前直接在**内核上下文中执行定制程序**，这使得在从硬件接收数据包之后尽可能早地进行自定义处理(包括重定向)。内核通过在加载时静态验证自定义程序来确保它们的**安全性**; 并且程序被动态地编译成本地机器指令以确保高性能

虽然这并不完全符合在相同硬件上基于 DPDK 的应用程序的最高可实现性能，但我们认为 XDP 系统通过提供几个比 DPDK 和其他**内核旁路解决**方案更引人注目的优势来弥补这一点

> 性能还是不如 DPDK

- 与常规网络堆栈协同集成，在内核中保留对硬件的完全控制。这保留了**内核的安全边界**，并且不需要对网络配置和管理工具进行任何更改。此外，任何具有 Linux 驱动程序的网络适配器都可以被 XDP 支持；**不需要特殊的硬件功能**，现有的驱动程序只需要进行修改以添加 XDP 执行钩子。
- 使得可以选择性地利用内核网络堆栈功能，如路由表和 TCP 堆栈，保持相同的配置接口，同时加速关键性能路径。
- 保证 eBPF 指令集和与其一起公开的编程接口（API）的稳定性。
- 在与基于正常套接层的工作负载交互时，不需要将数据包从用户空间**重新注入内核空间**，从而避免了昂贵的操作。
- 对主机上运行的应用程序是透明的，使得新的部署场景成为可能，例如在服务器上进行内联保护以抵御拒绝服务攻击。
- 可以动态重新编程而不会中断任何服务，这意味着可以在不中断网络流量的情况下动态添加或完全移除功能，并且处理可以根据系统其他部分的状况动态反应。
- 不需要将完整的 CPU 核心专用于数据包处理，这意味着较低的流量水平直接转化为较低的 CPU 使用率。这对效率和节能有重要影响。

> 不需要特殊 network adapter 和 driver 是很不错的一个点
>
> re-injection from user space into kernel space when interacting with workloads based on the normal socket layer
>
> 这是在哪发生的？是 dpdk 吗
>
> dedicating full CPU cores 也是一个很奇怪的点，low traffic 会占用 dpdk 一个核 100% 吗？

## RELATED WORK

Examples of such applica- tions include those performing single functions, such as switch- ing [47], routing [19], named-based forwarding [28], classifica- tion [48], caching [33] or traffic generation

为了在 Common Off The Shelf 通用现货(COTS)硬件上实现高性能的数据包处理，有必要消除网络接口卡 (NIC) 和执行数据包处理的程序之间的任何瓶颈。由于性能瓶颈的主要来源之一是操作系统内核和运行在其上的用户空间应用程序之间的 interface (因为系统调用的高开销和底层功能丰富的通用堆栈的复杂性) ，低级数据包处理框架必须以这样或那样的方式管理这种开销。支持上面提到的应用程序的现有框架采用多种方法来确保高性能; XDP 基于其中几种方法的技术。在接下来的文章中，我们将简要概述 XDP 与现有最常用框架之间的异同。

DataPlanDevelopmentKit (DPDK)[16]可能是用于高速数据包处理的最广泛使用的框架。它最初是一个 Intel 专用的硬件支持包，但是在 Linux 基金会的管理下已经得到了广泛的应用。DPDK 是一种所谓的内核旁路框架，它将网络硬件的控制从内核移到网络应用程序中，完全消除了内核-用户空间边界的开销。

however, as mentioned in the introduction, it has significant management, maintenance and security drawbacks.

> 还是不懂有什么安全和管理问题，能够具体一点吗，intro 也只是说需要重新实现网络栈、难以集成？

XDP 采用了一种与绕过内核相反的方法: 不是将网络硬件的控制移出内核，**而是将对性能敏感的数据包处理操作移入内核**，并在操作系统网络堆栈开始处理之前执行。这保留了去除网络硬件和包处理代码之间的内核-用户空间边界的优点，同时保持了内核对硬件的控制，从而保留了管理接口和操作系统提供的安全保证。实现这一点的关键创新是使用一个**虚拟执行环境来**验证加载的程序不会损害或使内核崩溃

> 提前处理？还是使用了虚拟环境 virtual execution environment

在引入 XDP 之前，将数据包处理功能作为内核模块实现是一种高成本的方法，因为错误可能使整个系统崩溃，而且内部内核 API 经常发生变化

> 包处理怎么引起系统崩溃？

XDP 通过提供一个安全的执行环境，并得到内核社区的支持，极大地降低了将处理迁移到内核中的应用程序的成本，从而提供了与内核向用户空间公开的其他接口相同的 API 稳定性保证。此外，XDP 程序可以**完全绕过网络堆栈**，这比需要挂接到现有堆栈的传统内核模块提供了更高的性能。

> bypass the networking stack 是怎么做，有类似的吗，是零拷贝吗？AF_XDP？不使用内核的网络栈处理数据包，自己处理？

虽然 XDP 允许数据包处理进入操作系统以获得最大的性能，但它也允许加载到内核的程序有选择地将数据包重定向到一种特殊的**用户空间套接字类型**，这种套接字类型绕过了正常的网络堆栈，甚至可以以**零拷贝**模式进行操作以进一步降低开销。

> 普通 linux NIC 接收到数据包，中断，缓冲区-内核空间，内核空间走网络栈（链路层、网络层、协议层）处理（校验、路由等等）、内核缓冲-用户空间缓冲、socket 接口。
>
> XDP NIC 接收到数据包，中断，缓冲区-内核空间复制，XDP hook 处理包，

programmable hardware achieve high-performance packet processing, NetFPGA,

In a sense, XDP can be thought of as a **“software offload”**, where performance-sensitive processing is offloaded to increase performance, while applications otherwise interact with the regular networking stack

> XDP 怎么知道是 performance-sensitive，还是说全部这么处理？额外处理不需要时间吗

## THE DESIGN OF XDP

This deep integration with the kernel obviously imposes some design constraints

> 什么限制？

![](https://s2.loli.net/2024/09/18/Nz15SqFbRCHoasE.png)

> 在数据包到达时，在接触数据包数据之前，设备驱动程序在主 XDP 挂钩中执行 eBPF 程序。这个程序可以选择丢弃数据包; 将它们发送回原来接收到的接口; 将它们重定向到另一个接口(包括虚拟机的 vNIC) ，或者通过特殊的 AF \_ XDP 套接字发送到用户空间; 或者允许它们进入常规的网络堆栈，
>
> 问题是，hook 的耗时会不会影响正常的包？

**The XDP driver hook** is the main entry point for an XDP program, and is executed when a packet is received from the hardware.

**The eBPF virtual machine** executes the byte code of the XDP program, and **just-in-time-compiles** it for increased performance.

**BPF maps are key/value stores** that serve as the primary communication channel to the rest of the system.

The **eBPF verifier** statically verifies programs before they are loaded to make sure they do not crash or corrupt the running kernel.

![](https://s2.loli.net/2024/09/18/Xb2RGpnD6iwqI1E.png)

> 当数据包到达时，程序首先解析数据包头以提取它将对之作出反应的信息。然后，它从多个源之一读取或更新元数据。最后，可以重写一个数据包，并确定对该数据包的最终判决。该程序可以在数据包解析、元数据查找和重写之间进行交替，所有这些都是可选的。
>
> map 是干嘛的

### The XDP Driver Hook

XDP programs run in the Extended BPF (eBPF) virtual machine. eBPF is an evolution of the original BSD packet filter (BPF) [37] which has seen extensive use in various packet filtering applications over the last decades. BPF uses a register-based virtual machine to describe filtering actions

> register-based virtual machine 会不会出现问题，比如 容量不够什么的

程序通常首先解析数据包数据，并通过尾调用将控制权传递给**不同的 XDP 程序**，从而将处理分成逻辑子单元（例如，基于 IP 头版本）。

## PERFORMANCE EVALUATION

现有系统中有许多用于高性能数据包处理的解决方案，并且在本文范围内对所有这些系统进行基准测试是不现实的。相反，我们注意到 DPDK 是现有解决方案中性能最高的[18]，并将其作为高速软件数据包处理当前技术水平的基准进行比较（使用 DPDK 18.05 版本附带的 testpmd 示例应用程序）。我们专注于原始数据包处理性能，使用合成基准测试，并将其与 Linux 内核网络堆栈的性能进行比较，以展示 XDP 在同一系统中提供的性能改进。在下一节中，我们将通过一些在 XDP 之上实现的实际应用示例来补充这些原始性能基准测试，以展示其在编程模型中的可行性。

- Packet drop performance 数据包丢弃性能。为了展示最大数据包处理性能，我们测量丢弃传入数据包的最简单操作的性能。这实际上测量了整个系统的开销，并作为实际数据包处理应用程序预期性能的上限。

- CPU usage CPU 使用率。如引言中所述，XDP 的优点之一是它根据数据包负载扩展 CPU 使用率，而不是专门为数据包处理分配 CPU 核心。我们通过测量 CPU 使用率如何随提供的网络负载扩展来量化这一点。

- Packet forwarding performance 数据包转发性能。一个不能转发数据包的数据包处理系统实用性有限。由于转发引入了与简单处理情况相比的额外复杂性（例如，与多个网络适配器交互，重写链路层头等），因此单独评估转发性能是有用的。我们在转发评估中包括吞吐量和延迟。

> 奇怪的指标，

我们已经验证，在全尺寸（1500 字节）数据包的情况下，我们的系统可以在半空闲的单个核心上以线速（100 Gbps）处理数据包。这清楚地表明，挑战在于**每秒处理大量数据包**，正如其他人也指出的那样[46]。因此，我们使用**最小尺寸（64 字节）**数据包进行所有测试，并测量系统可以处理的最大数据包数每秒。为了测量性能如何随 CPU 核心数量扩展，我们使用越来越多的专门用于数据包处理的核心重复测试。对于 XDP 和 Linux 网络堆栈（它们没有提供明确的方式来专门为数据包处理分配核心），我们通过配置**硬件接收端扩展（RSS）功能**，将流量引导到每个测试所需的多个核心来实现这一点。

### Packet Drop Performance

### CPU Usage

### Packet Forwarding Performance

> 适用案例：
> 软件路由（XDP routing）
>
> Linux 内核实现了一个功能完全的路由表，生态系统功能丰富，结合 XDP 包处理框架实现了一个完美的路由功能。其性能与常规的 Linux 内核网络栈相比提升了 2.5 - 3 倍左右。

> ACL/DDoS 防御
>
> XDP 可以直接在应用服务器上部署包过滤程序来防御此类攻击，无须修改应用代码。如果应用部署在虚拟机里，XDP 程序还可以部署在宿主机上，保护机器上所有的虚拟机。其性能单核可以轻松处理 10Gbps 的最小包 Dos 流量。这种 DDOS 防御的部署更加灵活。
>
> 相比 iptables 相对较晚的 hook 点，XDP 的丢包速率要比 iptables 高 4 倍左右。

> 负载均衡（load balancing）
>
> 其原理是通过对包头进行哈希，以此选择目标应用服务器，然后将数据包进行封装，发送给应用服务器，应用解封，处理请求，会包给客户端。在次过程中，XDP 服务哈希，封包发送。通过 bpf map 进行配置，其性能比 Linux 内核 IPVS 高 4 倍左右。

> XDP 允许使用 BPF（Berkeley Packet Filter）程序进行数据包处理，这些程序可以动态加载和更新，提供了极大的灵活性。
> 适用性强。高于 4.8 版本的内核和绝大多数高速网卡都是支持 XDP 的，无需专有硬件的支持。

> 2021 年，Yoann Ghigoff 等人更是基于 eBPF 和 XDP、TC 在内核中实现了一层 Memcached 的缓存，达到了比 DPDK 内核旁路方案还要高的性能。智能网卡也开始对 eBPF 卸载进行了支持，将包处理进一步从网卡驱动层卸载到了网卡，释放了更多的主机 CPU 资源，实现更高的性能。我们常用的虚拟交换机 OVS 的团队也在 2.12.0 版本就开始对 AF_XDP 进行探索，在 2021 年 SIGCOMM 会议上，发表了这些年他们对于数据面的探索，将 AF_XDP 选型用于其数据面，解决了很多 DPDK 解决不了的问题。其他的应用场景如负载均衡、流采样和监控……更多的可能正在被学术和工业界探索。
>
> https://cloud.tencent.com/developer/article/1909298

## Conclusion

> XDP 论文看起来只是 CPU 消耗好很多，但他的意义在于，真的合入了内核，开源，而且属于是里程碑真的引起了后续的多种技术发展，AF_XDP 等等

<!--

Your response should first include a summary of the following,  for the XDP paper.

1. the problem the paper addresses
2. the paper’s solution
3. the evidence the paper provides that the solution was correct.


Second, the response should answer the following, for the XDP paper.
1. whether the problem the paper solved is an important one; are there any holes in the motivation?
2. particular strengths of the solution
3. particular weaknesses of the solution
4. whether the paper was fun or enjoyable to read
Finally, for this class's papers, please reflect on the following questions in your response:

1. Do you view XDP as a modern way to download code as introduced by the exokernel paper?  Why or why not?

2. Does XDP, in your opinion solve some of the security risks associated with the other high performance networking stack?


## XDP

1. The paper, XDP, tackles the issue of slow packet processing in the operating system kernel using network stack and security and configuration problem with user-level network stacks. While some methods like DPDK bypass the kernel to speed up packet processing, compromising security and making it harder to manage, XDP uses eBPF programs and virtual env for packet processing to bypass network stack in Linux kernel, reducing overhead of packet handling and ensuring safety.
2. The authors propose the eXpress Data Path (XDP), an innovative approach that allows fast, programmable packet processing directly within the Linux kernel. XDP processes packets at a very early stage, in the network device driver using XDP hooks, before the kernel allocates any resources for the packet. XDP also provides a safe environment for custom packet processing applications, It runs eBPF programs in a virtual machine (VM) within the kernel, allowing custom packet processing, while BPF verifier ensures these eBPF programs are safe to execute by checking for issues like out-of-bounds memory access.
3. The paper presents several evidence to support the effectiveness of XDP, performance evaluation comparing to DPDK, CPU usage, and packet forwarding performance. The authors also demonstrate the flexibility of XDP with three example applications: routing, DDoS protection, and load balancing. Additionally, XDP, and later AF_XDP, have been integrated into the Linux kernel, Facebook also has some applications based on XDP.

4. The problem solved by the paper is indeed important, especially thinking of slow packet processing in the operating system kernel. Besides DPDK the user-level network stack, I think XDP is more general and safe to use, it does not need specialized NIC or rewriting applications throughly since XDP is already in Linux kernel. The motivation for XDP is strong because bypassing the kernel will raise security and manageability issues. I don't see significant holes in the motivation, the only thing I can think of is that XDP depends on the kernel support and network driver capability.
5. One of the key strengths of XDP is its ability to achieve high packet processing performance with significantly lower CPU usage compared to DPDK, and XDP still provides the safety and configuration that kernel offers. XDP processes packets directly within the kernel and utilizes eBPF programs that are verified for safety and efficiency, bypassing the linux kernel network stack, which improves efficiency.
6. XDP operates within the constraints of the linux kernel and network device driver, and the performance of XDP can vary depending on the where the XDP offloads. If XDP can be offloaded onto network driver, it will have better performance than offloading XDP onto Linux kernel.
7. The paper is excellent because it not only provides open-source implementation code and many examples but also XDP and AF_XDP have been integrated into the Linux kernel. Additionally, it might inspire us to experiment with XDP for our own projects.

## Questions

1. I think XDP can be seen as a modern way to download code into the kernel, similar to the concept introduced by the exokernel paper. Especially for the BPF and eBPF that XDP is using, BPF allows custom programs to be loaded into the Linux kernel in the form of bytecode, filtering or handling network packets. Exokernel achieves secure bindings by downloading application code. But there are still some differences, the eBPF verifier and virtual environment ensure the custom code (eBPF programs) is safe to execute, while exokernel leaves some security and safety issues to the applications, providing secure binding and isolation in the kernel.
2. Yes, XDP addresses several security risks associated with other high-performance networking stacks. XDP operates within the Linux kernel, maintaining the security and isolation provided by the kernel. And XPD uses eBPF verifier and virtual environment to ensure that eBPF programs loaded into the kernel are safe to execute. For example, XDP is used in applications like DDoS protection, where it can filter out malicious traffic at high speeds, preventing it from overwhelming the system.
 -->
