+++
title = 'Paper Reading: ServiceRouter: Hyperscale and Minimal Cost Service Mesh at Meta'
date = 2024-12-02T09:37:05-05:00
draft = false
tags = ['Paper Reading']
+++

## ServiceRouter: Hyperscale and Minimal Cost Service Mesh at Meta

Meta 的全球服务网格 ServiceRouter（SR）

> 服务网格（Service Mesh）是一种用于处理微服务架构中服务之间通信的软件层。它通过在服务之间插入代理，实现通信控制和监控，从而提高系统的可靠性、安全性和可观测性。
>
> 流量管理、负载均衡？

## Abstract

数据中心应用程序通常被构建为许多相互连接的微服务，而服务网格已成为在服务之间路由 RPC 流量的流行方法。

SR 与公开已知的服务网格在几个重要方面有所不同。首先，SR 是为超大规模设计的，目前使用数百万个 L7 路由器在数万个服务之间每秒路由数千亿个请求

其次，虽然 SR 采用了使用 sidecar 或远程代理来路由我们 fleet 中 1%的 RPC 请求的常见方法，但它采用了一个路由库，该库直接链接到服务可执行文件中，以将剩余的 99% 直接从客户端路由到服务器，**而无需通过代理的额外跳转**。这种方法显著降低了我们超大规模服务网格的硬件成本，节省了数十万台机器。第三，SR 内置支持**分片服务**，这些服务占我们舰队中 68%的 RPC 请求，而现有的通用服务网格不支持分片 sharded 服务。最后，SR 引入了**局部性环**的概念，以同时最小化 RPC 延迟并在地理分布的数据中心区域之间平衡负载，据我们所知，这是以前未曾尝试过的。

## Introduction

微服务架构

为了管理这些服务之间的远程过程调用（RPC）流量，许多组织使用服务网格 service mesh

