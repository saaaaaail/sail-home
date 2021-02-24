---
title: redis知识点详解
date: 2020-08-11 13:32:58
tags:
- redis
---

# Redis知识点详解

Redis是非关系型的内存键值对数据库，速度很快，可以存储键与五种类型的值之间的对应关系。

键的类型只支持字符串，值得类型包括: 字符串String、列表List、集合Set、散列表HashMap、有序集合ZSet

## 数据类型

|数据类型|可以存储的值|操作|
|-|-|-|
|STRING|字符串、整数、浮点数|对整个字符串或者字符串的一部分进行操作；对整数或浮点数执行自增减操作|
|LIST|列表|从两段压入、弹出元素|
|SET|无序集合|添加、获取、移除单个元素；检查一个元素是否存在于集合中；计算交集、并集、差集；从集合里面随机获取元素|
|HASH|包含键值对的无序散列表|添加、获取、移除单个键值对；获取所有键值对；检查键是否存在|
|ZSET|有序集合|添加、获取、删除元素；根据分值范围或者成员来获取元素；计算一个键值分数的排名|

### STRING
```redis
> set hello world
OK
> get hello
"world"
> del hello
(integer) 1
> get hello
(nil)
```

### LIST
```REDIS
> rpush list-key item
(integer) 1
> rpush list-key item2
(integer) 2
> rpush list-key item
(integer) 3

> lrange list-key 0 -1
1) "item"
2) "item2"
3) "item"

> lindex list-key 1
"item2"

> lpop list-key
"item"

> lrange list-key 0 -1
1) "item2"
2) "item"
```

### SET
```REDIS
> sadd set-key item
(integer) 1
> sadd set-key item2
(integer) 1
> sadd set-key item3
(integer) 1
> sadd set-key item
(integer) 0

> smembers set-key
1) "item"
2) "item2"
3) "item3"

> sismember set-key item4
(integer) 0
> sismember set-key item
(integer) 1
//删除 item2
> srem set-key item2
(integer) 1
> srem set-key item2
(integer) 0

> smembers set-key
1) "item"
2) "item3"
```
### HASH
```REDIS
> hset hash-key sub-key1 value1
(integer) 1
> hset hash-key sub-key2 value2
(integer) 1
> hset hash-key sub-key1 value1
(integer) 0

> hgetall hash-key
1) "sub-key1"
2) "value1"
3) "sub-key2"
4) "value2"

> hdel hash-key sub-key2
(integer) 1
> hdel hash-key sub-key2
(integer) 0

> hget hash-key sub-key1
"value1"

> hgetall hash-key
1) "sub-key1"
2) "value1"
```

### ZSET
```REDIS
//向zset-key这个键里面添加分值为728的member1值
> zadd zset-key 728 member1
(integer) 1
> zadd zset-key 982 member0
(integer) 1
> zadd zset-key 982 member0
(integer) 0
//打印从0开始的所有值
> zrange zset-key 0 -1 withscores
1) "member1"
2) "728"
3) "member0"
4) "982"
//按分数打印从0到800的所有值
> zrangebyscore zset-key 0 800 withscores
1) "member1"
2) "728"
//删除member1
> zrem zset-key member1
(integer) 1
> zrem zset-key member1
(integer) 0

> zrange zset-key 0 -1 withscores
1) "member0"
2) "982"
```

## Redis数据结构详解


## Redis使用场景
### 计数器
可以对String类型进行自增自减运算，从而实现计数器功能。

Redis内存性数据库的读写效率很高，很适合存储频繁读写的计数量。

### 缓存
将热点数据放到内存中，设置内存最大使用量，以及淘汰策略保证缓存的命中率。

### 查找表
DNS系统就很适合使用Redis进行存储。也是利用的Redis查找效率高的特性。

查找表里的数据不能失效，缓存可以失效，因为缓存不作为可靠的数据来源。

### 消息队列
- 基于List双向列表的 LPUSH+BRPOP 的实现写入和阻塞读取消息。  
    - 需要解决空闲连接的问题，如果线程一直阻塞在那儿，没有新的数据到来，服务端一般会主动断开连接，这时候blpop/brpop就会报错，因此要注意捕获异常并重试。

### 会话缓存
可以使用Redis来统一存储服务器集群的会话信息。

当应用服务器不再存储会话状态，用户就可以请求任意服务器。

### 分布式锁
在分布式场景下，无法使用单机环境的锁来对多个节点进程进行同步。
可以使用Redis自带的SETNX指令来实现分布式锁，除此以外还可以使用官方提供的RedLock分布式锁实现。

