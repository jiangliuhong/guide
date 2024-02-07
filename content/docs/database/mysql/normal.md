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
