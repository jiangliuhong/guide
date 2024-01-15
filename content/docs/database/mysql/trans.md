+++
title = '事务'
date = 2023-12-04T12:44:59+08:00
draft = false
+++

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