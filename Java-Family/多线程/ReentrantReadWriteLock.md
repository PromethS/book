## 一、读写锁简介

现实中有这样一种场景：对共享资源有读和写的操作，且写操作没有读操作那么频繁。在没有写操作的时候，多个线程同时读一个资源没有任何问题，所以应该允许多个线程同时读取共享资源；但是如果一个线程想去写这些共享资源，就不应该允许其他线程对该资源进行读和写的操作了。

针对这种场景，**JAVA的并发包提供了读写锁ReentrantReadWriteLock，它表示两个锁，一个是读操作相关的锁，称为共享锁；一个是写相关的锁，称为排他锁**，描述如下：

线程进入读锁的前提条件：

- 没有其他线程的写锁，

- 没有写请求或者**有写请求，但调用线程和持有锁的线程是同一个。**

线程进入写锁的前提条件：

- 没有其他线程的读锁

- 没有其他线程的写锁

而读写锁有以下三个重要的特性：

- （1）公平选择性：支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平。
- （2）重进入：读锁和写锁都支持线程重进入。
- （3）锁降级：遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级成为读锁。

### 锁升级、锁降级

锁降级是支持的，但是**锁升级是不支持的**。

```java
public class ReentrantReadWriteLockTest {

    
    public static void main(String[] args) throws InterruptedException {
        // testReenter();
        // testUpgrade();
         testDowngrade();
    }

    /**
     * 在同一个线程中，在没有释放写锁的情况下，就去申请读锁，这属于锁降级，ReentrantReadWriteLock是支持的，不过锁的释放有问题，获取到锁就要释放锁
     * 为什么有锁的降级，当前线程进行写操作后， 当前线程后面可能还有读的操作，把写锁降级为读锁后，其他的线程可以进行读，但是不能写，提高了效率。
     */
    public static void testDowngrade() {
        ReentrantReadWriteLock rtLock = new ReentrantReadWriteLock();
        rtLock.writeLock().lock();
        System.out.println("writeLock");
        rtLock.readLock().lock();
        System.out.println("get read lock");
    }

    // 在同一个线程中，在没有释放读锁的情况下，就去申请写锁，这属于锁升级，ReentrantReadWriteLock是不支持的。
    public static void testUpgrade() {
        ReentrantReadWriteLock rtLock = new ReentrantReadWriteLock();
        rtLock.readLock().lock();
        System.out.println("get readLock.");
        // rtLock.readLock().unlock();
        rtLock.writeLock().lock();
        System.out.println("blocking");
        rtLock.writeLock().unlock();
    }

    
    //可重入锁，就是说一个线程在获取某个锁后，还可以继续获取该锁，即允许一个线程多次获取同一个锁,要注意的是获取多少次锁就要释放多少次锁，不然会发生死锁，例子讲的是两次获取写锁
    public static void testReenter() throws InterruptedException {
        final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                lock.writeLock().lock();
                System.out.println("Thread real execute");
                lock.writeLock().unlock();
            }
        });

        lock.writeLock().lock();
        lock.writeLock().lock();
        t.start();
        Thread.sleep(200);
        System.out.println("realse one once");
        lock.writeLock().unlock();
        // lock.writeLock().unlock();
    }

}
```

## 二、源码解读

我们先来看下 ReentrantReadWriteLock 类的整体结构：

```java
public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable {

    /** 读锁 */
    private final ReentrantReadWriteLock.ReadLock readerLock;

    /** 写锁 */
    private final ReentrantReadWriteLock.WriteLock writerLock;

    final Sync sync;
    
    /** 使用默认（非公平）的排序属性创建一个新的 ReentrantReadWriteLock */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /** 使用给定的公平策略创建一个新的 ReentrantReadWriteLock */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    /** 返回用于写入操作的锁 */
    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    
    /** 返回用于读取操作的锁 */
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }


    abstract static class Sync extends AbstractQueuedSynchronizer {}

    static final class NonfairSync extends Sync {}

    static final class FairSync extends Sync {}

    public static class ReadLock implements Lock, java.io.Serializable {}

    public static class WriteLock implements Lock, java.io.Serializable {}
}
```

