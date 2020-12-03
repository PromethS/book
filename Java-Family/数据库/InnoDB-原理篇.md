# 前言

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200527163736)

上图为Mysql的架构图，sql的执行会经过连接器->缓存->分析器->优化器->执行器->存储引擎，具体每步的作用详见《MySql篇》

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200527163644)



上图为InnoDB引擎的架构图，整体分为**内存区、磁盘区**。最底下的是**存储引擎层**（Storage Engines），它决定了 MySQL 会怎样存储数据，怎样读取和写入数据，也在很大程度上决定了 MySQL 的读写性能和数据可靠性。存储引擎层有极大的扩展性，可以自定义使用innodb、myisam、memory或自定义。

# InnoDB 内存架构

## **Buffer Pool**

MySQL **不会直接去修改磁盘的数据**，因为这样做太慢了，MySQL **会先改内存，然后记录 redo log，等有空了再刷磁盘，如果内存里没有数据，就去磁盘 load**。

而这些数据存放的地方，就是 Buffer Pool。

MySQL 是以**「页」（page）为单位从磁盘读取数据**的，Buffer Pool 里的数据也是如此，实际上，Buffer Pool 是`a linked list of pages`，一个以页为元素的链表。

为什么是链表？因为和缓存一样，它也需要**一套淘汰算法来管理数据**。Buffer Pool 采用基于 **LRU**（least recently used） 的算法来管理内存：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200527165718)

InnoDB缓存池（buffer pool）缓存**索引**、**行的数据**、**自适应哈希索引**、**插入缓存**（Insert Buffer）、**锁** 还有其他的内部数据结构。

**Innodb_buffer_pool_size**

该参数定义了 InnoDB 存储引擎的表数据和索引数据的**最大内存缓冲区**大小。和 MyISAM 存储引擎不同，MyISAM 的 key_buffer_size只缓存索引键， 而 innodb_buffer_pool_size 却是同时为**数据块和索引块**做缓存，这个特性和 Oracle 是一样的。这个值设得越高，访问 表中数据需要的磁盘 I/O 就越少。在一个专用的数据库服务器上，可以设置这个参数达机器 物理内存大小的 **80%**。尽管如此，还是建议用户不要把它设置得太大，因为对物理内存的竞争可能在操作系统上导致内存调度。

将书的话解释过来就是该参数代表了用来缓存索引数据和表数据的内存大小，数据在内存中的读写速度是磁盘读写速度的很多倍，当数据满足条件后才将内存中的数据刷入磁盘中，innodb利用缓存池来帮助延迟写入，合并多个写入操作顺序写入磁盘。

Innodb_buffer_pool_size = 系统可用内存  - 系统正常运行内存 - （峰值时的连接数 * 每个连接需要的内存）

## **Change Buffer**

Change Buffer是Buffer Pool的一块区域，用于**记录对数据的修改**。

如果内存里没有对应「页」的数据，MySQL 就会去把数据从磁盘里 load 出来，如果每次需要的「页」都不同，或者不是相邻的「页」，那么每次 MySQL 都要去 load，这样就很慢了。

于是如果 MySQL 发现你要修改的页，不在内存里，就把你要对页的修改，**先记到一个叫 Change Buffer 的地方，同时记录 redo log**，然后再慢慢把数据 load 到内存，load 过来后，再把 Change Buffer 里记录的修改，应用到内存（Buffer Pool）中，这个动作叫做 **merge**；而把内存数据刷到磁盘的动作，叫 **purge**：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200527170105)

### 使用条件

**对于唯一索引**，所有更新操作都需要做唯一性约束的判断，必须将数据页读入内存，直接在内存中更新，**不使用 change buffer** 。

**对于普通索引，当数据页在内存中时，直接进行更新操作即可**；**当数据页不在内存中时，直接将更新操作写入 change buffer 即可。**

主要是为了解决二级索引的随机访问，因为二级索引可能分布的比较松散，**访问非常随机**。

### 使用场景

change buffer 的主要作用就是将记录的变更操作缓存下来，在 merge 之前， change buffer 记录的越多，收益就越大。

**适合**页面变更完之后**被马上访问概率较小**的场景。

**如果页面变更之后又要被访问，此时会立即触发 merge 过程**，这样反而增加了 change buffer 的维护代价，多了一个写 change buffer 的操作，此时关闭 change buffer 反而能提高效率；



