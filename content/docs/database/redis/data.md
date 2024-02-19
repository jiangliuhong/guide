+++
title = 'Redis数据结构'
date = 2024-02-18T11:53:59+08:00
draft = false
+++

# 数据结构

## 数据类型

### 5种基本类型

5 种常见的基本类型有：String、List、Set、Zset、Hash

类型|存储的值|读写能力
---|---|---
String|字符串、整数或浮点数|对整个字符串或字符串的一部分进行操作；对整数或者浮点数进行自增或自减操作
List|链表，链表每个节点都是一个字符串|对链表的两端进行push和pop操作，读取单个或者多个元素；根据值查找或删除元素
Set|字符串的无序集合|字符串的集合，基础操作有添加、删除、获取；同时还有计算交集、并集、差集等
Hash|包含键值对的无序散列表|基本方法有添加、获取、删除单个元素方法
Zset|与Hash相同，用于存储键值对|字符串成员与浮点数之间的有序映射，元素的排列顺序由分数的大小决定；基本方法有添加、获取、删除单个元素以及根据分数范围或成员来获取元素


#### String

String是redis中最基本的数据类型，一个key对应一个value。

String类型是二进制安全的，意思是 redis 的 string 可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象。

String类型的常用操作有：

命令|简述|使用
---|---|---
GET|获取存储在给定键中的值|GET name
SET|设置存储在给定键中的值|SET name value
DEL|删除存储在给定键中的值|DEL name
INCR|将键存储的值加1|INCR key
DECR|将键存储的值减1|DECR key
INCRBY|将键存储的值加上整数|INCRBY key amount
DECRBY|将键存储的值减去整数|DECRBY key amount

更加详细的string操作参考：[https://www.redis.net.cn/order/3544.html](https://www.redis.net.cn/order/3544.html)

string的常用场景：
- 缓存：把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低mysql的读写压力
- 计数器：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源。
- session：常见方案spring session + redis实现session共享，

#### List

List是redis中的链表，在redis中是使用双端链表实现的。

使用List结构，我们可以轻松地实现最新消息排队功能（比如新浪微博的TimeLine）。

List的另一个应用就是消息队列，可以利用List的 PUSH 操作，将任务存放在List中，然后工作线程再用 POP 操作将任务取出进行执行。

List类型的常用操作有：

命令|简述|使用
---|---|---
RPUSH|将给定值推入到列表右端|RPUSH key value
LPUSH|将给定值推入到列表左端|LPUSH key value
RPOP|从列表的右端弹出一个值，并返回被弹出的值|RPOP key
LPOP|从列表的左端弹出一个值，并返回被弹出的值|LPOP key
LRANGE|获取列表在给定范围上的所有值|LRANGE key 0 -1
LINDEX|通过索引获取列表中的元素。你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。|LINDEX key index
LTRIM|让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除|LTRIM KEY_NAME START STOP
BRPOP|移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止|BRPOP LIST1 LIST2 .. LISTN TIMEOUT 

更加详细的list操作参考：[https://www.redis.net.cn/order/3577.html](https://www.redis.net.cn/order/3577.html)

使用技巧：
- 作为桟使用：lpush + lpop
- 作为队列使用：lpush + rpop
- 作为有限集合使用：lpush + ltrim
- 作为消息队列使用：lpoush + brpop

list的常用场景：
- 微博TimeLine: 有人发布微博，用lpush加入时间轴，展示新的列表信息。
- 消息队列


#### Set

Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。

Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

Set类型的常用操作有：

命令|简述|使用
---|---|---
SADD|向集合添加一个或多个成员|SADD key value
SCARD|获取集合的成员数|SCARD key
SMEMBERS|返回集合中的所有成员|SMEMBERS key member
SISMEMBER|判断 member 元素是否是集合 key 的成员|SISMEMBER key member

更加详细的set操作参考：[https://www.redis.net.cn/order/3594.html](https://www.redis.net.cn/order/3594.html)

set的常用场景：
- 标签（tag）,给用户添加标签，或者用户给消息添加标签，这样有同一标签或者类似标签的可以给推荐关注的事或者关注的人。
- 点赞，或点踩，收藏等，可以放到set中实现


#### Hash

Redis hash 是一个 string 类型的 field（字段） 和 value（值） 的映射表，hash 特别适合用于存储对象。

hash类型的常用操作有：

命令|简述|使用
---|---|---
HSET|添加键值对|HSET hash-key sub-key1 value1
HGET|获取指定散列键的值|HGET hash-key key1
HGETALL|获取散列中包含的所有键值对|HGETALL hash-key
HDEL|如果给定键存在于散列中，那么就移除这个键|HDEL hash-key sub-key1

hash 主要是和维护对象信息，使缓存数据更加直观，同时也不string更节省空间。

更加详细的hash操作参考：[https://www.redis.net.cn/order/3564.html](https://www.redis.net.cn/order/3564.html)

#### Zset

Redis 有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员。

不同的是每个元素都会关联一个 double 类型的分数。redis 正是通过分数来为集合中的成员进行从小到大的排序。


有序集合的成员是唯一的, 但分数(score)却可以重复。有序集合是通过两种数据结构实现：

- 压缩列表（ziplist）:为了提高存储效率而设计的一种特殊编码的双向链表。它可以存储字符串或者整数，存储整数时是采用整数的二进制而不是字符串形式存储。它能在O(1)的时间复杂度下完成list两端的push和pop操作。但是因为每次操作都需要重新分配ziplist的内存，所以实际复杂度和ziplist的内存使用量相关。[详细说明](../../../arithmetic/data/#压缩列表ziplist)
- 跳跃表（zSkiplist）:跳跃表的性能可以保证在查找，删除，添加等操作的时候在对数期望时间内完成，这个性能是可以和平衡树来相比较的，而且在实现方面比平衡树要优雅，这是采用跳跃表的主要原因。跳跃表的复杂度是O(log(n))。[详细说明](../../../arithmetic/data/#跳跃表zskiplist)

zset类型的常用操作有：

命令|简述|使用
---|---|---
ZADD|将一个带有给定分值的成员添加到有序集合里面|ZADD zset-key 178 member1
ZRANGE|根据元素在有序集合中所处的位置，从有序集合中获取多个元素|ZRANGE zset-key 0-1 withccores
ZREM|如果给定元素成员存在于有序集合中，那么就移除这个元素|ZREM zset-key member1

更加详细的zset操作参考：[https://www.redis.net.cn/order/3609.html](https://www.redis.net.cn/order/3609.html)