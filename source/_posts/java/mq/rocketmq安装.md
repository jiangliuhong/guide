---
title: rocketmq安装
categories:
    - Java
    - 消息中间件
date: 2019-07-14 19:52:08
tags:
    - rocketmq
---

## 安装

1.下载rocketmq

```
# wget http://mirror.bit.edu.cn/apache/rocketmq/4.3.2/rocketmq-all-4.3.2-bin-release.zip
```

2.解压并从命名为rocketmq

```
# tar -zvxf rocketmq-all-4.3.2-bin-release.zip
# mv rocketmq-all-4.3.2-bin-release rocketmq
```

3.新建日志与存储目录

```
# cd /usr/local/rocketmq
# mkdir logs
# mkdir store
# mkdir store/commitlog  
# mkdir store/consumequeue  
# mkdir store/index 
# mkdir store/checkpoint  
# mkdir store/abort
```

4.修改rocketmq日志文件地址

```
# cd /usr/local/rocketmq
# sed -i 's#${user.home}#/usr/local/rocketmq#g' *.xml
```

5.配置文件

```
#所属集群名字
brokerClusterName=DefaultCluster
#broker 名字
brokerName=broker-a
#broker Id
brokerId=0
#nameServer 地址，如果有多个地址用分号分割
namesrvAddr=192.168.152.133:9876
#在发送消息时，自动创建服务器不存在的 topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建 Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
#删除文件时间点，默认凌晨 4 点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=120
#commitLog 每个文件的大小默认 1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue 每个文件默认存 30W 条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
#限制的消息大小
maxMessageSize=65536
#Broker 的角色
#- ASYNC_MASTER 异步复制 Master
#- SYNC_MASTER 同步双写 Master
#- SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
# 检测事务
checkTransactionMessageEnable=false
#发消息线程池数量
sendMessageThreadPoolNums=128
#拉消息线程池数量
pullMessageThreadPoolNums=128
```

6.修改启动参数

```
# cd /usr/local/rocketmq/bin
```

在`bin`目录中

## 启动

1.启动nameserver

```
# nohup sh mqnamesrv >/dev/null 2>&1 &
```

namesrc默认端口为9876，如果需要修改端口，创建一个配置文件，然后启动时指定该配置文件即可。

配置文件内容：

```
# listenPort=9877
```

```
# nohup sh mqnamesrv -c ../conf/namesrv.properties >/dev/null 2>&1 &
```

2.启动broker

```
# nohup sh mqbroker -c ../conf/2m-2s-async/broker-a.properties >/dev/null 2>&1 &
```

## 配置文件详解

| 配置项                           | 名称                                         | 备注                                                         |
| -------------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| brokerClusterName                | 所属集群名字                                 |                                                              |
| brokerName                       | broker 名字                                  | 不同的主节点应配置不同的名称                                 |
| brokerId                         | broker Id                                    | 0 表示 Master，>0 表示 Slave                                 |
| namesrvAddr                      | namesrvAddr地址                              | 多个地址用分号分隔                                           |
| defaultTopicQueueNums            | 默认主题队列数，默认4                        | 在发送消息时，自动创建服务器不存在的 topic，默认创建的队列数 |
| autoCreateTopicEnable            | 自动创建主题状态，默认false                  | 是否允许 Broker 自动创建 Topic，建议线下开启，线上关闭       |
| autoCreateSubscriptionGroup      | 自动创建订阅组状态，默认false                | 是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭       |
| listenPort                       | 监听端口                                     | Broker 对外服务的监听端口                                    |
| deleteWhen                       | 删除文件时间点，默认凌晨 4 点（04）          | 与清理机制有关                                               |
| fileReservedTime                 | 文件保留时间，默认48小时                     | 与清理机制有关                                               |
| mapedFileSizeCommitLog           | commitLog 每个文件的大小默认 1G              | 文件超过该值后，会新建一个文件                               |
| mapedFileSizeConsumeQueue        | ConsumeQueue 每个文件存储个数，默认存 30W 条 |                                                              |
| destroyMapedFileIntervalForcibly | 文件拒绝删除后存活的最大时间，毫秒           | 第一次拒绝删除之后能保留的最大时间                           |
| deletePhysicFilesInterval        | 删除物理文件间隔，毫秒                       | 因为在一次清除过程中，可能需要删除的文件不止一个，该值指定两次删除文件的间隔时间。 |
| diskMaxUsedSpaceRatio            | 检测物理文件磁盘空间，默认75                 |                                                              |
| diskSpaceWarningLevelRatio       | 磁盘空间警戒大小                             | 磁盘空间警戒大小，超过，则停止接收新消息（出于保护自身目的）默认是90 |
| diskSpaceCleanForciblyRatio      | 磁盘空间强制删除文件大小。默认是85           |                                                              |
| storePathRootDir                 | 文件存储路径                                 |                                                              |
| storePathCommitLog               | commitLog 存储路径                           | 存储消息                                                     |
| storePathConsumeQueue            | 消费队列存储路径存储路径                     |                                                              |
| storePathIndex                   | 消息索引存储路径                             |                                                              |
| storeCheckpoint                  | checkpoint 文件存储路径                      | 异常恢复时根据checkpoint点来恢复消息                         |
| abortFile                        | abort 文件存储路径                           | 临时文件，主要记录是否正常关闭                               |
| maxMessageSize                   | 消息最大大小                                 |                                                              |
| brokerRole                       | Broker 的角色                                | ASYNC_MASTER 异步复制主节点 ；SYNC_MASTER 同步双写主节点； SLAVE 从节点 |
| flushDiskType                    | 刷盘方式                                     | ASYNC_FLUSH 异步刷盘;SYNC_FLUSH 同步刷盘                     |
