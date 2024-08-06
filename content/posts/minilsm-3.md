+++
title = 'Minilsm 3'
date = 2024-07-27T22:10:57+08:00
draft = false
tags = ['Database', 'LSM']
+++

## Mini-LSM Week 1 Day3

Week1 Day3 的内容，实现 SST block 编解码和 iterator

## Task 1: Block Builder

前两章实现了 LSM 在内存中的结构，现在实现 on-disk 的结构，即 Block 块。

Blocks 一般 4-KB 大小（可能因存储介质不同），和操作系统以及 SSD 中的页 page 大小一致。块存储排序后的 key-value pairs。

SST 由多个 blocks 组成，当 memtables 的数量超过了 system limit，就会 flush memtables 成为 SST。这一章实现 encoding and decoding of a block.

需要修改的文件：

```
src/block/builder.rs
src/block.rs
```

教程的 block 的编码格式为：

```
----------------------------------------------------------------------------------------------------
|             Data Section             |              Offset Section             |      Extra      |
----------------------------------------------------------------------------------------------------
| Entry #1 | Entry #2 | ... | Entry #N | Offset #1 | Offset #2 | ... | Offset #N | num_of_elements |
----------------------------------------------------------------------------------------------------

```

每个 Entry 是 key-value 对：

```
-----------------------------------------------------------------------
|                           Entry #1                            | ... |
-----------------------------------------------------------------------
| key_len (2B) | key (keylen) | value_len (2B) | value (varlen) | ... |
-----------------------------------------------------------------------

```

key_len 和 value_len 都是 2 个字节，所以最长长度为 $2^16 = 65535$，一般使用 `u16` 表示

block 具有大小限制 `target_size`，除非第一个 key-value pair 超过了 block size，不然你需要保证编码后的 block size 小于或等于 `target_size` (提供的代码里，`target_size` 与 `block_size` 本质一样)

当 `build` 被调用时，`BlockBuilder` 会产生 data part 和 unencoded entry offsets。信息会被存在 `Block` 结构中，key-value entries 使用 raw 格式，offsets 用单独的 vector，这减少了解码数据时不必要的内存分配和处理开销，你只需要简单地拷贝 raw block data 到 `data` vector 并且每 2 个 bytes 进行 decode entry offsets，而不是创建 `Vec<(Vec<u8>, Vec<u8>)>` 这样的结构，在一个 block 和内存中去存所有的 key-value pairs。这样 compact memory layout 更高效。

在 `Block::encode` 和 `Block::decode`，你需要按照上述的结构 encode/decode block.

## Task 1: Solution

查看 `src/block/builder.rs` 观察 Block 结构体，`offsets Vec<u16>` 偏移量， `data vec<u8>` kv 数据等等，具有 `new(block_size: usize) -> Self`, `add(&mut self, key: KeySlice, value: &[u8]) -> bool`, `is_empty(&self) -> bool` 和 `build(self) -> Block` 方法。

```Rust
/// Builds a block.
pub struct BlockBuilder {
    /// Offsets of each key-value entries.
    offsets: Vec<u16>,
    /// All serialized key-value pairs in the block.
    data: Vec<u8>,
    /// The expected block size.
    block_size: usize,
    /// The first key in the block
    first_key: KeyVec,
}

/// Creates a new block builder.
pub fn new(block_size: usize) -> Self {
    Slef {
        offsets: Vec::new(),
        data: Vec::new(),
        block_size,
        first_key: KeyVec::new(),
    }
}
```

先实现 `new(block_size)` 方法创建一个 `BlockBuilder` 结构体，然后实现 `add` 添加一个 key-value 值函数：

```Rust

```

## Task 2: Block Iterator

`src/block/iterator.rs`，这一小节，因为有了 encoded block，需要实现 `BlockIterator` 接口，使得用户可以 lookup/scan blocks 里的 keys。

`BlockIterator` 可以被 `Arc<Block>` 实现，如果 `create_and_seek_to_first` 被调用，它会放在 block 的第一个 key。如果 `create_and_seek_to_key` 被调用，iterator 会被放在第一个 `>=` 大于等于相应 key 的位置，比如 `1, 3, 5` 在一个 Block 时

```rust
let mut iter = BlockIterator::create_and_seek_to_key(block, b"2"); // 创建 key 2
assert_eq!(iter.key(), b"3"); // 此时 iterator 位置在第一个大于等于 2 的位置即 3 的位置
```

上面的 `seek 2` 将使迭代器定位在下一个可用键 `2`，在本例中为 `3`。

iterator 应该从 block 拷贝 `key` 并且存到 iterator 本身（未来会有 key compression 压缩的内容），对于值 value，必须在 iterator 存储起始/结束 begin/end offset 偏移，并且不能拷贝。

当 `next` 被调用，iterator 会移动到下一个位置。如果抵达 block 结束位置，可以设置 `key` 为空然后从 `is_valid` 返回 `false`，这样调用者可以切换到另外的 block。
