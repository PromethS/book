# 简介

MongoDB 是一个基于**分布式文件存储的数据库**，由 C++ 编写，旨在为 WEB 应用提供可扩展、高性能的数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是**非关系数据库中功能最丰富**、**最像关系数据库**的。在高负载的情况下，添加更多的节点，可以保证服务器性能。

## 特点

- MongoDB 是一个**面向文档存储**的数据库，操作简单。

- 可以在MongoDB记录中设置任何属性的索引 (如：FirstName="Sameer",Age="8")来实现更快的排序。

- 可以通过本地或者网络创建数据镜像，从而使MongoDB有更强的扩展性。

- 如果负载的增加（需要更多的存储空间和更强的处理能力） ，它可以分布在计算机网络中的其他节点上，这就是所谓的**分片**。

- MongoDB支持**丰富的查询表达式**。查询指令使用JSON形式的标记，可轻易查询文档中内嵌的对象及数组。

- MongoDB 可以使用update()命令替换完成的文档（数据）或者一些指定的数据字段 。

- MongoDB中的**Map/Reduce**主要是用来对数据进行**批量处理和聚合操作**。

- Map和Reduce。Map函数调用emit(key,value)遍历集合中所有的记录，将key与value传给Reduce函数进行处理。

- Map函数和Reduce函数是使用Javascript编写的，并可以通过db.runCommand或mapreduce命令来执行MapReduce操作。

- **GridFS**是MongoDB中的一个内置功能，可以用于存放大量小文件。

  > MongoDb GridFS 是MongoDB的文件存储方案，主要用于存储和恢复那些超过16M（BSON文件限制）的文件(如：图片、音频等)，对大文件有着更好的性能。

- MongoDB允许在服务端执行脚本，可以用Javascript编写某个函数，直接在服务端执行，也可以把函数的定义存储在服务端，下次直接调用即可。

- MongoDB为多种编程语言提供了支持

# 一、副本集

## 1、基本概念

**一组副本集就是一组mongod实例掌管同一个数据集，实例可以在不同的机器上面**。实例中包含一个主导，接受客户端所有的写入操作，其他都是副本实例，从主服务器上获得数据并保持同步。

主服务器很重要，包含了所有的改变操作（写）的日志。但是副本服务器集群包含有所有的主服务器数据，因此当主服务器挂掉了，就会在副本服务器上重新选取一个成为主服务器。每个副本集还有一个仲裁者，**仲裁者不存储数据，只是负责通过心跳包来确认集群中集合的数量，并在主服务器选举的时候作为仲裁决定结果**。

常见MongoDB集群是一个主节点和两个副本节点。主节点负责处理客户端请求，其余的副本节点节点，负责复制主节点上的数据。**主节点记录在其上的所有操作oplog，副本节点定期轮询主节点获取这些操作，然后对自己的数据副本执行这些操作，从而保证从节点的数据与主节点一致**。

> 从节点的类型：
>
> - Secondary:标准从节点，可复制，可投票，可被选为主节点
> - Secondary-Only:不能成为主节点，只能作为从节点，防止一些性能不高的节点成为主节点。设置方法是将其priority = 0
> - Hidden:这类节点是不能够被客户端指定IP引用，也不能被设置为主节点，但是可以投票，一般用于备份数据。
> - Delayed：可以指定一个时间延迟从primary节点同步数据。主要用于备份数据，如果实时同步，误删除数据马上同步到从节点，恢复又恢复不了。

### 1）副本集的作用

MongoDB的Replica Set即副本集方式主要有两个目的，

- 一个是数据冗余做**故障恢复**使用，当发生硬件故障或者其它原因造成的宕机时，可以使用副本进行恢复。

- 另一个是做**读写分离**，读的请求分流到副本上，减轻主（Primary）服务器的读压力。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613014336.png)

### 2）oplog简介

  **oplog是local库下的一个固定集合**，Secondary就是通过查看Primary 的oplog这个集合来进行复制的。**每个节点都有oplog，记录这从主节点复制过来的信息**，这样每个成员都可以作为同步源给其他节点。

  副本集中数据同步的详细过程：**Primary节点写入数据，Secondary通过读取Primary的oplog得到复制信息，开始复制数据并且将复制信息写入到自己的oplog**。如果某个操作失败（只有当同步源的数据损坏或者数据与主节点不一致时才可能发生），则备份节点停止从当前数据源复制数据。如果某个备份节点由于某些原因挂掉了，当重新启动后，就会自动从oplog的最后一个操作开始同步，同步完成后，将信息写入自己的oplog，由于复制操作是先复制数据，复制完成后再写入oplog，有可能相同的操作会同步两份，不过MongoDB在设计之初就考虑到这个问题，将oplog的同一个操作执行多次，与执行一次的效果是一样的（**幂等性**）。

