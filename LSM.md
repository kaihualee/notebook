# LSM Tree

[PingCAP视频](https://www.bilibili.com/video/av47654581?from=search&seid=17023278500344978625)

- NoSQL
    + Bigtable
    + HBase
    + Cassandra
    + Scylla
    + MongoDB
- Storage Engine
    + LevelDB
    + RocksDB
    + MyRocks
- NewSQL
    + TiDB (A Distributed SQL Database)
    + CockroachDB

1996: The Log-Structured Merge-Tree(LSM-Tree)
- Memory is small
- Disk is slow fro randowm access(r/w)
- Originally designed for fast-growing History Table
    + Data and indexs
    + Write heavy
    + Read sparse

Point Query

- Read放大, 最差 I/O: 2*(N - 1 + files num of level-0)
- 优化: Bloom filter + Page cache/Block cache

Range Query

- Must seek every sorted run
- Bloom filter not support range query
- 优化
    + 并发查询, 降低频响, 没有降低IO
    + 前缀Bloom 过滤器 (RocksDB)
    + SuRF Succinct Range Filter (SIGMOD 2018)
        * Trie => SuRF-Base => SuRF-Hash/SuRF-Real

读操作总结

- 读放大 => 过滤
- 查询性能高
- 错误率低
- 不能false positive, no false negative

Compaction - Tiered vs Leveled
- compaction rate is a problem
    + Too fast - write amplification
    + Too slow - read amplification and space amplification.

Pipelined Compaction

- Pipelined compaction for the LSM-tree
- The compaction procedure:
    + 1: Read data blocks.
    + 2: Checksum
    + 3: Decompress
    + 4: Merge sort
    + 5: Compress
    + 6: Re-checksum
    + 7: Write to the disk
- Read -> Compute -> Write
    + It is difficult to deivde the stages evenly
    + If data blocks must flows through multiply processors, it will result in low CPU cache preformance. Lst S2~S6 as on stage will be more cpu cache

Compaction Buffer

Coprocessor
- Co-KV: A Collaborative Key-Value Store Using Near-Data Processing to Improve Compaction for the LSM-tree.
- 数据库遇见FPGA: X-DB异构计算如何实现百万TPS?
  