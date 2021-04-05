# ETCD

## 1.1 写流程

写数据流程（以 leader 节点为例例）：

- etcd 任⼀一节点的etcd server 模块收到 Client 写请求（如果是follower 节点，会先通过 Raft 模块将**请求转发**⾄leader 节点处理理）
- etcd server 将请求封装为**Raft 请求**，然后提交给Raft 模块处理理。
- leader 通过Raft 协议与集群中 follower 节点进⾏行行交互，将**消息复制**到follower 节点，于此同时，**并行**将日志持久化到WAL。
- follower 节点对该请求进⾏响应，回复⾃己是否同意该请求。
- 当集群中**超过半数**节点（(n/2)+1 members ）同意接收这条⽇日志数据时，表示该请求可以被Commit，Raft 模块通知etcd server 该⽇志数据已经Commit，可以进⾏ **Apply**。
- 各个节点的etcd server 的 applierV3 模块异步进⾏ Apply 操作，并通过 MVCC 模块写⼊入后端存储BoltDB。
- 当client 所连接的节点数据apply 成功后，会返回给客户端 apply 的结果。

![image-20210306121954605](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306121956.png)

## 1.2 读数据流程

- etcd 任⼀一节点的 etcd server 模块收到客户端读请求（Range 请求）
- 判断读请求类型，如果是**串行化读**（serializable）则直接进⼊入 Apply 流程
- 如果是**线性⼀一致性读**（linearizable），则进⼊入 Raft 模块
- Raft 模块向 **leader** 发出 ReadIndex 请求，获取当前集群已经提交的最新数据 Index
- 等待本地 AppliedIndex ⼤大于或等于 ReadIndex 获取的**CommittedIndex** 时，进⼊ Apply 流程
- Apply 流程：通过 Key 名从 KV Index 模块获取 Key 最新的Revision，再通过 Revision 从 BoltDB 获取对应的 Key 和Value。

> - **串行化读：**
>
>   串行化是有关事务的保证，或者超过一个以上对象一个以上操作的综合(group)，它保证基于多个条目的一系列事务(通常是读写操作)执行等同于事务的一些串行执行(总顺序)。
>
>   串行化类似传统ACID中的“i”或isolation隔离，如果用户的事务每个保护应用正确性(这也是“C”，但是是ACID的C，代表一致性consistency)，一个串行化执行也保护正确性，这样，串行化是一种保证数据库正确性的机制。
>
>   不像线性化，串行化并不通过自身强加任何实时约束在事务的顺序上，串行化也不是可组合的，串行化并不意味着任何一种确定的顺序，它只是简单需要一些等价的串行执行存在。
>
> - **线性一致性读**
>
>   线性一致性并非限定在分布式语境下面，在单机单核环境下可以简单地理解为“寄存器”特性。寄存器中的变量可以被原子地被修改，且后续对该变量的读取会返回当次修改的值，直到其被再此修改，简单来说是**保证读到最近更新的值**。
>
>   拓展到分布式语境下，对于系统中的状态或变量发起修改不再是瞬间完成的，从调用开始到调用返回会有一段时间开销。如何判断一个分布式系统是具备线性一致性？首先我们约定好线性一致性条件下请求之间的相互关系：
>
>   - 对于调用**时间有交叠的并发请求**，生效的顺序可以任意确定，比如分别有并发的读写请求，那么读请求可以返回新值，也可以返回旧值。
>   - 对于**调用时间有偏序的请求**，比如请求B开始在请求A结束之后，那么请求B的结果不能违背请求A已确定的结果。

![image-20210306122241874](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306122243.png)

# 消息广播rexbc

## 特性

- 解决应用启动与广播接收的顺序问题。若应用初始化接收之后才开始接收广播，会有概率丢失部分广播消息。
- Failover 情况下， client 端重刷新。
- 消费端，消费重试到成功为止，当然也可以应用里选择丢弃该广播消息。
- 中间件监控与告警。

## 方案选型

### Dubbo 广播

优点：

- 没有 server 层，直接 client 端的广播，简单清晰。

缺点：

- 不保证广播的可靠性，即广播失败则不重试。
- 现有路由的限制，同个机房下的实例可相互广播，但不能跨机房通知。
- 广播与接收的逻辑是共用 dubbo 的业务线程池的，有一定的阻塞正常业务的风险。

### Redis

优点：

- 整体架构简单。

缺点：

- 不保证广播的可靠性，Client 端有连接超时等抖动导致重连或者 Server 端某个节点宕机，Client 端会丢失部分事件。

### Apollo

优点：

- 有事件触发，则会保证事件推送成功。
- 整体稳定性不错。

缺点：

