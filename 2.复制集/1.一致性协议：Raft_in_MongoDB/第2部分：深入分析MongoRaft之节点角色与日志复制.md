# 3. 深入分析 MongoRaft
## 3.1 节点角色
Raft 论文中描述了 3 种节点角色：   
- Leader： 主节点，接收读写请求，承担日志复制，请求提交，心跳探活等任务。
- Follower：从节点，接收日志复制和应用，作为一个完整的数据副本。  
- Candidate：Follower长时间没有感知到主节点时，可以变为 Candidate 并发起选举。   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/33003281-72a3-44b5-8b3e-e826c17490e1" width=400>
</p>

此外，论文中也提到了 Follower 节点可以设置 non-voting 属性，不计入大多数提交时的副本数（Learner）。一般在新加入节点时，可以使用这个设置避免数据提交卡顿。  

MongoDB 中包含的节点角色也有主从状态。 其中主节点（Primary）支持读写，从节点（Secondary）支持读（这是和 Raft 不同之处）。另外从节点又[细分](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/member_state.h#L58-L70)为：  
- Secondary：正常状态下的从节点，支持用户读。  
- Rollback：回滚状态。常见的场景是主节点异常重启后变为从节点，需要将没有复制到其他节点的日志回滚掉。  
- Recovering：正在恢复的状态。节点启动后会短暂处于该状态，然后变成 Secondary。如果长期处于该状态，对应的场景是该节点的日志太旧，无法找到合适的节点同步日志，此时一般需要清空数据后重新做全量同步。  
- Startup：刚启动时的状态。  
- Startup2：初始化全量同步的状态。对应的场景是新加入一个空节点（或者手动将某个节点的数据清空），该节点需要通过 Initial Sync 流程全量同步数据。  
- Arbiter：投票节点，不存储数据。  
此外，如果节点异常，会显示 Unknown、Down、Removed(Other) 状态。

另外 MongoDB 中的每个节点可以配置多种属性：   
- Votes：是否有投票权。所谓的“多数派提交”和“多数派选举”中的“多数”，指的就是包含有 votes 属性的节点。一个 MongoDB 副本集中最多只有7 个包含 votes 属性的节点。通过将 votes 属性设置为 0 ，可以加更多的从节点，因此这个属性一般会用在各大云厂商的“只读实例”产品中。  
- Priority：节点被选为主的优先级。priority 设置的数值越高，越会被选举为主。通过将 priority 设置为0，可以避免主节点切换到某些节点。  
- Hidden：是否将从节点隐藏起来不对外提供服务。设置为隐藏后，客户端驱动无法通过 isMaster/Hello 探测命令感知这个节点，因此无法向其发读请求，但是通过 rs.conf() 和 rs.Status() 命令还是能看到其配置。Hidden 节点一般作为备份节点。由于主节点不能被隐藏，因此官方规定 hidden=true 的节点需要 priority=0。  
- BuildIndexes：从节点是否建索引。如果设置为 false， 则不会同步创建索引。理论上可以节省索引占用的存储空间以及维护索引的 CPU 和内存消耗。但是如果从节点被选举为主，此时由于索引缺失可能会导致一些问题，因此官方规定 buildIndexes=false 的节点需要 priority=0，而且不能对已经加入副本集的节点进行动态变更。  
- Tags：为节点打标签，一般用户从节点的请求分流。比如副本集中有 4 个节点，其中 2 个节点硬件配置高，可用于在线服务，则设置这 2 个节点的 tag 为 {"usage":"online"}；另外 2 个节点硬件配置较低，只用户离线特征分析，则设置这 2 个节点的 tag 为 {"usage":"offline"}。用户程序可以根据自身请求特点设置readPreference 说明希望将请求发往 "online" 还是 "offline" 节点。  
- ArbiterOnly：轻量级从节点，不同步和存储数据，只负责选主投票。一般用于解决偶数副本数的场景。比如副本集中只有 2 个节点，必须全部存活才能选出主。加入一个 arbiter 节点之后，副本个数变为 3，可以容忍一个节点失效。  
- SlaveDelay：强行设置某个从节点距离主节点的主从延迟。比如设置 60 秒，则该从节点的最新日志同步时间比主节点要延迟 60 秒或以上。一般用于业务数据快速回滚（比如误删）和测试场景。  

通过状态区分和灵活的配置，MongoDB 能够更好地支持多样化的业务场景。

## 3.2 日志复制
### 3.2.1 Raft 原理
Raft 采用推日志模型，由 leader 节点将日志并发推送到 follower 节点，并应用状态机。整体流程如下图所示：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/6796bf31-33ab-45c2-af62-b194d8cc9ba6" width=400>
</p>
   
（1）Client 往 leader 节点发写请求。   
（2）Leader 节点在本地生成日志，并通过 RPC 将日志并发推送到其他 follower 节点。   
（3）确保当前大多数节点（包括自己）已经完成了日志复制，则将日志应用到自身的状态机（存储引擎）中。   
（4）给 Client 返回提交成功。   
注意 follower 节点的日志应用是异步的。在上述流程结束后，leader 节点会推进自身的 commitIndex，在下一次日志推送以及定期的心跳请求中会携带上 commitIndex 发送给 follower，使得 follower 也将这部分日志应用到状态机。   

日志中包含了 index, term，执行命令等信息，如下所示：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/a8b3e351-9e92-4240-87d6-4ed335759409" width=400>
</p>

每个节点的日志顺序要保证完全一致且不能由空洞，不支持乱序提交。   

### 3.2.2 MongoRaft 原理
MongoDB 使用 oplog 完成主从节点之间的数据同步， oplog 采用递增的混合逻辑时钟作为“序号”（类似 Raft 中的 logIndex），并且会携带 term、库表名、操作类型和命令信息。   
以 4.2.24 内核版本为例，往 db1.coll1 表中插入一条文档 {a:1} 后生成的 oplog 如下：   
>{ "ts" : Timestamp(1698199568, 1), "t" : NumberLong(37), "h" : NumberLong(0), "v" : 2, "op" : "i", "ns" : "db1.coll1", "ui" : UUID("a3e5d8ac-f4b3-4a40-88c9-8b0b6c47b4c4"), "wall" : ISODate("2023-10-25T02:06:08.268Z"), "o" : { "_id" : ObjectId("6538780f07311975afc52751"), "a" : 1 } }

每条 Oplog 中都包含Timestamp 用于定序。Timestamp 由 int32 的秒级时间戳和 int32 的秒内计数器 2 部分组成，合起来可以用 int64 表示。该 Timestamp 由[主节点执行写请求时确定](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/ops/write_ops_exec.cpp#L294-L314)，从节点不作任何修改。
#### 日志同步模型
MongoRaft 中的日志同步采用了从节点拉日志的模型（Pull-Based)，和 Raft 的主节点推日志模型有着显著区别。    
以4.2.24内核版本为例，MongoDB节点启动后会加载配置，并最终调用[ReplicationCoordinatorExternalStateImpl::startSteadyStateReplication()](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/replication_coordinator_external_state_impl.cpp#L207) 启动复制流程，其中会涉及如下线程：   
1. **BackgroundSync**：一直向源节点发起 find+getmore 请求，并将获取的 oplog 数组存放到 **oplogBuffer** （本质上是一个 OplogBufferBlockingQueue, 即 std::queue\<BSONObj\>，最大 [256MB](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/oplog_buffer_blocking_queue.cpp#L40)）。   
2. **ReplBatcher**(参考 SyncTail::OpQueueBatcher) ：由 OplogApplier 启动的后台线程，这个线程不断消费 oplogBuffer 中的 oplog 日志，并传递到 **OpQueue**（本质上是一个 std::vector，也叫做一个 oplogBatch，最多 [5000 条](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/repl_server_parameters.idl#L246)和 [100MB](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/repl_server_parameters.idl#L258)）。在消费 oplogBuffer 的过程中，会对每一条 oplog 进行版本检查，并通过类型设置栅栏。比如对一个表进行了 ”文档插入“后又进行了“删表”，则“删表”操作不能和前面的“文档插入”放在一个 OpQueue 中。也就是要保证 OpQueue 是可以并发回放的。   
3. **OplogApplier**：管理  syncTail 对象，并在启动时调用 syncTail::oplogApplication 方法一直获取 **OpQueue** 中的 oplog 数组进行回放（**MultiApply**），回放流程首先按照{表，文档_id} 将 oplog 数组再哈希到多个数组中（参考 [fillWriterVectors](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L1163)），然后提交给 writerPool 线程池进行并发回放([_applyOps](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L1308))。   
4. **WriterPool 线程池**：真正执行 oplog 回放的线程，默认的最大线程个数为min([16](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/repl_server_parameters.idl#L220) ，[机器的核数*2](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/oplog_applier.cpp#L61) ），线程名字为 repl-writer-worker-*.   
5. **ApplyBatchFinalizerForJournal**:  oplogApplier 创建出来的守护线程，主要的作用是在应用完一批 oplog 并更新了 lastAppliedOpTime 之后，通知 syncSourceFeedback 主动给上游节点上报。另外还会等待这批 oplog 的请求刷完 journal 更新了 lastDurableOpTime 之后，通知 syncSourceFeedback 给上游节点上报。   
6. **SyncSourceFeedback**：给上游节点上报同步进度的守护线程。一般由 oplog 回放流程结束后通过条件变量触发，如果长时间没有oplog 应用，也会在 5秒后主动上报一次（起到探活的作用）。   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/ef2021bb-5c0a-4796-b69d-124d179b3d5e" width=1000>
</p>

从架构图中，可以看到 MongoDB 复制模型具有以下明显特征和优化点：   
- **主节点先应用状态机（写数据）并同时生成 oplog**，然后再将日志复制到其他节点。对于原生 Raft 来说，是先复制日志到其他节点，然后再提交。   
- 从节点**拉日志**。而原生 Raft 是主节点推日志到从节点，从节点被动接收。拉日志模型带来一个明显的好处就是可以形成链式复制结构，即从节点不一定非要到主节点上复制，使得复制链路更加灵活，容错性更高。另外对于一些跨地域的复制场景，链式复制能够明显降低网络带宽成本。比如 5 个节点的副本集，S1 是主节点， S1 和 S2 在北京，S3/S4/S5 在广州，则可能的复制链路为 S1->S2->S3->S4/S5，即 S4 和 S5 不需要跨地域复制。   
- 从节点**异步反馈复制进度**。而原生 Raft 是主节点在推送日志的 RPC 响应中感知日志的复制进度。而在 MongoDB 中，由于链式复制的存在，syncSourceFeedback 流程除了上报自己的进度之外，还会上报其他已知节点的进度。比如 3  节点的副本集，复制链路是 S1->S2->S3, S3 给 S2 上报进度，S2 给 S1 上报进度时也会带上 S3 的进度信息。如果不这样做，S1 只能通过心跳感知 S3 的复制进度，效率会大打折扣。另外，即使S2 一段时间没有日志推进，但是 S3 有日志推进，S3 给 S2 上报进度后，也会触发  S2 给 S1 进行一次上报。syncSourceFeedback 还有另外一个作用就是探活，比如 S1 和 S3 之间的网络有问题，心跳不通，但是 S1 能够通过 S2 的主动上报来感知 S3 的存活状态。   
- 使用**多线程**进行**流程解耦**。日志的拉取、回放和进度反馈都是不同的线程完成的，并通过内存队列和条件变量等数据结构进行多线程通信。解耦后各个子流程之间不会直接相互影响，尽可能提升拉日志、回放核心流程的效率。   
- **并发乱序回放oplog**。原生 Raft 规定 Follower 节点严格按照日志顺序进行回放，但是在 MongoDB 中日志是严格按照日志时间分 Batch进行拉取的，但是在一个 Batch 内部的回放是哈希打散之后分发给线程池并发完成。并发回放是必须的，因为 MongoDB 主节点提供并发写入，如果从节点只能严格按照时间顺序串行回放，性能肯定跟不上，导致主从延迟越来越大。但是并发又会带来一些问题，比如是否所有的操作都支持并发？按照什么规则进行拆分能保证正确性？并发操作是否会和唯一索引等约束条件产生冲突？并发带来的乱序是否会导致用户读到不一致的数据，MongoDB 是如何解决的呢？   
- **更灵活的复制状态**。原生 Raft 中 Follower 节点在日志持久化后才认为复制成功，但是在 MongoDB 中使用 lastAppliedOpTime 和 lastDurableOpTime 进行了区分。其中 lastAppliedOpTime 表示日志已经应用并在本地生成了 oplog，但是还没有异步写 jounal(WAL)，而 lastDurableOpTime 表示已经写  journal 的持久化状态。用户可以自定义写入级别，主节点将根据用户定义的写入级别依据 lastAppliedOpTime 或者 lastDurableOpTime 来更新提交状态。总结来说，MongoDB 将选择权交给用户，可以在请求延迟和数据持久性上做权衡。   

>注：在 [8.0 内核](https://www.mongodb.com/zh-cn/docs/manual/release-notes/8.0/#majority-write-concern) 对 majority 提交流程进行了一些变更：从节点在复制完 oplog 即反馈进度，而不是等待本地应用之后再反馈。这在一定程度上减少了 majority 提交的延迟。

#### 同步源节点的选择
如果用户关闭了副本集的链式复制功能，则所有节点只能到主节点同步。   
如果用户打开了链式复制功能（默认打开），则：   
- （默认）**自动选择**同步源节点。从节点根据自己的拓扑信息，选择一个 opTime 比自己新的，而且网络距离最近的节点作为同步源。   
- **手动选择**同步源节点。用户可以通过 replSetSyncFrom 命令，手动给一个从节点指定同步源。通过手动指定同步源，用户可以明确的控制整个复制链路。   

链式复制举例如下，B 从 A 同步，C和 D 从 B 同步。复制链路包括 拉oplog 流程（粗实线）和进度反馈流程（细实线），心跳流程是虚线部分。因此，最终副本集中各个节点的通信是**复制链路+心跳链路相结合**的形式。复制链路通信更频繁，是节点状态同步的核心路径（oplogFetch 和 updatePosition 命令的元数据中会携带节点的状态信息）而且由于链式的存在，能够很好容忍局部网络故障。心跳通信频率更低（默认 2 秒 1 次），能够作为节点状态同步非常好的补充，并且为网络故障时的同步源切换流程提供参考。两者相辅相成。

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/95fb398d-7558-48bc-a3bc-4190c3abda62" width=400>
</p>

#### 并发回放如何保证正确性
**如何并发：按照什么规则进行 Hash?**   
MongoDB 内部采用了 2 级 Hash 的策略，参考[SyncTail::_fillWriterVectors](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L1163):   
1. 首先按照表（db.collection）进行 Hash，因为不同表之间的修改没有直接关联。   
2. 其次再按照文档的 _id 进行 Hash，因为不同文档之间的修改也没有直接关联。   

同 1 个文档（_id 相同）的 oplog 会Hash 到一个桶中，并保持原始的时间顺序。   
当然也有一些特殊情况不会进行 2 级 hash。比如固定集合（capped collection）要满足“先插入先删除”的特性，对不同文档的插入顺序有严格要求，因此不能按 _id Hash 后乱序插入，也[不能进行 groupInsert](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/applier_helpers.cpp#L91) 的回放优化。

**DDL 操作如何处理？**   
Oplog 中除了常见的 CRUD 操作之外，还有例如删库删表等 DDL 操作。这些 DDL 操作无法和 CRUD 操作并行回放，例如对表 A 先插入一条数据，然后再删表。如果将这 2 条 oplog 并发回放的话，可能会出现先删表再插入数据，显然是不合适的。   
从节点的 ReplBatcher 线程调用 [OplogApplier::getNextApplierBatch](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/oplog_applier.cpp#L244) 方法将 oplogBuffer 队列中的 oplog 传递到 opQueue 中，opQueue 可以认为是一个批量并发回放的 batch。getNextApplierBatch 方法会甄别每条 oplog 的类型，如果是 DDL 操作，会将这些操作单独作为一个 batch 回放，具体可以参考 [mustProcessStandalone](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/oplog_applier.cpp#L211) 的处理逻辑。

**如何并发回放事务？**   
对于 4.0 内核版本的副本集事务，是比较好处理的。事务在提交时，会将所有操作通过一个数组打包好，并放在一条 oplog 中。这种方式虽然对事务大小有 [16MB 限制](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/#oplog-size-limit)，但是 secondary 节点在回放时非常好处理。回放时只需要将事务 oplog 中的数组解开，得到具体的写操作后进行 hash 就能并发回放了。本质上和普通的写操作并无太大区别。   

但是，从 4.2 内核版本开始，出现了很大的变化。   
4.2 版本对副本集事务进行了扩展，并且支持了基于 2 阶段提交的分布式事务。对应到 oplog 的实现方式上，出现了 2 个重大区别：   
1. 事务的 oplog 不再局限于 1 条，可能存在**多条**。因此，4.2 版本的事务大小也不再受限于 BSON 的 16MB 限制。   
2. 分布式事务包含 prepare 和 commit/abort **2 个阶段**，都有各自的 oplog。   

Oplog 的变化给回放流程带来了新的挑战，主要有：
1. 如何保证**原子性**。假如 1 个事务有 4 条 oplog，secondary 节点在第 1 次拉取时拉到了 2 条 oplog（每次拉 oplog 的大小也是有限制的，可能刚好拉到前 2 条 oplog 时就达到了大小限制，后面 2 条只能下一次再拉），那么前面 2 条 oplog 能被回放吗，只回放事务的部分操作，岂不是破坏了事务的原子性？另外，第 2 次将事务的后 2 条 oplog 也拉过来了，如何找到这个事务前 2 条 oplog 呢？   
2. 如何进行 **2 阶段事务**的并发。为了回放的正确性，要保证同一事务的 prepare 在 commit/abort 之前执行。Prepare 中的操作也不能直接解析成普通的写操作进行回放，因为此时还不确定事务能否成功提交。另外，Commit/abort 操作也不能和普通的写操作并发回放。   

我们通过一个副本集事务的 oplog 回放流程，来探讨如何解决回放的原子性问题。   
假设一个很大的副本集事务依次生成了 4 条 oplog，时间戳分别是 t1、t2、t3、t4。Secondary 节点第 1 批拉取到了 t1 和t2, 第 2 批拉取到了t3 和 t4。如下图所示：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/1bef164d-2d70-4baf-b27c-10e6231db278" width=800>
</p>

先看第 1 批的回放流程：   
1. Secondary节点先将拉取到的所有 oplog 都写到自己的 oplog 表中（异步执行，**包括 t1 和 t2**）。   
2. 执行 oplog 的 hash 流程：   
2.1. 处理到 t1, 发现是事务的一部分 oplog，即 partial = true。先不回放，而是将这条 oplog 放在[内存 cache](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L1199-L1212) 中。这里是为了方便后续回溯，而 第 1 步的oplog 写表异步任务可能还没完成，因此必须引入内存 cache。   
2.2. 处理到 t2，还是 partial = true, 也放到内存 cache 中。   
3. 等待第 1 步写 oplog 表的异步任务执行完，然后将第 1 批 oplog 全部回放完。在此过程中，t1 和 t2 中的操作并没有被执行，而且内存 cache 也被清空。    

接着再回放第 2 批：   
1. Secondary节点先将拉取到的所有 oplog 都写到自己的 oplog 表中（异步执行）。   
2. 执行 oplog 的 hash 流程：     
2.1. 处理到 t3，发现 partial = true, 放到内存 cache中。   
2.2. 处理到 t4，发现是事务的最后一条 oplog，要准备真正执行回放了。则通过 sessionId 找 cache 中的 t3，然后通过 t3->[prevOpTime](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/transaction_oplog_application.cpp#L269)字段找到上一条 oplog 的时间戳是 t2, 并使用这个时间戳到[本地 oplog 表](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/transaction_history_iterator.cpp#L59-L61)中把 t2 的完整 oplog 找到。接着用同样的方法找 t2->prevOpTime，得到t1 的完整 oplog。最后发现 t1->prevOpTime = null，此时 oplog 的回溯流程结束。得到事务的完整 oplog： t1、t2、t3、t4。   
2.3. 在回溯 oplog 的过程中，也会将上述 4 条 oplog 中包含的操作都解析出来，放在一个数组中并且[排好序](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/transaction_oplog_application.cpp#L290-L305)。然后将这个数组 hash 到多个并发线程中。   
3. 等待第 1 步写 oplog 表的异步任务执行完，然后执行这批 oplog 的操作。由于是在一个 batch 中回放完成的，因此能够保证事务回放的原子性。   

我们再通过一个例子，看看如何处理 2 阶段事务。   
假设事务包含 5 条 oplog，前 4 条是 prepare 阶段产生的，最后 1 条是 commit。如下图所示：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/3b14c204-ca2e-4aef-ad22-ff5a63d1771a" width=800>
</p>

第 1 阶段的回放流程和上述副本集事务例子的第 1 阶段相同，不再赘述。   
第2 阶段流程和副本集事务有些区别：   
1. 处理到 t3，由于 partial = true, 先放到 cache 中。   
2. 由于 t4 是 prepare = true 的 oplog， 会独占一个 oplog batch，不和其他操作共享。处理到 t4 时，会从 oplog 表中把  t3、t2、t1 都回溯出来，然后根据这些 oplog 中的操作执行 [prepareTransaction](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/transaction_oplog_application.cpp#L458)。注意这里是 prepareTransaction，而不是hash成普通的写操作执行，因为此时还不确定这些事务能否提交。   
3. 处理到 t5，这是 1 条 commit  oplog，会独占一个 oplog batch。Secondary 节点根据  commit oplog 中携带的 txnId sessionId 信息，找到之前 prepare 的上下文，然后[执行 commit](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/transaction_oplog_application.cpp#L163)。   


**回放过程中的不一致数据能读到么？**   
假设在主节点上依次插入 3 条数据 {x:1} -> {y:2} -> {z:3}，在从节点上并发回放的先后顺序可能是 {z:3} -> {y:2} -> {x:1}。如果用户在从节点上读取数据，显然是无法满足一致性要求的。   
因此，MongoDB 通过一些机制避免了用户读取正在回放中的数据：   
- 在 3.6 及之前的版本，用户对从节点的读取操作需要先获得 PBWM (Parallel Batch Writer Mode)锁。由于 oplog 的批量回放也需要先获取 PBWM 锁，因此能够互斥。但是这种方式效率很低，导致用户到从节点的请求经常卡顿，而且也会影响从节点的同步速度。   
- 在 4.0 及之后的版本，用户到从节点的读取操作变为直接读一致性快照（不依赖 PBWM锁）。所谓快照，就是读取不超过 lastAppliedOpTime 之后修改的数据版本。而每次 oplog 批量回放完成之后，都[会主动更新 lastAppliedOpTime](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L865)。**这样保证了用户在从节点上能够尽快获取尽可能新的一致性数据。**   

示意图如下：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/409df3b3-3b55-4800-b6dd-b8e4823d9234" width=800>
</p>


**乱序回放是否会打乱从节点的  oplog 顺序，导致主节点和从节点的 oplog 顺序不一致？**     
假设下图的场景：  
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/177d4c62-66d1-492f-93c6-f0c513dbed03" width=600>
</p>
   
(a) S1 是主节点，最新写入日志 3、4、5。S2 是从 S1 同步，日志进度为 1、2。S3 从 S2 同步，日志进度为 1。   
(b) S2 从 S1 拉取日志 3、4、5，然后批量乱序回放，本地形成日志 5、4、3 。   
(c) S3 从 S2 拉取日志 2、5、4，然后批量乱序回放，本地形成日志 4、2、5。批量回放结束后，4、2、5 的修改变为对用户可见，**此时用户会读取到不包含 3 的不一致数据！**  

上述假设场景的根本问题是日志回放顺序和生成顺序绑定，导致的日志乱序。MongoDB 通过以下机制保证了乱序回放情况下的日志严格有序：
- **每条 oplog 都会携带时间戳**（int64），而且这个时间戳只能是主节点生成，从节点只能遵守不能修改。这样保证 oplog 在副本集中每个节点的时间顺序都是一样的。   
- **Oplog 在 MongoDB 中严格按照时间戳存储和查询**，并通过可见性判断机制（只有 oplog batch 回放完，[才对外可见](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L876)）来保证从节点拉取的 oplog 不会出现空洞。   
- 另外，**从节点上写 oplog 和回放 oplog 的流程是分开的**，不像主节点将写用户库表和oplog放在一起提交。在从节点上，线程池[先并发写 oplog](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L1369)，再[并发回放 oplog](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/repl/sync_tail.cpp#L1410)。   

上述机制保证了从节点上的日志回放顺序不影响日志生成顺序，副本集中每个节点的日志顺序完全相同。所以，MongoDB 中真正的复制流程如下所示：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/9d257a15-5d08-405c-89ac-9787a15abb22" width=800>
</p>

(a) S1 是主节点，最新写入日志 3、4、5。S2 是从 S1 同步，日志进度为 1、2。S3 从 S2 同步，日志进度为 1。    
(b) S2 从 S1 拉取日志 3、4、5，虽然乱序回放时数据写入的先后顺序时 5->4->3， 但是日志的顺序仍然保持 3->4->5，和主节点一样。    
(c) S3 从 S2 拉取日志 2，3，4，乱序回放插入数据 4->2->3。回放完之后用户可以读取 S3 的数据为 1，2，3，4，虽然数据不是最新的，但是不会读到有空洞的非一致性数据。    


**如何处理乱序打破唯一性约束的问题？**   
乱序回放可能会破坏唯一索引的约束。以下图为例，在某个表上按照 "a" 字段创建了唯一索引。在主节点上对这表依次进行了 插入 {_id:1, a:1}、删除{_id:1, a:1} 和 插入{_id:2, a:1} 的操作。随后生成的 oplog 在从节点上按照 _id 哈希之后并发回放，则有可能 {_id:1, a:1} 和 {_id:2, a:1} 的插入操作先执行，此时就破坏了唯一索引的约束，导致错误。    
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/f023bf96-0ec6-4362-bcba-c12472792cde" width=600>
</p>
   
为了解决索引唯一性的问题，MongoDB 在从节点 oplog 回放阶段**临时放开了唯一性约束**，具体可以参考 [IndexCatalogImpl::_indexFilteredRecords](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/catalog/index_catalog_impl.cpp#L1334)、[IndexCatalogImpl::_updateRecord](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/catalog/index_catalog_impl.cpp#L1395) 和[IndexCatalogImpl::_unindexKeys](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/catalog/index_catalog_impl.cpp#L1441) 中如何调用 [prepareInsertDeleteOptions](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/catalog/index_catalog_impl.cpp#L1649) 生成操作参数。在 prepareInsertDeleteOptions 的逻辑中，对于从节点的索引操作放开了约束，即使是唯一索引也允许 duplicateKey 的存在，即设置 dupsAllowed = true。这个设置会使得操作存储引擎时[跳过索引的唯一性检查](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/storage/wiredtiger/wiredtiger_index.cpp#L1519-L1549)。   
跳过唯一性检查，意味着 oplog 回放阶段的唯一索引可能存在多个相同 key 的情况，但是在回放完成后能仍然能满足唯一性。基于前面的分析，在 **oplog 回放期间这部分不一致数据是用户读不到的**，因此不会对正确性产生影响。

>备注：  
业界广泛使用的 DTS(数据传输服务，将一个 MongoDB 集群的数据同步到另一个 MongoDB 集群) 也会遇到索引的唯一性约束问题。  
一些云产商为了实现方便，在检测到表有唯一索引后，会将这个表的数据同步变为串行（不再按_id 并发），单表的同步性能会下降。  
也有一些云产商采用了一些冲突检测机制尽量保留并发，比如 MongoShake 在 oplog 日志中增加了 uk 字段来说明表有哪些唯一索引以及本次改动的 key。如果有 2 条 oplog 改动了相同的 key，则不能放在一个 batch 中回放。如果改的 key 不同，则还是能放在同一个 batch 中并发回放。  
总体来说，MongoDB 内核中主从同步的冲突处理策略是最易理解，也是性能最强的，当然这也得益于内核代码本身就能进行更加精细化的控制。  

#### 日志同步性能探究
**GetMore 拉日志的困境**       
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/59ffbd58-6f8e-4936-bb97-e6512346a7e4" width=300>
</p>
  
虽然MongoDB 采用了将**日志拉取和回放操作分离**的设计，尽可能提升拉取日志的速度，但是本质上还是一个单连接模型，网络延迟仍然不能忽视。   
以上图为例，假设主节点和从节点之间的网络距离是 50 ms（跨地域部署的场景，图中虚线框所示），硬件配置无限高使得请求和响应的处理几乎不耗时，已知一批 oplog 的最大载荷是 16MB，则可以推算该场景下的极限同步速度：   
a) 0->50ms 时刻, 拉日志请求（find/getmore）由从节点发给主节点。   
b) 主节点准备好日志，封装好 response，极限情况下足够快，不耗时。   
c) 50->100ms 时刻, Response 由主节点发给从节点，极限情况下带宽无限。   
d) 从节点处理响应，将 oplog 放到内存队列，然后开始拉下一批，极限情况下不耗时。   
这样主从的极限同步速度为 16MB/100ms = 160MB/s。如果主节点上有大量写入，生成oplog日志的速度超过了 160MB/s，则主从延迟只会越拉越大。   

**MongoDB 的解决方案**
1. FlowControl 机制，主动限流   
从 4.2 内核版本，开始引入了 [Flow Control 机制](https://www.mongodb.com/docs/v4.4/tutorial/troubleshoot-replica-sets/#flow-control)，主节点在主从延迟较大时，会对写入请求进行限流，保证副本集的大多数提交时间点（majority committed point）不会超过设置的阈值（默认 10 秒），具体实现为主节点根据延迟情况计算每秒能够发出多少 ticket供写入，如果 ticket 消耗完毕，则写入请求会阻塞到下一秒有空余的 ticket 为止。具体的计算规则可以参考 [Github Wiki](https://github.com/mongodb/mongo/blob/r5.0.16/src/mongo/db/catalog/README.md#flow-control)，这里不作展开。   

2. 主节点流式推送日志，推拉结合   
从 4.4 版本开始，MongoDB 默认支持了 [Streaming Replication](https://www.mongodb.com/docs/v4.4/core/replica-set-sync/#streaming-replication), 如果节点 B 到节点 A 同步数据，会发起带 exhaust cursor 属性的拉日志请求，然后接收节点A 的日志推送(推拉结合)。后续节点 A 产生新的 oplog 日志后，可以直接推送给节点B，这样可以减少网络 round-trip 带来的延迟。提升了拉 oplog 日志的速度后，在以下方面会带来显著提升：   
    - 提升到 Secondary 节点读数据的新鲜度。   
    - 对于 {w:1} 只写主节点就确认成功的请求，能够降低主节点 crash 时数据回滚的风险。   
    - 对于{w:"majority"} 写多数节点的请求，能够有效降低处理延迟。   

以节点 A -> 节点 B 的同步为例。在 MongoDB 4.4 版本中，为了实现 streaming 复制流程，节点B的 OplogFetcher在通过 find/getmore 请求到节点A 上拉日志时，会[携带 exhaust 标记](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/db/repl/oplog_fetcher.cpp#L638-L656)。后续处理流程如下：   
- 节点A 在返回一批oplog后，**如果发现处理的是 exhuast 请求，则[自动生成一个新请求](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/transport/service_state_machine.cpp#L511-L517)（相当于在同一个连接上，自己给自己发了一个 getmore 命令）。然后将 ServiceStateMachine 的[状态置为 State::Process](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/transport/service_state_machine.cpp#L440) 方便后续直接处理这个请求**，非 exhaust 场景下会[置为 State::Source](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/transport/service_state_machine.cpp#L446) 接收节点 B 发下一个 getmore 命令。   
- 节点B  不断通过 [OplogFetcher::_getNextBatch()](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/db/repl/oplog_fetcher.cpp#L663) 获取新日志，底层调用 [DBClientCursor::requestMore()](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/client/dbclient_cursor.cpp#L261) **在判断请求是 exhaust 类型时，会直接[读取连接中的buffer数据](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/client/dbclient_cursor.cpp#L305)**，如果不是 exhuast 类型则[重新发起 getmore 命令](https://github.com/mongodb/mongo/blob/r4.4.24/src/mongo/client/dbclient_cursor.cpp#L277-L284)去源节点拉日志。   

Streaming 模式的示意图如下：  
<p align="center">
<img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/940da4b6-1406-43cd-b614-bca819c55ed1" width=700>
</p>

**业界横向对比1：ParallelRaft  的并行复制**    
不论是MongoRaft 的拉日志模型还是推拉结合模型，都是使用了单连接有序传输日志。日志的提交方式都是顺序提交，且不能有空洞。同理，原生 Raft 协议由于其顺序提交的特性，也不能很好地驾驭多连接日志同步的场景。   
[PolarFS](https://www.vldb.org/pvldb/vol11/p1849-cao.pdf) 的 ParallelRaft 算法提供了一个新思路：通过多个连接传输日志，并且支持乱序确认和乱序提交。为了解决乱序提交带来的冲突检测，每条日志上都有 look-behind buffer 记录前 N 条日志修改的 LBA （逻辑地址，可以类比为 MongoDB 中的 _id），在日志存在少量空洞的情况下也能完成冲突检测并顺利提交。

**业界横向对比2：TiKV 的 MultiRaft**    
既然单个 raft group 存在的性能问题不好解决，为何不将数据划分为多个 region，每个 region 作为 1 个独立的 raft group只管理少量的数据呢？    
[TiKV](https://tikv.org/deep-dive/scalability/multi-raft/#:~:text=Here%20Multi-Raft%20only%20means%20we%20manage%20multiple%20Raft,Raft%20groups%20in%20terms%20of%20partitions%2C%20namely%2C%20Region.) 的 MultiRaft 就是采用了这种方式。通过将数据切分为多个 region(1段 key range)，每个 region 对应独立的 raft group，TiKV 能够轻易搞定海量数据。    
可能有人会将 MultiRaft 和 MongoDB 的分片集群进行对比。在 MongoDB 分片集群下，海量数据按照 shard key 映射到底层多组 shard servers，每一组 shard servers 也是一个 raft group。分片集群的复制也能达到 MultiRaft 的效果。    
但是 MongoDB 分片集群不同的地方在于，raft 运行的单元还是一组独立的 mongod 进程（里面可能包含多个 chunk，即多个 key range）。而 TiKV 中，raft 的执行单元是一个 region，粒度更小。Region 对应到 MongoDB 中应该是 chunk，即 1段 key range。    


### 3.2.3 小结
MongoRaft 和 Raft 虽然都是强主模式，通过大多数节点的日志复制来完成提交，但是在具体实现上却有很大区别。主要体现在：  
1. Raft 采用**推日志模型**，leader 通过 RPC 将日志推送给 follower。而 MongoRaft 采用**拉日志模型**，并通过 streaming 推拉结合的方式提升速度。拉日志模型带来一个明显的好处就是能更好的支持链式复制。  
2. Raft 先复制日志到大多数节点，leader 节点才应用日志。而 MongoRaft 中每个节点的日志复制和日志应用是”一体“的。  
3. Raft 中的日志采用递增的序号来标记顺序。MongoRaft 中的 oplog 采用混合时钟来标记顺序，可以更灵活应对各种业务需求，比如按时间戳读、回档到某个时间点等。  

另外，MongoDB 在内核实现中，使用了大量的架构优化来提升性能，比较重要的优化点有：  
1. 采用多线程 + 内存队列，将日志的拉取和回放流程**解耦**，使得整个复制过程**流水线化**，充分利用多核硬件。  
2. 采用最极致的并发策略，充分利用多线程提升日志持久化和回放的效率。在并发乱序回放过程中保证了读取数据的一致性，并且严格按顺序提交日志。  
3. 通过 flow control 限流机制和 streaming replication 机制尽可能提升日志传输的效率，避免主从延迟带来的风险。  
