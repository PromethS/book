**目录**

[toc]
---
## JVM内存结构
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590157547348.png)
### 线程共享
-  **堆（Heap）**：内存占用最大的区域，存放了对象实例及数组，几乎所有对象实例都在这里分配内存，存放new生成的对象和数组。Java堆是垃圾收集器管理的内存区域，也称GC堆。堆最大内存默认是物理内存的1/4;为了避免频繁的调整空间大小，一般把最小与最大设置成一样。
-  **方法区**：用于存储已被虚拟机加载的**类信息、常量、静态变量**、即时编译后的代码等数据。运行时常量池包含字面量与符号引用
    -  符号引用：类的全限定名、字段名和属性、方法名和属性
    -  字面量：文本字符串、final的常量值、基本数据类型的值。
    -  在JDK6及以前，字符串常量池也在方法区内，后面转移到了堆中。在JDK1.8把方法区移除了，改为**元空间**，元空间使用的是直接内存。
### 线程私有
- **虚拟机栈**：虚拟机栈是Java执行方法的内存模型。每个方法被执行的时候，都会创建一个栈帧，把栈帧压人栈，当方法正常返回或者抛出未捕获的异常时，栈帧就会出栈。
  - **栈帧**：栈帧存储方法的相关信息，包含局部变量数表、返回值、操作数栈、动态链接a、局部变量表：包含了方法执行过程中的所有变量。局部变量数组所需要的空间在编译期间完成分配，在方法运行期间不会改变局部变量数组的大小。
    - 返回值：如果有返回值的话，压入调用者栈帧中的操作数栈中，并且把PC的值指向方法调用指令 后面的一条指令地址。
    - **操作数栈**：操作变量的内存模型。操作数栈的最大深度在编译的时候已经确定。
    - 动态链接：每个栈帧都持有在运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态链接。
-  **本地方法栈**：调用本地native的内存模型。
-  **程序计数器**：用于存储执行字节码的行号指示器。占用内存最小的区域，且不存在内存泄漏风险的区域。
## JMM（内存模型）
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590157566630.png)

- **1)** Java的并发采用“**共享内存**”模型，线程之间通过读写内存的公共状态进行通讯。多个线程之间是不能通过直接传递数据交互的，它们之间交互只能通过共享变量实现。
- **2)** Java内存模型规定所有变量都存储在主内存中，每个线程还有自己的**工作内存**。
    - 线程的工作内存中保存了被该线程使用到的**变量的拷贝**（从主内存中拷贝过来），线程对变量的所有操作都必须在工作内存中执行，而不能直接访问主内存中的变量。
    - 不同线程之间**无法直接访问**对方工作内存的变量，线程间变量值的传递都要通过主内存来完成。
    - **主内存主要对应Java堆中实例数据部分。工作内存对应于虚拟机栈中部分区域**。
- **3)** Java线程之间的通信由内存模型JMM（Java Memory Model）控制。
    - JMM决定一个线程对变量的写入何时对另一个线程可见。
    - 线程之间共享变量存储在主内存中
    - 每个线程有一个私有的本地内存，里面存储了读/写共享变量的副本。
    - JMM通过控制每个线程的本地内存之间的交互，来为程序员提供内存可见性保证。
- **4)** **可见性、有序性**
    - 当一个共享变量在多个本地内存中有副本时，如果一个本地内存修改了该变量的副本，其他变量应该能够看到修改后的值，此为**可见性**。
    - 保证线程的**有序执行**，这个为有序性。
- **5)** 内存间指令交互
    - **lock**（锁定）：作用于主内存的变量，把一个变量标识为一条线程独占状态。
    - **unlock**（解锁）：作用于主内存的变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
    - **read**（读取）：作用于主内存变量，把主内存的一个变量读取到工作内存中
    - **load**（载入）：作用于工作内存，把read操作读取到工作内存的变量载入到工作内存的变量副本中
    - **use**（使用）：作用于工作内存的变量，把工作内存中的变量值传递给一个执行引擎
    - **assign**（赋值）：作用于工作内存的变量。把执行引擎接收到的值赋值给工作内存的变量
    - **store**（存储）：把工作内存的变量的值传递给主内存
    - **write**（写入）：把store操作的值入到主内存的变量中
    - 注意：
      - 不允许read、load、store、write操作之一单独出现
        - 不允许一个线程丢弃assgin操作
        - 不允许一个线程不经过assgin操作，就把工作内存中的值同步到主内存中
        - 一个新的变量只能在主内存中生成
        - 一个变量同一时刻只允许一条线程对其**进行lock操作**。但lock操作可以被同一条线程执行多次，只有执行相同次数的unlock操作，变量才会解锁
        - 如果对一个变量进行lock操作，将会**清空**工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或者assgin操作初始化变量的值
        - 如果一个变量没有被锁定，不允许对其执行unlock操作，也不允许unlock一个被其他线程锁定的变量
        - 对一个变量执行unlock操作之前，需要将该变量同步回主内存中
## 堆(内存划分)
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590157579512.png)

Java堆的内存划分如图所示，分别为年轻代、Old Memory（老年代）、Perm（永久代）。其中在Jdk1.8中，永久代被移除，使用MetaSpace代替。

### 新生代
- 使用**复制清除算法**，原因是年轻代每次GC都要**回收大部分对象**。新生代里面分成一份较大的Eden空间和两份较小的Survivor空间。每次只使用Eden和其中一块Survivor空间，然后垃圾回收的时候，把存活对象放到未使用的Survivor（划分出from、to）空间中，清空Eden和刚才使用过的Survivor空间
- 分为**Eden**、**Survivor From**、**Survivor To**，比例默认为8：1：1
- 内存不足时发生**Minor GC**
### 老年代
- 采用**标记-整理算法**（mark-compact），原因是老年代每次GC只会回收少部分对象。与新生代默认比例是2：1
### 永久代、方法区（元空间）
- Perm的废除：在jdk1.8中，Perm被替换成**MetaSpace**，MetaSpace存放在本地内存中。可以避免永久代内存不够用，或者发生内存泄漏
- 元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：**元空间并不在虚拟机中**，而是使用本地内存
- 堆内存的划分在JVM里面的示意图:
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590157590290.png)
## GC垃圾回收
### 标记算法（可达性分析法）
>引用计数算法存在循环引用问题，一般都不使用
- 通过一系列“**GC Roots**”对象作为起点进行搜索，如果在“GC Roots”和一个对象之间没有可达路径，则称该对象是不可达的。不可达对象不一定会成为可回收对象。进入DEAD状态的线程还可以恢复，GC不会回收它的内存。以下对象会被认为是root对象：
    - 虚拟机栈（栈帧中本地变量表）中引用的对象（**局部变量**）
    - 方法区中静态属性引用的对象（**静态变量**）
    - 方法区中常量引用的对象（**常量**）
    - 本地方法栈中Native方法引用的对象
- 对象被判定可被回收，需要经历两个阶段：
    - 第一个阶段是可达性分析，分析该对象是否可达
    - 第二个阶段是当对象没有重写**finalize**()方法或者finalize()方法已经被调用过，虚拟机认为该对象不可以被救活，因此回收该对象（finalize()方法在垃圾回收中的作用是，给该对象一次救活的机会）
>finalize():（1） GC垃圾回收要回收一个对象的时候，调用该对象的finalize()方法。然后在下一次垃圾回收的时候，才去回收这个对象的内存。（2） 可以在该方法里面，指定一些对象在释放前必须执行的操作
### 垃圾回收算法
- Copying（**复制清除**算法）
    - 思想：将可用内存划分为大小相等的两块，每次只使用其中的一块。当进行垃圾回收的时候了，把其中存活对象全部复制到另外一块中，然后把已使用的内存空间一次清空掉
    - 优缺点：不容易产生内存碎片；可用内存空间少；存活对象多的话，效率低下
- Mark-Sweep（**标记-清除**算法）
    - 思想：标记清除算法分为两个阶段，标记阶段和清除阶段。标记阶段任务是标记出所有需要回收的对象，清除阶段就是清除被标记对象的空间
    - 优缺点：实现简单，容易产生内存碎片
- Mark-Compact（**标记-整理**算法）
    - 思想：先标记存活对象，然后把存活对象向一边移动，然后清理掉端边界以外的内存
    - 优缺点：不容易产生内存碎片；内存利用率高；存活对象多并且分散的时候，移动次数多，效率低下
- **分代收集算法**：（目前大部分JVM的垃圾收集器所采用的算法）
    - 新生代每次垃圾回收都要回收大部分对象，所以新生代采用Copying算法。分成一份较大的Eden空间和两份较小的Survivor空间。每次只使用Eden和其中一块Survivor空间，然后垃圾回收的时候，把存活对象放到未使用的Survivor（划分出from、to）空间中，清空Eden和刚才使用过的Survivor空间
    - 老年代每次只回收少量的对象，因此采用mark-compact算法
    

