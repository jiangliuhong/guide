---
title: MySQL备份与还原
categories: 
    - 数据库
date: 2018-07-16 10:52:08
tags:
    - MySQL
cover: https://static.jiangliuhong.top/images/picgo/20251018231638.png
---
# MySQL备份与还原

## 逻辑备份

> 逻辑备份为通过对数据库的操作导出数据文件，常用的逻辑备份有两种，一种是将数据转换为全量的INSERT语句，另一种是将数据以特定的分隔符进行隔后记录在文本文件中。

### 全量INSERT语句备份



在MySQL数据库中，一般通过MySQL数据库自带的工具mysqldump来生成INSERT语句的逻辑备份文件。

mysqldump常用的集中方法为：

- 导出整个数据库与所有数据

```sql
mysqldump -u username -p dbname > dbname.sql    
```

- 导出数据库结构

```sql
 mysqldump -u username -p -d dbname > dbname.sql    
```

- 导出数据库中的某张表的数据

```sql
mysqldump -u username -p dbname tablename > tablename.sql    
```

- 导出数据库中某张表的表结构（不含数据）

```sql
mysqldump -u username -p -d dbname tablename > tablename.sql   
```

- 按照指定条件导出数据

```sql
mysqldump -u username -p -h主机 数据库  a --where "条件语句" --no-建表> tablename.sql
```

mysqldump参数详解：

--databases：备份多个数据库，选项后跟多个库名。备份文件中会包含USE db_name

--no-create-info：不生成建表语句

--events    :  备份事件

--routines：备份存储过程和函数

--ignore-table=TableName :指定不需要备份的表

--tables：覆盖--databases 或 -B 选项。该选项后的名称参数均被认为是表名。备份指定的表

--default-character-set：指定备份文件的编码，和数据库编码无关 

--lock-all-tables：通过在备份期前加read lock锁定所有库的所有表。会自动关闭—single-transaction和—lock-tables。

--lock-tables：在备份数据库时对当前库添加read lock.

--master-data：在备份文件中添加二进制日志文件名和位置信息，会自动开始--lock-all-tables

--single-transaction：在备份前设置事务隔离级别为REPEATABLE READ并向server发送START TRANSACTION语句。

仅对事务型表如InnoDB有用。与--lock-tables互斥。对于大文件备份--single-transaction与--quick结合使用。

--flush-logs：刷新日志，生成一个新的二进制日志，主要用户做增量备份

--max-allowed-packet:可发送或接受的最大包分组长度 

--no-autocommit：在INSERT前后添加set autocommit=0和commit。

--order-by-primary:将备份的表中的行按主键排序或者第一个唯一键排序。

当备份MyISAM表且将被载入到InnoDB表时很有用，打包备份本身的时间会较长。

--quick:强制mysqldump将查询得到的结果直接输出到文件，不缓存到内存中

### 生成纯文本备份文件

> 除了上述的使用mysqldump命令生成INSERT语句之外，还可以以另一种方式备份数据库，即将数据库中的数据以特定的分隔字符的形式分隔记录在文本文件中，以达到逻辑备份的效果。这样的备份数据与INSERT语句相比，须要使用的存储空间更小，数据格式更加清晰明确，编辑方便。但是缺点是在同一备份文件中不能存在多个表的备份数据，没有数据结构的重建命令。以至于导致备份文件过多，维护和恢复成本会有所增加。

在MySQL中，它为我们提供了一种SELECT语法(select ...into outfile from ...)，让用户可以通过Query查询语句将某些特定数据以指定格式输出到文本文件中，同时也提供了使用的工具和相关的命令，比如source命令，可以方便地将这写备份文件导入到数据中。

语法结构如下：

```sql
SELECT column into outfile filename from table
```

在执行备份语句时需要注意以下几个参数：

- FIELDS ESCAPEDBY['name']：实现字符转义功能，将Query语句要转义的字符进行转义操作。
- FIELDS[OPTIONALLY] ENCLOSED BY 'name'：将字段的内容包装起来。
- FIELDS TERMINATED BY：设置两个字段之间的分隔符。
- LINES TERMINATED BY：设置MySQL输出文件在每条字符结束时要添加的分隔符。

