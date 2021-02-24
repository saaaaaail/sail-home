---
title: zookeeper
date: 2020-09-04 14:58:11
tags:
- zookeeper
---

# 什么是zookeeper
ZooKeeper是分布式协调服务，简单点说就是保证zookeeper里面的数据在其管辖下的所有服务之间保持一致。

是一个 文件系统 + 通知机制。

是一个类似文件系统的树形结构。

解决了分布式集群环境下的数据一致性。

> 关于数据一致性C   
> - 强一致性 - 用户写入了A机器一个值，然后B机器读取，这时候由A到B的数据同步还没完成，用户就只能阻塞住等待同步完成以后读取最新的值。
> - 弱一致性 - 指的是同步没有完成也可以读取系统当前的值。
> - 最终一致性 - 指的是刚开始读取允许读到旧值，但是同步完成以后再读又会读到最新的值。

## zookeeper是如何保证一致性的
因为Zookeeper实现了类似Paxos的Zab协议，解决了分布式数据一致性问题。

### ZAB协议
原子广播协议（zookeeper atomic broadcast）。
有 两种模式：  
- 崩溃恢复模式
- 原子广播模式

当服务启动或者leader挂掉以后，zab进入恢复模式，当leader被选举出来，并且大多数Server完成了和leader的状态同步以后恢复模式就结束了。

### 原子广播模式
> 首先什么是两阶段提交协议
> 就将leader与follower执行一个操作拆分为表决阶段与执行阶段。  
> 表决阶段就是leader开启一个事务，然后发送给follower，由followe告诉leader自己是提交还是取消事务。
> 执行阶段就是leader根据表决阶段的响应，只有当所有follower都同意提交事务，leader才会通知所有follower进行提交操作，否则取消执行。

类似于二阶段提交过程。
1. 客户端发送的写指令，会经由follower发送给leader，然后leader会将生成一个事务日志持久化到本地，然后将写指令封装成一个事务Proposal（分配一个递增zxid），发送给所有follower做预提交操作，先把数据传过去。
2. follower接收到以后如果他能够在自己的本地也成功持久化地生成这个事务日志的话，给leader返回ack响应。
3. 如果超过半数的follower响应了，leader就执行commit操作执行，然后发送给follower做commit执行。



# zookeeper能干嘛
- 统一的命名服务
    - 即能根据服务的名称、标识符找到对应的服务（Name Service如Dubbo服务注册中心）
- 统一的配置管理
    - 提供配置文件统一修改的目的（Configurration Management各种开源的配置管理框架）
- 集群管理
    - zookeeper自身集群的管理，
- 分布式消息同步和协调
    - 
- 负载均衡
    - 可以做，但没必要
- **对Dubbo的支持**
    - 提供了一套很好的分布式集群管理的机制，就是他这种基于层次型的目录树的数据结构，并对数中的节点进行有效的管理，从而可以设计出多种多样的分布式的数据管理模型，作为分布式系统的沟通调度桥梁。
- 备注

## zookeeper的应用场景
- 命名服务
- 配置管理
- 集群管理
- 分布式锁

# zookeeper数据模型
与unix类似，整体上看作一颗树，每个节点称做一个zookeeper node，简称ZNode，每个ZNode节点默认能够存储1MB的数据，每个ZNode都可以通过路径唯一标识。

并且 znode 自身仍然是一个树形结构，znode = 路径path + 节点值nodeValue + Stat结构体（记录当前ZNode的一些信息）

## zookeeper Stat 结构体
记录的是每个ZNode节点相关数据的这么一个结构体。
- czxid - 引起这个znode创建的zxid，创建节点的事务的zxid（Zookeeper Transaction Id）
- ctime - znode被创建的时间点
- mzxid - 最后修改znode的zxid值
- mtime - znode最后被修改的时间点
- pZxid - 当前znode的子节点最后更新的zxid
- cversion - znode子节点变化号，znode子节点修改次数
- dataversion - znode数据变化号，就是指令前面指令指向一次就加一次的编号
- aclVersion - znode访问控制列表的变化号
- ephemeralOwner - 如果是临时节点，这个是znode拥有者的sessionId，如果不是临时节点则是0。
- dataLength - znode的value的数据长度
- numChildren - znode的子节点数量

# zookeeper的存在类型 - 持久/临时
create指令：  
- -s 指给节点名称添加序列号排序，例```create -s /myNode v1```，执行完以后会生成一个/myNode0000000001形式的节点。
- -e 创建一个临时节点，客户端关闭以后就没有了，注意是客户端断开连接以后临时节点就没有了。
- -p 默认为持久化节点，不加-p也可以。

## 细分 
由于-s与-e可以同时支持。
- 默认是 持久化节点
- 持久化序列号节点
- 临时节点
- 临时序列号节点

# 常用命令
和redis的kv键值命令类似，不过key变成了path，v就是znode节点值。