## GC分类
>为什么GC时要停顿所有线程（STW）：
>
>因为可达性分析是判断GC Root对象到其他对象是否可达，假如分析过程中对象的引用关系在不断变化，分析结果的准确性就无法得到保证。
- **Young GC**

  新生代内存的垃圾收集事件称为 Young GC（又称 **Minor GC**），当 JVM无法为新对象分配在新生代内存空间时总会触发 Young GC。

  比如 Eden 区占满时，新对象分配频率越高，Young GC 的频率就越高。

  Young GC 每次都会引起**全线停顿**（Stop-The-World），暂停所有的应用线程，停顿时间相对老年代 GC 造成的停顿，几乎可以忽略不计。

- **Old GC**（**Major GC**）

  只清理老年代空间的 GC 事件，只有 **CMS** 的并发收集是这个模式（为了尽量减少GC的停顿时间，fgc前先进行ygc，然后再old gc）。

- **Full GC**

  清理整个堆的 GC 事件，包括新生代、老年代、元空间等。

- **Mixed GC**

  清理整个新生代以及部分老年代的 GC，只有 **G1** 有这个模式

### 触发MinorGC(Young GC)

虚拟机在进行minorGC之前会判断老年代**最大的可用连续空间**是否大于**新生代的所有对象总空间**。

1、如果大于的话，直接执行minorGC

2、如果小于，JVM会判断老年代的最大连续内存空间是否大于**历次晋升的大小**，如果小于直接执行FullGC

3、如果大于的话，执行minorGC

```c++
// HotSpot中空间分配检查的代码片段
bool TenuredGeneration::promotion_attempt_is_safe(size_t
max_promotion_in_bytes) const {
   // 老年代最大可用的连续空间
   size_t available = max_contiguous_available();  
   // 每次晋升到老年代的平均大小
   size_t av_promo  = (size_t)gc_stats()->avg_promoted()->padded_average();
   // 老年代可用空间是否大于平均晋升大小，或者老年代可用空间是否大于当此GC时新生代所有对象容量
   bool   res = (available >= av_promo) || (available >=
max_promotion_in_bytes);
  return res;
}
```

### 触发FullGC

- **老年代空间不足**

  如果创建一个大对象，Eden区域当中**放不下这个大对象**，会直接保存在老年代当中，如果老年代空间也不足，就会触发Full GC。为了避免这种情况，最好就是不要创建太大的对象。

- **YGC出现promotion failure**

  promotion failure发生在Young GC, 如果Survivor区当中**存活对象的年龄**达到了设定值，会就将Survivor区当中的对象拷贝到老年代，如果老年代的空间不足，就会发生promotion failure， 接下去就会发生Full GC.

- **统计YGC发生时晋升到老年代的平均总大小大于老年代的空闲空间**

  在发生YGC是会判断，是否安全，这里的安全指的是，当前老年代空间可以容纳YGC晋升的对象的平均大小，如果不安全，就不会执行YGC,转而执行Full GC。

- **显式调用System.gc**


## JVM垃圾回收器
1999 年随 JDK1.3.1 一起来的是串行方式的 Serial GC，它是第一款垃圾回收器；此后，JDK1.4 和 J2SE1.3 相继发布。2002 年 2 月 26 日，J2SE1.4 发布；Parallel GC 和Concurrent Mark Sweep （CMS）GC 跟随 JDK1.4.2 一起发布，并且 Parallel GC 在 JDK6 之后成为 HotSpot 默认 GC。这三个垃圾回收器也是各有千秋，Serial GC 适合最小化地使用内存和并行开销的场景、Parallel GC 适合最大化应用程序**吞吐量**的场景、CMS GC 适合**最小化中断或停顿时间**的场景。

![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590157607026.png)

在 JVM 中，具体实现有 Serial、ParNew、Parallel Scavenge、CMS、Serial Old（MSC）、Parallel Old、G1 等。不同垃圾回收器适合于 不同的内存区域，如果两个垃圾回收器之间存在连线，那么表示两者可以配合使用。

| 名称                      | 作用域                                                       | 算法                                                         | 特性                                                         | 设置                                                         |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Serial                    | Serial GC 作用于新生代，Serial Old GC 作用于老年代垃圾收集   | 二者皆采用了串行回收与 "Stop-the-World"，Serial 使用的是复制算法，而 Serial Old 使用的是标记-压缩算法 | 适用于硬件配置较低的平台上，如：单CPU或较小的内存            | 使用 -XX:+UseSerialGC 手动指定使用 Serial回收器执行内存回收任务 |
| Throughput<br />/Parallel | Parallel 作用于新生代，Parallel Old 作用于老年代             | 并行回收和 "Stop-the-World"，Parallel 使用的是复制算法，Parallel Old 使用的是标记-压缩算法 | JDK1.8的**默认回收器**，**吞吐量优先**，**自适应GC策略**（包括年轻代大小、Eden与幸存区的比例、晋升老年代的对象年龄（默认15），吞吐量和停顿时间（回收时间所占JVM总运行时间的比例）等）。 | 使用 **-XX:+UseParallelGC** 手动指定使用 Parallel 回收器执行内存回收任务 |
| CMS,Concurrent-Mark-Sweep | 老年代垃圾回收器，又称作 Mostly-Concurrent 回收器            | 使用了标记-清除算法，分为**初始标记**( Initial-Mark，Stop-the-World )、**并发标记**( Concurrent-Mark )、**再次标记**( Remark，Stop-the-World )、**并发清除**( Concurrent-Sweep ) | 适用于对**停顿时间**比较敏感，并且有相对较多存活时间较长的对象（老年代较大）的应用程序。经过CMS收集的堆会产生空间碎片，会带来堆内存的浪费 | 使用 **-XX:+UseConcMarkSweepGC** 来手动指定使用 CMS 回收器执行内存回收任务 |
| G1,Garbage First          | 没有采用传统物理隔离的新生代和老年代的布局方式，仅仅以逻辑上划分为新生代和老年代，选择的将 Java 堆区划分为 2048 个大小相同的独立 Region 块 | 使用了标记压缩算法                                           | 基于并行和并发、低延迟以及暂停时间更加可控的区域化分代式服务器类型的垃圾回收器 | 使用 **-XX:UseG1GC** 来手动指定使用 G1 回收器执行内存回收任务 |

>JDK 1.8默认的是 **UseParallelGC**
>ParallelGC 默认的是 Parallel Scavenge（新生代）+ Parallel Old（老年代）
>在JVM中是+XX配置实现的搭配组合：
>
>- UseSerialGC 表示 “Serial” + "Serial Old"组合
>- UseParNewGC 表示 “ParNew” + “Serial Old”
>- UseConcMarkSweepGC 表示 “ParNew” + “CMS”. 组合，“CMS” 是针对旧生代使用最多的
>- UseParallelGC 表示 “Parallel Scavenge” + "Parallel Old"组合
>- UseParallelOldGC 表示 “Parallel Scavenge” + "Parallel Old"组合
>- **在实践中使用UseConcMarkSweepGC 表示 “ParNew” + “CMS” 的组合是经常使用的**

实践中一般这样配置：

==-XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=75.==

### 单线程垃圾回收器
- **Serial**（-XX:+UseSerialGC）

  Serial回收器是最基本的新生代垃圾回收器，是单线程的垃圾回收器。由于垃圾清理时，Serial 回收器不存在线程间的切换，因此特别是在单 CPU 的环境下，它的垃圾清除效率比较高。Serial 新生代回收器采用的是**复制算法**。

- **Serial Old**（-XX:+UseSerialGC）

  Serial Old回收器是 Serial回收器的老生代版本，属于单线程回收器，它使用**标记-整理**算法。

![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158704853.png)

### 多线程垃圾回收器（吞吐量优先）
- **ParNew**（-XX:+UseParNewGC）

  ParNew 回收器是在 Serial 回收器的基础上演化而来的，属于Serial回收器的 多线程版本，同样运行在新生代区域。在实现上，两者共用很多代码。在不同运行环境下，根据 CPU 核数，开启不同的**线程数**，从而达到最优的垃圾回收效果。ParNew 新生代回收器 采用的是**复制算法**。对于那些 Server 模式的应用程序，如果考虑采用 CMS 作为老生代回收器时，ParNew 回收器是一个不错的选择。
  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158751297.png)

- **Parallel Old**（-XX:+UseParallelGC）

  Parallel Old 回收器是 Parallel Scavenge 回收器的 老生代版本，属于多线程回收器，采用 **标记-整理**算法。Parallel Old 回收器和 Parallel Scavenge回收器同样考虑了吞吐量优先 这一指标，非常适合那些注重吞吐量和CPU资源敏感的场合。
  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158763181.png)

