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

作为嵌入数据库，DuckDB 没有客户端协议接口或者服务器进程，而是使用 C/C++ API 访问。此外，DuckDB 提供了一个 SQLite 兼容层，可以让使用 SQLite 的程序通过 re-linking 或者 library overloading 连接。

DuckDB SQL parser 衍生自 Postgres' SQL parser，并且尽可能地分离了。可以为 DuckDB 提供全功能、稳定的解析器，可以处理最不稳定的输入形式：SQL 查询。解析器将 SQL 查询字符串作为输入，并且返回一个解析树（C 语言结构）。解析树会立即被转成 DuckDB 自己的解析树（C++ 类）以限制 Postgres 数据结构的范围。

> 为什么要 limit the reach of Pg's data structure?

logical planner 逻辑规划器具有两个部分：binder 和 plan generator。binder 解析表达式和其引用的 schema object 比如表、视图的列名和类型。logical planner generator 将解析树转换成 tree of logical query operators 比如 scan, filter, project 等等。在 planning 阶段后，就有了 fully type-resolved logical query plan. DuckDB 保留了存储数据的统计信息，这些数据会在 planning 阶段通过不同的表达式树传播 propagate. 这些统计信息会在 optimizer 中使用，并且也可以用来在升级类型时做预防整型溢出。

DuckDB optimizer 使用 dynamic programming 动态规划实现 join order optimization，使用贪心作为 fallback 处理复杂的 join graphs.

It performs flattening of arbitrary subqueries as described in Neumann et al. [9].

> Thomas Neumann and Alfons Kemper. 2015. Unnesting Arbitrary Queries. In Datenbanksysteme für Business, Technologie und Web (BTW), 16. Fachtagung des GI-Fachbereichs "Datenbanken und Informationssys- teme" (DBIS), 4.-6.3.2015 in Hamburg, Germany. Proceedings.
>
> 没看懂这里

此外，duckdb 还有一系列重写的规则，可以简化表达式树，比如 common subexpression elimination, constant folding. 基数估计 cardinality estimation 使用了采样和 HyperLog 实现。此过程的结果是查询的优化逻辑计划 optimized plan for the query. The physical planner 物理规划器将逻辑规划转成物理规划，选择合适的实现方法。比如，scan 可能决定使用现有的索引而不是根据 selectivity estimates 扫描 base tables, 或者根据 join 谓词选择 hash join 还是 merge join。

DuckDB 使用了 vectorized interpreted execution engine. 使用了 SQL 查询 JIT 即时编译，有利于可移植性。JIT 引擎依赖于大量的编译库（比如 LLVM），DuckDB 使用了固定最大值的向量（默认值 1024）。固定长度类型的值，比如整数可以表示为原生数组 native arrays. 可变长度值比如字符串，可以表示为原生数组指针 into a separate string heap. NULL 值使用单独的位向量表示，当 NULL 值出现在向量时才存在。 这允许 fast intersection (逻辑乘) of NULL vectors for binary vector operation and avoids redundant computation....

> Peter A. Boncz, Marcin Zukowski, and Niels Nes. 2005. MonetDB/X100: Hyper-Pipelining Query Execution. In CIDR 2005, Second Biennial Conference on Innovative Data Systems Research, Asilomar, CA, USA, January 4-7, 2005. 225–237.
>
> 现代查询引擎的两种优化方式 - Vectorization and Compilation，有机会可以看看

执行引擎使用 火山模型 执行查询，查询执行通过物理计划的根节点拉取第一个 "chunk" 的数据。 chunk 是结果集合、查询中间结果或 base table 的一个水平子集。节点会递归地从子节点拉取 chunks，最终达到 scan operator，通过读持续化的表来产生 chunks. 过程持续到根节点为空，此时查询就完成了。

DuckDB 通过 MVCC 提供 ACID，实现了 HyPer's 可串行化 MVCC 的变种，为 OLAP/OLTP 系统量身定做。这个变种会原地立刻更新数据，并且会单独的 undo buffer 保存之前的状态，用来进行并发事务和 abort。DuckDB 选择了 MVCC 和简单的 schemes 比如 Optimistic Concurrency Control。尽管 DuckDB 用于 analytics，但并行修改 tables
