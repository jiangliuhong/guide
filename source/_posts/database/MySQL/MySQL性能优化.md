---
title: MySQL性能优化
categories: 
    - 数据库
date: 2018-07-16 10:52:08
tags:
    - MySQL
cover: https://static.jiangliuhong.top/images/picgo/20251018231638.png
---
# MySQL性能优化

## 日志优化

不论是MySQL数据库还是其它数据库，特别是支持事务的数据库而言，其日志需要记录数据服务器中的CURD操作，从而会消耗IO资源，从而影响到数据库性能，特别是操作频繁且数据量大的数据表。对于MySQL而言，其日志主要为二进制日志（BinLog）、错误日志（Error Log）、慢查询日志（Slow Query Log），下面将分别对MySQL的日志优化作出说明。

### BinLog优化

在实际生产环境中，为保护数据库数据的安全性，我们一般都将会打开BinLog日志进行增量备份。对于MySQL的BinLog日志备份，有三种模式，分别为：行模式、语句模式、混合模式。下面对三种模式分别进行说明：

- 行模式：基于行的日志中事件信息记录每行的变化信息。记录没一行数据的修改细节，使用该模式，系统会产生大量的日志内容。
- 语句模式：基于语句的日志中事件信息包含执行的语句。每一条修改数据的Query语句都会记录在日志文件中。该模式不需要记录每一行数据的变化，减少了日志内容，降低了IO开销，提高了数据库性能。但由于该模式只记录Query语句，如果Query语句中包含了一些特定的函数等功能，则会使得MySQL复制出现问题。
- 混合模式：包含上两个模式的事件信息。在该模式下，MySQL会根据执行的每一条Query语句去动态决定该日志需要的日志模式。

在使用MySQL的BinLog日志时，我们应按照事迹情况选择日志模式，比如减少对于特定函数、存储过程的使用，从而提高语句模式的使用率，尽量减少行模式的使用。

**BinLog参数:**

```sql
show variables like '%binlog%';
```

| 名称                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| binlog_cache_size              | 表示事务过程中容纳二进制日志SQL语句的缓存大小。默认值为32K，如果事务过多需要增加该配置大小。 |
| innodb_locks_unsafe_for_binlog | innodb引擎特有的配置，设置是否启用间隙锁，默认为off，即开启间隙锁。 |
| max_binlog_cache_size          | 日志最大缓存大小。使用形式文件存储来自事务的变化。该参数不宜太小。 |
| max_binlog_size                | binlog的最大值，一边设置为523m或1G，但一般不超过1G。         |
| sync_binlog                    | 事务同步设置，在binlog中该参数尤为重要，该配置如果为0，则MySQL会让文件系统自行决定什么时候同步到磁盘中，如果为n，则代表经过n此事务后，将事务同步到磁盘中，如果为1，则代表每次提交事务后头同步到磁盘中。该参数默认值为0，如果要修改该参数，需要注意两点，一是设置为1能保证最大限度保证数据安全，但性能开销大；二是设置为其他值，可以减少性能消耗，但数据安全性较低。 |

### 慢查询日志

在MySQL中可以做通过慢查询日志查看系统中效率较低的Query语句。

查询慢日志相关的参数有：

| 名称                          | 说明                                         |
| ----------------------------- | -------------------------------------------- |
| slow_query_log                | 是否开启慢查询日志，1表示开启，0表示关闭     |
| slow-query-log-file           | MySQL数据库慢查询日志存储路径                |
| long_query_time               | 慢查询时间，超过设置的时间，系统将该查询记录 |
| log_queries_not_using_indexes | 未使用索引的查询也被记录到慢查询日志中       |
| log_output                    | 日志存储方式                                 |

对于慢查询日志，我们可以设置一个时间，从而统计出系统中超过预期的SQL，从而对其进行优化。

## Query Cache优化