## 2、副本集的基本架构

  具有三个存储数据的成员的复制集有：一个主节点；两个副本节点组成，主节点宕机时，这两个副本节点都可以被选为主节点。
![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613014635.png)
  当主节点宕机后,两个副本节点都会进行竞选，其中一个变为主节点，当主节点恢复后，作为副本节点加入当前的副本集群即可。

## 3、副本集选主过程

### 1）三个存储数据的集群

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613015913.png)
  MongoDB副本集故障转移功能得益于它的选举机制。选举机制采用了**Bully**算法，可以很方便从分布式节点中选出主节点。一个分布式集群架构中一般都有一个所谓的**主节点**，可以有很多用途，比如**缓存机器节点元数据，作为集群的访问入口**等等。

### 2）arbiter节点

![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613020047.png)

  由于**arbiter节点没有复制数据**，因此这个架构中仅提供一个完整的数据副本。arbiter节点只需要更少的资源，代价是更有限的冗余和容错。当主节点宕机时，将会选择副本节点成为主节点，主节点修复后，将其加入到现有的副本集群中即可。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613020118.png)

### 3）Bully算法

  Bully算法是一个分布式系统中动态选择master节点的算法，**进程号最大的非失效的**节点将被选为master。**Bully算法是一种协调者（主节点）竞选算法，主要思想是集群的每个成员都可以声明它是主节点并通知其他节点，别的节点可以选择接受这个声称或是拒绝并进入主节点竞争**。被其他所有节点接受的节点才能成为主节点。节点按照一些属性来判断谁应该胜出。这个属性可以是一个静态ID，也可以是更新的度量像最近一次事务ID（最新的节点会胜出）。

### 4）MongoDB的选主过程

1. **得到每个节点的最后操作时间戳**。每个MongoDB都有**oplog**机制会记录本机的操作，方便和主服务器进行对比数据是否同步还可以用于错误恢复。
2. **如果集群中大部分节点宕机了**，**保留活着的节点都为secondary状态并停止（主节点降备）**，停止选举。
3. 如果集群中**选举出来的主节点或者所有从节点最后一次同步时间看起来很旧了，停止选举**等待人来操作。
4. 如果上面都没有问题就**选择最后操作时间戳最新（保证数据是最新的）的服务器节点作为主节点**。
5. 如果最后操作时间戳都一致，则选择**进程号最大**的节点。

【**选主的触发条件**】：

- **初始化一个副本集时**。
- **副本集和主节点断开连接**，可能是网络问题。
- **主节点挂掉**。

### 5）为什么副本集数量最好为奇数？

  **官方推荐副本集的成员数量为奇数，最多12个副本集节点，最多7个节点参与选举**。最多12个副本集节点是因为没必要一份数据复制那么多份，备份太多反而增加了网络负载和拖慢了集群性能；而最多7个节点参与选举是因为内部选举机制节点数量太多就会导致1分钟内还选不出主节点，凡事只要适当就好。这个 “12”、 “7”数字还好，通过他们官方经过性能测试定义出来可以理解。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613020821.png)

假设四个节点被分成两个IDC，每个IDC各两台机器，如下图。但这样就出现了个问题，如果两个IDC网络断掉，这在广域网上很容易出现的问题，在上面选举中提到**只要主节点和集群中大部分节点断开链接就会开始一轮新的选举操作，不过MongoDB副本集两边都只有两个节点，但是选举要求参与的节点数量必须大于一半，这样所有集群节点都没办法参与选举，只会处于只读状态**。但是如果是奇数节点就不会出现这个问题，假设3个节点，只要有2个节点活着就可以选举，5个中的3个，7个中的4个。

### 6）心跳和同步机制

【**心跳**】：

  **综上所述，整个集群需要保持一定的通信才能知道哪些节点活着哪些节点挂掉**。MongoDB节点会**向副本集中的其他节点每两秒就会发送一次pings包，如果其他节点在10秒钟之内没有返回就标示为不能访问**。每个节点内部都会维护一个**状态映射表**，表明当前每个节点是什么角色、日志时间戳等关键信息。如果是主节点，除了维护映射表外还需要检查自己能否和集群中内大部分节点通讯，如果不能则把自己降级为secondary只读节点。