### 1、类的继承关系

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {}
```

说明：可以看到，ReentrantReadWriteLock实现了ReadWriteLock接口，**ReadWriteLock接口定义了获取读锁和写锁的规范**，具体需要实现类去实现；同时其还实现了Serializable接口，表示可以进行**序列化**，在源代码中可以看到ReentrantReadWriteLock实现了自己的序列化逻辑。

### 2、类的内部类

ReentrantReadWriteLock有五个内部类，五个内部类之间也是相互关联的。内部类的关系如下图所示。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200621173421.png)

说明：如上图所示，Sync继承自AQS、NonfairSync继承自Sync类、FairSync继承自Sync类（通过构造函数传入的布尔值决定要构造哪一种Sync实例）；ReadLock实现了Lock接口、WriteLock也实现了Lock接口。

#### Sync类

（1）类的继承关系

```java
abstract static class Sync extends AbstractQueuedSynchronizer {}
```

说明：Sync抽象类继承自AQS抽象类，Sync类提供了对ReentrantReadWriteLock的支持。

（2）类的内部类

Sync类内部存在两个内部类，分别为HoldCounter和ThreadLocalHoldCounter，其中HoldCounter主要与读锁配套使用，其中，HoldCounter源码如下。

```java
// 计数器
static final class HoldCounter {
    // 计数
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    // 获取当前线程的TID属性的值
    final long tid = getThreadId(Thread.currentThread());
}
```

说明：HoldCounter主要有两个属性，count和tid，其中**count表示某个读线程重入的次数**，**tid表示该线程的tid字段的值**，该字段可以用来唯一标识一个线程。ThreadLocalHoldCounter的源码如下

```java
// 本地线程计数器
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    // 重写初始化方法，在没有进行set的情况下，获取的都是该HoldCounter值
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

说明：ThreadLocalHoldCounter重写了ThreadLocal的initialValue方法，ThreadLocal类可以将线程与对象相关联。在没有进行set的情况下，get到的均是initialValue方法里面生成的那个HolderCounter对象。

（3）类的属性

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 版本序列号
    private static final long serialVersionUID = 6317671515068378041L;        
    // 高16位为读锁，低16位为写锁
    static final int SHARED_SHIFT   = 16;
    // 读锁单位
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    // 读锁最大数量
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    // 写锁最大数量
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    // 本地线程计数器
    private transient ThreadLocalHoldCounter readHolds;
    // 缓存的计数器
    private transient HoldCounter cachedHoldCounter;
    // 第一个读线程
    private transient Thread firstReader = null;
    // 第一个读线程的计数
    private transient int firstReaderHoldCount;
}
```

说明：该属性中包括了读锁、写锁线程的最大量。本地线程计数器等。

（4）类的构造函数

```java
// 构造函数
Sync() {
    // 本地线程计数器
    readHolds = new ThreadLocalHoldCounter();
    // 设置AQS的状态
    setState(getState()); // ensures visibility of readHolds
}
```

说明：在Sync的构造函数中设置了本地线程计数器和AQS的状态state。

### 3、读写状态的设计

同步状态在重入锁的实现中是表示被同一个线程重复获取的次数，即一个整形变量来维护，但是之前的那个表示仅仅表示是否锁定，而不用区分是读锁还是写锁。而读写锁需要在同步状态（一个整形变量）上维护多个读线程和一个写线程的状态。

读写锁对于同步状态的实现是在一个整形变量上通过“按位切割使用”：将变量切割成两部分，**高16位表示读，低16位表示写**。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200621174835.png)

假设当前同步状态值为S，get和set的操作如下：

（1）获取写状态：

  S&0x0000FFFF:将高16位全部抹去

（2）获取读状态：

  S>>>16:无符号补0，右移16位

（3）写状态加1：

   S+1

（4）读状态加1：

　　S+（1<<16）即S + 0x00010000

在代码层的判断中，如果S不等于0，当写状态（S&0x0000FFFF），而读状态（S>>>16）大于0，则表示该读写锁的读锁已被获取。

### 4、写锁的获取与释放

看下WriteLock类中的lock和unlock方法：

```java
public void lock() {
    sync.acquire(1);
}

