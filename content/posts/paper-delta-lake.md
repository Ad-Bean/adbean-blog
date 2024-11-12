+++
title = 'Paper Reading: Delta Lake High-Performance ACID Table Storage over Cloud Object Stores'
date = 2024-11-11T11:24:58-05:00
draft = false
tags = ['Paper Reading']
+++

## Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores

https://github.com/delta-io/delta

data lake? delta lake? data warehouse?

> databricks 的论文
>
> DeltaLake 类似的产品有 Hudi, Iceberg, Apache Paimon
>
> 其他论文笔记 https://www.cnblogs.com/Aitozi/p/17552466.html
>
> 大数据技术换代也太快了，但底层原理我还是一问三不知，还得练

> Towards Multi-Table Transactions in Delta Lake
>
> Delta 4.0 尝试解决多表事务的问题，感觉有点像 2PC
>
> https://www.databricks.com/dataaisummit/session/towards-multi-table-transactions-delta-lake

## Abstract

云对象存储（如 Amazon S3）是地球上最大且最具成本效益的存储系统之一

然而，它们作为键值存储的实现方式使得实现 **ACID 事务**和高性能变得困难：元数据操作（如列出对象）成本高昂，一致性保证有限。

在本文中，我们介绍了 Delta Lake，这是一个开源的 ACID 表存储层，最初由 Databricks 开发，用于云对象存储。Delta Lake 使用一个事务日志，该日志被压缩为 Apache Parquet 格式，以提供 ACID 属性、时间旅行，以及对大型表格数据集（例如，能够快速搜索与查询相关的数十亿个表分区）的元数据操作的显著加速。

它还利用这一设计提供高级功能，如自动数据布局优化、更新插入、缓存和审计日志。Delta Lake 表可以从 Apache Spark、Hive、Presto、Redshift 和其他系统访问。

> data lake 提出比较早，而 delta lake 支持 ACID

## INTRODUCTION

因此，许多组织现在使用 use cloud object stores to manage large structured datasets in data warehouses and data lakes。主要的开源“大数据”系统，包括 Apache Spark、Hive 和 Presto，支持使用 Apache Parquet 和 ORC [13, 12]等文件格式读写云对象存储。商业服务，包括 AWS Athena、Google BigQuery 和 Redshift Spectrum，也可以直接查询这些系统和这些开放文件格式。

不幸的是，尽管许多系统支持读写云对象存储，但在这些系统上实现**高性能和可变的表存储**是具有挑战性的，这使得在这些系统上实现数据仓库功能变得困难。与 HDFS [5]等分布式文件系统或 DBMS 中的自定义存储引擎不同，大多数云对象存储仅仅是**键值存储**，没有跨键一致性保证。它们的性能特性也与分布式文件系统大不相同，需要特别注意。

> 云对象存储和分布式文件系统有什么区别？
>
> 云对象存储底层可以是 kv 也可以是别的，S3 应该是没公开底层是什么
>
> etcd, dynamoDB, redis 就典型的分布式键值存储，一般通过上层提供 ACID
>
> hdfs, ceph 是典型的分布式文件系统，

在云对象存储中存储关系数据集的最常见方式是使用 Parquet 和 ORC 等列式文件格式，，其中每个**表存储为一组对象**（Parquet 或 ORC“文件”），可能按某些字段（例如，每个日期的一组单独对象）聚类为“分区”。**只要对象文件适度大**，这种方法可以为扫描工作负载提供可接受的性能。然而，它为更复杂的工作负载带来了正确性和性能挑战。首先，由于多对象更新不是原子的，**查询之间没有隔离**：例如，如果一个查询需要更新表中的多个对象（例如，删除所有 Parquet 文件中关于一个用户的记录），读者将在查询逐个更新每个对象时看到部分更新。回滚写操作也很困难：如果更新查询崩溃，表将处于损坏状态。其次，对于包含数百万个对象的大型表，元数据操作成本高昂。例如，Parquet 文件包含带有最小/最大统计信息的页脚，可用于在选择性查询中跳过读取它们。在 HDFS 上读取这样的页脚可能只需几毫秒，但云对象存储的延迟要高得多，以至于这些数据跳过检查可能比实际查询花费更长时间。

