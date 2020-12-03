## 前言

使用场景：

- 异步

  将同步的逻辑处理变成异步，减少系统耗时，避免长事务，降低业务间的耦合。

- 削峰

  在秒杀系统，或瞬时并发比较大的系统上，为了避免流量打挂服务器，可以将请求放到消息队列中。

- 解耦

  通过消息队列降低耦合度，提高扩展性。

带来的问题：

- 重复消费

  消费方需要做幂等性校验，如果是强校验场景需借助于DB，弱校验场景可以使用redis缓存校验。

- 消息丢失

  `RocketMQ`支持同步消息，当消息持久化了以后再响应结果，可以保证消息不丢失

- 顺序消费

  `RocketMQ`支持局部顺序与全局顺序，局部顺序采用的是将消息放到同一个queue，提供了`MessageQueueSelector`（hash取模法），全局顺序采用的是设置topic的队列数为1，在发送时保证有序即可。

- 数据一致性

  分布式服务都会存在该问题，一般采用分布式事务解决，如：`Seata`

- 可用性

  `RocketMQ`支持多master、多slave的部署结构，可以保障高可用。

各类MQ产品比较：

**ActiveMQ**和**RabbitMQ**这两着因为吞吐量还有**GitHub**的社区活跃度的原因，在各大互联网公司都已经基本上绝迹了

![image-20200518114945474](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518114945474.png)

## RocketMQ

**RocketMQ天生为金融互联网领域而生**，追求高可靠、高可用、高并发、低延迟，是一个阿里巴巴由内而外成功孕育的典范，除了阿里集团上千个应用外，根据我们不完全统计，国内至少有上百家单位、科研教育机构在使用。

![image-20200518115228470](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518115228470.png)

### NameServer：

> 主要负责对于源数据的管理，包括了对于**Topic**和路由信息的管理。

**NameServer**是一个功能齐全的服务器，其角色类似Dubbo中的Zookeeper，但NameServer与Zookeeper相比更轻量。主要是因为每个NameServer节点互相之间是独立的，没有任何信息交互。

**NameServer**压力不会太大，平时主要开销是在维持心跳和提供Topic-Broker的关系数据。

但有一点需要注意，Broker向NameServer发心跳时， 会带上当前自己所负责的所有**Topic**信息，如果**Topic**个数太多（万级别），会导致一次心跳中，就Topic的数据就几十M，网络情况差的话， 网络传输失败，心跳失败，导致NameServer误认为Broker心跳失败。

**NameServer** 被设计成几乎无状态的，可以横向扩展，节点之间相互之间无通信，通过部署多台机器来标记自己是一个伪集群。

每个 Broker 在启动的时候会到 NameServer 注册，Producer 在发送消息前会根据 Topic 到 **NameServer** 获取到 Broker 的路由信息，Consumer 也会定时获取 Topic 的路由信息。

### Producer

> 消息生产者，负责产生消息，一般由业务系统负责产生消息。

- **Producer**由用户进行分布式部署，消息由**Producer**通过多种负载均衡模式发送到**Broker**集群，发送低延时，支持快速失败。
- **RocketMQ** 提供了三种方式发送消息：同步、异步和单向
  - **同步发送**：同步发送指消息发送方发出数据后会在收到接收方发回响应之后才发下一个数据包。一般用于重要通知消息，例如重要通知邮件、营销短信。
  - **异步发送**：异步发送指发送方发出数据后，不等接收方发回响应，接着发送下个数据包，一般用于可能链路耗时较长而对响应时间敏感的业务场景，例如用户视频上传后通知启动转码服务。
  - **单向发送**：单向发送是指只负责发送消息而不等待服务器回应且没有回调函数触发，适用于某些耗时非常短但对可靠性要求并不高的场景，例如日志收集。

### Broker

> 消息中转角色，负责**存储消息**，转发消息。

- **Broker**是具体提供业务的服务器，单个Broker节点与所有的NameServer节点保持长连接及心跳，并会定时将**Topic**信息注册到NameServer，顺带一提底层的通信和连接都是**基于Netty实现**的。
- **Broker**负责消息存储，以Topic为纬度支持轻量级的队列，单机可以支撑上万队列规模，支持消息推拉模型。
- 官网上有数据显示：具有**上亿级消息堆积能力**，同时可**严格保证消息的有序性**。

### Consumer

> 消息消费者，负责消费消息，一般是后台系统负责异步消费。

- **Consumer**也由用户部署，支持PUSH和PULL两种消费模式，支持**集群消费**和**广播消息**，提供**实时的消息订阅机制**。
- **Pull**：拉取型消费者（Pull Consumer）主动从消息服务器拉取信息，只要批量拉取到消息，用户应用就会启动消费过程，所以 Pull 称为主动消费型。
- **Push**：推送型消费者（Push Consumer）封装了消息的拉取、消费进度和其他的内部维护工作，将消息到达时执行的回调接口留给用户应用程序来实现。所以 Push 称为被动消费类型，但从实现上看还是从消息服务器中拉取消息，不同于 Pull 的是 Push 首先要注册消费监听器，当监听器处触发后才开始消费消息。

### 消息领域模型

![image-20200518120544145](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518120544145.png)

#### Message

**Message**（消息）就是要传输的信息。

一条消息必须有一个主题（Topic），主题可以看做是你的信件要邮寄的地址。

一条消息也可以拥有一个可选的标签（Tag）和额处的键值对，它们可以用于设置一个业务 Key 并在 Broker 上查找此消息以便在开发期间查找问题。

#### Topic

**Topic**（主题）可以看做消息的规类，它是消息的第一级类型。比如一个电商系统可以分为：交易消息、物流消息等，一条消息必须有一个 Topic 。

