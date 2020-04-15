---
layout:     post                    # 使用的布局（不需要改）
title:      Redis Cluster集群功能测试与踩坑分享        # 标题 
subtitle:   Redis  #副标题
date:       2020-04-15          # 时间
author:     MistRay                      # 作者
# header-img: img/post-bg-7.jpg 这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Redis
    - 缓存
---
## 前言
写这篇文章的契机是这样的，现就职公司的**Redis**是直接购买阿里的云服务，阿里的云服务大家都懂，三个字总结就是**好而贵**，
虽然云服务内部一定是做了高可用的，但对于使用者(开发人员)来说，这些都是隐形的，是透明的。云服务中的Redis是个黑盒，它甚至仅借用了Redis的概念，
内部是如何实现的，我们一概不知。

所以我对此产生了浓厚的兴趣，我想知道原生Redis集群可以做到何种程度，为的是一旦未来从阿里云迁移到自建服务，至少需要对集群心理有底。

### Redis Cluster 与 Sentinel
关于Redis Sentinel和Redis Cluster的问题，是一个会消失还是需要结合使用，下面将进行介绍。

#### Sentinel
>Redis supports multiple slaves replicating data from a master node. 
This provides a backup node which has your data on it, ready to serve data. 
However, in order to provide automated failover you need some tool. 
For Redis this tool is called Sentinel. 

Redis支持多个从节点从主节点复制数据。这提供了一个备份节点，上面有您的数据，可以随时提供数据。
但是，为了提供自动故障转移，您需要一些工具。对于Redis，此工具称为Sentinel。
简单的说**Sentinel**就是一个failover（故障转移）的工具。  

它的主要功能有以下几点：
* 不时地监控redis是否按照预期良好地运行;
* 如果发现某个redis节点运行出现状况，能够通知另外一个进程(例如它的客户端);
* 能够进行自动切换。当一个master节点不可用时，能够选举出master的多个slave(如果有超过一个slave的话)中的一个来作为新的master,其它的slave节点会将它所追随的master的地址改为被提升为master的slave的新地址。

Sentinel本身也支持集群，只使用单个sentinel进程来监控redis集群是不可靠的，当sentinel进程宕掉后，sentinel本身也有单点问题。所以有必要将sentinel集群，这样有几个好处：
* 如果只有一个sentinel进程，如果这个进程运行出错，或者是网络堵塞，那么将无法实现redis集群的主备切换（单点问题）。
* 如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于redis集群中的信息。
* sentinel集群自身也需要多数机制，也就是2个sentinel进程时，挂掉一个另一个就不可用了。

#### Redis Cluster
>The use cases for Cluster evolve around either spreading out load (specifically writes) and surpassing single-instance memory capabilities. 
If you have 2T of data, do not want to write sharding code in your access code, but have a library which supports Cluster then you probably want Redis Cluster. 
If you have a high write volume to a wide range of keys and your client library supports Cluster, Cluster will also be a good fit.
 
Cluster的用例围绕分散负载（特别是写入）和超越单实例内存功能而发展。如果您有2T的数据，并且不想在访问代码中编写分片代码，但是拥有一个支持Cluster的库，
那么您可能需要Redis Cluster。如果对大量键的写入量很高，并且客户端库支持Cluster，那么Cluster也将非常适合。

cluster的功能有已下几点：
* 一个 Redis 集群包含 16384 个哈希槽（hash slot），数据库中的每个键都属于这 16384 个哈希槽的其中一个，集群中的每个节点负责处理一部分哈希槽。 例如一个集群有三个节点，其中：  
    节点 A 负责处理 0 号至 5500 号哈希槽。  
    节点 B 负责处理 5501 号至 11000 号哈希槽。  
    节点 C 负责处理 11001 号至 16384 号哈希槽。  
    这种将哈希槽分布到不同节点的做法使得用户可以很容易地向集群中添加或者删除节点。例如：  
    如果用户将新节点 D 添加到集群中， 那么集群只需要将节点 A 、B 、 C 中的某些槽移动到节点 D 就可以了。  
    如果用户要从集群中移除节点 A ， 那么集群只需要将节点 A 中的所有哈希槽移动到节点 B 和节点 C ， 然后再移除空白（不包含任何哈希槽）的节点 A 就可以了。  
* Redis 集群对节点使用了主从复制功能： 集群中的每个节点都有 1 个至 N 个复制品（replica）， 其中一个复制品为主节点（master）， 而其余的 N-1 个复制品为从节点（slave）。  
* Redis 集群的节点间通过Gossip协议通信。