> Parquet 列存

根据我们与云客户合作的经验，这些一致性和性能问题为企业数据团队带来了重大挑战。大多数企业数据集是持续更新的，因此它们需要原子写入的解决方案；大多数关于用户的数据集需要表范围的更新来实施隐私政策，如 GDPR 合规 [27]；即使是纯粹的内部数据集也可能需要更新来修复错误数据、合并延迟记录等。据传闻，在 Databricks 云服务（2014-2016 年）的最初几年，我们收到的约一半支持升级是由于云存储策略导致的数据损坏、一致性或性能问题（例如，撤销崩溃的更新作业的影响，或提高读取数万个对象的查询的性能）。

为了解决这些挑战，我们设计了 Delta Lake，这是一个在云对象存储上的 ACID 表存储层

Delta Lake 的核心思想很简单：**我们使用一个预写日志（write-ahead log）以 ACID 方式维护关于哪些对象是 Delta 表一部分的信息，**该日志本身存储在云对象存储中。对象本身以 **Parquet** 编码，使得从已经能够处理 Parquet 的引擎编写连接器变得容易。这种设计允许客户端以序列化方式一次性更新多个对象，用另一组对象替换子集等，同时仍然从对象本身实现高并行读写性能（类似于原始 Parquet）。日志还包含每个数据文件的最小/最大统计信息等元数据，使得元数据搜索比“对象存储中的文件”方法快一个数量级。**至关重要的是，我们将 Delta Lake 设计为所有元数据都在底层对象存储中，并使用针对对象存储的乐观并发协议实现事务（细节因云提供商而异）**。这意味着不需要运行服务器来维护 Delta 表的状态；用户仅在运行查询时需要启动服务器，并享受独立扩展计算和存储的好处

基于这种事务性设计，我们还在 Delta Lake 中添加了多个其他功能，这些功能在传统云数据湖中不可用，以解决常见的客户痛点，包括：

- Time travel to let users query point-in-time **snapshots** or roll back erroneous updates to their data.
- UPSERT, DELETE and MERGE operations, which efficiently rewrite the relevant objects to implement updates to
  archived data and compliance workflows (e.g., for GDPR [27]).
- Efficient **streaming I/O**, by letting streaming jobs write small
  objects into the table at low latency, then transactionally coalescing them into larger objects later for performance. Fast
  “tailing” reads of the new data added to a table are also supported, so that jobs can treat a Delta table as a message bus.
- Caching: Because the objects in a Delta table and its log
  are immutable, cluster nodes can safely cache them on local
  storage. We leverage this in the Databricks cloud service to
  implement a transparent SSD cache for Delta tables.
- Data layout optimization: Our cloud service includes a feature that automatically optimizes the size of objects in a table
  and the clustering of data records (e.g., storing records in Zorder to achieve locality along multiple dimensions) without
  impacting running queries.
- Schema evolution, allowing Delta to continue reading old
  Parquet files without rewriting them if a table’s schema changes.
- Audit logging based on the transaction log

这些功能共同提高了在云对象存储中处理数据的易管理性和性能，并实现了“湖仓”范式，结合了数据仓库和数据湖的关键特性：标准 DBMS 管理功能可直接用于低成本对象存储。

事实上，我们发现许多 Databricks 客户可以通过 Delta Lake 简化其整体数据架构，用提供适当功能的 Delta 表替换之前独立的数据湖、数据仓库和流存储系统。

图 1 展示了一个极端例子，一个包括对象存储、消息队列和两个数据仓库的数据管道（每个业务智能团队运行自己的计算资源）被替换为仅在对象存储上的 Delta 表，使用 Delta 的流式 I/O 和性能特性运行 ETL 和 BI。新管道仅使用低成本对象存储，并创建较少的数据副本，从而降低了存储成本和维护开销。