示例：

现有一张学生数据表，其中num代表学号，class_id代表班级id

```
id									name num     class_id
ad4cca53d9c111e8a0df000c2928fdc8	学生3	2013003	2
ad4cca5ed9c111e8a0df000c2928fdc8	学生4	2013004	2
ad4cca69d9c111e8a0df000c2928fdc8	学生5	2013005	2
ad4cca71d9c111e8a0df000c2928fdc8	学生6	2013006	2
ad4cca78d9c111e8a0df000c2928fdc8	学生7	2013007	2
ad4cca80d9c111e8a0df000c2928fdc8	学生8	2013008	2
ad4cca88d9c111e8a0df000c2928fdc8	学生9	2013009	2
ad4cca90d9c111e8a0df000c2928fdc8	学生10	2013010	2
ad4cca98d9c111e8a0df000c2928fdc8	学生11	2013011	3
ad4ccaa0d9c111e8a0df000c2928fdc8	学生12	2013012	3
ad4ccaa8d9c111e8a0df000c2928fdc8	学生13	2013013	3
ad4ccaafd9c111e8a0df000c2928fdc8	学生14	2013014	3
ad4ccab6d9c111e8a0df000c2928fdc8	学生15	2013015	3
ad4ccabed9c111e8a0df000c2928fdc8	学生16	2013016	4
ad4ccac4d9c111e8a0df000c2928fdc8	学生17	2013017	4
ad4ccacbd9c111e8a0df000c2928fdc8	学生18	2013018	4
ad4ccad1d9c111e8a0df000c2928fdc8	学生19	2013019	4
ad4ccad8d9c111e8a0df000c2928fdc8	学生20	2013020	4
```

现在需要备份所有班级id为2的学生数据

```sql
select * into outfile '/home/temp/test.txt' fields terminated by ',' optionally enclosed by '"' lines terminated by ';' from student where class_id = 2;
```

备份结果如下：

```
"2","test",0,"2";"ad4cca53d9c111e8a0df000c2928fdc8","学生3",2013003,"2";"ad4cca5ed9c111e8a0df000c2928fdc8","学生4",2013004,"2";"ad4cca69d9c111e8a0df000c2928fdc8","学生5",2013005,"2";"ad4cca71d9c111e8a0df000c2928fdc8","学生6",2013006,"2";"ad4cca78d9c111e8a0df000c2928fdc8","学生7",2013007,"2";"ad4cca80d9c111e8a0df000c2928fdc8","学生8",2013008,"2";"ad4cca88d9c111e8a0df000c2928fdc8","学生9",2013009,"2";"ad4cca90d9c111e8a0df000c2928fdc8","学生10",2013010,"2";
```

#### 错误分析

在执行SELECT...INTO OUTFILE FROM...备份命令时，第一次使用可能会提下一面的字符：

```
The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

这是因为MySQL中的secure-file-priv参数默认为NULL，该函数的含义为：

- 为NULL时：表示不允许mysql进行导入导出操作。
- 为一个具体路径时，如/home/temp：表示mysql只能在指定目录中进行导入导出操作。
- 为空字符串时：表示mysql可以在任意目录执行导入导出操作。

当然我们可以在mysql中查询secure-file-priv的值，查询命令为：

```
show global variables like '%secure_file_priv%';
```

另外，修改secure-file-priv值得方法为，在mysql的配置文件中[mysqld]下添加一行：

```
[mysql]
secure_file_priv=''
```

### 逻辑备份恢复

对于INSERT语句形式的备份文件的恢复最简单，直接使用source命令：

```
source ./home/temp.sql
```

对于纯文本的备份文件，需要使用LOAD DATA INFILE命令来进行，由于纯文本备份是针对单个表进行的，所以恢复同样的也只能针对单个表进行恢复。

LOAD DATA INFILE命令可以较快地将一个文本文件中的数据读取到数据表中的命令。它是SELECT ... INTO OUTFILE FROM...的补充，主要将其备份的文件导入到数据库中。

```
LOAD DATA [LOW_PRIORITY | CONCURRENT] [LOCAL] INFILE 'FILENAME'
    [REPLACE | IGNORE]
    INTO TABLE tbl_name
    [PARTITION (partition_name,...)]
    [CHARACTER SET charset_name]
    [{FIELDS | COLUMNS}
        [TERMINATED BY 'string']
        [[OPTIONALLY] ENCLOSED BY 'char']
        [ESCAPED BY 'char']
    ]
    [LINES
        [STARTING BY 'string']
        [TERMINATED BY 'string']
    ]
    [IGNORE number {LINES | ROWS}]
    [(col_name_or_user_var,...)]
    [SET col_name = expr,...]
