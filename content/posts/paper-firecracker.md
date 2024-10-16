+++
title = 'Paper Reading: Firecracker '
date = 2024-10-16T09:52:03-04:00
draft = false
tags = ['Paper Reading']
+++

## Firecracker: Lightweight Virtualization for Serverless Applications

开源的 VMM

> 不太感兴趣，看一眼 presentation 和实现细节就过了
>
> 比 QEMU 轻量级，更注重几个专用场景
>
> 比如没有 BIOS 没有 PCI
>
> 和容器化结合比较好
>
> motivation 来自 EC2 太大，lambda 又太小，一些隔离
>
> 兼容性看着不错，和 linux 是兼容的
>
> soft allocation 是指超卖吗，需要的时候才实际分配
>
> 解决的问题：QEMU/KVM 难以支持运行多个虚拟机？LXC 隔离性较弱？LibOS 兼容性问题？语言 VM 比如 JVM 隔离兼容性问题？
>
> 性能比较了 QEMU 和 FC 的一些指标：boot time (serial && 50 parallel) 后者生产环境表现一般，
>
> QD1 IO latency vs bare metal (深度为一，类似串行)
>
> QD32 IO throughput vs bare metal （吞吐量表现很差，被 QEMU 完虐，可能觉得还需要修改 iouring 之类的）
>
> 队列深度下的 IO 延迟和吞吐量
>
> 主要还是几个问题：兼容性，性能兼容性，一些包管理 rpm 的不确定性，
>
> 可以做的还有 IO 提升、scheduling (tail latency)、正确性证明 (isolation)
>
> 一些 feature 比如 snapshot
>
> rust-vmm
>
> 其他一些分享：https://draveness.me/papers-firecracker/
>
> https://easoncao.com/firecracker-paper-reading/#%E7%B8%BD%E7%B5%90

## Choosing an Isolation Solution

![](https://s2.loli.net/2024/10/16/WQht9oJ7d8sqFfI.png)
