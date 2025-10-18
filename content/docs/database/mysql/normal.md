+++
title = '常见面试题'
date = 2023-12-04T11:53:59+08:00
draft = false

+++

# 常见面试题

### 百万级大数据分页查询

首先模拟一张100万条记录的表：

```sql
-- 创建表
CREATE TABLE IF NOT EXISTS sys_app
(
    id          varchar(32)  not null,
    name        VARCHAR(30)  NOT NULL unique,
    title       VARCHAR(200) NOT NULL,
    create_time datetime,
    update_time datetime,
    create_user varchar(32),
    update_user varchar(32),
    PRIMARY KEY (id)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- 插入数据
INSERT INTO sys_app (id, name, title, create_time, update_time, create_user, update_user)
SELECT REPLACE(UUID(), '-', ''), 
       CONCAT('App', CAST(@counter := @counter + 1 AS CHAR(10))), 
       CONCAT('Title', FLOOR(RAND() * 1000000)), 
       NOW(), 
       NOW(), 
       'admin', 
       'admin'
FROM (SELECT NULL) AS dummy
JOIN (SELECT @counter := 0) AS init
CROSS JOIN (
    SELECT a.N + b.N * 10 + c.N * 100 + d.N * 1000 + e.N * 10000 + f.N * 100000 + g.N * 1000000 AS num
    FROM (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS a
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS b
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS c
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS d
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS e
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS f
    CROSS JOIN (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) AS g
) AS nums
LIMIT 10000000;

```

执行一次深分页查询：
```
select * from sys_app where name like 'App1%' limit  999999,10 ;
```
耗时： 1 s 228 ms

优化一：基于索引再排序

利用MySQL支持ORDER操作可以利用索引快速定位部分元组,避免全表扫描

```sql
select * from sys_app where id in (select id from (select id from sys_app where name like 'App1%' order by id limit 999999,10 ) t )
```

耗时：607 ms

如果id是自增id,同时也没有额外的查询条件，也可以使用下面的方式：

```sql
select * from sys_app table WHERE id>=10000 ORDER BY id ASC LIMIT 0,20
```

对于有额外查询条件，需要对额外查询条件增加索引此方法才能生效

优化二：利用子查询/连接+索引快速定位元组的位置,然后再读取元组.

```sql
SELECT a.* FROM sys_app a JOIN (select id from sys_app limit 999999,10) b ON a.id = b.id
```
耗时：135 ms

补充总结：
- mysql数据表记录数超过几十万时，使用limit进行分页，性能会比较差
- mysql推荐使用自增id作为数据表的主键，不要使用uuid作为数据表的主键
- mysql表的索引会影响查询的默认排序，并不绝对是按主键排序
- 分布式情况下推荐使用带时间属性的自增长id(分布式自增长id算法)

