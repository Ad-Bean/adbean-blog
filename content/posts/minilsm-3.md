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

当 `build` 被调用时，`BlockBuilder` 会产生 data part 和 unencoded entry offsets。
