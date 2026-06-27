# 1. 导语
MongoDB 使用多个节点组成副本集保证数据高可靠以及服务高可用。对于自动容错的分布式系统来说，如何保证HA、数据一致性、高吞吐、低延迟、协议易理解（易运维）是系统设计的关键。   
一致性协议（consensus algorithm， 共识算法）就是上述问题的解决方案。    

# 2. Raft 和 MongoRaft
目前业界比较流行的一致性协议有 Paxos 和 Raft。[《共识协议的技术变迁》](https://mp.weixin.qq.com/s/UY9TPMcuf0O7xS0kuXTcVw)一文中对一致性算法的发展历程进行了非常通俗易懂的描述。   
Paxos 诞生时间比较早，并使用在 Chubby，ZooKeeper 等系统中，但是协议理解起来比较复杂，学习路径有些陡峭。   
Raft 的诞生实践稍晚，相对 Paxos来说非常好理解。从 [Raft 论文](https://raft.github.io/raft.pdf)也可以看出，从设计之初就考虑了协议的通俗易懂，以及如何指导工程落地。在 etcd，redis 等开源项目中都能看到它的身影。   
正如 Raft 论文中所说，Raft 本身还有很多值得优化的地方，比如日志复制的性能问题等。另外，在实际的成功落地过程中，也有很多边界情况需要考虑。所以，业界很多系统使用的 Raft 算法时都是根据实际情况进行优化后的变种。截止目前，在 [raft.github.io](https://raft.github.io) 中登记的使用了Raft算法的开源项目已经有几十个（比如 TiKV, etcd 等），如果算上没有登记的项目（比如 MongoDB） 应该有上百个。    

MongoDB 使用的一致性协议也可以看作是 Raft 协议的一个变种，同样使用了强主模式来定序，使用日志复制到大多数节点来提交请求。但是 MongoDB 在落地过程中，相比于原生 Raft 协议还是有不少改进，比如更丰富的节点状态、Pull-Based 复制模型、链式复制、可调一致性、Logless Reconfig 等。    
正如 MongoDB 官方论文[《Fault-Tolerant Replication with Pull-Based Consensus in MongoDB》 ](https://www.usenix.org/conference/nsdi21/presentation/zhou)中所说，MongoDB 选取 Raft 而没有基于 Paxos 很大一定程度上也是基于工程落地方面的考虑。理论上来说，如果基于 Paxos 实现也是没有问题的。   
MongoDB 一致性协议的[发展历史](https://www.usenix.org/system/files/nsdi21_slides_zhou-siyuan.pdf)如下：   
<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/3b514626-b141-45ed-8499-f372c944489f" width=800>
</p>   

- MongoDB 1.0 是刀耕火种的时代，需要手动进行 failover.   
- MongoDB 1.6 引入了自动 failover 机制，但是是基于没有证明的私有协议。
- MongoDB 3.2 基于 Raft 重新改造了一致性协议，进行了 TLA+ 证明。同时在改造过程中保留了 MongoDB自身 的 Pull-Based 日志复制模型、链式复制等诸多优秀特性，使得MongoDB 的一致性协议在正确性、效率、性能等各方面都达到了非常高的标准。   

由于 MongoDB 官方已明确说明一致性协议基于 Raft，而且在 [《Design and Analysis of a Logless Dynamic Reconfiguration Protocol》](https://drops.dagstuhl.de/opus/volltexte/2022/15801/pdf/LIPIcs-OPODIS-2021-26.pdf)论文中更是将配置变更算法起名为 MongoRaftReconfig. 因此，为了描述方便，本文统一将 MongoDB 的一致性协议简称为 MongoRaft。

类比是非常好的学习方法。本文首先从原生 Raft 入手，从 Raft 论文简要了解其设计思想和流程。然后结合 MongoDB内核源码、官方论文以及github wiki，重点介绍 MongoDB 的一致性协议，并说明相比原生 Raft 所作的优化及效果。   
结合 Raft 论文对协议做的模块划分，以及我个人的理解。本文将分如下几个模块阐述：   
1. 节点状态及其变迁规则。    
2. 日志复制和提交。    
3. 如何选主。    
4. 配置变更。    
5. 持久化保证。    
6. 一致性保证。    