### 其他应用
Set可以实现并集、交集等操作，从而实现共同好友等功能
ZSet可以实现有序性操作，比如排行榜

## Redis与Memcached
### 数据类型
Memcached仅支持字符串类型，Redis支持五种不同的数据类型。

### 数据持久化
Redis支持两种持久化策略:RDB快照和AOF日志，Memcached不支持。

### 分布式
Memcached不支持，需要在客户端使用一致性哈希来实现分布式存储，每次存之前都要计算位于哪个节点。
Redis Cluster实现了分布式的支持。

### 内存
Redis数据不是一直存储在内存中。可以将很久没用的value交换到磁盘，而Memcached的数据一直会在内存中。

## Redis数据淘汰策略
可以设置内存最大使用量，如果超过就会实现淘汰策略。
Redis有6种淘汰策略:
|策略|描述|
|-|-|
|volatile-lru|从设置了过期时间的的数据集中挑选最近最少使用的数据淘汰|
|volatile-ttl|从已设置过期时间的数据集中挑选将要过期的数据淘汰|
|volatile-random|从已设置过期时间的数据集中挑选任意数据淘汰|
|allkeys-lru|从所有数据集中选择最近最少使用的数据淘汰|
|allkeys-random|从所有数据集中任意选择数据淘汰|
|noeviction|禁止淘汰数据|

使用 Redis 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是热点数据。可以将内存最大使用量设置为热点数据占用的内存量，然后启用 allkeys-lru 淘汰策略，将最近最少使用的数据淘汰。  
Redis 4.0 引入了 volatile-lfu 和 allkeys-lfu 淘汰策略，LFU 策略通过统计访问频率，将访问频率最少的键值对淘汰。

## Redis持久化
Redis是内存型数据库，为了保证断电后数据不丢失，需要将内存中的数据持久化到硬盘上。
### RDB持久化
即 将某个时间点的快照数据创建为rdb文件（二进制文件）并报存到磁盘里。然后将快照复制到其他服务器里，创建具有相同数据的副本。

- save指令触发RDB，是阻塞的方式，如果文件特别大会导致系统很长时间无响应，如果存在老RDB文件，直接替换掉，O(N)
- bgsave指令触发RDB，会fork一个子线程出来阻塞住完成rdb文件创建保存。不会阻塞客户端命令。
- 满足配置文件的任一条件，比如多少秒内修改了多少次，自动触发RDB，bgsave。

如果系统发生故障，将会丢失最后一次快照以后的数据。

### AOF日志
Redis会将将写入修改命令添加到AOF文件的末尾保存。

使用AOF文件需要设置同步选项，用来确保AOF文件内容即新加入的写命令同步到磁盘里的时机。这是因为对文件写入并不会马上将内存同步到磁盘，而是先放到缓冲区，然后由操作系统决定什么时候同步到磁盘。

这里Redis提供了触发操作系统同步的选项如下：  
|选项|同步频俩|
|-|-|
|always|每个写命令都触发一次同步|
|everySec|每秒触发一次同步|
|no|让操作系统自己决定什么时候同步比较好|

- always 会严重降低cpu的性能
- everySec 每秒一次比较合适，而且最多丢一秒数据，对性能也没影响
- no 选项并不能给服务器性能带来多大的提升，而且也会增加系统崩溃时数据丢失的数量。  

#### AOF重写
Redis还提供了对越来越大的AOF文件的重写机制，去除AOF文件中冗余的命令。
- 减少硬盘占用量
- 加快恢复的速度

AOF通过bgrewriteaof指令触发重写，或者通过配置文件设置自动触发时机。

## Redis事务
Redis中事务不支持原子性、持久性，不支持回滚。

Redis通过multi、exec命令通过一条连接连续执行多个命令来实现事务。

## Redis事件


## Redis复制
通过slaveof host port 命令来让一个服务器成为另一个服务器的从服务器。一个服务器只能有一个主服务器，并且不支持主主复制。

### 连接过程
1. 主服务器创建快照文件，发送给从服务器，并在发送期间使用缓冲区记录执行的写命令，快照文件发完以后，开始向从服务器发送存储在缓冲区的写命令。
2. 从服务器丢弃所有旧数据，载入主服务器发来的快照文件，完成以后开始接收主服务器发来的写命令。
3. 之后主服务器每执行一条写命令，就向从服务器发送相同的写命令。