**Topic** 与生产者和消费者的关系非常松散，一个 Topic 可以有0个、1个、多个生产者向其发送消息，一个生产者也可以同时向不同的 Topic 发送消息。

一个 Topic 也可以被 0个、1个、多个消费者订阅。

#### Tag

**Tag**（标签）可以看作子主题，它是消息的第二级类型，用于为用户提供额外的灵活性。使用标签，同一业务模块不同目的的消息就可以用相同 Topic 而不同的 **Tag** 来标识。比如交易消息又可以分为：交易创建消息、交易完成消息等，一条消息可以没有 **Tag** 。

标签有助于保持您的代码干净和连贯，并且还可以为 **RocketMQ** 提供的查询系统提供帮助。

#### Group

分组，一个组可以订阅多个Topic。

分为ProducerGroup，ConsumerGroup，代表某一类的生产者和消费者，一般来说同一个服务可以作为Group，同一个Group一般来说发送和消费的消息都是一样的

#### Queue

在**Kafka**中叫Partition，每个Queue内部是有序的，在**RocketMQ**中分为读和写两种队列，一般来说读写队列数量一致，如果不一致就会出现很多问题。

#### Message Queue

**Message Queue**（消息队列），主题被划分为一个或多个子主题，即消息队列。

一个 Topic 下可以设置多个消息队列，发送消息时执行该消息的 Topic ，RocketMQ 会轮询该 Topic 下的所有队列将消息发出去。

消息的物理管理单位。一个Topic下可以有多个Queue，Queue的引入使得消息的存储可以分布式集群化，具有了水平扩展能力。

#### Offset

在**RocketMQ** 中，所有消息队列都是持久化，长度无限的数据结构，所谓长度无限是指队列中的每个存储单元都是定长，访问其中的存储单元使用Offset 来访问，Offset 为 java long 类型，64 位，理论上在 100年内不会溢出，所以认为是长度无限。

也可以认为 Message Queue 是一个长度无限的数组，**Offset** 就是下标。

#### 消息消费模式

消息消费模式有两种：**Clustering**（集群消费）和**Broadcasting**（广播消费）。

默认情况下就是集群消费，该模式下一个消费者集群共同消费一个主题的多个队列，一个队列只会被一个消费者消费，如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。

而广播消费消息会发给消费者组中的每一个消费者进行消费。

#### Message Order

**Message Order**（消息顺序）有两种：**Orderly**（顺序消费）和**Concurrently**（并行消费）。

顺序消费表示消息消费的顺序同生产者为每个消息队列发送的顺序一致，所以如果正在处理全局顺序是强制性的场景，需要确保使用的主题只有一个消息队列。

并行消费不再保证消息顺序，消费的最大并行数量受每个消费者客户端指定的线程池限制。

### 通信流程

Producer 与 NameServer集群中的其中一个节点（随机选择）建立长连接，定期从 NameServer 获取 **Topic** 路由信息，并向提供 Topic 服务的 **Broker Master** 建立长连接，且定时向 **Broker** 发送心跳。

**Producer** 只能将消息发送到 Broker master，但是 **Consumer** 则不一样，它同时和提供 Topic 服务的 Master 和 Slave建立长连接，既可以从 Broker Master 订阅消息，也可以从 Broker Slave 订阅消息。

![image-20200518121503538](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518121503538.png)



#### NameServer启动流程

在org.apache.rocketmq.namesrv目录下的**NamesrvStartup**这个启动类基本上描述了他的启动过程我们可以看一下代码：

- 第一步是初始化配置
- 创建**NamesrvController**实例，并开启两个定时任务：
  - 每隔10s扫描一次**Broker**，移除处于不激活的**Broker**；
  - 每隔10s打印一次KV配置。
- 第三步注册钩子函数，启动服务器并监听Broker。

**NameServer还有很多东西的哈我这里就介绍他的启动流程，大家还可以去看看代码，还是很有意思的，比如路由注册会发送心跳包，还有**心跳包的处理流程**，**路由删除**，**路由发现**等等。

#### Producer

![image-20200518121748438](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518121748438.png)

通过轮训，**Producer**轮训某个**Topic**下面的所有队列实现发送方的负载均衡

![image-20200518121815745](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518121815745.png)

#### Broker

**Broker**在RocketMQ中是进行处理Producer发送消息请求，Consumer消费消息的请求，并且进行消息的持久化，以及HA策略和服务端过滤，就是集群中很重的工作都是交给了**Broker**进行处理。

**Broker**模块是通过BrokerStartup进行启动的，会实例化BrokerController，并且调用其初始化方法。

他的**初始化流程很冗长**，会根据配置创建很多线程池主要用来**发送消息**、**拉取消息**、**查询消息**、**客户端管理**和**消费者管理**，也有很多**定时任务**，同时也注册了很多**请求处理器**，用来发送拉取消息查询消息的。

#### Consumer

![image-20200518121909007](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518121909007.png)

消费端会通过**RebalanceService**线程，10秒钟做一次基于**Topic**下的所有队列负载。

## RocketMQ-面试题

### 优缺点

RocketMQ优点：

- 单机吞吐量：十万级
- 可用性：非常高，分布式架构
- 消息可靠性：经过参数优化配置，消息可以做到0丢失
- 功能支持：MQ功能较为完善，还是分布式的，扩展性好
- 支持10亿级别的消息堆积，不会因为堆积导致性能下降
- 源码是java，我们可以自己阅读源码，定制自己公司的MQ，可以掌控
- 天生为金融互联网领域而生，对于可靠性要求很高的场景，尤其是电商里面的订单扣款，以及业务削峰，在大量交易涌入时，后端可能无法及时处理的情况
- **RocketMQ**在稳定性上可能更值得信赖，这些业务场景在阿里双11已经经历了多次考验，如果你的业务有上述并发场景，建议可以选择**RocketMQ**