![Figure 1](https://s2.loli.net/2024/12/02/m5ZBfpQKPrzRgVI.png)

图 1 展示了最常见的第 7 层（L7，即应用层）服务网格形式。在这种架构中，每个服务进程都伴随着一个运行在同一台机器上的 **L7 边车代理**，该代理代表服务路由 RPC 请求。

例如，当机器 1 上的服务 A 向服务 B 发送请求时，机器 1 上的代理将请求在机器 2 和机器 3 之间进行负载均衡。如果自动扩展系统检测到负载增加并开始在机器 4 上启动服务 B 的新副本，控制平面的服务发现功能将通知机器 1 上的代理，该代理随后将机器 4 包含在其未来对服务 B 请求的负载均衡目标中。

本文介绍了 Meta 的全球服务网格，称为 ServiceRouter（SR）。SR 支持一系列全面的功能，包括服务发现、负载均衡、故障转移、认证、加密、可观测性、过载保护、分布式请求跟踪、容量管理的资源归属以及用于影子测试的流量复制。

Scalability: 传统上，软件定义网络使用集中式控制平面和分散式数据平面。大多数服务网格遵循这种方法，并使用**中央控制器**来配置每个边车代理的路由表

![](https://s2.loli.net/2024/12/02/sfEzcC3Tl4VOKAF.png)

图 2 展示了 SR 的可扩展架构

在顶部，不同的控制器独立执行诸如注册服务和生成每个服务的跨区域路由表等功能。每个控制器独立更新中央路由信息库（RIB），而不关心配置或管理单个 L7 路由器

> Routing Information Base (RIB) 负责服务发现、路由配置、跨区路由配置等等

Hardware cost. SR 通过提供一个名为 SRLib 的库来消除对代理及其相关硬件成本的需求。
为了满足服务的多样化需求，SR 实现了不同类型 L7 路由器的无缝共存，包括 Istio 风格的边车代理、AWS-ELB 风格的专用负载均衡器和 gRPC 风格的旁路负载均衡器

Sharded services SR 将分片支持作为首要任务，并使用单一框架来支持分片和复制。

Cross-region routing 现有的解决方案未针对跨地理分布的数据中心区域的路由进行优化。
为了更好地支持跨区域路由，我们引入了局部性环的概念，让服务表达它们在延迟和负载之间的首选权衡。

Contributions

- SR 是为超大规模设计的。
- SR 支持在一个服务网格中不同类型 L7 路由器的无缝共存，包括边车代理、专用负载均衡器、旁路负载均衡器和嵌入式路由库。
- 尽管现有服务网格仅关注未分片的服务，这些服务仅占我们舰队 RPC 请求的 32%，SR 提供了对分片服务的内置支持，这些服务占我们流量的 68%。
- 局部性环

## Comparison of Services Mesh Architectures

(1) which component fetches and caches the routing metadata, and (2) which component routes application RPC traffic.

### Different Types of L7 Routers in SR

SRLib

SRLookaside

SRSidecarProxy

SRRemoteProxy

![](https://s2.loli.net/2024/12/02/b8AfeCQ9nztm7MU.png)

> 分别是路由库，旁路负载均衡器， sidecar，远程代理
>
> SRLib 直接发送请求到 Server
>
> SRLookaside 负载均衡
>
> SRSidecarProxy 边车
>
> SRRemoteProxy 跨区域？

### Comparison of L7 Routers

> 对比了不同的 miniRIB 路由库？

接下来，我们将比较表 1 中的解决方案。解决方案(1)–(4)是不理想的，因为管理嵌入式库中的 miniRIB 会影响应用程序的性能，由于缺乏隔离。
解决方案(5)、(7)和(8)是不理想的，因为没有系统调用来访问内核中缓存的 miniRIB。例如，Cilium 的 eBPF 程序只能处理 L3/L4 协议，仍然需要使用边车代理来处理 L7 协议。与解决方案(6)类似，解决方案(10)和(14)是不理想的，因为难以在内核中实现高级 L7 路由功能。解决方案(12)是不理想的，因为它严格比(16)差，即如果路由由远程代理执行，最好也将 miniRIB 移到远程代理。理论上，解决方案(15)在客户端机器上使用的内存比(11)少。然而，(15)在 Meta 中没有使用，因为即使(11)也没有广泛使用，(15)的额外好处有限。最后，为了便于访问，我们在表 2 中总结了设计替代方案的比较。

## ServiceRouter Design

### Overview

SR 支持图 4 中描述的所有四种 L7 路由器类型。对于边车或远程代理设置，我们在 SRLib 代码之上添加了一个包装层，以使其作为独立代理运行。SR 的控制平面组件如图 6 所示，并进一步解释如下。

Routing Information Base （RIB）。RIB 是一个基于 Paxos 的键值存储

Global Registry Service（GRS）在 RIB 中维护服务和分片发现信息。图 7 显示了在 GRS 注册的两个示例服务。服务 A 是复制的但未分片。当集群管理器启动或停止服务 A 的容器时，它会通知 GRS 更新服务 A 的副本列表。我们将在 §3.3 中解释 SR 对分片服务的内置支持。

Configuration Management System (CMS) 允许为每个服务定制路由策略，包括 RPC 超时、连接重用、局部性路由偏好等。服务所有者遵循配置即代码范式来编写、审查和提交路由配置。它还支持自动配置更新。例如，延迟监控服务（LMS）定期聚合并提交与跨区域延迟相关的配置更新，以指导 SRLib 的路由决策。

Cross-region Routing Service (xRS). 与集中式负载均衡器相比，SRLib 只有一个客户端的本地视图，可能无法做出全局最优的路由决策。xRS 通过聚合每个服务的全局流量信息并计算跨区域路由表来解决这个问题，该路由表通过 RIB 分发并由 SRLib 使用以指导其路由决策

### Service Discovery

每个机器上运行一个 RIBDaemon，维护一个所谓的 miniRIB，缓存 RIB 中 RPC 客户端运行所需的具体部分。最初，miniRIB 是空的。当 SRLib 想要向特定服务（如服务 X）发送 RPC 请求时，它会从 RIBDaemon 请求服务 X 的路由元数据。RIBDaemon 从 RIB 副本中获取元数据，将其缓存到磁盘上以便在机器重启后仍然可用，订阅与服务 X 相关的未来更新，最后将元数据返回给 SRLib。SRLib 也订阅 RIBDaemon 的未来更新，并将元数据缓存在内存中（但不缓存到磁盘）以供后续重用，这样它就不会在每次 RPC 请求时都联系 RIBDaemon。

当服务 X 的部署在未来发生变化时，集群管理器通知 GRS 更新 RIB。更新会立即推送到所有 RIB 副本，这些副本进一步将更新推送到订阅服务 X 路由元数据的所有 RIBDaemon。最后，RIBDaemon 将更新推送到 SRLib。服务 X 可能部署在多个数据中心区域，每个区域的副本由不同的区域集群管理器管理。所有这些集群管理器都会通知 GRS 更新服务 X 的相同服务注册记录，以便客户端的 RPC 请求可以潜在地路由到任何区域的副本（§3.4.1）。服务的服务客户端可以选择仅向位于与客户端相同区域的服务器发送请求。在这种情况下，为了减少开销，RIBDaemon 仅订阅来自本地区域的更新。

在集群管理器的帮助下，客户端不需要通过超时独立发现服务器的故障。当服务器因计划维护（如代码部署）而关闭时，集群管理器首先更新 RIB 以通知客户端，然后停止服务器。对于计划外故障，集群管理器检测各种故障，如进程崩溃/挂起和机器故障，并更新 RIB 以通知客户端。

### Support for Sharded Services

SR 的分片映射抽象是通用的，目前支持数百个分片服务。其中大多数（但不是全部）由一个通用的分片管理器管理，当添加或删除新分片或现有分片在容器之间迁移时，分片管理器会通知 GRS 更新分片注册表。

通过 SR，分片和未分片服务共享并重用 SR 中的所有复杂组件（图 2 和图 6）。此外，分片服务的路由开箱即用，无需任何额外努力。相比之下，现有的通用服务网格不支持分片服务，应用程序必须开发自己的解决方案。

Design alternatives: 一致性哈希，
但一致性哈希对于高级分片用例是不够的，因为其确定性的键分配不支持响应分片负载变化的动态分片迁移。SR 提供了对一致性哈希和分片映射方法的内置支持。如我们之前的工作所示，在 Meta 的数百个分片服务中，选择使用灵活分片映射的服务数量是选择使用一致性哈希的服务数量的 5.4 倍，这证实了分片映射方法的重要性和有效性。

SR 的分片映射方法的另一个替代方案是允许服务提供自己的自定义旁路服务实现。

### Load Balancing

Pick-2 算法

> Michael Mitzenmacher. The power of two choices in randomized load balancing. IEEE Transactions on Parallel and Distributed Systems, 12(10):1094–1104, 2001.

开发了三种新技术来补充 Pick-2：1) 在抽取两个随机服务器时考虑区域局部性（§3.4.1）。2) 从服务器的稳定子集中抽取两个随机服务器，而不是所有服务器，以最大化连接重用（§3.4.2）。3) 根据工作负载特征采用自适应负载估计方法（§3.4.3）。以下部分提供了这些技术的更多细节。

