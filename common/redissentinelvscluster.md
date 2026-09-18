## 哨兵模式 Sentinel

Redis-Sentinel(哨兵模式)是Redis官方推荐的高可用性(HA)解决方案，当用Redis做Master-slave的高可用方案时，假如master宕机了，Redis本身(包括它的很多客户端)都没有实现自动进行主备切换，而Redis-sentinel本身也是一个独立运行的进程，它能监控多个master-slave集群，发现master宕机后能进行自懂切换。它的主要功能有以下几点：

- 不时地监控redis是否按照预期良好地运行;
- 如果发现某个redis节点运行出现状况，能够通知另外一个进程(例如它的客户端);
- 能够进行自动切换。当一个master节点不可用时，能够选举出master的多个slave(如果有超过一个slave的话)中的一个来作为新的master,其它的slave节点会将它所追随的master的地址改为被提升为master的slave的新地址。

Redis master-slave 模式如下图：

![](6743196bce7607ca010b8269b1019af7.png)

从上图片中可以看到，一个master 节点可以挂多个slave ，Redis Sentinel 管理Redis 节点结构如下：

![](bfeae3b0d4378e5f021e53ce1296e2dd.png)

从上图中可以得出Sentinel其实就是Client和Redis之间的桥梁，所有的客户端都通过Sentinel程序获取Redis的Master服务。首先Sentinel是集群部署的，Client可以链接任何一个Sentinel服务所获的结果都是一致的。其次，所有的Sentinel服务都会对Redis的主从服务进行监控，当监控到Master服务无响应的时候，Sentinel内部进行仲裁，从所有的 Slave选举出一个做为新的Master。并且把其他的slave作为新的Master的Slave。最后通知所有的客户端新的Master服务地址。如果旧的Master服务地址重新启动，这个时候，它将被设置为Slave服务。

Sentinel 可以管理master-slave节点，看似Redis的稳定性得到一个比较好的保障。但是如果Sentinel是单节点的话，如果Sentinel宕机了，那master-slave这种模式就不能发挥其作用了。幸好Sentinel也支持集群模式，Sentinel的集群模式主要有以下几个好处：

- 即使有一些sentinel进程宕掉了，依然可以进行redis集群的主备切换；
- 如果只有一个sentinel进程，如果这个进程运行出错，或者是网络堵塞，那么将无法实现redis集群的主备切换（单点问题）;
- 如果有多个sentinel，redis的客户端可以随意地连接任意一个sentinel来获得关于redis集群中的信息。

Redis Sentinel 集群模式可以增强整个Redis集群的稳定性与可靠性，但是当某个节点的master节点挂了要重新选取出新的master节点时，Redis Sentinel的集群模式选取的复杂度显然高于单点的Redis Sentinel 模式，此时需要一个比较靠谱的选取算法。下面就来介绍Redis Sentinel 集群模式的 “仲裁会”(多个Redis Sentinel共同商量谁是Redis 的 master节点)

1.1、Redis Sentinel 集群模式的 “仲裁会”

当一个master被sentinel集群监控时，需要为它指定一个参数，这个参数指定了当需要判决master为不可用，并且进行failover时，所需要的sentinel数量，本文中我们暂时称这个参数为票数,不过，当failover主备切换真正被触发后，failover并不会马上进行，还需要sentinel中的大多数sentinel授权后才可以进行failover。当ODOWN时，failover被触发。failover一旦被触发，尝试去进行failover的sentinel会去获得“大多数”sentinel的授权（如果票数比大多数还要大的时候，则询问更多的sentinel)这个区别看起来很微妙，但是很容易理解和使用。例如，集群中有5个sentinel，票数被设置为2，当2个sentinel认为一个master已经不可用了以后，将会触发failover，但是，进行failover的那个sentinel必须先获得至少3个sentinel的授权才可以实行failover。如果票数被设置为5，要达到ODOWN状态，必须所有5个sentinel都主观认为master为不可用，要进行failover，那么得获得所有5个sentinel的授权。

## 核心区别对比

### 1. 架构设计差异