【**同步**】：

  副本集同步分为初始化同步和keep复制。初始化同步指**全量从主节点同步数据**，如果主节点数据量比较大同步时间会比较长。而keep复制指**初始化同步过后，节点之间的实时同步一般是增量同步**。

# 二、MongoDB的路由、分片技术

## 1、MongoDB的Sharding架构

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021003.png)

从图中可以看到有四个组件：mongos、config server、shard、replica set。

【**mongos**】：

  **数据库集群请求的入口，所有的请求都通过mongos进行协调**，不需要在应用程序添加一个路由选择器，**mongos自己就是一个请求分发中心，它负责把对应的数据请求转发到对应的shard服务器上**。在生产环境通常有**多个mongos作为请求的入口**，防止其中一个挂掉所有的MongoDB请求都没有办法操作。

【**config server**】：

  **顾名思义为配置服务器，存储所有数据库元信息（路由、分片）的配置**。mongos本身没有物理存储分片服务器和数据路由信息，只是缓存在内存里，配置服务器则实际存储这些数据。mongos第一次启动或者关掉重启就会从 config server 加载配置信息，以后如果配置服务器信息变化会通知到所有的 mongos 更新自己的状态，这样 mongos 就能继续准确路由。在生产环境通常有**多个 config server 配置服务器**，因为它存储了分片路由的元数据，这个可不能丢失！就算挂掉其中一台，只要还有存货，MongoDB集群就不会挂掉。

【**shard**】：

  这就是分片，如下图：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021025.png)

一台机器的一个数据表 Collection1 存储了 1T 数据，压力太大了！在分给4个机器后，每个机器都是256G，则分摊了集中在一台机器的压力。实际运行的数据库还有硬盘的读写、网络的IO、CPU和内存的瓶颈。在MongoDB集群只要设置好了分片规则，通过mongos操作数据库就能自动把对应的数据操作请求转发到对应的分片机器上。在生产环境中分片的片键需要设置好，这个影响到了怎么**把数据均匀分到多个分片机器**上，不要出现其中一台机器分了1T，其他机器没有分到的情况，这样还不如不分片！

## 2、分片的优势

- **对集群进行抽象，让集群“不可见”**：mongos就是掌握统一路口的路由器，其会将客户端发来的请求准确无误的路由到集群中的一个或者一组服务器上，同时会把接收到的响应拼装起来发回到客户端。
- **保证集群总是可读写**：MongoDB通过多种途径来确保集群的可用性和可靠性。**将MongoDB的分片和复制功能结合使用，在确保数据分片到多台服务器的同时，也确保了每分数据都有相应的备份，这样就可以确保有服务器换掉时，其他的从库可以立即接替坏掉的部分继续工作**。
- **使集群易于扩展**：当系统需要更多的空间和资源的时候，MongoDB使我们可以按需方便的扩充系统容量。

## 3、集群中的数据分布

**在一个shard server内部，MongoDB还是会把数据分为chunks**，每个chunk代表这个shard server内部一部分数据。chunk的产生，会有以下两个用途：

【**Split**】：

  **当一个chunk的大小超过配置中的chunk size（默认是64M）时，MongoDB的后台进程会把这个chunk切分成更小的chunk，从而避免chunk过大的情况**。

【**Balance**】：

  在MongoDB中，**balancer是一个后台进程，负责chunk的迁移，从而均衡各个shard server的负载**，系统初始1个chunk，chunk size默认值64M,生产库上选择适合业务的chunk size是最好的。mongoDB会自动拆分和迁移chunks。

### 1）chunk分裂及迁移

随着数据的增长，其中的**数据大小超过了配置的chunk size，默认是64M，则这个chunk就会分裂成两个**。数据的增长会让chunk分裂得越来越多。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021217.png)

这时候，**各个shard上的chunk数量就会不平衡**。这时候，mongos中的一个组件**balancer就会执行自动平衡**。把chunk从chunk数量最多的shard节点挪动到数量最少的节点。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021245.png)

