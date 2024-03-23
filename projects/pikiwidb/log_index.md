# 基于 braft 实现 PikiwiDB 的 Raft 集群

## 基于 log index 与 sequence number 的快速启动方案
pikiwidb raft层面之下的状态机实际通过 RocksDB 做存储，RocksDB 是个基于 LSM 树的存储引擎，即便写入成功，也有可能是存在内存中而不是磁盘上，若这个时候服务器宕机，在关闭WAL的前提下，将会丢失内存中的数据。如果要保证raft集群的数据一致性，在做快照时必须保证将 Rocksdb 内存中的数据刷到磁盘，也就是每次 `on_snapshot_save` 时都需要 `Flush` 一次；但 braft 每隔一段时间就会调用快照保存接口，也就是每隔一段时间都必须 flush 一次，对性能影响较大。

现在的方案是，在每个节点刚加入集群时做一次全量快照加载，后续就不需要快照了。后续服务器宕机后重启采用快速恢复的方式完成同步。

### 快速恢复
一个宕机的节点重新启动时，本地是存在SST文件的，而丢失的数据分为两部分：1. raft 落后的操作日志 2. rocksdb 内存中没来得及 flush 的数据。  
对于第一种，raft协议本身就可以保证，对于第二种，想要通过braft的日志（rocksdb的WAL被关闭）需要本地做一些支持：在重新启动后，需要得到丢失内存数据种，最旧的写操作所对应的日志索引。  
Pikiwidb 中的每个 rocksdb 实例都有9个列族存储数据，不同列族 flush 的进度也不同，因此需要找到最落后的列族中原本内存当中最旧的操作，即最小 sequence number 写操作对应的 log index。内存的数据重启后一定是不能直接找回了，但可退而求其次，从落盘后的最新数据入手，即 flush 进度最慢的列族的已持久化数据中最后的写操作，即最大的 sequence number 的写操作，找到这个对应的 log index，从这个日志的后一个日志开始回放，也可以达到目的。
需要做的工作具体有：
1. 在运行时维护 seqno 与 applied log index 的关系：  
   每个 Redis 实例维护一个存放 seqno 与 logidx 映射的链表，在每次异步写成功后，在该链表后添加最新的映射—— applied log index （刚刚解析的binlog） 和 latest seqno （刚写成功的batch中的最后一个操作对应的seqno）。出于性能考虑，并不会每次都更新，而是有一定周期地更新。  
   在每次 Flush 完后，需要清理该链表。
1. Flush 时将 sequence number 和 log index 一同存入 SST 文件：  
   存 seqno 的目的是在重启后，对比不同列族之间的 flush 进度，最小 seqno 的对应最落后的列族，需要按照该列族的 log index 回放。这里通过 rocksdb table property 特性，在每次 Flush 时将本列族的 logidx 与 seqno  的映射持久化到 SST 文件，具体来说，每次 Flush 时会对每个 key 遍历检查，我们可以保存最大的 seqno，最后利用最大的 seqno 在上述维持 seqno 与 logidx 的容器中，找到不大于该 seqno 的最大 seqno 与 logidx 的映射，然后将这对映射写入SST。