#### 对比总结
如果内存需求超过了系统内存，或者需要在多个节点之间分配写操作以保持性能水平，那么使用Redis Cluster。如果寻求高可用，则需要更多地部署Sentinel。
对于是否需要结合使用，答案是不需要，Redis Cluster集群自带failover。使用Cluster时不需要再用Sentinel。

### 集群部署
这里使用的是docker-compose启动的4主4从集群，细节可以参考[使用docker-compose搭建Redis集群](https://www.mistray.site/2019/08/15/Docker-Compose%E6%90%AD%E5%BB%BARedis%E9%9B%86%E7%BE%A4/)。  
![Redis集群](/img/post_img/post_2020_04_15_01Redis集群.png)

### 可用性测试
#### 主从切换
先使用`docker exec -it node-80 redis-cli -p 6380 cluster info`命令查看集群状态  

![Redis集群状态01](/img/post_img/post_2020_04_15_02Redis集群状态01.png)

从上图可以看出集群状态显示为ok，说明集群在正常运转。

再使用`docker exec -it node-80 redis-cli -p 6380 cluster nodes`命令查看各节点状态  

![Redis节点状态01](/img/post_img/post_2020_04_15_03Redis节点状态01.png)

从上图可以看出各节点的主从关系，80和85为主从关系，85为主，80为从。  

下面插入5条数据：
```java
        redisClusterUtil.set("MistRay1","6666");
        redisClusterUtil.set("MistRay2","6666");
        redisClusterUtil.set("MistRay3","6666");
        redisClusterUtil.set("MistRay4","6666");
        redisClusterUtil.set("MistRay5","6666");
```
![主从数据状态01](/img/post_img/post_2020_04_15_04主从数据状态01.png)

由上图可以看出数据已经成功插入了。  
然后，我将node-85节点停掉。

![停掉85后](/img/post_img/post_2020_04_15_05停掉85后.png)

由上图可以看到80已经从slave节点升级成了master节点，而且集群的状态依然为ok。

接着测试用客户端取集群内80和85分片上的数据。 
 
![停掉85后查询](/img/post_img/post_2020_04_15_06停掉85后查询.png)

结果也是ok的，可以查到对应的数据，所以证明了RedisCluster集群的failover能力。

你以为这就结束了？当然没有。  
当停止了80节点后，有趣的事情发生了。  

![Redis节点状态01](/img/post_img/post_2020_04_15_07停掉80后.png)

集群状态变为了fail，那岂不是整个集群都不可用了。我可是拥有4主4从的集群，
仅down了两台的情况下，就导致整个集群不可用，是不是十分不合理。
那假设我有100台，50主50从，仅down了两台的情况下，剩下98台就都用不了了？
这种重大决策，一定是要交到使用者手中的，根据不同的需求，来选择是否在失去数据完整性的情况下，让集群部分可用。
接着我查阅了大量的资料，最终在RedisCluster的官方文档上找到了答案。

>cluster-require-full-coverage <yes/no>: If this is set to yes, as it is by default, 
the cluster stops accepting writes if some percentage of the key space is not covered by any node. 
If the option is set to no, the cluster will still serve queries even if only requests about a subset of keys can be processed.

该配置项可以控制当前节点在集群状态为fail的情况下是否继续正常提供服务，默认值为yes。  
用刚才的测试举个例子就是：80和85都停了，如果该项设置为no那么集群中剩下的机器可以正常提供服务，反之亦然。

修改了该配置项后，停掉80和85  

![停掉80和85后](/img/post_img/post_2020_04_15_08停掉80和85后.png)

可以看出，集群的状态并没有变成fail，依然是ok。

## 总结
RedisCluster高可用有两种模式，当`cluster-require-full-coverage`设置为yes时，只要集群不具备数据完整性，
那么整个集群直接会进入不可用的状态。当`cluster-require-full-coverage`设置为no时，即使集群不具备数据完整性，
活着的分片依然可以被操作（读和写）。


## Reference
[使用docker-compose搭建Redis集群](https://www.mistray.site/2019/08/15/Docker-Compose%E6%90%AD%E5%BB%BARedis%E9%9B%86%E7%BE%A4/)  
[Redis Cluster vs.Sentinel](http://23.253.120.235/sentinel-or-cluster/)  
[Redis/sentinel/cluster](https://www.jianshu.com/p/faabfcdf825d)  
[Redis Sentinel & Redis Cluster - what?](https://fnordig.de/2015/06/01/redis-sentinel-and-redis-cluster/)  
[Redis cluster 添加 删除 重分配 节点](https://segmentfault.com/a/1190000014499174)  
[cluster-tutorial](https://redis.io/topics/cluster-tutorial)  
## 转载

本文遵循 [CC 4.0 by-sa](https://creativecommons.org/licenses/by-sa/4.0/) 版权协议,转载请附上原文出处链接和本声明。