|**维度**|**Redis Cluster**|**Redis Sentinel**|
|---|---|---|
|数据分布|自动分片（16384个哈希槽）|主从复制（全量数据副本）|
|节点角色|所有节点平等（无中心节点）|明确的主从角色+监控节点|
|客户端支持|需要支持集群协议的客户端|兼容所有 Redis 客户端|
|扩展方式|水平扩展（增加分片）|垂直扩展（提升单节点性能）|

### 2. 容错能力对比

**故障恢复速度**：

- Cluster：秒级自动故障转移（依赖Gossip协议）
- Sentinel：分钟级切换（需多数Sentinel节点确认）

**数据一致性**：

- Cluster：异步复制，可能丢失少量数据
- Sentinel：同步复制（需配置`wait`命令）

**典型故障场景**：

- **主节点宕机时**  
    Cluster：自动选举新主，客户端自动重定向到新主节点  
    Sentinel：需要等待多数Sentinel节点确认故障，人工确认切换结果

### 3. 性能表现差异

| **场景**  | Cluster 吞吐量 | Sentinel 吞吐量 |
| ------- | ----------- | ------------ |
| 10节点读操作 | 120万 QPS    | 80万 QPS      |
| 跨节点事务   | 不支持         | 支持           |
| 热点数据访问  | 可能倾斜        | 均匀分布         |

## 选型决策因素

### 1. 数据规模边界

- **≤100GB**：Sentinel + 主从复制
- **＞100GB**：Cluster 自动分片
- **＞1TB**：Cluster + Proxy 分层架构

### 2. 业务特性匹配

**适合 Cluster 的场景**：

- 需要水平扩展（如社交平台用户数据）
- 读写比 8:2 以上的读密集型业务
- 允许少量数据丢失（如缓存场景）

**适合 Sentinel 的场景**：

- 强一致性要求（如金融交易记录）
- 事务操作频繁（如库存扣减）
- 已有客户端不支持集群协议

### 3. 运维复杂度

**Cluster 管理难点**：

- 扩缩容需迁移哈希槽
- 跨机房部署需要定制工具
- 监控指标分散（需聚合）

**Sentinel 管理优势**：

- 配置变更简单
- 监控体系成熟
- 故障排查直观

### 4. 成本对比

|**成本类型**|Cluster 方案|Sentinel 方案|
|---|---|---|
|硬件成本|节点数多 30%|冗余副本较少|
|人力成本|需要专职 DBA|普通运维可维护|
|迁移成本|数据重分布耗时|主从同步即可|

### 5. 生态兼容性

**常见兼容问题**：

- Cluster 不支持跨节点事务
- 部分客户端需要升级（如 Jedis 3.x+）
- 监控工具需要适配（如 Prometheus）

### Redis 集群架构选型流程图

![Redis 集群架构选型流程图](redis-cluster-vs-sentinel-choice.png)

## 生产环境配置示例

### Redis Cluster 部署方案

```text
6节点部署（3主3从）：
北京机房：
├─ 主A（槽0-5460）
├─ 从A1
上海机房：
├─ 主B（槽5461-10922）
├─ 从B1
广州机房：
├─ 主C（槽10923-16383）
├─ 从C1
```

### Redis Sentinel 高可用方案

```text
5节点部署：
主节点：redis-master
从节点：redis-replica1, redis-replica2
Sentinel节点：sentinel1, sentinel2, sentinel3（部署在独立服务器）
```

## 经典踩坑案例

### Case 1: 集群脑裂问题

**现象**：某电商大促期间，Cluster 跨机房网络中断导致数据不一致  
**解决方案**：

1. 设置`cluster-node-timeout 15000`
2. 配置`cluster-replica-validity-factor 10`
3. 启用`cluster-require-full-coverage no`

### Case 2: Sentinel误切换

**现象**：网络抖动导致 Sentinel 误判主节点下线  
**优化方案**：

```properties
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

## 总结建议

**下述场景选择 Redis Cluster**：

- 数据量超过单机内存容量
- 需要线性扩展读写能力
- 能够接受客户端改造

**下述场景选择 Redis Sentinel**：

- 数据规模可控（＜200GB）
- 需要完整 Redis 特性支持
- 已有基础设施不支持集群协议

**混合架构趋势**：大型互联网公司通常同时使用两种方案，用 Cluster 承载海量数据，Sentinel 保证核心业务的高可用。
