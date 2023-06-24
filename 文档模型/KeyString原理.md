# 1. BSON compare
## 1.1 BSON 中不同类型的比较问题
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

但是不论不同类型的比较是否有意义，MongoDB 需要保证每次排序都要返回稳定的结果。因此，MongoDB 规定了严格的不同类型之间的大小关系。参考：https://www.mongodb.com/docs/v4.0/reference/bson-type-comparison-order/, 从小到大的顺序为：
```
1. MinKey (internal type)
2. Null
3. Numbers (ints, longs, doubles, decimals)
4. Symbol, String
5. Object
6. Array
7. BinData
8. ObjectId
9. Boolean
10. Date
11. Timestamp
12. Regular Expression
13. MaxKey (internal type)
```

以上图的 Array 和 Date 为例，不管 Array 多长，Date 的时间多小，Array 都是比 Date 要小的。   
另外 Int, Long, Double, Decimal 被统一归类为 Numbers 类型，Symbol 和 String 也归类为同一种类型。

## 1.2 BSON compare 比较流程
BSON 的比较流程可以参考 BSONObj::woCompare 的实现（TODO src/mongo/bson/bsonobj.cpp:148）。   
基本流程为：   
1. 遍历 2 个 BSONObj 中的 BSONElement，使用 BSONElement::woCompare 对比 2 个 BSONElement 的大小；
2. 对每个 BSONElement，先比较类型，如果类型一样才使用 BSONElement::compareElements 比较值；
3. 根据类型不同，值比较的逻辑也不相同。比如 int 和 double 类型比较时，要都转成 double 进行比较；Object 这种嵌套类型会采用“递归”的方式进行比较；   

以 {"a" : NumberInt(1), "b" : {"c" : "1", "d": true}} 和 {"a" : NumberLong(1), "b" : {"c" : "1", "d": false}} 的比较为例，整体流程如下图所示：   
![image](https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/ed1f6c5f-aa8c-4744-bae3-2fd54ed0c1e1)
1. 比较第 1 个字段。首先比较 fieldName(可选)，相等；其次比较 ValueType, 归一化都都是 Numberic 类型， Value 值都是数值 1，也相等；
2. 比较第 2 个字段，fieldName 相同（可选），都是 "b", ValueType 都是 Object 嵌套类型，接着通过递归方式对比 Object 内部的值；
3. Object 内部第 1 个字段都是 "c", 包含的 Value 都是 "1";
4. Object 内部第 2 个字段都是 "d", 第 1 个 BSON 包含的值是 true, 第 2 个 BSON 包含的值是 false. 因此到这一步才决出 BSON1 > BSON2;