### CMS（-XX:+UseConcMarkSweepGC）
CMS（Concurrent Mark Sweep） 回收器是在**最短回收停顿时间**为前提的回收器，属于多线程回收器，采用**标记-清除**算法。
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158775538.png)

- **初始标记**（CMS initial mark）

  初始标记 仅仅是标记 GC Roots 内直接关联的对象。这个阶段**速度很快**，需要**Stop the World**。

- **并发标记**（CMS concurrent mark）

  并发标记进行的是GC Tracing，从GC Roots开始对堆进行**可达性分析**，找出存活对象。

- **重新标记**（CMS remark）

  重新标记阶段为了**修正**并发期间由于用户进行运作导致的标记变动的那一部分对象的标记记录。这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短，也需要**Stop The World**。

- **并发清除**（CMS concurrent sweep）

  并发清除阶段会**清除**垃圾对象。

> 初始标记（CMS initial mark）和重新标记（CMS remark）会导致用户线程卡顿，Stop the World 现象发生。
>
> 在整个过程中，CMS 回收器的内存回收基本上和用户线程并发执行。

CMS回收器的==缺点==：
- CMS回收器对**CPU资源**非常依赖

  CMS 回收器过分依赖于**多线程**环境，默认情况下，开启的线程数为（CPU 的数量 + 3）/ 4，当 CPU 数量少于**4**个时，CMS对用户查询的影响将会很大，因为他们要分出一半的运算能力去执行回收器线程；

- CMS回收器无法清除**浮动垃圾**

  由于CMS回收器清除已标记的垃圾（处于最后一个阶段）时，用户线程还在运行，内存在被回收的同时，也在被分配,因此会有**新的垃圾**产生。但是这部分垃圾**未被标记**，在下一次GC才能清除，因此被成为浮动垃圾。

  当老生代中的内存使用超过一定的比例时，系统将会进行垃圾回收；当剩余内存不能满足程序运行要求时，系统将会出现**Concurrent Mode Failure**，临时采用 **Serial Old**算法进行清除，此时的性能将会降低。

- 垃圾收集结束后残余**大量空间碎片**

  CMS回收器采用的**标记-清除**算法，本身存在垃圾收集结束后残余大量空间碎片的缺点。CMS 配合适当的内存整理策略，在一定程度上可以解决这个问题。

#### CMS的异常情况

有几种CMS并发周期失败的情况：

1. 并发模式失败（Concurrent mode failure）：CMS的目标就是在回收老年代对象的时候不要停止全部应用线程，在并发周期执行期间，**用户的线程依然在运行**，如果这时候如果应用线程向老年代请求分配的空间超过预留的空间（**担保失败**），就会触发concurrent mode failure，然后CMS的并发周期就会被一次Full GC代替——**停止全部应用进行垃圾收集，并进行空间压缩**。如果我们设置了*UseCMSInitiatingOccupancyOnly*和*CMSInitiatingOccupancyFraction*参数，其中*CMSInitiatingOccupancyFraction*的值是70，那预留空间就是老年代的30%。
2. 晋升失败：**新生代做minor gc的时候**，需要CMS的担保机制确认老年代是否有足够的空间容纳要晋升的对象，担保机制发现不够，则报concurrent mode failure，如果担保机制判断是够的，但是实际上由于碎片问题导致无法分配，就会报晋升失败。
3. 永久代空间（或Java8的元空间）耗尽，默认情况下,CMS不会对永久代进行收集，一旦永久代空间耗尽，就会触发Full GC。

解决办法：

1. 想办法**增大老年代的空间**，增加整个堆的大小，或者减少年轻代的大小
2. 以**更高的频率**执行后台的回收线程，即提高CMS并发周期发生的频率。设置*UseCMSInitiatingOccupancyOnly*和*CMSInitiatingOccupancyFraction*参数，调低*CMSInitiatingOccupancyFraction*的值，但是也不能调得太低，太低了会导致过多的无效的并发周期，会导致消耗CPU时间和更多的无效的停顿。通常来讲，这个过程需要几个迭代，但是还是有一定的套路，参见《Java性能权威指南》中给出的建议，摘抄如下：
   对特定的应用程序，该标志的更优值可以根据 GC 日志中 CMS 周期**首次启动失败时的值**得到。具体方法是，在垃圾回收日志中寻找并发模式失效，找到后再反向查找 CMS 周期最近的启动记录，然后**根据日志来计算**这时候的**老年代空间占用值**，然后设置一个比该值更小的值。
3. **增多回收线程的个数**
   CMS默认的垃圾收集线程数是*（CPU个数 + 3）/4*，这个公式的含义是：当CPU个数大于4个的时候，垃圾回收后台线程至少占用25%的CPU资源。举个例子：如果CPU核数是1-4个，那么会有1个CPU用于垃圾收集，如果CPU核数是5-8个，那么就会有2个CPU用于垃圾收集。

针对永久代的调优
如果永久代需要垃圾回收（或元空间扩容），就会触发Full GC。默认情况下，CMS不会处理永久代中的垃圾，可以通过开启*CMSPermGenSweepingEnabled*配置来开启永久代中的垃圾回收，开启后会有一组后台线程针对永久代做收集，需要注意的是，触发永久代进行垃圾收集的指标跟触发老年代进行垃圾收集的指标是独立的，老年代的阈值可以通过*CMSInitiatingPermOccupancyFraction*参数设置，这个参数的默认值是80%。开启对永久代的垃圾收集只是其中的一步，还需要开启另一个参数——*CMSClassUnloadingEnabled*，使得在垃圾收集的时候可以卸载不用的类。

#### CMS的参数

```shell
-Xmx4096M -Xms4096M -Xmn1536M 
-XX:MaxMetaspaceSize=512M -XX:MetaspaceSize=512M 
-XX:+UseConcMarkSweepGC # 启用CMS收集器
-XX:+UseCMSInitiatingOccupancyOnly # 关闭CMS的动态检查机制，只通过预设的阈值来判断是否启动并发收集周期
-XX:CMSInitiatingOccupancyFraction=70 # 老年代空间占用到多少的时候启动并发收集周期，跟UseCMSInitiatingOccupancyOnly一起使用
-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses # 将System.gc()触发的Full GC转换为一次CMS并发收集，并且在这个收集周期中卸载			Perm（Metaspace）区域中不需要的类
-XX:+CMSClassUnloadingEnabled # 在CMS收集周期中，是否卸载类
-XX:+ParallelRefProcEnabled # 是否开启并发引用处理
-XX:+CMSScavengeBeforeRemark # 如果开启这个参数，会在进入重新标记阶段之前强制触发一次minor gc
-XX:+UseCMSCompactAtFullCollection # 在不得不进行full gc（担保机制失败）时开启内存碎片的整理过程
-XX:ErrorFile=/home/admin/logs/xelephant/hs_err_pid%p.log 
-Xloggc:/home/admin/logs/xelephant/gc.log 
-XX:HeapDumpPath=/home/admin/logs/xelephant 
-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps 
-XX:+HeapDumpOnOutOfMemoryError
```

> **动态检查机制**：JVM会根据最近的回收历史，**估算**下一次老年代被耗尽的时间，快到这个时间的时候就启动一个并发周期。设置*UseCMSInitiatingOccupancyOnly*这个参数可以将这个特性关闭。

### G1回收器（垃圾区域Region优先）

G1 是JDK 1.7中正式投入使用的用于取代CMS的**压缩回收器**。它虽然没有在物理上隔断新生代 与老生代，但是仍然属于**分代垃圾回收器**。G1 仍然会区分年轻代与老年代，年轻代依然分有 Eden 区与 Survivor 区。

G1首先将堆分为**大小相等的Region**，避免**全区域** 的垃圾回收。然后追踪每个Region垃圾 堆积的价值大小，在后台维护一个**优先列表**，根据**允许的回收时间**优先回收价值最大的 Region。同时G1采用**Remembered Set**来存放Region之间的对象引用 ，其他回收器中的新生代 与老年代之间的对象引用，从而**避免全堆扫描**。G1 的分区示例如下图所示：
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158790315.png)

>这种使用 Region划分内存空间以及有优先级的区域回收方式，保证G1回收器在有限的时间内可以获得尽可能高的回收效率。

G1和CMS运作过程有很多相似之处，整个过程也分为4个步骤：
- **初始标记**（CMS initial mark）

  初始标记仅仅是标记 GC Roots 内**直接关联**的对象。这个阶段速度很快，需要Stop the World。

- **并发标记**（CMS concurrent mark）

  并发标记进行的是GC Tracing，从GC Roots开始对堆进行可达性分析，找出存活对象。

