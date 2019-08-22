---
layout:     post                    # 使用的布局（不需要改）
title:      使用docker-compose搭建Redis集群           # 标题 
subtitle:   docker-compose&redis #副标题
date:       2019-08-15              # 时间
author:     MistRay                      # 作者
header-img: img/home-bg.jpg   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - redis
    - docker
    - docker-compose
---

![redis](/img/docker_redis4.png)
`使用docker-compose一键搭建Redis集群`  
>[Demo](https://github.com/MistRay/redis-docker-compose)已上传github.

### 1.简介

大部分情况开发者不会在本地搭建集群.但有时在我们需要进行集群API测试,对集群性能及可用性验证,学习Redis集群特性等情况下
需要在本地搭建Redis集群,本项目诣在帮助没有集群部署经验的开发人员可以快速在本地搭建Redis集群

### 2.Redis集群
Redis集群可以把数据分散存储到n个节点中,同时可以对每个节点做备份,来保障Redis数据的高可用和稳定性

### 3.集群方案比较
##### redis高可用集群  
Redis集群是一个由多个主从节点群组成的分布式服务器群，它具有复制、高可用和分片特性。
Redis集群不需要sentinel哨兵也能完成节点移除和故障转移的功能。需要将每个节点设置成集群模式，
这种集群模式没有中心节点，可水平扩展，据官方文档称可以线性扩展到上万个节点(官方推荐不超过1000个节点)。
redis集群的性能和高可用性均优于之前版本的哨兵模式，且集群配置非常简单。
![RedisCluster](/img/post_img/redis_cluster.png)

##### redis哨兵集群  
在redis3.0以前的版本要实现集群一般是借助哨兵sentinel工具来监控master节点的状态，
如果master节点异常，则会做主从切换，将某一台slave作为master，哨兵的配置略微复杂，
并且性能和高可用性等各方面表现一般，特别是在主从切换的瞬间存在访问瞬断的情况，
而且哨兵模式只有一个主节点对外提供服务，没法支持很高的并发，且单个主节点内存也不宜设置得过大，
否则会导致持久化文件过大，影响数据恢复或主从同步的效率。
![redis_sentinel](/img/post_img/redis_sentinel.png)


### 4.快速开始

1.安装依赖: [docker](https://www.docker.com/),[docker-compose](https://docs.docker.com/compose/install/)及[python3.X](https://www.python.org/downloads/)  

2.启动
```shell
# 需在docker-compose.yml所在的文件夹下执行
docker-compose up -d 

docker exec -it node-80 redis-cli -p 6380 --cluster create 172.16.238.10:6380 172.16.238.11:6381 172.16.238.12:6382 172.16.238.13:6383 172.16.238.14:6384 172.16.238.15:6385 --cluster-replicas 1
```

### 5.说明

* 使用Redis版本为5.0.5-alpine,如有需要请自行更改.  
> redis官方在redis3.x和redis4.x提供了redis-trib.rb工具方便我们快速搭建集群,在redis5.x中更是可以直接使用redis-cli命令来直接完成集群的一键搭建,省去了redis-trib.rb依赖ruby环境的问题。
* docker-compose自动组网
> 使用docker-compose up启动容器后，这些容器都会被加入`{app_name}_default`网络中,但是ip不固定.所以该项目在docker-compose内使用了指定ip地址的方式,使每个容器的ip为定值.
* client端节点互通问题  
在redis配置文件中有如下配置:
```
# docker虚拟网卡的ip
cluster-announce-ip 10.1.1.5
# 节点映射端口
cluster-announce-port 6379
# 总线映射端口,通常为节点映射端口前加1
cluster-announce-bus-port 16379
```
> docker虚拟网卡地址为docker与外界互通所使用的虚拟ip,如果没有上述配置,在client端操作时,会有取docker内网ip的问题.
* 关于时区,项目内默认使用了上海时区,需要使用其他时区的同学请自行更改.
```yaml
    environment:
      # 设置时区为上海
      - TZ=Asia/Shanghai
```
### 6.Reference

* [medium.com](https://medium.com/@dhammikasamankumara/getting-started-with-redis-cluster-on-windows-6435d0ffd87)
* [redis.io](https://redis.io/topics/cluster-tutorial)
* [www.g5niusx.com](https://www.g5niusx.com/2019/04/redis-4.html)
* [juejin.im](https://juejin.im/post/5d4afaaf518825403769dd44)

### 7.转载
本文遵循 [CC 4.0 by-sa](https://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。