# Table of contents

* [README](README.md)
* [1. 文档模型](1.文档模型/README.md)
  * [1.1 BSON 文档](1.文档模型/1.BSON文档.md)
  * [1.2 KeyString 和索引](1.文档模型/2.KeyString和索引.md)
  * [1.3 文档模型和存储引擎](1.文档模型/3.文档模型和存储引擎.md)
* [2. 复制集](2.复制集/README.md)
  * [2.1 一致性协议：Raft in MongoDB](2.复制集/1.一致性协议：Raft\_in\_MongoDB.md)
  * [2.2 深入分析 oplog](2.复制集/2.深入分析oplog.md)
  * [2.3 ReadPreference：可调读写分离](2.复制集/3.ReadPreference：可调读写分离.md)
  * [2.4 WriteConcern：可调持久性](2.复制集/4.WriteConcern：可调持久性.md)
  * [2.5 ReadConcern：可调隔离级别](2.复制集/5.ReadConcern：可调隔离级别.md)
* [3. 分片集群](3.分片集群/README.md)
  * [3.1 基本概念](3.分片集群/1.基本概念.md)
  * [3.2 TODO-路由原理](3.分片集群/2.TODO-路由原理.md)
  * [3.3 TODO-Balancer过程](3.分片集群/3.TODO-Balancer过程.md)
  * [3.4 TODO-拓扑探测](3.分片集群/4.TODO-拓扑探测.md)
  * [3.5 TODO-内部连接池模型和使用注意事项](3.分片集群/5.TODO-内部连接池模型和使用注意事项.md)
* [4. TODO-执行器](4.TODO-执行器/README.md)
  * [4.1 TODO-锁与并发控制](4.TODO-执行器/1.TODO-锁与并发控制.md)
  * [4.2 TODO-执行计划](4.TODO-执行器/2.TODO-执行计划.md)
  * [4.3 TODO-副本集事务](4.TODO-执行器/3.TODO-副本集事务.md)
  * [4.4 TODO-分布式事务](4.TODO-执行器/4.TODO-分布式事务.md)
* [5. 最佳实践](5.最佳实践/README.md)
  * [5.1 TODO-警惕连接和认证带来的性能抖动](5.最佳实践/1.TODO-警惕连接和认证带来的性能抖动.md)
  * [5.2 警惕 TTL 索引带来的性能抖动](5.最佳实践/2.警惕TTL索引带来的性能抖动.md)
  * [5.3 警惕请求挤压，防止雪崩](5.最佳实践/3.警惕请求挤压，防止雪崩.md)
  * [5.4 TODO-存储引擎刷盘及调优思路](5.最佳实践/4.TODO-WiredTiger\_Cache的Eviction问题，以及调优思路.md)
  * [5.5 TODO-监控及问题排查](5.最佳实践/5.TODO-一些值得关注的关键监控指标，以及问题定位工具.md)
* [6. 一些有趣的问题](6.一些有趣的问题/README.md)
  * [6.1 如何入手 MongoDB 内核代码](6.一些有趣的问题/1.如何入手MongoDB内核代码.md)
  * [6.2 逻辑删除是否会释放磁盘空间](6.一些有趣的问题/2.逻辑删除是否会释放磁盘空间.md)
  * [6.3 TODO- 一个 100GB 的表，btree 高度是多少](6.一些有趣的问题/3.TODO-一个100GB的表，Btree高度是多少.md)