- **重新标记**（CMS remark）

  重新标记阶段为了修正并发期间由于用户进行运作导致的标记变动的那一部分对象的标记记录。这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短，也需要Stop The World。

- **筛选回收**

  首先对各个Region的**回收价值和成本**进行排序，根据用户所期望的**GC停顿时间**来制定回收计划。这个阶段可以与用户程序一起**并发执行**，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高回收效率。
---
与其它GC回收相比，G1 具备如下4个**特点**：
- **并行与并发**

  使用多个CPU来缩短Stop-the-World的停顿时间，部分其他回收器需要停顿Java线程执行的GC 动作，G1回收器仍然可以通过并发的方式让Java 程序继续执行。

- **分代回收**

  与其他回收器一样，分代概念在G1中依然得以保留。虽然G1可以不需要其他回收器配合就能独立管理整个GC堆，但它能够采用**不同的策略**去处理新创建的对象和已经存活 一段时间、熬过多次 GC的旧对象，以获取更好的回收效果。新生代和老年代不再是物理隔离，是多个大小相等的独立 Region。

- **空间整合**

  与CMS的标记—清理算法不同，G1从整体来看是基于**标记—整理**算法实现的回收器。从 局部（两个Region之间）上来看是基于复制算法实现的。但无论如何，这两种算法都意味着G1运作期间不会产生**内存空间碎片**，回收后能提供规整的可用内存。这种特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次GC。

- **可预测的停顿**

  这是G1相对于CMS的另一大优势，降低停顿时间是G1和CMS共同的关注点。G1除了追求低停顿外，还能建立**可预测的停顿时间模型**，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在垃圾回收上的时间不得超过N毫秒。（后台维护的**优先列表**，优先回收价值大的Region）。

#### 与CMS的区别
CMS（并发标记清扫算法）在触发老年代垃圾回收时（有两种情况会触发老年代垃圾回收，一种是年轻代对象往老年代拷贝的时候，发现老年代无法分配一块合适大小的连续内存；另一种是统计多次GC的STW时间，发现GC时间占比较大），通过多线程并发标记可回收的内存块，在回收过程中**不会挪动**被引用的对象。因此GC之后，存活的对象是散落在整个老年代中的。这样随着时间的推移，存活的对象会分布得极为散乱，导致大量内存碎片，以致最后不得不进行一次full GC来达到整理内存碎片的目的。

G1把整个堆划分成很多相同大小的Region（一般设为**32MB**）。无论Young GC，还是Old GC，都选择一部分Region，将其中存活的对象拷贝到若干个全新的Region上，然后释放旧的Region的空间。而所谓G1，就是Garbage First的简写，也就是在选择一批Region做GC时，会优先选择那些垃圾占比最高的Region。

G1类似于CMS，也会分年轻代和老年代。不同的地方在于，G1的年轻代并不像CMS那样分配两个连续的内存块，然后两个内存块交替使用。G1的年轻代是由分散的多个Region组成，而且Region的个数会随着前面多次young GC的STW时间动态调整。若之前的young GC耗时比较长，则G1会调小young区大小，反之则调大，因为更大的young区明显会导致更长的STW耗时。

另外，G1一旦触发老年代垃圾回收，会将待回收的Region分成若干批次，每一批次从young区和old区中按照Garbage First策略选若干个Region进行垃圾回收，每一批垃圾回收都叫做mixed GC。因此G1本质上是将一次大的内存整理过程分摊成多次小的内存整理过程，从而达到控制STW延迟和避免full GC的目的。

综上所述，CMS和G1 GC本质区别如下：

- CMS老年代GC并不会挪动对象，只有在做full GC的时候才会挪动对象，处理碎片问题。所以，理论上使用**CMS无法避免**STW的full GC。而G1可以通过多次mixed GC增量地处理内存碎片，所以，G1有能力完全避免STW的fullGC。
- 由于G1的老年代回收是增量式的，所以**G1更加适合大堆**。

#### 分代模型

##### 分代

分代垃圾收集可以将关注点集中在最近被分配的对象上，而**无需整堆扫描**，避免长命对象的拷贝，同时独立收集有助于降低响应时间。虽然分区使得内存分配不再要求紧凑的内存空间，但G1依然使用了分代的思想。与其他垃圾收集器类似，G1将内存在逻辑上划分为年轻代和老年代，其中年轻代又划分为Eden空间和Survivor空间。但年轻代空间并不是固定不变的，当现有年轻代分区占满时，JVM会分配新的空闲分区加入到年轻代空间。

整个年轻代内存会在初始空间`-XX:G1NewSizePercent`(默认整堆5%)与最大空间`-XX:G1MaxNewSizePercent`(默认60%)之间动态变化，且由参数目标暂停时间`-XX:MaxGCPauseMillis`(默认200ms)、需要扩缩容的大小以及分区的已记忆集合(RSet)计算得到。当然，G1依然可以设置固定的年轻代大小(参数`-XX:NewRatio`、`-Xmn`)，但同时暂停目标将失去意义。

##### 本地分配缓冲

本地分配缓冲 Local allocation buffer (Lab)

值得注意的是，由于分区的思想，每个线程均可以"认领"某个分区用于线程本地的内存分配，而不需要顾及分区是否连续。因此，每个应用线程和GC线程都会独立的使用分区，进而减少同步时间，提升GC效率，这个分区称为本地分配缓冲区(Lab)。

其中，**应用线程可以独占一个本地缓冲区(TLAB)来创建的对象**，而大部分都会落入Eden区域(巨型对象或分配失败除外)，因此**TLAB的分区属于Eden空间**；而每次垃圾收集时，每个GC线程同样可以独占**一个本地缓冲区(GCLAB)**用来转移对象，每次回收会将对象复制到Suvivor空间或老年代空间；对于从Eden/Survivor空间晋升(Promotion)到Survivor/老年代空间的对象，同样有GC独占的本地缓冲区进行操作，该部分称为**晋升本地缓冲区**(PLAB)。

#### 分区

### 分区

G1采用了分区(Region)的思路，将整个堆空间分成若干个大小相等的内存区域，每次分配对象空间将逐段地使用内存。因此，在堆的使用上，G1并不要求对象的存储一定是物理上连续的，只要逻辑上连续即可；每个分区也不会确定地为某个代服务，可以按需在年轻代和老年代之间切换。启动时可以通过参数`-XX:G1HeapRegionSize=n`可指定分区大小(1MB~32MB，且必须是2的幂)，默认将整堆划分为**2048个分区**。

##### 卡片

在每个分区内部又被分成了若干个大小为**512 Byte卡片**(Card)，标识堆内存最小可用粒度所有分区的卡片将会记录在全**局卡片**表(Global Card Table)中，分配的对象会占用物理上连续的若干个卡片，当查找对分区内对象的引用时便可通过记录**卡片来查找该引用对象**(见RSet)。每次对内存的回收，都是对指定分区的卡片进行处理。

##### 已记忆集合

**在串行和并行收集器中，GC通过整堆扫描**，来确定对象是否处于可达路径中。然而G1为了避免STW式的整堆扫描，在每个分区记录了一个**已记忆集合**(RSet)，内部**类似一个反向指针**，**记录引用分区内对象**的卡片索引。当要回收该分区时，通过扫描分区的RSet，来确定引用本分区内的对象是否存活，进而确定本分区内的对象存活情况。

事实上，并非所有的引用都需要记录在RSet中，如果一个分区确定需要扫描，那么无需RSet也可以无遗漏的得到引用关系。那么**引用源自本分区**的对象，当然不用落入RSet中；同时，G1 GC每次都会对年轻代进行整体收集，因此引用源自年轻代的对象，也不需要在RSet中记录。最后只有老年代的分区可能会有RSet记录，这些分区称为拥有RSet分区(an RSet’s owning region)。

##### 收集集合 CSet

收集集合(CSet)代表每次GC暂停时回收的**一系列目标分区**。在任意一次收集暂停中，**CSet所有分区都会被释放**，内部存活的对象都会被转移到分配的空闲分区中。因此无论是年轻代收集，还是混合收集，工作的机制都是一致的。年轻代收集CSet只容纳年轻代分区，而混合收集会通过启发式算法，在老年代候选回收分区中，筛选出回收收益最高的分区添加到CSet中。

候选老年代分区的CSet准入条件，可以通过活跃度阈值`-XX:G1MixedGCLiveThresholdPercent`(默认85%)进行设置，从而拦截那些回收开销巨大的对象；同时，每次混合收集可以包含候选老年代分区，可根据CSet对堆的总大小占比`-XX:G1OldCSetRegionThresholdPercent`(默认10%)设置数量上限。

#### 参数配置

