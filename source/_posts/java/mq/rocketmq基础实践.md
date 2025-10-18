---
title: rocketmq基础实践
categories:
    - Java
    - 消息中间件
date: 2019-07-14 20:52:08
tags:
    - rocketmq
cover: https://static.jiangliuhong.top/blogimg/mq/%E7%89%A9%E7%90%86%E7%BB%93%E6%9E%84%E5%9B%BE.png
---


# Rocketmq基础实践

## 为什么使用RocketMQ

> [https://rocketmq.apache.org/docs/motivation/](https://rocketmq.apache.org/docs/motivation/)
>
> [https://rocketmq.apache.org/rocketmq/how-to-support-more-queues-in-rocketmq/](https://rocketmq.apache.org/rocketmq/how-to-support-more-queues-in-rocketmq/)

**消息队列的优点：**

- 解耦
- 异步
- 削峰

**消息队列缺点：**

- 系统可用性降低
- 系统复杂度提高
- 存在一致性问题

**常见MQ对比**

| MQ       | 单机吞吐量 | 时效性 | 可用性 | 备注                                                         |
| -------- | ---------- | ------ | ------ | ------------------------------------------------------------ |
| ActiveMQ | 万级       | ms     | 高     | 社区不活跃                                                   |
| RabbitMQ | 万级       | μs     | 高     | 并发性能很强，性能较好，延时低                               |
| RocketMQ | 10万级     | ms     | 非常高 | 分布式系统，适用于topic较多（几百、几千）的场景              |
| Kafaka   | 100万级    | ms     | 非常高 | 一般配合大数据类的系统来进行实时数据计算、日志采集等场景(ELK+Kafka)，不适用topic较多的场景 |

## RocketMQ 特点

> RocketMQ 是阿里巴巴在2012年开源的分布式消息中间件，目前已经捐赠给 Apache 软件基金会，并于2017年9月25日成为 Apache 的顶级项目。作为经历过多次阿里巴巴双十一这种“超级工程”的洗礼并有稳定出色表现的国产中间件，以其高性能、低延时和高可靠等特性近年来已经也被越来越多的国内企业使用。
>
> 目前RocketMQ在阿里集团被广泛应用在订单，交易，充值，流计算，消息推送，日志流式处理，binglog分发等场景。

- 灵活可扩展性
- 海量消息堆积能力
- 支持顺序消息
- 支持多种消息过滤方式
- 支持事务消息
- 支持回溯消费

## RocketMQ物理结构

 ![物理结构图](https://static.jiangliuhong.top/blogimg/mq/%E7%89%A9%E7%90%86%E7%BB%93%E6%9E%84%E5%9B%BE.png)

**NameServer：**

- NameServer是RocketMQ的寻址服务，存储Broker的路由信息以及配置信息，用户端（生成者消费者）依靠NameServer去选择对于的Broker服务

- NameServer集群成员之间互补通信
- NameServer本身不存储数据，其数据均来自Broker与用户端

**Broker：**

Broker负责存储生产者发送的消息，并为消费者提供消费支撑。

- Broker以group分开，每个group只允许存在一个master
- Master、Slave之间数据同步可选择同步、异步复制，同理Master与Master之间也存在同步
- Broker向所有NameServer节点简历长连接，注册Topic信息

## 安装

> 服务器：CentOS7 64位
>
> JDK：1.8
>
> 内存：官方建议8G内存

###  下载软件包

```shell
# cd /usr/local/
# wget http://mirror.bit.edu.cn/apache/rocketmq/4.3.2/rocketmq-all-4.3.2-bin-release.zip
# unzip rocketmq-all-4.3.2-bin-release.zip
# mv rocketmq-all-4.3.2-bin-release rocketmq
```

### 修改日志目录

在`rocketmq/conf`目录下的`logback*.log`文件中配置了日志目录为`${user.home}/logs/....`，如果想要改变目录只需将`${user.home}`改为指定目录即可。

可以使用send命令替换所有logback配置文件中的${usr.home}

```shell
# cd /usr/local/rocketmq/conf/
# sed -i 's#${user.home}#/usr/local/rocketmq#g' *.xml
```

修改启动内存

rocketmq官方预设的NameServer内存为4G，Broker内存为8G。而在开发测试的往往服务器资源较低，建议降低内存大小。

切换到rocketmq的bin目录下（`cd /usr/local/rocketmq/bin`），分别修改`runbroker.sh`与`runserver.sh`脚本中的`JAVA_OPT`参数

runserver.sh

```sh
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

runbroker.sh

```sh
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
```

### 启动RocketMQ

> 对于NameServer与Broker的启动，均可以在启动命令中增加参数，比如使用`-c`指定配置文件，

因为Broker要依赖NameServer，所以应先启动NameServer再启动Broker

**启动NameServer**

```shell
# cd /usr/local/rocketmq/conf/bin
# nohup sh mqnamesrv >/dev/null 2>&1 &
# 也可以使用-c为其指定配置文件
# nohup sh mqnamesrv -c namesrv.conf >/dev/null 2>&1 &
```

**启动Broker**

启动Broker之前先为其指定一个配置文件(`broker.conf`)，内容如下：

```properties
namesrvAddr=192.168.152.134:9876
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
storePathRootDir=/usr/local/rocketmq/store
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
```

启动Broker同时通过-c指定该配置文件

```shell
# nohup sh mqbroker -c ../conf/broker.conf >/dev/null 2>&1 &
```

**检测服务是否启动**

使用`jps`命令查看当前java进程，也可以使用`ps`命令

```shell
[root@rocketmq-nameserver2 bin]# jps
3104 Jps
2689 NamesrvStartup
2979 BrokerStartup
[root@rocketmq-nameserver2 bin]# ps -ef|grep mqnamesrv
root       2683   2303  0 03:57 pts/0    00:00:00 sh mqnamesrv
root       3120   2303  0 04:08 pts/0    00:00:00 grep --color=auto mqnamesrv
[root@rocketmq-nameserver2 bin]# ps -ef|grep mqbroker
root       2972   2303  0 04:06 pts/0    00:00:00 sh mqbroker -c ../conf/broker.conf
root       3122   2303  0 04:08 pts/0    00:00:00 grep --color=auto mqbroker
```

### 控制台安装

> apache/rocketmq-externals： [https://github.com/apache/rocketmq-externals](https://github.com/apache/rocketmq-externals)
>
>rocketmq-externals中的rocketmq-console目录为控制台源码，下载源码，使用`mvn spring-boot:run`命令启动项目，访问地址：http://127.0.0.1:8080


1. 目前官方版本pom文件存在问题，如果直接启动会导致控制台报如下错误：

```
[ERROR] Failed to execute goal on project rocketmq-console-ng: Could not resolve dependencies for project org.apache:rocketmq-console-ng:jar:1.0.0: Failed to collect dependencies for [org.springframework.boot:spring-boot-starter-web:jar:1.4.3.RELEASE (compile), org.springframework.boot:spring-boot-starter-actuator:jar:1.4.3.RELEASE (compile), org.springframework.boot:spring-boot-starter-test:jar:1.4.3.RELEASE (test), commons-collections:commons-collections:jar:3.2.2 (compile), org.apache.rocketmq:rocketmq-tools:jar:4.4.0-SNAPSHOT (compile), org.apache.rocketmq:rocketmq-namesrv:jar:4.4.0-SNAPSHOT (compile), org.apache.rocketmq:rocketmq-broker:jar:4.4.0-SNAPSHOT (compile), com.google.guava:guava:jar:16.0.1 (compile), org.aspectj:aspectjrt:jar:1.6.11 (compile), org.aspectj:aspectjweaver:jar:1.6.11 (compile), cglib:cglib:jar:2.2.2 (compile), org.jooq:joor:jar:0.9.6 (compile)]: Failed to read artifact descriptor for org.apache.rocketmq:rocketmq-tools:jar:4.4.0-SNAPSHOT: Could not transfer artifact org.apache.rocketmq:rocketmq-tools:pom:4.4.0-SNAPSHOT from/to nexus (http://repo.thunisoft.com/maven2/content/groups/public-snapshots/)
```

官方issues说是因为pom文件中rocketmq版本问题，将其改为4.4.0即可，原文地址：[https://github.com/apache/rocketmq-externals/issues/208](https://github.com/apache/rocketmq-externals/issues/208)

2. 在`rocketmq-console\src\main\resources\application.properties`文件中指定配置

```properties
server.contextPath=
server.port=8080
#spring.application.index=true
spring.application.name=rocketmq-console
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
logging.config=classpath:logback.xml
#if this value is empty,use env value rocketmq.config.namesrvAddr  NAMESRV_ADDR | now, you can set it in ops page.default localhost:9876
rocketmq.config.namesrvAddr=192.168.152.134:9876
#if you use rocketmq version < 3.5.8, rocketmq.config.isVIPChannel should be false.default true
rocketmq.config.isVIPChannel=
#rocketmq-console's data path:dashboard/monitor
rocketmq.config.dataPath=/tmp/rocketmq-console/data
#set it false if you don't want use dashboard.default true
rocketmq.config.enableDashBoardCollect=true
#set the message track trace topic if you don't want use the default one
rocketmq.config.msgTrackTopicName=
```

3. RocketMQ-Console中的消息状态，对于源码包中的`org.apache.rocketmq.tools.admin.api.TrackType`
   1. CONSUMED 代表该消息已经被消费
   2. NOT_CONSUME_YET 还没被消费
   3. UNKNOW_EXCEPTION 消费出现异常
   4. NOT_ONLINE 消费者离线

### NameServer配置项

| 配置项                          | 名称                                                         | 备注                                                         |
| ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| listenPort                      | 监昕端口，值默认9876                                         |                                                              |
| serverWorkerThreads             | Netty 业务线程池线程个数，默认值为8                          |                                                              |
| serverCallbackExecutorThreads   | Netty public 任务线程池线程个数                              | Netty网络设计，根据业务类型会创建不同的线程池，比如处理消息发送、消息消费、心跳检测等 |
| serverSelectorThreads           | IO 线程池线程个数，默认为3，主要是 NameServer Broker 端解析请求、返回相应的线程个数 | 这类线程主要是处理网络请求的，解析请求包， 然后转发到各个业务线程池完成具体的业务操作，然后将结果再返回调用方 |
| serverOnewaySemaphoreValue      | 单次消息最大并发度，默认256                                  | 消息请求并发度                                               |
| serverAsyncSemaphoreValue       | 异步消息最大并发度，默认64                                   |                                                              |
| serverChannelMaxIdleTimeSeconds | 网络最大空闲时间，默认120秒                                  |                                                              |

### Broker配置项

| 配置项                           | 名称                                         | 备注                                                         |
| -------------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| brokerClusterName                | 所属集群名字                                 | 默认值DefaultCluster                                         |
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

### Broker高可用

**主从同步方式：**

- 同步双写：Master节点收到消息后，同步消息到Slave节点，主备都写成功，再向返回成功

- 异步复制：Master节点收到消息后，先返回成功，再同步到Slave节点

**刷盘方式：**

- 同步刷盘：节点收到消息后，将数据持久化到硬盘后，再返回成功

- 异步刷盘：节点收到消息后，将消息存储在内存中，先返回成功，再持久化到硬盘中

**官方提供的配置：**

在`rocketmq/conf`目录下存在3个配置文件示例，分别为：

- 2m-2s-async 异步复制、异步刷盘
- 2m-2s-sync 同步双写，异步刷盘
- 2m-noslave 多主节点、异步刷盘

**推荐使用：**同步双写，异步刷盘

**最稳妥的方式：**同步双写、同步刷盘

## 消息存储

### 消息存储结构

![消息结构图](https://static.jiangliuhong.top/blogimg/mq/%E6%B6%88%E6%81%AF%E7%BB%93%E6%9E%84%E5%9B%BE.png)

### 偏移量（`Offset`）

- Offset是消费进度的核心
- Offset的存储实现分为远程存储于本地存储两种
- 对于PushConsumer（集群），Offset存储在Broker端；对于PullComsumer，Offset需要消费者自己维护，将其存在在消费者端
- 集群消费模式，Offset存储在Broker端；广播消费模式，Offset春常在消费者端
- Consumer Offset用于标记Consumer Group在一条Consumer Queue上的消费进度

## 生成者

### 核心参数

- producerGroup：组名唯一
- createTopicKey：创建主题时需要的密钥
- defaultTopicQueueNums：在发送消息时，自动创建服务器不存在的 topic，默认创建的队列数，默认4
- sendMsgTimeout：发送消息超时时间
- compressMsgBodyOverHowmuch：消息压缩字节，默认4096字节，超过该值，rocketmq就会对消息进行压缩
- retryTimesWhenSendFailed：重发策略，同理存在异步的（retryTimesWhenSendAsyncFailed）
- retryAnotherBrokerWhenNotStoreOk:默认false
- maxMessagerSize：消息最大容量，默认128k

### 发送消息

同步发送消息：DefaultMQProducerImpl.producer.send(msg);

异步发送消息：producer.send(Message msg,SendCallback sendCallback);

## 消费者

### 消费模式

**集群模式：**

- RocketMQ默认采用集群消费模式
- 同一ComsumerGroup中的消费者只消费一次

**广播模式：**

- 广播模式下，每个Consumer都会对消息进行消费

### 消息类型

> 首先先定义一个MessageListenerConcurrently

```java
public class MyMessageListenerConcurrently implements MessageListenerConcurrently {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list,
        ConsumeConcurrentlyContext consumeConcurrentlyContext) {
        try {
            MessageExt messageExt = list.get(0);
            String body = new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET);
            System.out.printf("QueueId:%s;Book:%s%n", messageExt.getQueueId(), body);
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
    }
}
```

#### 定时消息

> 定时消息是指消息发到 Broker 后，不能立刻被 Consumer 消费，要到特定的时间点或者等待特定的时间后才能被消费

目前Rocket只支持固定精度的定时消息，官方解释为：

> 如果要支持任意的时间精度，在 Broker 层面，必须要做消息排序，如果再涉及到持久化，那么消息排序要不可避免的产生巨大性能开销

其精度如下：

| 延迟级别 | 时间 | 延迟级别 | 时间 |
| -------- | ---- | -------- | ---- |
| 1        | 1s   | 2        | 5s   |
| 3        | 10s  | 4        | 30s  |
| 5        | 1m   | 6        | 2m   |
| 7        | 3m   | 8        | 4m   |
| 9        | 5m   | 10       | 6m   |
| 11       | 7m   | 12       | 8m   |
| 13       | 9m   | 14       | 10m  |
| 15       | 20m  | 16       | 30m  |
| 17       | 1h   | 18       | 2h   |

**producer代码：**

```java
public class Producer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("test_producer_group");
        producer.setNamesrvAddr(MessageConstants.NAMESRV_ADDR);
        producer.start();
        int totalMessagesToSend = 100;
        for (int i = 0; i < totalMessagesToSend; i++) {
            Message message = new Message("schedule_message_test_topic", ("Hello scheduled message " + i).getBytes());
            //设置级别为3，延迟10s
            message.setDelayTimeLevel(3);
            producer.send(message);
        }
        producer.shutdown();
    }
}
```

**consumer代码：**

```java
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("test_consumer_group");
        consumer.setNamesrvAddr(MessageConstants.NAMESRV_ADDR);
        consumer.subscribe("test_scheduled", "*");
        consumer.registerMessageListener(new MyMessageListenerConcurrently());
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

#### 顺序消息

- 顺序消息：指的是消息的消费顺序与生存顺序相同
- 全局顺序：在某个topic下，所有消息都要保证顺序，设置一个队列
- 局部顺序：只要保证每一组消息被顺序消费即可，根据消息特性，投放到指定队列

**producer代码：**

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("test_producer_group");
        producer.setNamesrvAddr(MessageConstants.NAMESRV_ADDR);
        producer.start();
        Map<Integer, Book> bookMap = new HashMap<>();
        bookMap.put(1, new Book(1, "语文"));
        bookMap.put(2, new Book(2, "数学"));
        bookMap.put(3, new Book(3, "英语"));
        bookMap.put(4, new Book(4, "物理"));
        for (int i = 1; i <= 8; i++) {
            try {
                int bookId = i % 4;
                if (bookId == 0) {
                    bookId = 4;
                }
                Book book = bookMap.get(bookId);
                Message msg = new Message("test_book", "TagA",
                    JSON.toJSONString(book).getBytes(RemotingHelper.DEFAULT_CHARSET));
                msg.setKeys("book_id_" + book.getId());
                SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                        Integer id = (Integer)o;
                        int index = id % list.size();
                        return list.get(index);
                    }
                }, book.getId());
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        producer.shutdown();
    }
}
class Book {
    private int id;
    private String name;
    public Book(int id, String name) {
        this.id = id;
        this.name = name;
    }
    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}
