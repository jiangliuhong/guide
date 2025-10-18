---
title: SQL CRUD语句
categories: 
    - 数据库
date: 2018-07-16 10:52:08
tags:
  - SQL
---
# SQL CRUD语句

> 下文以MySQL为例进行说明。
> `CRUD`即增加(Create)、查询(Retrieve)、更新(Update)、删除(Delete)四个单词的首字母缩写。

## 准备

首先准备在MySQL数据库中创建两张表：学生表（student）、班级表(class)，建表语句如下：

```sql
create table class(
id varchar(32) not null,
name varchar(45) not null comment '班级名称',
 PRIMARY KEY (id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
CREATE TABLE student (
  id varchar(32) NOT NULL,
  name varchar(45) NOT NULL comment '学生姓名',
  num int NOT NULL comment '学生学号',
	class_id varchar(32) not null comment '班级id',
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

像表中插入一些测试数据，插入四个班级，往每个班级分别插入五名学生：

```
INSERT INTO class VALUES ('1','1班'),('2','2班'),('3','3班'),('4','4班');

INSERT INTO student VALUE
(replace(UUID(),'-',''),'学生1',2013001,'1')
,(replace(UUID(),'-',''),'学生2',2013002,'1')
,(replace(UUID(),'-',''),'学生3',2013003,'1')
,(replace(UUID(),'-',''),'学生4',2013004,'1')
,(replace(UUID(),'-',''),'学生5',2013005,'1')
,(replace(UUID(),'-',''),'学生6',2013006,'2')
,(replace(UUID(),'-',''),'学生7',2013007,'2')
,(replace(UUID(),'-',''),'学生8',2013008,'2')
,(replace(UUID(),'-',''),'学生9',2013009,'2')
,(replace(UUID(),'-',''),'学生10',2013010,'2')
,(replace(UUID(),'-',''),'学生11',2013011,'3')
,(replace(UUID(),'-',''),'学生12',2013012,'3')
,(replace(UUID(),'-',''),'学生13',2013013,'3')
,(replace(UUID(),'-',''),'学生14',2013014,'3')
,(replace(UUID(),'-',''),'学生15',2013015,'3')
,(replace(UUID(),'-',''),'学生16',2013016,'4')
,(replace(UUID(),'-',''),'学生17',2013017,'4')
,(replace(UUID(),'-',''),'学生18',2013018,'4')
,(replace(UUID(),'-',''),'学生19',2013019,'4')
,(replace(UUID(),'-',''),'学生20',2013020,'4')
```

其中：

UUID()为MySQL中的函数，作用为生成一个随机字符串

replace为MySQL中的函数，作用为替换字符串中的'-'字符

## 别名

在使用SQL的过程中，我们可以为列名称和表名称指定别名（Alias）。

别名在执行CRUD操作时有着很大的作用。

指定别名的语法为在原名称后添加 AS XXX（别名），其中AS可以省略。

对于上面的两张表，我们可以为class、student分别指定别名cla,stu。

```
SELECT cla.id,lca.name FROM class as cla
SELECT stu.id,stu,name,stu.num,stu.class_id FROM student stu 
```

## 增加(Create)

增加数据只用INSERT INTO语句：

```
INSERT INTO TABLE VALUE (value1,value2)
```

上面的语句是默认给表的所有字段插入值，VALUE后面的值的顺序与表字段顺序一直，其个数也必须与表字段个数一直。

当然，我们也可指定要插入数据的列：

```sql
INSERT INTO TABLE (column1,column2) VALUE (value1,value2)
```

如果要一次插入多条数据（批量插入），可以在VALUE后添加多个代码块：

```sql
INSERT INTO TABLE VALUE (value1,value2),(value1,value2)
```

在执行批量插入时，应注意SQL的长度限制，批量插入的脚本不能超过数据库设置的SQL最大长度。

在执行插入语句时还可以增加子查询语句，但语法要求是，子查询返回的字段要与INSERT的插入字段一致

```sql
INSERT INTO table SELECT * FROM table1;
INSERT INTO table1 (column1,column2) SELECT column1,column2 FROM table2
```

## 查询(Retrieve)

查询语句使用SELECT命令即可：

```sql
SELECT column1,column2 FROM table
```

查询所有字段可使用*，不过一般不推荐这样操作

```sql
SELECT * from table
```

### 子查询：

SQL子查询也叫嵌套SELECT语句，一个SELECT语句的查询结果能够作为另一个语句的输入值。子查询能够出现的地方有：

- WHERE子句：左右外层SQL的条件

查询1班、2班的所有学生：

```sql
SELECT stu.name,stu.num FROM student as stu where stu.class_id in (SELECT id FROM class as cla where cla.name in ('1班','2班'));
```

查询结果为:

```tex
学生1	2013001
学生2	2013002
学生3	2013003
学生4	2013004
学生5	2013005
学生6	2013006
学生7	2013007
学生8	2013008
学生9	2013009
学生10 2013010
```

- FROM后：作为一个临时表

从1班、2班所有学生中查询出学号在2013004到2014006之间的学生：

```sql
SELECT name,num
FROM (SELECT stu.name,stu.num FROM student as stu where stu.class_id in (SELECT id FROM class as cla where cla.name in ('1班','2班'))) temp_talbe
WHERE num BETWEEN 2013004 AND 2013006
```

查询结果为:

```tex
学生4	2013004
学生5	2013005
学生6	2013006
```

- COLUMN：COLUMN，即SELECT后面，作为一个字段值来返回，这里内层SQL只能返回一个字段

根据学号查询学生的姓名、班级信息

```sql
SELECT stu.name,stu.num,(SELECT cla.name FROM class as cla where cla.id = stu.class_id) as class_name FROM student as stu where stu.num = 2013004
```

查询结果为：

```tex
学生4	2013004	1班
```

- JOIN子句：作为一个临时表

从1班、2班所有学生中查询出学号在201304到201406之间的学生：

```sql
SELECT stu.name,stu.num
FROM student as stu 
LEFT JOIN (SELECT id FROM class as cla where cla.name in ('1班','2班')) as cla
ON cla.id = stu.class_id
WHERE stu.num BETWEEN 2013004 AND 2013006 
```

查询结果为：

```
学生4	2013004
学生5	2013005
学生6	2013006
```

## 更新(Update)

更新表使用UPDATE语法，将已有的老数据更新为新数据。

```
UPDATE table SET column1 = value1 WHERE 条件
```

注意，如果不加WHERE条件，UPDATE语句会默认更新所有表，慎用。

一次更新多个字段，在SET后面添加多个column：

```
UPDATE table SET column1 = value1,column1 = value1 WHERE 条件
```

### 联表更新

联表更新，根据表一字段的值去设置表二字段的值。其实现方法由于数据库不同可能会有一定差异，下面以MySQL为例。

例如：将学号在2013003到2013005之间的学生的class_id设置为二班的id。

对于MySQL而言，联表更新主要的方式有二种：

- UPDATE table1,table2 set column1 = xxx where ....

```sql
UPDATE student stu,class cla
set stu.class_id = cla.id
where stu.num BETWEEN 2013003 AND 2013005
AND cla.name = '2班';
```

- 使用INNER JOIN（LEFT JOIN）进行更新：

```sql
UPDATE student stu
INNER JOIN class cla ON cla.name = '2班'
SET stu.class_id = cla.id
where stu.num BETWEEN 2013003 AND 2013005
```

执行下面SQL验证结果：

```sql
SELECT name,num,class_id FROM student where num BETWEEN 2013003 AND 2013005
学生3	2013003	2
学生4	2013004	2
学生5	2013005	2
```

在进行UPDATE语句是也可以在其中加入子查询语句。

## 删除(Delete)

删除使用DELETE语法

```sql
DELETE FROM table WHERE column1 = value1
```

DELETE语法与UPDATE语法一直，如果没有WHERE条件则会删除指定表的所有数据。

#### 联表删除

- 根据table2的条件删除table1的数据

删除所有一班的学生：

```sql
DELETE student FROM student,class WHERE class.id = student.class_id and class.name = '1班'
```

```sql
DELETE student FROM student INNER JOIN class ON class.id = student.class_id 
where class.name = '1班'
```


注意，此处的student,即表名，不能使用别名。

- 同时删除两个表的记录

删除一班，并删除一班下面的所有学生：

```sql
DELETE class,student FROM class LEFT JOIN student ON  class.id = student.class_id WHERE class.name = '1班'
```