RocketMQ缺点：

- 支持的客户端语言不多，目前是java及c++，其中c++不成熟
- 社区活跃度不是特别活跃那种
- 没有在 mq 核心中去实现**JMS**等接口，有些系统要迁移需要修改大量代码

### 消息去重

去重原则：使用业务端逻辑保持幂等性

**幂等性**：就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用，数据库的结果都是唯一的，不可变的。

只要保持幂等性，不管来多少条重复消息，最后处理的结果都一样，需要业务端来实现。

**去重策略**：保证每条消息都有唯一编号(**比如唯一流水号)**，且保证消息处理成功与去重表的日志同时出现。

建立一个消息表，拿到这个消息做数据库的insert操作。给这个消息做一个唯一主键（primary key）或者唯一约束，那么就算出现重复消费的情况，就会导致主键冲突，那么就不再处理这条消息。

弱校验场景下可以使用Redis缓存一定时间内消费的消息标识，通过Redis做幂等性校验

### 消息重复

消息领域有一个对消息投递的QoS定义，分为：

- 最多一次（At most once）
- 至少一次（At least once）
- 仅一次（ Exactly once）

> QoS：Quality of Service，服务质量

几乎所有的MQ产品都声称自己做到了**At least once**。

既然是至少一次，那避免不了消息重复，尤其是在分布式网络环境下。

比如：网络原因闪断，ACK返回失败等等故障，确认信息没有传送到消息队列，导致消息队列不知道自己已经消费过该消息了，再次将该消息分发给其他的消费者。

不同的消息队列发送的确认信息形式不同，例如**RabbitMQ**是发送一个ACK确认消息，**RocketMQ**是返回一个CONSUME_SUCCESS成功标志，**Kafka**实际上有个offset的概念。

**RocketMQ**没有内置消息去重的解决方案，最新版本是否支持还需确认。

### 消息的可用性

当我们选择好了集群模式之后，那么我们需要关心的就是怎么去存储和复制这个数据，**RocketMQ**对消息的刷盘提供了同步和异步的策略来满足我们的，当我们选择同步刷盘之后，如果刷盘超时会给返回FLUSH_DISK_TIMEOUT，如果是异步刷盘不会返回刷盘相关信息，选择同步刷盘可以尽最大程度满足我们的消息不会丢失。

除了存储有选择之后，我们的主从同步提供了同步和异步两种模式来进行复制，当然选择同步可以提升可用性，但是消息的发送RT时间会下降10%左右。

**RocketMQ**采用的是混合型的存储结构，即为**Broker**单个实例下所有的队列共用一个日志数据文件（即为CommitLog）来存储。

而**Kafka**采用的是独立型的存储结构，每个队列一个文件。

**RocketMQ**采用混合型存储结构的缺点在于，会存在较多的随机读操作，因此读的效率偏低。同时消费消息需要依赖**ConsumeQueue**，构建该逻辑消费队列需要一定开销。

### RocketMQ 刷盘实现

**Broker** 在消息的存取时直接操作的是内存（内存映射文件），这可以提供系统的吞吐量，但是无法避免机器掉电时数据丢失，所以需要持久化到磁盘中。

刷盘的最终实现都是使用**NIO**中的 **MappedByteBuffer.force()** 将映射区的数据写入到磁盘，如果是同步刷盘的话，在**Broker**把消息写到**CommitLog**映射区后，就会等待写入完成。

异步而言，只是唤醒对应的线程，不保证执行的时机，流程如图所示。

![image-20200518173752653](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518173752653.png)

### 顺序消息：

我简单的说一下我们使用的**RocketMQ**里面的一个简单实现吧。

生产者消费者一般需要保证顺序消息的话，可能就是一个业务场景下的，比如订单的创建、支付、发货、收货。

那这些东西是不是一个订单号呢？一个订单的肯定是一个订单号的说，那简单了呀。

**一个topic下有多个队列**，为了保证发送有序，**RocketMQ**提供了**MessageQueueSelector**队列选择机制，他有三种实现:

![image-20200518193320709](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518193320709.png)

我们可使用**Hash取模法**，让同一个订单发送到同一个队列中，再使用同步发送，只有同个订单的创建消息发送成功，再发送支付消息。这样，我们保证了发送有序。

**RocketMQ**的topic内的队列机制,可以保证存储满足**FIFO**（First Input First Output 简单说就是指先进先出）,剩下的只需要消费者顺序消费即可。

**RocketMQ**仅保证顺序发送，**顺序消费由消费者业务保证！！**消费者要使用**ConsumeOrderlyContext**（顺序消费，不使用线程池），而不能使用`ConsumeConcurrentlyContext`

### 分布式事务：

#### Half Message(半消息)

**是指暂不能被Consumer消费的消息**。Producer 已经把消息成功发送到了 Broker 端，但此消息被标记为`暂不能投递`状态，处于该种状态下的消息称为半消息。需要 Producer

对消息的`二次确认`后，Consumer才能去消费它。

#### 消息回查

由于网络闪段，生产者应用重启等原因。导致 **Producer** 端一直没有对 **Half Message(半消息)** 进行 **二次确认**。这是**Brock**服务器会定时扫描`长期处于半消息的消息`，会

主动询问 **Producer**端 该消息的最终状态(**Commit或者Rollback**),该消息即为 **消息回查**。

![image-20200518193534671](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518193534671.png)