- help
- ls
- ls2
    - 比ls多了znode的stat结构体
- stat
- set
- get
- create
- delete
- rmr
    - 递归删除znode节点

# 四字命令
举例ruok询问zookeeper服务器状态:```echo ruok | nc localhost 2181```   
在linux中的指令。
- stat - 输出性能和连接的客户端列表
- conf - 输出相关服务配置的详细信息
- cons - 所有连接到服务器的客户端的完全连接/会话的详细信息
- dump - 列出未经处理的会话和临时节点
- envi - 输出服务环境的详细信息
- reqs - 列出未经处理的请求
- wchs - 列出服务器watch的详细信息
- wchc - 通过session列出服务器watch的详细信息
- wchp - 通过路径列出服务器watch的详细信息

# 通知机制
## 会话
客户端使用某种语言绑定创建一个服务的句柄的时候，就建立了一个Zoookeeper会话，会话创建后句柄就处于CONNECTING状态，客户端会试图连接到Zookeeper服务器，连接成功句柄处于CONNECTED状态，如果发生不可恢复的错误就进入到CLOSED状态。

## watch
是异步+回调的触发机制。

watch是设置一个watcher观察者观察ZNode节点，节点有发生变化就通知客户端执行watcher方法的回调方法，而且根据测试是异步执行。

# zookeeper集群
集群配置文件书写方式：  
（服务器编号N、服务器ip地址、LF通信端口A、选举端口B）  
例：  
```server.N=ip:A:B:observer```

其中A表示Leader-Follower的通信端口，表示与该服务器与Leader交换信息的端口。  
其中B表示选举端口，当Leader挂掉时，其他服务器会相互通信的端口，选举出新的Leader。
最后的observer，如果不添加的话，那么节点就是非observer节点，只有添加了这个词，这个节点就是observer节点了。

## zookeeper中的角色
- Leader： 用来发出提案Proposal，或者提出选举。
- Follower： 可以接受提案或者拒绝提案，还可以在选举过程中发起投票。
- Observer： 可以接受客户端连接，将写请求发给leader，不进行提案的表决也不参加投票过程，只同步leader的状态。
  
每个Server节点有三种状态：  
- LOOKING 当前节点还不知道谁是leader
- LEADING 当前节点是Leader
- FOLLOWING 当前节点找到了Leader，现在是Follower

一般，一个集群中的不同服务器ip不同，但是使用相同的A端口与B端口。  

# zookeeper选举机制

## 过半机制
就是投票选举的时候，zookeeper在QuorumMaj类里面进行过半选票的判断，里面有个half变量等于集群节点的n/2，然后选票必须大于half。这保证了集群不管怎么拆分，网络断联，都不可能产生选出第二个leader：  
- 因为即便是集群对半断开连接，两个集群会因为两方都没有大于half节点而无法选出leader，停止提供服务。
- 如果不是对半分，也保证了数量多的一块集群能选出新leader，数量少于half的节点的集群无法选出新leader而停止了服务。

## 关于zookeeper集群奇数个的原因
部署5台机器与6台机器都是最多只能挂掉两台，挂掉超过两台就整个集群停止服务了。因此选择部署尽量少的5台机器即可，这是部署奇数个主机说法的原因。

但是增加机器能提高读请求的吞吐量。但是会降低leader两阶段提交同步follower的效率。

这时候就可以将这个节点设置为observer节点，不参与投票，只同步数据，同步的地方位于二阶段提交第二阶段，当leader通知其他follower节点commit以后，会跟着通知所有observer执行这个写请求同步数据（要注意一阶段的预处理的时候，leader是没有给observer节点发送请求的）。然后leader会执行自己的commit操作。

选举有三种情况：  
- 集群启动的时候
- leader挂调的时候
- follower挂掉的后当前leader没有过半follower跟随自己了

### 服务器启动过程
- 解析配置
- （根据配置文件里的config.server判断是否大于0，走集群路线。）
- （QuorumPeer这个类记录集群模式下某一服务器节点的信息，并继承了Thread类，是一个线程。）
- QuorumPeer.start()方法
    - loadDataBase() 从磁盘加载服务器节点的数据到内存中。
    - cnxnFactory.start()方法 开启一个线程使用nio去监听端口连接
    - startLeaderElection()方法 仅仅进行领导者选举的准备工作
    - super.start() 真正的选举过程发生在线程启动以后的run方法里
- QuorumPeer.run()方法
- 进入一个while循环 main loop
- 如果当前服务器节点是LOOKING，那么会选择一个服务器节点投票，如果投完以后就break出switch结构，继续循环。
    - 通过lookforLeader()方法，进行投票选举。就是下面的集群启动选举过程。当前服务器在选举过程中主线程会一直阻塞在lookforLeader()该方法，导致不能接收任何请求。
    - setCurrentVote()设置当前服务器，经过上面选举过程后的leader节点。
    - 这样LOOKING节点就结束了，服务器有了新的状态，重新进入了主循环。