Change Buffer **只在操作「二级索引」**（secondary index）且**不是唯一索引**时才使用，原因是**「聚簇索引」（clustered indexes）必须是「唯一」的**，也就意味着每次插入、更新，都需要检查是否已经有相同的字段存在，也就没有必要使用 Change Buffer 了；另外，**「聚簇索引」操作的随机性比较小**，通常在相邻的「页」进行操作，比如使用了自增主键的「聚簇索引」，**那么 insert 时就是递增、有序的**，不像「二级索引」，**访问非常随机**。

> change buffer包含了insert buffer，可以对**insert/delete/update**操作的修改信息进行缓存

## Adaptive Hash Index（自适应Hash索引）

MySQL 索引，不管是在磁盘里，还是被 load 到内存后，都是 B+ 树，B+ 树的查找次数取决于树的深度。你看，数据都已经放到内存了，还不能“一下子”就找到它，还要“几下子”，这空间牺牲的是不是不太值得？

尤其是那些频繁被访问的数据，每次过来都要走 B+ 树来查询，这时就会想到，我用一个指针把数据的位置记录下来不就好了？

这就是「**自适应哈希索引**」（Adaptive Hash Index）。自适应，顾名思义，MySQL 会自动评估使用自适应索引是否值得，如果观察到建立哈希索引可以提升速度，则建立。

## log buffer

从上面架构图可以看到，Log Buffer 里的 redo log，会被刷到磁盘里：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200527170414)

## Operating System Cache

在内存和磁盘之间，你看到 MySQL 画了一层叫做 Operating System Cache 的东西，其实这个不属于 InnoDB 的能力，而是操作系统为了提升性能，在磁盘前面加的一层高速缓存

# InnoDB 磁盘架构

除了表结构定义和索引，还有一些**为了高性能和高可靠而设计的角色**，比如 **redo log、undo log、Change Buffer，以及 Doublewrite Buffer** 等等。

## Doublewrite Buffer

**如果说 Change Buffer 是提升性能，那么 Doublewrite Buffer 就是保证数据页的可靠性。**

前面提到过，MySQL 以「页」为读取和写入单位，一个「页」里面有多行数据，写入数据时，MySQL 会先写内存中的页，然后再刷新到磁盘中的页。

这时问题来了，假设在某一次从内存刷新到磁盘的过程中，一个「页」刷了一半，突然操作系统或者 MySQL 进程奔溃了，这时候，内存里的页数据被清除了，而磁盘里的页数据，刷了一半，处于一个中间状态，不尴不尬，可以说是一个「不完整」，甚至是「坏掉的」的页。

有同学说，不是有 Redo Log 么？其实这个时候 Redo Log 也已经无力回天，Redo Log 是要在磁盘中的页数据是正常的、没有损坏的情况下，才能把磁盘里页数据 load 到内存，然后应用 Redo Log。而如果磁盘中的页数据已经损坏，是无法应用 Redo Log 的。

所以，**MySQL 在刷数据到磁盘之前，要先把数据写到另外一个地方，也就是 Doublewrite Buffer，写完后，再开始写磁盘**。Doublewrite Buffer 可以理解为是一个备份（recovery），万一真的发生 crash，就可以利用 Doublewrite Buffer 来修复磁盘里的数据。

## redo log

**数据来了，写磁盘，还是写内存？**

写磁盘，嫌太慢？写内存，又不安全？

MySQL 的解决方案是：**既写磁盘又写内存。**

数据写内存，另外**再往磁盘写 redo log**.

这时，内存的数据是新的、正确的，而磁盘数据是旧的、过时的，所以我们称这时的磁盘对应的页数据，为**「脏页」**。然后在空闲的时候，再把内存的数据刷到磁盘。redo log 还是要写磁盘，那不还是很慢？

并不是，把 redo log 写到磁盘，比一般的写磁盘要快，原因有：

- 一般我们写磁盘，都是「随机写」，**而 redo log，是「顺序写」**
- MySQL 在写 redo log 上做了优化，比如**「组提交」**

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200527173148)

**redo log 怎么存储：**

- redo log 记录了 sql 语句以及其他 api，对表数据产生的变化，也就是说，redo log 记录的是数据的**物理变化**，这也是它和后面要讲的 binlog 一个最大的区别，binlog 记录的是数据的**逻辑变化**，这也是为什么 redo log 可以用来 crash recovery，而 binlog 不能的原因之一
- redo log 是存储在磁盘的，用于 crash recovery 后修正数据的，也就是我们常说的故障恢复，比如掉电，宕机等等
- redo log 默认有两个文件
- redo log 是循环写的（circular）