1. A服务先发送个Half Message给Broker端，消息中携带 B服务 即将要+100元的信息。
2. 当A服务知道Half Message发送成功后，那么开始第3步执行本地事务。
3. 执行本地事务(会有三种情况1、执行成功。2、执行失败。3、网络等原因导致没有响应)
4. 如果本地事务成功，那么Product向Broker服务器发送Commit,这样B服务就可以消费该message。
5. 如果本地事务失败，那么Product向Broker服务器发送Rollback,那么就会直接删除上面这条半消息。
6. 如果因为网络等原因迟迟没有返回失败还是成功，那么会执行RocketMQ的回调接口,来进行事务的回查。

> - 默认等待6S后第一次回查，后续间隔60S查询一次，共总15次
> - Half Message对应的是一个**独立的topic**，只有commit后消费者才可以接收的到

### 消息过滤

- **Broker**端消息过滤：在**Broker**中，按照**Consumer**的要求做过滤，优点是减少了对于**Consumer**无用消息的网络传输。缺点是增加了Broker的负担，实现相对复杂。
- **Consumer**端消息过滤：这种过滤方式可由应用完全自定义实现，但是缺点是很多无用的消息要传输到**Consumer**端。

### Broker的Buffer问题

Broker的**Buffer**通常指的是Broker中一个队列的内存Buffer大小，这类**Buffer**通常大小有限。

另外，RocketMQ没有内存**Buffer**概念，RocketMQ的队列都是持久化磁盘，数据定期清除。

RocketMQ同其他MQ有非常显著的区别，RocketMQ的内存**Buffer**抽象成一个无限长度的队列，不管有多少数据进来都能装得下，这个无限是有前提的，Broker会定期删除过期的数据。

例如Broker只保存3天的消息，那么这个**Buffer**虽然长度无限，但是3天前的数据会被从队尾删除。

### 回溯消费

回溯消费是指Consumer已经消费成功的消息，由于业务上的需求需要重新消费，要支持此功能，Broker在向Consumer投递成功消息后，消息仍然需要保留。并且重新消费一般是按照时间维度。

例如由于Consumer系统故障，恢复后需要重新消费1小时前的数据，那么Broker要提供一种机制，可以按照时间维度来回退消费进度。

**RocketMQ**支持按照时间回溯消费，时间维度精确到毫秒，可以向前回溯，也可以向后回溯。

### 消息堆积

消息中间件的主要功能是异步解耦，还有个重要功能是挡住前端的数据洪峰，保证后端系统的稳定性，这就要求消息中间件具有一定的消息堆积能力，消息堆积分以下两种情况：

- 消息堆积在内存**Buffer**，一旦超过内存**Buffer**，可以根据一定的丢弃策略来丢弃消息，如CORBA Notification规范中描述。适合能容忍丢弃消息的业务，这种情况消息的堆积能力主要在于内存**Buffer**大小，而且消息堆积后，性能下降不会太大，因为内存中数据多少对于对外提供的访问能力影响有限。
- 消息堆积到持久化存储系统中，例如DB，KV存储，文件记录形式。 当消息不能在内存Cache命中时，要不可避免的访问磁盘，会产生大量读IO，读IO的吞吐量直接决定了消息堆积后的访问能力。
- 评估消息堆积能力主要有以下四点：
  - 消息能堆积多少条，多少字节？即消息的堆积容量。
  - 消息堆积后，发消息的吞吐量大小，是否会受堆积影响？
  - 消息堆积后，正常消费的Consumer是否会受影响？
  - 消息堆积后，访问堆积在磁盘的消息时，吞吐量有多大？

### 定时消息

定时消息是指消息发到**Broker**后，不能立刻被**Consumer**消费，要到特定的时间点或者等待特定的时间后才能被消费。

如果要支持任意的时间精度，在**Broker**层面，必须要做消息排序，如果再涉及到持久化，那么消息排序要不可避免的产生巨大性能开销。

**RocketMQ**支持定时消息，但是不支持任意时间精度，支持特定的level，例如定时5s，10s，1m等。

**实现原理**：所有的延迟消息由producer发出之后，都会存放到同一个topic（**SCHEDULE_TOPIC_XXXX**）下，不同的延迟级别会对应不同的队列序号，当延迟时间到之后，由定时线程读取转换为普通的消息存的真实指定的topic下，此时对于consumer端此消息才可见，从而被consumer消费。

RocketMQ 不支持任意时间精度，仅支持特定的 level。可以在服务器端（rocketmq-broker端）配置。

```properties
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

### 消息重试

RocketMQ的消息重试及时分为两种，一种是Producer端重试，一种是Consume端重试。

- **Producer端重试 ：**

  消息发没发成功，默认情况下是3次重试。

- **2、Consumer端重试：**

  - **exception的情况**，一般重复16次 10s、30s、1mins、2mins、3mins等。注意reconsumeTimes这个参数；

  - **超时情况**，这种情况MQ会无限制的发送给消费端。这种情况就是Consumer端没有返回*ConsumeConcurrentlyStatus.CONSUME_SUCCESS，也没有返回*ConsumeConcurrentlyStatus.RECONSUME_LATER。Consumner超时的情况我们还分为一个Producer和一个Consumer的场景和一个Producer和多个Consumer（Consumer集群）的场景。

```pro
messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
```

```java
// Producer端消息重试
public static void main(String[] args) {
    DefaultMQProducer producer = new DefaultMQProducer("push_consumer");
    producer.setNamesrvAddr("192.168.31.176:9876;192.168.31.165:9876");
    try {
        // 设置实例名称
        producer.setInstanceName("quick_start_producer");
        // 设置重试次数,默认2
        producer.setRetryTimesWhenSendFailed(3);
        //设置发送超时时间，默认是3000
        producer.setSendMsgTimeout(6000);
        // 开启生产者
        producer.start();
        // 创建一条消息
        Message msg = new Message("PushTopic_tt1", "TagB", "OrderID0034", "uniform_just_for_test".getBytes());
        SendResult send = producer.send(msg);
        System.out.println("id:--->" + send.getMsgId() + ",result:--->" + send.getSendStatus());
    } catch (MQClientException e) {
        e.printStackTrace();
    } catch (RemotingException e) {
        e.printStackTrace();
    } catch (MQBrokerException e) {
        e.printStackTrace();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } 
    producer.shutdown();
}
```

```java