![](https://s2.loli.net/2024/11/12/WycDzeGK2PldUbF.png)

开源 Delta Lake 项目 [26] 包括与 Apache Spark（批处理或流式）、Hive、Presto、AWS Athena、Redshift 和 Snowflake 的连接器，**并且可以在多个云对象存储上或 HDFS 上运行**。在本文的其余部分，我们将介绍 Delta Lake 的动机和设计，以及推动我们设计的客户用例和性能实验。

## MOTIVATION: CHARACTERISTICS AND CHALLENGES OF OBJECT STORES

### Object Store APIs

云对象存储，如 Amazon S3 [4]、Azure Blob Storage [17]、Google Cloud Storage [30] 和 OpenStack Swift [38]，提供了一个简单但易于扩展的键值存储接口。这些系统允许用户创建存储多个对象的桶，每个对象是一个大小可达几 TB 的二进制大对象（例如，在 S3 上，对象大小的限制是 5TB [4]）

不幸的是，这些元数据 API 通常是昂贵的：例如，S3 的 LIST 每次调用最多返回 1000 个键，每次调用需要几十到几百毫秒的时间，因此使用顺序实现列出包含数百万个对象的数据集可能需要几分钟的时间。

在读取对象时，云对象存储通常支持字节范围请求，因此可以高效地读取大对象中的某个范围

一些云供应商还在 blob 存储之上实现了分布式文件系统接口，例如 Azure 的 ADLS Gen2 [18]，它提供了与 Hadoop 的 HDFS 类似的语义（例如，目录和原子重命名）。尽管如此，Delta Lake 解决的许多问题，如小文件 [36] 和跨多个目录的原子更新，即使在分布式文件系统上仍然存在——实际上，多个用户在 HDFS 上运行 Delta Lake。

### Consistency Properties

最流行的云对象存储为每个键提供最终一致性

> eventual consistency 更新不是立刻看到

具体的一致性模型因云提供商而异，可能相当复杂。作为一个具体的例子，Amazon S3 为写入新对象的客户端提供了 read-after-write consistency，这意味着像 S3 的 GET 这样的读取操作将在 PUT 之后返回对象内容。

> negative caching 是什么

### Performance Characteristics

在使用对象存储时实现**高吞吐量**需要仔细平衡大块**顺序 I/O** 和并行性

对于**读取操作**，最细粒度的操作是读取顺序字节范围，如前所述。每个读取操作通常会产生至少 5-10 毫秒的基础延迟，并且随后可以以大约 50-100 MB/s 的速度读取数据，因此一个操作需要读取至少几百 KB 的数据才能达到顺序读取峰值吞吐量的一半，并且需要读取几 MB 的数据才能接近峰值吞吐量。此外，在典型的虚拟机配置中，应用程序需要并行运行多个读取操作以最大化吞吐量。例如，AWS 上最常用于分析的虚拟机类型至少具有 10 Gbps 的网络带宽，因此它们需要并行运行 8-10 个读取操作以充分利用这一带宽。

LIST 操作也需要显著的并行性来快速列出大量对象。在 Delta Lake 中，可用对象的元数据（包括它们的名称和数据统计信息）存储在 Delta 日志中，但我们也在集群上并行读取此日志

写操作通常必须替换整个对象（或追加到对象中），如第 2.1 节所述。这意味着如果一个表预计会接收点更新，那么其中的对象应该保持较小，这与支持**大读取操作**相矛盾。或者，可以使用 log-structured storage format.

Implications for Table Storage

1. Keep **frequently accessed data** close-by **sequentially**, which generally leads to choosing columnar formats.
2. Make objects **large**, but not too large. Large objects increase
   the cost of **updating data** (e.g., deleting all data about one
   user) because they must be fully rewritten.
3. Avoid LIST operations, and make these operations request
   lexicographic key ranges when possible.

### Existing Approaches for Table Storage

1. Directories of Files 这种方法很有吸引力，因为表格“只是一堆对象”，可以从许多工具中访问，而无需运行任何额外的数据存储或系统。它起源于 HDFS 上的 Apache Hive [45]，并与在文件系统上使用 Parquet、Hive 和其他大数据软件相匹配。

这种方法的挑战。如引言中所述，“只是一堆文件”的方法在云对象存储上存在性能和**一致性**问题。客户遇到的最常见的挑战包括：

- 跨多个对象的原子性缺失：任何需要写入或更新多个对象的事务都可能面临部分写入对其他客户端可见的风险。此外，如果此类事务失败，数据将处于损坏状态。
- 最终一致性：即使在成功的事务中，客户端也可能看到一些更新的对象，而看不到其他对象。
- 性能不佳：列出对象以找到与查询相关的对象是昂贵的，即使它们按键分区到目录中。此外，访问存储在 Parquet 或 ORC 文件中的每个对象的统计信息也很昂贵，因为它需要为每个特征进行额外的高延迟读取。
- 缺乏管理功能：对象存储不实现数据仓库中常见的标准实用程序，如表格版本控制或审计日志。

2. Custom Storage Engines 例如 Snowflake 数据仓库，可以通过在单独的强一致性服务中自行管理元数据来绕过云对象存储的许多一致性挑战，该服务持有关于哪些对象构成表格的 "source of truth"

在这些引擎中，云对象存储可以被视为一个 dumb block device，并且可以使用标准技术在云对象上实现高效的元数据存储、搜索、更新等。然而，这种方法需要**运行一个高度可用的服务来管理元数据**，这可能很昂贵，在使用外部计算引擎查询数据时会增加开销，并且可能将用户锁定在一个提供商上。

这种方法的挑战：

- 所有对表格的 I/O 操作都需要联系 metadata 服务，这会增加其资源成本并降低性能和可用性。例如，当使用 Spark 访问 Snowflake 数据集时，从 Snowflake 的 Spark 连接器读取数据会通过 Snowflake 的服务流式传输数据，与直接从云对象存储读取相比，性能有所降低。
- 连接到现有计算引擎需要更多的工程工作来实现，而不是重用现有的开放格式（如 Parquet）。根据我们的经验，数据团队希望在其数据上使用广泛的计算引擎（例如 Spark、TensorFlow、PyTorch 等），因此使连接器易于实现非常重要。
- 专有的元数据服务将用户绑定到特定的服务提供商，而基于直接访问云存储中的对象的方法使用户能够始终使用不同的技术访问其数据。

> 所以现在都用 open formats? iceberg 是新的技术趋势吗

Apache Hive ACID [32] 通过使用 Hive Metastore（一个事务性 RDBMS，如 MySQL）来跟踪存储在 ORC 格式中的表格的多个更新文件，在 HDFS 或对象存储上实现了类似的方法。然而，这种方法受到元数据存储性能的限制，根据我们的经验，对于包含数百万对象的表格，元数据存储可能成为瓶颈。

> metastore metadata 这里不太理解，这些数据的 metadata 为什么要分开存呢

3. Metadata in Object Stores. Delta Lake 的方法是将**事务日志**和**元数据**直接存储在云对象存储中，并使用一组**对象存储操作协议**来实现序列化。
   表格中的数据随后以 Parquet 格式存储，使得只要有一个最小的连接器来发现要读取的对象集，就可以从任何已经支持 Parquet 的软件中轻松访问。尽管我们认为 Delta Lake 是第一个使用这种设计的系统（始于 2016 年），但另外两个软件包现在也支持它——Apache Hudi [8] 和 Apache Iceberg [10]。Delta Lake 提供了这些系统不支持的许多独特功能，例如 Z-order 聚类、缓存和后台优化。我们将在第 8 节中更详细地讨论这些系统之间的相似性和差异。

> Iceberg, Hudi 等技术目前工业界应该是很有前景的
>
> Deltalake 的特色是什么？

## DELTA LAKE STORAGE FORMAT AND ACCESS PROTOCOLS

Delta Lake 表格是云对象存储或文件系统上的一个目录，其中包含表格内容的文件对象和事务操作的日志 (with occasional checkpoints)

客户端使用我们为云对象存储特性定制的乐观并发控制协议来更新这些数据结构。在本节中，我们将描述 Delta Lake 的存储格式和这些访问协议。我们还将描述 Delta Lake 的事务隔离级别，包括表格内的序列化和快照隔离。

### Storage Format

![Figure 2](https://s2.loli.net/2024/11/12/wbJ5xnQMKFHyrPX.png)

图 2 展示了 Delta 表格的存储格式。每个表格都存储在**文件系统目录**（此处为 mytable）中，或者作为对象存储中以相同“目录”键前缀开头的对象。

#### Data Objects

表格内容存储在 Apache Parquet 对象中

选择 Parquet 作为底层数据格式，因为它面向列，提供多种压缩更新，支持半结构化数据的嵌套数据类型，并且在许多引擎中已经有高性能的实现。基于现有的开放文件格式也确保了 Delta Lake 可以继续利用 Parquet 库的新发布更新，并简化了开发与其他引擎的连接器（第 4.8 节）。其他开源格式，如 ORC [12]，可能也能以类似的方式工作，但 Parquet 在 Spark 中拥有最成熟的支持。

Delta 中的每个数据对象都有一个唯一的名称，通常由写入者通过生成 GUID 来选择。然而，哪些对象是表格每个版本的组成部分由事务日志决定。

#### Log

日志存储在表格内的\_delta_log 子目录中。它包含一系列带有递增的零填充数字 ID 的 JSON 对象，用于存储日志记录，以及偶尔的检查点，用于特定日志对象，以 Parquet 格式总结到该点的日志。正如我们在第 3.2 节中讨论的那样，一些简单的访问协议（取决于每个对象存储中可用的原子操作）用于创建新的日志条目或检查点，并使客户端就事务顺序达成一致。

每个日志记录对象（例如，000003.json）包含一个操作数组，这些操作应用于表格的前一个版本以生成下一个版本。可用的操作包括：

Change Metadata.

Add or Remove Files.

Protocol Evolution.

Add Provenance Information.

Update Application Transaction IDs.

#### Log Checkpoints

为了性能，有必要定期将日志压缩成检查点。检查点以 Parquet 格式存储表格日志中到某个日志记录 ID 的所有非冗余操作。

### Access Protocols

Delta Lake 的访问协议旨在让客户端仅使用对象存储上的操作实现**序列化事务**，尽管对象存储具有最终一致性保证。

#### Reading from Tables

只读事务

1. 读取表格日志目录中的 `_last_checkpoint` 对象（如果存在），以获取最近的检查点 ID。
2. 使用 LIST 操作，其起始键是最后一个检查点 ID（如果存在），否则为 0，以查找表格日志目录中任何更新的 `.json` 和 `.parquet` 文件。
3. 使用上一步中标识的检查点（如果存在）和后续日志记录重建表格的状态——即，具有添加记录但没有相应删除记录的数据对象集及其关联的数据统计信息。
4. 使用统计信息识别与读取查询相关的数据对象文件集。
5. 查询对象存储以读取相关的数据对象，可能在集群中并行进行

我们注意到，此协议设计为在每一步都 tolerate eventual consistency

> 第一步读到旧的，第二步可能拿到新的？

#### Write Transactions

写入数据的事务通常会根据事务中的操作分为最多五个步骤：

> 省略细节了，这篇论文图太少了，很不清晰

请注意，第五步，即写入检查点然后更新 `_last_checkpoint` 对象，仅影响性能，客户端在此步骤中的任何地方失败**都不会损坏数据**。例如，如果客户端未能写入检查点，或者写入了检查点 Parquet 对象但没有更新 `_last_checkpoint` ，那么其他客户端仍然可以使用较早的检查点读取表格。如果步骤 4 成功，事务将原子提交。

Adding Log Records Atomically 步骤 4，即创建 r + 1 的.json 日志记录对象，需要是原子的：只有一个客户端应该成功创建具有该名称的对象。不幸的是，并非所有大规模存储系统都具有原子的“不存在则放置”操作，但我们能够为不同的存储系统以不同的方式实现此步骤：

> 不同 cloud provider 不同实现 AWS S3, Azure Blob, google cloud storage

### Available Isolation Levels

Given Delta Lake’s concurrency control protocols, all transactions
that perform writes are **serializable**, leading to a **serial** schedule in
increasing order of log record IDs 这是由于写入事务的提交协议，其中只有一个事务可以写入每个记录 ID 的记录。

读取事务可以实现**快照隔离**或序列化。

> snapshot isolation 快照隔离
>
> 写的事务隔离级别很高，是序列化

### Transaction Rates

Delta Lake 的写入事务速率受限于写入新日志记录的“不存在则放置”操作的延迟，如第 3.2.2 节所述。与任何乐观并发控制协议一样，**高写入事务速率将导致提交失败**。

我们相信一个自定义的 LogStore，类似于我们的 S3 提交服务，可以通过协调对日志的访问来提供显著更快的提交时间（例如，通过在低延迟 DBMS 中持久化日志的末尾，并异步将其写入对象存储）。当然，快照隔离级别的读取事务不会产生争用，因为它们只读取对象存储中的对象，因此可以同时运行任意数量的这些事务。

> 概念太多了这里，
>
> 首先，在提交时，如果多个事务尝试更新同一资源，就会发生冲突。由于每个事务都试图写入相同的日志记录位置，这些冲突会导致某些事务提交失败，需要回滚和重试。
>
> LogSotre 用于管理事务日志记录。它负责日志的持久化和存取，确保写入操作的原子性和一致性。
> 如果需要更高的写入速率，可以通过定制的 LogStore 进行优化，如在低延迟数据库中持久化日志结尾，**并异步写入对象存储**。

## HIGHER-LEVEL FEATURES IN DELTA

### Time Travel and Rollbacks

由于 Delta Lake 的数据对象和日志是不可变的，Delta Lake 使得查询数据的过去快照变得简单，就像典型的 MVCC 实现一样。

### Efficient UPSERT, DELETE andMERGE

在传统的数据湖存储格式中，例如 S3 上的 Parquet 文件目录，很难在不停止并发读取器的情况下执行这些更新。即使如此，更新作业也必须小心执行，因为作业期间的失败将使表格处于部分更新的状态。使用 Delta Lake，所有这些操作都可以事务性地执行，通过 Delta 日志中的新添加和删除记录替换任何更新的对象。Delta Lake 支持标准的 SQL UPSERT、DELETE 和 MERGE 语法。

### Streaming Ingest and Consumption

许多数据团队希望部署流式管道以**实时 ETL** 或聚合数据，但传统的云数据湖难以用于此目的。

Write Compaction.

Exactly-Once Streaming Writes.

Efficient Log Tailing.

> 这部分需要再仔细看看原文 https://www.vldb.org/pvldb/vol13/p3411-armbrust.pdf

### Data Layout Optimization

OPTIMIZE Command。用户可以手动对表格运行 OPTIMIZE 命令，该命令在不影响正在进行的事务的情况下压缩小对象，并计算任何缺失的统计信息。默认情况下，此操作旨在使每个数据对象的大小为 1 GB，我们发现此值适用于许多工作负载，但用户可以自定义此值。

Z-Ordering by Multiple Attributes

AUTO OPTIMIZE.

### Caching

缓存是安全的，因为 Delta Lake 表格中的数据、日志和检查点对象是不可变的。正如我们在第 6 节所示，从缓存中读取可以显著提高查询性能。

### Audit Logging

> 略

### Schema Evolution and Enforcement

Delta Lake 可以事务性地执行模式更改，并在需要时更新底层对象（例如，删除用户不再希望保留的列）。

> Schema Evolution + 事务，应该比较安全？

### Connectors to Query and ETL Engines

Delta Lake 通过 Apache Spark 的数据源 API [16] 提供了对 Spark SQL 和结构化流的全功能连接器。此外，它目前还提供了对多个其他系统的只读集成：Apache Hive、Presto、AWS Athena、AWS Redshift 和 Snowflake，使用户可以使用熟悉的工具查询 Delta 表格，并将它们与这些系统中的数据进行连接

> 数据存储层 Iceberg/Hudi/Delta Lake 然后接入查询层
>
> 是否是一种存算分离？

## DELTA LAKE USE CASES

在这些使用案例中，我们发现客户经常使用 Delta Lake 来简化其企业数据架构，通过直接在云对象存储上运行更多工作负载，并创建一个兼具数据湖和事务功能的“湖仓”系统。例如，考虑一个典型的数据管道，它从多个来源加载记录——例如，来自 OLTP 数据库的 CDC 日志和来自设施的传感器数据——然后通过 ETL 步骤传递，以使派生表格可用于数据仓库和数据科学工作负载（如图 1 所示）。传统的实现需要结合消息队列（如 Apache Kafka [11]）来计算需要实时计算的任何结果，数据湖用于长期存储，以及数据仓库（如 Redshift [3]）用于需要利用索引和快速节点附加存储设备（例如 SSD）进行快速分析查询的用户。这需要多个数据副本，并持续将数据导入每个系统。通过 Delta Lake，可以根据工作负载将其中几个存储系统替换为对象存储表格，利用 ACID 事务、流式 I/O 和 SSD 缓存等功能来恢复每个专用系统中的一些性能优化。尽管 Delta Lake 显然不能替代我们列出的所有系统功能，但我们发现，在许多情况下，它至少可以替代其中一些。Delta 的连接器（§4.8）还支持从许多现有引擎查询它。

> 为什么就不需要消息队列了呢？因为 ACID 可以做流批？
>
> 一般 delta lake 有高吞吐和低延迟的保证吗
>
> 而且本身就当作 data warehouse 了，比如 redshift, bigquery, snowflake 等等

### Data Engineering and ETL

augmenting traditional enterprise data sources (e.g., pointof-sale events in OLTP system)

ML workloads

### Data Warehousing and BI

business intelligence
The key technical features to support these workloads are usually efficient
storage formats (e.g. **columnar formats**), data access optimizations
such as clustering and indexing, fast storage hardware, and a suitably **optimized query engine**

### Compliance and Reproducibility

MLflow 是 Databricks 开发的开源模型管理平台，可以自动记录用于训练 ML 模型的数据集版本，并让开发人员重新加载它。

### Specialized Use Cases

1. 记录了广泛的计算机系统事件，例如网络上的 TCP 和 UDP 流、身份验证请求、SSH 登录等，并将其记录到一组跨度达数 PB 的 Delta Lake 表格中。多个程序化 ETL、SQL、图分析和机器学习作业随后在这些表格上运行，以搜索已知模式，这些模式表明存在入侵（例如，来自用户的可疑登录事件，或一组服务器导出大量数据）。其中许多是流式作业，以最小化检测问题的时间。此外，超过 100 名分析师直接查询源和派生的 Delta Lake 表格，以调查可疑警报或设计新的自动化监控作业。

   因此该组织使用 Delta Lake 的 ZORDER BY 功能重新排列 Parquet 对象中的记录，以提供跨多个维度的聚类。

2. 尽管传统的生物信息学工具使用了自定义数据格式（如 SAM、BAM 和 VCF [34, 24]），但许多组织现在将这些数据存储在数据湖格式（如 Parquet）中。Big Data Genomics 项目 [37] 开创了这种方法。Delta Lake 通过启用快速的多维查询（通过 Z-ordering）、ACID 事务和高效的 UPSERT 和 MERGE，进一步增强了生物信息学工作负载。

3. 使用 Delta Lake 管理多媒体数据集，例如上传到网站的一组图像

> ML 这里到底怎么做的，图片二进制编码，怎么变成 parquet

## PERFORMANCE EXPERIMENTS

研究了（1）具有**大量对象**或**分区的**表格对开源大数据系统的影响，这促使 Delta Lake 决定将**元数据和统计信息集中到检查点中**，以及（2）Z-ordering 对来自大型 Delta Lake 使用案例的选择性查询工作负载的影响。我们还展示了 Delta 在 TPC-DS 上的查询性能优于 Parquet，并且对写入工作负载没有显著的开销。

### Impact of Many Objects or Partitions

Delta Lake 的许多设计决策源于云对象存储中列出和读取对象的高延迟

小文件在 HDFS [36] 中也是一个问题，但在云存储中的性能影响更严重。

![](https://s2.loli.net/2024/11/12/vpUZKHs3iGnkr6a.png)

### Impact of Z-Ordering

> Z-Ordering 是一种用于多维数据的空间填充曲线优化技术，旨在提高数据读取性能，特别是在大数据环境中。它通过将数据按多维空间中的 Z 形顺序重新排序，最大限度地提高数据的局部性，从而提高查询效率。Z-Ordering 在处理具有多列过滤条件的查询时特别有效
>
> 多维查询：当查询包含多个过滤条件（如 age 和 income）时，Z-Ordering 可以显著提高查询性能。
>
> 大数据处理：在处理大量数据时，通过 Z-Ordering 可以减少 I/O 操作，提高数据读取效率。

### TPC-DS Performance

为了评估 Delta Lake 在标准 DBMS 基准测试中的端到端性能，我们在 Databricks Runtime（我们的 Apache Spark 实现）上使用 Delta Lake 和 Parquet 文件格式，以及在流行云服务中的 Spark 和 Presto 实现上运行了 TPC-DS 功率测试

### Write Performance

## DISCUSSION AND LIMITATIONS

我们在 Delta Lake 上的经验表明，许多企业数据处理工作负载可以在云对象存储上实现 ACID 事务，并且它们可以支持**大规模的流式、批处理和交互式工作负载**。Delta Lake 的设计特别有吸引力，因为它不需要任何其他重量级系统来调解对云存储的访问，使其易于部署，并且可以直接从支持 Parquet 的广泛查询引擎访问。Delta Lake 对 ACID 的支持然后启用了其他强大的性能和管理功能。

尽管如此，Delta Lake 的设计和当前实现有一些限制，这些限制是未来工作的有趣途径。首先，Delta Lake 目前仅在**单个表格内提供序列化事务**，因为每个表格都有自己的事务日志。**跨多个表格共享事务日志将消除这一限制**，但可能会增加通过乐观并发追加日志记录的争用。对于非常高的事务量，协调器也可以在不成为数据对象的读写路径一部分的情况下调解对日志的写访问。

其次，对于流式工作负载，Delta Lake 受限于底层云对象存储的延迟。例如，使用对象存储操作很难实现**毫秒级的流式延迟**。然而，我们发现，对于希望运行并行作业的大规模企业工作负载，使用 Delta Lake 表格的秒级延迟是可以接受的。

第三，Delta Lake 目前**不支持二级索引**（除了每个数据对象的最小/最大统计信息），但我们已经开始原型化基于 Bloom 过滤器的索引。Delta 的 ACID 事务允许我们与基础数据的更改一起事务性地更新此类索引。

## RELATED WORK

## CONCLUSION