Query Cache即对客户端请求的Query语句（Select语句）的结果进行一个缓存操作，将Query语句通过Hash计算得到hash值，将该值作为KEY，查询结果（Result Set）作为VALUE存储在内存中，对于下一个Query语句，MySQL会先进行Hash运行，然后从内存中寻找对应的Query Cache，如果有直接返回，没有则执行语句，并将其加入到Query Cache中，对于频繁执行的Query语句，MySQL直接从Query Cache中获取结果集，从而较少IO开销。

虽然Query Cache能将查询结果缓存以减少下次查询的等待时间，但查询语句不是一成不变，查询的表中的数据同样也会产生变化。下面就细数一下Query Cache的几个缺点：

- Query Cache缓存失效：对于Query Cache缓存的原理是将Query语句查询结果集缓存，如果表的数据变化，MySQL则会清除该Query语句对应的Query Cache缓存。对于变更和查询较为频繁的表，Query Cache 每次进行的Hash运算，每次进行的结果集缓存操作都是对服务器的一大消耗（数据量过大的情况下）。
- Query Cache缓存内存浪费：Query Cache缓存的是一个Query语句的结果集（Result Set），注意此处不是数据表而是结果集，也就是一张表对应有100个不同Query语句，则就会产生100个Query Cache，即同一条记录被多次缓存。从而使得服务器资源消耗大。当然也可限制Query Cache缓存的大小，不过这样的话缓存效率可能较低。

虽然Query Cache具有一些负面影响，但因为其优点，在某些情景下，这些负面影响并不影响我们使用它。当然在使用Query Cache时应注意不要过度依赖Query Cache，我们理应做到扬长避短，充分发挥其优势。

对于上述两个缺点，主要为表数据的变化，Query语句的不同而导致的缓存数据较多。所以我们可以将Query Cache适用的场景做以下归纳：

- 适用于数据变化不频繁的表。
- 结果集不是太大的表，如果太大，可限制缓存大小。

SQL启动与关闭Query Cache缓存：

- SQL_NO_CACHE：强制不使用Query Cache，示例，SELECT SQL_NO_CACHE * from...
- SQL_CACHE：强制使用Query Cache，示例，SELECT SQL_CACHE * from...

**Query Cachede系统变量**

查询SQL为：

```
show variables like '%query_cache%';
```

| 名称                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| query_cache_limit            | 存放单条Query Cahe的最大结果集内存大小，默认为1M             |
| query_cache_min_res_unit     | 每个Query Cache的最小结果集内存大小，默认为4k                |
| query_cache_size             | 系统中用于Query Cache的内存大小                              |
| query_cache_type             | Query Cache开启状态                                          |
| query_cache_wlock_invalidate | 针对MyISAM存储引擎，设置当有WRITE LOCK在某个Table上时，读请求是要等WRITE LOCK释放资源后再查询还是允许直接从Query Cache中读取结果，默认为FALSE（可以直接从Query Cache中取得结果） |

对于Query Cache不仅限于上述的五点配置，还有一些状态变量，使用下面的查询语句可查看其它变量：

```sql
show status like 'Qcache';
```

| 名称                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| Qcache_free_blocks      | Query Cache中目前还有多少剩余的blocks。如果该值显示较大，则说明Query Cache中的内存碎片过多了，可能须要寻找合适的机会进行整理 |
| Qcache_free_memory      | Query Cache中目前剩余的内存大小。通过这个参数可以较为准确地观察出当前系统中的Query Cache内存大小是否足够，是须要增加还是过多了 |
| Qcache_hits             | 多少次命中。通过这个参数可以查看到Query Cache的基本效果      |
| Qcache_inserts          | 多少次未命中然后插入。通过“Qcache_hits”和“Qcache_inserts”两个参数可以算出Query Cache的命中率 |
| Qcache_lowmem_prunes    | 该数值表示有多少query因内存不足而被清楚的Query Cache         |
| Qcache_not_cached       | 表示query_cache_type的设置或者不能被cache的query的数量       |
| Qcache_queries_in_cache | 当前Query Cache中的数量                                      |
| Qcache_total_blocks     | 当前Query Cache中被block的数量                               |

