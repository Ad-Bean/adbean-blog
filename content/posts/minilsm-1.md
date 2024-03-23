+++
title = 'Mini-LSM Week 1 Day1'
date = 2024-03-22T14:04:33-04:00
draft = false
tags = ['Database', 'LSM']
+++

## Mini-LSM Week 1 Day1

记录下 LSM 的学习过程，感谢迟先生的教程 https://skyzh.github.io/mini-lsm/

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

### Task1: SkipList Memtable

`src/mem_table.rs` 基于 [crossbeam_skiplist](https://docs.rs/crossbeam-skiplist/latest/crossbeam_skiplist/) 跳表，支持 lock-free 并发读写。跳表是一个允许**并发读写**的有序 key-value map。

crossbeam-skiplist 提供了与 Rust std `BTreeMap` 类似的接口：insert, get 和 iter。区别在于修改的接口比如 insert 需要一个 **immutable reference** 到 skiplist 而不是一个 mutable。实现不应该使用任何 mutex 锁。

`MemTable` 结构没有 `delete` 接口，在 mini-lsm 项目中，deletion 可以用 key 和 empty value 表示。

Task1 需要实现 `MemTable::get` 和 `MemTable::put` 使得可以修改 memtable

`bytes` crate 包用于存取数据，`bytes::Byte` 和 `Arc<[u8]>` 类似。当 clone `Bytes` 或使用 `Bytes` 的切片时，数据不会被 copy, it is cheap to clone. 所以只是 create a new reference to the storage. When there are no references to the area, it will be freed.

> 跳表 skiplist 好像在之前并发的课上学过，但当时不理解有什么用。
>
> Redis 中的有序集合 zset，LevelDB、RocksDB、HBase 中 Memtable 都用到了 SkipList
>
> 查询的期望时间复杂度 O(log(n)) 和红黑树等平衡二叉树的区别？
>
> 基于有序链表，引入分层的概念，使得搜索可以像二分一样从高层往底层查找。
>
> 每个位于第 i 层的节点有 p 的概率出现在第 i+1 层
>
> 查找、删除、增加的简单实现：https://leetcode.cn/problems/design-skiplist/

### Task1 Solution

在 `src/mem_table.rs` 中实现 `impl Memtable` 中的

- `pub fn get(&self, key: &[u8]) -> Option<Bytes> {}` 函数
- `pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {}` 函数

Memtable 中有四个成员

- `map: Arc<SkipMap<Bytes, Bytes>>` 即跳表
- `wal: Option<Wal>` 是 WAL 日志
- `id: usize`
- `approximate_size: Arc<AtomicUsize>`

由于不需要关心跳表的具体实现方法，对于 Memtable 只需要调用 `map` 的 `insert` 和 `get` 就可以：

1. 首先实现 `Memtable::get` ，从跳表中获取数据，但 `key: &[u8]` 是一个切片比如 `b"key1"`。

   实现方法：使用 `self.map.get(key)` 获取相应的 value，由于 `map.get()` 的返回值是 `Option<Entry<'_, K, V>>`，需要将其转成 `Bytes`，所以使用 `map.get(key).map(|v| v.value())` 但此时 `v` 的类型是 `Option<&bytes::Bytes>` 还需要转换，可以使用 `clone()` 将结果转成函数返回类型的 `Option<Bytes>`

   ```Rust
   pub fn get(&self, key: &[u8]) -> Option<Bytes> {
       self.map.get(key).map(|v| v.value().clone())
       // v: Entry<'_, Bytes, Bytes>
   }
   ```

2. 再实现 `Memtable::put`，存入 `key: &[u8]` 和 `value: &[u8]` 并且返回 `Result<()>`。由于 task1 不需要关心跳表的实现，也不需要关心 WAL 日志，直接使用 `self.map.insert()`。但是该函数的签名是 `insert(&self, key: K, value: V) -> Entry<'_, K, V>`，此时 K 和 V 都是 `Bytes` 所以需要将 `key` 和 `value` 转换成 `Bytes` (使用 bytes 库提供的 `Bytes::copy_from_slice` 方法。)

   然后将 `insert()` 的返回值转成 `Result<()>` （这里参照其他函数直接返回 `OK(())`）

   ```Rust
   pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
   self.map
       .insert(Bytes::copy_from_slice(key), Bytes::copy_from_slice(value));
   Ok(())
   }
   ```

实现 `MemTable::create` 后，可以通过 `mini-lsm-starter tests::week1_day1::test_task1_memtable_overwrite` 和 `mini-lsm-starter tests::week1_day1::test_task1_memtable_get` 两个测试

### Task2: A Single Memtable in the Engine

Task2 需要修改 `src/lsm_storage.rs`，将 `Memtable` 加入到 `LSM State` 中。

在 `LsmStorageState::create` 中根据不同的 `compaction_options` 创建了一个 LSM 结构 `memtable, imm_memtable, l0_sstables, levels, sstables`。

默认情况下 `memtable: Arc::new(MemTable::create(0))` 创建了一个 id 为 0 的 `mutable memtable`。任何时间，引擎只会有一个 `mutable memtable`，通常有个大小限制（比如 256MB），并且当存满后会被 frozen 变成 immutable memtable

在 `lsm_storage.rs` 文件中，有两个结构表示了存储引擎：`MiniLSM` 和 `LsmStorageInner`，前者是一个 thin wrapper:

```Rust
pub struct MiniLsm {
    pub(crate) inner: Arc<LsmStorageInner>,
    flush_notifier: crossbeam_channel::Sender<()>,
    flush_thread: Mutex<Option<std::thread::JoinHandle<()>>>,
    compaction_notifier: crossbeam_channel::Sender<()>,
    compaction_thread: Mutex<Option<std::thread::JoinHandle<()>>>,
}
```

需要实现的方法大部分在 `LsmStorageInner` 直到 Week2 Compaction，这个结构包含了当前的 LSM storage engine 结构。Week 1 只用到 `memtable` 结构，并且是 **mutable memtable**。

Task2 需要实现 `LsmStorageInner::get`, `LsmStorageInner::put` 和 `LsmStorageInner::delete`，这些接口会分发请求到 `memtable`

考虑 `delete` 函数，应该是简单的将 `key` 对应的 `value` 修改为空，称为 `delete tombstone` 标记删除而不是真的直接删除。实现 `get` 时应该注意这点。

> 不直接删除的原因是什么？是为了 MVCC 还是为了后面的 compaction

为了访问 `memtable` 需要使用锁，`state lock`，实现 `put` 时只需要 `immutable reference`，所以修改 memtable 时候只需要调用 `state` 读锁。这种方法允许多线程对 memtable 并发访问。

> 如果 put 用了不可变的引用，则不会修改原数据结构，所以

### Task2 Solution

1. 先实现 `LsmStorageInner::put` 和 `LsmStorageInner::delete`，前者调用读锁，后者存入空值，然后在 `get` 中处理

   ```Rust
    pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
        self.state.read().memtable.put(key, value)
    }

    /// Remove a key from the storage by writing an empty value.
    pub fn delete(&self, key: &[u8]) -> Result<()> {
        self.state.read().memtable.put(key, b"") // handle empty value in get
    }
   ```

2. `get` 实现需要额外判断取出来的 `Bytes` 是不是空值

   ```Rust
    pub fn get(&self, key: &[u8]) -> Result<Option<Bytes>> {
        if let Some(v) = self.state.read().memtable.get(key) {
            if v.is_empty() {
                return Ok(None);
            }
            return Ok(Some(v));
        }
        Ok(None)
    }
   ```

### Task 3: Write Path - Freezing a Memtable

在 `src/lsm_storage.rs` 和 `src/mem_table.rs` 实现 freeze a memtable 功能。由于 memtable 无法一直增长，所以需要冷冻 freeze 起来，在当 memtable 超过大小限制时可以 flush，在 `lsm_storage` 中 `target_sst_size` 字段表示 SST table size 同时也是 memtable 的大致大小（比如 `1 << 20` 1MB）。

本 task 需要在 `put/delete` 时，估算 memtable 的大小，比如 put 时简单地将 keys and values 的字节大小加起来（如果 put 两次，你也可以计算两次）。一旦 memtable 达到大小限制，调用 `force_freeze_memtable` 冻结 memtable 然后创建一个新的 memtable。

因为此时可能有多个线程从存储引擎中获取数据，`force_freeze_memtable` 可能会被多个线程并行调用，需要考虑避免 race condition

有多个地方可以更改 LSM state: freeze a mutable memtable, flush memtable to SST, and GC/compaction. 所有的修改，都可能是 IO 操作，为了实现 `freeze_memtable` 需要调用 state 中的写锁：`let state = self.state.write()`, `state.immutable_memtable.push(/* something */)` 和 `state.memtable = create();`

但是，当需要为每个 memtable 创建 write-ahead log 时 `state.memtable = MemTable::create_with_wal()?` 由于需要几个毫秒比较耗时，其他线程就需要等待，造成 spike of latency 延迟尖峰。

可以将 IO 操作放在锁区域外，来解决这种问题（创建 Memtable 实际上不需要锁）：

```Rust
fn freeze_memtable(&self) {
    let memtable = MemTable::create_with_wal()?; // <- could take several milliseconds
    {
        let state = self.state.write();
        state.immutable_memtable.push(/* something */);
        state.memtable = memtable;
    }
}
```

但这引起了另一种情况：memtable 快要达到容量限制，并且两个线程成功 `put` 了两个 keys 到 memtable，并且都发现了 memtable 达到了容量限制。两个线程都会检查 size 然后决定调用 freeze，于是两个空的 memtable 都被创建了。

为了解决这种情况，所有的 state 修改操作都需要通过 state lock 同步：

```Rust
fn put(&self, key: &[u8], value: &[u8]) {
    // put things into the memtable, checks capacity, and drop the read lock on LSM state
    if memtable_reaches_capacity_on_put {
        let state_lock = self.state_lock.lock();
        if /* check again current memtable reaches capacity */ {
            self.freeze_memtable(&state_lock)?;
        }
    }
}
```

这种模式非常常见，比如 L0 flush 也是这样：

```Rust
fn force_flush_next_imm_memtable(&self) {
    let state_lock = self.state_lock.lock();
    // get the oldest memtable and drop the read lock on LSM state
    // write the contents to the disk
    // get the write lock on LSM state and update the state
}
```

在这个 Task，需要修改 `put` 和 `delete` 满足 memtable 的 soft capacity limit，当到达限制时调用 `force_freeze_memtable` 冻结 memtable。测试文件并没有测试并发场景，所以需要自己考虑多种 race condition 竞争情况。并且，时刻记住检查锁的区域，保证 critical section 临界区是最小的。

可以简单地赋给下一个 memtable id 为 `self.next_ssd_id()`, 注意 `imm_memtables` 存储了最新到最旧的 memtables，`imm_memtables.first()` 是最后一个被冻结的。

1. 修改 `Memtable::put`，计算 key 和 value 的长度，增加原子变量（使用 Relaxed 不需要线性一致）

   ```Rust
    pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
        let sz = key.len() + value.len();
        self.map
            .insert(Bytes::copy_from_slice(key), Bytes::copy_from_slice(value));
        self.approximate_size.fetch_add(sz, Ordering::Relaxed);
        Ok(())
    }
   ```

2. 实现 `LsmStorageInner::force_freeze_memtable`：

   ```Rust
    pub fn force_freeze_memtable(&self, _state_lock_observer: &MutexGuard<'_, ()>) -> Result<()> {
        let memtable_id = self.next_sst_id();
        let memtable = if self.options.enable_wal {
            Arc::new(MemTable::create_with_wal(
                memtable_id,
                self.path_of_wal(memtable_id),
            )?)
        } else {
            Arc::new(MemTable::create(memtable_id))
        };
        let mut state = self.state.write(); // acquire the lock
        let mut snapshot = state.as_ref().clone(); // first use as_ref() to convert Arc to &Arc and then clone() to make it mutable

        let old_memtable = std::mem::replace(&mut snapshot.memtable, memtable);
        snapshot.imm_memtables.insert(0, old_memtable.clone()); // insert to the front

        *state = Arc::new(snapshot); // update the state
        drop(state); // release the lock

        // old_memtable.sync_wal()?;
        Ok(())
    }
   ```

   获取新的 `memtable_id` 后根据 option 创建新的 Memtable，随后获取写锁，获取 `state` 的 mut 状态得到旧的 memtable，使用 `std::mem::replace` 替换成新的，随后释放写锁并且更新 `*state = Arc::new(snapshot)`。

### Task 4: Read Path - Get

修改 `src/lsm_storage.rs` read path 中的 `get` 函数，来获取最新版本的 key，保证你 probe 扫描了最新到最旧的 memtable:

```Rust
pub fn get(&self, key: &[u8]) -> Result<Option<Bytes>> {
  let state = self.state.read();
  if let Some(v) = state.memtable.get(key) {
      if v.is_empty() {
          return Ok(None);
      }
      return Ok(Some(v));
  }
  for memtable in state.imm_memtables.iter() {
      if let Some(v) = memtable.get(key) {
          if v.is_empty() {
              return Ok(None);
          }
          return Ok(Some(v));
      }
  }
  Ok(None)
}
```

### Conclusion

第一次写 Rust 项目，有很多不熟悉，写了蛮久。

通过 `cargo x copy-test --week 1 --day 1` 和 `cargo x scheck` 可以通过第一天的所有测试

- 为什么 memtable 不提供 delete 接口？

> 猜测是因为需要保持原子操作，比如说多版本操作 MVCC，或者 WAL 不好直接删，等到 compaction 时再处理

- 可以用其他数据结构实现 LSM 中的 memtable 吗？使用跳表的优劣势是什么？

> 可以使用红黑树等自平衡树结构，来实现平均时间复杂度 O(logn) 查询。但红黑树非常难写，跳表实现则很直观，同时且支持范围查询，内存占用也少一些。

- 为什么需要 `state` 和 `state_lock` 的组合？可以直接使用 `state.read()` 和 `state.write()` 吗？

> 在 `LsmStorageInner` 结构体中有 `state: Arc<RwLock<Arc<LsmStorageState>>>` 和 `state_lock: Mutex<()>` 组合
>
> 虽然在 week1 day1 可以不用 state_lcok （按理来说应该在 force_freeze_memtable 用 mutex 锁，防止竞争？）
>
> 这个问题不太懂，可能 Mutex 是为了防止多个线程同时 put 导致同时 freeze 的情况，此时 mutex 保护 size

- 为什么 store 和 probe memtables 的顺序很重要，如果一个 key 在多个 memtables，应该返回哪个版本的 value？

> 最先访问应该是内存中的 memtable，然后从 imm_memtable 从第一个开始探测（第一个是最新的），所以返回的也是最新的 latest version

- Is the memory layout of the memtable efficient / does it have good data locality? (Think of how `Byte` is implemented and stored in the skiplist...) What are the possible optimizations to make the memtable more efficient?

- So we are using `parking_lot` locks in this tutorial. Is its read-write lock a fair lock? What might happen to the readers trying to acquire the lock if there is one writer waiting for existing readers to stop?

- After freezing the memtable, is it possible that some threads still hold the old LSM state and wrote into these immutable memtables? How does your solution prevent it from happening?

- There are several places that you might first acquire a read lock on state, then drop it and acquire a write lock (these two operations might be in different functions but they happened sequentially due to one function calls the other). How does it differ from directly upgrading the read lock to a write lock? Is it necessary to upgrade instead of acquiring and dropping and what is the cost of doing the upgrade?

- More Memtable Formats. You may implement other memtable formats. For example, **BTree memtable**, **vector memtable**, and **ART memtable**.