/**
 * 消费端重试的情况 :异常情况
 */

public static void main(String[] args) throws MQClientException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("push_consumer");
    consumer.setNamesrvAddr("192.168.31.176:9876;192.168.31.165:9876");
    // 批量消费,每次拉取10条
    consumer.setConsumeMessageBatchMaxSize(10);// ①

    // 程序第一次启动从消息队列头取数据
    // 如果非第一次启动，那么按照上次消费的位置继续消费
    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
    // 订阅PushTopic下Tag为push的消息
    consumer.subscribe("PushTopic_tt1", "*");
    consumer.registerMessageListener(new MessageListenerConcurrently() {
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {

            MessageExt msg = msgs.get(0);
            try {
                String topic = msg.getTopic();
                String msgBody = new String(msg.getBody(), "utf-8");
                String tags = msg.getTags();
                System.out.println("收到消息:topic:" + topic + ",tags:" + tags + ",msg:" + msg + "msgBody:" + msgBody);
                if ("message4".equals(msgBody)) {
                    System.out.println("====失败消息开始=====");
                    System.out.println("msg:" + msg);
                    System.out.println("====失败消息结束=====");
                    // 发生异常
                    int i = 1 / 0;
                    System.out.println(i);
                }
            } catch (Exception e) {
                e.printStackTrace();
                // 如果重试了三次就返回成功
                if (msg.getReconsumeTimes() == 3) {
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                }
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }

            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    });
    consumer.start();
    System.out.println("Consumer Started.");
    // consumer.suspend();

}

