## 3.3 选主协议
Raft 和 MongoRaft 都是 “强主”协议，所有的写操作都在主节点上完成，并复制到从节点。因此，如何快速安全的选出主节点非常关键。简单来说，主节点的选举由副本集中大多数节点（包括candidate节点自己）投赞成票而来，然而实际生产环境中还需要解决一些异常场景，包括：   
1. 触发选主的条件是什么？
2. 如何避免网络分区导致同时出现多个主节点？
3. 如何避免在配置变更时出现多个主节点？
4. 如何在出现投票冲突时快速达成一致？
5. 如何主动进行主节点切换？
6. 如何保证新主节点一定包含已提交的数据，即不会出现已提交的数据由于切主而丢失？

下面我们带着这些问题，分析一下 Raft 和 MongoRaft 的处理流程。其中配置变更时的选主安全性问题，我们单独在配置变更章节描述。

### 3.3.1 Raft 原理
#### 何时触发   
Leader 定期给 follower 发送心跳。如果 follower 节点在一段时间内（Election timeout）没有接收到心跳，则状态变为 candidate 并发起选主。   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/c849c661-4c9a-4eef-ae0a-8bd31b6fbb69" width=300>
</p>

#### 选主流程   
每轮选主流程都在某一个 term （任期）下进行，term 对应了一个严格递增的整数，可以用来标识节点的状态是否过期。     
在一个 term 内，每个投票者最多只能投一次赞成票，因此最多只会选举出一个主。当然，也有可能因为超时或者选举冲突等原因，某个 term 内选不出主。    

Candidate 首先将 term 加 1，然后并发给副本集中的其他节点发 RequestVote 请求。Candidate 节点接下来可能会遇到 3 种场景：   
1. 副本集中大多数节点（包括 candidate 自己）投了赞成票，此时 candidate 成功变为 leader. 投票节点在每个 term 最多只会投出 1 张赞成票，而且为了防止出现已提交数据被回滚的情况，candidate 的日志必须足够新才能得到赞成票。   
2. 收到另外一个节点声明自己是 leader 的心跳信息。如果那个新 leader 的 term 大于等于自己当前的 term，则 candidate 变为 follower，否则忽略那个心跳。   
3. 没有收到大多数节点的赞成票，则 candidate 等待一段时间后 term 加1，重新发起选举。出现这种场景，可能是副本集中同时有多个 candidate，每个 candidate 都只得到了少数票。也有可能是副本集当前只有少数节点存活。   

对于上述第 3 种场景，raft 采用的解决方式是 candidate 等待随机一段时间后再重新发起选举，这样下次选举的冲突几率就非常低了。   
最初 raft 打算采用给 candidate 派优先级的方式来解决冲突（ranking system），优先级低的 candidate 如果发现有更高优先级的 candidate 存在，则出动退回到 follower 状态。这个方案看起来合理，但是实际测试中发现系统的可用性（无主时间）会受到较大影响，而且有很多极端场景导致的bug. 最终，还是选择了随机回退的方案。   
从 raft论文 给出的测试结果来看，随机回退能够有效避免选举冲突（split vote）的情况。   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/40a79a7b-7ee7-4e09-82d8-44a4cdc41beb" width=400>
</p>
   
在上面的半张图中，150-150ms 的场景是没有任何随机性的，可能选主需要长达 10 秒才收敛。如果增加 5 ms 的随机性，即选主超时在 150-155ms 中的随机值，平均 287 ms 即收敛，可以说效果非常明显了。   
在下面的半张图中，raft 分析了将 election timeout 设置为多长更合适的问题。一般来说设置的短一点，follower 能很快发现无主并发起选主，降低系统无主的时间，比如实验中设置 12-24ms 的 election timeout, 平均只需要 35ms 就能选出新主。   
但是设置的太短也有问题，如果出现一些网络延迟抖动，可能导致一些不必要的选主和切主。比如 3 副本分别在中国、北美和南美，网络 RTT 就在百 ms 级别，而且网络不稳定，如果设置 election timeout 为 200 ms, 可能会导致选主非常频繁。**因此，election timeout 的设置要依据网络距离，网络质量等因素进行综合评估。**   

#### 选主成功后的流程
如果 candidate 成功变为 leader，则立刻向其他节点发送心跳昭告自己的状态，并避免新的选举发生。

