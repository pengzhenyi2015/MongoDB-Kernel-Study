## 3.5 持久性保证
Raft 使用 log 来进行复制，并写入磁盘来保证数据持久性。随着写请求的持续，log 的大小会持续增加。因此，需要一种清理 log 的机制，并且不影响数据持久性。

### 3.5.1 Raft 原理
#### Snapshot
Raft 论文的 log compaction 章节说明了 raft 的机制：各个节点在特定条件下（时间周期和 log 大小），会将状态机中的已提交的数据做 snapshot 到磁盘，然后将之前的日志清除（raft 论文中说的是committed entries。Etcd采用的是 [appliedIndex](https://github.com/etcd-io/etcd/blob/release-3.4/contrib/raftexample/raft.go#L378C1-L378C1)，根据 raft 的执行机制，appliedIndex及之前的数据都是 committed）。        
以下图中的节点状态为例，在做 snapshot 时一共进行了 7 次操作，包含 7 条日志。其中序号为 1-5 的日志复制到了大多数节点并提交， 6-7 日志暂时未提交。进行 snapshot 后的结果为：   
- Last included index: 5    
- Last included term: 3    
- State machine state: 包含 1-5 的日志操作记录，即 x = 0, y = 9    

做完 snapshot 之后， 1-5 的日志可以清除，保留 6-7 的日志。   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/d389b0ec-4198-4b7c-92cf-1bb90e639754" width=400>
</p>

#### 新增节点
如果副本集中新增了一个空白节点，可以先将 snapshot 传输到空白节点上，然后再将 log 推送过去。也就是常见的全量+增量的方式。

#### 回滚
如果 leader在持久化日志之后，还没有推送到其他节点就异常退出，然后副本集中选出了新 leader 。则挂掉的节点重启后会成为 follower，并且将没有提交的冲突日志回滚掉。   
由于 Snapshot 中都是已提交的数据，因此不需要回滚状态机，只会涉及日志回滚。而日志回滚是很简单的，只需要将冲突的部分切除掉即可。   

### 3.5.2 MongoRaft 原理
MongoRaft 采取了和 Raft 类似的策略，但是具体实现上要更加复杂一些。    
首先是概念上的差异：    
- Checkpoint: 对应了 Raft 中的 snapshot，即某个时刻的全量数据的持久性快照。和 Raft 一样，MongoDB 的 checkpoint也遵循 Copy-On-Write 。
- Oplog：对应了 Raft 中的 log，但是在 MongoDB 中 oplog 也是一个表，并非等同于 journal 和 WAL（Write Ahead Log）的概念。
- Journal（其他系统可能叫 WAL）：持久化日志，存储在特定的文件中，通过 fsync 的方式写入来保证持久性。

Raft 中的 log 概念，对应了 MongoRaft 中的 oplog + journal。   
MongoRaft 在接受写请求时，会更新状态机（库表和索引）、插入 oplog、写 oplog 的 journal。
**用户的库表和索引是不需要写 journal 的，它们可以通过 oplog 保证持久性，而 journal 又保证了 oplog 的持久性。事实上，只有 local 库的部分表需要写 journal， 而local 库中的 oplog 表就是其中之一。具体可以参考 [WiredTigerUtil::useTableLogging](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/storage/wiredtiger/wiredtiger_util.cpp#L567) 的实现。    

另外需要说明的是，MongoDB 通过 Wiredtiger 提交事务时，不负责将 journal 持久化(fsync)到磁盘中。具体可以参考 [wiredtiger_open](https://source.wiredtiger.com/3.2.1/group__wt.html#gacbe8d118f978f5bfc8ccb4c77c9e8813) 和 [commit_transaction](https://source.wiredtiger.com/3.2.1/struct_w_t___s_e_s_s_i_o_n.html#a712226eca5ade5bd123026c624468fa2) 接口的使用说明：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/b68b6b33-eb77-44b2-a08c-0a761f829fa7" width=650>
</p>

Journal 的持久化通过调用另外的 [log_flush](https://source.wiredtiger.com/3.2.1/struct_w_t___s_e_s_s_i_o_n.html#a1843292630960309129dcfe00e1a3817) 接口完成，具体在 MongoDB 内核流程中，有多种触发方式：    
- 周期性触发。后台 WTJournalFlusher 线程每隔 [commitIntervalMs](https://www.mongodb.com/docs/v4.2/reference/configuration-options/#storage.journal.commitIntervalMs) 触发，默认是 100 ms，可配置。   
- 主动触发。包括依赖持久化的内部流程（比如前面介绍的 oplogTruncateAfterPoint），以及指定 { j : true} 的用户请求等。事实上，每次事务执行结束（普通写请求在内部也是事务，保证写表、索引、oplog 的原子性），都会触发守护线程异步刷 journal 并及时更新oplog 可见性。因此，MongoDB 刷  journal 也是比较频繁的。   

将 journal 的写流程和 fsync 持久化流程分离，为用户带来了更多的灵活性，用户可以根据业务需求在 处理延时 和 持久性 2 个方面进行权衡。
MongoDB 在内核实现上通过 lastAppliedOpTime 和 lastDurableOpTime 进行了区分。    

MongoDB 依赖 Wiredtiger 存储引擎接口来生成 checkpoint。在 Wiredtiger 中，1个表的 checkpoint 包含如下信息：   
- 数据块组织信息：包括 3 个跳表分别维护了已分配（allocated list）、可分配（available list）和已删除（discard list）的数据块信息。
- Root page 的位置。
- 文件偏移。用于 checkpoint 恢复。
- Write generation, 主要用于 salvage 流程。    

示意图如下：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/be243e09-0092-42d4-ac12-c0d15ed8a991" width= 700>
</p>

Wiredtiger 在运行时，每个表有一个 “live checkpoint”, 是可读写的。除此之外，任何已持久化的 checkpoint 都是只读的，用户不能直接修改。    
如果当前的 "live checkpoint" 删除了之前  checkpoint 中的数据块，只能将其放入自己的 discard list 进行标记删除，对应的数据块不能被擦除或者马上重用。如果需要分配新的数据块持久化数据，只能从 available list拿， available list 当前空间不足时可以扩展文件加入新的数据块。    
Wiredtiger 采用了命名 checkpoint 的方式，MongoDB 每分钟定期创建的 checkpoint 采用的是默认名字 WiredTigerCheckpoint。    
每次创建时，会将之前的同名  checkpoint 删除。而 checkpoint 的删除会将其 allocated list 和 discard list 合并到下一个checkpoint，并对下一个 checkpoint 中合并后的 list 进行检查。如果有数据块同时出现在 allocated 和 discard list，则将其从对应的 list 中删除，并移动到 live checkpoint 的 available list 中进行重用。     

个人认为 Wiredtiger 的 checkpoint 机制有 2 点非常关键：    
1. Copy-On-Write 机制。在更新和删除已持久化 checkpoint 中的数据块时，不直接对该数据块进行修改，而是通过 discard list 将原数据块标记删除，数据写入新数据块。然后通过checkpoint 删除时的合并策略回收空间，同时也保证了 checkpoint 的只读特性。   
2. 每个 checkpoint 对应了固定大小的文件偏移。这个设计使得基于 checkpoint 的恢复流程非常快：从元数据找到 checkpoint 的长度信息，然后执行文件  truncate 操作即可。   

除了上述概念上的差异，和 Raft 相比，MongoRaft 中的状态机应用和日志生成是在一个原子操作中完成的。Primary 节点在更新状态机的同时生成了 oplog ，然后才复制到其他 secondary 节点。显然 **primary节点生成日志的时刻，这条日志是没有提交到大多数节点的，而此时 primary 节点的状态机已经更新了。**    
作为对比，Raft 先将日志提交到大多数节点，然后才更新 leader 的状态机。    
在 MongoDB 4.0 版本之前，primary 节点做 checkpoint 时会带入这些未提交到大多数节点的数据。一旦出现异常重启导致回滚，会同时涉及日志和状态机的回滚。需要明确的是：   
- **日志回滚是容易的**。只需要计算好需要回滚的区间，然后清除即可。
- **状态机回滚是复杂、耗时的**。我们并不能根据 oplog 反推出之前的状态，比如删除了 1 个表，oplog只会记录删除表的动作，这个表删除之前包含哪些数据并不能根据 oplog 推断出来。对于这种场景，回滚操作需要从最新的同步源节点重新查找数据（[refetch](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/bgsync.cpp#L724)），数据量大的时候非常耗时。

MongoDB 从 4.0 版本开始，引入了 RTT（Recover To Timestamp） 算法，实现了快速物理回滚。其核心是效仿 Raft 引入了 Stable Checkpoint。至此，MongoDB 中主要包含 2 种 Checkpoint：   
- Stable Checkpoint：将提交到大多数节点的时间点为 commit point，stable checkpoint 中只包含状态机中 commit point 及之前版本的数据，但是 oplog 表以及 local 库中其他会写 journal 的表例外。在回滚时，先快速将状态机变为 stable checkpoint 状态（具体来说就是一些文件的 truncate 操作），然后回放 oplog 即可。    
- Unstable Checkpoint: 之前版本的 checkpoint。回滚复杂，可能需要进行 refetch 操作。    

为了实现 Stable Checkpoint，需要依赖 Wiredtiger 存储引擎按照时间戳管理多版本的能力，而且需要感知 commit point 时间戳。为此，MongoDB 节点在每次 commit point 有推进时，会[调用 Wiredtiger 的 set_timestamp 接口](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/storage/wiredtiger/wiredtiger_kv_engine.cpp#L1868)告知当前的时间戳，并且在 Checkpoint 后台线程中[使用 use_timestamp 方式](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/storage/wiredtiger/wiredtiger_kv_engine.cpp#L378)来生成 Stable Checkpoint。   

基于 Stable Checkpoint 的示意图如下： 
  
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/2f4e4b0c-e61e-4de2-a944-fa770ab75f9a" width=800>
</p>

对 primary节点依次写入了 5 条数据，其中 1-4 复制到了 secondary-1 节点， 1-3 复制到了 secondary-2 节点。则 primary 节点的 mcp(majority committed point) = 4，stable = 4。 secondary-1 节点通过心跳和拉日志请求中的 [replsetMetadata](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/rpc/metadata/repl_set_metadata.h#L68) 推进 mcp = 4, stable = 4。而 secondary-2 节点mcp = 4，无奈自己才复制到日志 3，所以 stable = 3 (选取算法参考[ReplicationCoordinatorImpl::_chooseStableOpTimeFromCandidates](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/replication_coordinator_impl.cpp#L3700) 的实现 )。   
如果此时各个节点做 stable checkpoint, 则：   
- Primary 节点会将用户表中 1-4 的数据持久化到 checkpoint，而 oplog 表由于会写 journal，不受 stable optime 的影响，会持久化 1-5。   
- Secondary-1 节点将用户表1-4 和 oplog表1-4 都持久化到 checkpoint。   
- Secondary-2 节点将用户表 1-3 和 oplog 表 1-3 持久化到 checkpoint。   


如果 primary 节点异常退出，则 secondary-2 会成为新主并接收写入，原 primary 节点重启后会进入回滚流程。    
回滚流程的示意图如下：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/938ee5b5-2a70-4b22-8dc6-e1e61c955856" width=500>
</p>

回滚节点的大致执行步骤为：   
1. 确定 stable checkpoint 的时间点： stable optime（[调用](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/storage/wiredtiger/wiredtiger_kv_engine.cpp#L2254)存储引擎的接口获取）。   
2. 对比自己的 oplog 和主节点的 oplog，找到“分叉点”，也就是代码中的 [common point](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/rollback_impl.cpp#L914-L973)，并检查其合法性：common point >= stable optime。  
3. 执行数据库回滚：[调用](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/storage/wiredtiger/wiredtiger_kv_engine.cpp#L2027) Wiredtiger 引擎的 [rollback_to_stable](https://source.wiredtiger.com/3.2.1/struct_w_t___c_o_n_n_e_c_t_i_o_n.html#a93dbc74accb426582b3c5c2f69e04b28) 接口将数据库回滚到 stable checkpoint状态。在此之前，会将[回滚数据写到本地文件](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/rollback_impl.cpp#L480-L488)中，也可以配置 [createRollbackDataFiles](https://www.mongodb.com/docs/v4.2/reference/parameters/#param.createRollbackDataFiles) 参数关闭此功能。   
4. [清理](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/replication_recovery.cpp#L388-L396) common point 后的 oplog。   
5. [回放](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/replication_recovery.cpp#L423-L428) [stable, common point] 之间的 oplog，这部分 oplog 存储在本地。   
6. 结束回滚流程，进行正常的主从同步。  

### 3.5.3 小结
Raft 和 MongoRaft 都采用物理快照+日志保证持久性和 log compaction。但是具体实现上存在以下明显的区别：    
1. 概念上存在区别。Raft 中的 log 对应了 MongoRaft 中的 oplog+journal，其中 oplog 是系统表，journal是日志文件。个人认为，将 oplog 和 journal 分离至少有 2 个好处：   
a). 方便对日志进行检索操作。比如基于 oplog 的查找过滤，changeStream 等高级特性 。   
b). 方便集成各种插件式存储引擎。其中 journal 和存储引擎深度绑定，比如 Wiredtiger 和 RocksDB 的journal 就不相同，满足单机持久化需求，不参与主从复制；而oplog 不和存储引擎绑定，参与主从复制。对 MongoDB 代码来说，不用感知不同存储引擎的 journal 区别。   
2. MongoDB 节点的状态机变更和日志复制是在同一阶段完成的，状态机中包含未提交的数据。但是从 4.0 版本开始，也引入了 stable checkpoint 机制，在不影响现有复制流程的前提下，数据回滚效率大大提升。   
3. MongoDB 将事务提交时写 journal 和 持久化journal 分开控制，内部通过 lastAppliedOpTime 和 lastDurableOpTime 进行区分。用户可以自行在延时和持久性方面做权衡。   