```



### 高可用

与其他传统的主从架构一样，Master负责接收数据之后，再将数据同步给Slave，Master和Slave都会向NameServer注册路由信息，同时每隔30秒发送一次心跳。RocketMQ主从架构数据同步采用的是**Pull**模式，Slave不停的发送请求到Master上去拉取消息

与其他传统的主从架构不一样的是，RocketMQ的主从架构**并不是纯粹的读写分离**。写的话还是依靠Master，而读的话，既可能是从Master读，也可能是从Slave读，这取决于Master。

读消息的时候，第一次是从Master上去读的，之后是去Master还是Slave上读消息，是由Master根据自身的负载情况和Slave的数据同步情况来向消费者建议。如果Master的写负载已经很高的话，那么就会建议消费者下次拉取消息的话就去Slave上拉取；如果Slave数据同步较慢，还没有完全同步Master的消息的话，那么下一次还是会来Master上拉取消息。

当Master节点宕机之后，一般来说会对Slave节点进行选举然后选出新的Master继续提供服务。但是在RocketMQ 4.5版本之前，还不支持自动故障切换，需要手工调整才能完成。在RocketMQ 4.5版本之后，引入了**Dledger**技术，是基于**Raft**协议实现的。DLedger引入之后，可以让一个Master对应多个Slave，也就是存在多个副本，一旦Master宕机了，多个Slave之间就会通过DLedger技术和Raft协议算法进行选举，选出来的Slave节点就作为新的Master继续提供服务。

DLedger是基于Raft协议来进行选主的，大致流程为：
当Leader宕机，我们假设此时有3个Follower，分别为Broker1、Broker2、Broker3，此时他们都没有接收到其他人的投票，所以此时都会投给自己，这一轮无法选出新的Leader，接下来所有的Follower随机休眠一段时间，比如依次休眠1s、2s、3s，优先苏醒的Broker1还会投票给自己，然后将投票信息发送给其他的节点，此时Broker2苏醒，接收到了Broker1的投票信息，就会尊重他的决定，也给Broker1投票，接着Broker3苏醒之后，也是一样的给Broker1投票，此时Broker1就获得了大多数节点的支持，当有一个节点获得了（N / 2） + 1个，也就是大多数节点的支持之后，就能成功当选。

DLedger基于Raft协议进行多副本同步，大致流程为：
DLedger多副本数据同步，Leader Broker收到一条消息，通过DledgerServer组件发送一个uncommited消息给Follow Broker的DledgerServer，Follower接收到消息之后发送一个ack确认消息给Leader，多数Follower都确认之后，Leader再发送一个commited消息到Follower上，Follower将接收到的消息置为commited，此时就完成了数据同步。

![image-20200518214515302](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518214515302.png)

### offset

在集群模式下，offset存储在broker端，因为消费组需要共享消费进度。在广播模式下，offset存储在consumer端，不同的consumer相互隔离。

### rebalance

 Rebalance(再均衡)机制指的是：将一个Topic下的**多个队列(或称之为分区)**，在同一个消费者组(consumer group)下的**多个消费者实例**(consumer instance)之间进行重新分配。 

从本质上来说，触发Rebalance的根本因素无非是两个：

- 1 ) **订阅Topic的队列数量变化  **

  典型场景：broker宕机broker升级等运维操作队列扩容/缩容

- **2）消费者组信息变化。**

  典型场景：日常发布过程中的停止与启动消费者异常宕机网络异常导致消费者与Broker断开连接主动进行消费者数量扩容/缩容Topic订阅信息发生变化

如果同一个消费组的不同消费者**订阅不同的Topic**，将导致消费异常：只能消费一半消息，心跳包通知broker订阅信息，rebalance不断覆盖订阅信息。

## RocketMQ-存储架构

![image-20200518221800882](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518221800882.png)

上图即为RocketMQ的消息存储整体架构，RocketMQ采用的是混合型的存储结构，即为Broker单个实例下所有的队列共用一个日志数据文件（即为CommitLog，1G）来存储。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200525230950.png)

Consume Queue相当于kafka中的partition，是一个逻辑队列，存储了这个Queue在CommiLog中的   起始offset，log大小和MessageTag的hashCode。

每次读取消息队列先读取consumerQueue,然后再通过consumerQueue去commitLog中拿到消息主体。

- ConsumerQueue

  `consumerqueue`的设计以`topic`作为逻辑分区，每个`topic`下分多个消息队列进行，具体多少消息队列存储参照`broker`的配置参数，队列名称以数组0开始，比如配置0,1,2,3 四个消息队列。

  `commitlog`存储着`broker`上所有的消息，`consumerqueue`的设计之初就是为了快速定位到对应的消费者可以消费的消息，当然`RocketMQ`也提供了`indexfile`，俗称索引文件，主要是解决通过key快速定位消息的方式。

  ![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200525230131.png)

  在`consumerqueue`的结构设计，在`consumequeue`的条目设计是固定的，并且它整好对应一条消息。`consumerqueue`单个文件默认是30w个条目，单个文件长度30w * 20字节。

### Kafka存储架构

rocketMQ的设计理念很大程度借鉴了kafka，所以有必要介绍下kafka的存储结构设计:

![image-20200518222624151](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518222624151.png)

存储特点： 和RocketMQ类似，每个Topic有多个partition(queue),kafka的每个partition都是一个独立的物理文件，消息直接从里面读写。

根据之前阿里中间件团队的测试，一旦kafka中Topic的partitoin数量过多，队列文件会过多，会给磁盘的IO读写造成很大的压力，造成tps迅速下降。

所以RocketMQ进行了上述这样设计，**consumerQueue中只存储很少的数据**，消息主体都是通过CommitLog来进行读写。

ps：上一行加粗理解：consumerQueue存储少量数据，即使数量很多，但是数据量不大，文件可以控制得非常小，**绝大部分的访问还是Page Cache的访问，而不是磁盘访问**。正式部署也可以将CommitLog和consumerQueue放在不同的物理SSD，避免多类文件进行IO竞争。

### RocketMQ与Kafka比较

- 1.为什么RocketMQ使用mmap+write的方式，而kafka采用sendfile的方式。并且sendfile可以更好的减少copy次数（2次），而mmap最少需要3次？

  因为kafka每个partition(queue)对应一个单独的文件，可以基于sendfile直接把文件指定偏移量传给客户端（可以批量传输）。而RocketMQ的数据统一在commitlog中，需要根据ConsumerQueue先查出偏移量，无法批量传输（每个queue对应的offset可能是不连续的）。

- 2.为什么RocketMQ中的commitlog固定为1G大小？

  因为mmap方式对文件大小有限制，一次只能映射1.5~2G 的文件至用户态的虚拟内存。
  
- 3.为什么kafka在partition多了以后性能会很差，而RocketMQ可以支撑上万个Topic?

  因为kafka中Topic的partitoin数量过多，队列文件会过多，会导致随机读很严重，无法有效使用pagecache。而RocketMQ对应的是一个CommitLog文件，虽然内部也是随机读，但是整体来看的话是顺序读，依然可以很好的使用pagecache。

### 存储架构优缺点

- 优点：

  队列轻量化，单个队列数据量非常少。对磁盘的访问串行化，避免磁盘竟争，不会因为队列增加导致IOWAIT增高。

- 缺点：

  写虽然完全是顺序写，但是读却变成了完全的随机读。

  读一条消息，会先读ConsumeQueue，再读CommitLog，增加了开销。

  要保证CommitLog与ConsumeQueue完全的一致，增加了编程的复杂度。

**缺点克服：**

随机读，尽可能让读命中page cache，减少IO读操作，所以内存越大越好。如果系统中堆积的消息过多，读数据要访问磁盘不会由于随机读导致系统性能急剧下降。 

访问page cache 时，即使只访问1k的消息，系统也会提前预读出更多数据，在下次读时，就可能命中内存。 

随机访问Commit Log磁盘数据，系统IO调度算法设置为**NOOP**方式，会在一定程度上将完全的随机读变成**顺序跳跃方式**，而顺序跳跃方式读较完全的随机读性能会高5倍以上。 

>  另外4k的消息在完全随机访问情况下，仍然可以达到8K次每秒以上的读性能。 

由于Consume Queue存储数据量极少，而且是顺序读，在PAGECACHE预读作用下，Consume Queue的读性能几乎与内存一致，即使堆积情况下。所以可认为Consume Queue完全不会阻碍读性能。 

Commit Log中存储了所有的元信息，包含消息体，类似于Mysql、Oracle的redolog，所以只要有Commit Log在，Consume Queue即使数据丢失，仍然可以恢复出来。

### 底层存储实现

#### MappedByteBuffer

RocketMQ中的文件读写主要就是通过MappedByteBuffer进行操作，来进行文件映射。利用了nio中的FileChannel模型，可以直接将物理文件映射到缓冲区，提高读写速度。

这种**Mmap**的方式减少了传统IO将磁盘文件数据在**内核空间**的缓冲区和**用户空间**的缓冲区之间来回进行拷贝的性能开销。

> MappedByteBuffer.force() 将映射区的数据写入到磁盘

**这里需要注意的是，采用MappedByteBuffer这种内存映射的方式有几个限制，其中之一是一次只能映射1.5~2G 的文件至用户态的虚拟内存，这也是为何RocketMQ默认设置单个CommitLog日志数据文件为1G的原因了。**

#### page cache

刚刚提到的缓冲区，也就是page cache。

通俗的说：pageCache是系统读写磁盘时为了提高性能将部分文件缓存到内存中，下面是详细解释：

page cache:这里所提及到的page cache，是linux中vfs虚拟文件系统层的cache层，一般pageCache默认是4K大小，它被操作系统的内存管理模块所管理，文件被映射到内存，一般都是被**mmap()**函数映射上去的。

mmap()函数会返回一个指针，指向逻辑地址空间中的逻辑地址，逻辑地址通过MMU映射到page cache上。

pageCache缺点：

内核把可用的内存分配给Page Cache后，free的内存相对就会变少，如果程序有新的内存分配需求或者缺页中断，恰好free的内存不够，内核还需要花费一点时间将热度低的Page Cache的内存回收掉，对性能非常苛刻的系统会**产生毛刺**。

### 消息发送、消费逻辑

#### 发送逻辑

发送时，**Producer不直接与Consume Queue打交道**。RMQ所有的消息都会存放在Commit Log中，为了使消息存储不发生混乱，**对Commit Log进行写之前就会上锁**。

> putMessageLock有两种实现：1->基于ReentrantLock实现的可重入锁；2->基于Atomic原子类中的CAS操作实现的乐观锁（自旋锁）。

消息持久被锁串行化后，对Commit Log就是顺序写，也就是常说的Append操作。配合上Page Cache，RMQ在写Commit Log时效率会非常高。

Broker端的后台服务线程—**ReputMessageService**不停地分发请求并**异步构建ConsumeQueue（逻辑消费队列）和IndexFile（索引文件）数据，不停的轮询**，将当前的consumeQueue中的offSet和commitLog中的offSet进行对比，将多出来的offSet进行解析，然后put到consumeQueue中的MapedFile中。

ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。而IndexFile（索引文件）则只是为了消息查询提供了一种通过key或时间区间来查询消息的方法（ps：这种通过IndexFile来查找消息的方法不影响发送与消费消息的主流程）。

#### 消费逻辑

消费时，**Consumer不直接与Commit Log打交道，而是从Consume Queue中去拉取数据**。拉取的顺序从旧到新，在文件表示每一个Consume Queue都是顺序读，充分利用了Page Cache。光拉取Consume Queue是没有数据的，里面只有一个对Commit Log的引用，所以再次拉取Commit Log。

但整个RMQ只有一个Commit Log，虽然是随机读，但**整体还是有序地读**，只要那整块区域还在Page Cache的范围内，还是可以充分利用Page Cache。(**dstat**命令)

对于CommitLog消息存储的日志数据文件来说，读取消息内容时候会产生较多的随机访问读取，严重影响性能。如果选择合适的系统IO调度算法，比如设置调度算法为**Noop**（此时块存储采用SSD的话），随机读的性能也会有所提升。

### 文件存储模型图

![image-20200518224159502](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200518224159502.png)

### （1）RocketMQ业务处理器层

Broker端对消息进行读取和写入的业务逻辑入口，这一层主要包含了业务逻辑相关处理操作（根据解析RemotingCommand中的RequestCode来区分具体的业务操作类型，进而执行不同的业务处理流程），比如前置的检查和校验步骤、构造MessageExtBrokerInner对象、decode反序列化、构造Response返回对象等。

### （2）RocketMQ数据存储组件层

该层主要是RocketMQ的存储核心类—DefaultMessageStore，其为RocketMQ消息数据文件的访问入口，通过该类的“putMessage()”和“getMessage()”方法完成对CommitLog消息存储的日志数据文件进行读写操作（具体的读写访问操作还是依赖下一层中CommitLog对象模型提供的方法）；另外，在该组件初始化时候，还会启动很多存储相关的后台服务线程，包括AllocateMappedFileService（MappedFile预分配服务线程）、ReputMessageService（回放存储消息服务线程）、HAService（Broker主从同步高可用服务线程）、StoreStatsService（消息存储统计服务线程）、IndexService（索引文件服务线程）等。

### （3）RocketMQ存储逻辑对象层

该层主要包含了RocketMQ数据文件存储直接相关的三个模型类IndexFile、ConsumerQueue和CommitLog。IndexFile为索引数据文件提供访问服务，ConsumerQueue为逻辑消息队列提供访问服务，CommitLog则为消息存储的日志数据文件提供访问服务。这三个模型类也是构成了RocketMQ存储层的整体结构（对于这三个模型类的深入分析将放在后续篇幅中）。

### （4）封装的文件内存映射层

RocketMQ主要采用JDK NIO中的MappedByteBuffer和FileChannel两种方式完成数据文件的读写。其中，采用MappedByteBuffer这种内存映射磁盘文件的方式完成对大文件的读写，在RocketMQ中将该类封装成MappedFile类。这里限制的问题在上面已经讲过；对于每类大文件

（IndexFile/ConsumerQueue/CommitLog），在存储时分隔成多个固定大小的文件（单个IndexFile文件大小约为400M、单个ConsumerQueue文件大小约5.72M、单个CommitLog文件大小为1G），其中每个分隔文件的文件名为前面所有文件的字节大小数+1，即为文件的起始偏移量，从而实现了整个大文件的串联。这里，每一种类的单个文件均由MappedFile类提供读写操作服务（其中，MappedFile类提供了顺序写/随机读、内存数据刷盘、内存清理等和文件相关的服务）。

### （5）磁盘存储层

主要指的是部署RocketMQ服务器所用的磁盘。这里，需要考虑不同磁盘类型（如SSD或者普通的HDD）特性以及磁盘的性能参数（如IOPS、吞吐量和访问时延等指标）对顺序写/随机读操作带来的影响。

### offset

首先来明确一下 Offset 的含义， RocketMQ 中， 一 种类型的消息会放到 一 个 Topic 里，为了能够并行， 一般一个 Topic 会有多个 Message Queue (也可以 设置成一个)， **Offset是指某个 Topic下的一条消息在某个 Message Queue里的 位置**，通过 Offset的值可以定位到这条消息，或者指示 Consumer从这条消息 开始向后继续处理 。

Offset主要分为本地文件类型和 Broker代存 的类型两种 。

![image-20200601174541675](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200601174550.png)

Rocketmq集群有两种消费模式

- 默认是 CLUSTERING 模 式，也就是同一个 Consumer group 里的多个消费者每人消费一部分，各自收到 的消息内容不一样 。 这种情况下，由 Broker 端存储和控制 Offset 的值，使用 **RemoteBrokerOffsetStore** 结构 。
- BROADCASTING模式下，每个 Consumer 都收到这个 Topic 的全部消息，各个 Consumer 间相互没有干扰， RocketMQ 使 用 LocalfileOffsetStore，把 Offset存到本地 。

使用DefaultMQPushConsumer的时候，我们不用关心OffsetStore的 事，使用PullConsumer，我们就要自己处理 OffsetStore。

DefaultMQPushConsumer类里有个函数用来设置从哪儿开始消费 消 息:比如 setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_ FIRST_OFFSET)，这个语句设置从最小的 Offset开始读取。

如果从队列开始到 感兴趣的消息之间有很大的范围，用 CONSUME_FROM_FIRST_OFFSET参数 就不合适了，可以设置从某个时间开始消 费消息 ， 比如 Consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_TIMESTAMP), Consumer. setConsumeTimestamp("20131223171201”)， 时间戳格式是精确到秒的 。

Consumer. setConsumeTimestamp("20131223171201”)， 

```java
/**
* Consumer从哪里开始消费<br>
*
* @author shijia.wxr<vintage.wang@gmail.com>
*/public enum ConsumeFromWhere {    /**
    * 一个新的订阅组第一次启动从队列的最后位置开始消费<br>
    * 后续再启动接着上次消费的进度开始消费
    */
   CONSUME_FROM_LAST_OFFSET,    @Deprecated
   CONSUME_FROM_LAST_OFFSET_AND_FROM_MIN_WHEN_BOOT_FIRST,    @Deprecated
   CONSUME_FROM_MIN_OFFSET,    @Deprecated
   CONSUME_FROM_MAX_OFFSET,    /**
    * 一个新的订阅组第一次启动从队列的最前位置开始消费<br>
    * 后续再启动接着上次消费的进度开始消费
    */
   CONSUME_FROM_FIRST_OFFSET,    /**
    * 一个新的订阅组第一次启动从指定时间点开始消费<br>
    * 后续再启动接着上次消费的进度开始消费<br>
    * 时间点设置参见DefaultMQPushConsumer.consumeTimestamp参数
    */
   CONSUME_FROM_TIMESTAMP,
}
```

注意设置读取位置不是每次都有效，它的优先级默认在 Offset Store后面 ， 比如 在 DefaultMQPushConsumer 的 BROADCASTING 方式 下 ，默 认 是 从 Broker 里读取某个 Topic 对 应 ConsumerGroup 的 Offset， **当读取不到 Offset 的时候， ConsumeFromWhere 的设置才生效** 。 大部分情况下这个设置在 Consumer Group初次启动时有效。 如果 Consumer正常运行后被停止， 然后再启动， 会接着上次的 Offset开始消费， ConsumeFromWhere 的设置元效。

## 主从同步机制（HA）

主从同步同步的是消息相当于给数据做**”备份“**，主节点服务器Broker宕机后，消费者可以从从节点的服务消费消息，可以保证业务的正常运行。 **主从同步的原理图**

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200706000938.jpg)

增加slave从节点的优点：

- **数据备份**：保证了两/多台机器上的数据冗余，特别是在主从同步复制的情况下，一定程度上保证了Master出现不可恢复的故障以后，数据不丢失。 **高可用性**：即使Master掉线， Consumer会自动重连到对应的Slave机器，不会出现消费停滞的情况。 
- **提高性能**：主要表现为可分担Master读的压力，当从Master拉取消息，拉取消息的最大物理偏移与本地存储的最大物理偏移的差值超过一定值，会转向Slave(默认brokerId=1)进行读取，**减轻了Master压力**。 

- **消费实时**：master宕机后消费者可以从slave上消费保证消息的实时性，但是slave不能接收producer发送的消息，slave只能同步master数据（RocketMQ4.5版本之前），**4.5版本开始增加多副本机制**，根据RAFT算法，master宕机会自动选择其中一个副本节点作为master保证消息可以正常的生产消费。

主从数据同步有两种方式同步复制、异步复制

| 复制方式 | 优点                                                         | 缺点                                                         | 适应场景                 |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------ |
| 同步复制 | slave保证了与master一致的数据副本，如果master宕机，数据依然在slave中找到其数据和master的数据一致 | 由于需要slave确认效率上会有一定的损失                        | 数据可靠性要求很高的场景 |
| 异步复制 | 无需等待slave确认消息是否存储成功效率上要高于同步复制        | 如果master宕机，由于数据同步有延迟导致slave和master存在一定程度的数据不一致问题 | 数据可靠性要求一般的场景 |

从节点同步commitlog时同样需要同步相关的**配置信息**，**主题列表信息**、**消费组信息**、**消费进度信息**等元数据信息。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200706001105.jpg)

commitlog文件复制流程图

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200706001311.jpg)

1、slave的broker向master连接心跳包，报告master当前slave的数据偏移量，等待master发送最新的消息。

2、master启动时开启一个监听等待slave发送心跳包，接收到slave发送的请求封装成HAConnection对象中两个属性，WriteSocketService处理slave发送的心跳包，ReadSocketService发送slave的数据请求。