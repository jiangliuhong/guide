---
title: MySQL基础
categories: 
    - 数据库
date: 2023-05-16 10:52:08
tags:
    - MySQL
cover: https://static.jiangliuhong.top/images/picgo/20251018231638.png
desc: MySQL基础知识整理，包括数据存储、索引、事务
---

![MySQL思维导图](
https://static.jiangliuhong.top/images/my/xmind-MySQL.png)

# 数据存储

以InnoDB为例，InnoDB 是将数据存储在磁盘中，需要处理数据时，再将数据读取到内存中进行处理，对于 InnoDB 引擎会将数据划分为若干页，以页做为磁盘和内存交互的基本单位。InnoDB中页的大小一般为16KB。

在MySQL服务运行的过程中不可以修改页的大小，只能在初始化数据目录的时候指定。

## 存储的文件

假设目前存在一个数据库为`testdb`，在数据库中存在一个表为`user`，那么在MySQL 的数据目录`/var/lib/mysql`下将会存在这样的目录

```
|-- testdb
	|-- db.opt  
  |-- user.frm  
  |-- user.ibd
```

- db.opt，用来存储当前数据库的默认字符集和字符校验规则。
- t_order.frm ，t_order 的**表结构**会保存在这个文件。在 MySQL 中建立一张表都会生成一个.frm 文件，该文件是用来保存每个表的元数据信息的，主要包含表结构定义。
- t_order.ibd，t_order 的**表数据**会保存在这个文件。表数据既可以存在共享表空间文件（文件名：ibdata1）里，也可以存放在独占表空间文件（文件名：表名字.idb）。这个行为是由参数 innodb_file_per_table 控制的，若设置了参数 innodb_file_per_table 为 1，则会将存储的数据、索引等信息单独存储在一个独占表空间，从 MySQL 5.6.6 版本开始，它的默认值就是 1 了，因此从这个版本之后， MySQL 中每一张表的数据都存放在一个独立的 .idb 文件。

## 存储文件结构

**表空间由段（segment）、区（extent）、页（page）、行（row）组成**，InnoDB存储引擎的逻辑存储结构大致如图：

![表空间结构](https://static.jiangliuhong.top/images/2024/1/tablespacestruct.png)

行：数据库表中的记录都是按行（row）进行存放的，每行记录根据不同的行格式，有不同的存储结构。

页：

- 记录是按照行来存储的，但是数据库的读取并不以「行」为单位，否则一次读取（也就是一次 I/O 操作）只能处理一行数据，效率会非常低。

- **InnoDB 的数据是按「页」为单位来读写的**，也就是说，当需要读一条记录的时候，并不是将这个行记录从磁盘读出来，而是以页为单位，将其整体读入内存。

- **默认每个页的大小为 16KB**，也就是最多能保证 16KB 的连续存储空间。

区：

B+ 树中每一层都是通过双向链表连接起来的，如果是以页为单位来分配存储空间，那么链表中相邻的两个页之间的物理位置并不是连续的，可能离得非常远，那么磁盘查询时就会有大量的随机I/O，随机 I/O 是非常慢的。

解决办法就是让链表中相邻的页的物理位置也相邻，这样就可以使用顺序 I/O 了，那么在范围查询（扫描叶子节点）的时候性能就会很高。

在表中数据量大的时候，为某个索引分配空间的时候就不再按照页为单位分配了，而是按照区（extent）为单位分配。每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了。

段：表空间是由各个段（segment）组成的，段是由多个区（extent）组成的。段一般分为数据段、索引段和回滚段等。

- 索引段：存放 B + 树的非叶子节点的区的集合；
- 数据段：存放 B + 树的叶子节点的区的集合；
- 回滚段：存放的是回滚数据的区的集合

## InnoDB 行格式

行格式（row_format）：一条数据记录在磁盘上的存储结构。

InnoDB 提供了 4 种行格式，分别是 Redundant、Compact、Dynamic和 Compressed 行格式。

我们可以在创建表或者修改表的语句中指定所使用的行格式

```sql
create table 'table info ..' row_format = '行格式名称'
alter table 'table name' row_format = '行格式名称'
```

InnoDB 提供了 4 种行格式，分别是 Redundant、Compact、Dynamic和 Compressed 行格式：

1. **Compact（默认）：** COMPACT是MySQL的默认行格式，它对于读密集型的工作负载通常表现良好。它采用变长的存储方式，并且对于短字段使用前缀压缩。
2. **Redundant：** REDUNDANT行格式使用了更多的存储空间，但在某些特殊情况下可能提供更好的性能。它通常用于处理特定的事务性工作负载。
3. **Dynamic：** DYNAMIC行格式采用变长字段，但相较于COMPACT，它对于短字段的压缩更为灵活，因此在某些情况下可能会更加节省空间。
4. **Compressed：** BARRACUDA是一种文件格式，而不是行格式。在MySQL 5.6之后的版本，InnoDB引擎引入了支持动态格式的文件格式BARRACUDA。

其中Compressed与Redundant存储格式与Compact相似；Redundant是一种古老的格式，如今使用微乎其微。

Compact格式主要分为两部分：记录额外信息、记录的真是数据，如图：

![15mysql_compact.png](https://static.jiangliuhong.top/images/2024/1/15mysql_compact.png)

### 记录的额外信息

记录的额外信息包含 3 个部分：变长字段长度列表、NULL 值列表、记录头信息。

### 变长字段长度列表

在存储数据的时候，需要要把数据占用的大小存起来，存到「变长字段长度列表」里面，读取数据的时候才能根据这个「变长字段长度列表」去读取对应长度的数据。VARCHAR、TEXT、BLOB 等变长字段都是这样实现的。

首先创建一个表`t_user`并插入数据

```sql
CREATE TABLE `t_user` (
  `id` int(11) NOT NULL,
  `name` VARCHAR(20) NOT NULL,
  `phone` VARCHAR(20) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB DEFAULT CHARACTER SET = ascii ROW_FORMAT = COMPACT;

insert into t_user  values (1,'a','123',18),(2,'bb','1234',null),(3,'ccc',null,null)
```

现在t_user表的数据为：

```
id|name|phone|age|
--+----+-----+---+
 1|a   |123  | 18|
 2|bb  |1234 |   |
 3|ccc |     |   |
```

对于第一条数据而言：

- name 列的值为 a，真实数据占用的字节数是 1 字节，十六进制 0x01；
- phone 列的值为 123，真实数据占用的字节数是 3 字节，十六进制 0x03；

这些变长字段的真实数据占用的字节数会按照列的顺序**逆序存放**（[为什么要逆序存放](#为什么逆序存放)），所以「变长字段长度列表」里的内容是「 03 01」，而不是 「01 03」。

![15mysql_compact1.png](https://static.jiangliuhong.top/images/2024/1/15mysql_compact1.png)

第二条记录的行格式中，「变长字段长度列表」里的内容是「 04 02」

![15mysql_compact2.png](https://static.jiangliuhong.top/images/2024/1/15mysql_compact2.png)

第三条记录中 phone 列的值是 NULL，NULL 是不会存放在行格式中记录的真实数据部分里的，所以「变长字段长度列表」里不需要保存值为 NULL 的变长字段的长度。

![15mysql_compact3.png](https://static.jiangliuhong.top/images/2024/1/15mysql_compact3.png)

### 为什么逆序存放

当 InnoDB 存储引擎处理一行数据时，它通常是从行的末尾开始向前处理的。这是因为行的末尾通常包含了一些固定长度的信息（如行ID、事务ID等），这些信息的位置是固定的，因此可以很容易地找到。而变长字段则位于这些固定长度信息的前面。

如果按照列的顺序直接存放变长字段的长度值，那么 InnoDB 在处理变长字段时就需要从行的开头开始逐个读取长度值，直到找到目标字段的长度。这样做可能会涉及到多次的磁盘块读写操作，因为每个字段的长度值可能分散在不同的磁盘块中。

但是，如果按照列的顺序逆序存放这些长度值，InnoDB 就可以在处理变长字段时直接从行的末尾开始读取长度值。由于长度值是逆序存放的，因此目标字段的长度值会更快地被找到。这样，InnoDB 就可以更快速地定位到目标字段的数据在磁盘块中的位置，从而减少磁盘 I/O 操作的次数。

### 记录头信息

记录头信息是由固定的5个字节组成，5个字节也就是40个二进制位，不同的位代表不同的意思，这些头信息会在后面的一些功能中看到。

| 名称         | 大小（单位：bit） | 描述                                                         |
| ------------ | ----------------- | ------------------------------------------------------------ |
| 预留位1      | 1                 | 没有使用                                                     |
| 预留位2      | 1                 | 没有使用                                                     |
| delete_mask  | 1                 | 标识此条数据是否被删除。从这里可以知道，我们执行 detele 删除记录的时候，并不会真正的删除记录，只是将这个记录的 delete_mask 标记为 1 |
| min_rec_mask | 1                 | B+树的每层非叶子节点中的最小记录标记为1                      |
| n_owned      | 4                 | 当前记录拥有的记录数                                         |
| heap_no      | 13                | 索引堆中该条记录的排序记录                                   |
| record_ type | 3                 | 记录类型，000（0）表示普通记录，001（1）表示B+树非叶子节点记录，010（2）表示最小记录，011（3）表示最大记录 |
| next_record  | 16                | 下一条记录的位置。从这里可以知道，记录与记录之间是通过链表组织的。在前面我也提到了，指向的是下一条记录的「记录头信息」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便 |
| Total        | 40                |                                                              |

**其中比较重要的是：delete_mask、delete_mask、record_type**

### NULL 值列表

表中的某些列可能会存储 NULL 值，如果把这些 NULL 值都放到记录的真实数据中会比较浪费空间，所以 Compact 行格式把这些值为 NULL 的列存储到 NULL值列表中。

如果存在允许 NULL 值的列，则每个列对应一个二进制位（bit），二进制位按照列的顺序逆序排列。

- 二进制位的值为`1`时，代表该列的值为NULL。
- 二进制位的值为`0`时，代表该列的值不为NULL。

另外，NULL 值列表必须用整数个字节的位表示（1字节8位），如果使用的二进制位个数不足整数个字节，则在字节的高位补 `0`。

以上面的`t_user`数据为例：

列数据：

```
id|name|phone| age |
--+----+-----+----+
 1|a   |123  | 18 |
 2|bb  |1234 |NULL|
 3|ccc |NULL |NULL|
```

对于第一条记录，所有列都有值，不存在 NULL 值，所以用二进制来表示如下图所示：

![](https://static.jiangliuhong.top/images/2024/1/15mysql_compact_null1.png)

但是 InnoDB 是用整数字节的二进制位来表示 NULL 值列表的，现在不足 8 位，所以要在高位补 0，最终用二进制来表示是下面这样的：

![](https://static.jiangliuhong.top/images/2024/1/15mysql_compact_null2.png)

下面来看第二条记录， age 列是 NULL 值，所以，对于第二条数据，NULL值列表用十六进制表示是 0x04。

![](https://static.jiangliuhong.top/images/2024/1/15mysql_compact_null3.png)

最后第三条记录， phone 列 和 age 列是 NULL 值，所以，对于第三条数据，NULL 值列表用十六进制表示是 0x06。

![](https://static.jiangliuhong.top/images/2024/1/15mysql_compact_null4.png)

当三条记录的 NULL 值列表都填充完毕后，它们的行格式最终是这样的：

![](https://static.jiangliuhong.top/images/2024/1/15mysql_compact_null5.png)

### 记录的真实数据

记录真实数据部分除了我们定义的字段，还有三个隐藏字段，分别为：row_id、trx_id、roll_pointer

- row_id:如果我们建表的时候指定了主键或者唯一约束列，那么就没有 row_id 隐藏字段了。如果既没有指定主键，又没有唯一约束，那么 InnoDB 就会为记录添加 row_id 隐藏字段。row_id不是必需的，占用 6 个字节。
- trx_id:事务id，表示这个数据是由哪个事务生成的。 trx_id是必需的，占用 6 个字节。

- roll_pointer:这条记录上一个版本的指针。roll_pointer 是必需的，占用 7 个字节。

![16mysql_compact_real.png](https://static.jiangliuhong.top/images/2024/1/16mysql_compact_real.png)

其中trx_id 和 roll_pointer主要在MVCC 机制中起作用：[MVCC机制](../trans#mvcc机制)

### 行溢出处理

MySQL 中磁盘和内存交互的基本单位是页，一个页的大小一般是 `16KB`，也就是 `16384字节`，而一个 varchar(n) 类型的列最多可以存储 `65532字节`，一些大对象如 TEXT、BLOB 可能存储更多的数据，这时一个页可能就存不了一条记录。这个时候就会**发生行溢出，多的数据就会存到另外的「溢出页」中**。

如果一个数据页存不了一条记录，InnoDB 存储引擎会自动将溢出的数据存放到「溢出页」中。在一般情况下，InnoDB 的数据都是存放在 「数据页」中。但是当发生行溢出时，溢出的数据会存放到「溢出页」中。

当发生行溢出时，在记录的真实数据处只会保存该列的一部分数据，而把剩余的数据放在「溢出页」中，然后真实数据处用 20 字节存储指向溢出页的地址，从而可以找到剩余数据所在的页。大致如下图所示。

![16mysql_row.png](https://static.jiangliuhong.top/images/2024/1/16mysql_row.png)

上面这个是 Compact 行格式在发生行溢出后的处理。

Compressed 和 Dynamic 这两个行格式和 Compact 非常类似，主要的区别在于处理行溢出数据时有些区别。

这两种格式采用完全的行溢出方式，记录的真实数据处不会存储该列的一部分数据，只存储 20 个字节的指针来指向溢出页。而实际的数据都存储在溢出页中，看起来就像下面这样：

![16mysql_row2.png](https://static.jiangliuhong.top/images/2024/1/16mysql_row2.png)


> 参考链接：
>
>  [MySQL的null值是怎么存储的?](https://www.cnblogs.com/xiaolincoding/p/16941244.html)
>
> [MySQL 的 NULL 值是怎么存放的?](https://www.51cto.com/article/771121.html)


# 索引概念

## 索引是什么

索引是对数据库表中一列或多列的值进行排序的一种数据结构，能实现快速定位数据的一种存储结构，其设计思想是以空间换时间。

在关系型数据库中，索引是一种单独的、物理的对数据库表中的一列或者多列的值进行排序的一种存储结构，它是某个表中一列或若干列值的集合和相应的指向表中物理标识，这些值的数据页的逻辑指针清单。索引的作用相当于图书的目录，可以根据目录中的页码快速找到所需的内容。

### 索引的分类

按**数据结构**分类：

- B+tree索引
- Hash索引
- Full-text 索引

按**物理存储**分类：

- 聚簇索引（主键索引）
- 二级索引（辅助索引）

按**字段特性**分类：

- 主键索引
- 唯一索引
- 普通索引
- 前缀索引

按**字段个数**分类：

- 单列索引
- 联合索引

#### 唯一索引

唯一索引和普通索引类似，主要区别在于，唯一索引限制列的值必须唯一，但允许存在空值（只能有一个）。主键索引不允许有空值。

#### 全文索引

在执行模糊查询的时候，如`like "value%"`，这种情况下，需要考虑使用全文搜索的方式进行优化。全文搜索在MySQL中是一个FULLTEXT类型索引。全文索引主要用来查找文本中的关键字，而不是直接与索引中的值进行比较，它更像是一个搜索引擎，而不是简单的where语句的参数匹配。目前只有char/vachar/text列上可以创建全文索引，默认Mysql不支持中文全文搜索。Mysql全文搜索只是一个临时方案，对于全文搜索场景，更专业的做法是使用全文搜索引擎，如ElasticSearch。

#### Hash 索引

Hash索引是一种基于哈希算法的索引类型。它通过将索引键值通过哈希函数转换为固定长度的哈希码，然后将哈希码映射到实际存储位置。Hash索引适用于等值查询，例如在WHERE子句中使用`=`条件。Hash索引适用于等值查询的场景，但由于哈希碰撞（不同键值得到相同的哈希码）可能导致性能下降，因此在某些情况下不如B树索引。

#### 组合索引

组合索引是指在多个列上创建的索引，这样可以更有效地支持多列的查询条件。组合索引按照索引的列顺序建立，从左到右，左侧列的顺序性更强。当查询中涉及多个列作为查询条件时，组合索引能够更好地提高查询性能。但要注意，组合索引的列顺序要考虑到查询频率较高的列放在前面。

#### 聚簇索引与非聚簇索引

聚簇索引是一种特殊的索引，它决定了数据表中数据的物理排列顺序，使得索引和数据行保存在一起。InnoDB存储引擎中的主键索引就是聚簇索引。

与聚簇索引相对应的是非聚簇索引。非聚簇索引中索引和数据行是分开存储的，索引仅包含指向实际数据行的指针。

需要注意的是，当查询列不在非聚簇索引上时，会引发回表。

## MySQL 索引机制

### 为什么InnoDB要使用 B+ 树，而不是 B 树

首先，由于索引本身数据量大，所以只能以索引文件的形式存储在磁盘上，也就导致每次读取索引都会产生磁盘 I/O 消耗，所以选用的数据结构能获取更多的信息并且 I/O 消耗更低就尤为重要。

#### B 树概念

B 树是一种自平衡的二叉树，它维护有序数据并允许对树进行搜索、顺序访问、插入和删除。它是二叉搜索树的一种演化，在 B 树中，一个父节点可以有多个子节点。

B 树是一种平衡的多分树，通常我们说 m 阶的 B 树，他必须满足如下条件：

- 每个节点最多有 m 个子节点
- 每个非叶子节点（除去根节点）具有至少 m/2 个子节点
- 根节点至少有两个子节点
- 具有 k 个子节点的非叶子节点包含 k-1 个键

![B树结构](https://static.jiangliuhong.top/images/2023/12/131702451517376.png)

B 树的阶，指的是 B 树中节点的子节点数目的最大值。例如在上图的书中，「13,16,19」拥有的子节点数目最多，一共有四个子节点（灰色节点）。所以该 B 树的阶为 4，该树称为 4 阶 B 树。在实际应用中，B 树应用于 MongoDb 的索引。

#### B+ 树概念

B+ 树是由 B 树演变的，是使用文件系统使用的数据结构：
- 有 m 个子树的中间节点包含有 m 个元素，每个元素不保持数据，只作为索引使用。
- 所有的叶子节点中包含了关键字的信息，以及这些关键字记录的指针，并且叶子节点本身按照关键字的大小从大到小的顺序排列。

![B+树结构](https://static.jiangliuhong.top/images/2023/12/1416181905626579.jpg)

与 B 树相比，B+ 树的有点为：

- B+ 树的磁盘读写代驾更低：B+ 树的内部节点并没有指向关键字的指针信息，所以内部节点所使用的空间更小，对于相同大小能存放的关键字信息就更多，所以一次读入内存的关键字也就更多，从而减少 I/O 次数。
- B+ 树查询效率更加稳定：由于 B+ 非终节点并不实际指向文件内容，只是存储叶子节点的关键字索引，所以 B+ 树中任何关键字的查询必须从根节点查询到叶子节点，所有关键字的查询的遍历层级是相同的，也就是导致数据查询效率相当。
- B+ 树更适合用于范围查找：对于遍历，B+ 树只需要遍历叶子节点就可以实现整棵树的遍历。

总结：

- B+树是一棵平衡树，每个叶子节点到根节点的路径长度相同，查找效率较高
- B+树的所有关键字都在叶子节点上，因此范围查询时只需要遍历一遍叶子节点即可
- B+树的叶子节点都按照关键字大小顺序存放，因此可以快速地支持按照关键字大小进行排序
- B+树的非叶子节点不存储实际数据，因此可以存储更多的索引/数据；
- B+树的非叶子节点使用指针连接子节点，因此可以快速地支持范围查询和倒序查询
- B+树的叶子节点之间通过双向链表链接，方便进行范围查询。

#### B+ 树与其他数据结构对比

与B 树：

- B 树非叶子节点也要存储数据，相同磁盘 IO，B+树能查询更多节点
- B+树双向链表适合范围查询，B 树的中序遍历会更麻烦

二叉树：

- B+树查询时间复杂度为logdN,二叉树为logN
- B+树存储千万数据也需要二～三层，进行二～三次 IO，二叉树则 IO 更多

Hash：

- Hash 无法进行范围查询
- Hash维护索引成本更低

红黑树：

- 红黑树是一种二叉搜索树，每个节点最多只能包含两个子节点
- B树是一种多路搜索树，它的每个节点可以包含多个键值和子节点
- 红黑树更适用于实现集合和映射等数据结构，以及其它查找频繁的场景
- B树更适用于实现数据库索引等需要频繁插入和删除操作的场景



#### 高度为 3 的 B+ 树能存多少数据

TODO

### InnoDB 与 MyISAM 引擎下的索引区别

#### Innodb 中 B+ 树是如何产生的



TODO



#### Innodb 是如何支持范围查找能走索引的



TODO



### 索引下推

索引下推（ICP）是 MySQL5.6 针对扫描二级索引的一项优化改造。通过把索引过滤条件下推到存储引擎，来减少 MySQL 存储引擎访问基表的次数以及 MySQL 服务层访问存储引擎的次数。ICP 适用于 MYISAM 和 INNODB 引擎。

#### 认识mysql架构



![MySQL 架构](https://static.jiangliuhong.top/images/2024/1/21704179867555.png)

- MySQL 服务层：也就是 SERVER 层，用来解析 SQL 的语法、语义、生成查询计划、接管从 MySQL 存储引/擎层上推的数据进行二次过滤等等。
- MySQL 存储引擎层：按照 MySQL 服务层下发的请求，通过索引或者全表扫描等方式把数据上传到MySQL 服务层。
- MySQL 索引扫描：根据指定索引过滤条件，遍历索引找到索引键对应的主键值后回表过滤剩余过滤条件。
- MySQL 索引过滤：通过索引扫描并且基于索引进行二次条件过滤后再回表。



#### 索引下推的作用

**作用：减少回表次数**

现在以一个例子展示索引下推：

```sql
-- 创建表
create table user (
 id int primary key comment 'id' ,
 name varchar(20) comment '姓名',
 age int comment '年龄',
 card int comment '身份证',
 key idx_name_age (name,age)
)engine=InnoDB default charset=utf8mb4;
-- 插入数据
insert into user values (1,'李四',18,1),(2,'李五',20,2),(3,'王五',23,3),(4,'张三',30,4);
```

查询执行计划：

```sql
explain select * from user where name like '李%' and age >= 18;
-- 结果为：Using where
```

设置索引下推

```sql
SET optimizer_switch='index_condition_pushdown=on';
```

再次查询执行记录

```sql
explain select * from user where name like '李%' and age >= 18;
-- 结果为：Using index condition
```

从索引计划可以看出，执行计划打印为`Using index condition`则代表使用了索引下推。

假设执行sql为`select * from user where name like '李%' and age = 18`，通过下面的图可以很明显的看见两种情况下的查询逻辑

未使用索引下推时的查询：

![未使用索引下推时的查询](https://static.jiangliuhong.top/images/2024/1/31704273987720.png)

使用索引下推时的查询：

![使用索引下推时的查询](https://static.jiangliuhong.top/images/2024/1/31704274030853.png)

命令总结：

```sql
＃ 查看索引下推是否开启
select @@optimizer_switch
#开启索引下推
SET optimizer_switch='index_condition_pushdown=on';
# 关闭索引下推
SET optimizer_Switch="index_condition_pushdown=off";
```

#### 索引下推的使用条件

- 索引下推的目标是减少全行记录读取，从而减少 IO 操作，只能用于非聚簇索引。（聚簇索引本身已包含行数据，不存在回表）
- 只能用于 `range`、`ref`、`eq_ref`、`ref_or_null`等操作
- where条件中使用and的时候（or为排除记录，不需要查询行数据）
- 适用于分区表
- 不支持在虚拟列上建立索引（例如：函数索引）
- 不支持引用子查询作为查询条件
- 不支持存储函数作为条件，因为在存储引擎中无法调用存储函数

### 索引排序内部流程

## 索引失效

### 什么情况下索引失效

在MySQL8中，索引失效的场景有：

- `like`查询左边带`%`时可能会失效
- 隐式类型转换，即索引字段与查询条件或关联字段类型不一致，MySQL会对其进行类型转换，从而导致索引失效
- `where`条件中对索引列使用运算符或函数会导致失效
- 使用`OR`查询，并且存在非索引时会导致失效
- 使用` IN `查询可能会导致索引失效，在 MySQL 中，通过环境变量`eq_range_index_dive_limt`的值从而影响 `IN`查询，在 MySQL8 中，当该值 为 200 时，使用 IN 查询的条件个数大于 200则不会走索引
- 使用非主键进行范围查询时，可能会失效
- 使用`order by`可能会导致失效
- `is null`、`is not null` `≠`可能会失效

#### 字段为 null 索引是否失效

在MySQL中，对于字段为null的情况，索引并不会失效，而是涉及到优化器的选择。MySQL会考虑走索引与不走索引的成本，并在执行查询时选择最优的执行计划。

对于字段为null的情况，使用`is null`、`is not null`或 `≠`条件，索引仍然可以被利用。优化器会计算索引扫描的成本以及回表操作的成本。如果走索引扫描的效率高于全表扫描，优化器将选择使用索引扫描，然后进行回表操作。

需要注意的是，如果结果列的大小相对于行数量较小，优化器更倾向于执行索引扫描。这是因为索引扫描后再回表的成本相对较低。反之，如果结果列数量较大，那么索引扫描后再回表的效率可能远低于全表扫描，此时优化器可能选择不使用索引。

因此，索引对于字段为null的情况并不失效，而是在优化器根据具体情况进行智能选择，以提高查询性能。

首先`is null`、`is not null` `≠`都是可以走索引的，在MySQL中，MySQL会计算走索引与不走索引的成本，因为如果走索引扫描，那么必然会存在回表操作，MySQL 会计算结果列的大小，如果结果列远低于行数量，那么优化器就会执行索引扫描，然后再回表查询数据，反之，如果结果列数量较大，那么索引扫描后再回表的效率就远低于全表扫描。

关于mysql null值更详细的说明：[null值的存储](../storage#null值的存储)

#### LIKE 索引失效问题

首先索引的数据结构是B+树，在 B+ 树中，数据是有序的，从下图中可以看出 4 个aba -> abb -> abc -> abc -> abe 是有序排列的，当输入条件为 like 'a%'时，在 B+ 树中是有序查找，所以like前模糊匹配是可以走索引的，但如果缓存后模糊匹配，由于结尾并不是有序排列的，所以此时索引会失效。

![151705284761044.png](https://static.jiangliuhong.top/images/2024/1/151705284761044.png)

但是在某些特定情况，前模糊匹配也可能失效：

```sql
-- 创建表
create table user (
 id int primary key comment 'id' ,
 name varchar(20) comment '姓名',
 card int comment '身份证',
 key idx_name (name)
)engine=InnoDB default charset=utf8mb4;
-- 插入数据
insert into user values (1,'李四',1),(2,'李五',2),(3,'王五',3),(4,'张三',4);

-- 创建表
create table user_exp(
	id int primary key comment 'id',
  name varchar(20) comment '姓名',
  key idx_name (name)
)engine=InnoDB default charset=utf8mb4;
-- 插入数据
insert into user_exp values (1,'李四'),(2,'李五'),(3,'王五'),(4,'张三');
```

执行：

```sql
explain select * from user where name like '%五';
```

打印结果为：

```
id|select_type|table|partitions|type|possible_keys|key|key_len|ref|rows|filtered|Extra      |
--+-----------+-----+----------+----+-------------+---+-------+---+----+--------+-----------+
 1|SIMPLE     |user |          |ALL |             |   |       |   |   4|    25.0|Using where|
```

执行：

```sql
explain select id,name from user where name like '%五';
```

结果：

```
id|select_type|table|partitions|type |possible_keys|key     |key_len|ref|rows|filtered|Extra                   |
--+-----------+-----+----------+-----+-------------+--------+-------+---+----+--------+------------------------+
 1|SIMPLE     |user |          |index|             |idx_name|83     |   |   4|    25.0|Using where; Using index|
```

执行：

```sql
explain select id,name from user where user_exp like '%五';
explain select * from user where user_exp like '%五';
```

两个的结果均为：

```
id|select_type|table   |partitions|type |possible_keys|key     |key_len|ref|rows|filtered|Extra                   |
--+-----------+--------+----------+-----+-------------+--------+-------+---+----+--------+------------------------+
 1|SIMPLE     |user_exp|          |index|             |idx_name|83     |   |   4|    25.0|Using where; Using index|
```

对于上面的例子，首先我们需要查询的 `id`、` name` 这两个字段都在我们的辅助索引中，叶子节点存的索引值和主键值，所以我们只要查辅助索引就可以直接拿到我们的需要的结果了，那么这个叫做索引|覆盖。我们观察执行计划会发现它的查询级别是 index，其实也是全表遍历了辅助索引。

对于第一个例子，查询的是所有字段，而`card`字段不在辅助索引中，如果遍历辅助索引，则还需要回标，效率远没有直接遍历高。

# 事务

## 事务 ACID 属性

事务是由一组SQL语句组成的逻辑处理单元,事务具有以下4个属性,通常简称为事务的ACID属性。

 1、原子性(Atomicity)

事务是一个原子操作单元,其对数据的修改,要么全都执行,要么全都不执行。

2、一致性(Consistent)

在事务开始和完成时,数据都必须保持一致状态。这意味着所有相关的数据规则都必须应用于事务的修改,以保持数据的完整性;事务结束时,所有的内部数据结构(如B树索引或双向链表)也都必须是正确的。

3、隔离性(Isolation)

数据库系统提供一定的隔离机制,保证事务在不受外部并发操作影响的“独立”环境执行。这意味着事务处理过程中的中间状态对外部是不可见的,反之亦然。

4、持久性(Durable)

事务完成之后,它对于数据的修改是永久性的,即使出现系统故障也能够保持。

## 隔离级别

MySQL 事务隔离级别分为 4 个：
- READ UNCOMMITTED：读未提交。
- READ COMMITTED：读已提交。
- REPEATABLE READ：可重复读。
- SERIALIZABLE：序列化。

### READ UNCOMMITTED

读取未提交的数据，该隔离级别的事务可以看到其他事务下未提交的数据。

同时由于未提交的数据可能会发生回滚，因此我们把该级别读取到的数据称之为脏数据，把这个问题称之为脏读。

### READ COMMITTED

读取已提交的数据，所有的数据都是已提交的，所以不会出现脏读的情况，但由于在不同事务中可以读取到其他事务已提交的数据，所以在不同的sql中，可能出现读取的数据不一致的情况，这种情况叫做不可重复读。

### REPEATABLE READ

可重复度，这是 MySQL 默认的事务隔离级别，该隔离级别可以解决 READ COMMITED 所产生的不可重复读问题，但由于同一个事务的不同时间点，使用同一个 SQL 查询数据时，可能出现不同的结果，这种情况叫做幻读。

一个典型的例子：

执行sql：

```sql
select id,name from user
```

第一次执行结果：

id|name
---|---
1|test1


第一次执行结果：

id|name
---|---
1|test1
2|test2

其中，id 为 2 记录则是一个幻读的行

### SERIALIZABLE

序列化，这是 MySQL 事务隔离级别中最高的级别，它会强制事务排序，使之不会发生冲突，从而解决了脏读、不可重复读和幻读问题，但因为执行效率低，所以真正使用的场景并不多。


### 幻读和不可重复读区别

幻读和不可重复读的侧重点不同的：
- 不可重复读侧重于数据修改，两次读取到的同一行数据不一样。
- 幻读侧重于添加或删除，两次查询返回的数据行数不同。

### 不同隔离级别总结

隔离级别|脏读|不可重复读|幻读
---|---|---|---
READ UNCOMMITTED|Y|Y|Y
READ COMMITTED|N|Y|Y
REPEATABLE READ|N|N|Y
SERIALIZABLE｜N|N|N

### 隔离级别实现方式

MySQL 的隔离级别基于锁和 MVCC 机制共同实现的。

SERIALIZABLE 隔离级别是通过锁来实现的，READ-COMMITTED 和 REPEATABLE-READ 隔离级别是基于 MVCC 实现的。不过， SERIALIZABLE 之外的其他隔离级别可能也需要用到锁机制，就比如 REPEATABLE-READ 在当前读情况下需要使用加锁读来保证不会出现幻读。

### 如何选择隔离级别

在选择隔离级别时，需要根据应用的需求和场景进行权衡。如果需要较高的并发性能，可以选择较低的隔离级别，但需注意可能出现的数据一致性问题。如果数据一致性是首要考虑因素，可选择较高的隔离级别，但需要牺牲一些并发性能。

一般来说，大多数应用会使用默认的隔离级别（通常是REPEATABLE READ），并根据具体情况对特定的事务调整隔离级别。

注意，在使用较高隔离级别时，可能需要更多的数据库锁定，这可能会影响性能。因此，需要在保证数据一致性的同时，权衡并发性能和资源开销。

## MVCC机制

MVCC 就是多版本并发控制。MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问。

为什么需要MVCC呢？

数据库通常使用锁来实现隔离性。最原生的锁，锁住一个资源后会禁止其他任何线程访问同一个资源。但是很多应用的一个特点都是读多写少的场景，很多数据的读取次数远大于修改的次数，而读取数据间互相排斥显得不是很必要。所以就使用了一种读写锁的方法，读锁和读锁之间不互斥，而写锁和写锁、读锁都互斥。这样就很大提升了系统的并发能力。

之后人们发现并发读还是不够，又提出了能不能让读写之间也不冲突的方法，就是读取数据时通过一种类似快照的方式将数据保存下来，这样读锁就和写锁不冲突了，不同的事务session会看到自己特定版本的数据。当然快照是一种概念模型，不同的数据库可能用不同的方式来实现这种功能。

### InnoDB与MVCC

MVCC只在 READ COMMITTED 和 REPEATABLE READ 两个隔离级别下工作。其他两个隔离级别够和MVCC不兼容, 因为 READ UNCOMMITTED 总是读取最新的数据行, 而不是符合当前事务版本的数据行。而 SERIALIZABLE 则会对所有读取的行都加锁。

主要的日志有三个：**Redo log, bin log, Undo log**

**InnoDB中通过undo log实现了数据的多版本，而并发控制通过锁来实现。**

binlog：是mysql服务层产生的日志，常用来进行数据恢复、数据库复制，常见的mysql主从架构，就是采用slave同步master的binlog实现的, 另外通过解析binlog能够实现mysql到其他数据源（如ElasticSearch)的数据复制。

redo log：

- 记录了数据操作在物理层面的修改，mysql中使用了大量缓存，缓存存在于内存中，修改操作时会直接修改内存，而不是立刻修改磁盘，当内存和磁盘的数据不一致时，称内存中的数据为脏页(dirty page)。
- 为了保证数据的安全性，事务进行中时会不断的产生redo log，在事务提交时进行一次flush操作，保存到磁盘中, redo log是按照顺序写入的，磁盘的顺序读写的速度远大于随机读写。
- 当数据库或主机失效重启时，会根据redo log进行数据的恢复，如果redo log中有事务提交，则进行事务提交修改数据。这样实现了事务的原子性、一致性和持久性。

undo log: 

- 除了实现MVCC外，还用于事务的回滚。MySQL Innodb中存在多种日志，除了错误日志、查询日志外，还有很多和数据持久性、一致性有关的日志。
- 除了记录redo log外，当进行数据修改时还会记录undo log，undo log用于数据的撤回操作，它记录了修改的反向操作，比如，插入对应删除，修改对应修改为原来的数据，通过undo log可以实现事务回滚，并且可以根据undo log回溯到某个特定的版本的数据，实现MVCC。

### 版本链与Undo log

innodb中通过B+树作为索引的数据结构，并且主键所在的索引为ClusterIndex(聚簇索引), ClusterIndex中的叶子节点中保存了对应的数据内容。一个表只能有一个主键，所以只能有一个聚簇索引，如果表没有定义主键，则选择第一个非NULL唯一索引作为聚簇索引，如果还没有则生成一个隐藏id列作为聚簇索引。

除了Cluster Index外的索引是Secondary Index(辅助索引)。辅助索引中的叶子节点保存的是聚簇索引的叶子节点的值。

InnoDB行记录中除了刚才提到的rowid外，还有trx_id和db_roll_ptr, trx_id表示最近修改的事务的id,db_roll_ptr指向undo segment中的undo log。

新增一个事务时事务id会增加，trx_id能够表示事务开始的先后顺序。

Undo log分为Insert和Update两种，delete可以看做是一种特殊的update，即在记录上修改删除标记。

update undo log记录了数据之前的数据信息，通过这些信息可以还原到之前版本的状态。

当进行插入操作时，生成的Insert undo log在事务提交后即可删除，因为其他事务不需要这个undo log。

进行删除修改操作时，会生成对应的undo log，并将当前数据记录中的db_roll_ptr指向新的undo log。

![](https://static.jiangliuhong.top/images/2024/1/171705458677618.png)

### ReadView

已提交读和可重复读的区别就在于它们生成ReadView的策略不同。

ReadView中主要就是有个列表来存储我们系统中当前活跃着的读写事务，也就是begin了还未提交的事务。通过这个列表来判断记录的某个版本是否对当前事务可见。其中最主要的与可见性相关的属性如下：

**up_limit_id**：当前已经提交的事务号 + 1，事务号 < up_limit_id ，对于当前Read View都是可见的。理解起来就是创建Read View视图的时候，之前已经提交的事务对于该事务肯定是可见的。

**low_limit_id**：当前最大的事务号 + 1，事务号 >= low_limit_id，对于当前Read View都是不可见的。理解起来就是在创建Read View视图之后创建的事务对于该事务肯定是不可见的。

**trx_ids**：为活跃事务id列表，即Read View初始化时当前未提交的事务列表。所以当进行RR读的时候，trx_ids中的事务对于本事务是不可见的（除了自身事务，自身事务对于表的修改对于自己当然是可见的）。理解起来就是创建RV时，将当前活跃事务ID记录下来，后续即使他们提交对于本事务也是不可见的。

![171705459074838.png](https://static.jiangliuhong.top/images/2024/1/171705459074838.png)

假如存在表user，其中数据如下：

```
id|name|trx_id|db_roll_ptr|
--+----+------+-----------+
 1|MVCC|    50|上版本地址   |
```

执行sql:

```sql
update user set name = 'MVCC2' where id = 1;
```

此时undo log存在版本链如下:

![](https://static.jiangliuhong.top/images/2024/1/171705459412621.png)

提交事务id是60的记录后，接着有一个事务id为100的事务，修改name=MVCC3，但是事务还没提交。则此时的版本链是：

![](https://static.jiangliuhong.top/images/2024/1/171705459527113.png)

此时另一个事务发起select语句查询id=1的记录，因为trx_ids当前只有事务id为100的，所以该条记录不可见，继续查询下一条，发现trx_id=60的事务号小于up_limit_id，则可见，直接返回结果`MVCC2`。

那这时候我们把事务id为100的事务提交了，并且新建了一个事务id为110也修改id为1的记录name=MVCC4，并且不提交事务。这时候版本链就是：

![](https://static.jiangliuhong.top/images/2024/1/171705459659779.png)

这时候之前那个select事务又执行了一次查询,要查询id为1的记录。

如果你是已提交读隔离级别READ_COMMITED，这时候你会重新一个ReadView，那你的活动事务列表中的值就变了，变成了[110]。按照上的说法，你去版本链通过trx_id对比查找到合适的结果就是`MVCC4`。

如果你是可重复读隔离级别REPEATABLE_READ，这时候你的ReadView还是第一次select时候生成的ReadView,也就是列表的值还是[100]。所以select的结果是`MVCC3`。所以第二次select结果和第一次一样，所以叫可重复读！

也就是说已提交读隔离级别下的事务在每次查询的开始都会生成一个独立的ReadView,而可重复读隔离级别则在第一次读的时候生成一个ReadView，之后的读都复用之前的ReadView。

这就是Mysql的MVCC,通过版本链，实现多版本，可并发读-写，写-读。通过ReadView生成策略的不同实现不同的隔离级别。





> 参考链接：
>
> [一文理解Mysql MVCC](https://zhuanlan.zhihu.com/p/66791480)