### 2）chunkSize对分裂及迁移的影响

  MongoDB **默认的 chunkSize 为64MB**，如无特殊需求，建议保持默认值；chunkSize 会直接影响到 chunk 分裂、迁移的行为。

  chunkSize 越小，chunk 分裂及迁移越多，数据分布越均衡；反之，chunkSize 越大，chunk 分裂及迁移会更少，但可能导致数据分布不均。

  chunkSize太小，容易出现 jumbo chunk（即shardKey 的某个取值出现频率很高，这些文档只能放到一个chunk里，无法再分裂）而无法迁移；chunkSize越大，则可能出现chunk内文档数太多（chunk 内文档数不能超过250000）而无法迁移。

  chunk自动分裂只会在数据写入时触发，所以如果将chunkSize 改小，系统需要一定的时间来将chunk分裂到指定的大小。

  chunk只会分裂，不会合并，所以即使将chunkSize改大，现有的chunk数量不会减少，但chunk大小会随着写入不断增长，直到达到目标大小。

## 3、数据区分

  **MongoDB中数据的分片是以集合为基本单位的，集合中的数据通过片键（Shard key）被分成多部分。其实片键就是在集合中选一个键，用该键的值作为数据拆分的依据**。注意：分片键是不可变的而且必须有索引。

### 1）基于范围的分片

  Sharded Cluster支持将单个集合的数据分散存储在多shard上，用户可以指定根据集合内文档的某个字段即shard key来进行范围分片（range sharding）。

  **对于基于范围的分片，MongoDB按照片键的范围把数据分成不同部分**。假设有一个数字的片键:想象一个从负无穷到正无穷的直线，每一个片键的值都在直线上画了一个点。MongoDB把这条直线划分为更短的不重叠的片段，并称之为数据块，每个数据块包含了片键在一定范围内的数据。在使用片键做范围划分的系统中，拥有“相近”片键的文档很可能存储在同一个数据块中，因此也会存储在同一个分片中。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021354.png)

### 2）基于哈希的分片

  分片过程中利用哈希索引作为分片的单个键，且哈希分片的片键只能使用一个字段，而**基于哈希片键最大的好处就是保证数据在各个节点分布基本均匀**。

![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021413.png)

  对于基于哈希的分片，MongoDB计算一个字段的哈希值，并用这个哈希值来创建数据块。在使用基于哈希分片的系统中，拥有“相近”片键的文档很可能不会存储在同一个数据块中，因此数据的分离性更好一些。

  **Hash分片与范围分片互补，能将文档随机的分散到各个chunk，充分的扩展写能力，弥补了范围分片的不足，但不能高效的服务范围查询**，所有的范围查询要分发到后端所有的Shard才能找出满足条件的文档。

# 三、MongoDB 索引

## 1、索引类型

索引通常能够极大的提高查询的效率,如果没有索引,MongoDB在读取数据时必须扫描集合中的每个文件并选取那些符合查询条件的记录。索引主要用于**排序和检索**

**单键索引**

在某一个特定的属性上建立索引,例如：`db.users. createIndex({age:-1});`

- mongoDB在ID上建立了唯一的单键索引,所以经常会使用id来进行查询；
- 在索引字段上进行精确匹配、排序以及范围查找都会使用此索引；

**复合索引**

在多个特定的属性上建立索引,例如：`db.users. createIndex({username:1,age:-1,country:1});`

- 复合索引键的排序顺序,可以确定该索引是否可以支持排序操作；
- 在索引字段上进行精确匹配、排序以及范围查找都会使用此索引,但与索引的顺序有关；
- 为了性能考虑,应删除存在与第一个键相同的单键索引

**多键索引**

在数组的属性上建立索引,例如：`db.users.createIndex({favorites.city:1})`针对这个数组的任意值的查询都会定位到这个文档,既多个索引入口或者键值引用同一个文档

**哈希索引**

不同于传统的B-树索引,哈希索引使用hash函数来创建索引。例如：`db.users.createIndex({username : 'hashed'})`

- 在索引字段上进行精确匹配,但不支持范围查询,不支持多键hash；
- Hash索引上的入口是均匀分布的,在分片集合中非常有用；

## 2、索引语法

MongoDB使用 ensureIndex() 方法来创建索引,ensureIndex()方法基本语法格式如下所示:

`db.collection.createIndex(keys, options)`

- 语法中 Key 值为要创建的索引字段,**1为指定按升序**创建索引,如果你想按降序来创建索引**指定为-1**,也可以指定为**hashed**（哈希索引）。
- 语法中options为索引的属性,属性说明见下表；

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613021937.jpg)

**创建索引**

