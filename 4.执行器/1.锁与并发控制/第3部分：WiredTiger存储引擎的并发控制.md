# 3. WiredTiger 存储引擎的并发控制
上一章节讲述了 Mongo Server 层如何使用 Global、DB、Collection 级别的锁进行并发控制，但是在更低级别的并发控制上仍然存在一些疑问：    
1. 多个线程并发读取同一行数据，是否要加行锁呢？
2. 多个线程访问同一个 page，比如读写流程需要持有 page，eviction 流程也需要持有 page 并可能释放 page，是否需要加“页锁”？
3. Wiredtiger 引擎中的 DDL 操作是否也要加更高级别的锁呢？比如有一个线程正在删表，而 checkpoint 线程此时正在做一致性快照，两者之间如何互斥？

带着这些问题，我们通过锁机制和 MVCC并发控制 2 个方面来研究一下 wiredtiger 引擎。

## 3.1 存储引擎中的锁
下面从资源的维度，自顶向下介绍 WT 中的锁机制，从全局（WT_CONNECTION）、表（WT_DATA_HANDLE）、页（WT_REF、WT_PAGE）和行（WT_ROW） 4 个级别进行阐述。    
由于行锁的设计较为复杂，涉及到 MVCC(Multi-Version concurrency Control) 的实现机制，因此会单独作为一个小章节。    

### 3.1.1 WT_CONNECTION
Mongod 进程启动时，会调用 wiredtiger_open 初始化 WT 引擎（包括初始化一些数据结构，恢复一致性状态以及启动一些后台线程等），并生成一个 WT_CONNECTION。对于针对 WT 引擎的读写操作，会由 WT_CONNECTION 生成 WT_SESSION 和 WT_CURSOR来完成。    
因此，WT_CONNECTION 可以看作是一个顶层的数据结构，其包含的锁也具有一定的全局属性。    