public void unlock() {
    sync.release(1);
}
```

可以看到就是调用的独占式同步状态的获取与释放，因此真实的实现就是Sync的 tryAcquire和 tryRelease。

#### tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    //当前线程
    Thread current = Thread.currentThread();
    //获取状态
    int c = getState();
    //写线程数量（即获取独占锁的重入数）
    int w = exclusiveCount(c);

    //当前同步状态state != 0，说明已经有其他线程获取了读锁或写锁
    if (c != 0) {
        // 当前state不为0，此时：如果写锁状态为0说明读锁此时被占用返回false；
        // 如果写锁状态不为0且写锁没有被当前线程持有返回false
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;

        //判断同一线程获取写锁是否超过最大次数（65535），支持可重入
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //更新状态
        //此时当前线程已持有写锁，现在是重入，所以只需要修改锁的数量即可。
        setState(c + acquires);
        return true;
    }

    //到这里说明此时c=0,读锁和写锁都没有被获取
    //writerShouldBlock表示是否阻塞（公平锁与非公平锁的区别，公平锁需判断是否为head节点）
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;

    //设置锁为当前线程所有
    setExclusiveOwnerThread(current);
    return true;
}
```

其中exclusiveCount方法表示占有写锁的线程数量，源码如下：

```java
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

说明：直接将状态state和（2^16 - 1）做与运算，其等效于将state模上2^16。写锁数量由state的低十六位表示。

从源代码可以看出，获取写锁的步骤如下：

- （1）首先获取c、w。c表示当前锁状态；w表示写线程数量。然后判断同步状态state是否为0。如果state!=0，说明已经有其他线程获取了读锁或写锁，执行(2)；否则执行(5)。

- （2）如果锁状态不为零（c != 0），而写锁的状态为0（w = 0），说明读锁此时被其他线程占用，所以当前线程不能获取写锁，自然返回false。或者锁状态不为零，而写锁的状态也不为0，但是获取写锁的线程不是当前线程，则当前线程也不能获取写锁。

- （3）判断当前线程获取写锁是否超过最大次数，若超过，抛异常，反之更新同步状态（此时当前线程已获取写锁，更新是线程安全的），返回true。

- （4）如果state为0，此时读锁或写锁都没有被获取，判断是否需要阻塞（公平和非公平方式实现不同），在非公平策略下总是不会被阻塞，在公平策略下会进行判断（**判断同步队列中是否有等待时间更长的线程**，若存在，则需要被阻塞，否则，无需阻塞），如果不需要阻塞，则CAS更新同步状态，若CAS成功则返回true，失败则说明锁被别的线程抢去了，返回false。如果需要阻塞则也返回false。

- （5）成功获取写锁后，将当前线程设置为占有写锁的线程，返回true。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200621175836.png)

#### tryRelease

```java
protected final boolean tryRelease(int releases) {
    //若锁的持有者不是当前线程，抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //写锁的新线程数
    int nextc = getState() - releases;
    //如果独占模式重入数为0了，说明独占模式被释放
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        //若写锁的新线程数为0，则将锁的持有者设置为null
        setExclusiveOwnerThread(null);
    //设置写锁的新线程数
    //不管独占模式是否被释放，更新独占重入数
    setState(nextc);
    return free;
}
```

写锁的释放过程还是相对而言比较简单的：首先查看当前线程是否为写锁的持有者，如果不是抛出异常。然后检查释放后写锁的线程数是否为0，如果为0则表示写锁空闲了，释放锁资源将锁的持有线程设置为null，否则释放仅仅只是一次重入锁而已，并不能将写锁的线程清空。

说明：此方法用于释放写锁资源，首先会判断该线程是否为独占线程，若不为独占线程，则抛出异常，否则，计算释放资源后的写锁的数量，若为0，表示成功释放，资源不将被占用，否则，表示资源还被占用。其方法流程图如下。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200621180154.png)

### 5、读锁的获取与释放

#### acquireShared

类似于写锁，读锁的lock和unlock的实际实现对应Sync的 tryAcquireShared 和 tryReleaseShared方法。

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

##### tryAcquireShared

```java
protected final int tryAcquireShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    // 获取状态
    int c = getState();

    //如果写锁线程数 != 0 ，且独占锁不是当前线程则返回失败（因为存在锁降级）
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 读锁数量
    int r = sharedCount(c);
    /*
     * readerShouldBlock():读锁是否需要等待（公平锁原则）
     * r < MAX_COUNT：持有线程小于最大数（65535）
     * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
     */
     // 读线程是否应该被阻塞、并且小于最大值、并且比较设置成功
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        //r == 0，表示第一个读锁线程，第一个读锁firstRead是不会加入到readHolds中
        if (r == 0) { // 读锁数量为0
            // 设置第一个读线程
            firstReader = current;
            // 读线程占用的资源数为1
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 当前线程为第一个读线程，表示第一个读锁线程重入
            // 占用资源数加1
            firstReaderHoldCount++;
        } else { // 读锁数量不为0并且不为当前线程
            // 获取计数器
            HoldCounter rh = cachedHoldCounter;
            // 计数器为空或者计数器的tid不为当前正在运行的线程的tid
            if (rh == null || rh.tid != getThreadId(current))
                // 获取当前线程对应的计数器
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0) // 计数为0
                //加入到readHolds中
                readHolds.set(rh);
            //计数+1
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

 其中sharedCount方法表示占有读锁的线程数量，源码如下：