- 单键唯一索引：`db.users. createIndex({username :1},{unique:true});`
- 单键唯一稀疏索引：`db.users. createIndex({username :1},{unique:true,sparse:true});`
- 复合唯一稀疏索引：`db.users. createIndex({username:1,age:-1},{unique:true,sparse:true});`
- 创建哈希索引并后台运行：`db.users. createIndex({username :'hashed'},{background:true});`


**删除索引**

- 根据索引名字删除某一个指定索引：`db.users.dropIndex("username_1");`
- 删除某集合上所有索引：`db.users.dropIndexs();`
- 重建某集合上所有索引：`db.users.reIndex();`
- 查询集合上所有索引：`db.users.getIndexes();`

## 3、查询优化

### 1）**找出慢速查询**

**开启内置的查询分析器,记录读写操作效率:**

`db.setProfilingLevel(n,{m})`，n的取值可选0,1,2；

> - 0是默认值表示不记录；
>
> - 1表示记录慢速操作,如果值为1,m必须赋值单位为ms,用于定义慢速查询时间的阈值；
> - 2表示记录所有的读写操作；
>
> 例如:db.setProfilingLevel(1,300)

**查询监控结果**

监控结果保存在一个特殊的盖子集合system.profile里,这个集合分配了128kb的空间,要确保监控分析数据不会消耗太多的系统性资源；盖子集合维护了自然的插入顺序,可以使用$natural操作符进行排序,如：`db.system.profile.find().sort({'$natural':-1}).limit(5)`

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613022534.jpg)

### 2）分析慢速查询

找出慢速查询的原因比较棘手,原因可能有多个:应用程序设计不合理、不正确的数据模型、硬件配置问题,缺少索引等；接下来对于缺少索引的情况进行分析:

**使用explain分析慢速查询**

例如：`db.orders.find({'price':{'$lt':2000}}).explain('executionStats')`

explain的入参可选值为:

-  `queryPlanner` 是默认值,表示仅仅展示执行计划信息；

-  **`executionStats`** 表示展示执行计划信息同时展示被选中的执行计划的执行情况信息；
-  `allPlansExecution` 表示展示执行计划信息,并展示被选中的执行计划的执行情况信息,还展示备选的执行计划的执行情况信息；

**解读explain结果**

queryPlanner（执行计划描述）

​      winningPlan（被选中的执行计划）

​          stage（可选项:COLLSCAN 没有走索引；**IXSCAN**使用了索引）

​      rejectedPlans(候选的执行计划)

  executionStats(执行情况描述)

​      nReturned （返回的文档个数）

​      executionTimeMillis（执行时间ms）

​      totalKeysExamined （检查的索引键值个数）

​      totalDocsExamined （检查的文档个数）

优化目标 Tips:

1. 根据需求建立索引
2. 每个查询都要使用索引以提高查询效率, **winningPlan. stage 必须为IXSCAN** ；
3. 追求**totalDocsExamined = nReturned**

### 3）关于索引的建议

1. 索引很有用,但是它也是**有成本的**——它占内存，让写入变慢；
2. mongoDB通常在一次查询里**使用一个索引**，所以多个字段的查询或者排序需要复合索引才能更加高效；
3. **复合索引的顺序**非常重要
4. 在生成环境构建索引往往开销很大，时间也不可以接受，在数据量庞大之前尽量进行查询优化和构建索引；
5. 避免昂贵的查询,使用查询分析器记录那些开销很大的查询便于问题排查；
6. 通过减少扫描文档数量来优化查询,使用explai对开销大的查询进行分析并优化；
7. 索引是用来查询小范围数据的，**不适合使用索引**的情况：
   1. 每次查询**都需要返回大部分数据**的文档，避免使用索引

   2. **写比读多**

# 四、WiredTiger 存储引擎

存储引擎是MongoDB的核心组件，负责管理数据如何存储在硬盘和内存上。从MongoDB 3.2 版本开始，MongoDB 支持多数据存储引擎，MongoDB支持的存储引擎有：**WiredTiger**，**MMAPv1**和**In-Memory**。

**从mongodb3.2开始默认的存储引擎是WiredTiger**，3.2版本之前的默认存储引擎是MMAPv1，mongodb4.x版本不再支持MMAPv1存储引擎。