- 如果当前服务器节点是OBSERVING节点，
- 如果当前服务器节点是FOLLOWING节点，
- 如果当前服务器节点是LEADING节点，


## 集群启动的时候选举
1. sendNotifications() 首先各个节点都是先投给自己。
2. 然后将自己当前投的leader封装成多个选票，放入sendBlockingQueue，另一个worker线程会根据选票里的myid将队列里的选票分别发送给不同的参与者，通过各个节点第二个port去进行投票。
3. （如果当前节点还是LOOKING状态，就会一直在while里面自旋）然后从recvBlockingQueue接受其他服务器的选票，进行选票pk（epoch与zxid与myid）
     - 如从recvBlockingQueue取到的选票为空，那么当前服务器就会主动的与其他参与者进行socket连接connectAll，遍历参与者依次connectOne，而且只有myid大的能向myid小的建立连接，连接建立好以后，就自己的服务器开两个线程一个是sendWorker，取sendBlockingQueue里的选票进行发送，一个是recvWorker从接收socket发来的选票，并放入当前服务器节点的recvBlockingQueue中。以上就是当从recvQueue里面取得一个数据为空的时候，会去进行了一个连接的新建。

     - 如果从recvBlockingQueue取到的选票不为空，那么判断发来选票的服务器的状态，如果是LOOKING状态就要进行选票PK了，那么就根据epoch与zxid与myid来pk，前面大的就赢，前面相等就比较后面大的赢，如果选票更新成功了就再次通知给其他参与者。
4. 各个节点通过pk不断修改投票对象，并发送给其他服务器节点sendNotifications()。
5. 各个节点将自己与接受到的其他选票都在各自的服务器上维护一个Map作为投票箱。那么前面接收到其他选票并且pk完了以后，覆盖对应服务器myid的最新投票结果。
6. 处理投票，通过过半机制。将投票箱里的选票与当前服务器所投的目标服务器一致的选票放入一个set中通过过半机制验证是否大于half，如果大于half就选出一个准leader。
7. 过半机制返回true以后没完，还会再次从recvQueue里面取选票数据，如果有更优秀的选票，那么break重新进入外层循环，走投票箱-过半机制等等。如果没有新的选票了返回最终得leader。

## leader挂掉以后

## follower挂掉


## 当集群已经启动leader已经选定，新来的节点如何知道自己的角色？
- 如果leader或者follower收到新加入节点的选票以后会将其当前服务器所选的currentVote返回给新来的节点，就是放到自己的sendQueue里面。
- 新来的节点由于自己是LOOKING状态，一直处在一个while循环里面，从自己的recvQueue里面取得新来的数据，这时候发了数据的服务器节点肯定不是LOOKING状态，如果是其他状态的话就直接将选票里的信息new一个vote对象出来，并保存当前的leader是谁。

## 当集群leader假死，新集群选了一个新leader，旧leader复活了怎么处理？
首先新leader选定说明有集群中有过半的follower跟随了他，然后旧leader复活，这时候旧leader的提案epoch都落后于其他follower，那么会被直接拒绝掉，当他发现没有过半的follower跟随自己时会触发选举，状态改为LOOKING，然后按照选举逻辑首先将自己的选票发给其他follower，其他follower如果不是LOOKING状态的，就直接将当前的leader返回给旧leader，旧leader作为新leader的follower接入集群。

# zookeeper数据传输
zookeeper在QuorumPeer初始化阶段。

createElectionAlgorithm方法里定义了FastLeaderElection选举算法。  

在此之前呢，会createCnxnManager创建数据传输的数据结构。

- senderWorkerMap保存对应的服务节点的senderWorker线程
- queueSendMap保存对应服务节点的sendQueue阻塞队列
- lastMessageSent发送给每台服务器最近的消息，也是一个map，value是byteBuffer  
以上三个都是ConcurrentHashMap。   

recvQueue 接收其他服务器发送的数据。

listenr 监听器，是一个线程，通过start启动以后，while循环中通过socket.accept来不断地监听连接，监听的是选举的端口。

对应服务器节点的senderWorker线程从sendQueue里面取数据通过socket发送给对应的服务器节点，而通过receiverWorker线程

以上是数据传输的数据结构。

下面讲解应用层的数据存储结构。

即在FastLeaderElection的构造方法中开始应用层的数据结构初始化。

sendqueue 应用层只管将选票数据放到该发送队列中，  
recvqueue 应用层只管从该队列中取出选票数据。  
Messager 里面new了两个线程一个叫workerSender一个叫workerReceiver，与之前的数据发送相关的线程是不同的。

![zookeeper数据传输架构.png](zookeeper数据传输架构.png)