- 推送成功是最终的数据推送成功，非过程中所有的事件都推送得到。
- 推送的数据量问题，若某个 namespace 下的一个 key 有更新操作，则会推送这个 namespace 下所有的 kv 数据给 client，包括没有发生变更的 kv 数据也会推送。
- 性能问题， 5000 client 关联的话，某个 namespace 下的 key 数量小于 10 ，全局 key 更新 qps 为 1 ，则该批次 apollo 推送完需要 10+ 秒 。

### NSQ

优点：

- 保证每个事件推送成功。
- 稳定性和运维能力高。

缺点：

- 默认的广播模式是 topic 级别广播，一般一个应用下的所有实例是订阅同一个 channel ，在NSQ机制里面只会让同一份 channel 里面的其中一台机器收到某条消息。若想达到全部机器实例都接收到消息，需要使用发布/订阅模式，即各个机器实例订阅不同的 channel 。 这让channel 与机器实例的关系维护代价大 ， 有两个点：
  - 第一个点是现在 nsq 的 channel 是需要手动创建的，应用扩容机器的时候会需要关注这个点。
  - 第二个点是机器实例与 channel 的关系维护很麻烦，因为后面 docker 容器化之后，还是需要运维来帮忙给各个机器实例打标签。
- 有临时 channel 的玩法，但是网络抖动的话，channel 会被清理然后用新的，需要触发一次重刷新操作。

### Kafka

#### 优点：

- 保证每个事件推送成功。
- 稳定性和运维能力（具体可靠性需要咨询运维同学）
- group 与 offset 的日志记录会自动过期删除，不需要人工去删除。

#### 缺点：

- 暂时没有发现。

### ETCD

使用ETCD集群作为广播事件数据存储，利用ETCD的watch机制通知各个client节点处理通知

#### 优点：

- ETCD自带mvcc机制，支持版本存储，能够顺序推送同一个节点的变更事件；
- 在ETCD集群无压力的情况下不需要proxy层，开发难度低；
- 有赞中间件使用ETCD集群的场景较多，熟悉程度较高，运维能力强；

#### 劣势：

- ETCD集群难以扩容，需要提前评估集群压力；
- ETCD的watch没有ack和重试机制，事件延迟与准确率依赖ETCD集群性能；
- 有赞中间件目前使用ETCD的场景与本方案相差较大，需要对高qps写入与watch的场景进行ETCD压力与性能评估；

### 总结

考虑到性能、可扩展性、运维支持以及 sla 能力，**最终选用 kafka** 。

# TMC

## 整体架构

TMC作为透明多级缓存的总称, 它包括业务无感知的各个缓存服务, 目前支持KV存储服务的本地缓存和RPC调用的本地缓存, 一方面是**提供热点的加速访问能力**, 一方面是提供**保护底层服务**的能力, 让服务不会因为某些热门数据影响到整体服务能力. 由于少量的热门数据在存储端很难横向扩展提供服务能力, 目前TMC是通过将热门数据加载到离上层业务更近的地方来提供热门数据的快速访问能力, 并减少到服务端的网络请求. 整体架构如下:

![tmc multi arch](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306123319.png)

![tmc arch](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306123326.png)

从上到下涉及多个层次的缓存能力, 目前包括几个主要部分: 对于Java业务来说, 所有的kv服务通过封装过的jedis客户端自动提供本地缓存功能, 所有的RPC调用通过Tether自动提供rpc本地缓存能力, 而对于非Java的php或者node业务, 目前都是通过tether来访问codis和kvds的, 因此不管是kv还是rpc都是tether提供的本地缓存功能来保证将热点数据加载到业务本地.

## KV本地缓存的热key计算和下发

Java封装的jedis sdk, 集成了热key计算能力, sdk会主动上报数据的访问统计, 由后台的热key计算集群统计出热门数据后, 自动下发到各个客户端, 客户端sdk会根据下发的统计结果来决定是否将某些数据进行本地缓存.=

为了动态控制本地缓存行为, sdk会动态的获取apollo配置来决定**本地缓存的key数量和内存占用**, 避免资源过多占用. 还会动态调整热点阈值, 来动态判断热点数据, 并且支持动态配置预热, 在活动前提前加载热点数据, 也支持黑名单, 防止某些数据被缓存.

新版本的TMC, 数据上报和热点下发统一走**cache notifier**服务, 不依赖etcd和天网日志通道.

Tether这边实现了热key的配置接口, 可以通过接口主动下发指定的热key, 并且支持本地热key自动探测能力, 提供保底的拦截热key能力. 大部分参数都可以动态调整适配不同的负载.

活动预热数据, 两个方式都支持设置白名单的方式提前加载数据到本地缓存.

## 关于缓存清理和一致性