```shell
-XX:+UseG1GC -Xmx32g -XX:MaxGCPauseMillis=200 -XX:+PrintGCDetails
```

前面2个参数都好理解，后面这个MaxGCPauseMillis参数该怎么配置呢？这个参数从字面的意思上看，就是**允许的GC最大的暂停时间**。G1尽量确保每次GC暂停的时间都在设置的MaxGCPauseMillis范围内。 那G1是如何做到最大暂停时间的呢？这涉及到另一个概念，CSet(collection set)。它的意思是在一次垃圾收集器中被收集的区域集合。

- Young GC：选定所有新生代里的region。通过控制新生代的region个数来控制young GC的开销。

- Mixed GC：选定所有新生代里的region，外加根据global concurrent marking统计得出收集收益高的若干老年代region。在用户指定的开销目标范围内尽可能选择收益高的老年代region。

在理解了这些后，我们再设置最大暂停时间就好办了。 首先，我们能容忍的最大暂停时间是有一个限度的，我们需要在这个限度范围内设置。但是应该设置的值是多少呢？我们需要在吞吐量跟MaxGCPauseMillis之间做一个平衡。如果MaxGCPauseMillis设置的过小，那么GC就会频繁，吞吐量就会下降。如果MaxGCPauseMillis设置的过大，应用程序暂停时间就会变长。G1的**默认暂停时间是200毫秒**，我们可以从这里入手，调整合适的时间。

其他调优参数

```shell
-XX:G1HeapRegionSize=n
```

设置的 G1 分区的大小。值是 2 的幂，范围是 1 MB 到 32 MB 之间。目标是根据最小的 Java 堆大小划分出约 2048 个分区。

```shell
-XX:ParallelGCThreads=n
```

设置 STW 工作线程数的值。将 n 的值设置为逻辑处理器的数量。n 的值与逻辑处理器的数量相同，最多为 8。

如果逻辑处理器不止八个，则将 n 的值设置为逻辑处理器数的 **5/8 左右**。这适用于大多数情况，除非是较大的 SPARC 系统，其中 n 的值可以是逻辑处理器数的 5/16 左右。

```shell
-XX:ConcGCThreads=n
```

设置并行标记的线程数。将 n 设置为并行垃圾回收线程数 (ParallelGCThreads) 的 1/4 左右。

```shell
-XX:InitiatingHeapOccupancyPercent=45
```

设置触发标记周期的 Java 堆占用率阈值。默认占用率是整个 Java 堆的 45%。

避免使用以下参数：

**避免使用 -Xmn 选项或 -XX:NewRatio 等其他相关选项显式设置年轻代大小**。**固定年轻代的大小会覆盖暂停时间目标**。

**触发Full GC**

在某些情况下，G1触发了Full GC，**这时G1会退化使用Serial收集器**来完成垃圾的清理工作，它仅仅使用单线程来完成GC工作，GC暂停时间将达到秒级别的。整个应用处于假死状态，不能处理任何请求，我们的程序当然不希望看到这些。那么发生Full GC的情况有哪些呢？

- 并发模式失败

  G1启动标记周期，**但在Mix GC之前，老年代就被填满**，这时候G1会放弃标记周期。这种情形下，需要增加堆大小，或者调整周期（例如增加线程数-XX:ConcGCThreads等）。

- 晋升失败或者疏散失败

  G1在进行GC的时候**没有足够的内存**供存活对象或晋升对象使用，由此触发了Full GC。可以在日志中看到(to-space exhausted)或者（to-space overflow）。

**解决这种问题的方式是：**

1. 增加 -XX:G1ReservePercent 选项的值（并相应增加总的堆大小），为“目标空间”增加预留内存量。
2. 通过减少 -XX:InitiatingHeapOccupancyPercent 提前启动标记周期。
3. 也可以通过增加 -XX:ConcGCThreads 选项的值来增加并行标记线程的数目。

**巨型对象分配失败**

当巨型对象找不到合适的空间进行分配时，就会启动Full GC，来释放空间。这种情况下，应该避免分配大量的巨型对象，增加内存或者增大-XX:G1HeapRegionSize，使巨型对象不再是巨型对象。


## GC调优
大多数情况下对 Java 程序进行 GC 调优，主要关注两个目标：
- **响应速度**（Responsiveness）：响应速度指程序或系统对一个请求的响应有多迅速。比如，用户订单查询响应时间，对响应速度要求很高的系统，较大的停顿时间是不可接受的。调优的重点是在短的时间内快速响应。
- **吞吐量**（Throughput）：吞吐量关注在一个特定时间段内应用系统的大工作量。例如每小时批处理系统能完成的任务数量，在吞吐量方面优化的系统，较长的GC停顿时间也是可以接受的，因为高吞吐量应用更关心的是如何尽可能快地完成整个任务，不考虑快速响应用户请求。
***
- **1）** 一般来说，当survivor区不够大或者占用量达到**50%**，就会把一些对象放到老年区。通过设置合理的eden区，survivor区及使用率，可以将年轻对象保存在年轻代，从而避免full GC，使用-Xmn设置年轻代的大小

  > 一般刚上线后先设置的大一点，观察ygc与fgc（可以通过日志分析，也可以通过jstat），一般老年代设置为fullgc后的四倍以上，幸存区设置为ygc后两倍以上。如资源比较充足可以适当设置大一点，减少gc的次数，但还要关注gc的耗时

- **2）** 设置最小堆和最大堆。**-Xmx和-Xms**大小一致可以减少GC次数，但是增加每次GC的时间，因为每次GC要把堆的大小维持在一个区间内。
- **3）** 对于占用内存比较多的**大对象**，一般会选择在老年代分配内存。如果在年轻代给大对象分配内存，年轻代内存不够了，就要在eden区移动大量对象到老年代，然后这些移动的对象可能很快消亡，因此导致full GC。通过设置参数：-XX:PetenureSizeThreshold=1000000，单位为B，标明对象大小超过1M时，在老年代(tenured)分配内存空间。
- **4）** 一般情况下，年轻对象放在eden区，当第一次GC后，如果对象还存活，放到survivor区，此后，每GC一次，年龄增加1，当对象的年龄达到阈值（默认15），就被放到老年区。这个阈值可以**通过-XX:MaxTenuringThreshold**:设置。如果想让对象留在年轻代，可以设置比较大的阈值。
- **5）** 通过增**大吞吐量**提高系统性能，可以通过设置并行垃圾回收收集器。（1）**-XX:+UseParallelGC**:年轻代使用并行垃圾回收收集器。这是一个关注吞吐量的收集器，可以尽可能的减少垃圾回收时间。（2）**-XX:+UseParallelOldGC**:设置老年代使用并行垃圾回收收集器。
- **6）** 使用非占用的垃圾收集器。**-XX:+UseConcMarkSweepGC**老年代使用CMS收集器降低停顿。
- **7）** 设置survivor与eden区的比例，可以避免大对象到老年代。**-XXSurvivorRatio**=3，表示年轻代中的分配比率：survivor:eden = 2:3
- **8）** JVM性能调优的工具：
    - **jstat**
    JVM自带命令行工具，可用于统计内存分配速率、GC 次数，GC 耗时。
        - **jstat -gc** <pid><统计间隔时间><统计次数>;例如：jstat -gc 32683 1000 10，统计 pid=32683 的进程，每秒统计 1 次，统计 10 次。
          ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158806570.png)
    - **jmap**
    JVM自带命令行工具，可用于了解系统运行时的对象分布。
        - **jmap -heap** <pid>; 显示Java堆详细信息
        - **jmap -histo** <pid>; 生成堆内存转储快照，在当前目录下导出dump.hrpof的二进制文件
        - **jmap -dump:live**,format=b, file=dump.hprof <pid>; 可以用eclipse的MAT图形化工具分析
    - **jinfo**
        - jinfo <pid>; 用来查看正在运行的 Java 应用程序的扩展参数，包括 Java System 属性和 JVM 命令行参数。
    - 其他监控工具
        - 监控告警系统：**Zabbix**、Prometheus、Open-Falcon
        - jdk 自动实时内存监控工具：**VisualVM**
        - GC 日志分析：GCViewer、**gceasy**
        - GC 参数检查和优化：http://xxfox.perfma.com/


## 内存溢出
### 分类
- **堆溢出**

  这种场景最为常见，报错信息：`java.lang.OutOfMemoryError: Java heap space`
  
    - **原因**
        - 1、代码中可能存在大对象分配；
        - 2、可能存在内存泄露，导致在多次GC之后，还是无法找到一块足够大的内存容纳当前对象。
    - **解决方法**
        - 1、检查是否存在大对象的分配，最有可能的是大数组分配
        - 2、通过jmap命令，把堆内存dump下来，使用mat工具分析一下，检查是否存在内存泄露的问题
        - 3、如果没有找到明显的内存泄露，使用 -Xmx 加大堆内存
        - 4、还有一点容易被忽略，检查是否有大量的自定义的 Finalizable对象，也有可能是框架内部提供的，考虑其存在的必要性
  
