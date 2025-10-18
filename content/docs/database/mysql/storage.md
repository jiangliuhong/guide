+++
title = '数据存储'
date = 2024-01-14T11:53:59+08:00
draft = false

+++

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