[WT_CONNECTION](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/include/connection.h#L216) 中对锁的定义如下：    

|锁|类型|描述|常见的使用场景|
|:--|:--|:--|:--|
|api_lock|WT_SPINLOCK|Connection API spinlock|添加 collator/compressor/extractor，创建 session 等。一般用于保护 WT_CONNECTION 内部的 dlhqh /storagesrcqh /extractorqh /dsrcqh /compqh /collqh /sessions /session_cnt 等链表、数组和计数器等对象，防止并发修改导致数据错乱。|
|**checkpoint_lock**|WT_SPINLOCK|Checkpoint spinlock|用于 checkpoint 流程和其他流程进行互斥。比如保证同一时刻只能有 1 个 checkpoint 在执行，checkpoint 不能和 引擎的reconfig/rollback 以及表的  alter/drop/rename/alvage/trucate/upgrade/verify 操作并发执行。|
|fh_lock|WT_SPINLOCK|File handle queue spinlock|保护 WT_CONNECTION 内部的  file handle 链表，防止并发修改导致的数据错乱。|
|flush_tier_lock|WT_SPINLOCK|Flush tier spinlock|---|
|**metadata_lock**|WT_SPINLOCK|Metadata update spinlock|保护 Wiredtiger.wt 元数据表|
|**reconfig_lock**|WT_SPINLOCK|Single thread reconfigure|防止对 WT 引擎进行并发 reconfigure。这里的 reconfigure 是 WT 引擎层面的，比如 cacheSize，evict 阈值和线程数等。|
|**schema_lock**|WT_SPINLOCK|Schema operation spinlock|保护对 schema 的变更操作。包括表的  alter/drop/rename/alvage/trucate/upgrade/verify/compact 操作，获取/释放 data_handle，rollback_to_stable, 使用 backup_cursor 进行物理备份等。|
|table_lock|WT_RWLOCK|Table list lock|保护 WT_CONNECTION 内部的 dbhash, fhhash 等 hash table。<br>读锁场景： checkpoint流程。<br>写锁场景：sweep 长期不用的btree，表的 open，create, rename,drop, compact 等。|
|tiered_lock|WT_SPINLOCK|Tiered work queue spinlock|---|
|**turtle_lock**|WT_SPINLOCK|Turtle file spinlock|保护 WiredTiger.turtle，对其的读写都是串行执行。|
|**dhandle_lock**|WT_RWLOCK|Data handle list lock|保护 WT_CONNECTION 内部的  dhqh（data handle list）。<br>读锁场景：dhqh 的遍历操作，比如 WT_SESSION 查找 WT_CONNECTION 中的 data handle 并缓存到自己的结构体中。<br>写锁场景：dhqh 的修改操作，比如 WT_CONNECTION 中新建一个 data handle 并插入到自己的list 中。|

### 3.1.2 WT_DATA_HANDLE
WT 引擎中每个表都有对应的 [WT_DATA_HANDLE](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/include/dhandle.h#L62)  对象，可以理解为这个表的句柄。    

其中包含的锁有：    
|锁|类型|描述|常见的使用场景|
|:--|:--|:--|:--|
|rwlock|WT_RWLOCK|Lock for shared/exclusive ops|对 WT_DATA_HANDLE 的共享/独占 访问。<br>一般来说，共享访问的场景就是对表的普通CRUD操作；独占访问的场景是对表的 drop/rename/create等DDL 操作。|
|close_lock|WT_SPINLOCK|Lock to close the handle|关闭WT_DATA_HANDLE |

除了锁之外，WT_DATA_HANDLE 内部也维护了 [flags](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/include/dhandle.h#L118-L129)   来标记当前的使用状态：    
```
#define WT_DHANDLE_DEAD 0x001u         /* Dead, awaiting discard */
#define WT_DHANDLE_DISCARD 0x002u      /* Close on release */
#define WT_DHANDLE_DISCARD_KILL 0x004u /* Mark dead on release */
#define WT_DHANDLE_DROPPED 0x008u      /* Handle is dropped */
#define WT_DHANDLE_EVICTED 0x010u      /* Btree is evicted (advisory) */
#define WT_DHANDLE_EXCLUSIVE 0x020u    /* Exclusive access */
#define WT_DHANDLE_HS 0x040u           /* History store table */
#define WT_DHANDLE_IS_METADATA 0x080u  /* Metadata handle */
#define WT_DHANDLE_LOCK_ONLY 0x100u    /* Handle only used as a lock */
#define WT_DHANDLE_OPEN 0x200u         /* Handle is open */
```

如果采用 WT_DHANDLE_EXCLUSIVE 方式访问，在此期间会对 rwlock 加写锁。这种方式对应的场景一般是表的DDL 类操作，比如 create/drop/alter，以及 sweep 清理等操作。    
MongoServer 层创建 WT_CURSOR 进行数据读写时，底层会调用 [__wt_session_get_dhandle](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/session/session_dhandle.c#L466) ->  [__wt_session_lock_dhandle](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/session/session_dhandle.c#L103) 对 WT_DATA_HANDLE 上读锁。
用完之后，会调用 [__wt_session_release_dhandle](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/session/session_dhandle.c#L224) 释放锁。

### 3.1.3 WT_REF/WT_PAGE
对于普通的读写操作，可以共享一个 page 的访问。对于 evict 流程，涉及到 page 的逐出、分裂等流程，需要采用独占 page。    

理论上来说，可以参考 WT_CONNECTION 和 WT_DATA_HANDLE，给 page设计一个 RWLOCK 就可以搞定上述需求。      
但是需要注意的是，page 的访问非常频繁。对于一次查询请求，可能只会对 WT_CONNECTION 和 WT_DATA_HANDLE 加1 次或者若干次共享锁，但是对 page 的加锁次数可能有很多次。如果使用引用计数实现的 RWLOCK，会出现多个 CPU 频繁修改同一个内存值的情况，此时造成的 cpu cache miss 问题，会导致多核场景下的性能下降。    

因此，WT 使用了 hazard pointer （后文简称 HP）来作为 page 的读写锁，而不是 RWLOCK。在 HP 机制下，每个WT_SESSION（一般对应一个独立的线程）都维护着自己的 HP 数组，当需要使用共享方式访问某个 page 时，就在自己的 HP 数组中引用该 page 的指针 WT_REF。当 evict 流程需要独占式使用这个 page 时，需要保证没有 WT_SESSION 包含这个 page 的 HP。     
为了保证上述流程的安全性，还为每个 WT_REF 定义了多种使用状态，如下所示：     

```
#define WT_REF_DISK 0       /* Page is on disk */
#define WT_REF_DELETED 1    /* Page is on disk, but deleted */
#define WT_REF_LOCKED 2     /* Page locked for exclusive access */
#define WT_REF_MEM 3        /* Page is in cache and valid */
#define WT_REF_SPLIT 4      /* Parent page split (WT_REF dead) */
```

这里要说明的是 WT_REF_LOCKED 状态，在独占式访问时，需要将 WT_REF 设置为该状态。    
假如有 3 个线程（WT_SESSION）同时对一个 page 加锁，则流程如下（_前置条件：page在内存中，且没有被独占访问，状态为 WT_REF_MEM_）：    
|线程1（给 page 写数据，**共享**锁）|线程2（读 page 数据，**共享**锁）|EVICT_SERVER线程（逐出 page，**独占**锁|
|:--|:--|:--|
|1. 设置自己的 HP = WT_REF|1. 设置自己的 HP = WT_REF|1. 使用CAS将 WT_REF 的状态变更为LOCKED，保存之前的状态用于回滚。并先清理自己 session 的 hp 对 page 的引用。如果这一步出错，则操作回滚并返回报错|
|2. 检查 WT_REF 状态仍然是WT_REF_MEM（是否为 LOCKED）|2. 检查 WT_REF 状态仍然是WT_REF_MEM（是否为 LOCKED）|2. 检查**所有 WT_SESSION， 确保没有 HP 引用该 WT_REF**|
|3. 如果第 2 步确认不是 WT_REF_MEM，则回滚 HP=NULL, 加锁失败|3. 如果第 2 步确认不是 WT_REF_MEM，则回滚 HP=NULL, 加锁失败|3. 如果第 2 步检查失败，则加锁失败，并回滚 WT_REF 的状态|
|4. 如果第2 步是 WT_REF_MEM，则加锁成功|4. 如果第2 步是 WT_REF_MEM，则加锁成功|4. 如果第 2 步检查成功，则加锁成功|

需要注意的是，上述 3 个流程中的 第 1 步和第 2 步访问了不同的数据，是有可能乱序执行的。为了防止出现指令重排乱序执行，WT 代码中也实现了相应的 BARRIER 函数，底层会调用 mfence/lfence/sfence 内存屏障，来保证代码严格按照上表中的步骤去执行。    

在 page 使用完毕后，会有对应的 release 释放锁的流程，可以看做是 加锁的逆过程，这里就不再赘述。    

对于加共享锁的流程，可以参考 [__wt_page_in_func](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/btree/bt_read.c#L203) -> [__wt_hazard_set_func](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/support/hazard.c#L64) 的实现：    
```
int __wt_hazard_set_func(WT_SESSION_IMPL *session, WT_REF *ref, bool *busyp, const char *func, int line) {
    ...
  
    hp->ref = ref;  // 设置 hp 指针
    WT_FULL_BARRIER();  // 内存屏障, 避免设置 hp 和下面读 page state 的流程乱序
    current_state = ref->state;
    if (current_state == WT_REF_MEM) {  // page 状态正常，当前没有其他线程独占访问
        ++session->nhazard;
        WT_READ_BARRIER();
        return (0);  // 加锁成功
    }
    hp->ref = NULL;  // 否则加锁失败，重置 hp
    *busyp = true;
    return (0);
}
```

对于加独占锁的流程，可以参考 [__wt_page_release_evict](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/evict/evict_page.c#L53) -> [__wt_evict](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/evict/evict_page.c#L176) -> [__evict_exclusive](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/evict/evict_page.c#L33) -> [__wt_hazard_check](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/support/hazard.c#L293) 的实现：    

```
// __wt_page_release_evict
int __wt_page_release_evict(WT_SESSION_IMPL *session, WT_REF *ref, uint32_t flags) {
    ...
    previous_state = ref->state;
    // 当前的状态是  WT_REF_MEM， 则使用 CAS 变更为 LOCKED 状态
    locked =
      previous_state == WT_REF_MEM && WT_REF_CAS_STATE(session, ref, previous_state, WT_REF_LOCKED);
    // 尝试清理自身session 中 hp 对这个 page 的引用
    if ((ret = __wt_hazard_clear(session, ref)) != 0 || !locked) {
        if (locked)
            WT_REF_SET_STATE(ref, previous_state); // 复原状态
        return (ret == 0 ? EBUSY : ret); // 返回错误
    }
    ... 
    ret = __wt_evict(session, ref, previous_state, evict_flags);
    ...
}

// __evict_exclusive
static inline int __evict_exclusive(WT_SESSION_IMPL *session, WT_REF *ref)
{
    // 首先确保 page state 已经设置为了 LOCKED 状态
    WT_ASSERT(session, ref->state == WT_REF_LOCKED); 

    // 检查是否有 session 包含了这个 page的 hp， 如果没有，则加独占锁成功
    if (__wt_hazard_check(session, ref, NULL) == NULL)
        return (0);

    // 如果检查 hp 失败，则加锁失败
    WT_STAT_CONN_DATA_INCR(session, cache_eviction_hazard);
    return (__wt_set_return(session, EBUSY));
}

// __wt_hazard_check: 检查系统中是否有 session 的 hp 引用了指定的 page
WT_HAZARD * __wt_hazard_check(WT_SESSION_IMPL *session, WT_REF *ref, WT_SESSION_IMPL **sessionp)
{
    ...
    
    WT_ORDERED_READ(session_cnt, conn->session_cnt);
    for (s = conn->sessions, i = j = max = walk_cnt = 0; i < session_cnt; ++s, ++i) {
        if (!s->active)  // 如果 session 不是 active，则跳过
            continue;

        hazard_get_reference(s, &hp, &hazard_inuse);
        ... 
        for (j = 0; j < hazard_inuse; ++hp, ++j) {
            ++walk_cnt;
            if (hp->ref == ref) {  // 找到了引用的ref 
                WT_STAT_CONN_INCRV(session, cache_hazard_walks, walk_cnt);
                if (sessionp != NULL)
                    *sessionp = s;
                goto done;  // 返回这个 hp的指针
            }
        }
    }
    WT_STAT_CONN_INCRV(session, cache_hazard_walks, walk_cnt);
    hp = NULL;  // 没有找到引用这个 page 的 hp，返回 NULL

done:
    ...
    return (hp);
}
```

## 3.2 MVCC 并发控制
如何保证对同一行的并发读写正确执行呢？有 2 种常见的锁机制：    
- 悲观锁。锁一般分为共享锁和排他锁。先锁定资源，然后再执行更新操作。比如 `select ... for update` 就是采用这种方式。
- [乐观锁](https://www.ibm.com/docs/en/db2/11.5?topic=overview-optimistic-locking)。一般实现方式为冲突检测+数据更新。系统中会维护一行数据的多个版本（MVCC），每个版本有对应的 id 或者 timestamp 进行标志。并发的读写操作由于对应的是不同版本的数据，因此不需要加锁进行互斥，能更大程度实现高并发。如果有多个写请求同时操作同一行数据，只有 1 个写请求能够成功，其余请求会在冲突检测时出现写冲突导致失败，然后由上层发起重试。

WiredTiger 就采用了 MVCC 机制，在充分保证读并发的情形下，也带来了一些系统复杂度和负面效果，比如：冲突检测机制、写冲突和重试机制、更灵活但是也更复杂的隔离机制。

### 3.2.1 MVCC 总体概览

**内存中的 MVCC**

WiredTiger 中所有的读写操作都是在内存中进行的，page 被加载到内存中后，里面存储的 KV 对会形成一个 WT_ROW 数组。对 page 的插入操作，会有单独的 forward skip-list 结构的 WT_INSERT 链表记录；对 page 的更新、删除操作，会有单独的 forward-list 结构的 WT_UPDATE 链表记录。     

整体架构图如下：    
<p align="center">
   <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/c010f41e-2ce1-45ea-bdc6-d914b6cf6f5e" width=1000>
</p>

为了画图方便，上图只列举了 row leaf page 的关键数据结构，基于 MongoDB 4.2 内核代码。上图假设 page 中原始的数据只有 3 条，真实场景下可能很多。    

首先，对于 WT_ROW 原始数据，有一一对应的 WT_UPDATE forward-list 记录每个版本的修改，最新版本的修改在 list 头部。每个 WT_UPDATE 结构就对应了一个版本，包含了 txnid, size, state, data 等信息。而 txnid 能关联到具体的事务信息，这些信息对版本的可见性判断是必不可少的。     

其次，对于 page 中新插入的数据，有 N:N+1 对应的 WT_INSERT_HEAD* 数组来记录。WT_ROW 中的 Key 可以看做是 WT_INSERT_HEAD* 数组中每个元素之间的分割点。WT_INSERT_HEAD 维护了一个 WT_INSERT 组成的 forward skip-list，每个 WT_INSERT 表示了插入的 key，其包含的数据又维护在 WT_UPDATE 组成的 forward-list 中。

>**为什么 WT_UPDATE 使用的是 forward-list, 而 WT_INSERT 使用的是 forward skip-list?**
个人认为的原因：    
>1. 一个 Key 的 WT_UPDATE 一般不会特别多，并且会有不定期的回收策略，因此 list 长度可控。另外在版本管理上，新版本必须插入到 list 头部，判断可见性时也需要从头部顺序遍历。因此，设计成 forward-list 是最经济实用的方式。
>2. 一段 Key-Range 的 WT_INSERT 可能会比较多，而且 WT_INSERT 之间并没有什么直接联系，访问的方式也主要是 “单点查询”。因此，设计成 forward skip-list 能够快速利用二分的方式加速查找。


>**事务回滚后，对应的版本如何清理？**    
>每个 [__wt_txn](https://github.com/mongodb/mongo/blob/r4.2.25/src/third_party/wiredtiger/src/include/txn.h#L255) 事务结构中，维护了 WT_TXN_OP 链表，记录了这个事务的所有操作。如果发生回滚（[__wt_txn_rollback](https://github.com/mongodb/mongo/blob/r4.2.25/src/third_party/wiredtiger/src/txn/txn.c#L1336)），会遍历 WT_TXN_OP 链表，将所有的 WT_UPDATE 版本的 txnid 设置为 WT_TXN_ABORTED.      
后续如果其他事务对这条数据也进行了变更，或者执行 reconcile 页面回收时，会将这个版本彻底删除（参考 [__wt_update_obsolete_check](https://github.com/mongodb/mongo/blob/r4.2.25/src/third_party/wiredtiger/src/btree/row_modify.c#L287)）。     
>
>在其他数据库引擎，采用的方法并不完全相同。比如 PostgreSQL 中，当事务回滚时，会将事务状态设置为 abort，并提交 clog 日志（commit log）。**但是并不会立刻遍历每条数据执行回滚，而且已回滚的数据是可能刷入磁盘的**。当其他事务读取到这个版本时会设置相应的标志位，在执行 prune 或者 vacuum 流程时，可能才会将这条数据彻底清理。
>
>个人认为，WiredTiger 的事务回滚，理论上耗时会更长。但是这种策略减少了 clog 的维护开销。而且由于 MongoDB 的事务一般不会很大，而且 MVCC 版本一般都是在内存中，因此性能也能接受。

--- 

**硬盘上的 MVCC：History Store**    

如果 MVCC list 中的多个版本都在被使用，或者存在被访问的合法性，则 list 中的每个版本都应该被保留。如果都放在内存中是不合适的，因为内存一般比较小，承载不了太多 page 的太多版本的数据。    
因此，必须有一种策略，将内存中暂时没有访问到的 MVCC list 数据唤出到磁盘中存储（类似 swap)。    

首先想到的一种策略，就是把 MVCC list 打包之后，和 page 中的内容一起写入到用户表的 WT 文件中。后续如果再访问这个 page， 则跟随 page 中的其他数据一起读到内存中，并可以根据 timestamp 后台进行 MVCC list 的清理。    
这种策略似乎挺合理，但是无法保证短期内有读写流程一定会访问到这个 page。如果很长时间没有访问，那么 MVCC list 会一直存在 WT 文件中，一直浪费存储空间。为了解决这个问题，还要专门设计后台线程去清理每个表文件中的脏数据（类似 PostgreSQL 中的 [vacuum](https://www.interdb.jp/pg/pgsql06/01.html) 流程）。    

>在 otter tune 上有[文章](https://ottertune.com/blog/the-part-of-postgresql-we-hate-the-most)专门分析了上述 PostgreSQL MVCC 机制的缺陷，总结如下：    
>1. Version copying：即使修改了某一列数据的值，在硬盘上也要存储完整的版本拷贝。而 MySQL/Oracle 支持存储差值（git diff），MongoDB 的 WiredTiger 也支持 Update-In-Place 的策略，不过触发这个策略会有比较严格的检查，并且只在内存的 MVCC 中使用。存储差值，会在回放历史版本的时候多一些计算，但是能节省很多存储空间。
> 2. Table bloat：将历史版本和当前版本存放在同一个文件，甚至同一个 page 中，导致 IO 效率非常低下。每次 IO 以 page 为单位，可能读出来很多不需要的版本甚至一些该被清理的垃圾值（dead tuple）。另外，这些 dead tuple 也可能干扰统计分析影响查询计划。
> 3. Secondary index maintenance：PostgreSQL 中索引存储的是物理地址（page 的位置），即使更新操作没有改变索引字段对应的值，但是由于物理位置的变化也会导致索引变更，这无疑会导致一些额外的索引维护开销。文章中有一项调查发现能利用 HOT 特性不更新索引的操作只有 46%，也就是超过 50% 的更新请求都会带来额外的索引维护开销。不过，我个人认为这些开销主要是索引机制导致的，MVCC 机制不是主要原因。
> 4. Vacuum management：后台的 vacuum 流程管理机制比较复杂，针对各种复杂的业务场景，可能需要进行调优。另外 vacuum 流程也可能会因为一些长事务而阻塞。

**WT 并未采用上述策略。**

为了解决 MVCC list 的换入换出问题，从 MongoDB 4.0 版本的 WT 引入了 LAS（lookaside table）机制，从 MongoDB 4.2 版本的WT 变为了 HS（history store table） 机制。下面描述一下 HS 的工作原理。    
在 HS 机制下，WT 引擎会创建一个特殊的 history store 表，对应底层的 WiredTigerHS.wt 文件。这个表存储了 WT 引擎中所有数据的  MVCC list，每个表的每条数据的每个版本在 HS 中对应了一行数据，在 HS 中存储的 KV 形式为：Key(btreeId+recordkey+startTime+Counter) + Value(stopTime+durableTime+updateType+value)。    

引用 [WT 官方文档](http://source.wiredtiger.com/develop/arch-hs.html)上的例子，在 id 为 1000 的 btree 上，修改了key 为 “AAA” 的数据，维护了  4  个版本并全部都是提交状态：

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/08b25118-38eb-4e90-9137-3ecd94964c5d" width=400>
</p>

则 U1、U2、U3 可以换出到 HS表中，存储的数据格式如下：    

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/875d2ef4-ac9f-4c1e-95df-baf36ff0ebd9" width=900>
</p>

那么 U4 去哪里了呢？    
WT 在处理 MVCC list 时，会根据数据版本的新旧，执行不一样的策略。从旧到新列举如下：    
1. 所有事务不可见：直接将数据从内存中的 MVCC list 删除。
2. 所有事务可见或者部分事务可见的已提交版本，但又不是最新提交的版本：换出到 HS 表。如上例中的 U1,U2,U3。
3. 最新提交的版本（newest committed）：写入到用户表的 WT 文件中（disk image）。如上例中的 U4。
4. 未提交版本：继续在内存中保留。

当一个读请求来查数据时，会根据数据版本从新到旧的顺序来进行版本判断，并返回合适的数据。具体来说，顺序为：内存中的 MVCC list -> disk image 的版本 -> HS 中的版本。    

>需要注意的是，HS 从设计上杜绝了自身的数据存在 MVCC list 的情况。否则在 evict  HS-page 时，可能会出现 [HS 循环依赖](http://source.wiredtiger.com/develop/eviction.html#hs_eviction)的问题。

---

**其他部分的 MVCC**

对一个分布式数据库来说，除了上文介绍的叶子节点，MVCC 涉及到方方面面：    
1. **非叶子节点**。由于非叶子节点并不直接存储用户数据，因此它的修改主要来源是下属子节点分裂，或者自身的分裂。非叶子节点使用 [WT_PAGE_INDEX](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/include/btmem.h#L492)（本质上是一个数组） 来存储下一级的叶子节点的索引，并记录该 WT_PAGE_INDEX 的指针：    
    a). 假设非叶子节点为 P，其下属的叶子节点有 C1, C2, C3。    
    b). 假设线程 A 正在对  C3 节点执行分裂，则会获取 C3 节点的互斥锁。    
    c). 如果线程 B 此时对 P 及其叶子节点进行正常访问，可能会获取到 P 的旧版本数组，然后访问到 C3，但是会卡住并等到 C3 分裂完成。    
    d). 线程 A 对 C3 分裂完成后，修改 P 中的 WT_PAGE_INDEX ，执行指针替换，并设置 C3 节点的状态为 WT_REF_SPLIT（参考 [__split_parent](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/btree/bt_split.c#L632)）。    
    e). 线程 B 发现 C3 状态是 WT_REF_SPLIT，则进入[错误重试](https://github.com/mongodb/mongo/blob/r5.0.26/src/third_party/wiredtiger/src/btree/bt_read.c#L290)逻辑：重新执行遍历流程，通过 P 的新版本数组，正常访问到分裂后的叶子节点。     
2. **索引**。索引和表一样使用 Btree，只是存储的数据不同，因此 MVCC 机制一样。
3. **[Catalog](https://github.com/mongodb/mongo/blob/r5.0.26/src/mongo/db/catalog/README.md#two-phase-collection-and-index-drop)**。当一个线程执行了删表操作时，底层的表能被立即销毁吗？可能有更老的操作在进行快照读。因此，__mdb_catalog 元数据也有 MVCC 控制，并在合适的时机对表进行延迟删除。
4. **路由**。在分片集群中，数据可能在分片之间迁移，如何保证一致性读呢？关于路由的 MVCC 控制机制，在分片集群章节详细介绍。

--- 

**创建 Snapshot**    

MongoDB 使用 snapshot 快照来解决读写冲突。    
在开启 WiredTiger 事务时，会创建一个 snapshot 快照（参考 [__wt_txn_begin](https://github.com/mongodb/mongo/blob/r4.2.25/src/third_party/wiredtiger/src/include/txn.i#L801)->[__wt_txn_get_snapshot](https://github.com/mongodb/mongo/blob/r4.2.25/src/third_party/wiredtiger/src/txn/txn.c#L148)）。     
创建 Snapshot 的主要流程是确定 snapshot_min 、snapshot_max 以及在此区间内的活跃事务列表。       
并且会对活跃事务列表进行排序，方便后续查找，参考 [__txn_sort_snapshot](https://github.com/mongodb/mongo/blob/r4.2.25/src/third_party/wiredtiger/src/txn/txn.c#L95)。

### 3.2.2 MVCC 下的读
读请求读取一行数据时，会从头遍历 WT_UPDATE list, 返回第一个对其可见的版本。    
具体参考 [__wt_txn_read](https://github.com/mongodb/mongo/blob/r4.2.24/src/third_party/wiredtiger/src/include/txn.i#L761) 的实现（如何调用到这个函数，请参考附录中的堆栈信息）：
```
/*
 * __wt_txn_read --
 *     Get the first visible update in a list (or NULL if none are visible).
 */
static inline int
__wt_txn_read(WT_SESSION_IMPL *session, WT_UPDATE *upd, WT_UPDATE **updp)
{
    static WT_UPDATE tombstone = {.txnid = WT_TXN_NONE, .type = WT_UPDATE_TOMBSTONE};
    WT_VISIBLE_TYPE upd_visible;
    uint8_t type;
    bool skipped_birthmark;

    *updp = NULL;

    type = WT_UPDATE_INVALID; /* [-Wconditional-uninitialized] */
    // 遍历 WT_UPDATE list 链表
    for (skipped_birthmark = false; upd != NULL; upd = upd->next) {
        WT_ORDERED_READ(type, upd->type);

        /* Skip reserved place-holders, they're never visible. */
        if (type != WT_UPDATE_RESERVE) {
            upd_visible = __wt_txn_upd_visible_type(session, upd);
            // 找到可见的版本，返回结果
            if (upd_visible == WT_VISIBLE_TRUE)
                break;
            if (upd_visible == WT_VISIBLE_PREPARE)
                return (WT_PREPARE_CONFLICT);
        }
        /* An invisible birthmark is equivalent to a tombstone. */
        if (type == WT_UPDATE_BIRTHMARK)
            skipped_birthmark = true;
    }

    if (upd == NULL && skipped_birthmark) {
        upd = &tombstone;
        type = upd->type;
    }

    *updp = upd == NULL || type == WT_UPDATE_BIRTHMARK ? NULL : upd;
    return (0);
}
```

[__wt_txn_visible](https://github.com/mongodb/mongo/blob/r4.2.24/src/third_party/wiredtiger/src/include/txn.i#L685) 根据 WT_UPDATE 中的 txnid 和 timestamp，判断是否对当前的 txn 可见，包含 2 层判断：    
- txnid 可见
- timestamp 可见

代码实现：    
```
/*
 * __wt_txn_visible --
 *     Can the current transaction see the given ID / timestamp?
 */
static inline bool
__wt_txn_visible(WT_SESSION_IMPL *session, uint64_t id, wt_timestamp_t timestamp)
{
    WT_TXN *txn;

    txn = &session->txn;

    // 事务 ID 判断
    if (!__txn_visible_id(session, id))
        return (false);

    /* Transactions read their writes, regardless of timestamps. */
    if (F_ISSET(&session->txn, WT_TXN_HAS_ID) && id == session->txn.id)
        return (true);

    /* Timestamp check. */
    if (!F_ISSET(txn, WT_TXN_HAS_TS_READ) || timestamp == WT_TS_NONE)
        return (true);

    // timestamp 判断
    return (timestamp <= txn->read_timestamp);
}
```

具体来说，[__txn_visible_id](https://github.com/mongodb/mongo/blob/r4.2.24/src/third_party/wiredtiger/src/include/txn.i#L635) 根据 WT_UPDATE 中的 txnid，判断是否对当前的 txn 可见：    
```
/*
 * __txn_visible_id --
 *     Can the current transaction see the given ID?
 */
static inline bool
__txn_visible_id(WT_SESSION_IMPL *session, uint64_t id)
{
    WT_TXN *txn;
    bool found;

    txn = &session->txn;

    /* Changes with no associated transaction are always visible. */
    if (id == WT_TXN_NONE)
        return (true);

    /* Nobody sees the results of aborted transactions. */
    if (id == WT_TXN_ABORTED)
        return (false);

    /* Read-uncommitted transactions see all other changes. */
    if (txn->isolation == WT_ISO_READ_UNCOMMITTED)
        return (true);

    /*
     * If we don't have a transactional snapshot, only make stable updates visible.
     */
    if (!F_ISSET(txn, WT_TXN_HAS_SNAPSHOT))
        return (__txn_visible_all_id(session, id));

    /* Transactions see their own changes. */
    if (id == txn->id)
        return (true);

    /*
     * WT_ISO_SNAPSHOT, WT_ISO_READ_COMMITTED: the ID is visible if it is not the result of a
     * concurrent transaction, that is, if was committed before the snapshot was taken.
     *
     * The order here is important: anything newer than the maximum ID we saw when taking the
     * snapshot should be invisible, even if the snapshot is empty.
     */
    // 大于等于 snap_max 的都不可见
    if (WT_TXNID_LE(txn->snap_max, id))
        return (false);
    // 小于 snap_min 的都可见
    if (txn->snapshot_count == 0 || WT_TXNID_LT(id, txn->snap_min))
        return (true);

    // 如果位于 [snap_min, snap_max] 之间，则进行二分查找。如果找到就不可见（因为不能看到活跃事务），否则可见
    WT_BINARY_SEARCH(id, txn->snapshot, txn->snapshot_count, found);
    return (!found);
}
```

### 3.2.3 MVCC 下的写
插入、删除和更新操作，最终调用 __wt_row_modify 来执行。    

以 update 操作为例，最重要的 2 步操作为：      
1.使用 [__wt_txn_update_check](https://github.com/mongodb/mongo/blob/r4.2.24/src/third_party/wiredtiger/src/include/txn.i#L988) 检查要修改的数据是否能被修改，即本次操作是否和其他操作存在冲突，如果存在冲突则返回“写冲突”报错，而不是继续更新。    
```
/*
 * __wt_txn_update_check --
 *     Check if the current transaction can update an item.
 */
static inline int
__wt_txn_update_check(WT_SESSION_IMPL *session, WT_UPDATE *upd)
{
    WT_TXN *txn;
    WT_TXN_GLOBAL *txn_global;
    bool ignore_prepare_set;

    txn = &session->txn;
    txn_global = &S2C(session)->txn_global;

    if (txn->isolation != WT_ISO_SNAPSHOT)
        return (0);

    if (txn_global->debug_rollback != 0 &&
      ++txn_global->debug_ops % txn_global->debug_rollback == 0)
        return (__wt_txn_rollback_required(session, "debug mode simulated conflict"));
    /*
     * Always include prepared transactions in this check: they are not supposed to affect
     * visibility for update operations.
     */
    ignore_prepare_set = F_ISSET(txn, WT_TXN_IGNORE_PREPARE);
    F_CLR(txn, WT_TXN_IGNORE_PREPARE);
    // 从头遍历 WT_UPDATE list
    // 1. 要么 WT_UPDATE 可见，则本次校验通过，不需要再检查 list 后面的 WT_UPDATE 的可见性
    // 2. 要么 WT_UPDATE 不可见，但是是 ABORTED 状态，此时继续检查 list 后面的 WT_UPDATE
    // 3. 如果不满足上述 2 个条件，说明有其他流程再更新这一行数据。此时报错写冲突 
    for (; upd != NULL && !__wt_txn_upd_visible(session, upd); upd = upd->next) {
        if (upd->txnid != WT_TXN_ABORTED) {
            if (ignore_prepare_set)
                F_SET(txn, WT_TXN_IGNORE_PREPARE);
            WT_STAT_CONN_INCR(session, txn_update_conflict);
            WT_STAT_DATA_INCR(session, txn_update_conflict);
            return (__wt_txn_rollback_required(session, "conflict between concurrent operations"));
        }
    }

    if (ignore_prepare_set)
        F_SET(txn, WT_TXN_IGNORE_PREPARE);
    return (0);
}
```
2.生成 WT_UPDATE 结构，并使用 CAS 方式插入到 forward-list 头部。如果成功插入则更新成功；如果在此期间链表头节点变了，说明其他请求成功更新，则本次操作失败。    
```
/*
 * __wt_update_serial --
 *     Update a row or column-store entry.
 */
static inline int
__wt_update_serial(WT_SESSION_IMPL *session, WT_PAGE *page, WT_UPDATE **srch_upd, WT_UPDATE **updp,
  size_t upd_size, bool exclusive)
{
    WT_DECL_RET;
    WT_UPDATE *obsolete, *upd;
    wt_timestamp_t obsolete_timestamp;
    uint64_t txn;

    /* Clear references to memory we now own and must free on error. */
    upd = *updp;
    *updp = NULL;

    /*
     * All structure setup must be flushed before the structure is entered into the list. We need a
     * write barrier here, our callers depend on it.
     *
     * Swap the update into place. If that fails, a new update was added after our search, we raced.
     * Check if our update is still permitted.
     */
    // 对 WT_UPDATE list 的头部进行 CAS 更新
    while (!__wt_atomic_cas_ptr(srch_upd, upd->next, upd)) {
        // 如果 CAS 失败了，说明有其他流程已经更新了 list 头部
        // 则重新进行 update_check:
        // 1. update_check 失败，则本次流程直接失败退出
        // 2. update_check 成功，更新 upd->next 为当前的队头，然后发起下一轮 CAS 重试
        if ((ret = __wt_txn_update_check(session, upd->next = *srch_upd)) != 0) {
            /* Free unused memory on error. */
            __wt_free(session, upd);
            return (ret);
        }
    }

    ...

    return (0);
}
```

MVCC 机制采用谁先更新谁成功的策略（first-update-win，不一定是 commit），其余的写操作会返回写冲突错误，然后依赖上层代码重试（自动更新 snapshot ，然后重新发起请求）。       
如果上层同时修改相同的数据，会有非常多的写冲突，并大量浪费系统资源。在业务侧进行架构设计时，应该尽量避免这种情况。       
MongoDB 的 ServerStatus 命令可以查看写冲突的统计情况，shell 命令如下：
```
db.serverStatus().metrics.operation.writeConflicts
```

# 4. 后记
在 5.0 内核版本开始，充分利用快照读的特性，减少对 DB 和 Collection 的加锁操作。    
参考[官方文档](https://github.com/mongodb/mongo/blob/master/src/mongo/db/catalog/README.md#consistent-data-view-with-operations-running-nested-lock-free-reads)：    

<p align="center">
  <img src="https://github.com/user-attachments/assets/c6b3387f-5663-48ff-a681-ff73c158d209" width=800>
</p>

# 5. 附录
增加一些函数调用的堆栈信息，方便大家快速梳理代码。    
读多版本，调用 __wt_txn_read 的堆栈信息：    

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/b256ae88-6f9a-4efb-b690-df45466d8953" width=800>
</p>

写多版本，插入操作，调用 __wt_row_modify 的堆栈信息：

<p align="center">
  <img src="https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/8a76ee17-683a-4908-876e-f121d97c4bb6" width=800>
</p>

# 6. 参考文档
1. https://mp.weixin.qq.com/s/FxaUhtRho5YOpCFgmk1Mdg
2. https://mongoing.com/archives/74865
3. https://zhuanlan.zhihu.com/p/513635278
4. https://mongoing.com/archives/82187
5. https://zhuanlan.zhihu.com/p/31664488
6. https://www.jianshu.com/p/d2ac26ca6525
7. https://www.ibm.com/docs/en/db2/11.5?topic=overview-optimistic-locking
8. https://github.com/mongodb/mongo/blob/r4.2.24/src/third_party/wiredtiger
9.  https://github.com/mongodb/mongo/tree/r5.0.26
10. https://zhuanlan.zhihu.com/p/641668739
11. http://source.wiredtiger.com/develop/eviction.html#hs_eviction
12. http://source.wiredtiger.com/develop/arch-hs.html
13. https://www.interdb.jp/pg/index.html
14. https://github.com/mongodb/mongo/blob/r5.0.26/src/mongo/db/catalog/README.md#two-phase-collection-and-index-drop
15. https://ottertune.com/blog/the-part-of-postgresql-we-hate-the-most
16. https://github.com/mongodb/mongo/blob/master/src/mongo/db/catalog/README.md#consistent-data-view-with-operations-running-nested-lock-free-reads