- **GC overhead limit exceeded**

  这个异常比较的罕见，报错信息：`java.lang.OutOfMemoryError：GC overhead limit exceeded`
  
    - **原因**
      
      这个是JDK6新加的错误类型，一般都是堆太小导致的。Sun官方对此的定义：超过98%的时间用来做GC并且回收了不到2%的堆内存时会抛出此异常。
      
    - **解决方法**
      
        - 1、检查项目中是否有大量的死循环或有使用大内存的代码，优化代码。
        - 2、添加参数 -XX:-UseGCOverheadLimit禁用这个检查，其实这个参数解决不了内存问题，只是把错误的信息延后，最终出现 `java.lang.OutOfMemoryError: Java heap space`。
        - 3、dump内存，检查是否存在内存泄露，如果没有，加大内存。
  
- **方法栈溢出**

  报错信息：java.lang.OutOfMemoryError : unable to create new native Thread
  
    - **原因**
      
      出现这种异常，基本上都是创建的了大量的线程导致的，以前碰到过一次，通过jstack出来一共8000多个线程。
      
    - **解决方法**
      
        - 1、通过 -Xss 降低的每个线程栈大小的容量
        - 2、线程总数也受到系统空闲内存和操作系统的限制，检查是否该系统下有此限制：
            - /proc/sys/kernel/pid_max
            - /proc/sys/kernel/thread-max
            - maxuserprocess（ulimit -u）
            - /proc/sys/vm/maxmapcount
            
### **排查方法**
- 1.使用ps -aux|grep java找出对应的进程
- 2.使用jstat -gcutil pid查看gc情况（--可不做），使用jmap -heap pid查看内存使用情况
- 3.使用jmap -histo:live pid | more，查看对象统计信息
- 4.使用jmap -dump:live,format=b,file=/data/dump.hprof pid，把堆空间dump下来
- 5.使用mat分析dump下的内存，常用histogram视图（柱状图），再使用List objects或 Merge Shortest Paths to GC roots 等功能继续钻取数据；Dominator Tree（支配树）视图；

### **MAT使用**
Eclipse Memory Analyzer Tool（MAT）是一个强大的基于Eclipse的内存分析工具，可以帮助我们找到内存泄露，减少内存消耗。
#### Overview
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158854260.png)

#### Histogram视图
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158866501.png)
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158875506.png)

该视图以Class类的维度展示每个Class类的实例存在的个数、占用的[Shallow内存]和[Retained内存]大小，可以分别排序显示。

从Histogram视图可以看出，哪个Class类的对象实例数量比较多，以及占用的内存比较大，不过，多数情况下，在Histogram视图看到实例对象数量比较多的类都是一些**基础类型**，如char[]（因为其构成了String）、String、byte[]，所以仅从这些是无法判断出具体导致内存泄露的类或者方法的，可以使用 **List objects** 或 **Merge Shortest Paths to GC roots** 等功能继续钻取数据。如果Histogram视图展示的数量多的实例对象不是基础类型，是有嫌疑的某个类，如项目代码中的bean类型，那么就要重点关注了。

#### Dominator Tree（支配树）视图
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158889198.png)
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158898516.png)

该视图以实例对象的维度展示当前堆内存中Retained Heap占用**最大的对象**，以及依赖这些对象存活的对象的树状结构

视图中展示了实例对象名、Shallow Heap大小、Retained Heap大小、以及当前对象的Retained Heap在整个堆中的占比

点开Dominator Tree实例对象左侧的“+”，会展示出下一层（next level），当所有引用了当前实例对象的引用都被清除后，下一层列出的objects就会被垃圾回收

这也阐明了“支配”的含义：父节点的回收会导致子节点也被回收，即因为父节点的存在使得子节点存活

Dominator Tree支配树可以很方便的找出占用**Retained Heap**内存最多的几个对象，并表示出某些objects的是因为哪些objects的原因而存活


### **遇到的故障**
- 1.使用redisson的localCacheMap（本次缓存），使用RedissonClient获取map对象，底层实现是每次都new一个对象，由于研发并不清楚底层实现，在网关每次接口访问时获取对象，然后set。由于该对象与redis事件订阅相关，并不能被GC回收，如果频繁创建且不释放则会导致内存泄漏。
- 2.生成用户头像时，需要加载字体，使用Font加载字体文件，由于字体文件较大，且未使用单例，导致内存泄漏，在测试环境压测时出现的。通过MAT分析发现大量的TrueTypeFont对象
- 3.数据导出时，由于多次DB操作，使用低级的循环DB操作，导出数据量较大，且查出所有字段。导致事务执行时间很长，且大量字符串占用内存
- 4.hashmap的死循环，且JDK1.8已经规避了这个问题
- 5.由于和外部系统是紧耦合，并未使用连接池及超时时间，导致服务器连接及线程被打满，造成一种假死状态

## JIT（just in time）
**即时编译器**。静态编译是把代码转换为**字节码**，而JIT是把字节码翻译成机器可以识别的**机器码**。

当JVM发现某些方法或代码运行特别频繁的时候，会认为这些方法或代码为“**热点代码**”。然后JTI会把部分“热点代码”翻译成机器码，并进行优化，且把翻译后的机器码**缓存**起来。如果这些代码本身或将来只会执行一次，那么就不会翻译成机器码。

**并不是所有的对象都分配到堆上**，因为JIT编译时会进行**逃逸分析**（创建的对象是否会存在外部引用），如果对象**并没有逃逸出方法**内，那么有可能**内存分配被优化到栈上**，但这个**并不是绝对的**。并不是所有的对象都分配到栈上，而是经过算法分析后部分分配到堆上，部分分配到栈上。

> **栈上分配**
>  如果能够确定一个对象不会逃逸到方法之外，可以**在栈上分配对象的内存**，这样对象占用的内存* 空间可以随着栈帧出栈而销毁，减少gc的压力；
>
> **同步消除**
>  如果逃逸分析得出对象不会逃逸到线程之外，那么对象的**同步措施可以消除**。
>
> **标量替换**
>  如果逃逸分析证明一个对象不会被外部访问，并且这个对象可以被拆解，那么程序执行的时候可能不创建这个对象，改为在栈上分配这个方法所用到的对象的成员变量。

HotSpot 采用了**惰性评估**(Lazy Evaluation)的做法，根据二八定律，消耗大部分系统资源的只有那一小部分的代码（热点代码），而这也就是 JIT 所需要编译的部分。JVM 会根据代码每次被执行的情况收集信息并相应地做出一些优化，因此执行的次数越多，它的速度就越快。JDK 9 引入了一种新的编译模式 **AOT**(Ahead of Time Compilation)，它是**直接将字节码编译成机器码**，这样就避免了 JIT 预热等各方面的开销。JDK 支持分层编译和 AOT 协作使用。但是 ，AOT 编译器的编译质量是肯定比不上 JIT 编译器的。

> AOT 编译器从编译质量上来看，肯定比不上 JIT 编译器。其存在的目的在于避免 JIT 编译器的运行时性能消耗或内存消耗，或者避免解释程序的早期性能开销。

> 还有如下的优化：
>
> 无用代码消除（Dead Code Elimination）、循环展开（Loop Unrolling）、循环表达式外提（Loop Expression Hoisting）、消除公共子表达式（Common Subexpression Elimination）、常量传播（Constant Propagation）、基本块冲排序（Basic Block Reordering）等，还会实施一些与Java 语言特性密切相关的优化技术，如范围检查消除（Range Check Elimination）、空值检查消除（Null Check Elimination ，不过并非所有的空值检查消除都是依赖编译器优化的，有一些是在代码运行过程中自动优化 了）等。另外，还可能根据解释器或Client Compiler提供的性能监控信息，进行一些不稳定的激进优化，如守护内联（Guarded Inlining）、分支频率预测（Branch Frequency Prediction）等。

### HotSpot的分层编译模式

HotSpot内置了**C1编译器**和**C2编译器**。默认情况下，JVM采取**解释器**和其中一个**编译器**直接配合的运行模式，编译器的选择，根据自身的版本以及宿主机器的硬件性能自动选择。此外，用户也可以通过JVM参数强制JVM的运行模式。如下表：

