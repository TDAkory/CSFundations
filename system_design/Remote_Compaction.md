# `Remote Compaction`调研

## 为什么需要Compaction

compaction在以LSM-Tree为架构的系统中是非常关键的模块，log append的方式带来了高吞吐的写，内存中的数据到达上限后不断刷盘，数据范围互相交叠的层越来越多，相同key的数据不断积累，引起读性能下降和空间膨胀。因此，compaction机制被引入，通过周期性的后台任务不断的回收旧版本数据和将多层合并为一层的方式来优化读性能和空间问题。

compaction的主要作用是数据的gc和归并排序，是lsm-tree系统正常运转必须要做的操作，但是compaction任务运行期间会带来很大的资源开销，压缩/解压缩、数据拷贝和compare消耗大量cpu，读写数据引起disk I/O。compaction策略约束了lsm-tree的形状，决定哪些文件需要合并、任务的大小和触发的条件，不同的策略对读写放大、空间放大和临时空间的大小有不同的影响，一般系统会支持不同的策略并配有多个调整参数，可根据不同的应用场景选取更合适的方式。

## Ref

* [深入探讨LSM Compaction机制](https://developer.aliyun.com/article/758369)
* [[RocksDB剖析系列] Remote Compaction](https://segmentfault.com/a/1190000041289857)
* [Rocksdb Compaction源码详解（二）：Compaction 完整实现过程 概览](https://blog.csdn.net/Z_Stand/article/details/107592966)
* [ToplingDB的分布式Compact和RocksDB的RemoteCompaction有什么不同](https://docs.pingcode.com/ask/32348.html)
* [关于LSM-tree 的 Remote Compaction调度](https://blog.csdn.net/Z_Stand/article/details/119700257)
* [Compaction management in distributed key-value datastores](https://www.vldb.org/pvldb/vol8/p850-ahmad.pdf)
* [Rockdb compaction](https://github.com/facebook/rocksdb/wiki/Compaction?spm=a2c6h.12873639.article-detail.9.6f5c3f0eVW0Dpi)
* [以加速 compaction 和 scan 为例：谈 GPU 与 LSM-tree 的优化](https://open.oceanbase.com/blog/10900271)