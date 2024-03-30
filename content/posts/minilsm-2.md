+++
title = 'Mini-LSM Week 1 Da2'
date = 2024-03-23T16:51:28-04:00
draft = false
tags = ['Database', 'LSM']
+++

## Mini-LSM Week 1 Day2

Week1 Day2 的内容，实现 Merge Iterator

https://skyzh.github.io/mini-lsm/week1-02-merge-iterator.html

## Merge Iterator

![](https://s2.loli.net/2024/03/24/ZBOrfH1gUo54Cnp.png)

本次需要实现：

- Memtable Iterator
- Merge Iterator
- LSM read path `scan` for memtables

## Task1: Memtable Iterator

修改 `src/mem_table.rs`，实现 `scan` 接口，在一组 key-value pairs 上创建 iterator API 来迭代。在上一节已经实现了 `get` 和创建 immutable memtable 的逻辑，此时 LSM state 有多个 memtables。所以 `scan` 需要在一个 memtable 上创建 iterator，然后在所有 memtables 上创建一个 merge iterator

所有 LSM iterators 在 `lsm_iterator.rs` 中的 `StorageIterator` 特征，有四个函数 `key`, `value`, `next` 和 `is_valid`。当 iterator 创建后，cursor 会停在某些元素，`key/value` 会返回 memtable/block/SST 中的第一个满足 start condition 的 key，比如 start key。这两个接口会返回 `&[u8]` 引用类型防止 copy。本节的 iterator interface 和 Rust iterator 不太一样。

`next` 将 cursor 移动到下一个元素，`is_valid` 检查 iterator 是否终止或者发生错误。可以假设 `next` 会在 `is_valid` 返回成功时才会被调用。`FusedIterator` 是一个 wrapper 包含了所有的 iterator，会防止不正确的 `next` 调用，比如 not valid。

回到 memtable iterator, 这个结构体没有任何生命周期。如果你创建了一个 `Vec<u64>` 并且调用 `vec.iter()`，这个 iterator 类型会是 `VecIterator<'a>` 表示 vec 的生命周期。同样地，`SkipMap` 也是一样的，`iter` 接口返回一个带生命周期的 iterator。但是，在 Mini LSM 中，我们不希望这样的生命周期出现在迭代器中，使得整个系统变得过分复杂和难以编译。

如果迭代器没有生命周期参数 lifetime generics parameter，我们应该保证不管什么时候用 iterator，底层的 skiplist 都不应该被释放。这样的唯一做法是将 `Arc<SkipMap>` 对象放到 iterator 本身：

```Rust
pub struct MemetableIteraotr {
    map: Arc<SkipMap<Bytes, Bytes>>,
    iter: SkipMapRangeIter<'???>,
}
```

> 题外话，为什么有些语言的 struct 用逗号分隔，有些用分号，有些用换行呢？写多了会不会混淆

此时有新的问题：我们想表示 `iterator` 的生命周期和 `map` 是一样的，应该怎么做？

这是 Mini LSM 中最 tricky 的 Rust 技巧：`self-referential` 结构：

```Rust
pub struct MemtableIterator { // <- with lifetime 'this
    map: Arc<SkipMap<Bytes, Bytes>>,
    iter: SkipMapRangeIter<'this>,
}
```

这样就解决了问题，也可以用第三方库 `ouroboros` 来解决这个问题，它提供了一个简单的方法定义 `self-referential` 结构，也可以用 unsafe Rust 来解决（`ouroboros` 就是用 unsafe Rust 实现的）

教程已经定义了 `self-referential` `MemtableIterator`，只需要实现 `MemtableIterator` 和 `Memtable::scan` 接口：

## Task1: Solution

观察 `Memtable::scan` 函数，`pub fn scan(&self, _lower: Bound<&[u8]>, _upper: Bound<&[u8]>) -> MemTableIterator`，其中 `lower` 和 `upper` 属于 `Bound` 类型，表示一个范围的 keys（左闭右开），比如 `(1..12).start_bound(), Included(&1)`, `(1..12).end_bound(), Excluded(&12)`。`BTreeMap` 通过 `map.range((Excluded(3), Included(8)))` 来获取 `(3, 8]` 范围内的 key-value。所以对于 memtable 的 scan，应该获取 `[lower, upper]` 的起始迭代器：

```Rust
/// Get an iterator over a range of keys.
pub fn scan(&self, lower: Bound<&[u8]>, upper: Bound<&[u8]>) -> MemTableIterator {
    let mut iter = MemTableIterator::new(
        self.map.clone(),                                      // Arc<SkipMap<Bytes, Bytes>>
        |map| map.range((map_bound(lower), map_bound(upper))), // iter FnOnce
        (Bytes::new(), Bytes::new()),                          // Stores the key-value pair.
    );
    iter.next().unwrap();
    iter
}
```

调用 `MemTableIterator::new` 新建一个 MemTableIterator，其结构体为：

```Rust
type SkipMapRangeIter<'a> =
    crossbeam_skiplist::map::Range<'a, Bytes, (Bound<Bytes>, Bound<Bytes>), Bytes, Bytes>;

#[self_referencing]
pub struct MemTableIterator {
    /// Stores a reference to the skipmap.
    map: Arc<SkipMap<Bytes, Bytes>>,
    /// Stores a skipmap iterator that refers to the lifetime of `MemTableIterator` itself.
    #[borrows(map)]
    #[not_covariant]
    iter: SkipMapRangeIter<'this>,
    /// Stores the current key-value pair.
    item: (Bytes, Bytes),
}
```

这一个结构体令 Rust 初学的我非常懵，尤其是 `#[borrows(map)]` 找了很久没有找到在哪，而且 `iter` 我以为传入一个结构体，但实际上应该传入一个函数闭包，这个函数闭包的返回类型是 `Range`。所以第一个参数是一个 Arc 变量 `self.map.clone()` 返回当前的 map。第二个参数是 `|map| map.range((map_bound(lower), map_bound(upper)))`，本身其实是一个 iter，可以调用 next 等等，然后是 `item` 用于存当前的 key-value 对。创建完成后需要调用一次 `iter.next().unwrap()` 走到第一个 Range，然后返回。

然后为 `MemTableIterator` 实现 `StorageIterator` 特征，具有 `value`, `key`, `is_valid` 和 `next` 接口：

```Rust
impl StorageIterator for MemTableIterator {
    type KeyType<'a> = KeySlice<'a>;

    fn value(&self) -> &[u8] {
        &self.borrow_item().1.chunk()
    }

    fn key(&self) -> KeySlice {
        KeySlice::from_slice(&self.borrow_item().0.chunk())
    }

    fn is_valid(&self) -> bool {
        !&self.borrow_item().0.is_empty()
    }

    fn next(&mut self) -> Result<()> {
        let entry = self.with_iter_mut(|iter| iter.next());
        let item = entry.map(|en| (en.key().clone(), en.value().clone()));
        self.with_mut(|iter| *iter.item = item.unwrap_or_default());
        Ok(())
    }
}
```

这里的 `&self.borrow_item()` 表示借用当前的 item，所以是引用，并且用 `.chunk()` 将 `Bytes` 转成 `&[u8]` 类型，`value`, `key` 和 `is_valid` 接口实现都很类似。

而 `next` 实现起来比较麻烦，大致思路是先通过 iter 得到下一个 Entry，然后将其值转成 `(Bytes, Bytes)` 类型然后赋给当前 `self.item`，同时修改本体所以需要用 `self.with_mut()`

此时调用 `cargo x scheck` 可以通过前两个测试

## Task2: Merge Iterator

本节需要修改 `src/iterators/merge_iterator.rs`。

现在你已经有多个 memtables，你需要创建多个 memtable iterators，并且需要 merge 所有 memtables 返回的结果，并且每个 key 对应最新的结果。

`MergeIterator` 维护一个 binary heap，你需要处理错误（比如当 iterator not valid 时），并且需要保证 key-value 对是最新的，比如：

```
iter1: b->del, c->4, d->5
iter2: a->1, b->2, c->3
iter3: e->4
```

merge iterator 返回的结果应该是：

```
a->1, b->del, c->4, d->5, e->4
```

MergeIterator 的构造器 constructor 接受一个 vector of iterators 参数，假设 index 小的是新的。

但是错误处理有个陷阱，比如：

```Rust
let Some(mut inner_iter) = self.iters.peek_mut() {
    inner_iter.next()?;
}
```

如果 `next` 返回了错误（disk failure, network failure, checksum error, etc），但是当跳出了 if 条件并且返回错误给调用者时，`PeekMut` 的结果会移动 heap 里的元素，导致访问到 invalid iterator。所以，需要额外的错误处理，而不是使用在 `PeekMut` 范围里用 `?`。

我们想避免尽可能 `dynamic dispatch`，所以我们不使用 `Box<dynStorageIterator>`。反而使用静态的分发，使用泛型，并且 `StorageIterator` 使用了 `generic associated type (GAT)`，所以它可以支持 `KeySlice` 和 `&[u8]` 同时作为 key 的类型。我们会在将来的作业改变 `KeySlice` 使其包含 timestamp。

> Rust 中的动态分发，比如 Trait Objects，类似虚继承？
>
> Generic Associated Type GAT 指的是通用关联类型，可以使得关联类型依赖于 trait 方法
>
> 比如 `trait Processor{ type Output<'a>; fn process<'a>(&self, data: &a str) -> Self::Output<'a>; }` 在 `impl` 时再指定 type 是什么。

为了开始本节，我们会使用 `Key<T>` 表示 LSM key 的类型，并且区分它的值类型。你应该使用 `Key<T>` 提供的接口而不是直接访问其内部值。我们会为这个 key 添加时间戳，并用这个 key abstraction 过渡会比较顺利。所以目前 `KeySlice` 和 `&[u8]` 是一样的，`KeyVec` 和 `Vec<u8>` 一样，`KeyBates` 和 `Bytes` 也是一样的。

## Task2: Solution

这一节主要实现 `src/iterators/merge_iterators` 中的 `MergeIterator::key`,`MergeIterator::value`, `MergeIterator::is_valid` 和 `MergeIterator::next`，并且实现 `MergeIterator::create`

首先是 `MergeIterator::create`，接受一个 iters 数组，需要将 `iters: Vec<Box<I>>` 转换成 `BinaryHeap<HeapWrapper<I>, Global>`，即优先队列 / 堆 / 完全二叉树，教程已经实现了 `HeapWrapper` 的比较特征，所以只需要将 `iters` 遍历一遍，并且入堆。

比如传入 `[[("a", 1), ("b", 2), ("c", 3)], [("a", 1.2), ("d", 4)]]` 这样的数组，先实现 `MergeIterator::create`：

```Rust
impl<I: StorageIterator> MergeIterator<I> {
    pub fn create(iters: Vec<Box<I>>) -> Self {
        let mut heap = BinaryHeap::new();
        if iters.is_empty() {
            return Self {
                iters: heap,
                current: None,
            };
        }
        for (i, iter) in iters.into_iter().enumerate() {
            if iter.is_valid() {
                heap.push(HeapWrapper(i, iter));
            }
        }

        let current = heap.pop();
        Self {
            iters: heap,
            current,
        }
    }
}
```

先检查 `iters.is_empty()` 传入的数组是否为空，然后遍历 `iters` 数组，使用 `into_iter()` 消耗原数组，转移所有权，然后 `enumerate()` 遍历，需要检查 `iter.key()` 是否是空的。

```Rust
fn key(&self) -> KeySlice {
    self.current.as_ref().unwrap().1.key()
}

fn value(&self) -> &[u8] {
    self.current.as_ref().unwrap().1.value()
}

fn is_valid(&self) -> bool {
    self.current
        .as_ref()
        .map(|x| x.1.is_valid())
        .unwrap_or(false)
}
```

`key`, `value`, `is_valid` 接口不再重述，但需要注意用 `as_ref()` 取得引用。`next` 的实现比较绕，因为对于 `MergeIterator`，有时候前面的 key 会小于堆顶的 key，所以需要进行交换：

```Rust
fn next(&mut self) -> Result<()> {
    let current = self.current.as_mut().unwrap();
    while let Some(mut iter) = self.iters.peek_mut() {
        if iter.1.key() == current.1.key() {
            let res = iter.1.next();
            if let Err(e) = res {
                PeekMut::pop(iter);
                return Err(e);
            } else {
                if !iter.1.is_valid() {
                    PeekMut::pop(iter);
                }
            }
        } else {
            break;
        }
    }

    current.1.next()?;
    if !current.1.is_valid() {
        if let Some(iter) = self.iters.pop() {
            *current = iter;
        }
    } else {
        // if the current key is smaller, swap it with the top of the heap
        // e.g. current "e" 101 < heap top iter key "d" 100
        // PartialOrd for HeapWrapper will reverse the ordering
        // so that the top of the heap is the smallest key
        if let Some(mut iter) = self.iters.peek_mut() {
            if !(*iter < *current) {
                std::mem::swap(&mut *iter, current);
            }
        }
    }
    Ok(())
}
```

首先是用 `while` 循环检查当前堆顶的 `iter.1.key()` 如果和当前的 `current.1.key()` 相同，则应该以 `current` 为准（最新版本），并且调用 `iter.1.next()` 走到下一个 `key`。

然后使用 `current.1.next()` 走到下一个 `key`，但此时就要处理新版本中的 `key` 实际小于 `iters.peek_mut()` 时的情况，但由于为了实现堆，`HeapWrapper` 重载了 `PartialOrd` 并反转了结果，所以堆顶的 key 是小的，那么比较堆顶 `iter` 和 `current` 就需要反过来。

## Task3: LSM Iterator + Fused Iterator

本节需要修改 `src/lsm_iterator.rs`

我们使用 `LsmIterator` 结构表示 LSM 内部的 iterators，在整个 LSM 教程中，会有多个 iterators 被加进系统，所以你需要多次修改这个结构。现在由于只有多个 memtables 所以定义为

```Rust
type LsmIteratorInner = MergeIterator<MemTableIterator>;
```

你可以提前实现 `LsmIterator` 结构，调用 inner iterator 并且跳过 deleted keys.

但本节不测试 `LsmIterator`，但会在下一节任务 Task4 有个 integration 整合。

我们想提供额外的安全性，防止用户用错 iterators。在 iterator not valid 时用户不应该调用 `key` `value` 或 `next` 接口。同时，当 `next` 返回错误时，用户不应该再使用这个 iterator。`FusedIterator` 是一个 iterator wrapper 用于 normalize 规范化所有 iterators 的行为。

## Task3: Solution

观察 `FusedIterator`，它包含了一个 iter 和一个 bool 字段表示是否是错误的：

```Rust
pub struct FusedIterator<I: StorageIterator> {
    iter: I,
    has_errored: bool,
}
```

本节需要实现 `FusedIterator::is_valid`, `FusedIterator::key`, `FusedIterator::value`, `FusedIterator::next`，同时保证如果 `!self.is_valid()` 就应该 `panic`:

```Rust
impl<I: StorageIterator> StorageIterator for FusedIterator<I> {
    type KeyType<'a> = I::KeyType<'a> where Self: 'a;

    fn is_valid(&self) -> bool {
        // first check if the iterator has errored
        // iter.is_valid() may iter to next
        !self.has_errored && self.iter.is_valid()
    }

    fn key(&self) -> Self::KeyType<'_> {
        if !self.is_valid() {
            panic!("called key on invalid iterator");
        }
        self.iter.key()
    }

    fn value(&self) -> &[u8] {
        if !self.is_valid() {
            panic!("called value on invalid iterator");
        }
        self.iter.value()
    }

    fn next(&mut self) -> Result<()> {
        if self.has_errored {
            return Err(anyhow::anyhow!("called next on invalid iterator"));
        }

        if self.iter.is_valid() {
            let res = self.iter.next();
            if let Err(e) = res {
                self.has_errored = true;
                return Err(e);
            }
        }
        Ok(())
    }
}
```

逻辑比较简单，但 `is_valid()` 最好是先检查 `self.has_errored`，不然先调用 `iter.is_valid()` 可能会走 `next` 导致 index 不对

## Task4: Read Path - Scan

本节任务需要修改 `src/lsm_storage.rs`

当所有 iterators 实现后，你就可以实现 LSM 引擎 `scan` 接口了。你可以简单地创建一个 LSM iterator 和 memtable iterator（记得在 merge iterator 最前面放最新的 memtable ），此时你的 存储引擎就可以做 scan 扫描请求了。

## Task4: Solution

结合 Day1 和 Day2，从实现 memtable 和 memtable iterator，到多个 memtables 和多个 iterators，现在需要将这些 memtable iterators 结合起来，实现一个扫描方法，回顾单个 `Memtable::scan`，接受一个 `lower` 和 `upper` 参数，返回一个 `MemtableIterator` 迭代器，包含满足 `(lower, upper)` 的键值对：

```Rust
/// Get an iterator over a range of keys.
pub fn scan(&self, lower: Bound<&[u8]>, upper: Bound<&[u8]>) -> MemTableIterator {
    let mut iter = MemTableIterator::new(
        self.map.clone(),                                      // Arc<SkipMap<Bytes, Bytes>>
        |map| map.range((map_bound(lower), map_bound(upper))), // iter FnOnce
        (Bytes::new(), Bytes::new()),                          // Stores the key-value pair.
    );
    iter.next().unwrap();
    iter
}
```

同理，`LsmStorageInner` 也是接受一个 `lower` 和 `upper` 参数，返回一个 `FusedIterator<LsmIterator>` 融合的迭代器。

具体地，需要先拿出所有 memtables iterators，创建一个数组存储这些 `MemtableIterator`，先存入最新的 `memtable` 然后推入后面的 `imm_memtables`，并使用 `MergeIterator::create()` 创建 MergeIterator 之后创建 FusedIterator

```Rust
pub fn scan(
    &self,
    lower: Bound<&[u8]>,
    upper: Bound<&[u8]>,
) -> Result<FusedIterator<LsmIterator>> {
    let guard = self.state.read();

    let mut memtable_iters = Vec::new();
    memtable_iters.push(Box::new(guard.memtable.scan(lower, upper)));
    for memtable in guard.imm_memtables.iter() {
        memtable_iters.push(Box::new(memtable.scan(lower, upper)));
    }
    let memtable_iter = MergeIterator::create(memtable_iters);

    drop(guard);
    Ok(FusedIterator::new(LsmIterator::new(memtable_iter)?))
}
```

但我的实现方法锁粒度大，checkpoint 使用了 `Arc::clone(&guard)` 克隆了一个原子变量，这样临界区小，可以立刻释放锁还保证了每个线程都能读到一样的 snapshot：

```Rust
let snapshot = {
    let guard = self.state.read();
    Arc::clone(&guard)
}; // drop global lock here
```

但此时还无法通过测试，观察测试可以发现，`LsmStorageInner` 在 `delete` 某个 key 后，我的实现方法无法将其忽略掉，所以要重新考虑得到一个 `LsmIterator` 迭代器后，如果 `next` 为空或者 deleted key，应该怎么做：

```Rust
impl LsmIterator {
    pub(crate) fn new(iter: LsmIteratorInner) -> Result<Self> {
        let mut iter = Self { inner: iter };
        iter.move_to_non_delete()?;
        Ok(iter)
    }
}

impl LsmIterator {
    fn move_to_non_delete(&mut self) -> Result<()> {
        while self.is_valid() && self.inner.value().is_empty() {
            self.inner.next()?;
        }
        Ok(())
    }
}

impl StorageIterator for LsmIterator {
    // ...
    fn next(&mut self) -> Result<()> {
        self.inner.next()?;
        self.move_to_non_delete()?;
        Ok(())
    }
}
```

这里 `self.inner.next()` 逻辑不变，但新增一个函数判断经过 `next` 后当前 `self.is_empty`，如果是则继续迭代，同时还需要修改 `new()` 因为新建 `LsmIterator` 时候也可能传入带有删除键的 `LsmIteratorInner` （测试就是这样做的）

## Conclusion

这次写下来比较吃力，一方面对 Rust 语法不太熟悉，尤其是类型不太懂。另一方面是对 MVCC 和多线程并发如何在 Rust 中实现还没有好的理解，需要看一下 Rust 多线程编程的一些例子。

- What is the time/space complexity of using your merge iterator?

  Merge Iterator 时间复杂度看上去是 O(log N \* M)，N 个 memtable，每个大小 M，空间复杂度则应该是 O(N \* M)

2. Why do we need a self-referential structure for memtable iterator?

   自引用, 指的是一个结构体中, 有一个字段需要引用自己的另一个 field, 在 `mem_table.rs` 中有

   ```Rust
   #[self_referencing]
   pub struct MemTableIterator {
       map: Arc<SkipMap<Bytes, Bytes>>,
       /// Stores a skipmap iterator that refers to the lifetime of `MemTableIterator` itself.
       #[borrows(map)]
       #[not_covariant]
       iter: SkipMapRangeIter<'this>,  // 需要引用 map 的值
   }
   ```

   就是教程中说的，为了标记 `MemtableIterator` 的生命周期和 `map` 一致, 结合 `<'this>` 与 `self_referencing` 使得他们的生命周期一致. 否则会出现 使用值 和 值的引用 同时出现, 最终所有权转移 和 借用一起发生

3. If a key is removed (there is a delete tombstone), do you need to return it to the user? Where did you handle this logic?

   这里参考了 checkpoint 的实现, 跳过了 deleted key, 没有返回给用户

4. If we want to get rid of self-referential structure and have a lifetime on the memtable iterator (i.e., MemtableIterator<'a>, where 'a = memtable or LsmStorageInner lifetime), is it still possible to implement the scan functionality?
5. What happens if (1) we create an iterator on the skiplist memtable (2) someone inserts new keys into the memtable (3) will the iterator see the new key?
6. What happens if your key comparator cannot give the binary heap implementation a stable order?
7. Why do we need to ensure the merge iterator returns data in the iterator construction order?
8. Is it possible to implement a Rust-style iterator (i.e., next(&self) -> (Key, Value)) for LSM iterators? What are the pros/cons?
9. The scan interface is like fn scan(&self, lower: Bound<&[u8]>, upper: Bound<&[u8]>). How to make this API compatible with Rust-style range (i.e., key_a..key_b)? If you implement this, try to pass a full range .. to the interface and see what will happen.
10. The starter code provides the merge iterator interface to store Box<I> instead of I. What might be the reason behind that?

> 剩下的还是等写完全部有个具体概念回来再来补充
