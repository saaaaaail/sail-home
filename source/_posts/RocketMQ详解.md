---
title: RocketMQ详解
date: 2020-09-11 15:28:43
tags:
- RocketMQ
---

# 简单介绍
1. 能够保证严格的消息顺序
2. 提供丰富的消息拉取模式
3. 高效的订阅者水平扩展能力
4. 实时的消息订阅机制
5. 支持事务消息
6. 亿级消息堆积能力

# 启动
在conf文件夹下包含三个默认的
- 2m-2s-async ：2主2从 异步传输数据
- 2m-2s-sync  ：2主2从 同步传输数据
- 2m-noslave  ：2主 没有从节点

修改bin目录下的runbroker.sh文件里的jvm启动参数
- 设置xmx最大堆内存大小、xms最小堆内存大小、xmn新生代大小、xss单个虚拟机栈大小
  
修改bin目录下的runserver.sh文件里的jvm启动参数
- 设置xmx最大堆内存大小、xms最小堆内存大小、xmn新生代大小、xss单个虚拟机栈大小、-XX:MetaspaceSize=128M、-XX:MaxMetaspaceSize=320M为合适的大小

先启动nameServer、然后启动broker    

# 消息
- 普通消息
- 顺序消息 - 能实现有序的顺序消费
- 事务消息 - 能解决分布式事务实现最终一致性问题

两种消费模式：
- DefaultMQPushConsumer
- DefaullMQPullConsumer


# 消息生产者

DefaultMQProducer 生产者对象  
- producerGroup: 生产者组
- nameServer

Message 消息对象  
- topic: 主题，用来与消费者消费对应的数据
- tags: 标记，用来与消费者对应，过滤主题里的数据
- keys: 消息的唯一值
- 消息数据体

# 消息消费者

DefaultMQPushConsumer 消费者对象，push模式  
- consumerGroup: 消费者组
- nameServer

配置消费什么对象  
- topic: 指定消费的主题
- tags: 过滤规则

消息监听  MessageListenerConcurrently监听器
能获得批量拉取的消息list，能设置一次拉取的消息数量上限，对批量消息list迭代消费掉。
- 消息返回消费成功标志位。
- 消息消费异常则返回消息重试标志位。

# 顺序消息
由于生产者组向broker发送消息，消息会负载到不同的队列中，很难保证顺序消费。

因此要保证顺序消费：
- 消息被发送时保持顺序
- 消息被存储时保持和发送的顺序一致
- 消息被消费时保持和发送的顺序一致

## 原理
- 生产者发送消息选择一个topic队列，然后将顺序消息信息发往这一个队列。 
- 消费者对于顺序消息要保证有序消费，那么就修改普通消息的MessageListenerConcurrently监听器为MessageListenerOrderly监听器进行有序消费。


# 事务消息
1. 生产者组首先向broker发送预提交消息到HalfTopic
2. 预提交消息发送成功以后回调生产者组的执行本地事务的方法
3. 判断生产者组本地事务是否执行成功，成功则看第4步，失败看第7步
4. 将本地执行成功的消息发送到broker的OpTopic 
5. broker会将OpTopic的数据提交到RealTopic中
6. 此时消费者端就可以在RealTopic中消费到这个事务消息，并且通过重试机制解决消费端失败的问题。
7. 如果失败则返回超时或者Unknow状态，由于之前发送了预提交，现在broker想知道进一步的情况，会进行消息回查（会比较HalfTopic与OpTopic里消息的区别确定要进行回查的消息）

在普通消息的基础上：
- 指定消息的事务监听对象TransactionListener，用于执行本地事务与消息回查
- 重写上面接口的executeLocalTransaction执行本地事务
- 重写上面接口的checkLocalTransaction进行消息回查

## RocketMQ分布式事务流程
- 第一阶段. 预提交消息，提交到HalfTopic，broker会拿到事务消息的id值。
- 第二阶段. broker根据事务id回调执行本地事务，如果执行成功会Commit将事务消息写到OpTopic，执行失败会Rollback清理HalfTopic。
- 第三阶段. 如果没有收到之前的预提交的事务消息的确认会回调检查事务状态，返回给broker去做Commit或Rollback

 

# 广播消息
在默认的集群消息模式时，同一个消费组中只有一个消费组消费消息。

在广播模式中同一个消费组中的所有消费者都能消费同一个消息。

# 高级功能
## 消息存储和发送
消息存储采用文件系统的顺序写方式。

消息发送采用的是零拷贝技术。

## 消息存储数据结构
CommitLog文件夹: 存储消息的元数据，包括topic、queueId、Message，这么一个文件的是顺序写入的，采用了一个ConsumerQueue记录这么1g的文件里面各个消息的索引值，包括最小的消息偏移量，已经消费的消息偏移量，最大的消息偏移量。

可以通过CommitLog文件生成ConsumerQueue文件。  

index文件夹: 也是一个索引文件，ConsumerQueue是一个文件，按照消息id的索引值得到偏移量，而index提供了按照消息key、时间戳来查询消息的索引文件。