## 1.3 BSON compare 的不足
从前面的分析可以看出，BSON 的比较是非常复杂而且消耗资源的。每次比较都伴随着 2 个 BSON 解析，并包含类型转换和比较过程，比如 number 类型之间的 int/long/double 转换等。   
正如 MongoDB [官方文档](https://github.com/mongodb/mongo/blob/r5.0.0/src/mongo/db/catalog/README.md#keystring)中所描述的，MongoDB 运行过程中要处理大量的 BSON 比较。比如表中有一个 {x:1, y:1} 联合索引，现在按照  {x:42.0, y:"hello"} 查询条件找目标文档，在数据量比较大的场景下，可能需要 10 多次比较。如果使用 BSON 原生的比较方式，会暴漏非常严重的性能问题：   
1. 有没有一种方式，能够将 BSON 比较转换成 memcmp 二进制比较，从而提升性能？
2. MongoDB 底层采用 KV 引擎存储索引。如果 KV 引擎还需要理解文档模型，支持 BSON 比较，必然会破坏 MongoDB 的架构设计，增加存储引擎的复杂度；

# 2. KeyString

BSON 的比较过程比较繁琐，涉及到反序列化，类型转换等。BSON 虽然使用二进制进行存储，却不能直接使用二进制方式进行比较。  
BSON 不能直接使用 memcmp 进行二进制比较的原因至少包含以下几点：  
1. __存在干扰信息__。BSON 会将自身长度使用小端模式放在头部 4 字节，通过上面的例子可以看到，BSON 的大小和自身的长度并没有直接关系。如果直接使用二进制比较，可能会得到一个错误结果；  
2. __类型问题__。Int/long/double 等数字类型在 BSON 中都有不同的 type，不能用来比较。必须归一化成同一种 type, 放在同一个维度作比较；   
3. __顺序问题__。BSON 中多个字段的比较顺序可能有区别。比如创建了一个 {a : 1, b : 1, c : -1} 索引，a 和 b 升序，c 降序，BSON 很难处理这种二进制比较场景；   
4. __大小端问题__。BSON 中采用小端模式存储数字，这种低地址存低字节的方式并不适合按地址进行二进制比较；   
   ![大小端问题](https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/a67bf1f0-7338-4c90-a9da-4d0a629e2d4c)
5. __变长 Value 的比较问题__。String/BinData/Object/Array 这种长度不固定的类型，可能会对比较产生干扰。比如 { a : <String1>, b : <Int1>} 与 {a : <String2>, b : <Int2>} 进行比较，如果 String1 是 String2 的前缀，String2 长度更长，则进行二进制比较时可能出现 Int1 和 String2 末尾部分进行比较的情况。如果出现这种情况，很有可能产生错误的比较结果。 

为了解决上述 BSON compare 的不足，KeyString 营运而生。   
KeyString 可以看作一个可以直接通过 memcmp 进行比较的 string. 它提供了一种方法 t, 将 BSON 的比较等价到 KeyString 的比较。   
对于 BSON x 和 BSON y, 存在规则 t, 使得：   
```
x < y ⇔ memcmp(t(x),t(y)) < 0
x > y ⇔ memcmp(t(x),t(y)) > 0
x = y ⇔ memcmp(t(x),t(y)) = 0
```

## 2.1 KeyString 组织方式
KeyString 的组成方式为：   
```
字段1类型 +  字段1二进制 + 字段2类型  +  字段2二进制 + ... + <discriminator> + 结尾标识符(0x04) + <recordId>
```
相比 BSON，KeyString 在组织方式上有以下改动：  
### 1. 排除了干扰信息
不再包含 BSON 整体长度，以及 String/Array/Object 等类型的长度信息。另外在 KeyString 用于索引场景时，还会去除 fieldName 信息来节省空间。因为索引包含哪些字段，以及对应的顺序都是固定好的。下文介绍 KeyString 编码时，都按不包含 fieldName 描述；   
### 2. 增加了 discriminator 和 recordId 字段
这 2 个是可选字段，一般用于索引场景。后文在介绍 KeyString 在索引中的使用时，会详细展开介绍；  
### 3. 使用归一化类型
Int/Long/Double 等统一使用 Numberic 类型，String/Symbol 统一使用 StringLike 类型。由于 symbol 类型目前已废弃，因此后面主要分析 Numberic 的类型转换流程。另外，为了方便将 KeyString 中的数据转回到 BSON，KeyString 还维护了 [TypeBits](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.h#L64-L263) 按位存储原始类型信息，TypeBits 作为 KeyString 的元数据存储，不参与 KeyString 的二进制比较。需要注意的是并不是每一种数据类型都要再 TypeBits 中记录，只有 Int/Long/Double/String/Symbol 等存在归一化类型转换的类型才需要记录原始类型。对于 Array, BinData 等类型则不需要记录；  
### 4. 使用按位取反的方式解决字段的升降序问题
MongoDB 中使用 [Ordering](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/bson/ordering.h#L46-L91) 来表示每个字段的顺序，Ordering 本质上是一个 32 位的 unsigned int, 其中每个 bit 表示一个字段的顺序。比如 {a : 1, b : -1} 索引，Ordering 中第 1 个 bit 是 0， 表示 "a" 字段升序；第 2 个 bit 是 1，表示 "b" 字段降序，在 KeyString 编码时，会将 "b" 字段的类型和二进制内容[全部按位取反](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L1073)。另外，由于 Ordering 只能表示 32 个字段的信息，因此，MongoDB 在创建索引时，最多只能支持 32 个字段的联合索引;  
### 5. 使用大端模式表示数值
与 BSON 不同，KeyString 会对 Int/Long/Double 等数值类型进行处理之后转为大端模式存储，包括数值类型派生出的 [Timestamp, Date](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L447-L459) 等类型也是如此;  
### 6. 使用分界符处理变长类型的比较问题
对于 String/Array/Object 等变长类型，如果采用和其他类型相同的编码方式，可能出现跨字段比较的问题。比如 {"a": "abcd", "b": 1} 和  {"a": "abc", "b": 2} 比较，由于第 1 个 KeyString 的 "a" 字段多一个字符 'd', 在按字节比较时，可能出现第一个 KeyString 中 “a” 字段的 'd' 和第二个 KeyString 中 "b" 字段类型比较的情形，这样结果是不准确的 。因此，KeyString 会在这类字段的编码结尾处添加 0x00 分界符 来避免跨字段错位比较导致结果不准确的问题。示意图如下：
![跨字段比较的问题](https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/fc801cbb-66f5-44ae-ab07-640da657036f)

而如果 String 中已经包含了 0x00，则使用 0x00FF 分界符来替代。通过这种处理方式，能保证 String 的前缀值一定是小于 String 本身的。  
BinData 类型比较特别，在 KeyString 编码时，会将长度信息放在二进制内容前面，因为在 BSON 的比较规则中，[BinData 长度越长，则值越大](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/bson/bsonelement.cpp#L452-L458)。KeyString 的比较规则和 BSON 保持一致。

## 2.2 KeyString 对数值类型的编码
在所有 BSON 类型中，数值类型（Numberic）的编码是最关键最复杂的部分。很大一部分原因是 Double  采用 IEEE 754 标准的 8 字节方式表示， Double 相互直接能通过二进制比较，但是不能和 Long 直接进行二进制比较。举例如下：  
![Double 和 Long 的二进制比较](https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/5ab3373b-236b-4183-8b80-1fb8ae19c3d1)
  
除此之外，Int 占 4 字节，Long 占 8 字节，也不能直接使用原始的大端模式进行比较。  
对于这些问题，KeyString 的解决方案是：  
1. __对于不同长度的整型 Int  和 Long, 按照真实有效字节数重新划分子类型__。这里说的真实有效字节是指排除了前导 0 后剩余的字节，通过只编码真实有效字节来实现不同长度整型的比较。比如 Int(0x00001234) 和 Long(0x0000000000001234) 同属于 "2字节整型" 的子类型， KeyString 只会再对有效部分 0x1234 进行编码，这样保证上述 2 个整型的类型和值都相等。
2. __对于 Double 类型和 Int/Long 整型之间的比较，需要分情况讨论__。由于Double 能表示的范围是整型的超集，因此超出整型数值范围的 Double 可以直接通过子类型进行区分和比较；Double 和 Long 数值重合的部分，子类型应该相同，整数部分的表示也应该相同，小数部分编码在整数部分的后面。

参考[代码定义](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L82-L103)，数值子类型从小到大的顺序如下：
```
const uint8_t kNumericNaN = kNumeric + 0;
// 小于 Long 能表示的最小负数，在这个范围内， Double 和 Long 没有交集，此时可以直接通过类型区分。
const uint8_t kNumericNegativeLargeMagnitude = kNumeric + 1;  // <= -2**63 including -Inf
// 有效字节为 8 字节的负数 Long， 或者整数部分有效字节为 8 字节的负数 Double
const uint8_t kNumericNegative8ByteInt = kNumeric + 2;
// 有效字节为 7 字节的负数 Long， 或者整数部分有效字节为 7 字节的负数 Double
const uint8_t kNumericNegative7ByteInt = kNumeric + 3;
// 有效字节为 6 字节的负数 Long， 或者整数部分有效字节为 6 字节的负数 Double
const uint8_t kNumericNegative6ByteInt = kNumeric + 4;
// 有效字节为 5 字节的负数 Long， 或者整数部分有效字节为 5 字节的负数 Double
const uint8_t kNumericNegative5ByteInt = kNumeric + 5;
// 有效字节为 4 字节的负数 Long， 或者整数部分有效字节为 4 字节的负数 Double
const uint8_t kNumericNegative4ByteInt = kNumeric + 6;
// 有效字节为 3 字节的负数 Long， 或者整数部分有效字节为 3 字节的负数 Double
const uint8_t kNumericNegative3ByteInt = kNumeric + 7;
// 有效字节为 2 字节的负数 Long， 或者整数部分有效字节为 2 字节的负数 Double
const uint8_t kNumericNegative2ByteInt = kNumeric + 8;
// 有效字节为 1 字节的负数 Long， 或者整数部分有效字节为 1 字节的负数 Double
const uint8_t kNumericNegative1ByteInt = kNumeric + 9;
// (-1, 0) 之间的 Double
const uint8_t kNumericNegativeSmallMagnitude = kNumeric + 10;  // between 0 and -1 exclusive
// 0 
const uint8_t kNumericZero = kNumeric + 11;
// (0, 1) 之间的 Double
const uint8_t kNumericPositiveSmallMagnitude = kNumeric + 12;  // between 0 and 1 exclusive
// 有效字节为 1 字节的正数 Long， 或者整数部分有效字节为 1 字节的正数 Double
const uint8_t kNumericPositive1ByteInt = kNumeric + 13;
// 有效字节为 2 字节的正数 Long， 或者整数部分有效字节为 2 字节的正数 Double
const uint8_t kNumericPositive2ByteInt = kNumeric + 14;
// 有效字节为 3 字节的正数 Long， 或者整数部分有效字节为 3 字节的正数 Double
const uint8_t kNumericPositive3ByteInt = kNumeric + 15;
// 有效字节为 4 字节的正数 Long， 或者整数部分有效字节为 4 字节的正数 Double
const uint8_t kNumericPositive4ByteInt = kNumeric + 16;
// 有效字节为 5 字节的正数 Long， 或者整数部分有效字节为 5 字节的正数 Double
const uint8_t kNumericPositive5ByteInt = kNumeric + 17;
// 有效字节为 6 字节的正数 Long， 或者整数部分有效字节为 6 字节的正数 Double
const uint8_t kNumericPositive6ByteInt = kNumeric + 18;
// 有效字节为 7 字节的正数 Long， 或者整数部分有效字节为 7 字节的正数 Double
const uint8_t kNumericPositive7ByteInt = kNumeric + 19;
// 有效字节为 8 字节的正数 Long， 或者整数部分有效字节为 8 字节的正数 Double
const uint8_t kNumericPositive8ByteInt = kNumeric + 20;
// 大于 Long 能表示的最大正数，在此范围内 Long 和 Double 不再有交集，可以直接通过类型进行区分
const uint8_t kNumericPositiveLargeMagnitude = kNumeric + 21;  // >= 2**63 including +Inf
```


对于 Double 和 Long 不存在交集的部分，比如 (-1, 0), (0, 1), (-∞, min<long>), (max<long>, +∞)，由于可以直接通过类型区分 Long 和 Double 的大小，所以值比较只会存在 Double 类型内部。因此，KeyString 对这些子类型的 Double 的编码相对容易很多，只需要按照 Double 原有格式稍作处理，然后使用大端模式编码即可。具体可以参考代码中 [KeyString::_appendSmallDouble](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L887-L923) 和 [KeyString::_appendLargeDouble](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L925-L948) 的处理流程。  

对于 Double 和 Long 存在交集的部分，KeyString 对 Double 的编码需要在数据类型和整数格式上与 Long 对齐，然后再加上小数部分。由于小数部分导致 Double 的编码可能更长，因此会出现前文提到的变长类型比较错位的情况。比如 {"a" : NumberLong(129), "b" : "1"} 与 {"a": Double(129.125), "b": "2"} 转成 KeyString 进行比较，可能出现前一个值中的 "b" 字段与后一个值中 "a" 字段的小数部分进行比较的情况，这样会导致结果不准确。  
KeyString 对整数部分的编码进行了巧妙的设计，来避免跨字段错位比较的问题。还是上面的例子，我们既然已经知道 Double(129.125) 存在有效小数位，可以直接通过整数部分的编码使得 Double(129.125) 的整数部分大于 Long(129)。具体的实现方式为：  
1. 对于整型（Int/Long），[编码时左移一位](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L1042)： value << 1， 比如 129 编码成 258;  
2. 对于 Double，整数部分编码时左移一位，如果存在有效小数位，[则再加 1](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L603)，比如 129.125 的整数部分编码为: (129<<1) + 1 = 259；如果 Double 不存在有效小数位，则编码方式和 Long 相同，[只移位不加1](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L566).  

![Double 转 KeyString流程](https://github.com/pengzhenyi2015/MongoDB-Kernel-Study/assets/16788801/d8e956e7-1c8a-46d3-a8ad-a2a208a77995)


上述流程位于 [KeyString::_appendDoubleWithoutTypeBits](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L589-L607) 中，上述例子的执行情况如下，通过注释来标识每一步的结果：  
```
//integerPart= 129, fractionalBytes = 6
const size_t fractionalBytes = countLeadingZeros64(integerPart << 1) / 8;
//ctype=44
const auto ctype = isNegative ? CType::kNumericNegative8ByteInt + fractionalBytes
                                          : CType::kNumericPositive8ByteInt - fractionalBytes;
_append(static_cast<uint8_t>(ctype), invert);

// Multiplying the double by 256 to the power X is logically equivalent to shifting the
// fraction left by X bytes.
// encoding = 36345456367763456, 0x 00 00 00 00 00 20 81 00
uint64_t encoding = static_cast<uint64_t>(magnitude * kPow256[fractionalBytes]);
dassert(encoding == magnitude * kPow256[fractionalBytes]);

// Merge in the bit indicating the value has a fractional part by doubling the integer
// part and adding 1. This leaves encoding with the high 8-fractionalBytes bytes in the
// same form they'd have with _appendPreshiftedIntegerPortion(). The remaining low bytes
// are the fractional bytes left-shifted by 2 bits to make room for the DCM.
// encoding = 72937203340148736, 0x 00 00 00 00 00 20 03 01
encoding += (integerPart + 1) << (fractionalBytes * 8);
invariant((encoding & 0x3ULL) == 0);
// encoding = 72937203340148736
encoding |= dcm;
// encoding = 2097921, 0x 01 03 20 00 00 00 00 00
encoding = endian::nativeToBig(encoding);
_append(encoding, isNegative ? !invert : invert);
```

# 3. KeyString 性能
再回到我们最开始的问题，BSON 的比较太复杂导致性能不佳，所以诞生了 KeyString，那么：   
1. KeyString 的实际性能如何？
2. 相比 BSON Compare，KeyString 能有多大改善？

为了说明上述问题，我们对 KeyString 和 BSON 的比较性能进行性能测试。   

## 3.1 测试方法
1. 测试 2 条 BSON 文档，BSON compare 的平均耗时；
2. 将 BSON 文档序列化成 KeyString（不包含 fieldName）， 测试 2 个 KeyString compare 的平均耗时；

用于测试的 2 条 BSON 文档为：
```
第 1 条 BSON 文档：
{"id":"zhenyitest", "description": "test performance of BSON compare", "int": NumberInt(1234), "long": NumberLong(123456789), "double1": 1.23456789, "double2":12345678.9}
第 2 条 BSON 文档：
{"id":"zhenyitest", "description": "test performance of BSON compare", "int": NumberInt(1234), "long": NumberLong(123456789), "double1": 1.23456789, "double2":12345678.8}
```
值的注意的是，这 2 条 BSON 唯一的区别是最后一个 Double 类型的小数位不同。这样 BSON/KeyString 在比较时，会依次比较 String/Int/Long/Double 多种类型，并在最后一个 Double 类型上决一胜负。

## 3.2 测试工具
测试工具为 MongoDB [内核代码内置的 Benchmark 框架](https://github.com/mongodb/mongo/wiki/Write-Benchmark-Tests), 源于 Google Benchmark.

测试代码为：
```
BSONObj bson1 = BSON("id" << "zhenyitest"
                    << "description" << "test performance of BSON compare"
                    << "int" << int(1234)
                    << "long" << long(123456789)
                    << "double1" << double(1.23456789)
                    << "double2" << double(12345678.9));  // 只有最后的字段不同
BSONObj bson2 = BSON("id" << "zhenyitest"
                    << "description" << "test performance of BSON compare"
                    << "int" << int(1234)
                    << "long" << long(123456789)
                    << "double1" << double(1.23456789)
                    << "double2" << double(12345678.8));  // 只有最后的字段不同

void BM_BSONCompare(benchmark::State& state) {
    if (state.thread_index == 0) {
        // Nothing
    }
    for (auto keepRunning : state) {
        bson1.woCompare(bson2);
    }
    if (state.thread_index == 0) {
        // Nothing
    }
}
BENCHMARK(BM_BSONCompare)
    ->ThreadRange(1, 1);

void BM_BSON2KeyStringCompare(benchmark::State& state) {
    if (state.thread_index == 0) {
        // Nothing
    }
    for (auto keepRunning : state) {
        // bson1
        BSONObjBuilder strippedKeyValue1;
        for (const auto& elem : bson1) {
            strippedKeyValue1.appendAs(elem, ""_sd);
        }
        KeyString ks1(KeyString::Version::V1, 
                    strippedKeyValue1.done(), 
                    Ordering::make(BSONObj())/*ALL_ASCENDING*/);
        // bson2
        BSONObjBuilder strippedKeyValue2;
        for (const auto& elem : bson2) {
            strippedKeyValue2.appendAs(elem, ""_sd);
        }
        KeyString ks2(KeyString::Version::V1, 
                    strippedKeyValue2.done(), 
                    Ordering::make(BSONObj())/*ALL_ASCENDING*/);

        ks1.compare(ks2);
    }
    if (state.thread_index == 0) {
        // Nothing
    }
}
BENCHMARK(BM_BSON2KeyStringCompare)
    ->ThreadRange(1, 1);

void BM_KeyStringCompare(benchmark::State& state) {
    KeyString ks1(KeyString::Version::V1);
    KeyString ks2(KeyString::Version::V1);
    if (state.thread_index == 0) {
         // bson1
        BSONObjBuilder strippedKeyValue1;
        for (const auto& elem : bson1) {
            strippedKeyValue1.appendAs(elem, ""_sd);
        }
        ks1.resetToKey(strippedKeyValue1.done(), 
                    Ordering::make(BSONObj())/*ALL_ASCENDING*/);
        // bson2
        BSONObjBuilder strippedKeyValue2;
        for (const auto& elem : bson2) {
            strippedKeyValue2.appendAs(elem, ""_sd);
        }
        ks2.resetToKey(strippedKeyValue2.done(), 
                    Ordering::make(BSONObj())/*ALL_ASCENDING*/);
    }
    for (auto keepRunning : state) {
        ks1.compare(ks2);
    }
    if (state.thread_index == 0) {
        // Nothing
    }
}
BENCHMARK(BM_KeyStringCompare)
    ->ThreadRange(1, 1);

void BM_BSON2KeyString(benchmark::State& state) {
    if (state.thread_index == 0) {
        // Nothing
    }
    for (auto keepRunning : state) {
        BSONObjBuilder strippedKeyValue;
        for (const auto& elem : bson1) {
            strippedKeyValue.appendAs(elem, ""_sd);
        }
        KeyString ks(KeyString::Version::V1, 
                    strippedKeyValue.done(), 
                    Ordering::make(BSONObj())/*ALL_ASCENDING*/);
    }
    if (state.thread_index == 0) {
        // Nothing
    }
}
BENCHMARK(BM_BSON2KeyString)
    ->ThreadRange(1, 1);

void BM_BSON2Strip(benchmark::State& state) {
    if (state.thread_index == 0) {
        // Nothing
    }
    for (auto keepRunning : state) {
        BSONObjBuilder strippedKeyValue;
        for (const auto& elem : bson1) {
            strippedKeyValue.appendAs(elem, ""_sd);
        }
        strippedKeyValue.done();
    }
    if (state.thread_index == 0) {
        // Nothing
    }
}
BENCHMARK(BM_BSON2Strip)
    ->ThreadRange(1, 1);

void BM_StripBSON2KeyString(benchmark::State& state) {
    BSONObj stripBSON;
    if (state.thread_index == 0) {
        BSONObjBuilder strippedKeyValue;
        for (const auto& elem : bson1) {
            strippedKeyValue.appendAs(elem, ""_sd);
        }
        stripBSON = strippedKeyValue.done();
    }
    for (auto keepRunning : state) {
        
        KeyString ks(KeyString::Version::V1, 
                    stripBSON, 
                    Ordering::make(BSONObj())/*ALL_ASCENDING*/);
    }
    if (state.thread_index == 0) {
        // Nothing
    }
}
BENCHMARK(BM_StripBSON2KeyString)
    ->ThreadRange(1, 1);
```

## 3.3 结果分析
测试结果如下：
```
Run on (8 X 2992.97 MHz CPU s)
----------------------------------------------------------------------------
Benchmark                                     Time           CPU Iterations
----------------------------------------------------------------------------
BM_BSONCompare/threads:1                    173 ns        173 ns    4152481
BM_BSON2KeyStringCompare/threads:1          689 ns        688 ns     840071
BM_KeyStringCompare/threads:1                 6 ns          6 ns  122880289
BM_BSON2KeyString/threads:1                 316 ns        315 ns    2218813
BM_BSON2Strip/threads:1                     106 ns        105 ns    7145237
BM_StripBSON2KeyString/threads:1            217 ns        215 ns    3229084
```

BSON 直接比较平均耗时： __173 ns__；  
2 个 BSON 都转成 KeyString 再比较：__689 ns__  
- 其中，KeyString 的比较只需要 __6 ns__  , 但是每个 BSON 转 KeyString 需要 300+ ns  
    - 其中转换的第一步：strip BSON 的 fieldName 耗时 100+ ns  
    - 转换的第二步：BSON 转 KeyString 耗时 200+ns  

总结：  
1.  KeyString 的比较性能相比 BSON 提升了1-2 个数量级；  
2.  但是 BSON 生成 KeyString 的代价比较大。因此**适合 1 次生成多次比较的场景**，索引就是最典型的场景；  
3.  对于非一次生成多次使用的场景，直接使用 BSON 进行比较反而更合适。比如[内存排序（SORT_STAGE）比较算法](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/exec/sort.cpp#L64)就是直接使用的 BSON compare.

# 4. KeyString 在索引中的使用
## 4.1 MongoDB 索引的组织形式
MongoDB 中的索引，在 WiredTiger 存储引擎中使用 Key-Value 对的方式进行组织和存储。我们知道数据库中索引的作用，就是根据索引字段的值快速找到文档/数据行的存储位置，即建立索引字段到存储位置的映射。其中索引字段的值采用 KeyString 来表示，而存储位置则是由 RecordId 来表示。    

KeyString 在本文已经有了详细描述，这里有必要说明一下 RecordId 的概念。我们知道 MongoDB 是 schema-free 的，每个表没有明确的主键，那么 MongoDB 表在底层 KV 引擎中怎么组织呢？或者说在 KV 引擎中存储的时候 Value 是 BSON 文档，那么 Key 是什么？   

>可能有人会反驳说 `_id` 是 MongoDB 中的主键，但我认为并不准确。原因1： MongoDB 并不使用 _id 来组织表，如果直接使用 _id 字段查数据，也要先查 _id 索引，再查表，通俗一点说就是要查 2 个 Btree；原因2：有些表没有 _id 字段（比如 oplog 表）。因此，_id 索引一个特殊的索引，自带唯一属性，行为上和二级索引类似。

为了解决没有主键的问题，MongoDB 在 KV 引擎中存储数据时，为每条文档指定了一个唯一的 RecordId. MongoDB 中每个表都有独立的 RecordId 命名空间，RecordId 本质上是一个从 1 开始的 int64 整数。MongoDB 使用 (RecordId, BSON) 的 KV 对形式在 KV 引擎中存储数据。   

MongoDB 中使用索引查询数据会有 2 个阶段：  
1. __查索引__，通过索引字段的 KeyString 找到对应的 RecordId；
2. __查数据__, 根据 RecordId 找到 BSON 文档；

RecordId 可以作为一个可选项在 KeyString 存储，一般会作为后缀的形式存在。在 Keystring 的编码上，并没有直接使用 8 字节存储 RecordId 的值，而是采用了变长编码的形式。规则如下：
1. 头字节的头 3 个 bit，以及尾字节的末尾 3 个 bit, 存储了编码所需的额外字节数。由于 3 个 bit 最大能表示的数是 7，所以 RecordId 最多能占用的字节数为 7+2=9, 其中数据的有效 bit 占位为 9*8-(2*3) = 66。RecordId 最少占用的字节数为 2 字节；
2. 除去头尾各 3 个 bit 外，其余的 bit 表示了 RecordId 的 int64 值；

举例如下，看看 RecordId(259) 和 RecordId(65536) 在 KeyString 中是如何编码的：   
<TODO 图>

相关代码可以参考 [KeyString::appendRecordId](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/key_string.cpp#L387-L431)
> 为什么 RecordId 要使用变长编码，而不是直接用 8 字节表示。我认为是 RecordId 作为一个从 0 开始自增的 id, 绝大多数情况下都是不足 8 字节的，甚至不足 4 字节。变长编码应该还是为了节省空间。

结合 [官方文档](https://github.com/mongodb/mongo/blob/r5.0.0/src/mongo/db/catalog/README.md#use-in-wiredtiger-indexes) 和代码，将 MongoDB 中索引的组织形式总结如下：  

|Index type|Key|Value|参考代码|    
|:-|:-|:-|:-|    
|`_id` index|`KeyString` without `RecordId`|`RecordId` and optionally `TypeBits`| [WiredTigerIndexUnique::_insertTimestampUnsafe(r4.0.28)](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/wiredtiger/wiredtiger_index.cpp#L1344-L1407)|
|non-unique index|`KeyString` with `RecordId`|optionally `TypeBits`| [WiredTigerIndexStandard::_insert](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/wiredtiger/wiredtiger_index.cpp#L1649-L1675)|
|unique secondary index (new)|`KeyString` with `RecordId`|optionally `TypeBits`|[WiredTigerIndexUnique::_insertTimestampSafe(r4.0.28)](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/wiredtiger/wiredtiger_index.cpp#L1409-L1471)|
|unique secondary index (old)|`KeyString` without `RecordId`|`RecordId` and opt. `TypeBits`| [WiredTigerIndexUnique::_insertTimestampUnsafe(r4.0.28)](https://github.com/mongodb/mongo/blob/r4.0.28/src/mongo/db/storage/wiredtiger/wiredtiger_index.cpp#L1344-L1407) |

看到上表的内容，脑海中可能会浮现以下问题：   
1. __为什么 RecordId 要放在 Key 部分？__   
如果是根据 KeyString 找 RecordId，直观的想法是 KeyString 作为 Key，RecordId 作为 Value, 放在 KV 引擎中存储。当然可能还有 TypeBits 用于类型转换，和 RecordId 统一放在 Value 部分即可。  
但是观察 non-unique index 可以发现 RecordId 是作为 KeyString 的后缀放在一起的。原因在于，KV 引擎中的 Key 是唯一的，但是索引中的 Key 是不唯一的，比如 {a: 1} 索引，很多文档的 "a" 字段都能等于 1，因此在存储索引的时候必须通过方法进行区分。
在 MongoDB 中，由于每个文档都有独立的 RecordId，因此将 RecordId 作为后缀能够实现将非唯一的索引 Key 存放在（Key唯一）的 KV 引擎中。当然，带来的后果是在查询时需要通过__前缀匹配__来完成，因此查询时只知道索引字段，RecordId 是不知晓的。
2. __唯一索引为什么也是用 RecordId 作为后缀？__   
如果索引有唯一属性，就不存在上述第 1 个问题了。但是为什么 unique secondary index(new) 还是将 RecordId 作为 KeyString 的后缀呢？   
的确，在早期 unique secondary index(old) 的设计中，是将 RecordId 放在 Value 部分的。但是问题在于，主从同步的时候，从节点的 oplog 回放是存在乱序的，回放时的乱序可能会临时打破索引的唯一性质，所以还是面临“非唯一” 到 “唯一”（KV引擎）的映射问题。当然，回到 RecordId 后缀方式之后，索引唯一性的检查逻辑会更复杂。   
细心的同学可能发现，_id 也是（自带的）唯一索引，为啥就能把 RecordId 放在 Value 部分呢？因为 oplog 回放首先会通过 _id 进行哈希分桶，然后多个桶之间并发回放。也就是说 _id 不会存在乱序回放的问题，因此将 RecordId 挡在 Value 部分是没问题的。
## 4.2 举例说明索引的查找过程


# 5. 总结
