---
title: PostgreSQL安装
categories: 
    - 数据库
    - postgresql
date: 2018-07-16 10:52:08
tags:
---
# PostgreSQL安装

## Linux(CentOs)安装方法

### 使用yum安装

安装存储库RPM： 

```
yum install https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
```

安装客户端软件包： 

```
yum install postgresql10
```

安装服务器包： 

```
yum install postgresql10-server
```

初始化数据库并启用自动启动： 

```
/ usr / pgsql-10 / bin / postgresql-10-setup initdb 
systemctl enable postgresql-10 
systemctl start postgresql-10
```

### 使用tar包安装

#### 准备环境

在进行安装之前，需要创建一个用户与组，否则在初始化数据库的时候会报如下错误：

```
Running in debug mode.
initdb: cannot be run as root
Please log in (using, e.g., "su") as the (unprivileged) user that will
```

新建组与用户

```
groupadd postgres
useradd -g postgres postgres
```

在安装之前需要安装以下：

make、gcc、gcc-c++、zlib-devel、readline-devel 

```
[root@linhp local]# yum -y install gcc
[root@linhp local]# yum -y install gcc-c++
[root@linhp local]# yum -y install readline-devel
[root@linhp local]# yum -y install zlib-devel
[root@linhp postgresql-9.3.1]# yum -y install make
```

如果有，请勿重复安装

#### 开始安装

首先前往官网下载安装包，下载地址：[https://www.postgresql.org/ftp/source/](https://www.postgresql.org/ftp/source/)

你可以选择任意版本，不过不推荐选择`beta`版



```
[root@localhost local]# wget https://ftp.postgresql.org/pub/source/v10.4/postgresql-10.4.tar.gz
[root@localhost local]# tar -zvxf postgresql-10.4.tar.gz
[root@localhost local]# mv postgresql-10.4/ postgresql
```

编译安装

```
[root@localhost postgresql]#  ./configure --prefix=/usr/local/postgresql --without-readline
[root@localhost postgresql]# make && make install
#安装contrib目录下的一些工具，是第三方组织的一些工具代码，建议安装
[root@localhost postgresql]# cd contrib/
[root@localhost postgresql]# make && make install
```

将postgrepsql加入到环境变量

```
[root@localhost contrib]# vim /etc/profile
```

```
export PGHOME=/usr/local/postgresql
export PGDATA=/var/postgresql/data
export PATH=$PGHOME/bin:$PATH
export MANPATH=$PGHOME/share/man:$MANPATH
```

```
[root@localhost contrib]# source /etc/profile
```

初始化数据库

```
[root@localhost postgresql]# su - postgres
[root@localhost postgresql]# initdb -d /usr/local/postgresql/data/
```

启动数据库服务

```
[postgres@localhost postgresql]$ pg_ctl -D /usr/local/postgresql/data -l /usr/local/postgresql/logfile start
```

### 连接数据库

使用`psql`进入控制台

```
[postgres@localhost postgresql]$ psql
psql (10.4)
Type "help" for help.

postgres=# 
```

在控制台中进行一下操作

```
postgres=# create database test;
CREATE DATABASE
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# create table test (id integer, name text);
CREATE TABLE
test=# insert into test values (1,'david');
INSERT 0 1
test=# select * from test ;
 id | name  
----+-------
  1 | david
(1 row)

test=# 
```

顺便一提，在postgre控制台，退出需要使用`\q`命令

### 远程访问

修改PostgreSql配置文件

```
[postgres@localhost postgresql]$ vim data/postgresql.conf 
```

在其最下方添加一行

```
host    all         all         0.0.0.0/0             trust
```

不过出安全，不推荐这样做

创建超级用户

```
[postgres@localhost postgresql]$ createuser -P -s -U postgres -p 5432 psql
```

### 辅助脚本

为了方便操作，这里写三个命令，你可以把这个三个命令写到脚本文件里，这样可以方便执行

**启动命令**

```
pg_ctl -D /usr/local/postgresql/data -l /usr/local/postgresql/logfile start
```

**重启命令**

```
pg_ctl -D /usr/local/postgresql/data -l /usr/local/postgresql/logfile restart
```

**停止命令**

```
pg_ctl -D /usr/local/postgresql/data -l /usr/local/postgresql/logfile stop
```