MongoDB不仅能将数据持久化存储到硬盘文件中，而且还能将数据**只保存到内存中**；In-Memory存储引擎用于将数据只存储在内存中，只将少量的元数据和诊断日志（Diagnostic）存储到硬盘文件中，由于不需要Disk的IO操作，就能获取索取的数据，In-Memory存储引擎大幅度降低了数据查询的延迟（Latency）

### 1、逻辑架构

Mongodb里一个典型的Wiredtiger数据库存储布局大致如下：

```shell
$tree
.
├── journal

│   ├── WiredTigerLog.0000000003

│   └── WiredTigerPreplog.0000000001

├── WiredTiger

├── WiredTiger.basecfg

├── WiredTiger.lock

├── WiredTiger.turtle

├── admin

│   ├── table1.wt

│   └── table2.wt

├── local

│   ├── table1.wt

│   └── table2.wt

└── WiredTiger.wt
```

- WiredTiger.basecfg存储基本配置信息
- WiredTiger.lock用于防止多个进程连接同一个Wiredtiger数据库
- table*.wt存储各个tale（数据库中的表）的数据
- WiredTiger.wt是特殊的table，用于存储所有其他table的元数据信息
- WiredTiger.turtle存储WiredTiger.wt的元数据信息
- journal存储**Write ahead log**

按照Mongodb默认的配置，WiredTiger的**写操作会先写入Cache**，并**持久化到WAL**(Write ahead log)，**每60s**或log文件**达到2GB时**会做一次**Checkpoint**，将当前的数据持久化，产生一个新的快照。Wiredtiger连接初始化时，首先将数据恢复至最新的快照状态，然后根据WAL恢复数据，以保证存储可靠性。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613030852.png)

Wiredtiger的Cache**采用Btree**的方式组织，每个Btree节点为一个page，root page是btree的根节点，internal page是btree的中间索引节点，leaf page是**真正存储数据**的叶子节点；btree的数据**以page为单位**按需**从磁盘加载或写入磁盘**。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613031941.jpg)

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613031139.jpeg)



Wiredtiger采用**Copy on write（写时复制）**的方式管理修改操作（insert、update、delete），修改操作会**先缓存在cache里**，持久化时，修改操作不会在原来的leaf page上进行，而是**写入新分配的page**，**每次checkpoint都会产生一个新的root page**。

Checkpoint时，wiredtiger需要将btree修改过的PAGE都进行持久化存储，每个btree**对应磁盘上一个物理文件**，btree的每个PAGE以文件里的**extent**形式（由文件offset + size标识）存储，一个Checkpoit包含如下元数据：

- root page地址，地址由文件offset，size及内容的checksum组成
- alloc extent list地址，存储从上次checkpoint起新分配的extent列表
- discard extent list地址，存储从上次checkpoint起丢弃的extent列表
- available extent list地址，存储可分配的extent列表，只有最新的checkpoint包含该列表
- file size 如需恢复到该checkpoint的状态，将文件truncate到file size即可

### 2、特性

#### 1）文档级并发

WiredTiger使用**文档级并发**控制进行写操作。因此，多个客户端**可以同时修改集合的不同文档**。

对于大多数读写操作，WiredTiger使用乐观并发控制。WiredTiger仅在全局，数据库和集合级别使用意图锁。当存储引擎检测到两个操作之间的冲突时，会发生写入冲突，导致MongoDB透明地重试该操作。

一些全局操作（通常是涉及多个数据库的短期操作）仍然需要全局“实例范围”锁定。其他一些操作（例如删除集合）仍需要独占数据库锁。

#### 2）checkpoint

在Checkpoint操作开始时，WiredTiger提供指定时间点（point-in-time）的**数据库快照**（Snapshot），该Snapshot呈现的是**内存中数据的一致性视图**。当向Disk写入数据时，WiredTiger将Snapshot中的所有数据以一致性方式写入到数据文件（Disk Files）中。一旦Checkpoint创建成功，WiredTiger保证数据文件和内存数据是**一致性的**，因此，Checkpoint担当的是**还原点**（Recovery Point），Checkpoint操作能够缩短MongoDB从Journal日志文件还原数据的时间。

当WiredTiger创建Checkpoint时，MongoDB将数据刷新到数据文件（Disk Files）中，在默认情况下，WiredTiger创建Checkpoint的**时间间隔是60s**，或产生**2GB**的Journal文件。在WiredTiger创建新的Checkpoint期间，上一个Checkpoint仍然是有效的，这意味着，即使MongoDB在创建新的Checkpoint期间遭遇到错误而异常终止运行，只要重启，MongoDB就能从上一个有效的Checkpoint开始还原数据。

