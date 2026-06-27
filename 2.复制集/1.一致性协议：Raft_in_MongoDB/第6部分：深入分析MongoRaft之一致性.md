## 3.6 一致性
这里要讨论的一致性不是数据库事务 ACID 中的一致性，而是分布式存储系统中的一致性。具体分为 2 个维度：      
- 数据维度（Data Centric）。结合经典的 CAP 及其扩展后的 PACELC 理论，探讨 C(Consistency)、A(Availability)、P(Partition)，以及 L(Lantency)、R(Recency) 如何在 Raft 和 MongoRaft 中落地。   
- 客户端维度（Client Centric）。探讨因果一致性（单调读、单调写、写后读、读后写）和[线性一致性](https://github.com/Vonng/ddia/blob/master/ch9.md)。

### 3.6.1 Raft 原理
从数据维度来看，写请求必须要复制到大多数节点才算提交成功。因此，Raft 是强一致性协议。   
从客户端维度来看，读写请求必须都在 leader 节点上执行（clients of Raft send all of their requests to the leader），而且 leader 状态机上的数据都是已经提交的数据（when the entry has been safely replicated , the leader applies the entry to its state machine and returns the result of that execution to the client）。因此，如果不考虑网络分区切主的话，Raft 能够很好地满足单调读、单调写、写后读，读后写这些因果一致性。   

分布式系统一致性的目标，还是让多个节点像单个节点一样工作，客户端感知不到分布式系统的存在。这就不得不提线性一致性。   
在线性一致性保证下，一旦数据完成写入，所有客户端就能读到新版本的数据，旧版本的数据不会在之后的读操作中如幽灵般重现。显然，在不考虑网络分区切主，以及不读 follower 节点的情况下，Raft 也是能轻易满足的。   
然而现实中的分布式系统，是可能出现网络分区的。如果出现网络分区选出了新 leader，可能会出现：   
- 客户端读到新 leader（多数派），但是新 leader 上的状态机还没来得及 apply 已提交的日志。    
- 客户端读到旧 leader（少数派），旧 leader 上没有最新 commit 的日志，状态机也没有更新。   

总结来说，就是读取的节点的 appliedIndex < committedIndex。显然，这是不能满足线性一致性的。   
另外，很多实际的业务场景都会做读写分离，通过读 follower 节点来扩展读能力。所以，读 follower 节点仍然保证线性一致性也是一个切实的需求。   

解决上述问题的核心思路，就是避免读取时节点的 appliedIndex < committedIndex。参考 Raft 论文和一些数据库的解决办法，将常见的解决方案总结如下：   
|方案|应对的场景|基本思路|
|:--|:--|:--|
|Log Read|网络分区|为读操作也生成 1 条no-op log，走一遍  Raft 的提交流程，宣示 leader 的主权 。确保当前是多数派 leader 后再返回读结果。<br>这样可以防止出现网络分区后，仍然在旧 leader上读数据的情况。<br>这种方案比较直观。但是读操作的流程较长，性能会比较差。另外通过日志提交将读操作线性化后，读并发会受到影响。<br>**MongoDB 的线性一致性设计采用了这个思想。**|
|Read Index|网络分区|Leader 收到读请求后先记录 readIndex = committeIndex，然后**通过心跳宣示主权**，确认自己不是少数派。然后确保 appliedIndex >= readIndex 后再返回读结果。<br>Raft 论文原文：1) Leader commit a blank no-op entry into the log at the start of its term.2) Leader exchange heartbeat messages with a majority of the cluster before responding to read-only requests. |
|Lease Read|网络分区|对 Read Index 的一种优化，不要每次读请求都触发 heartbeat，而是**基于 heartbeat 来形成租约机制**。在租约的有效期内，各个节点不要选新主。<br>但是租约机制一般都有比较明显的缺点：**非常依赖时钟来保证租约的准确性**。<br>Raft 论文原文：Alternatively, the leader could rely on the heartbeat mechanism to provide a form of lease, but this would rely on timing for safety (it assumes bounded clock skew).|
|Follower Read|读 follower|Follower 节点收到读请求之后，先向 leader 节点询问当前的 committedIndex。然后**等待日志复制到 committedIndex 并且 apply 到状态机**之后，再处理读请求。<br>问题：怎么保证自己不是少数派的 follower 呢，然后又询问了少数派 leader？是否需要发心跳给大多数节点确认一下自己的 term 是否最新？<br>缺点：每次都去问 leader，网络开销会较大。<br><br>[PolarDB](https://mp.weixin.qq.com/s/yxRsxvWrqumeCahH528mdA) 借鉴了这个方案的思想，但是进行了 3 点性能改进：<br>1. **线性Lamport时间戳，降低询问 leader 的次数**。<br>Follower每次拿到leader的时间戳（committedIndex）时，都会把这个时间戳保存在本地，并且会记录获取到该时间戳的时间，如果某个请求的到达时间早于本地缓存时间戳的获取时间，则该请求可以直接使用该时间戳。<br>2. **分层的细粒度的修改跟踪，避免非必要的等待**。<br>维护三层修改信息：全局的最新修改的时间戳（committedIndex），表级的时间戳和页面（page）级别的时间戳。如果要读取的表最近没有修改，则可以直接执行读操作，而不需要等待全局的日志同步。<br>3. **基于RDMA的日志传输，降低网络延迟和 CPU 消耗**。|


### 3.6.2 MongoRaft 原理
MongoRaft 基于 Raft 进行了扩展，支持可调一致性。具体来说，支持以下 3 个维度的配置：    
1. [Write Concern](https://www.mongodb.com/docs/v4.2/reference/write-concern/)：写请求执行到什么程度再给客户端返回结果。包含 3 个配置：“w” 表示写请求同步到多少个节点，可以指定非负整数，也可以指定 "majority" 表示大多数节点，或者按照节点的 tag 进行自定义；"j" 表示是否要等到节点刷完 journal 才返回，可以指定 true 或者 false；"wtimeout" 表示等待复制的超时。

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/73a30e74-889a-4406-94b4-3f3e686c2345" width=700>
</p>   

2. [Read Preference](https://www.mongodb.com/docs/v4.2/core/read-preference/)：到哪个节点执行读请求。可以指定 primary, secondary, primaryPreferred, secondaryPreferred, nearest 多种模式，还可以结合节点的 tag、主从延迟情况信息进行灵活选择。

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/b8085800-06df-4e1b-8760-f81ad35f4cf9" width=300>
</p>

3. [Read Concern](https://www.mongodb.com/docs/v4.2/reference/read-concern/)：读取节点上哪个版本的数据。支持 available, local, majority, snapshot, linearizable 多种级别。在 MongoDB 复制模型中，主节点先应用状态机，再将 oplog 复制到其他节点。每个节点的状态机中都可能包含未提交（majority commit）的数据，这类数据只在本地作了提交（local commit）。如果和传统数据库中的事务隔离级别进行类比，{readConcern: local} 可以类比为 read uncommitted, {readConcern: majority} 可以类比为 read committed。    
MongoDB 通过混合时钟对底层数据进行了多版本（MVCC）管理，**因此如何按照 readConcern 读数据的问题，说白了就是如何按时间戳读数据**。    
Snapshot 和 linearizable 相对特殊。Snapshot 是读取固定时间戳版本的数据，不会随着持续的数据写入和 majority commit point 推进而发生变化，最初只用在事务中，从 [5.0 内核](https://www.mongodb.com/docs/v5.0/reference/read-concern-snapshot/)开始给普通读请求使用。Linearizable 保证线性一致性，只能在 primary 节点执行，primary 节点会等待读请求之前的写操作全部 majority 提交，再执行该读请求。    
不少人将 {readConcern: majority} 和 Cassandra 等系统中的 quorum read 机制混淆。Quorum read 是**读取大多数节点**，然后返回最新版本的记录，是存在读放大的。而 {readConcern: majority} 是根据 readPreference **读取 1个节点**，返回其 majority 提交的记录，不存在读放大。    

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/dd67d047-6e32-4c82-8fb1-324613cebf94" width=700>
</p>

对于 writeConcern、readPreference、readConcern 的实现原理，后续会有独立的章节进行分析。    
值得注意的是，上述都是请求级别（operation level）或者事务级别（transaction level）的配置。也就是说，用户在使用同一个 MongoDB 集群时，也能根据自身业务特点随时进行调整。比如一个在线编辑业务，可以对”自动保存“请求的writeConcern 设置为 {w : 1}, 而将 ”提交“ 请求设置为 {w: "majority"}.    
如果设置 {writeConcern:"majority", j:true}, {readConcern:"majority"} 和 {readPreference: "primary"}， 则MongoDB的一致性保障和 Raft 相同。因此，可以认为 **MongoRaft 的一致性是 Raft的超集**。   

当我们在设计和使用分布式系统时，不可避免要做权衡（tradeoff）。比如 CAP 理论中，在保证分区容忍性 P 的前提下，无法同时满足 C 和 A，需要根据业务特点进行取舍。    
在真实的业务场景中，除了 CAP 之外，我们经常还要关注吞吐能力、请求延迟、数据新鲜度等维度是否满足需求。理想情况下系统的表现应该是一个”六边形战士“，面面俱到。但是现实情况是，我们必须理解自己的业务系统需要强化哪些能力，弱化甚至放弃哪些能力。    

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/046af1ee-6814-43fd-8bea-955a29314d0c" width=400>
</p>

- **吞吐**。MongoDB 可以添加 secondary 节点来提升读吞吐能力。但是需要容忍 secondary 节点的数据存在延迟的问题，无法保障数据新鲜度。一般应对的是分析型业务场景，比如对历史数据进行分析等。或者业务自身有容忍或者鉴别过期数据的能力。   
- **延迟**。通过设置较低级别的  write concern 来较低写请求处理延迟，通过设置 {readPreference : "nearest"} 将读请求打散到多个节点能够降低读请求延迟。但是需要牺牲一定的一致性和数据新鲜度。   
- **新鲜度**。读 primary 节点，并且设置 readConcern 为 local，可以读取到最新版本的数据。但是读到的数据可能只是 local commit， 而不是 majority commit，因此可能会由于  HA 被回滚，需要牺牲一致性。用户也可以通过设置 readConcern 为 linearizable 来同时保证新鲜度和一致性，但是请求延迟会大大增加，而且由于只能在 primary 节点完成，吞吐量会大幅下降。   
- **可用性**。将 write concern 设置为较低级别，比如 1，可以提升写操作的成功率，在 secondary 节点宕机和延迟较大的情况下保证可用性。将副本集中 vote 权限只给少量节点，同样也能在节点宕机的情况下保证选主成功和写操作成功。但是上述策略都在一定程度上牺牲了一致性。    
- **一致性**。通过将 write concern 设置为较高级别，可以强化数据一致性。但是在 secondary 宕机和延迟较大的情况下，可能导致写操作超时失败，因此会牺牲可用性。    
- **分区容忍性**。分布式系统都需要保证这项能力，不存在”可调“空间。   

官方论文[《Tunable Consistency in MongoDB》](https://www.vldb.org/pvldb/vol12/p2071-schultz.pdf)对运行在 Atlas 平台的 14,820 个 4.0.6 内核版本的请求特征进行了统计分析。发现广大用户对 writeConcern 和 readConcern 的使用方式如下：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/a5164fee-583b-4b9c-b7c9-f0c41f9c9762" width=400>
</p>

从官方的统计中，我们能够得到以下启示：   
1. 可调一致性给业务带来了更大的灵活性。从表中可以看到每一种 readConcern 和 writeConcern 都有被使用。
2. 绝大多数都是使用的默认配置。我认为这不说明系统默认配置刚好就能满足绝大多数场景，而是很多用户并不清楚这项配置，也不知道如何在业务代码中进行自定义设置。我之前遇到过一些业务由于没有正确配置 writeConcern 导致有数据丢失的问题，可见业务在使用时有必要确认系统默认配置并且明确其带来的风险。另外，MongoDB 在不同内核版本下的默认配置可能是不一样的，比如 5.0 版本 writeConcern 的默认配置是 { w:”majority“}， 而 4.0 版本的 writeConcern 默认配置是 {w : 1}。因此，我的建议是**不要依赖系统默认配置，尽量在业务代码中进行明确配置，并知晓配置的风险**。


从客户端一致性的维度，MongoDB 提供可调的因果一致性：    
- Read own writes，写后读。本次读请求操作的是执行完上一个写请求的状态机。
- Monotonic reads，单调读。本次读操作得到的数据版本，不会比上一次读操作的版本更低。
- Monotonic writes，单调写。本次写请求操作的是执行完上一个写请求的状态机。
- Writes follow reads，读后写。本次写请求操作的状态机包含上一次读取操作的版本。    

如果因果一致性必须包含持久性，则必须 {readConcern: "majority"} 且 {writeConcern: "majority"} 才能满足要求：

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/94fa0054-e5a0-4363-ba24-1fe9439727ae" width=700>
</p>

我们可以稍作推理来验证上述结论。以 read own writes 为例，设置 writeConcern 为 {w:1} 无法保证数据被 majority 提交，存在会被回滚的风险，因此无法保证下个操作一定能读到。设置 readConcern 为 local，可能读到存在回滚风险的数据（比如网络分区时，读到了少数派节点），也不满足需求。更多分析可以参考[官方文档](https://www.mongodb.com/docs/v4.2/core/causal-consistency-read-write-concerns/)。   
在因果一致性不需要完全满足持久性的情况下，可以放松上述要求。

MongoDB 作为一个商业数据库，性能是其架构设计时的重要考虑因素。因此，MongoDB 没有采用全局因果一致性方案(full dependency tracking)，因为维护和分析全局的依赖关系非常耗性能。
MongoDB 通过[混合逻辑时钟](https://dl.acm.org/doi/pdf/10.1145/3299869.3314049)(Hybrid Logical Clock, HLC) 来维护每个 client 内部多个请求的顺序。**HLC 由 <int32: unix second> 和 <int32: counter> 共同组成**，也叫 clusterTime 在集群中传播。Primary 节点在执行写请求时推进 HLC，并将其写入到 oplog，也会跟随数据写入到存储引擎，作为 MVCC 版本控制的依据。    
Primary 节点作为 HLC 的推动者，集群中的其他节点（client, mongos, config server, shard secondary）作为 HLC 的传播者，通过 gossip 机制实现全局同步。   

下图说明 Client 如何通过 clusterTime 在 secondary 节点上完成"read own write":    

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/91397078-d153-4106-9891-7f8b93d895ba" width=500>
</p>

1. Client 给 primary 节点发送写请求。
2. Primary 执行请求，并推进 opTime 到 T2，并记录 oplog 进行异步复制。
3. Primary 将 opTime T2 作为返回结果的元数据传递到 Client.
4. Client 推进 opTime 到 T2.
5. Client 给 secondary 节点发起读请求，并携带 {afterClusterTime: T2}，表示要读取 T2 及之后版本的数据。
6. Secondary 判断自己的同步进度有没有到 T2, 如果没有则等待。如果设置了 readConcernMajority，则会等待 MajorityCommitPoint 推进到期望的时间点，否则，只需要等待本地的 lastAppliedOpTime 到指定时间点即可。参考 [waitUntilOpTimeForRead](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/read_concern_mongod.cpp#L307) 到 [ReplicationCoordinatorImpl::_waitUntilOpTime](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/repl/replication_coordinator_impl.cpp#L1418) 的函数调用和实现。
7. 如果满足条件，则执行读请求，并返回文档。

HLC 主要还是解决如何定序，管理数据版本的问题。在此之前，业界已经存在一些经典的解决方案，比较知名的有 Lamport 的逻辑时钟，Google Spanner 的时钟服务等。MongoDB 最终还是采用了 HLC 方案，可以从以下几个问题进行分析。    
1. **为什么不采用逻辑时钟，而加入了物理时间（wall clock）?**   
真实的业务场景往往是伴随物理时间的，比如按时间点读取数据，按时间点回档数据库等，能精确到 xx年xx月xx 日xx时xx分xx秒。如果只用逻辑时间，很多业务场景都无法满足。
2. **为什么不使用 Spanner 的 TrueTime 时钟服务？**   
MongoDB 官方认为会增加读写请求的延迟（每次执行请求之前都要调用 TrueTime 的 API），而且增加部署难度（额外的服务和硬件）。
3. **为什么不直接用更精细的纳秒时钟，而是采用 秒级时钟 + 秒内counter 的方式？**   
引入物理时钟之后，面临的问题是如何保证时钟一直会持续递增？一般部署 MongoDB 的机器都会采用 NTP 服务进行时钟对齐，那物理时间就有回拨的风险。引入 counter 能够保证即使机器时钟回拨，也不倒退 HLC的物理时钟，而是推进 counter 来保证递增。**换句话说，机器时间只是 HLC 的参考，但不是绝对权威**，具体可以参考 [LogicalClock::reserveTicks](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/logical_clock.cpp#L98) 代码的实现。但是一般来说，NTP 能够保证机器时钟不会有巨幅波动，所以可认为 HLC 的物理时间 == 机器时间。
4. **HLC如何保证安全性？**   
如果由于黑客攻击，将集群中的 HLC 推进到了 Unix 时间的最大值，则集群将无法继续处理写请求。因此，MongoDB 对集群中传递的 HLC 增加了 Hash 签名校验，密钥只有集群内部知道，而且定期更新。另外代码中也对 HLC的推进作了限制，默认每次推进的时间不能超过 1年。
Hash 校验保证了安全性，但是牺牲了计算资源。MongoDB 为了提升性能进行了2点针对性地优化：      
a. 区分[特权认证](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/logical_time_validator.cpp#L200)的连接，这些被信任连接的请求可以免去 Hash 校验， 一般是内部节点通信使用的 `__system` 角色的用户， 以及内核代码流程自己进行函数调用的 DirectClient；     
b. 充分利用 HLC [只能向前推进](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/logical_time_validator.cpp#L161)的特性：如果请求携带的 ClusterTime 比本地的还旧，不足以推进本地的 ClusterTime，也就没有校验的必要;      
c. 通过[缓存](https://github.com/mongodb/mongo/blob/r4.2.24/src/mongo/db/time_proof_service.h#L44)避免重复的 Hash 计算：内核侧使用 [16bit 的 Mask](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/time_proof_service.cpp#L45)，对 ClusterTime 进行[批量校验](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/time_proof_service.cpp#L61)（忽略 ClusterTime 中最后 16 位 counter 的差异）。也就是说在 1 秒内，每连续 65536 个 ClusterTime 共用一个 Hash 校验。可想而知，缓存命中率非常之高了。       
5. **HLC 如何传播？**   
采用 gossip 机制，所有读写请求涉及的节点（Client 和 MongoDB中的各个节点）都会参与传播。但是只有 primary 节点有权限推进 HLC。

### 3.6.3 小结
1. Raft 提供强一致性协议。所有读写请求都集中在 leader 节点处理，都是操作的 majority committed 数据，能够满足因果一致性。   
2. MongoRaft 支持 writeConcern、readPreference、readConcern 动态配置，提供了可调一致性，并且这些配置下放到了请求级别。用户可以根据自身在吞吐、延迟、一致性、数据新鲜度、可用性方面的业务需求，进行灵活配置。   
3. MongoRaft 采用了混合逻辑时钟进行全局时钟同步，支持客户端级别的因果一致性。   