```java
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
```

说明：直接将state右移16位，就可以得到读锁的线程数量，因为state的高16位表示读锁，对应的第十六位表示写锁数量。

读锁获取锁的过程比写锁稍微复杂些，首先判断写锁是否为0并且当前线程不占有独占锁，直接返回；否则，判断读线程是否需要被阻塞并且读锁数量是否小于最大值并且比较设置状态成功，若当前没有读锁，则设置第一个读线程firstReader和firstReaderHoldCount；若当前线程线程为第一个读线程，则增加firstReaderHoldCount；否则，将设置当前线程对应的HoldCounter对象的值。流程图如下。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200621182938.png)

注意：更新成功后会在firstReaderHoldCount中或readHolds(ThreadLocal类型的)的本线程副本中记录当前线程重入数（23行至43行代码），这是为了实现jdk1.6中加入的getReadHoldCount()方法的，这个方法能获取当前线程重入共享锁的次数(state中记录的是多个线程的总重入次数)，加入了这个方法让代码复杂了不少，但是其原理还是很简单的：如果当前只有一个线程的话，还不需要动用ThreadLocal，直接往firstReaderHoldCount这个成员变量里存重入数，当有第二个线程来的时候，就要动用ThreadLocal变量readHolds了，**每个线程拥有自己的副本，用来保存自己的重入数**。

##### fullTryAcquireShared