Locality Awareness

SR 不是从候选池中随机抽取两个服务器，而是使用所谓的局部性环来过滤掉远离客户端的长延迟服务器，然后从剩余的附近服务器中抽取。每个服务可以定义一组具有递增延迟的环，例如 [ring1: 5 毫秒 | ring2: 35 毫秒 | ring3: 80 毫秒 | ring4: 无限制]。延迟监控服务（LMS）定期更新区域之间的 RTT，RPC 客户端通过 CMS 获取这些信息。

通过局部性环过滤可以减少路由延迟，但由于缺乏全局视图，仍然存在局限性。

Design alternative: xRS, 然而，SR 没有采用这种方法，因为根据排队理论和我们的生产经验，在高利用率下建模延迟并不稳健。
总的来说，在网络 RTT 可能从 100 毫秒到 100 毫秒变化三个数量级的地理分布环境中，以延迟为中心的方法并不稳健。

### RPC Connection Reuse

建立一个新的 TLS/TCP 连接需要 1.6 毫秒，并且在每侧消耗 14KB 的内存。为了减少这种开销，SR 保持 TLS/TCP 连接并在不同的 RPC 请求之间重用它们。然而，Pick-2 使用的随机化使得连接重用无效。

为了提高连接重用，

在 SR 中，每个 RPC 客户端使用 Rendezvous Hashing 来选择 ā 个稳定服务器，这实现了上述理想的特性