### 主从链
如果主服务器负载不断地上升，无法更新所有地从服务器，这时候可以在原来地一对多关系里加一层中间层节点，来分担主服务器地复制工作，中间层服务器是上层服务器地从服务器，是下层服务器的主服务器。

例：  
一个主服务器0有对应9个从服务器（编号1~9），现在负载太高，在中间加三个服务器（A、B、C），现在主服务器0的从节点变成了三个，而A服务器的从节点为（1、2、3），B服务器的从节点为（4、5、6），C服务器的从节点为（7、8、9），可以看到每个服务器的负载都为3个，缓解主服务器的超载的情况。


## Redis哨兵
上面就是Redis的主从结构，主要是备份关系，数据一致，所以从服务器比较清闲，可以让它提供读服务，主服务器提供写服务，达到读写分离的目的。

主从结构如果主服务器挂掉了，这时候就要使用哨兵来监听集群中的其他节点，在主服务器下线状态时，自动从从服务器中选举出新的主服务器。

## Redis集群

> [redis的哨兵模式和集群模式](https://www.jianshu.com/p/d6d2325a5ec7)

### 如何保证redis集群高可用和高并发？redis主从复制原理介绍一下？redis的哨兵模式介绍一下？
redis高可用自然要使用多台机器来保证自己挂掉以后服务不会挂掉。
首先是redis的主从架构。 一主多从，主负责写，并将数据复制到其他从节点，从节点负责读。所有的读请求全部走从节点。

**主从复制核心原理**：  
当启动一个 slave node 的时候，它会发送一个 PSYNC 命令给 master node，如果这是 slave node 初次连接到 master node，那么会触发一次 full resynchronization 全量复制。此时 master 会启动一个后台线程，开始生成一份 RDB 快照文件，同时还会将从客户端 client 新收到的所有写命令缓存在内存中。 RDB 文件生成完毕后， master 会将这个 RDB 发送给 slave，slave 会先写入本地磁盘，然后再从本地磁盘加载到内存中，接着 master 会将内存中缓存的新来的写命令发送到 slave，slave 也会同步这些数据。slave node 如果跟 master node 有网络故障，断开了连接，会自动重连，连接之后 master node 仅会复制给 slave 部分缺少的数据。在保持连接的后续过程中，master node会持续将写命令异步复制给slave node。

**增量复制**：  
如果全量复制过程中，master-slave 网络连接断掉，那么 slave 重新连接 master 时，会触发增量复制。
master 直接从自己的 repl-backlog 中获取部分丢失的数据，发送给 slave node，默认 backlog 就是 1MB。
master 就是根据 slave 发送的 psync 中的 offset（各个从节点自己维护自己的偏移量） 来从 backlog 中获取数据的。

**那么如何才能做到Redis的高可用呢？**
就必须保证当master node挂掉以后，能进行故障转移或者主备切换。即将某个slave node 自动切换为 master node。可以通过redis哨兵来实现高可用。

#### **redis哨兵**
可以做： 
- 集群监控，负责监控master和slave是否正常工作。
- 消息通知，如果集群中有发生故障要进行通知。
- 故障转移，将某slave转成master。
- 配置中心，如果故障发生了转移要通知client更新master地址。

当master挂了以后，需要一半以上的哨兵同意才行，涉及分布式选举的问题。  
哨兵本身也必须是集群部署的，不然哨兵挂了，谁来监控哨兵。  
- 哨兵集群必须至少三个实例，保证自己的健壮性。两台哨兵的话要求一半而是两台都能投票，但是如果挂了一台，就没办法投票了。

**主备切换有数据丢失：**  
- 异步复制数据导致的数据丢失  
  有可能宕机的master上还有部分数据没有同步到slave，这部分数据就丢了
- 脑裂导致数据丢失  
  可能旧master只是暂时脱离网络，但是哨兵认为master宕机了，重新选举了一个slave作为新master，这时候旧的master又恢复了，集群中就有两个master。这时候旧master恢复不久，可能client端还没修改为新master，继续向旧master写数据，但是旧master不久就被作为slave节点，挂到了新master上，数据被清空。新的master也缺少了旧master刚刚恢复阶段被写入的数据。这部分数据就没了。
**主备切换数据丢失解决方案：**  
配置哨兵集群
```
min-slaves-to-write 1
min-slaves-max-lag 10
```
保证 至少有1个slave，数据复制和同步的延迟不能超过10s。那么上面俩问题数据丢失最多维持在10s内。

#### sdown 和 odown 转换机制
- sdown 是主观宕机，就一个哨兵如果自己觉得一个 master 宕机了，那么就是主观宕机
- odown 是客观宕机，如果 quorum 数量的哨兵都觉得一个 master 宕机了，那么就是客观宕机
  
#### 哨兵集群的自动发现机制
哨兵互相之间的发现，是通过 Redis 的 pub/sub 系统实现的，每个哨兵都会往 __sentinel__:hello 这个 channel 里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在。

每隔两秒钟，每个哨兵都会往自己监控的某个 master+slaves 对应的 __sentinel__:hello channel 里发送一个消息，内容是自己的 host、ip 和 runid 还有对这个 master 的监控配置。

每个哨兵也会去监听自己监控的每个 master+slaves 对应的 __sentinel__:hello channel，然后去感知到同样在监听这个 master+slaves 的其他哨兵的存在。

每个哨兵还会跟其他哨兵交换对 master 的监控配置，互相进行监控配置的同步。

#### slave->master 选举算法
如果一个 master 被认为 odown 了，而且 majority 数量的哨兵都允许主备切换，那么某个哨兵就会执行主备切换操作，此时首先要选举一个 slave 来，会考虑 slave 的一些信息：

- 跟 master 断开连接的时长
- slave 优先级
- 复制 offset
- run id

接下来会对 slave 进行排序：
- 首先排除断开时间超过了 down-after-milliseconds 的 10 倍的slave
- 按照 slave 优先级进行排序，slave priority 越低，优先级就越高。
- 如果 slave priority 相同，那么看 replica offset，哪个 slave 复制了越多的数据，offset 越靠后，优先级就越高。
- 如果上面两个条件都相同，那么选择一个 run id 比较小的那个 slave。

### Redis 集群模式的工作原理能说一下么？在集群模式下，Redis 的 key 是如何寻址的？分布式寻址都有哪些算法？了解一致性 hash 算法吗？
#### Redis cluster 介绍
- 自动将数据进行分片，每个 master 上放一部分数据
- 提供内置的高可用支持，部分 master 不可用时，还是可以继续工作的

Redis cluster 节点间采用 gossip 协议进行通信。

#### 分布式寻址算法
- 一致性hash算法
指的是将hash空间定义成1~2的32次方-1的这么一个圆环，然后将服务器节点的ip或者host通过hash以后，分布在圆环的某一个索引值上。然后对key节点计算hash并计算在圆环上的索引值，从这个索引值出发逆时针寻找到第一个服务器节点作为自己存储的节点。

这时候要新增服务器节点的话，也只会影响新服务器插入位置的后一个服务器节点的数据。

如果服务器节点很少，只有俩容易造成数据分布不均，这时候使用虚拟节点，对每一个服务器节点求多个hash索引值，设置多个虚拟节点，比如两台服务器A、B就可以均分圆环，一半的虚拟节点属于A，一半的虚拟节点属于B，如果key值命中了A1就A去处理，命中了B1由B节点去处理。


- Redis cluster 的 hash slot 算法






#### Redis cluster 的高可用与主备切换原理
Redis cluster 的高可用的原理，几乎跟哨兵是类似的。

1. 判断节点宕机
如果一个节点认为另外一个节点宕机，那么就是 pfail ，主观宕机。如果多个节点都认为另外一个节点宕机了，那么就是 fail ，客观宕机，跟哨兵的原理几乎一样，sdown，odown。

在 cluster-node-timeout 内，某个节点一直没有返回 pong ，那么就被认为 pfail 。

如果一个节点认为某个节点 pfail 了，那么会在 gossip ping 消息中， ping 给其他节点，如果超过半数的节点都认为 pfail 了，那么就会变成 fail 。

2. 从节点过滤
对宕机的 master node，从其所有的 slave node 中，选择一个切换成 master node。
检查每个 slave node 与 master node 断开连接的时间，如果超过了 cluster-node-timeout * cluster-slave-validity-factor ，那么就没有资格切换成 master 。

3. 从节点选举
每个从节点，都根据自己对 master 复制数据的 offset，来设置一个选举时间，offset 越大（复制数据越多）的从节点，选举时间越靠前，优先进行选举。

所有的 master node 开始 slave 选举投票，给要进行选举的 slave 进行投票，如果大部分 master node （N/2 + 1） 都投票给了某个从节点，那么选举通过，那个从节点可以切换成 master。

从节点执行主备切换，从节点切换为主节点。

4. 与哨兵比较
整个流程跟哨兵相比，非常类似，所以说，Redis cluster 功能强大，直接集成了 replication 和 sentinel 的功能。


## 面试题
### 谈一谈redis的持久化机制
Redis的持久化机制通过rdb文件与aof日志来实现持久化。  
rdb文件是redis将某一时刻在内存中的数据保存到这个.rdb二进制文件中生成一个数据快照。可以通过配置文件添加save的触发条件，就是多少秒内修改了多少次触发。  
aof文件是redis会将每个写指令追加到aof文件的末尾，然后可以通过配置触发多久将aof文件与磁盘进行一次同步。  
redis重启的时候会将aof里的写指令重新执行一遍。  

### 缓存雪崩
就是大量的缓存在同一时间过期，大量的请求直接去访问数据库。会对cpu和数据库产生巨大的压力。  
**解决方案：**  
- redis高可用
- 限流降级  
  信号量、  
  线程池+有界阻塞队列+第一种拒绝策略、  
  滑动窗口（用一个窗口在数组上滑动，窗口6格，每格10s，用户在哪个计数就在哪个格子+1，当窗口总和大于100就触发了限流，也就是每10s丢掉最前面格子的访问次数，而最后补上了一个空格子）、  
  令牌桶（就是按照固定速率向桶里添加令牌，超出上限就丢弃令牌，每个请求来要取一个令牌才能通过，获取不到要么等待，要么丢弃请求）、  
  漏桶（将每个到来的请求入队等待，每个处理器固定速率取出请求处理）
- 数据预热  
  将可能过期的数据重新添加一遍，设置不同的过期时间，让缓存均匀失效。


### 缓存穿透
就是指频繁请求一个数据库没有的数据，那么缓存中自然也不会存，那么每次访问都会去请求数据库。

**解决方案：**  
- 布隆过滤器
- 数据查询为空也向redis存入空值，超时时间5min

#### 布隆过滤器
错误率假设 Hash 函数以等概率条件选择并设置 Bit Array 中的某一位，假定由每个 Hash 计算出需要设置的位（bit） 的位置是相互独立, m 是该位数组的大小，k 是 Hash 函数的个数。  
位数组中某一特定的位在进行元素插入时的 Hash 操作中没有被置位的概率是：  
$1-\frac{1}{m}$   

一个元素在所有 k 次 Hash 操作后该位都没有被置 "1" 的概率是：  
$(1-\frac{1}{m})$<sup>k</sup>  

如果我们插入了 n 个元素，那么某一位仍然为 "0" 的概率是：  
$(1-\frac{1}{m})$<sup>kn</sup> 

该位为 "1"的概率是：  
$1-(1-\frac{1}{m})$<sup>kn</sup> 

检测某一元素是否在该集合中。标明某个元素是否在集合中所需的 k 个位置都按照如上的方法设置为 "1"，如果这k个位置都恰好为1，那么我们就认为这个数可能存在于集合中，那么就被误判了，这个概率就是错误率，但是该方法可能会使算法错误的认为某一原本不在集合中的元素却被检测为在该集合中（False Positives），该概率由以下公式确定：  
$(1-(1-\frac{1}{m})^{kn}))^k$ 约等于   

