---
title: Redis安装
categories: 
    - 数据库
    - Redis
date: 2018-07-16 10:52:08
tags:
---

# Redis安装

## 开始安装

```
[root@localhost home]# wget http://download.redis.io/releases/redis-4.0.10.tar.gz
[root@localhost home]# mv redis-4.0.10 /usr/local/
[root@localhost home]# cd /usr/local/redis-4.0.10/\
[root@localhost redis-4.0.10]# make
```

此时可能会提示如下错误

```
make[1]: Entering directory `/usr/local/redis-4.0.10/src'
    CC Makefile.dep
make[1]: Leaving directory `/usr/local/redis-4.0.10/src'
make[1]: Entering directory `/usr/local/redis-4.0.10/src'
    CC adlist.o
In file included from adlist.c:34:0:
zmalloc.h:50:31: fatal error: jemalloc/jemalloc.h: No such file or directory
```

遇见这样的情况使用一下命令就好

```
[root@localhost redis-4.0.10]# make MALLOC=libc
```

启动redis

```
[root@localhost redis-4.0.10]# src/redis-server 
```

客户端连接

```
[root@localhost redis-4.0.10]# src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```

## 配置文件

关于redis配置文件此处着重说明以下几个地方：

| 字段           | 默认值                  | 说明                                                         |
| -------------- | ----------------------- | ------------------------------------------------------------ |
| requirepass    | foobared                | 设置认证密码                                                 |
| daemonize      | no                      | 设置为yes时，会将redis作为守护进程运行                       |
| protected-mode | yes                     | 是否开启保护模式，远程连接时设置为no                         |
| pidfile        | /var/run/redis_6379.pid | 定义pid文件路径                                              |
| port           | 6379                    | 服务监听端口，默认6379                                       |
| tcp-backlog    | 511                     | TCP 监听的最大容纳数量                                       |
| timeout        | 0                       | 指定在一个 client 空闲多少秒之后关闭连接（0 就是不管它）     |
| databases      | 16                      | 设置数据库的数目                                             |
| dbfilename     | dump.rdb                | 设置 dump 的文件位置                                         |
| bind           | 127.0.0.1               | 运行连接的ip，你如果只想让它在一个网络接口上监听，那你就绑定一个IP或者多个IP，同时也可以设置0.0.0.0 |

更多的你可以查询官方文档 [https://redis.io/documentation](https://redis.io/documentation)