### 3.3.2 MongoRaft 原理
#### 何时触发
mongo 的选主支持主动和被动 2 种触发方式。     
主动触发的方法有：   
- 主节点主动 stepDown。在主节点上主动执行 [replsetStepDown](https://www.mongodb.com/docs/v4.4/reference/command/replSetStepDown/#replsetstepdown) 命令，主节点会马上转为 secondary, 并很快选举出新主节点。相比重启主节点来触发选主，replsetStepDown 方式触发选主的[时间更短](https://www.mongodb.com/docs/v4.4/reference/command/replSetStepDown/#election-handoff)，从节点不需要等待 election timeout （默认 10 秒）就发起选主。   
- Priority takeover。如果某个节点发现自己配置了比主节点更高的 priority，则会在等待一段时间后发起选举（参考 [getPriorityTakeoverDelay](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/repl/repl_set_config.cpp#L898)，一般来说，如果要发起的节点的 priority 全局最高，会在 electionTimeout 后发起选举）。对于更高 priority 的从节点，只有在和副本集中最新的节点日志相近时（参考[TopologyCoordinator::_amIFreshEnoughForPriorityTakeover](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/repl/topology_coordinator.cpp#L1279)），才能发起选主。      

>**备注1：** 如何判断和最新的节点日志相近，适合发起 priority takeover呢？ 从前到后有 3 条规则：     
>1. 如果当前节点 term 和最新节点的 term 不同，则不相近；     
>2. 如果当前节点最新应用的日志和最新节点的日志时间相差小于等于 [2 秒](https://github.com/mongodb/mongo/blob/r4.2.25/src/mongo/db/repl/topology_coordinator.idl#L40)，则相近；      
>3. 如果日志的秒数相同，则计算日志相差的条数小于等于 1000 条，则相近。这一条规则可以看作是对第 2 条规则在时钟跳变场景的补充，比如旧主节点之前时间比较新，后来通过时钟同步(比如NTP同步)之后又跳变回来，此时其产生的 oplog 秒数会一直维持在这个高位（但是很长一段时间不会再增长），而是靠 TimeStamp 中的秒内 inc 递增来维持“逻辑时钟推进”。此时，通过计算 inc 差值在 1000 以内，是一个不错的判断准则。

>**备注2：** Priority takeover + 后面提到的catch up 机制，是一套安全切主的组合拳。          
>想象一种场景：一个副本集有 3 个节点，部署在北京（primary）、上海和广州（secondary）。副本集持续在写入，现在希望切主到上海。则可以等到上海的节点在满足延迟窗口之后选举为主，同时通过 catch up 机制将北京节点上最新的那一小段日志也安全的同步完毕，然后才接收客户端的新写入。 


被动触发一般对应节点不通的场景：   
- 主节点一段时间内（election timeout, 默认 10 秒）感知不到大多数节点的存在，则主动 stepDown。  
- 从节点一段时间内（election timeout, 默认 10 秒）感知不到主节点的存在，则发起选举。  

Mongo 也使用了心跳机制来感知各个节点的状态，但是与 raft 不同的是，Mongo 中的任意 2 个节点都能相互发心跳请求：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/607a6fd1-6969-4a41-a042-c1f2f0a8a9cf" width=500>
</p>

由于任意 2 个节点之间都能互发心跳，对于 N 个节点的副本集，平均每个心跳周期内，副本集内的总心跳请求有 N(N-1) 个，如果节点太多会出现爆发式增长。这也是 MongoDB 限制一个副本集最多 50 个节点的原因之一。   
这种心跳机制的好处是，副本集中每个节点都能知道全局的节点状态信息，因为心跳信息中包含了节点的 opTime 同步进度，replsetConfig 版本，主节点的状态和 term 等信息。    
基于上述机制，副本集中的从节点可以根据一定的规则选择合适的同步源节点，并形成链式复制结构（生成无环的 spanning tree）。   
另外，在 MongoDB 的复制模型中，拉取 oplog 的请求和更新oplog 复制位置的请求会携带包含节点状态的元数据。   
以下图为例，副本集中有 5 个节点，在某个时刻主节点 A 和 C, D, E 网络不通，但是 B 和所有节点都能正常通信：   

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/f5450ee0-bcec-40ce-a376-cfad9a2c4bd4" width=600>
</p>

对于左侧 Raft 来说，C, D, E 在一段时间内接收不到主 A 的心跳，会发起选主。   
对于右侧的 MongoRaft 来说，C, D, E 可以从 B 节点同步，链式复制结构会让 A 通过 B 节点感知到 C、D、E的状态，而不用重新发起选主。   
不过需要注意的是，如果节点 B 是 arbiter 或者数据非常落后，则不会形成上述链式复制结构。   

#### 选主流程
选主流程分为 2 步：   
1. Dry-run election, 即试探性发起选举。其目的在于试探集群中有多少节点会支持自己，增加后续真正选举的成功率。发起 dry-run election 时不会增加自己的 term，所以不会造成 term 无意义的递增，而且也不会导致当前的 Primary 节点 stepDown.   
2. Real election，candidate将 term 加 1，然后真正发起选举（旧主如果收到这个 term 更大的 real election 请求，会主动 step down）。首先 candidate 会投票给自己，然后并发给集群中有投票权的节点发起投票请求。如果获得了大多数赞成票，则选举成功。   

从 voter 的视角来看，它在收到 candidate 发过来的 requestVotes 命令时，先判断 term 是否比自己的新，并更新自己的 term. 然后判断自己是否应该投赞成票，如果满足以下条件，会投反对票：  
- Candidate 的 term 更低。   
- 配置不匹配，replSetName 不匹配。   
- Candidate 的数据更旧（lastAppliedOptime更小）。   
- 对于这个 term，voter 已经投过一次票了（当然 dry-run 流程的不算数）。   
- Voter 是一个 arbiter, 而且它能感知到当前存活了另一个主，并且这个主的 priority 不比 candidate 低。这个策略主要是应对 Primary-Secondary-Arbiter 架构下，primary 和 secondary 不能通信，arbiter 能够和它们都正常通信的场景。如果 arbiter 没有这样的投票策略，可能同时会出现 2 个主。   

一旦节点给自己或者其他节点投票，则会将 “lastVote” 信息持久化到本地的 local.replset.election 表中，避免节点重启之后这些信息丢失，导致给同一个 term 多次投票的情况。   

#### 如何保证选主流程的安全
一个节点只给一个 term 投 一次票的机制，避免了一个 term 出现多个主节点的情况，但在某些时刻，集群中可能出现 2 个主的情况，分属于不同的 term。    
举例如下：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/b13756c9-9c85-47e5-9c0c-406ad4acf66d" width=400>
</p>

(a). 所有节点运行在 term 1, 其中S1 为主节点。    
(b). S1和S5 出现网络隔离，S5 发起选举并获取 S3、S4、S5的投票成为了新主。S1、S2 运行在 term 1, S3、S4、S5运行在 term 2。    
(c). S1 和 S5 都能接收客户端的写请求，S5 的写请求能提交（到大多数节点）。S1 的写请求只能复制到 S1 和 S2，其他节点由于网络不通，因此会卡住不能提交。    
(d). S1 的日志复制到 S3、S4、S5, 但是term 太低并拒绝。    

对于 Raft 来说，S1 推送给 S3、S4、S5 的复制请求会由于自身 term 太低被拒绝。    
而对于 MongoRaft 来说，由于采用了从节点拉日志的方式，情况要更复杂一些。在上述例子中，S3 和 S4 的日志复制取决为当时的同步源节点是否切换为 S5，如果切换为 S5 则压根不会从 S1 节点复制日志。如果同步源还是 S1 ，则会在应用日志后，通过 updatePosition 命令给 S1 反馈日志同步情况，该命令中会携带 S3/S4 自身的 term 为 2。 S1 在收到这个命令后，明白自身的 term 太低，主动 stepDown，并不会将该写请求置为提交。   
MongoDB 论文中列举了不一样的例子：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/d3583214-26e2-4bf1-a84e-cf24fe277f20" width=400>
</p>

(a). A、B运行在 term 2, A 作为主节点提交了日志 “2”；C、D、E 运行在 term 3， E 作为主节点提交了日志 “3”.   
(b). Raft 协议：A的日志只能复制到节点B。C、D、E 节点由于 term 更高会明确拒绝。   
(c). MongoRaft 协议：A 的日志可以复制到 C 和 D.   
(d). MongoRaft 协议：C 和 D 在 updatePosition 反馈时表明自己是 term 3，A 收到后主动 stepDown，不会提交日志 “2”。   
(e). MongoRaft 协议：E 节点的日志“3”复制到所有节点，并 overwrite A节点的日志“2”。   
整体来看，虽然由于复制模型导致处理过程不一样，但是 MongoRaft 和 Raft 殊途同归。   

引入 term 机制后确实避免了同时出现多主并同时提交日志的问题。但是也引入了另外一个问题：主节点是否能直接提交之前 term （older term）的日志？   
对于 Raft 和 MongoRaft 来说，答案都是否定的。   
首先引用 Raft 论文中的一个例子，看看提交主节点如果提交之前 term 的日志会有什么问题：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/bf8eb6f9-4b64-4b96-a2fd-a73a39d83c60" width=400>
</p>

(a). S1 是主节点，并将最新的日志 "2" 同步到了 S1 和 S2，还没有提交。    
(b). S1 挂了，S5 被 S3、S4、S5 选举为新主，运行在 term 3，并写入了日志 “3” 但没有复制到其他节点。    
(c). S5 挂，S1 重启并成为新主。然后继续将之前的日志 “2” 复制到 S3, 此时已经复制到了大多数节点（已经复制到了S1、S2、S3），但还没有提交。    
(d). S1 没有提交日志 “2” 就挂掉，则 S5 可能重新被选为主，并在选主之后会将之前已经复制到大多数的日志“2” 抹掉（overwrite）。   
(e). S1 提交了当前 term 的日志“4” 才挂掉，则 S5 不可能被选为主。日志 “2”在日志“4”之前，才会保持提交状态。  

>需要特别说明的是，为什么提交了日志 “4”， S5 就不会被选举为新主，而之前“提交”了日志 “2”， S5 却能被选举为新主呢？   
>Raft 选举时，voter 给 candidate 是否投票，有一项主要的依据就是比较日志的新旧：如果 voter 的日志更新，就不会给 candidate 投票。而比较日志的新旧，首先就是要比较日志中的 term 大小。
在上述例子中，日志“2” 的 term小于日志“3” 的 term，日志 “3” 的 term 小于日志“4” 的 term。

从上述场景(d) 中可以看到，即使older term 的日志复制到了大多数节点，还是可能被抹掉。因此，Raft 中的主节点不会通过计算日志是否复制到了大多数节点来提交日志，而是**通过提交当前 term 的日志来间接提交之前 term 的日志**（场景(e))。   
Raft 论文中也提到，采用上述“一刀切”的机制还是方便协议的理解落地，降低复杂性。理论上来说，如果在上述步骤 (c) S1 挂掉之前，日志 “2” 复制到了所有节点（并 overwrite 了 S3 的日志“3”），其实也能肯定日志“2” 的提交状态。   

MongoDB 也采用了一样的机制，如果从节点反馈的 updatePosition 中的 term 较低，主节点不会真正提交这条日志。只有收到的反馈中 term 和主节点当前的 term 相同，才会真正提交这条日志。在某一条日志被提交之后，比其更早（opTime 更小、term 更小）的日志也间接提交了。   

**这也是为啥 MongoDB 在选举新主节点之后，会马上生成一条当前 term 的 oplog 日志，其实就是为了提交更早 term 的日志。**   

#### 选主成功后的流程
新主节点在选举成功之后会进入如下流程：   
1. 停止当前的同步逻辑，并将自己选举成功的消息通过心跳告知其他节点。   
2. 检查自己是否需要 catch-up. 比如副本集中有 5 个节点，数据从新到旧的排序为 A>B>C>D>E，B得到了 BCDE 的投票成为新主。A 节点上存在一些更新的日志，虽然没有提交，但是 B 节点也会同步过去。具体实现流程为步骤1 中的心跳响应中包含了各个节点的 lastAppliedOpTime 信息，如果有节点比自己新，则这个新主节点就会进入 catch-up 阶段尽可能多同步一些日志。在进入 catch-up 节点之前，新主节点会启动一个定时器，防止catch-up 时间不可控，集群长期无主影响可用性。如果 catch-up 超时也没有关系，因为这只是一个尽力而为的操作。   
3. Catch-up 不论成功与否，接下来会进入 drain-mode. 新主节点将拉过来的日志进行回放（apply）。   
4. 新主节点写一条 “new primary” 的 oplog，方便提交之前 term 的日志。   
5. 新主节点删除上述临时表、终止残留的事务。此时选主操作才真正完成，新主节点可以对外提供写服务。   

从上述流程可以看出，catch-up 阶段时选主成功后 MongoRaft 和 Raft 最大的区别。MongoRaft 通过这个机制可以尽量保留已经写入到主节点但是还没有提交（到大多数节点）的数据。主要是因为MongoDB 支持非 majority的写入方式（local commit），比如客户端为了性能考虑，可以在只写主成功后即返回。原则上来说，这些数据是可能被回滚的，MongoDB 也不保证这种写入方式的持久性。但 MongoDB 还是在设计上尽量避免发生回滚。

### 3.3.3 小结
纵观 Raft 和 MongoRaft 的选主流程，在心跳探活、term、大多数选举等机制上大致相同。但是 MongoRaft 相比 Raft，还是在不少方面做了改进，包括：   
- 探活机制方面，MongoRaft 任意 2 个节点之间都有心跳，容错性更高。另外，MongoRaft 支持灵活的链式复制架构生成 spanning tree 并传递节点状态信息。使得集群对网络的容错性也更高，避免发生不必要的选举。
- 选主流程上，MongoRaft 引入了 dry-run 机制，提升了真正选举时的成功率，避免不必要的term 递增以及由此造成的主节点 stepDown。
- 选主成功后，MongoRaft 引入了 catch-up 机制，尽可能地保留已经写到主节点但是还没有提交（到大多数节点）的数据，增强了数据持久性，减少回滚操作。