$(1-e^{-\frac{nk}{m}}))^k$ 

如何使得错误率最小，对于给定的m和n，当  的时候取值最小。



### Memcache与Redis的区别都有哪些？
1. memcached所有的值均是简单的字符串，Redis作为其替代者，支持更为丰富的数据类型
2. Redis的速度比memcached快很多
3. redis可持久化数据
4. 支持主从备份。
5. redis的value范围最大值为512M，memcached为1M

### 为什么redis快
- 纯内存操作
- 单线程操作，不需要上下文切换
- 采用了非阻塞的多路复用io

### redis的过期策略及内存淘汰机制
redis采用的是定期删除+惰性删除策略。
在redis.conf中有一行配置
maxmemory-policy volatile-lru


### 分布式锁
- redis ，使用key作为锁，使用value判断哪个线程获得了锁，value = host + 线程名 + 时间戳。当加锁结束以后，只会有一个线程成功，哪个线程知道自己成功了呢，就再查一遍，看看这个key的value是否跟自己的value相等，相等的加锁成功。
- Mysql
    - 悲观锁 ， insert 一条唯一的name作为锁 走索引，数据库保证只有一个线程可以成功，失败的抛异常无影响。成功的就认为加锁成功了继续向后执行，执行完成，delete form table where name = 'xxx'即可;
  