## 刷盘机制
RocketMQ将消息以文件系统的方式存储到了磁盘上面。有两种写磁盘方式，同步刷盘、异步刷盘。
- 同步刷盘: 指的是消息发送到MQ之后，保存到内存里，立即进行磁盘的写入，写入完成以后才会返回该请求写成功的响应。
- 异步刷盘: 指消息被保存到内存以后就立即返回了写入成功的响应，当内存的消息数量积累到一定程度才会统一触发写磁盘的操作。

这两种机制都能通过broker的配置文件进行配置。

## RocketMQ高可用

### nameServer的高可用
首先生产组集群、消费组集群、broker集群的主机会注册到nameServer集群中。
NameServer集群本身的高可用是通过部署多台NameServer服务器来实现的，但是匹此之间互不通信，因此集群各个nameServer主机之间在某一时刻的数据并不会完全相同的。每个nameServer都是独立的对broker进行心跳检测、路由注册、路由删除的。

检测的话主要通过心跳，Broker启动以后每隔30s会向所有NameServer结点发送心跳包，NameServer收到心跳包会在自己的brokerLiveTable中缓存brokerId与最近发送心跳包的时间，然后NameServer会每隔10s扫描brokerLiveTable，如果这个表里面有连续120s没有收到心跳包的broker，NameServer就移除这个broker的注册信息，并且关闭socket连接。

### broker的高可用

#### 多主多从 异步复制模式
集群中存在多个master节点，每个master节点至少存在一个slave节点。master节点可读可写，slave节点只能读。

因此对于生产者写的话，只能写入master节点，一个master节点宕机了，还可以写入其他master节点。

对于消费组读的话，master宕机以后，仍然能从所有slave节点中读取数据。

优点：高可用   
缺点：使用异步复制的数据同步方式有可能会有消息丢失的问题。

#### 多主多从 同步双写模式

架构上与上面的类似，只是数据同步上面，采用的是同步双写的模式。

- 同步复制：master与slave数据同步完成以后，才对生产者写入反馈成功响应。好处就是master与slave数据保证一致，但是坏出增大了写入延迟，减少了吞吐量。

- 异步复制: 只要master写入成功就可以反馈客户端写成功状态，好处延迟较低，高吞吐，坏处就是Master宕机的时候可能有部分数据还没有同步到slave上。

在broker的配置文件中brokerRole参数，设置成ASYNC_MASTER、SYNC_MASTER、SLAVE（从节点配成这个）值中的一个。

#### 实际应用

将刷盘配置成 异步刷盘
将主从同步配置成 同步复制

## RocketMQ的负载均衡

### 生产者的负载均衡
默认会通过轮询的方式将消息发送到不同的messageQueue中，然后messageQueue可以均匀地落在不同地broker服务器节点上。

### 消费者地负载均衡
#### 集群模式
在集群模式下，同一个消费组中的消费者会均摊接收当前topic的所有messageQueue，即假设有6个messageQueue，在当前消费组中有3个消费者，那么编号为0、1的messageQueue的消息由1号消费者消费，编号为 2、3的messageQueue的消息由2号消费者消费，编号为4、5的messageQueue的消息由3号消费者消费。

还有一种分摊算法是按照broker节点来均匀分摊，即每个消费者分摊各个broker节点中的一个messageQueue。

由于采用的是一种分摊的方法，因此为了避免出现有消费者没有分到messageQueue而导致无消息消费的情况，要求messageQueue的总数量大于消费组中消费者的数量。

#### 广播模式
不是负载均衡的形式。

处于广播模式的消费组中的每一个消费者会消耗每一个messageQueue里的消息。


## 消息重试
由于RocketMQ中消费消息存在普通消息与顺序消息。
- 对于顺序消息，如果一个消息不断失败，那么其会自动不断重试，间隔1s，就会出现后续消息积压无法消费的情况。
- 对于无序消息（普通、定时、延时、事务消息），如果消费失败了，要人为返回错误的状态达到消息重试的结果。无序消息重试只针对集群模式生效。广播方式不提供失败重试机制。

### 重试次数
默认每条消息最多重试16次，每次重试时间间隔会不断增大，当16次都失败以后的消息将不再重试。

### 重试情况
以下情况都会触发消息的重试:   
- 消费消息以后返回了消费失败的状态码
- 返回null
- 抛出异常


## 死信队列
当消息重试次数超过最大次数以后会被放到死信队列。

- 死信队列的消息不会再被正常消费。
- 有效期与正常消息相同均为3天，3天后自动被删除

死信队列的特性:  
- 一个死信队列对应一个Group ID就是一个消费组，而不是对应单个消费者实例
- 如果Group ID未产生死信消息，那么不会新建死信队列
- 一个死信队列包含了对应Group ID产生的死信消息，不论该消息属于哪个topic的

## 消息幂等
指的是消费方接收到多个重复的消息的时候，出现一次与出现多次的结果是一致的。


### 处理方法
Message ID是有可能出现重复的，因此使用Message ID作为处理依据是不靠谱的。

因此使用业务唯一标识作为幂等的关键依据，而业务的唯一标识可以通过Message Key进行设置。

在消费方将每一个消费的消息的key存起来，然后在处理消息之前判断这个消息key有没有被消费过。