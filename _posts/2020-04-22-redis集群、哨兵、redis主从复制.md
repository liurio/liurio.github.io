---
logout: post
title: redis集群、哨兵、redis主从复制
tags: [redis,all]
---

redis集群和哨兵机制有什么区别？

redis服务器的高可以用，如何保证备份的机器是原始服务器的完整备份呢？需要用到哨兵和复制。

哨兵：可以管理多个redis服务器，它提供了监控提醒以及自动的故障转移功能；

复制：负责让一个redis服务器可以配备多个备份的服务器。

#### 哨兵

redis哨兵的主要功能：

1. 集群监控：负责监控master和slave是否正常工作
2. 消息通知：如果某个redis实例有故障，那么哨兵负责发送消息作为报警通知给管理员
3. 故障转移：如果master挂掉了，会自动转移到slave上(Raft选举算法)
4. 配置中心：如果故障转移发生了，通知client客户端新的master地址

redis哨兵的高可用：

原理：当主节点发生故障，由redis哨兵自动完成故障发现和转移，并通知应用方，实现高可用。

![拼多多社招面经：Redis是重点，https是怎么做到安全的？](https://imgconvert.csdnimg.cn/aHR0cDovL3AzLnBzdGF0cC5jb20vbGFyZ2UvcGdjLWltYWdlLzNiZDMyMGQ2MzZlNDRhMDZhMjg3Zjc4ZTczODBjOWNk?x-oss-process=image/format,png)

1. 哨兵机制建立了多个哨兵节点(进程)，共同监控数据节点的运行情况。
2. 同时哨兵节点之间也相互通信，交换对主从节点的监控状况。
3. 每隔1秒每个哨兵会向整个集群发送一个ping命令做一次心跳检测。

### 主从复制

![分布式系统之Redis主从架构](http://p1.pstatp.com/large/pgc-image/c3da100513f54a91b889642b27a47bda)

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3AzLnBzdGF0cC5jb20vbGFyZ2UvcGdjLWltYWdlLzA0MTQ5Yjc4NDQ4NTQyOGVhNTRhNzc4YjA4ZTJlZmQ1?x-oss-process=image/format,png)

1. 从数据库向主数据库发送`sync`命令
2. 主数据库接收到`sync`后，执行`bgsave`命令，保存快照，创建一个RDB文件
3. 当主数据库执行完快照后，会向从节点发送rdb文件，而从节点会接收并载入该文件。
4. 主节点将缓冲区的所有写命令发送非从节点执行。
5. 以上操作完成后，之后主节点每执行一个写命令，都会发送给从库。(增量复制)，当网络断开，也是增量复制的。

总结：

主从复制是为了数据备份，哨兵是为了高可用，redis主节点挂掉哨兵可以切换到从节点。集群模式是为了解决单机redis容量有限的问题，将数据按照一定的规则分配到多台机器，提高并发量。读写分离就是在master上进行写操作，在各个slave上进行读操作。

### redis cluster实现数据分区

分区分配规则一般有**哈希取余分区、一致性哈希分区、虚拟槽位分区**(reids-cluster采用的方式)

#### hash分区

是按照节点数目取余，余数相同的都会存储到一个节点上。`hash(key)%4`，当节点数目发生改变的时候，所有缓存在一定时间内是失效的，当应用无法从缓存中获取数据时，会向数据库请求。

#### 一致性hash分区

客户端进行分片，哈希+顺时针取余。

[https://blog.csdn.net/wlccomeon/article/details/86553831](https://blog.csdn.net/wlccomeon/article/details/86553831)

#### 虚拟槽位分区

分布式数据库要解决的就是将整块数据，按照分配规则分到多个缓存节点，解决的是单个缓存节点处理数据量大的问题。

槽slot是redis用来存放缓存信息的单位，在redis中将存储空间分成了16384`2^4*2^10`。缓存信息通常是按照key-value存储的，在存储信息的时候，集群会对key进行CRC16校验并对16384进行取模`slot=CRC16(key)%16384`。得到的结果就是key-value所放入的槽，从而实现自动分割数据到不同的节点上。然后再将这些槽分配到不同的缓存节点中。

![不懂Redis Cluster原理，我被同事diss了！](http://p3.pstatp.com/large/pgc-image/0b34af15848e4e339245959257f45dcd)

如图所示，假设有三个缓存节点分别是 1、2、3。Redis Cluster 将存放缓存数据的槽(Slot)分别放入这三个节点中：

- 缓存节点 1 存放的是(0-5000)Slot 的数据。
- 缓存节点 2 存放的是(5001-10000)Slot 的数据。
- 缓存节点 3 存放的是(10000-16383)Slot 的数据。

### redis过期淘汰策略

redis通过配置`maxmemory`来配置最大容量（阈值），当数据占有空间超过所设定的阈值就会触发内部的内存淘汰机制。究竟淘汰哪些数据？

- noeviction：当内存不足以容纳新数据时，新写入会报错。
- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。
- allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。
- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。(默认)
- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。
- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先删除。