```

参数解释：

- FILENAME：备份文件地址。
- LOW_PRIORITY：锁表，如果有客户端执行查询操作，则会被阻塞。
- CONCURRENT：取消锁表，允许在客户端在恢复过程查询数据。
- LOCAL：客服端服务器均配置后可以查找客户端上的备份文件。
- REPLACE 与 IGNORE：控制输入的行与唯一主键的重复。
- REPLACE：输入数据替换已经存在的数据。
- IGNORE：输入数据与已经存在的数据的主键或唯一索引重复，则丢弃。
- LINES STARTING BY 'prefix_string'：跳过指定的前缀prefix_string（以及前缀前面所有的字符），如果该行不包括指定的前缀，则整个行都被跳过。
-  CHARACTER SET：指定编码格式。
- FIELDS ESCAPEDBY['name']：实现字符转义功能，将Query语句要转义的字符进行转义操作。
- FIELDS[OPTIONALLY] ENCLOSED BY 'name'：安装字段的包装进行解析。
- FIELDS TERMINATED BY：根据字段分隔符进行解析。
- LINES TERMINATED BY：根据行分隔符进行解析。

示例，针对上面的备份进行恢复:

```sql
load data infile '/home/temp/test2.txt' ignore into table student fields terminated by ',' enclosed by '"' lines terminated by ';';
```

### 逻辑备份总结

逻辑备份需要手动进行操作，或者书写相应的服务器脚本进行操作，并且逻辑备份不能备份数据的每一个时刻的数据，同样的逻辑备份不能快速恢复，或者说当数据量过大时，使用命令进行恢复的速度会十分缓慢。

当然逻辑备份也不是一无是处，其优点如下：

- 通过逻辑备份，可以执行相关的Query命令把数据库中的数据完全恢复到备份时的状态，而不影响其他的数据。
- 通过全库的逻辑备份，可以在一个全新的MySQL环境下完全重建一个与备份时候完全一样的数据库，并不受MySQL所处的平台差异（Linux、Windows）的影响。
- 听过特定的条件的逻辑备份，可以将某写特定的数据轻松迁移（同步）其他数据库环境（如果目标环境不是MySQL环境，需要按照目标数据库设置其备份的格式）。
- 通过逻辑备份，可以仅仅恢复备份集中的部分数据而不需要全部恢复。

## 日志备份

> 对于MySQL而言，除了全量备份，还有一个根据二进制日志进行增量备份的方式，这也是MySQL常用的一种备份方式。

### binlog日志

首先需要在配置文件中开启binlog，开启方式为在MySQL配置文件中设置log文件地址：

```
log-bin=文件路径
```

还有一些其他与binlog相关的配置为：

```
binlog_format   = ROW #binlog日志格式，默认为STATEMENT
log-bin = 文件路径  # binlog日志文件
expire_logs_days= 7   #binlog过期清理时间
max_binlog_size = 100m   #binlog每个日志文件大小
binlog_cache_size   = 4m #binlog缓存大小
max_binlog_cache_size   = 512m #最大binlog缓存大小
```

你也可以使用命令查询binlog状态，查询结果为ON则代表binlog已开启。

```
mysql> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

对于binlog而言，主要有以下几个命名可以对日志文件进行操作。

