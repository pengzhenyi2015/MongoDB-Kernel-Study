# BSON compare
## 不同类型的比较问题
文档数据库的魅力就在于，你可以随时增加和删除字段，并且可以随时变换数据类型。但是这也带一个问题：不同数据类型之间如何比较大小。   
比如在一个表中插入如下数据：
```
{ "_id" : "1" } // "a" 不存在
{ "_id" : "2", "a" : ISODate("2023-06-13T09:23:17.005Z") } // "a" 是 Date 类型
{ "_id" : "3", "a" : ObjectId("6488358fbb697a2e3b9eee95") } // "a" 是 ObjectId 类型
{ "_id" : "4", "a" : 1 } // "a" 是 Int 类型
{ "_id" : "5", "a" : NumberLong(2) } // "a" 是 NumberLong 类型
{ "_id" : "6", "a" : 3 } // "a" 是 Double 类型
{ "_id" : "7", "a" : [ "x", "y", 1 ] } // "a" 是 Array 类型
```
如果执行如下按 "a" 字段排序的命令，MongoDB 会如何对不同类型的值进行比较呢？
```
db.coll.find().sort({a:1})
```
在讨论 MongoDB 如何比较不同类型的值之前，我们先思考一下不同类型的比较是否有意义：  
- 有些类型之间的比较是没有意义的。比如 ISODate("2023-06-13T09:23:17.005Z") 和 [ "x", "y", 1 ] 相比，没有足够的理由来判定谁大谁小；
- 有些类型之间的比较是有必要的，典型的就是数值类型。比如 long(4) > double(3.5) > int(3) 是能被接受的，但是反过来就不行； 

但是不论不同类型的比较是否有意义，MongoDB 需要保证每次排序都要返回稳定的结果。因此，MongoDB 规定了严格的不同类型之间的大小关系。参考：https://www.mongodb.com/docs/v4.0/reference/bson-type-comparison-order/

<TODO 图>

以上图的 Array 和 Date 为例，不管 Array 多长，Date 的时间多小，Array 都是比 Date 要小的。   
另外 Int, Long, Double, Decimal 被统一归类为 Numbers 类型，Symbol 和 String 也归类为同一种类型。

## BSON compare 比较流程
BSON 的比较流程可以参考 BSONObj::woCompare 的实现（TODO src/mongo/bson/bsonobj.cpp:148）。   
基本流程为：   
1. 遍历 2 个 BSONObj 中的 BSONElement，使用 BSONElement::woCompare 对比 2 个 BSONElement 的大小；
2. 对每个 BSONElement，先比较类型，如果类型一样才使用 BSONElement::compareElements 比较值；
3. 根据类型不同，值比较的逻辑也不相同。比如 int 和 double 类型比较时，要都转成 double 进行比较；Object 这种嵌套类型会采用“递归”的方式进行比较；   

<TODO , 使用一个具体的图进行举例>

## BSON compare 的不足
从前面的分析可以看出，BSON 的比较是非常复杂而且消耗资源的。每次比较都伴随着 2 个 BSON 解析，并包含类型转换和比较过程，比如 number 类型之间的 int/long/double 转换等。   
正如 MongoDB [官方文档](https://github.com/mongodb/mongo/blob/r5.0.0/src/mongo/db/catalog/README.md#keystring)中所描述的，MongoDB 运行过程中要处理大量的 BSON 比较。比如表中有一个 {x:1, y:1} 联合索引，现在按照  {x:42.0, y:"hello"} 查询条件找目标文档，在数据量比较大的场景下，可能需要 10 多次比较。如果使用 BSON 原生的比较方式，会暴漏非常严重的性能问题：   
1. 有没有一种方式，能够将 BSON 比较转换成 memcmp 二进制比较，从而提升性能？
2. MongoDB 底层采用 KV 引擎存储索引。如果 KV 引擎还需要理解文档模型，支持 BSON 比较，必然会破坏 MongoDB 的架构设计，增加存储引擎的复杂度；

# KeyString
为了解决上述 BSON compare 的不足，KeyString 营运而生。   
KeyString 可以看作一个可以直接通过 memcmp 进行比较的 string. 它提供了一种方法 t, 将 BSON 的比较等价到 KeyString 的比较。   
比如有 BSON x 和 BSON y, 则：   
```
x < y ⇔ memcmp(t(x),t(y)) < 0
x > y ⇔ memcmp(t(x),t(y)) > 0
x = y ⇔ memcmp(t(x),t(y)) = 0
```
在介绍 KeyString 的原理之前，我们先了解 2 个概念： Ordering 和 TypeBits.   
<TODO 每个索引最多能包含 32 个字段：https://www.mongodb.com/docs/v4.0/reference/limits/#Number-of-Indexed-Fields-in-a-Compound-Index>