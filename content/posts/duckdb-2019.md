+++
title = 'Paper Reading: DuckDB: an Embeddable Analytical Database'
date = 2024-06-07T15:37:41+08:00
draft = false
tags = ['Paper Reading']
+++

## DuckDB: an Embeddable Analytical Database

读一下 DuckDB 的论文，对 OLAP 没什么了解，但论文很短也可以看看。

## ABSTRACT

SQLite 应用很广，unobtrusive in-process data management 是有必要的，但目前没有系统做 这样的 analytical workloads。

DuckDB 提出了 analytical SQL queries while embedded in another process.

DuckDB 开源，使用 CPP 实现：[DuckDB Github](https://github.com/duckdb/duckdb)

> 等有空可以学学 SQLite，是怎么做到体积小但是又应用这么广泛的。
>
> unobtrusive 表示什么？不引人注目？
>
> embbed database 和 database server 的区别在于，前者不属于服务器，体积小，和应用程序运行在同一进程。后者服务器独立运行。

## INTRODUCTION

数据处理现在有许多 large monolithic database servers running as stand-alone processes，是因为需要解决多客户端，高并发带来的数据完整性问题。

而嵌入式数据库完全不同，只需要嵌入到宿主进程。比如专注事务处理 OLTP 的 SQLite，使用 B-Tree 和行存作为存储引擎，所以在 OLAP 表现不好。

可嵌入的 OLAP 是有必要的：1. Interactive data analysis 2. edge computing

比如 R, Python 进行数据分析，缺少查询优化和事务处理。边缘计算用嵌入数据库可以做一些在线分析，论文举了个 power meters 带宽限制的例子。

论文分析了几个嵌入式 OLAP 数据库的需求：

1. 高效率 OLAP 工作负载，但不完全牺牲 OLTP 性能：比如 dashboard 中的数据并发修改，OLAP 进行可视化，OLTP 进行数据更新
2. 高效地从 tables 到 database 的传输：嵌入式数据库和应用在同一个进程和地址空间，可以高效地实现 data sharing
3. 高度稳定性：如果嵌入数据库崩溃，比如 OOM 可能会把宿主进程都带走，所以查询如果用完了资源，必须是可以 aborted cleanly 的，而且系统需要可以 gracefully 适应资源竞争。
4. 实用的可嵌入性和可移植性，数据库需要可以运行在不论主机运行在的任何环境。外部依赖是有问题的、信号处理也不行。

论文提出了 DuckDB，开源的，可嵌入的关系型数据库。具有完整的 SQL 接口。

## DESIGN AND IMPLEMENTATION

DuckDB 设计目的：嵌入式分析，进行了组件分离：parser, logical planner, optimizer, physical planner, execution engine. 以及 Orthogonal components: transaction, storage managers.