| 参数    | 运行模式               | 说明                                                         |
| ------- | ---------------------- | ------------------------------------------------------------ |
| -Xint   | 解释模式               | 编译器不介入工作，所有代码都使用解释器解释执行               |
| -Xcomp  | 编译模式               | 优先采用编译方式执行，但是在编译无法进行的情况下，还是会进行解释执行 |
| -client | 混合模式（Client模式） | 解释器搭配C1的**混合模式**，适用于对于执行时间较短的，或者对启动性能有要求的程序 |
| -server | 混合模式（Server模式） | 解释器搭配C2的**混合模式**，适用于执行时间较长的，或者对峰值性能有要求的程序 |

JVM有两种运行模式**Server**与**Client**。两种模式的区别在于，**Client模式启动速度较快**，**Server模式启动较慢**；**但是启动进入稳定期长期运行之后Server模式的程序运行速度比Client要快很多**。这是因为Server模式启动的JVM采用的是重量级的虚拟机，对程序采用了更多的优化；而Client模式启动的JVM采用的是轻量级的虚拟机。所以Server启动慢，但稳定后速度比Client远远要快。



## JVM对象模型
每一个Java类，在被JVM加载的时候，JVM会给这个类创建一个**instanceKlass**，保存在方法区，用来在JVM层表示该Java类。当我们在Java代码中，使用new创建一个对象的时候，JVM会创建一个**instanceOopDesc**对象，这个对象中包含了对象头以及实例数据。
>对象的实例（instantOopDesc)保存在堆上，对象的元数据（instantKlass）保存在方法区，对象的引用保存在栈上。
### 对象内存布局
以HotSpot虚拟机为例，对象在堆内存的布局分为三个区域，分别是对象头（Header）、实例数据（Instance Data）、对齐填充（Padding）
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158918377.png)

- **对象头**

  对象头包括两部分信息分别是Mark World和元数据指针，Mark World用于存储对象运行时的数据，比如HashCode、锁状态标志、GC分代年龄等。而元数据指针用于指向方法区的中目标类的类型信息，通过元数据指针可以确定对象的具体类型。

  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158929393.png)

- **实例数据**

  用于存储对象中的各种类型的字段信息（包括从父类继承来的）。

- **对齐填充**

  对齐填充不一定存在，起到了占位符的作用，没有特别的含义。

### 对象创建过程
- **1）** **判断对象对应的类是否加载、链接、初始化**

  虚拟机遇到一条new指令时，首先检查这个指令的参数能否在常量池中定位到一个类的符号引用，并检查这个符号引用代表的类是否已经加载、连接和初始化。如果没有，就执行该类的**加载**过程。

- **2）** **为对象分配内存**

  类加载完成后，接着会在Java堆中划分一块内存分配给对象。内存分配根据Java堆是否规整（Java堆的内存是否规整根据所采用的来及收集器是否带有压缩整理功能有关），有两种方式：
  
    - **指针碰撞**：如果Java堆的内存是规整，即所有用过的内存放在一边，而空闲的的放在另一边。分配内存时将位于中间的指针指示器向空闲的内存移动一段与对象大小相等的距离，这样便完成分配内存工作。

    - **空闲列表**：如果Java堆的内存不是规整的，则需要由虚拟机维护一个列表来记录那些内存是可用的，这样在分配的时候可以从列表中查询到足够大的内存分配给对象，并在分配后更新列表记录。
  
    - **处理并发安全问题**
  
      创建对象是一个非常频繁的操作，所以需要解决并发的问题，有两种方式：
      
        - 对分配内存空间的动作进行同步处理，比如在虚拟机采用**CAS算法**并配上失败重试的方式保证更新操作的原子性。
        - 每个线程在Java堆中预先分配一小块内存，这块内存称为本地线程分配缓冲（Thread Local Allocation Buffer）简写为**TLAB**，线程需要分配内存时，就在对应线程的TLAB上分配内存，当线程中的TLAB用完并且被分配到了新的TLAB时，这时候才需要同步锁定。通过-XX:+/-UserTLAB参数来设定虚拟机是否使用TLAB。
  
- **3）** **初始化零值**

  将分配到的内存，除了对象头都初始化为零值。

- **4）** **设置对象的对象头**

  将对象的所属类、对象的HashCode和对象的GC分代年龄等数据存储在对象的对象头中。

- **5）** **执行init方法进行初始化**

  执行init方法，初始化对象的成员变量、调用类的构造方法，这样一个对象就被创建了出来。

### 对象的定位访问方式
- **句柄**：使用句柄的方式，Java堆中将会划分出一块内存作为作为句柄池，引用中存储的就是对象的句柄的地址。

- **直接指针**：使用直接指针的方式，引用中存储的就是对象的地址。Java堆对象的布局必须必须考虑如何去访问对象类型数据。

  两种方式各有优点：
> - 使用句柄访问的好处是引用中存放的是稳定的句柄地址，当对象被移动（比如说垃圾回收时移动对象），只会改变句柄中实例数据指针，而引用本身不会被修改。
> - 使用直接指针，节省了一次指针定位的时间开销。

### MESI--CPU缓存一致性协议
在CPU处理器用MESI协议保障缓存数据的一致性。

| 状态         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| M(Modified)  | 这行数据有效，数据被修改了，和内存中的数据不一致，数据只存在于本Cache中。 |
| E(Exclusive) | 这行数据有效，数据和内存中的数据一致，数据只存在于本Cache中。 |
| S(Shared)    | 这行数据有效，数据和内存中的数据一致，数据存在于很多Cache中。 |
| I(Invalid)   | 这行数据无效                                                 |

### 指针压缩

jvm指针压缩指的是：

- java对象的Object Head里的**Klass Pointer**由8字节压缩为4字节，Klass Pointer指向当前对象的Class对象的内存位置
- java对象内部的**复杂类型属性的引用**由8字节压缩为4字节，就是说当前对象持有复杂类型的属性对象的内存位置
- **数组对象的长度**由8字节压缩为4字节，同时数组对象持有的每个元素的内存引用由8字节压缩为4字节

以上机制生效的前提是jvm**在64位机器上**，**32位机器内存上限是4G，内存指针默认是4字节**。

在64位机器上当分片的jvm**内存小于32G时，JVM默认开启指针压缩**，此时指针表示的不是引用对象的实际内存地址，是一个相对的地址。**如何用一个32位的数字表示内存32G内存内的任意地址？**

在大于32G的机器上分配32G内存给jvm应用，首先这32G内存是需要连续的，因此jvm内存的低位相较于机器内存的低位有一个偏移量，计 **offset0**，这个偏移量在通过内存指针计算实际内存地址时是需要考虑进去的。

32G内存有 1024*1024*1024*32 = 2^35个字节，4个字节是32位，最大能表示 2^32-1 ，这中间还差了 2^3 的数量级，该如何圆场？网上说4个字节有40多亿，表示40多亿个对象。但是一个对象至少16字节，2^32 * 16 = 2^36 超过了 2^35 的字节上限了，所以**指针指向的也不是实际的对象地址**。

后来想到了java对象大小是有一个对齐机制的：Object Header + 实例数据 + 对齐填充。**对齐填充保证java对象的大小始终是8字节的倍数**，也就是一个对象的大小都是8字节的倍数，他们有一个公约数8。把 2^35 / 8 = 2^32 刚好等于4个字节所能表示的数值的区间。**把jvm内存按8字节划分为一个一个格子区间**，指针指向的是这个格子区间的序号。

根据指针计算内存的实际地址的公式近视为： **内存地址(字节单位) = jvm内存偏移量 + 指针 * 8**

通过这种巧妙的设计，能够将jvm内对象的地址从8字节压缩为4字节，极大的提高jvm对内存的使用效率。**当jvm超过32G时，这种算法就失效了**，在8字节的指针情况下，需要内存达到40-50G时才能和4字节指针时32G内存具有相同的存储空间。

> 可以节省20%左右的内存，减少GC时间，增加了空间利用率

## 类加载机制

### 类的生命周期
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158947348.png)

类从被加载到内存中开始，到卸载出内存，经历了**加载**、**连接**、**初始化**、**使用**四个阶段，其中连接又包含了**验证**、**准备**、**解析**三个步骤。这些步骤总体上是按照图中顺序进行的，但是Java语言本身支持运行时绑定，所以**解析阶段也可以是在初始化之后**进行的。以上顺序都只是说开始的顺序，实际过程中是交叉进行的，加载过程中可能就已经开始验证了。

- **加载**

  加载是类加载过程中的一个阶段， 这个阶段会在内存（方法区）中生成一个代表这个类的 java.lang.Class **对象**， 作为方法区这个类的各种数据的入口。注意这里不一定非得要从一个Class文件获取，这里既可以从 ZIP 包中读取（比如从jar包和war包中读取），也可以在运行时计算生成（动态代理），也可以由其它文件生成（比如将 JSP 文件转换成对应的 Class 类）。

- **验证**

  这一阶段的主要目的是为了确保Class文件的字节流中包含的信息是否符合当前虚拟机的**要求**，并且不会危害虚拟机自身的安全。

