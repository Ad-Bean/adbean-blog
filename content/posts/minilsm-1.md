+++
title = 'Mini-LSM 01'
date = 2024-03-14T15:04:33-04:00
draft = false
tags = ['Database', 'LSM']
+++

## Mini-LSM Day1

> 磨洋工，天天写作业摸鱼到了三月，找不到工作实在是焦虑和迷茫，但又什么都不想干。记录下 LSM 的学习过程
>
> https://skyzh.github.io/mini-lsm/

## 前言

使用 Rust 实现 LSM-Tree 存储结构

### 什么是 LSM，为什么 LSM

LSM, Log-structured merge trees, 是一种维护 key-value 对的数据结构。这种数据结构广泛用于分布式数据库（TiDB， CockroachDB）及其存储引擎。RocksDB 基于 LevelDB 实现了 LSM-Tree 存储引擎。LSM 提供许多 key-value 访问函数，并且被用于许多生产系统。

LSM Tree 属于 append-friendly 添加友好的数据结构。将 LSM 与 B-Tree 或 RB-Tree 等 key-value 数据结构比较。对于后者，所有数据都在原地操作，比如你想要更新 key 对应的 value，存储引擎会用新值直接覆盖原有的值（内存或硬盘）。而在 LSM-Tree 中，所有写操作 (比如 insert, update, delete) 会 lazily 懒地应用到存储中，存储引擎批量地将这些操作分成 SST (sorted string table) 文件并且写入磁盘。一旦被写入到磁盘，存储引擎不会直接修改它们。而是在 compaction 后台任务中，存储引擎会合并这些文件，应用这些更新或删除。

这个架构设计使得 LSM-Tree 易于使用：

1. 数据在存储中是不可变的 immutable。并发控制更加直观。将后台任务（compaction）卸载? (offload) 到远程服务器是可行的。在云原生存储系统比如 S3 存储或提供数据也变得可行。
2. 改变 compaction 算法允许存储引擎在读放大、写放大、空间放大之间取得平衡。LSM-Tree 是多功能的，通过调整 compaction 参数，可以优化 LSM 数据结构在不同工作负载中的表现。

### 前提条件

- Rust
- key-value store engine ([PingCap Bitcask 项目](https://github.com/pingcap/talent-plan/tree/master/courses/rust/projects/project-2))
- LevelDB 一些概念能帮助理解 可变和不可变 mem-table, SST, compaction, WAL 等等。

### 期望

学习了解基于 LSM-Tree 的存储引擎是如何工作的，尤其是从各种 tradeoffs 权衡中寻找最优的、符合你的工作负载的需求或目标。

## 总览

![LSM-Tree](https://s2.loli.net/2024/03/15/lmvd4PQyrZazuKc.png)

第一部分主要研究 LSM Tree 的存储结构和存储格式。

第二部分主要关于 compaction 和实现持久化。

第三部分实现 MVCC 多版本控制。

### LSM

LSM 存储大致含有三部分：

1. Write-Ahead Log 用于持久化数据（Recovery）
2. SSTs 在磁盘用于维护 LSM-Tree 结构
3. Mem-Tables 在内存用于 batching 批处理写操作

存储引擎大致提供这些接口：

- `Put(key, value)`: 在 LSM Tree 存储一个 key-value 键值对
- `Delete(key)`: 根据 key 删除一个 key-value 键值对
- `Get(key)`: 根据 key 获得 value
- `Scan(range)`: 获得一个范围的 key-value 键值对

为了保证持续化：

- `Sync()`: 确保所有操作在 `sync` 之前都被持久化到硬盘

一些引擎会结合 `Put` 和 `Delete` 成为一个操作 `WriteBatch`，接受一批键值对。

MiniLSM 教程假设 LSM 使用 leveled compaction，是广泛使用的。

### 写路径

![](https://s2.loli.net/2024/03/15/VwoSxREh23zgaMe.png)

写操作包含四步：

1. 在 WAL 写入键值对，在引擎 crash 时可以 recovery
2. 在 memtable 写入键值对。当 1 和 2 完成后，通知用户写操作完成
3. (后台) 当 memtable 满时，freeze 它们成为 immutable mem-tables 然后 flush 到硬盘成为 SST files
4. (后台) 引擎 compact 这些文件，按 levels 到一些低层的，维护一个好的 LSM 结构，减少读放大

### 读路径

![](https://s2.loli.net/2024/03/15/v9wUOk3GJElqXVH.png)

当需要读一个 key 时

1. probe 所有的 mem-tables，从最近的到最旧的
2. 如果 key 没找到，就搜索整个 LSM tree 包括 SSTs，直到找到这个数据

有两种读操作：lookup 和 scan，lookup 找到一个 key，scan 在一个范围里找到所有的 keys。

## MiniLSM

第一章需要构建必要的存储格式、读路径、写路径，以及一个基于 LSM 的 key-value store。

### Mem-Table

in-memory 的读写路径

![](https://s2.loli.net/2024/03/15/ybQxUzwYIpS6fju.png)

- 基于 skiplist 实现 memtables
- 实现 freezing memtable 的逻辑
- 实现 LSM 的 memtable 读路径 `get`

#### Task1 SkipList Memtable

`src/mem_table.rs` 基于 [crossbeam_skiplist](https://docs.rs/crossbeam-skiplist/latest/crossbeam_skiplist/) 跳表，支持 lock-free 并发读写。跳表是一个允许并发读写的有序 key-value map。

crossbeam-skiplist 提供了与 Rust std `BTreeMap` 类似的接口：insert, get 和 iter。区别在于修改的接口比如 insert 需要一个 immutable reference 到 skiplist 而不是一个 mutable。