我们常说事务具有 ACID 四个特性，其中 D（durability），数据持久性，意味着，**一旦事务提交，它的状态就必须保持提交，不能回滚**，哪怕你系统宕机了、奔溃了，你也要想办法把事务做到提交，把数据给我保存进去。

**所以我们说，innodb 在实现高性能写数据的同时，利用 redo log，实现了事务的持久性。**

## undo log

undo log 主要为**事务的回滚服务**。在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。undo log**记录了数据在每个操作前的状态**，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。单个事务的回滚，只会回滚当前事务做的操作，并不会影响到其他的事务做的操作。

Undo记录的是已部分完成并且写入硬盘的未完成的事务，默认情况下回滚日志是记录下表空间中的（共享表空间或者独享表空间）

二种日志均可以视为一种恢复操作，**redo_log是恢复提交事务修改的页操作，而undo_log是回滚行记录到特定版本**。二者记录的内容也不同，**redo_log是物理日志，记录页的物理修改操作，而undo_log是逻辑日志，根据每行记录进行记录**。

## binlog

在update 语句执行的时候，除了生成 redo log，还会生成 binlog。

binlog 和 redo log 有很多不同，有一点是一定要知道的，就是 redo log 只是 innodb 存储引擎的功能，而 binlog 是 MySQL server 层的功能，也就是说，**redo log 只在使用了 innodb 作为存储引擎的 MySQL 上才有，而 binlog，只要你是 MySQL，就会有。**

binlog 还有另一个作用：**主从复制**，主库把 binlog 发给从库，从库把 binlog 保存了下来，然后去执行它，这样就实现了主从同步。

当然，我们还能让自己的业务应用，去监听主库的 binlog，当数据库的数据发生变动时，去做特定的事情，比如进行数据实时统计。

## 事务提交过程-两阶段提交

当我执行一条 update 语句时，redo log 和 binlog 是在什么时候被写入的呢？这就有了我们常说的「两阶段提交」：

- 写入：redo log（prepare）
- 写入：binlog
- 写入：redo log（commit）

为什么 redo log 要分两个阶段： prepare 和 commit ？redo log 就不能一次写入吗？

我们分两种情况讨论：

- 先写 redo log，再写 binlog
- 先写 binlog，再写 redo log

1、先写 redo log，再写 binlog

这样会出现 redo log 写入到磁盘了，但是 binlog 还没写入磁盘，于是当发生 crash recovery 时，恢复后，主库会应用 redo log，恢复数据，但是由于没有 binlog，从库就不会同步这些数据，主库比从库“新”，造成主从不一致

2、先写 binlog，再写 redo log

跟上一种情况类似，很容易知道，这样会反过来，造成从库比主库“新”，也会造成主从不一致

而两阶段提交，就解决这个问题，crash recovery 时：

- 如果 redo log 已经 commit，那毫不犹豫的，把事务提交

- 如果 redo log 处于 prepare，则去判断事务对应的 binlog 是不是完整的

- - 是，则把事务提交
  - 否，则事务回滚

两阶段提交，其实是为了保证 redo log 和 binlog 的逻辑一致性。

# InnoDB 逻辑存储架构

从InnoDB存储引擎的逻辑结构看，所有数据都被逻辑地存放在一个空间内，称为表空间，而表空间由段（sengment）、区（extent）、页（page）组成。ps：页在一些文档中又称块（block）。

InnoDB存储引擎的逻辑存储结构大致如下：

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200701233229.png)

**一、表空间（table space）**

表空间分为了两种，这里简单的概括一下：

1. 独立表空间：每一个表都将会生成以独立的文件方式来进行存储，每一个表都有一个.frm表描述文件，还有一个.ibd文件。 其中这个文件包括了单独一个表的数据内容以及索引内容，默认情况下它的存储位置也是在表的位置之中。

2. 共享表空间： Innodb的所有数据保存在一个单独的表空间里面，而这个表空间**可以由很多个文件**组成，一个表可以跨多个文件存在，所以其大小限制不再是文件大小的限制，而是其自身的限制。从Innodb的官方文档中可以看到，其表空间的最大限制为64TB，也就是说，Innodb的单表限制基本上也在64TB左右了，当然这个大小是包括这个表的所有索引等其他相关数据。

InnoDB把数据保存在表空间内，表空间可以看作是InnoDB存储引擎逻辑结构的**最高层**。本质上是一个由一个或多个磁盘文件组成的**虚拟文件系统**。InnoDB用表空间并不只是存储表和索引，还保存了回滚段、双写缓冲区等。

**二、段（segment）**