Design alternative 我们更喜欢 Rendezvous Hashing 而不是一致性哈希，因为它允许 SR 使用加权哈希按服务器的计算能力成比例地分配客户端连接。

> 更加负载均衡，Rendezvous Hashing 比起一致性哈希

### Adaptive Load Estimation

为了使 Pick-2 在两个候选服务器之间选择路由目标，它需要知道负载信息。默认情况下，SR 使用 RPC 服务器上的未完成请求数量来表示其负载

为了确定服务器的负载，客户端有两个选项：

1. 在决定是否向服务器发送请求之前，轮询服务器以获取其负载，这会带来额外的开销和延迟。
2. 让服务器在其响应中包含其负载信息，然后在客户端缓存以供后续重用，这很高效，但可能导致客户端使用过时的负载信息并导致负载不平衡。

Design alternative. LI [13] 试图通过仅使用方法 1 和方法 3 来解决负载估计问题，而不使用方法 2（轮询）。我们生产系统中的数据显示，使用 SR 的自适应机制，大约 50%、25% 和 25% 的 RPC 请求分别使用方法 1、方法 2 和方法 3。这证实了引入 just-in-time polling 方法的有用性。

## Evaluation

1. Does SR scale well? (§ 4.1)
2. To what extent does SRLib save hardware costs, and when
   should one use **SRProxy** versus **SRLib**? (§ 4.2)
3. Can SR balance load within and across regions? (§ 4.3)
4. Are sharded services important, and can SR effectively
   support both **sharded and unsharded services**? (§ 4.4)

### Scalability

超大规模是区分 SR 与大多数现有服务网格的关键设计目标。

过去，我们使用 ZooKeeper 作为 RIB 的数据存储，它无法扩展到几 GB 以上，因此我们分片了 RIB。现在我们使用内部数据存储，它扩展性良好，没有进一步分片 RIB 的紧迫性。总体而言，目前 RIB 不是瓶颈，如果需要，可以进一步分片以扩展。

### Hardware Cost

SRLib 和 SRProxy 的 CPU 开销

### SRLib versus SRProxy

为了量化硬件成本，我们进行了一项实验，比较了三种 RPC 设置：1) SRLib，客户端使用 SRLib 将请求路由到一个在 10 台机器上运行的简单服务；2) SRProxy，客户端将请求发送到远程 SRProxy，后者将请求转发到服务器；3) Thrift，裸客户端硬编码一个最有效的方式随机选择 10 台服务器之一，并使用 Thrift RPC 协议调用它。SRLib 和 SRProxy 的内部实现也使用 Thrift，但在其上添加了额外的逻辑。因此，Thrift 代表了下限基线。

#### Case Study of When to Use SRProxy

E-Comm.

Key-value store

### Load Balancing

SR 在区域内和跨区域执行负载均衡。

### Sharded Services

目前，我们的舰队运行数百个分片服务。尽管它们仅占我们数万个服务的约 3%，但它们产生的流量比其他 97% 的未分片服务更多，因为许多分片服务是我们最大和最高流量的服务之一。

我们的 memcache 是分片的，

## Limitations of SRLib and Our Solutions

Dynamic policy updates:

Source code modification: SRLib 的一个缺点是它需要对服务进行代码更改。传统的 RPC 使用 IP 地址和端口号来获取 RPC 客户端，而 SRLib 使用服务名称获取 RPC 客户端

Library code deployment 部署新版本的 SRLib 比部署新版本的边车代理更困难。这是因为 SRLib 被编译到数万个服务中，每个服务都有自己的部署计划。

Bugs in SRLib: 如果 SRLib 的新代码有错误，可能很难立即回滚所有服务
