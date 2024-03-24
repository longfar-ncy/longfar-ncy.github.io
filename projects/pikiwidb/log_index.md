# 基于 braft 实现 PikiwiDB 的 Raft 集群

## 基于 log index 与 sequence number 映射的快速重启方案
pikiwidb raft层面之下的状态机实际通过 RocksDB 做存储，RocksDB 是个基于 LSM 树的存储引擎，即便写入成功，也有可能是存在内存中而不是磁盘上，若这个时候服务器宕机，在关闭WAL的前提下，将会丢失内存中的数据。  

如果要保证raft集群的数据一致性，在做快照时必须保证将 Rocksdb 内存中的数据刷到磁盘，也就是每次 `on_snapshot_save` 时都需要 `Flush` 一次；但 braft 每隔一段时间就会调用快照保存接口，也就是每隔一段时间都必须 flush 一次，对性能影响较大。

现在的方案是，在每个节点刚加入集群时做一次全量快照加载，后续就不需要保存快照了，这样就避免了每隔一段时间都要 flush 的性能问题。后续服务器宕机后重启时，通过 braft 日志恢复 rocksdb 内存中丢失的数据和 raft 落后的日志，与原本 raft 协议有所不同的是，raft 从落后的日志开始回放，本项目应该从 rocksdb 丢失内存对应的最早的日志处重放。

### 快速恢复方案
一个宕机的节点重新启动时，本地是存在SST文件的，而丢失的数据分为两部分：
1. raft 层面落后的操作日志（未apply） 
1. rocksdb 内存中没来得及 flush 的数据（已apply）  

对于第一种，raft 协议本身就可以保证同步；对于第二种，想要通过braft的日志找到rocksdb内存中的数据（rocksdb的WAL被关闭），需要本地做一些支持：在重新启动后，需要得到丢失内存数据中，最旧的写操作所对应的日志索引。 

Pikiwidb 中的每个 rocksdb 实例都有9个列族存储数据，不同列族 flush 的进度也不同，因此需要找到最落后的列族中原内存中最旧的操作（最小 sequence number）对应的 log index。内存的数据重启后一定是不能直接找回了，但可退而求其次，从落盘后的最新数据入手，即 flush 进度最慢的列族的已持久化数据中最后的写操作（最大 seqno）对应的 log index，从这个日志的后一个日志开始回放，也可以达到目的。

### 需要做的具体工作：
1. 在运行时维护 applied log index 与 sequence number 的关系：  
   每个 Redis 实例维护一个存放 logidx 与 seqno 映射的链表，在每次异步写成功后，在该链表后添加最新的映射—— applied log index（刚刚解析的binlog 在 braft 中的 idx）和 本次 WriteBatch 之前的 latest seqno + 1。这样就可以在 flush 时通过最大的 seqno 找到不超过该 seqno 的最新日志索引，flush 后该索引以及之前的日志操作都是 rocksdb 已经持久化过的。  
   出于性能考虑，并不会每次都添加新映射，而是有一定周期地更新。
3. Flush 时将 sequence number 和 log index 一同存入 SST 文件：  
   （存seqno只是为了debug，未来可删去）这里通过 rocksdb table property 特性，在每次 Flush 时将本列族的 logidx 与 seqno  的映射持久化到 SST 文件；具体来说，每次 Flush 时会对每个 key 遍历检查，我们可以保存最大的 seqno，最后利用最大的 seqno 在上述维持 seqno 与 logidx 的链表中，找到不大于该 seqno 的最大 seqno 与 logidx 的映射，这个映射中的 logidx 就是本次持久化对应的已知最新的 logidx，然后将该 logidx 写入SST，以供重启时提取。

   > 例如：Log100(Seq200, Seq201) , Log101(Seq202, Seq203)  
   > 当我们 apply Log100 后，就应当在链表后添加 100:200 的映射，apply Log101 后，就应该添加 101:202 的映射  
   > 
   > 当我们 flush 的内存中最新操作为 Seq200或201 时，应当在链表中找到 100:200 这个映射，然后将日志索引 100 写入 SST；当最新的操作为 Seq202或203 时，我们就应当找到 101:202 这个映射，把 101 写入SST。  
   > 
   > 如果周期为2，假设 Log100 添加到链表中，Log101 没有添加到链表中，Log102(Seq204...) 添加到链表中。那么 Seq200-203 都应当找到 100:200 的映射，然后将 100 写入 SST，而不应该找到 102:204。

4. 在每次 Flush 完后，需要清理该链表：  
   通过 rocksdb 提供的 event listener 特性实现，在每次 flush 完毕后触发清理操作，清理掉链表中已经持久化的最新映射之前的映射。
   > 例如：链表中已经有 100:200 102:204   
   > - 当我们 flush 的最大 seqno 为 200-203 时，就不能删除
   > - 当我们 flush 的最大 seqno >= 204 时，就可以删除 100:200