表空间是由各个段组成的，常见的段有**数据段**、**索引段**、**回滚段**等。前一篇文章（[MySQL InnoDB 索引组织表 & 主键作用](https://www.cnblogs.com/wilburxu/p/9419275.html)）已经介绍了关于InnoDB存储引擎室索引组织（index organized），因此数据即索引，索引即数据。那么数据段即为B+树段叶子节点（Leaf node segment），索引段即为B+树段非索引节点。

**三、区（extent）**

区是由连续的页（Page）组成的空间，在任何情况下每个区大小都为**1MB**，为了保证页的连续性，InnoDB存储引擎每次从磁盘一次申请4-5个区。默认情况下，InnoDB存储引擎的页大小为16KB，即一个区中有**64个连续的页**。 （1MB／16KB=64）

InnoDB1.0.x版本开始引入压缩页，每个页的大小可以通过参数KEY_BLOCK_SIZE设置为2K、4K、8K，因此每个区对应的页尾512、256、128.

InnpDB1.2.x版本新增了参数innodb_page_size，通过该参数可以将默认页的大小设置为4K、8K，但是页中的数据不是压缩的。

但是有时候为了节约磁盘容量的开销，创建表默认大小是96KB，区中是64个连续的页。（对于一些小表）

**四、页（Page）**

页是InnoDB存储引擎磁盘管理的最小单位，每个页默认16KB；InnoDB存储引擎从1.2.x版本碍事，可以通过参数innodb_page_size将页的大小设置为4K、8K、16K。若设置完成，则所有表中页的大小都为innodb_page_size，不可以再次对其进行修改，除非通过mysqldump导入和导出操作来产生新的库。

innoDB存储引擎中，常见的页类型有：

1. 数据页（B-tree Node)

2. undo页（undo Log Page）

3. 系统页 （System Page）

4. 事物数据页 （Transaction System Page）

5. 插入缓冲位图页（Insert Buffer Bitmap）

6. 插入缓冲空闲列表页（Insert Buffer Free List）

7. 未压缩的二进制大对象页（Uncompressed BLOB Page）

8. 压缩的二进制大对象页 （compressed BLOB Page）

**五、行（row）**

InnoDB存储引擎是面向列的（row-oriented)，也就是说数据是按行进行存放的，每个页存放的行记录也是有硬性定义的，最多允许存放16KB/2-200，即7992行记录。

# InnoDB 四大特性

## 插入缓冲（insert buffer)
插入缓冲（Insert Buffer/Change Buffer）：**提升插入性能**，change buffer是insert buffer的加强，insert buffer只针对insert有效，change buffering对insert、delete、update(delete+insert)、purge都有效

**只对于非聚集索引（非唯一）的插入和更新有效**，对于每一次的插入不是写到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，如果在则直接插入；若不在，则先放到Insert Buffer 中，再按照一定的频率进行合并操作，再写回disk。这样通常能将多个插入合并到一个操作中，**目的还是为了减少随机IO带来性能损耗。**

使用插入缓冲的条件：
**\* 非聚集索引**
**\* 非唯一索引**

Change buffer是作为**buffer pool中的一部分**存在。

innodb_change_buffering，设置的值有：inserts、deletes、purges、changes（inserts和deletes）、all（默认）、none。

all: 默认值，缓存insert, delete, purges操作
none: 不缓存
inserts: 缓存insert操作
deletes: 缓存delete操作
changes: 缓存insert和delete操作
purges: 缓存后台执行的物理删除操作

可以通过参数控制其使用的大小：
innodb_change_buffer_max_size，默认是25%，即缓冲池的1/4。最大可设置为50%。

MySQL实例中有大量的修改操作时，要考虑增大**innodb_change_buffer_max_size**

上面提过在一定频率下进行合并，那所谓的频率是什么条件？

1）辅助索引页被读取到缓冲池中。**正常的select先检查Insert Buffer是否有该非聚集索引页存在，若有则合并插入**。

2）**辅助索引页没有可用空间**。空间小于1/32页的大小，则会强制合并操作。

3）Master Thread 每秒和**每10秒的合并操作**。

## 二次写(double write)

Doublewrite缓存是位于系统表空间的存储区域，用来缓存InnoDB的数据页从innodb buffer pool中flush之后并**写入到数据文件之前**，当操作系统或者数据库进程在数据页写磁盘的过程中崩溃，Innodb可以在doublewrite缓存中找到数据页的备份而用来执行crash恢复。数据页写入到doublewrite缓存的动作所需要的IO消耗要小于写入到数据文件的消耗，因为**此写入操作会以一次大的连续块的方式写入**