```

**consumer代码：**

```java
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("test_consumer_group");
        consumer.setNamesrvAddr(MessageConstants.NAMESRV_ADDR);
        consumer.subscribe("test_book", "TagA");
        consumer.registerMessageListener(new MyMessageListenerConcurrently());
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

#### 事务消息

![事务消息](https://static.jiangliuhong.top/blogimg/mq/%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF.png)

**MyTransactionListener代码：**

```java
public class MyTransactionListener implements TransactionListener {
    @Override
    public LocalTransactionState executeLocalTransaction(Message message, Object o) {
        System.out.printf("executeLocalTransaction，Obejct:%s%n", JSON.toJSONString(o));
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 该方法主要是设置本地事务状态，与业务方代码在一个事务中，只要本地事务提交成功，该方法也会提交成功
        return LocalTransactionState.COMMIT_MESSAGE;
    }
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
        // 告知RocketMQ是提交还是回滚
        // 新版本将该方法与executeLocalTransaction进行合并
        return null;
    }
}
```

**producer代码：**

```java
public class Producer {
    public static void main(String[] args) throws MQClientException {
        TransactionMQProducer producer = new TransactionMQProducer("test_trans_group");
        producer.setNamesrvAddr(MessageConstants.NAMESRV_ADDR);
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS,
            new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r);
                    thread.setName("TransactionMQProducer-CheckThread");
                    return thread;
                }
            });
        producer.setExecutorService(executorService);
        producer.setTransactionListener(new MyTransactionListener());
        producer.start();
        ArgBean arg = new ArgBean();
        arg.setA("testa");
        arg.setB("testb");
        Message message
            = new Message("test_trans", "TagA", ("test trans messsage!!obj:" + JSON.toJSONString(arg)).getBytes());
        producer.sendMessageInTransaction(message, arg);
    }
}
class ArgBean {
    private String a;
    private String b;
    public String getA() {
        return a;
    }
    public void setA(String a) {
        this.a = a;
    }
    public String getB() {
        return b;
    }
    public void setB(String b) {
        this.b = b;
    }
}
```

**consumer代码：**

```java
public class Consumer {
    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("test_consumer_group");
        consumer.setNamesrvAddr(MessageConstants.NAMESRV_ADDR);
        consumer.subscribe("test_trans", "TagA");
        consumer.registerMessageListener(new MyMessageListenerConcurrently());
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

### 消费总览

![rocketmq消费者消费消息流程](https://static.jiangliuhong.top/blogimg/mq/rocketmq%E6%B6%88%E8%B4%B9%E8%80%85%E6%B6%88%E8%B4%B9%E6%B6%88%E6%81%AF%E6%B5%81%E7%A8%8B.png)

几个重要的变量：

- PullMessageService.pullRequestQueue 记录要发送到broker的请求
- PullRequest.processQueue 流量控制，控制ConsumeRequest的并发访问
- PullRequestHoldService.pullRequestTable 保存正在进行长轮询的请求信息
- ManyPullRequest.pullRequestList 记录正在进行长轮询的PullRequest

### Rebalance介绍

#### Consumer与ConsumerQueue

![消费者与队列的关系](https://static.jiangliuhong.top/blogimg/mq/%E6%B6%88%E8%B4%B9%E8%80%85%E4%B8%8E%E9%98%9F%E5%88%97%E7%9A%84%E5%85%B3%E7%B3%BB.png)

#### 平衡算法

核心代码`RebalanceImpl#rebalanceByTopic`:

```java
allocateResult = strategy.allocate(
                            this.consumerGroup,
                            this.mQClientFactory.getClientId(),
                            mqAll,
                            cidAll);
```

平衡算法`AllocateMessageQueueStrategy`的实现类

## 消息队列同步实践


**涉及知识点：**

- 利用观察者模式，收集数据的变化，触发SpringEvent事件，从而进行消息发送
- 利用SpringEvent负责收集消息进行发送；消费消息时分责分发到不同的处理方法

![同步实践](https://static.jiangliuhong.top/blogimg/mq/%E7%9B%91%E7%8B%B1%E4%B8%80%E7%AB%99%E5%BC%8F%E7%BD%AA%E7%8A%AF%E4%BF%A1%E6%81%AF%E5%90%8C%E6%AD%A5%E5%8E%9F%E7%90%86.png)

##  钉钉社区

- 阿里中间件Aliware开发者中心 21711817
- Apache RocketMQ 中国开发者钉钉群 21791227