由于本地缓存数据会分布到多台不同的业务机器上, 因此缓存清理和失效时间会存在一定窗口的不一致. 大部分本地缓存的数据ttl一般控制在3~5s左右, 因此被缓存的热点数据可能在此窗口期读到的是老数据.

缓存失效**Volcano**目前是通过etcd的watch触发所有客户端清理缓存的, 一般情况下在1s左右广播完成.

对于不能容忍这种短时间不一致的情况, 也可以配置黑名单, 加载本地缓存时将不会缓存此类数据.注意本地缓存自动加载只会加载热门的数据, 非热门数据无需配置黑名单.

新版本的TMC, 缓存失效统一走失效广播服务, 不依赖etcd, 失效广播服务会聚合多次相同的失效请求.

## 热点探测组件

热点探测组件是由一个叫做 volcano 的应用来承接的，主要实现了key访问的两条路径：

对get操作的上报 -> 计算 -> 热key发现 -> 热key下发

对 delete/set/expire 操作的 探测 -> 本地热key失效 -> 广播 -> 分组下其他机器热key失效

#### 老通道

![tmc Volcano](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306123506.png)

一方面：

```text
client 通过 skynet-client 直连天网搭建的 tmc-rsyslog 上报本地key的访问数据，
rsyslog聚合生产到kafka

volcano-server通过消费kafka进行近实时的计算

计算结果 写到etcd 对应groupName的路径下

client通过watch etcd对应路径，拿到相应的事件

判断内容是FIND事件，则本地标记为热key
```

另一方面：

```text
client通过感知 delete/set/expire 事件，通过etcd写入group下对应key的失效事件

其他client watch到该事件后 删除该key的本地热key标记
```

#### 新通道

![tmc Volcano](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306123528.png)

该方式目前只适合服务化的应用，client会暴露一个dubbo接口:`com.youzan.volcano.sdk.${application.name}.HermesHotKeyService`

client的上报下发，都统一走tether侧的一个 `cache-notifier` 组件

- client 将请求数据通过 grpc 聚合上报给 `cache-notifier`
- `cache-notifier` 会二次聚合上报数据，再通过 grpc 发送到 `volcano-server`
- `volcano-server`再将热key计算结果发到 `cache-notifier`
- `cache-notifier`通过`pilot`拿到待下发数据的所有机器，通过tether并发广播
- client通过感知 delete/set/expire 事件，将过期通知通过`cache-notifier`的并发服务化广播能力广播到该group下所有应用的其他机器上

## cache-notifier

1. 解决什么问题？

   TMC 旧架构强依赖 etcd 和kafka

   1. etcd write 1w/s， 击穿时 tmc 产生的 变更事件远超这个量级；
   2. kafka 链路长，延迟较高且潜在风险较高。

2. 架构图
   ![企业微信截图_eadb92c4-6235-4188-b3eb-59743a5571b6](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306124049.png)

3. 功能点：

   1. 按 group+key 维度聚合
   2. 跨注册中心的需求，采用 notifier 互调实现
   3. 对于 kvproxy 的 hotkey error，触发立即下发热点key的逻辑

4. notifier 内存泄露问题优化

   1. 原因
      1. 双缓冲队列：通过覆盖写的方式，只能解决突发流量洪峰，丢弃 limit 以外的流量；
      2. 下游 sweep（Tether 广播或者上报 volcano-server）性能受限时，内存中的 message 会产生积压，造成泄漏；
   2. 解决方法：
      1. use LRU to replace debounced data map 

# grpc

站在2021年，再来看 gRPC。

我们梳理一下这三年 gRPC 的发展：

- 现在的 gRPC 是 CNCF 基金会下的孵化项目
- 智能代理 Envoy 将 gRPC 作为一等公民，并且 envoy 与控制层交互的 xDS 基于 gRPC 实现
- Istio 等 Service Mesh 解决方案将 gRPC 作为一等公民
- gRPC 开始支持xDS协议，实现无代理Service Mesh
- dubbogo 3.0：牵手 gRPC 走向云原生时代
- CNCF 推出了 UDPA （“Universal Data Plane API”的缩写， “通用数据平面API），该规范基于gRPC 实现，API将涵盖服务发现，负载均衡分配，路由发现，监听器配置，安全发现，负载报告，运行状况检查委托等。这意味着未来服务注册和发现中心都需要支持gRPC。比如nacos2.0 通信层通过gRPC 实现长连接RPC 调用和推送能力。

通过这些发展，我们可以看出，**谷歌正在把gRPC打造成云原生时代通信层事实上的标准**。

其他回答里，有的同学提到：”**grpc和thrift区别真心不太大，差的最多的是你自己要填坑的那些东西：连接池、服务框架、服务发现、服务治理、trace、打点、context日志……“**

那么很明显谷歌给出的解决方案是Istio。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20210306131009.jpg)