- reset master：清空日志文件，重新记录日志
- reset salve：删除master.info、relay-log.info文件，开始一个全新的日志文件
- flush logs：重新开始计算日志，产生一个新的日志文件
- show binary logs：展示日志文件
- show binlog events：查看日志文件中的事件信息，默认显示第一个二进制文件中的事件

显示指定文件的事件信息

```
show binlog events in '文件名'
```

使用from可指定从某一行开始

使用limit可设置查询的事件数量

```
 show binlog events in '文件名' from 2 limit 2,5
```

- purge binary logs：删除二进制文件

```
purge binary logs to '文件名' #删除指定文件之前的所有文件
purge binary logs before '事件' #删除指定事件之前的所有文件
```

MySQL的二进制日志文件格式包含行模式、语句模式、混合模式。

- 行模式：基于行的日志中事件信息记录每行的变化信息。
- 语句模式：基于语句的日志中事件信息包含执行的语句。
- 混合模式：混合模式包含上两个模式的事件信息。

在MySQL中，系统提供了一个工具：mysqlbinlog，利用该工具，可以查看详细的日志。常用的命令有：

- -v(--verbose)：将事件重构为被注释掉的伪sql语句，-vv，展示更详细的信息。
- --start-position，--stop-position：按照指定位置精确解析binlog日志（精确），如不接--stop-positiion则一直到binlog日志结尾
- --start-datetime，--stop-datetime：按照指定时间解析binlog日志（模糊，不准确），如不接--stop-datetime则一直到binlog日志结尾
- -d：指定库的binlog
- -r：重定向到指定文件
- --read-from-remote-server：从远程服务器读取binlog日志文件，此时需要一些参数，-h、-p、-u等。

```
[root@localhost binlog]# mysqlbinlog -v logs.000001
...省略部分输出内容

# at 350
#181027  9:45:37 server id 1  end_log_pos 399 CRC32 0x53e0e255  Delete_rows: table id 109 flags: STMT_END_F
BINLOG '
AWzUWxMBAAAAOwAAAF4BAAAAAG0AAAAAAAEABHRlc3QAB3N0dWRlbnQABA8PAw8GYACHAGAAAN6E
TRU=
AWzUWyABAAAAMQAAAI8BAAAAAG0AAAAAAAEAAgAE//ABMgR0ZXN0AAAAAAEyVeLgUw==
'/*!*/;
### DELETE FROM `test`.`student`
### WHERE
###   @1='2'
###   @2='test'
###   @3=0
###   @4='2'
# at 399
#181027  9:45:37 server id 1  end_log_pos 430 CRC32 0xd9e0d019  Xid = 30
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

### 基于日志恢复

首先查看binlog日志文件，从中找出被删除的记录。在上面一个示例中，清晰可见该日志记录一个数据库的删除记录。在日志文件中有一句`at 350`，该标记的意思为该日志的事件开始位置，另外在末尾有一个`at 399`，该标记表示该条日志的事件结束位置。如果我们想恢复这条记录，需要先将数据库恢复到位置350之前，如果这是最后一个日志，那么恢复操作就完成了。但是大多数情况下，该条日志后肯定是存在日志的。所以此时需要在恢复到位置350之前后，跳过该日志，继续恢复后面的日志信息。当然，由于日志文件中，清晰的存储了该行数据的所以信息，所以你也可以根据这条日志，手动在数据库中插入。另外也可以根据事件发生的时间来进行恢复，使用时间恢复的话可以免除寻找时间位置的过程，但前提是你的清晰记得操作时间。

恢复数据库到指定位置命令：

```
mysqlbinlog --stop-position=350 logs.000001 |mysql -uroot -p
mysqlbinlog --start-position=399 logs.000001 |mysql -uroot -p 
```

恢复数据库到指定日期命令：

```
[root@localhost /]# mysqlbinlog --stop-datetime="2017-09-28 04:00:00" logs.000001 | mysql -u root -p
[root@localhost /]# mysqlbinlog --start-datetime="2017-09-28 40:00:00" logs.000001 | mysql -u root -p
```