当MongoDB以**原子方式**更新WiredTiger的元数据表，使其引用新的Checkpoint时，表明新的Checkpoint创建成功，MongoDB将老的Checkpoint占用的Disk空间释放。使用WiredTiger 存储引擎，如果没有记录数据更新的日志，MongoDB只能还原到上一个Checkpoint；如果要还原在上一个Checkpoint之后执行的修改操作，必须**使用Jounal日志**文件。

#### 3）journal日志

WiredTiger使用**预写日志的机制**，在数据更新时，先将数据更新写入到日志文件，然后在创建Checkpoint操作开始时，将日志文件中记录的操作，刷新到数据文件，就是说，通过预写日志和Checkpoint，将数据更新持久化到数据文件中，实现数据的一致性。WiredTiger 日志文件会持久化记录从上一次Checkpoint操作之后发生的所有数据更新，在MongoDB系统崩溃时，通过日志文件能够还原从上次Checkpoint操作之后发生的数据更新。

#### 4）压缩

使用WiredTiger，MongoDB支持对所有集合和索引进行压缩。压缩可以以额外的CPU为代价最大限度地**减少磁盘的使用**。

默认情况下WiredTiger存储引擎使用**snappy**方式压缩所有集合，前缀压缩方式压缩索引。

对于集合的压缩还可以使用**zlib**方式压缩。

#### 5）内存使用

最大化可用缓存： WiredTiger**最大限度地利用**可用内存作为缓存来减少I / O瓶颈。使用了两个缓存：**WiredTiger缓存**和**文件系统缓存**。

- WiredTiger缓存存储**未压缩的数据**并提供类似内存的性能。
- 操作系统的文件系统缓存**存储压缩数据**。当在WiredTiger缓存中找不到数据时，WiredTiger将在文件系统缓存中查找数据。

mongodb从3.4版本开始默认使用内存为下面两个中的最大一个：

- 50% of (RAM - 1 GB)
- 256MB

默认情况下，WiredTiger对所有集合使用Snappy块压缩，对所有索引使用前缀压缩。压缩默认值可在全局级别配置，也可以在集合和索引创建时单独指定压缩级别。

wiredTiger内部缓存中的数据与磁盘格式使用**不同的表示形式**。

#### 6）disk回收

当从MongoDB中删除文档（Documents）或集合（Collections）后，MongoDB**不会将Disk空间释放**给OS，MongoDB在数据文件（Data Files）中维护**Empty Records**的列表。当重新插入数据后，MongoDB从Empty Records列表中分配存储空间给新的Document，因此，不需要重新开辟空间。为了更新有效的**重用Disk空间**，必须重新整理数据碎片。

> 由于MongoDB的机制，被删除的数据在磁盘上并没有真正的删除，只是做了一个标记，称之为**空洞**，这些大量的空洞也会被加载到内存，导致内存的利用率降低，类似于磁盘碎片。
>
> MongoDB提供了一个在线的空洞整理命令**compact**，针对表级别的空洞整理，但是这个命令的性能比较差，如果在线运行的时候，会影响用户的实际体验，推荐只在在线用户比较少的时候进行。
>
> `db.runCommand ( { compact: '<collection>' } )`

# 五、重要机制

## 1、MongoDB数据文件内部结构

![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613033453.png)

- MongoDB在数据存储上按命名空间来划分，一个Collection是一个命名空间，一个索引也是一个命名空间。
- **同一个命名空间的数据被分成很多个Extent，Extent之间使用双向链表连接**。
- **在每一个Extent中，保存了具体每一行的数据，这些数据也是通过双向链表来连接的**。
- 每一行数据存储空间不仅包括数据占用空间，还可能包含一部分附加空间，这使得在数据Update变大后可以不移动位置。
- **索引以BTree结构实现**。

## 2、MongoDB数据同步

![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613033508.png)

- 红色箭头表示写操作可以写到Primary上，然后**异步同步**到多个Secondary上。
- 蓝色箭头表示读操作可以从Primary或Secondary任意一个中读取。
- **各个Primary与Secondary之间一直保持心跳同步检测，用于判断Replica Sets的状态**。

## 3、分片机制

![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613033549.png)