参考：[https://www.jianshu.com/p/029070a3ca83](https://www.jianshu.com/p/029070a3ca83)

### MySQL 怎么知道 varchar(n) 实际占用数据的大小

MySQL 的 Compact 行格式中会用「变长字段长度列表」存储变长字段实际占用的数据大小。

### varchar(n) 中 n 最大取值为多少

我们要清楚一点，**MySQL 规定除了 TEXT、BLOBs 这种大对象类型之外，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节**。

也就是说，一行记录除了 TEXT、BLOBs 类型的列，限制最大为 65535 字节，注意是一行的总长度，不是一列。

知道了这个前提之后，我们再来看看这个问题：「varchar(n) 中 n 最大取值为多少？」

varchar(n) 字段类型的 n 代表的是最多存储的字符数量，并不是字节大小哦。

要算 varchar(n) 最大能允许存储的字节数，还要看数据库表的字符集，因为字符集代表着，1个字符要占用多少字节，比如 ascii 字符集， 1 个字符占用 1 字节，那么 varchar(100) 意味着最大能允许存储 100 字节的数据。

首先根据[MySQL行格式](../storage/#innodb-行格式)可以知道varchar(n)存储分为三部分：

- 真实数据
- 真实数据占用的字节数
- NULL 标识，如果不允许为NULL，这部分不需要

所以可以得出下面的公式：

**n 的最大值 = 65535 -（「变长字段长度列表」「NULL 值列表」所占用的字节数）**

注意，上面的仅为单字段的计算公式，**如果是多字段的话，要保证所有字段的长度 + 变长字段字节数列表所占用的字节数 + NULL值列表所占用的字节数 <= 65535**。

### 行溢出后，MySQL 是怎么处理的

如果一个数据页存不了一条记录，InnoDB 存储引擎会自动将溢出的数据存放到「溢出页」中。

Compact 行格式针对行溢出的处理是这样的：当发生行溢出时，在记录的真实数据处只会保存该列的一部分数据，而把剩余的数据放在「溢出页」中，然后真实数据处用 20 字节存储指向溢出页的地址，从而可以找到剩余数据所在的页。

Compressed 和 Dynamic 这两种格式采用完全的行溢出方式，记录的真实数据处不会存储该列的一部分数据，只存储 20 个字节的指针来指向溢出页。而实际的数据都存储在溢出页中。

### null值相关问题

#### null 值引发的坑有哪些

1. 精度丢失： 在进行count(*)或count(列)时，可能出现结果不一致的问题。count(列)自动去除结果为NULL的列，导致可能出现count(列)小于count(*)的情况。使用count带distinct同样可能引发精度缺失，例如select count(id,name)，当name为NULL时，select count(distinct id,name)的值会变小。

2. null值对比永远为false： 所有与NULL值的比较（大于、小于、等于）的结果都为false。对于NULL值的比较，必须使用IS NULL、IS NOT NULL或ISNULL函数。ISNULL函数相比其他两者效率较高。

3. null值与其他进行运算结果均为null： 在加减乘除等运算中，NULL值与其他值进行操作的结果均为NULL。使用concat函数拼接NULL值的结果也为NULL。

4. SUM引发空指针问题： 使用SUM(NULL)会导致空指针错误。

5. GROUP BY、ORDER BY不会过滤NULL值： 在使用GROUP BY或ORDER BY时，NULL值不会被过滤，可能影响结果的排序和分组。



#### 变长字段长度列表为什么逆序存放

因为「记录头信息」中指向下一个记录的指针，指向的是下一条记录的「记录头信息」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便。

「变长字段长度列表」中的信息之所以要逆序存放，是因为这样可以**使得位置靠前的记录的真实数据和数据对应的字段长度信息可以同时在一个 CPU Cache Line 中，这样就可以提高 CPU Cache 的命中率**。

同样的道理， NULL 值列表的信息也需要逆序存放。

#### 每个数据库表的行格式都有「NULL 值列表」吗？

NULL 值列表也不是必须的。

**当数据表的字段都定义成 NOT NULL 的时候，这时候表里的行格式就不会有 NULL 值列表了**。所以在设计数据库表的时候，通常都是建议将字段设置为 NOT NULL，这样可以节省 1 字节的空间（NULL 值列表占用 1 字节空间）。

#### null 值空间问题

**null 值会占用空间吗？**

会的，因为一个字段需要用 1 字节来表示「NULL 值列表」，而一个字节可以表示 8 个 NULL 值

所以占用空间为： Math.ceil((double)NULL字段个数/8)  字节。

**「NULL 值列表」是固定 1 字节空间吗？**

「NULL 值列表」的空间不是固定 1 字节的。

**一条记录有 9 个字段值都是 NULL，这时候怎么表示？**

当一条记录有 9 个字段值都是 NULL，那么就会创建 2 字节空间的「NULL 值列表」，以此类推。

### 分库分表

#### 什么是分库分表

一种数据库架构设计模式，主要用于解决由于数据量过大而导致数据库性能降低的问题。它将原来独立的数据库拆分成若干数据库组成，将数据大表拆分成若干数据表组成，使得单一数据库、单一数据表的数据量变小，从而达到提升数据库性能的目的。在生产环境中，分库分表通常包括垂直分库、水平分库、垂直分表、水平分表四种方式。

**垂直分库**是根据业务耦合性，将关联度低的不同表存储在不同的数据库。做法与大系统拆分为多个小系统类似，按业务分类进行独立划分。与"微服务治理"的做法相似，每个微服务使用单独的一个数据库。

**水平分库**则是把一个表的数据分到多个库中，每个库的结构都一样，每个库的数据都不一样，没有交集。库所有的表都没有主键，主键由每个库的库名+每个表的主键组成。

**垂直分表**是将一张宽表按列拆分成多张表的过程。原始表中的某些列被分离出来，形成新的表，而新表与原始表通过主键相互关联。这种拆分通常基于列的访问频率、数据大小或业务逻辑来进行。

**示例**：

假设有一个用户表（`user`），包含以下列：`id`、`username`、`password`、`email`、`phone`、`address`、`bio`。如果`address`和`bio`字段通常不一起访问，且包含大量文本数据，可以考虑将它们拆分到另一张表中。

拆分后可能得到两张表：

1. `user_core`：包含`id`、`username`、`password`、`email`、`phone`。
2. `user_profile`：包含`user_id`（与`user_core`的`id`对应）、`address`、`bio`。

**优点**：

* 减少IO负担，因为可以只读取需要的列。
* 某些列的数据类型或大小可能更适合单独的存储或索引策略。

**缺点**：

* 需要管理冗余的主键或外键。
* 事务处理可能变得复杂，特别是当需要在多个表之间保持数据一致性时。
* 查询可能需要跨多个表进行，增加了查询的复杂性。

**水平分表**是将一张表的数据行按照某种规则（如哈希、范围、目录等）分布到多个结构相同的表中。每个表都包含原始表的一部分数据，但表结构保持一致。

**示例**：

假设有一个订单表（`orders`），随着业务增长，该表的数据量变得非常庞大。为了分散负载和提高性能，可以将订单数据按照订单ID的范围或哈希值分布到多个表中，如`orders_001`、`orders_002`、`orders_003`等。

**优点**：

* 解决了单一表数据量过大的问题。
* 可以将不同表分布在不同的物理存储上，提高IO性能。
* 易于扩展，只需添加更多的表即可容纳更多的数据。

**缺点**：

* 事务一致性难以维护，特别是当事务涉及多个分表时。
* 跨多个表的查询和报表生成可能变得复杂和低效。
* 数据迁移、备份和恢复可能更加困难。
* 需要额外的逻辑来管理数据的分布和路由。

#### 怎样分库分表

首选需要搞懂，**为什么需要分库分表**，分库分表主要为了解决数据量过大时的数据库瓶颈问题，例如：

- 数据库资源不足：在高并发的业务场景下，大量的用户请求需要同时访问数据库，如果数据库连接数有限，就会导致部分请求无法获得数据库连接而阻塞。通过分库，可以将请求分散到不同的数据库中，从而减轻单个数据库的连接压力。
- 磁盘 IO 瓶颈：当单个数据库的数据量过大时，磁盘IO会成为性能瓶颈。通过分库分表，可以将数据分散到多个数据库和表中，从而降低单个数据库和表的磁盘IO压力。
- 检索数据耗时：对于数据量极大的单表，即使使用了索引，查询效率也可能会非常低下。分表可以解决单表数据量过大导致的查询性能问题。通过将大表拆分成多个小表，可以降低查询时需要扫描的数据量，从而提高查询效率。
- CPU 瓶颈：数据库在处理大量数据时，CPU资源也可能成为瓶颈。通过分库分表，可以将数据处理任务分散到多个数据库服务器上，从而充分利用多台服务器的CPU资源。

总体来说就是性能出现瓶颈，并且没有其他优化手段去进行优化（索引优化，增加从库等）时则需要考虑分库分表：
- 单表瓶颈：单表数据量较大，导致读写性能较慢
- 单库出现瓶颈：
  - CPU 压力过大（busy、load过大）导致读写性能较慢
  - 内存不足（缓存命中低，磁盘 IO 过高）导致读写性能较慢
  - 磁盘空间不足导致无法写入数据
  - 网络带宽不足导致读写性能较慢

对于分库分表又分多种情况：

- 只分表：
  - 单表数据量较大
  - 评估单库容量和性能是否可以支撑未来几年的业务增长
- 只分库：数据库读写压力较大，数据库出现存储性能瓶颈
- 既分库又分表：同时具备以上两种特点，单表压力大，数据库读写压力大

#### 亿级数据分库分表

1. 评估

对于一个亿级系统，首先需要进行**评估**：需要拆分为几个库、几个表；读写能力需要提升多少倍、负载需要降低多少、数据库容量需要支持未来几年的发展。

2. 确定切分策略

其次需要确定切分策略，常见的策略有：范围切分、中间表映射、hash切分

**范围切分**：根据某一个值的范围进行切分，支持水平扩展，单表大小可控。缺点是存在明显的读写偏移。

![131710295765488.png](https://jlhblog.oss-cn-beijing.aliyuncs.com/images/2024/3/131710295765488.png)

写偏移：id一般是按顺序新增，所以所有某一段时间，写数据的操作会集中在某一张表上

读偏移：一般情况下，新增的数据查下效率会较高

**中间表映射**：将所有数据库和id的值维护在一张中间表中,每次查询时，先通过中间表确认查询来源。缺点是引入了额外的表，增加了复杂度，同时中间表可能会他别大，很难保证该表的性能，这种方法仅适用一些特殊的场景。

![131710296099360.png](https://jlhblog.oss-cn-beijing.aliyuncs.com/images/2024/3/131710296099360.png)

**hash切分**：对目标key进行hash取模，从而判断数据要落到哪个库。使用hash切分，优点是数据分片比较均匀，不容易出现热点和并发访问的瓶颈。缺点是后续扩容需要迁移数据、存在跨节点查下问题。

![131710298130668.png](https://jlhblog.oss-cn-beijing.aliyuncs.com/images/2024/3/131710298130668.png)

3. 确定分表字段

分表字段选择应该尽量减少跨库跨表查询的出现。在选择分表字段时可以参考以下几点：

- 覆盖的场景要尽可能多
- 一般是多个关联表的公共字段
- 可以采用基因算法融合字段
- 多种场景下，可以不用纠结把所有字段融合，可以采用冗余表的方法

4. 进行代码改造

- 写入：单写老库 => 双写 => 单写新库
- 读取：读老库 => 部分读老、部分读新  =>  读新库
- 灰度：根据业务场景，指定一部分灰度，或者按照比例进行灰度（这里的灰度是指到新库上进行测试）

其中双写是保证增量数据在新库和老库都存在，其中方案主要有一下几点：

- 同步双写，同时写入老库和新库中，一般推荐使用aop进行实现
- 异步双写，写入老库，通过监听数据库变化（binlog）同步到新库，也可以使用同步工具，通过一定规则将数据同步到目标表

需要注意的是，在不停服进行改造的情况下，在进行存量数据同步的时候，也会进行增量同步，需要避免并发处理同一条数据，从而导致系统异常。

5. 数据一致性校验、优化、补偿

在进行数据库切换之前，需要保证新库数据完全正确，所以需要对新库进行增量数据校验、存量数据校验操作。

只有校验通过才能进行数据库切换。

6. 灰度切换

灰度切换时必须保证一下原则：
- 有问题随时可切回老库
- 灰度放量先慢后快，每次放量需要观察一段时间
- 灰度切换需要支持灵活的规则

完整的流程：
![131710313893035.png](https://jlhblog.oss-cn-beijing.aliyuncs.com/images/2024/3/131710313893035.png)

#### 分库分表引发的问题

1. 分布式唯一 ID
- UUID：本地生成，性能高；但UUID 更占用存储空间，并且不适合作为 MYSQL 主键（无序主键会导致磁盘随机 IO 较高，索引树变高）
- 雪花算法：41bit时间戳 10bit机器id 12bit序列号，每秒可生存 409 万个
- 号段模式，使用一个额外表的自增id作为分布式 id，每次读取数据看时调用数据库批量生成一批，放在缓存中进行使用

2. 分布式事务

两阶段提交：

![131710317813991.png](https://jlhblog.oss-cn-beijing.aliyuncs.com/images/2024/3/131710317813991.png)

tcc(try confirm cancel):

![131710317899451.png](https://jlhblog.oss-cn-beijing.aliyuncs.com/images/2024/3/131710317899451.png)

最终一致性：回滚、重试、监控、告警、幂等、人工核查

3. 跨库JOIN/分页查询

- 选择合适的分表字段
- 引入搜索引擎
- 分开查询，内存中聚合
- 冗余字段

4. 历史数据处理

对历史数据进行冷热处理，热数据存在mysql中，冷数据放在hbase或者tidb中


