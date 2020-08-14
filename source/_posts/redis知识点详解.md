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
即 将某个时间点的数据创建为rdb文件（二进制文件）并报存到磁盘里。然后将快照复制到其他服务器里，创建具有相同数据的副本。

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