- MongoDB的分片是指定一个分片key来进行，数据按范围分成**不同的chunk**，每个chunk的大小有限制。
- 有多个分片节点保存这些chunk，每个节点保存一部分的chunk。
- **每一个分片节点都是一个Replica Sets**，这样保证数据的安全性。
- **当一个chunk超过其限制的最大体积时，会分裂成两个小的chunk**。
- **当chunk在分片节点中分布不均衡时，会引发chunk迁移操作**。

## 4、服务器角色

![（图）](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613033637.png)

- 客户端访问路由节点**mongos来进行数据读写**。
- **config服务器保存了两个映射关系，一个是key值的区间对应哪一个chunk的映射关系，另一个是chunk存在哪一个分片节点的映射关系**。
- **路由节点通过config服务器获取数据信息，通过这些信息，找到真正存放数据的分片节点进行对应操作**。
- 路由节点还会在写操作时判断当前chunk是否超出限定大小。如果超出，就分列成两个chunk。
- 对于按分片key进行的查询和update操作来说，路由节点会查到具体的chunk然后再进行相关的工作。
- 对于不按分片key进行的查询和update操作来说，mongos会对所有下属节点发送请求然后再对返回结果进行合并。

# 六、MongoDB数据一致性问题

## 1、WriteConcern

![在这里插入图片描述](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200613033749.png)
  

mongodb的写入策略有多种方式，写入策略是指当客户端发起写入请求后，数据库什么时候给应答，mongodb有三种处理策略：客户端发出去的时候，服务器收到请求的时候，服务器写入磁盘的时候。

- **Replica Acknowledged**：这个方式和Acknowledged是一样的意思，适用于Replica sets模式。Acknowledged模式下只有一台机器收到了请求就返回了，对于复制集模式有多台机器的情况，**可以要求有多台机器收到写入请求后再响应客户端。这种更安全，但是导致了客户端耗时增加**，所以要结合自己的场景设置合适的策略。
- **writeConcern: { w: “majority” }**，**majority表示多数节点写入成功后才响应客户端**，**也可以替换成具体的数子**，比如w:2表示至少写入2个节点才返回。wtimeout表示超时时间，还有一个参数可以设置true，false表示是否是写入日志才返回。

## 2、ReadConcern

  readConcern 决定到某个读取数据时，能读到什么样的数据。

  Local：**能读取任意数据，这个是默认设置**。

  Majority：**只能读取到成功写入到大多数节点的数据**。

## 3、MongoDB如何保证数据一致性？

  WriteConcern 的语义是说，「**写大多数副本成功后才向客户端确认**」，所以它只能保证「server 端告诉你写成功了，那么它一定成功的写到大多数节点了」；**而当 WriteConcern: majority 失败时，它不能保证一定就失败了，具体几个节点成功是不确定的**（比如错误是网络超时，你没法确定 server 端最终的状态）。

  WriteConcern 跟一致性是2个不同的问题，数据的一致性在 MongoDB 里是由 **RAFT** 最终来保证的。

  **如果读写都在主节点上，当主节点挂掉以后通过bully算法重新选主，在这个过程中整个系统暂时不可用；选主过程中，副本节点通过日志复制保证和主节点的数据一致性，当新的主节点选出来以后，整个系统对外是保证了数据一致性的，但是牺牲了一些可用性**。

# 七、为什么MongoDB使用B树，而MySQL使用B+树

B树的树内存储数据，因此查询单条数据的时候，B树的查询效率不固定，最好的情况是O(1)。我们可以认为在做单一数据查询的时候，使用B树平均性能更好。但是，由于B树中各节点之间没有指针相邻，因此**B树不适合做一些数据遍历操作**。

B+树的数据只出现在叶子节点上，因此在查询单条数据的时候，查询速度非常稳定。因此，**在做单一数据的查询上，其平均性能并不如B树**。但是，B+树的叶子节点上有指针进行相连，因此在做数据遍历的时候，只需要对叶子节点进行遍历即可，这个特性使得**B+树非常适合做范围查询**。

因此，我们可以做一个推论:没准是Mysql中数据遍历操作比较多，所以用B+树作为索引结构。而Mongodb是做单一查询比较多，数据遍历操作比较少，所以用B树作为索引结构。

那么为什么Mysql做数据遍历操作多？而Mongodb做数据遍历操作少呢？

因为Mysql是关系型数据库，而Mongodb是非关系型数据。

MySQL经常需要联表查询，这时就需要做范围查询了。

而MongoDB是非关系数据库，是松散的结构，一般是用于单表查询，因此使用B树会更合适。