```java
final int fullTryAcquireShared(Thread current) {

    HoldCounter rh = null;
    for (;;) { // 无限循环
        // 获取状态
        int c = getState();
        if (exclusiveCount(c) != 0) { // 写线程数量不为0
            if (getExclusiveOwnerThread() != current) // 不为当前线程
                return -1;
        } else if (readerShouldBlock()) { // 写线程数量为0并且读线程被阻塞
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) { // 当前线程为第一个读线程
                // assert firstReaderHoldCount > 0;
            } else { // 当前线程不为第一个读线程
                if (rh == null) { // 计数器不为空
                    // 
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) { // 计数器为空或者计数器的tid不为当前正在运行的线程的tid
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT) // 读锁数量为最大值，抛出异常
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // 比较并且设置成功
            if (sharedCount(c) == 0) { // 读线程数量为0
                // 设置第一个读线程
                firstReader = current;
                // 
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

说明：在tryAcquireShared函数中，如果下列三个条件不满足（读线程是否应该被阻塞、小于最大值、比较设置成功）则会进行fullTryAcquireShared函数中，它用来保证相关操作可以成功。其逻辑与tryAcquireShared逻辑类似，不再累赘。

##### doAcquireShared

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

- 执行doAcquireShared的当前线程想要获取到共享锁。
- addWaiter将当前线程包装成一个共享模式的node放到队尾上去。

for循环的过程分析：

执行到tryAcquireShared后可能有两种情况：

- 如果tryAcquireShared的返回值>=0，说明线程获取共享锁成功了，那么调用setHeadAndPropagate，然后函数即将返回。
- 如果tryAcquireShared的返回值<0，说明线程获取共享锁失败了，那么调用shouldParkAfterFailedAcquire。
  - 这个shouldParkAfterFailedAcquire一般来说，得至少执行两遍才能将返回true：第一次shouldParkAfterFailedAcquirenode把前驱设置为SIGNAL状态，第二次检测到SIGNAL才返回true。
  - 既然上一条说了，shouldParkAfterFailedAcquirenode一般执行两遍，那么很有可能第二遍的时候，发现自己的前驱突然变成head了并且获取共享锁成功，又或者本来第一遍的前驱就是head但第二遍获取共享锁成功了。不用觉得第一遍的SIGNAL白设置了，因为设置前驱SIGNAL本来就是为了让前驱唤醒自己的，现在自己处于醒着的状态就获得了共享锁，那就接着执行setHeadAndPropagate就好。
  - 剩下的就是常见情况了。线程调用两次shouldParkAfterFailedAcquire，和一次parkAndCheckInterrupt后，便阻塞了。之后就只能等待别人unpark自己了，以后如果自己唤醒了，又会走以上这个流程。
    总之，执行doAcquireShared的线程一定会是局部变量node所代表的那个线程（即这个node的thread成员）。

##### setHeadAndPropagate

```java
private void setHeadAndPropagate(Node node, long propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

- 入参node所代表的线程一定是当前执行的线程，propagate则代表tryAcquireShared的**返回值**，由于有if (r >= 0)的保证，propagate必定为>=0，这里返回值的意思是：如果>0，说明我这次获取共享锁成功后，还有剩余共享锁可以获取；如果=0，说明我这次获取共享锁成功后，没有剩余共享锁可以获取。
- `Node h = head; setHead(node);`执行完这两句，h保存了旧的head，但现在head已经变成node了。
- `h == null`和`(h = head) == null和s == null`是为了防止空指针异常发生的标准写法，但这不代表就一定会发现它们为空的情况。这里的话，h == null和(h = head) == null是不可能成立，因为只要执行过addWaiter，CHL队列至少也会有一个node存在的；但s == null是可能发生的，比如node已经是队列的最后一个节点。
- 看第一个if的判断：
  - 如果propagate > 0成立的话，说明还有剩余共享锁可以获取，那么短路后面条件。
  - 中间穿插一下doReleaseShared的介绍：它不依靠参数，直接在调用中获取head，并在一定情况unparkSuccessor这个head。但注意，unpark head的后继之后，被唤醒的线程可能因为获取不到共享锁而再次阻塞（见上一章的流程分析）。
  - 如果propagate = 0成立的话，说明没有剩余共享锁可以获取了，按理说不需要唤醒后继的。也就是说，很多情况下，调用doReleaseShared，会造成acquire thread不必要的唤醒。之所以说不必要，是因为唤醒后因为没有共享锁可以获取而再次阻塞了。
  - 继续看，如果propagate > 0不成立，而h.waitStatus < 0成立。这说明旧head的status<0。但如果你看doReleaseShared的逻辑，会发现在unparkSuccessor之前就会CAS设置head的status为0的，在unparkSuccessor也会进行一次CAS尝试，因为head的status为0代表一种中间状态（head的后继代表的线程已经唤醒，但它还没有做完工作），或者代表head是tail。而这里旧head的status<0，只能是由于doReleaseShared里的compareAndSetWaitStatus(h, 0, Node.PROPAGATE)的操作，而且由于当前执行setHeadAndPropagate的线程只会在最后一句才执行doReleaseShared，所以出现这种情况，一定是因为有另一个线程在调用doReleaseShared才能造成，而这很可能是因为在中间状态时，又有人释放了共享锁。propagate == 0只能代表**当时tryAcquireShared后没有共享锁剩余，但之后的时刻很可能又有共享锁释放出来了**。
  - 继续看，如果propagate > 0不成立，且h.waitStatus < 0不成立，而第二个h.waitStatus < 0成立。注意，第二个h.waitStatus < 0里的h是新head（很可能就是入参node）。第一个h.waitStatus < 0不成立很正常，因为它一般为0（考虑别的线程可能不会那么碰巧读到一个中间状态）。第二个h.waitStatus < 0成立也很正常，因为只要新head不是队尾，那么新head的status肯定是SIGNAL。所以这种情况只会造成不必要的唤醒。
  - 看第二个if的判断：
    - `s == null`完全可能成立，当node是队尾时。此时会调用doReleaseShared，但doReleaseShared里会检测队列中是否存在两个node。
    - 当`s != null`且`s.isShared()`，也会调用doReleaseShared。

#### releaseShared

```java
public void unlock() {
    sync.releaseShared(1);
}
```

进入AQS追踪releaseShared方法：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

##### tryReleaseShared

```java
protected final boolean tryReleaseShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    if (firstReader == current) { // 当前线程为第一个读线程
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1) // 读线程占用的资源数为1
            firstReader = null;
        else // 减少占用的资源
            firstReaderHoldCount--;
    } else { // 当前线程不为第一个读线程
        // 获取缓存的计数器
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current)) // 计数器为空或者计数器的tid不为当前正在运行的线程的tid
            // 获取当前线程对应的计数器
            rh = readHolds.get();
        // 获取计数
        int count = rh.count;
        if (count <= 1) { // 计数小于等于1
            // 移除
            readHolds.remove();
            if (count <= 0) // 计数小于等于0，抛出异常
                throw unmatchedUnlockException();
        }
        // 减少计数
        --rh.count;
    }
    for (;;) { // 无限循环
        // 获取状态
        int c = getState();
        // 获取状态
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc)) // 比较并进行设置
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

说明：此方法表示读锁线程释放锁。首先判断当前线程是否为第一个读线程firstReader，若是，则判断第一个读线程占有的资源数firstReaderHoldCount是否为1，若是，则设置第一个读线程firstReader为空，否则，将第一个读线程占有的资源数firstReaderHoldCount减1；若当前线程不是第一个读线程，那么首先会获取缓存计数器（上一个读锁线程对应的计数器 ），若计数器为空或者tid不等于当前线程的tid值，则获取当前线程的计数器，如果计数器的计数count小于等于1，则移除当前线程对应的计数器，如果计数器的计数count小于等于0，则抛出异常，之后再减少计数即可。无论何种情况，都会进入无限循环，该循环可以确保成功设置状态state。其流程图如下。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200621183808.png)

在读锁的获取、释放过程中，总是会有一个对象存在着，同时该对象在获取线程获取读锁是+1，释放读锁时-1，该对象就是HoldCounter。

要明白HoldCounter就要先明白读锁。前面提过读锁的内在实现机制就是**共享锁**，对于共享锁其实我们可以稍微的认为它不是一个锁的概念，它更加像一个计数器的概念。一次共享锁操作就相当于一次计数器的操作，获取共享锁计数器+1，释放共享锁计数器-1。只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以HoldCounter的作用就是当前线程持有共享锁的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常。

先看**读锁获取锁**的部分：

```java
if (r == 0) {//r == 0，表示第一个读锁线程，第一个读锁firstRead是不会加入到readHolds中
    firstReader = current;
    firstReaderHoldCount = 1;
} else if (firstReader == current) {//第一个读锁线程重入
    firstReaderHoldCount++;    
} else {    //非firstReader计数
    HoldCounter rh = cachedHoldCounter;//readHoldCounter缓存
    //rh == null 或者 rh.tid != current.getId()，需要获取rh
    if (rh == null || rh.tid != current.getId())    
        cachedHoldCounter = rh = readHolds.get();
    else if (rh.count == 0)
        readHolds.set(rh);  //加入到readHolds中
    rh.count++; //计数+1
}
```

这里为什么要搞一个firstRead、firstReaderHoldCount呢？而不是直接使用else那段代码？这是为了一个效率问题，firstReader是不会放入到readHolds中的，如果读锁仅有一个的情况下就会避免查找readHolds。可能就看这个代码还不是很理解HoldCounter。我们先看firstReader、firstReaderHoldCount的定义：

```java
private transient Thread firstReader = null;
private transient int firstReaderHoldCount;
```

这两个变量比较简单，一个表示线程，当然该线程是一个特殊的线程，一个是firstReader的重入计数。

HoldCounter的定义：

```java
static final class HoldCounter {
    int count = 0;
    final long tid = Thread.currentThread().getId();
}
```

在HoldCounter中仅有count和tid两个变量，其中count代表着计数器，tid是线程的id。但是如果要将一个对象和线程绑定起来仅记录tid肯定不够的，而且HoldCounter根本不能起到绑定对象的作用，只是记录线程tid而已。

诚然，在java中，我们知道如果要将一个线程和对象绑定在一起只有ThreadLocal才能实现。所以如下：

```java
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

ThreadLocalHoldCounter继承ThreadLocal，并且重写了initialValue方法。  

故而，HoldCounter应该就是绑定线程上的一个计数器，而ThradLocalHoldCounter则是线程绑定的ThreadLocal。从上面我们可以看到ThreadLocal将HoldCounter绑定到当前线程上，同时HoldCounter也持有线程Id，这样在释放锁的时候才能知道ReadWriteLock里面缓存的上一个读取线程（cachedHoldCounter）**是否是当前线程**。这样做的好处是**可以减少ThreadLocal.get()的次数，因为这也是一个耗时操作**。需要说明的是这样HoldCounter绑定线程id而不绑定线程对象的原因是避免HoldCounter和ThreadLocal互相绑定**而GC难以释放它们**（尽管GC能够智能的发现这种引用而回收它们，但是这需要一定的代价），所以其实这样做只是为了帮助GC快速回收对象而已。

##### doReleaseShared

```java
private void doReleaseShared() {
    // 唤醒后续的写线程
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { // 第一种：如果状态是-1，说明后面肯定有阻塞的任务，要去唤醒它
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            } // ws等于0说明是最后一个节点了，此时将Node的ws设置为PROPAGATE，因为后续没有节点了，所以不用唤醒
            else if (ws == 0 &&!compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)  // 不等于head说明又有新的读锁进来了，这时要继续循环
            break;
    }
}
```

此方法用于**唤醒读锁后处于挂起状态的锁**，读锁后处于挂起状态的锁有两种：第一种是写锁，这很好理解，如果有读锁被占用，写锁过来的时候肯定需要挂起等读锁执行完（非相同线程），读锁执行完之后唤醒这个写锁；第二种是读锁，为什么当前执行的是读锁而后面还会有读锁被挂起呢？，在读锁加锁中我们讲过一个 apparentlyFirstQueuedIsExclusive 方法，该方法会**判断队列中第一个排队的是不是写锁**，**如果是写锁则让当前的读锁挂起不去竞争锁**，而若在队首写锁等待的过程中有多个读锁过来，则这多个读锁都会被依次挂起，这时就会出现第二种情况，即读锁执行的时候后面还有一个读锁被挂起，执行完之后需唤醒它。

此处第二种读锁唤醒读锁的场景，是在读锁加锁时触发的。在**doAcquireShared**方法中，有个setHeadAndPropagate方法，在该方法中会检测下一个节点是不是读锁，如果是就调用doReleaseShared方法唤醒它。

## 三、总结

通过上面的源码分析，我们可以发现一个现象：

在线程持有读锁的情况下，该线程不能取得写锁(因为获取写锁的时候，如果发现当前的读锁被占用，就马上获取失败，不管读锁是不是被当前线程持有)。

在线程持有写锁的情况下，该线程可以继续获取读锁（获取读锁时如果发现写锁被占用，只有写锁没有被当前线程占用的情况才会获取失败）。

仔细想想，这个设计是合理的：因为当线程获取读锁的时候，可能有其他线程同时也在持有读锁，因此不能把获取读锁的线程“升级”为写锁；而对于获得写锁的线程，它一定独占了读写锁，因此可以继续让它获取读锁，当它同时获取了写锁和读锁后，还可以先释放写锁继续持有读锁，这样一个写锁就“降级”为了读锁。

综上：

一个线程要想同时持有写锁和读锁，**必须先获取写锁再获取读锁**；**写锁可以“降级”为读锁**；**读锁不能“升级”为写锁**。

## 四、关于head的更新问题

在AQS中只有`setHead`方法会更新head节点，而setHead只有当加锁时才会调用，那么释放锁时仅仅只是唤醒后继结节，让后继节点竞争锁，竞争到锁以后再更新head

```java
 /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
private transient volatile Node head;


/**
     * Sets head of queue to be node, thus dequeuing. Called only by
     * acquire methods.  Also nulls out unused fields for sake of GC
     * and to suppress unnecessary signals and traversals.
     *
     * @param node the node
     */
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

```java
// 唤醒后继节点，但是这时后继节点并不一定会获取锁
private void unparkSuccessor(Node node) {
    /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

```java
// 加锁的方法，
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 获得锁，并且前一个锁是head节点，则更新head节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 线程堵塞，
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

