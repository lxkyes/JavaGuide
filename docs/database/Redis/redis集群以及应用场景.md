相关阅读：

- [史上最全Redis高可用技术解决方案大全](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247484850&idx=1&sn=3238360bfa8105cf758dcf7354af2814&chksm=cea24a79f9d5c36fb2399aafa91d7fb2699b5006d8d037fe8aaf2e5577ff20ae322868b04a87&token=1082669959&lang=zh_CN&scene=21#wechat_redirect)
- [Raft协议实战之Redis Sentinel的选举Leader源码解析](http://weizijun.cn/2015/04/30/Raft%E5%8D%8F%E8%AE%AE%E5%AE%9E%E6%88%98%E4%B9%8BRedis%20Sentinel%E7%9A%84%E9%80%89%E4%B8%BELeader%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)

目录：

<!-- TOC -->

- [Redis 集群以及应用](#redis-集群以及应用)
    - [集群](#集群)
        - [主从复制](#主从复制)
            - [主从链(拓扑结构)](#主从链拓扑结构)
            - [复制模式](#复制模式)
            - [问题点](#问题点)
        - [哨兵机制](#哨兵机制)
            - [拓扑图](#拓扑图)
            - [节点下线](#节点下线)
            - [Leader选举](#Leader选举)
            - [故障转移](#故障转移)
            - [读写分离](#读写分离)
            - [定时任务](#定时任务)
        - [分布式集群(Cluster)](#分布式集群cluster)
            - [拓扑图](#拓扑图)
            - [通讯](#通讯)
                - [集中式](#集中式)
                - [Gossip](#gossip)
            - [寻址分片](#寻址分片)
                - [hash取模](#hash取模)
                - [一致性hash](#一致性hash)
                - [hash槽](#hash槽)
    - [使用场景](#使用场景)
        - [热点数据](#热点数据)
        - [会话维持 Session](#会话维持-session)
        - [分布式锁 SETNX](#分布式锁-setnx)
        - [表缓存](#表缓存)
        - [消息队列 list](#消息队列-list)
        - [计数器 string](#计数器-string)
    - [缓存设计](#缓存设计)
        - [更新策略](#更新策略)
        - [更新一致性](#更新一致性)
        - [缓存粒度](#缓存粒度)
        - [缓存穿透](#缓存穿透)
            - [解决方案](#解决方案)
        - [缓存雪崩](#缓存雪崩)
            - [出现后应对](#出现后应对)
            - [请求过程](#请求过程)

<!-- /MarkdownTOC -->

# Redis 集群以及应用

## 集群

### 主从复制

==1）主节点Master**可读、可写.**==

==（2）从节点Slave**只读**。（read-only）==

==== 

==因此，主从模型可以提高**读的能力**，在一定程度上缓解了写的能力。因为能写仍然只有Master节点一个，可以将读的操作全部移交到从节点上，变相提高了写能力。==

#### 主从链(拓扑结构)



![主从](https://user-images.githubusercontent.com/26766909/67539461-d1a26c00-f714-11e9-81ae-61fa89faf156.png)

![主从](https://user-images.githubusercontent.com/26766909/67539485-e0891e80-f714-11e9-8980-d253239fcd8b.png)

### 哨兵机制

Redis 的 Sentinel （哨兵）系统用于管理多个 Redis 服务器（instance）， 该系统执行以下三个任务：

**监控（Monitoring****）**： Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。

**提醒（Notification****）**： 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。

**自动故障迁移（Automatic failover****）**： 当一个==主服务器==不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会进行选举，==将其中一个从服务器升级为新的主服务器==， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

#### 拓扑图

![哨兵机制-拓扑图](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-11/哨兵机制-拓扑图.png)

### 分布式集群(Cluster)

#### 拓扑图

![image](https://user-images.githubusercontent.com/26766909/67539510-f8f93900-f714-11e9-9d8d-08afdecff95a.png)

sentinel模式基本可以满足一般生产的需求，具备高可用性。==但是当数据量过大到一台服务器存放不下的情况时，主从模式或sentinel模式就不能满足需求了，这个时候需要对存储的数据进行分片，将数据存储到多个Redis实例中==。cluster模式的出现就是为了解决单机Redis容量有限的问题，将Redis的数据根据一定的规则分配到多台机器。


## 使用场景

### 热点数据

存取数据优先从 Redis 操作，如果不存在再从文件（例如 MySQL）中操作，从文件操作完后将数据存储到 Redis 中并返回。同时有个定时任务后台定时扫描 Redis 的 key，根据业务规则进行淘汰，防止某些只访问一两次的数据一直存在 Redis 中。
>例如使用 Zset 数据结构，存储 Key 的访问次数/最后访问时间作为 Score，最后做排序，来淘汰那些最少访问的 Key。  

如果企业级应用，可以参考：[阿里云的 Redis 混合存储版][1]

### 会话维持 Session

会话维持 Session 场景，即使用 Redis 作为分布式场景下的登录中心存储应用。每次不同的服务在登录的时候，都会去统一的 Redis 去验证 Session 是否正确。但是在微服务场景，一般会考虑 Redis + JWT 做 Oauth2 模块。
>其中 Redis 存储 JWT 的相关信息主要是留出口子，方便以后做统一的防刷接口，或者做登录设备限制等。

### 分布式锁 SETNX

命令格式：`SETNX key value`：当且仅当 key 不存在，将 key 的值设为 value。若给定的 key 已经存在，则 SETNX 不做任何动作。

1. 超时时间设置：获取锁的同时，启动守护线程，使用 expire 进行定时更新超时时间。如果该业务机器宕机，守护线程也挂掉，这样也会自动过期。如果该业务不是宕机，而是真的需要这么久的操作时间，那么增加超时时间在业务上也是可以接受的，但是肯定有个最大的阈值。
2. 但是为了增加高可用，需要使用多台 Redis，就增加了复杂性，就可以参考 Redlock：[Redlock分布式锁](Redlock分布式锁.md#怎么在单节点上实现分布式锁)

### 表缓存

Redis 缓存表的场景有黑名单、禁言表等。访问频率较高，即读高。根据业务需求，可以使用后台定时任务定时刷新 Redis 的缓存表数据。

### 消息队列 list

主要使用了 List 数据结构。  
List 支持在头部和尾部操作，因此可以实现简单的消息队列。
1. 发消息：在 List 尾部塞入数据。
2. 消费消息：在 List 头部拿出数据。

同时可以使用多个 List，来实现多个队列，根据不同的业务消息，塞入不同的 List，来增加吞吐量。

### 计数器 string

主要使用了 INCR、DECR、INCRBY、DECRBY 方法。

INCR key：给 key 的 value 值增加一 
DECR key：给 key 的 value 值减去一

## 缓存设计

### 更新策略

- LRU、LFU、FIFO 算法自动清除：一致性最差，维护成本低。
- 超时自动清除(key expire)：一致性较差，维护成本低。
- 主动更新：代码层面控制生命周期，一致性最好，维护成本高。

在 Redis 根据在 redis.conf 的参数 `maxmemory` 来做更新淘汰策略：
1. noeviction: 不删除策略, 达到最大内存限制时, 如果需要更多内存, 直接返回错误信息。大多数写命令都会导致占用更多的内存(有极少数会例外, 如 DEL 命令)。
2. allkeys-lru: 所有 key 通用; 优先删除最近最少使用(less recently used ,LRU) 的 key。
3. volatile-lru: 只限于设置了 expire 的部分; 优先删除最近最少使用(less recently used ,LRU) 的 key。
4. allkeys-random: 所有key通用; 随机删除一部分 key。
5. volatile-random: 只限于设置了 expire 的部分; 随机删除一部分 key。
6. volatile-ttl: 只限于设置了 expire 的部分; 优先删除剩余时间(time to live,TTL) 短的key。

### 更新一致性

- 读请求：先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应。
- 写请求：先删除缓存，然后再更新数据库(避免大量地写、却又不经常读的数据导致缓存频繁更新)。

### 缓存粒度

- 通用性：全量属性更好。
- 占用空间：部分属性更好。
- 代码维护成本。