在应用（apply）重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本来还原该页，再进行重做，这就是double write
doublewrite组成：
**内存中的doublewrite buffer,大小2M。**

**物理磁盘上共享表空间中连续的128个页，即2个区（extend-簇），大小同样为2M。**

对缓冲池的脏页进行刷新时，不是直接写磁盘，而是会通过memcpy()函数将脏页先复制到内存中的doublewrite buffer，之后**通过doublewrite 再分两次，每次1M顺序地写入共享表空间的物理磁盘上，在这个过程中，因为doublewrite页是连续的，因此这个过程是顺序写的，开销并不是很大**。在完成doublewrite页的写入后，再将doublewrite buffer 中的页写入各个 表空间文件中，此时的写入则是离散的。如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，innodb可以从共享表空间中的doublewrite中找到该页的一个副本，将其复制到表空间文件，再应用重做日志。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200528142249.jpg)

## 自适应哈希索引

Adaptive Hash index属性使得InnoDB更像是内存数据库。该属性通过innodb_adapitve_hash_index开启，也可以通过—skip-innodb_adaptive_hash_index参数
关闭

Innodb存储引擎会监控对表上二级索引的查找，如果**发现某二级索引被频繁访问，二级索引成为热数据，建立哈希索引可以带来速度的提升**

经常访问的二级索引数据会自动被生成到hash索引里面去(最近连续被访问三次的数据)，自适应哈希索引通过缓冲池的B+树构造而来，因此建立的速度很快。
**哈希（hash）是一种非常快的等值查找方法，在一般情况下这种查找的时间复杂度为O(1)**,即一般仅需要一次查找就能定位数据。而B+树的查找次数，取决于B+树的高度，在生产环境中，B+树的高度一般3-4层，故需要3-4次的查询。

innodb会监控对表上个索引页的查询。如果观察到建立哈希索引可以带来速度提升，则**自动建立哈希索引，称之为自适应哈希索引**（Adaptive Hash Index，AHI）。
AHI有一个要求，就是对这个页的连续访问模式必须是一样的。
例如对于（a,b）访问模式情况：
where a = xxx
where a = xxx and b = xxx

- 特点
  　　1、无序，没有树高
    2、降低对二级索引树的频繁访问资源，索引树高<=4，访问索引：访问树、根节点、叶子节点
    3、自适应
- 缺陷
  　　1、hash自适应索引会占用innodb buffer pool；
    2、自适应hash索引**只适合搜索等值的查询**，如select * from table where index_col='xxx'，而对于其他查找类型，如范围查找，是不能使用的；
    3、极端情况下，自适应hash索引才有比较大的意义，可以降低逻辑读。

## **预读(read ahead)**
InnoDB使用两种预读算法来提高I/O性能：线性预读（linear read-ahead）和随机预读（randomread-ahead）
为了区分这两种预读的方式，我们可以把**线性预读放到以extent为单位**，**而随机预读放到以extent中的page为单位**。线性预读着眼于将下一个extent提前读取到buffer pool中，而随机预读着眼于将当前extent中的剩余的page提前读取到buffer pool中。

### 线性预读（linear read-ahead）

有一个很重要的变量控制是否将下一个extent预读到buffer pool中，通过使用配置参数innodb_read_ahead_threshold，可以控制Innodb执行预读操作的时间。如果一个extent中的被顺序读取的page超过或者等于该参数变量时，Innodb将会**异步的将下一个extent**读取到buffer pool中，innodb_read_ahead_threshold可以设置为0-64的任何值，默认值为56，值越高，访问模式检查越严格
例如，如果将值设置为48，则InnoDB只有在顺序访问当前extent中的48个pages时才触发线性预读请求，将下一个extent读到内存中。如果值为8，InnoDB触发异步预读，即使程序段中只有8页被顺序访问。你可以在MySQL配置文件中设置此参数的值，或者使用SET GLOBAL需要该SUPER权限的命令动态更改该参数。
在没有该变量之前，当访问到extent的最后一个page的时候，Innodb会决定是否将下一个extent放入到buffer pool中。

### 随机预读（randomread-ahead）

随机预读方式则是表示当同一个extent中的一些page在buffer pool中发现时，Innodb会将该extent中的剩余page一并读到buffer pool中，由于随机预读方式给Innodb code带来了一些不必要的复杂性，同时在性能也存在不稳定性，在5.5中已经将这种预读方式废弃。要启用此功能，请将配置变量设置innodb_random_read_ahead为ON。

> **簇是由64个连续的页组成的，每个页大小为16KB，即每个簇的大小为1MB**。一个簇是物理上**连续分配**的一个段空间。