- **准备**

  准备阶段是正式为类变量分配内存并设置类变量的**初始值**阶段，即在方法区中分配这些变量所使用的内存空间。

- **解析**

  解析阶段是指虚拟机将常量池中的符号引用替换为**直接引用**的过程。

- **初始化**

  初始化阶段是类加载最后一个阶段，前面的类加载阶段之后，除了在加载阶段可以自定义类加载
  器以外，其它操作都由 JVM 主导。到了初始阶段，才开始真正执行类中定义的 Java 程序代码。

### 类加载的时机
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158960609.png)
> 其中情况1中的4条字节码指令在Java里最常见的场景是：
>
> 1 . new一个对象时
>
> 2 . set或者get一个类的静态字段（除去那种被final修饰放入常量池的静态字段）
>
> 3 . 调用一个类的静态方法

### 类加载器
![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158971869.png)
- **启动类加载器**(Bootstrap ClassLoader)

  负责加载 **JAVA_HOME\lib** 目录中的， 或通过-Xbootclasspath 参数指定路径中的， 且被虚拟机认可（按文件名识别， 如 **rt.jar**） 的类

- **扩展类加载器**(Extension ClassLoader)

  负责加载 **JAVA_HOME\lib\ext** 目录中的，或通过 java.ext.dirs 系统变量指定路径中的类库。

- **应用程序类加载器**(Application ClassLoader)

  负责加载用户路径（classpath）上的类库。

- **自定义类加载器**

  通过继承 java.lang.ClassLoader实现自定义的类加载器，必须重写**findClass**方法。
  ![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590158982070.png)

### 双亲委派模型
当一个类收到了类加载请求，他首先不会尝试自己去加载这个类，而是把这个请求**委派给父类**去完成，每一个层次类加载器都是如此，因此所有的加载请求都应该传送到启动类加载其中，只有当父类加载器反馈自己无法完成这个请求的时候（在它的加载路径下没有找到所需加载的Class）， 子类加载器才会**尝试自己去加载**。

采用双亲委派的一个好处是比如加载位于 rt.jar 包中的类 java.lang.Object，不管是哪个加载器加载这个类，最终都是委托给顶层的**启动类加载器**进行加载，这样就保证了使用不同的类加载器最终得到的都是同样一个 Object 对象。

- **为什么要使用双亲委派模型**
    - 通过这样的层级关系，可以避免类的**重复加载**，当父类加载器已经加载过类了，那么就没必要子类加载器再次加载；
    - **安全性**比较高。比如自定义类加载器去加载Java的核心API（如String），那么通过双亲委派模式则会传递到顶层类加载器去加载，防止类被篡改。JVM在判断一个对象是否属于某个类时，如果对象的类加载器与待比较类的类加载器不同，那么就认为为false，可以防止类被**篡改**。
- **破坏双亲委派模型**
    - 重写 **loadClass()** 方法
    - 逆向使用类加载器，引入**线程上下文类加载器**,例如Tomcat的应用加载器，JNDI，JDBC，JCE，JAXB，JBI等
    - 追求程序的动态性：代码热替换、模块热部署等技术
- **tomcat如何破坏双亲委派模型**

![image](https://520li.oss-cn-hangzhou.aliyuncs.com/img/clipboard-1590159001953.png)

使用 Thread.getContextClassLoader() - 当前线程的上下文加载器，该加载器可通过 Thread.setContextClassLoader() 在代码运行时动态设置。

默认情况下，**Thread 上下文加载器继承自父线程**，也就是说所有线程默认上下文加载器都与第一个启动的线程相同，也就是 main 线程，它的上下文加载器是 AppClassLoader。

Tomcat 就是在StandardContext启动时首先初始化一个WebappClassLoader然后设置为当前线程的上下文加载器，最后将其封装为Loader对象，借助容器之间的父子关系，在加载Servlet类时使用。

### 引用类型
- **强引用**（StrongReference）

  `Object 0bj = new Object();`根据JVM的垃圾回收策略来回收

- **软引用**（SoftReference）

  `new SoftReference(new Object());`当内存不足时就会回收该对象

- **弱引用**（WeakReference）

  `new WeakReference<>(new Object());`当垃圾回收器扫描到时就会回收对象，也就是存活到下次垃圾回收时

- **虚引用** （PhantomReference）

  `new PhantomReference<Object>(obj,refQueue);`在任何时候都可能被回收;
  使用场景：可以用来跟踪对象被垃圾回收的活动。虚引用可以用来回收堆外内容，通过回收虚引用的对象达到回收堆外内存的目的。


## JVM参数速查表（1.8）-附件
### GC信息打印
- -verbose:gc
开启输出JVM GC日志
- -verbose:class
查看类加载信息明细
- **-XX:+PrintGCDetails**
GC日志打印详细信息
- **-XX:+PrintGCDateStamps**
GC日志打印时间戳信息
- -XX:+PrintHeapAtGC
在GC前后打印GC日志
- -XX:+PrintGCApplicationStoppedTime
打印应用暂停时间
- -XX:+PrintGCApplicationConcurrentTime
打印每次垃圾回收前，程序未中断的执行时间
- **-Xloggc:./gc.log**
指定GC日志目录何文件名
- **-XX:+HeapDumpOnOutOfMemoryError**
当发生 OOM(OutOfMemory)时，自动转储堆内存快照，缺省情况未指定目录时，JVM 会创建一个名称为 java_pidPID.hprof 的堆 dump 文件在 JVM 的工作目录下
- **-XX:HeapDumpPath**=/data/log/gc/dump/
指定OOM时堆内存转储快照位置
- -XX:+PrintClassHistogramBeforeFullGC、-XX:+PrintClassHistogramAfterFullGC
Full GC前后打印跟踪类视图
- -XX:+PrintTenuringDistribution
打印Young GC各个年龄段的对象分布
- -XX:+PrintTLAB
打印TLAB(线程本地分配缓存区)空间使用情况

### CMS/G1通用内存区域设置
- **-Xmx**1024M
JVM最大堆内存大小
- **-Xms**1024M
JVM初始内存大小，建议与-Xmx一致
- **-Xmn**1536M
年轻代空间大小，使用G1收集器是不建议设置该值
- **-Xss**1M
每个线程的堆栈大小
- -XX:MaxMetaspaceSize=512M
最大元空间大小
- -XX:MetaspaceSize=512M
初始元空间大小
- **-XX:SurvivorRatio**=8
年轻代中Eden区与Survivor区的大小比值，缺省默认值为8
- -XX:MaxDirectMemorySize=40M
最大堆外内存大小

### CMS/G1通用阈值设置
- -XX:MaxTenuringThreshold=15
设置新生代需要经历多少次GC晋升到老年代中的最大阈值，缺省默认值为15
- -XX:PretenureSizeThreshold=1M
代表分配在新生代一个对象占用内存最大值，超过该最大值对象直接在old区分配，默认值缺省是0，代表对象不管多大都是先在Eden中分配内存

### CMS/G1通用开关设置
- -XX:+DisableExplicitGC
设置忽略System.gc()的调用，不建议设置该参数，因为部分依赖Java NIO的框架(例如Netty)在内存异常耗尽时，会主动调用System.gc()，触发Full GC，回收DirectByteBuffer对象，作为回收堆外内存的最后保障机制，设置该参数之后会导致在该情况下堆外内存得不到清理 参考：为什么不推荐使用-XX:+DisableExplicitGC
- -XX:+ParallelRefProcEnabled
开启尽可能并行处理Reference对象，建议开启

### CMS/G1通用线程数设置
- -XX:ParallelGCThreads=10
设置并行收集垃圾器在应用线程STW期间时GC处理线程数
- -XX:ConcGCThreads=10
设置垃圾收集器在与应用线程并发执行标记处理(非STW阶段)时的线程数

### CMS设置
- -XX:+UseConcMarkSweepGC 激活CMS收集器
- -XX:ConcGCThreads 设置CMS线程的数量
- **-XX:+UseCMSInitiatingOccupancyOnly** 只根据老年代使用比例来决定是否进行CMS
- **-XX:CMSInitiatingOccupancyFraction** 设置触发CMS老年代回收的内存使用率占比
- **-XX:+CMSParallelRemarkEnabled** 并行运行最终标记阶段，加快最终标记的速度
- **-XX:+UseCMSCompactAtFullCollection** 每次触发CMS Full GC的时候都整理一次碎片
- -XX:CMSFullGCsBeforeCompaction=* 经过几次CMS Full GC的时候整理一次碎片
- -XX:+CMSClassUnloadingEnabled 让CMS可以收集永久带，默认不会收集
- -XX:+CMSScavengeBeforeRemark 最终标记之前强制进行一个Minor GC