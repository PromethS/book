# 1. AQS简介

AQS，全名AbstractQueuedSynchronizer，是一个抽象类的队列式同步器，它的内部通过维护一个状态volatile int state(共享资源)，一个FIFO线程等待队列来实现同步功能。

state用关键字volatile修饰，代表着该共享资源的状态一更改就能被所有线程可见，而AQS的加锁方式本质上就是多个线程在**竞争state**，当state为0时代表线程可以竞争锁，不为0时代表当前对象锁已经被占有，其他线程来加锁时则会失败，加锁失败的线程会被放入一个FIFO的等待队列中，这些线程会被`UNSAFE.park()`操作挂起，等待其他获取锁的线程释放锁才能够被唤醒。

而这个等待队列其实就相当于一个CLH队列，用一张原理图来表示大致如下：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220165705.png)

**基础定义**

AQS支持两种资源分享的方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。

自定义的同步器继承AQS后，只需要实现共享资源state的获取和释放方式即可，其他如线程队列的维护（如获取资源失败入队/唤醒出队等）等操作，AQS在顶层已经实现了

AQS代码内部提供了一系列操作锁和线程队列的方法，主要操作锁的方法包含以下几个：

- compareAndSetState()：利用CAS的操作来设置state的值
- tryAcquire(int)：独占方式获取锁。成功则返回true，失败则返回false。
- tryRelease(int)：独占方式释放锁。成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式释放锁。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式释放锁。如果释放后允许唤醒后续等待结点返回true，否则返回false。

像ReentrantLock就是实现了自定义的tryAcquire-tryRelease，从而操作state的值来实现同步效果。

除此之外，AQS内部还定义了一个静态类Node，表示CLH队列的每一个结点，该结点的作用是对每一个等待获取资源做了封装，包含了需要同步的线程本身、线程等待状态.....看下该类的一些重点变量：

```java
static final class Node {
  /** 表示共享模式下等待的Node */
  static final Node SHARED = new Node();
  /** 表示独占模式下等待的mode */
  static final Node EXCLUSIVE = null;

  /** 下面几个为waitStatus的具体值 */
  static final int CANCELLED =  1;
  static final int SIGNAL    = -1;
  static final int CONDITION = -2;
  static final int PROPAGATE = -3;

  volatile int waitStatus;

  /** 表示前面的结点 */
  volatile Node prev;
  /** 表示后面的结点 */
  volatile Node next;
  /**当前结点装载的线程，初始化时被创建，使用后会置空*/
  volatile Thread thread;
  /**链接到下一个节点的等待条件，用到Condition的时候会使用到*/
  Node nextWaiter;
}
```

代码里面定义了一个表示当前Node结点等待状态的字段`waitStatus`，该字段的取值包含了CANCELLED(1)、SIGNAL(-1)、CONDITION(-2)、PROPAGATE(-3)、0，这五个值代表了不同的特定场景：

- **CANCELLED**：表示当前结点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
- **SIGNAL**：表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL（记住这个-1的值，因为后面我们讲的时候经常会提到）
- **CONDITION**：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将**从等待队列转移到同步队列中**，等待获取同步锁。(注：**Condition**是AQS的一个组件，后面会细说)
- **PROPAGATE**：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
- **0**：新结点入队时的默认状态。

也就是说，当waitStatus为**负值表示结点处于有效等待状态，为正值的时候表示结点已被取消。**

在AQS内部中还维护了两个Node对象`head`和`tail`，一开始默认都为null

```
private transient volatile Node head;
private transient volatile Node tail;   
```

讲完了AQS的一些基础定义，我们就可以开始学习同步的具体运行机制了，为了更好的演示，我们用ReentrantLock作为使用入口，一步步跟进源码探究AQS底层是如何运作的，**这里说明一下，因为ReentrantLock底层调用的AQS是独占模式，所以下文讲解的AQS源码也是针对独占模式的操作**

# 2. 独占模式

## 2.1 加锁过程

ReentrantLock的加锁和解锁方法分别为lock()和unLock()，我们先来看获取锁的方法，

```java
final void lock() {
 if (compareAndSetState(0, 1))
  setExclusiveOwnerThread(Thread.currentThread());
 else
  acquire(1);
}
```

逻辑很简单，线程进来后直接利用`CAS`尝试抢占锁，如果抢占成功`state`值回被改为1，且设置对象独占锁线程为当前线程，否则就调用`acquire(1)`再次尝试获取锁。

我们假定有两个线程A和B同时竞争锁，A进来先抢占到锁，此时的AQS模型图就类似这样：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220170258.png)

继续走下面的方法，

```java
public final void acquire(int arg) {
 if (!tryAcquire(arg) &&
  acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
  selfInterrupt();
}
```

`acquire`包含了几个函数的调用，

`tryAcquire`：尝试直接获取锁，如果成功就直接返回；

`addWaiter`：将该线程加入等待队列FIFO的尾部，并标记为独占模式；

`acquireQueued`：线程阻塞在等待队列中获取锁，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。

`selfInterrupt`：自我中断，就是既拿不到锁，又在等待时被中断了，线程就会进行自我中断selfInterrupt()，将中断补上。

我们一个个来看源码，并结合上面的两个线程来做场景分析。

### 2.1.1 tryAcquire

不用多说，就是为了再次尝试获取锁

```java
protected final boolean tryAcquire(int acquires) {
 return nonfairTryAcquire(acquires);
}
// 非公平锁，如果是公平锁，则会额外判断线程是否在队列的头部header
final boolean nonfairTryAcquire(int acquires) {
 final Thread current = Thread.currentThread();
 int c = getState();
 if (c == 0) {
  if (compareAndSetState(0, acquires)) {
   setExclusiveOwnerThread(current);
   return true;
  }
 }
 else if (current == getExclusiveOwnerThread()) {
  int nextc = c + acquires;
  if (nextc < 0) // overflow
   throw new Error("Maximum lock count exceeded");
  setState(nextc);
  return true;
 }
 return false;
}
```

当线程B进来后，**nonfairTryAcquire**方法首先会获取state的值，如果为0，则正常获取该锁，不为0的话判断是否是当前线程占用了，是的话就累加state的值，这里的累加也是为了配合释放锁时候的次数，从而实现可重入锁的效果。

当然，因为之前锁已经被线程A占领了，所以这时候`tryAcquire`会返回false，继续下面的流程。

### 2.1.2 addWaiter

```java
private Node addWaiter(Node mode) {
 Node node = new Node(Thread.currentThread(), mode);
 // Try the fast path of enq; backup to full enq on failure
 Node pred = tail;
 if (pred != null) {
  node.prev = pred;
  if (compareAndSetTail(pred, node)) {
   pred.next = node;
   return node;
  }
 }
 enq(node);
 return node;
}
```

这段代码首先会创建一个和当前线程绑定的`Node`节点，`Node`为双向链表。此时等待队列中的`tail`指针为空，直接调用`enq(node)`方法将当前线程加入等待队列尾部，然后返回当前结点的前驱结点，

```java
private Node enq(final Node node) {
 // CAS"自旋"，直到成功加入队尾
    for (;;) {
        Node t = tail;
        if (t == null) {
         // 队列为空，初始化一个Node结点作为Head结点，并将tail结点也指向它
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
         // 把当前结点插入队列尾部
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

第一遍循环时，tail指针为空，初始化一个Node结点，并把head和tail结点都指向它，然后第二次循环进来之后，tail结点不为空了，就将当前的结点加入到tail结点后面，也就是这样：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220170813.png)

如果此时有另一个线程C进来的话，发现锁已经被A拿走了，然后队列里已经有了线程B，那么线程C就只能乖乖排到线程B的后面去

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220170918.png)

### 2.1.3 acquireQueued

接着解读方法，通过tryAcquire()和addWaiter()，我们的线程还是没有拿到资源，并且还被排到了队列的尾部，如果让你来设计的话，这个时候你会怎么处理线程呢？其实答案也很简单，能做的事无非两个：

**1、循环让线程再抢资源。但仔细一推敲就知道不合理，因为如果有多个线程都参与的话，你抢我也抢只会降低系统性能**

**2、进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源**

毫无疑问，选择2更加靠谱，acquireQueued方法做的也是这样的处理：

```java
final boolean acquireQueued(final Node node, int arg) {
 boolean failed = true;
 try {
  // 标记是否会被中断
  boolean interrupted = false;
  // CAS自旋
  for (;;) {
   // 获取当前结点的前结点
   final Node p = node.predecessor();
   if (p == head && tryAcquire(arg)) {
    setHead(node);
    p.next = null; // help GC
    failed = false;
    return interrupted;
   }
   if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
  }
 } finally {
  if (failed)
   // 获取锁失败，则将此线程对应的node的waitStatus改为CANCEL
   cancelAcquire(node);
 }
}
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
 int ws = pred.waitStatus;
 if (ws == Node.SIGNAL)
  // 前驱结点等待状态为"SIGNAL"，那么自己就可以安心等待被唤醒了
  return true;
 if (ws > 0) {
  /*
   * 前驱结点被取消了，通过循环一直往前找，直到找到等待状态有效的结点(等待状态值小于等于0) ，
   * 然后排在他们的后边，至于那些被当前Node强制"靠后"的结点，因为已经被取消了，也没有引用链，
   * 就等着被GC了
   */
  do {
   node.prev = pred = pred.prev;
  } while (pred.waitStatus > 0);
  pred.next = node;
 } else {
  // 如果前驱正常，那就把前驱的状态设置成SIGNAL
  compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
 }
 return false;
}
private final boolean parkAndCheckInterrupt() {
 LockSupport.park(this);
 return Thread.interrupted();
}
```

`acquireQueued`方法的流程是这样的：

1、CAS自旋，先判断当前传入的Node的前结点是否为head结点，是的话就尝试获取锁，获取锁成功的话就把当前结点置为head，之前的head置为null(方便GC)，然后返回

2、如果前驱结点不是head或者加锁失败的话，就调用`shouldParkAfterFailedAcquire`，将前驱节点的**waitStatus**变为了**SIGNAL=-1**，最后执行`parkAndChecknIterrupt`方法，调用`LockSupport.park()`挂起当前线程，`parkAndCheckInterrupt`在挂起线程后会判断线程是否被中断，如果被中断的话，就会重新跑`acquireQueued`方法的CAS自旋操作，直到获取资源。

ps：LockSupport.park方法会让当前线程进入waitting状态，在这种状态下，线程被唤醒的情况有两种，一是被unpark()，二是被interrupt()，所以，如果是第二种情况的话，需要返回被中断的标志，然后在`acquire`顶层方法的窗口那里自我中断补上

此时，因为线程A还未释放锁，所以线程B状态都是被挂起的，

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220171713.png)

到这里，加锁的流程就分析完了，其实整体来说也并不复杂，而且当你理解了独占模式加锁的过程，后面释放锁和共享模式的运行机制也没什么难懂的了，所以整个加锁的过程还是有必要多消化下的，也是AQS的重中之重。

为了方便你们更加清晰理解，我加多一张流程图吧（这个作者也太暖了吧，哈哈）

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220171734.png)

## 2. 释放锁

说完了加锁，我们来看看释放锁是怎么做的，AQS中释放锁的方法是`release()`，当调用该方法时会释放指定量的资源 (也就是锁) ，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。

```java
public final boolean release(int arg) {
 if (tryRelease(arg)) {
  Node h = head;
  if (h != null && h.waitStatus != 0)
   unparkSuccessor(h);
  return true;
 }
 return false;
}
```

### 2.1 tryRelease

代码上可以看出，核心的逻辑都在`tryRelease`方法中，该方法的作用是释放资源，AQS里该方法没有具体的实现，需要由自定义的同步器去实现，我们看下**ReentrantLock**代码中对应方法的源码：

```java
protected final boolean tryRelease(int releases) {
 int c = getState() - releases;
 if (Thread.currentThread() != getExclusiveOwnerThread())
  throw new IllegalMonitorStateException();
 boolean free = false;
 if (c == 0) {
  free = true;
  setExclusiveOwnerThread(null);
 }
 setState(c);
 return free;
}
```

`tryRelease`方法会减去state对应的值，如果state为0，也就是已经彻底释放资源，就返回true，并且把独占的线程置为null，否则返回false。

此时AQS中的数据就会变成这样：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220172020.png)

完全释放资源后，当前线程要做的就是唤醒CLH队列中第一个在等待资源的线程，也就是head结点后面的线程，此时调用的方法是`unparkSuccessor()`，

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
     //将head结点的状态置为0
        compareAndSetWaitStatus(node, ws, 0);
 //找到下一个需要唤醒的结点s
    Node s = node.next;
    //如果为空或已取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后向前，直到找到等待状态小于0的结点，前面说了，结点waitStatus小于0时才有效
        for (Node t = tail; t != null && t != node; t = t.prev) 
            if (t.waitStatus <= 0)
                s = t;
    }
    // 找到有效的结点，直接唤醒
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒
}
```

方法的逻辑很简单，就是先将head的结点状态置为0，避免下面找结点的时候再找到head，然后找到队列中最前面的有效结点，然后唤醒，我们假设这个时候线程A已经释放锁，那么此时队列中排最前边竞争锁的线程B就会被唤醒。然后被唤醒的线程B就会尝试用CAS获取锁，回到`acquireQueued`方法的逻辑，

```java
for (;;) {
 // 获取当前结点的前结点
 final Node p = node.predecessor();
 if (p == head && tryAcquire(arg)) {
  setHead(node);
  p.next = null; // help GC
  failed = false;
  return interrupted;
 }
 if (shouldParkAfterFailedAcquire(p, node) &&
  parkAndCheckInterrupt())
  interrupted = true;
}
```

当线程B获取锁之后，会把当前结点赋值给head，然后原先的前驱结点 (也就是原来的head结点) 去掉引用链，方便回收，这样一来，线程B获取锁的整个过程就完成了，此时AQS的数据就会变成这样：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220172135.png)

到这里，我们已经分析完了AQS独占模式下加锁和释放锁的过程，也就是**tryAccquire->tryRelease**这一链条的逻辑，除此之外，AQS中还支持共享模式的同步，这种模式下关于锁的操作核心其实就是**tryAcquireShared->tryReleaseShared**这两个方法，我们可以简单看下

# 2. 共享模式

## 2.1 获取锁

AQS中，共享模式获取锁的顶层入口方法是`acquireShared`，该方法会获取指定数量的资源，成功的话就直接返回，失败的话就进入等待队列，直到获取资源，

```java
public final void acquireShared(int arg) {
 if (tryAcquireShared(arg) < 0)
  doAcquireShared(arg);
}
```

该方法里包含了两个方法的调用，

tryAcquireShared：尝试获取一定资源的锁，返回的值代表获取锁的状态。

doAcquireShared：进入等待队列，并循环尝试获取锁，直到成功。

### 2.1.1 tryAcquireShared

`tryAcquireShared`在AQS里没有实现，同样由自定义的同步器去完成具体的逻辑，像一些较为常见的并发工具Semaphore、CountDownLatch里就有对该方法的自定义实现，虽然实现的逻辑不同，但方法的作用是一样的，就是获取一定资源的资源，然后根据返回值判断是否还有剩余资源，从而决定下一步的操作。

返回值有三种定义：

- 负值代表获取失败；
- 0代表获取成功，但没有剩余的资源，也就是state已经为0；
- 正值代表获取成功，而且state还有剩余，其他线程可以继续领取

当返回值小于0时，证明此次获取一定数量的锁失败了，然后就会走`doAcquireShared`方法

### 2.1.2 doAcquireShared

此方法的作用是将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回，这是它的源码：

```java
private void doAcquireShared(int arg) {
 // 加入队列尾部
 final Node node = addWaiter(Node.SHARED);
 boolean failed = true;
 try {
  boolean interrupted = false;
  // CAS自旋
  for (;;) {
   final Node p = node.predecessor();
   // 判断前驱结点是否是head
   if (p == head) {
    // 尝试获取一定数量的锁
    int r = tryAcquireShared(arg);
    if (r >= 0) {
     // 获取锁成功，而且还有剩余资源，就设置当前结点为head，并继续唤醒下一个线程
     setHeadAndPropagate(node, r);
     // 让前驱结点去掉引用链，方便被GC
     p.next = null; // help GC
     if (interrupted)
      selfInterrupt();
     failed = false;
     return;
    }
   }
   // 跟独占模式一样，改前驱结点waitStatus为-1，并且当前线程挂起，等待被唤醒
   if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
  }
 } finally {
  if (failed)
   cancelAcquire(node);
 }
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    // head指向自己
    setHead(node);
     // 如果还有剩余量，继续唤醒下一个邻居线程
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

看到这里，你会不会一点熟悉的感觉，这个方法的逻辑怎么跟上面那个`acquireQueued()`那么类似啊？对的，其实两个流程并没有太大的差别。只是`doAcquireShared()`比起独占模式下的获取锁上多了一步唤醒后继线程的操作，当获取完一定的资源后，发现还有剩余的资源，就继续唤醒下一个邻居线程，这才符合"共享"的思想嘛。

这里我们可以提出一个疑问，共享模式下，当前线程释放了一定数量的资源，但这部分资源满足不了下一个等待结点的需要的话，那么会怎么样？

按照正常的思维，共享模式是可以多个线程同时执行的才对，所以，多个线程的情况下，如果老大释放完资源，但这部分资源满足不了老二，但能满足老三，那么老三就可以拿到资源。可事实是，从源码设计中可以看出，如果真的发生了这种情况，老三是拿不到资源的，因为等待队列是按顺序排列的，老二的资源需求量大，会把后面量小的老三以及老四、老五等都给卡住。**从这一个角度来看，虽然AQS严格保证了顺序，但也降低了并发能力**

接着往下说吧，唤醒下一个邻居线程的逻辑在`doReleaseShared()`中，我们放到下面的释放锁来解析。

## 2.2 释放锁

共享模式释放锁的顶层方法是`releaseShared`，它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。下面是releaseShared()的源码：

```java
public final boolean releaseShared(int arg) {
 if (tryReleaseShared(arg)) {
  doReleaseShared();
  return true;
 }
 return false;
}
```

该方法同样包含两部分的逻辑：

tryReleaseShared：释放资源。

doAcquireShared：唤醒后继结点。

跟`tryAcquireShared`方法一样，`tryReleaseShared`在AQS中没有具体的实现，由子同步器自己去定义，但功能都一样，就是释放一定数量的资源。

释放完资源后，线程不会马上就收工，而是唤醒等待队列里最前排的等待结点。

### 2.2.1 doReleaseShared

唤醒后继结点的工作在`doReleaseShared()`方法中完成，我们可以看下它的源码：

```java
private void doReleaseShared() {
 for (;;) {
  // 获取等待队列中的head结点
  Node h = head;
  if (h != null && h != tail) {
   int ws = h.waitStatus;
   // head结点waitStatus = -1,唤醒下一个结点对应的线程
   if (ws == Node.SIGNAL) {
    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
     continue;            // loop to recheck cases
    // 唤醒后继结点
    unparkSuccessor(h);
   }
   else if (ws == 0 &&
      !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
    continue;                // loop on failed CAS
  }
  if (h == head)                   // loop if head changed
   break;
 }
}
```

代码没什么特别的，就是如果等待队列head结点的waitStatus为-1的话，就直接唤醒后继结点，唤醒的方法`unparkSuccessor()`在上面已经讲过了，这里也没必要再复述。

总的来看，AQS共享模式的运作流程和独占模式很相似，只要掌握了独占模式的流程运转，共享模式什么的不就那样吗，没难度。这也是我为什么共享模式讲解中不画流程图的原因，没必要嘛。

## 2.3 Condition

在AQS中，除了提供独占/共享模式的加锁/解锁功能，它还对外提供了关于**Condition**的一些操作方法。

**Condition**是个接口，在jdk1.5版本后设计的，基本的方法就是`await()`和`signal()`方法，功能大概就对应**Object**的`wait()`和`notify()`，Condition必须要配合锁一起使用，因为对共享状态变量的访问发生在多线程环境下。一个Condition的实例必须与一个Lock绑定，因此Condition一般都是作为Lock的内部实现 ，AQS中就定义了一个类**ConditionObject**来实现了这个接口，

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220200153.png)

那么它应该怎么用呢？我们可以简单写个demo来看下效果

```java
public class ConditionDemo {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        Thread tA = new Thread(() -> {
            lock.lock();
            try {
                System.out.println("线程A加锁成功");
                System.out.println("线程A执行await被挂起");
                condition.await();
                System.out.println("线程A被唤醒成功");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println("线程A释放锁成功");
            }
        });

        Thread tB = new Thread(() -> {
            lock.lock();
            try {
                System.out.println("线程B加锁成功");
                condition.signal();
                System.out.println("线程B唤醒线程A");
            } finally {
                lock.unlock();
                System.out.println("线程B释放锁成功");
            }
        });
        tA.start();
        tB.start();
    }
}
```

执行main函数后结果输出为：

```javascript
线程A加锁成功
线程A执行await被挂起
线程B加锁成功
线程B唤醒线程A
线程B释放锁成功
线程A被唤醒成功
线程A释放锁成功
```

代码执行的结果很容易理解，线程A先获取锁，然后调用`await()`方法挂起当前线程并释放锁，线程B这时候拿到锁，然后调用`signal`唤醒线程A。

毫无疑问，这两个方法让线程的状态发生了变化，我们仔细来研究一下

翻看AQS的源码，我们会发现Condition中定义了两个属性`firstWaiter`和`lastWaiter`，前面说了，AQS中包含了一个FIFO的CLH等待队列，每个Conditon对象就包含这样一个等待队列，而这两个属性分别表示的是等待队列中的首尾结点

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

**注意：Condition当中的等待队列和AQS主体的同步等待队列是分开的，两个队列虽然结构体相同，但是作用域是分开的**

### 2.3.1 await

看`await()`的源码：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程加入到等待队列中
    Node node = addConditionWaiter();
    // 完全释放占有的资源，并返回资源数
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 循环判断当前结点是不是在Condition的队列中，是的话挂起
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

当一个线程调用**Condition.await()*方法，将会以当前线程构造结点，这个结点的`waitStatus`赋值为*Node.CONDITION**，也就是-2，并将结点从尾部加入等待队列，然后尾部结点就会指向这个新增的结点，

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

我们依然用上面的demo来演示，此时，线程A获取锁并调用**Condition.await()**方法后，AQS内部的数据结构会变成这样：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220200517.png)

在Condition队列中插入对应的结点后，线程A会释放所持有的资源，走到while循环那层逻辑，

```java
while (!isOnSyncQueue(node)) {
 LockSupport.park(this);
 if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
  break;
}
```

`isOnSyncQueue`方法的会判断当前的线程节点是不是在同步队列中，这个时候此结点还在Condition队列中，所以该方法返回false，这样的话循环会一直持续下去，线程被挂起，等待被唤醒，此时，线程A的流程暂时停止了。

当线程A调用`await()`方法挂起的时候，线程B获取到了线程A释放的资源，然后执行`signal()`方法：

### 2.3.2 signal

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

先判断当前线程是否为获取锁的线程，如果不是则直接抛出异常。接着调用`doSignal()`方法来唤醒线程。

```java
private void doSignal(Node first) {
 // 循环，从队列一直往后找不为空的首结点
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
 // CAS循环，将结点的waitStatus改为0
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
 // 上面已经分析过，此方法会把当前结点加入到等待队列中，并返回前驱结点
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

从`doSignal`的代码中可以看出，这时候程序寻找的是Condition等待队列中首结点firstWaiter的结点，此时该结点指向的是线程A的结点，所以之后的流程作用的都是线程A的结点。

这里分析下`transferForSignal`方法，先通过CAS自旋将结点**waitStatus**改为0，然后就把结点放入到同步队列 (此队列不是Condition的等待队列) 中，然后再用CAS将同步队列中该结点的前驱结点waitStatus改为Node.SIGNAL，也就是-1，此时AQS的数据结构大概如下(额.....少画了个箭头，大家就当head结点是线程A结点的前驱结点就好)：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220200759.png)

回到`await()`方法，当线程A的结点被加入同步队列中时，`isOnSyncQueue()`会返回true，跳出循环，

```java
while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
```

接着执行`acquireQueued()`方法，这里就不用多说了吧，尝试重新获取锁，如果获取锁失败继续会被挂起，直到另外线程释放锁才被唤醒。

所以，当线程B释放完锁后，线程A被唤醒，继续尝试获取锁，至此流程结束。

对于这整个通信过程，我们可以画一张流程图展示下：

![图片](https://520li.oss-cn-hangzhou.aliyuncs.com/img/book/20201220201001.png)

### 总结

说完了Condition的使用和底层运行机制，我们再来总结下它跟普通 wait/notify 的比较，一般这也是问的比较多的，Condition大概有以下两点优势：

- Condition 需要结合 Lock 进行控制，使用的时候要注意一定要对应的unlock()，可以对多个不同条件进行控制，只要new 多个 Condition对象就可以为多个线程控制通信，wait/notify 只能和 synchronized 关键字一起使用，并且只能唤醒一个或者全部的等待队列；
- Condition 有类似于 await 的机制，因此不会产生加锁方式而产生的死锁出现，同时底层实现的是 park/unpark 的机制，因此也不会产生先唤醒再挂起的死锁，一句话就是不会产生死锁，但是 wait/notify 会产生先唤醒再挂起的死锁。

# 3. 其他

## 3.1 读写锁

常用的synchronized和ReentrantLock都是**排它锁**，同一时刻只能有一个线程访问，而**读写锁**可以允许多个线程同时访问。内部维护了两把锁（读锁、写锁），通过分离读写锁，使得在**读多写少**的场景下提高性能。

- ReentrantReadWriteLock是⼀个读写锁，如果读的线程⽐写的线程要多很多的话，那可以考虑使⽤它。它使⽤state的变量**⾼16位是读锁，低16位是写锁**
- **写锁可以降级为读锁，读锁不能升级为写锁**
- **写锁是互斥的，读锁是共享的**。

详见： [ReentrantReadWriteLock](ReentrantReadWriteLock.md) 

### ReentrantReadWirteLock

这是一个非抽象类，它是ReadWriteLock接口的JDK默认实现。它与ReentrantLock的功能类似，同样是可重入的，支持非公平锁和公平锁。不同的是，它还支持”读写锁“。

但它有一个小弊端，就是在“写”操作的时候，其它线程不能写也不能读。我们称这种现象为“**写饥饿**”

```java
// 内部结构
private final ReentrantReadWriteLock.ReadLock readerLock;
private final ReentrantReadWriteLock.WriteLock writerLock;
final Sync sync;
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 具体实现
}
static final class NonfairSync extends Sync {
    // 具体实现
}
static final class FairSync extends Sync {
    // 具体实现
}
public static class ReadLock implements Lock, java.io.Serializable {
    private final Sync sync;
    protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
    }
    // 具体实现
}
public static class WriteLock implements Lock, java.io.Serializable {
    private final Sync sync;
    protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
    }
    // 具体实现
}

// 构造方法，初始化两个锁
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

// 获取读锁和写锁的方法
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

### StampedLock

`StampedLock`类是在Java 8 才发布的，也是Doug Lea大神所写，有人号称它为锁的**性能之王**。它没有实现Lock接口和`ReadWriteLock`接口，但它其实是实现了“读写锁”的功能，并且性能比`ReentrantReadWriteLock`更高。`StampedLock`还把读锁分为了“乐观读锁”和“悲观读锁”两种。

核心思想在于，**在读的时候如果发生了写，应该通过重试的方式来获取新的值，而不应该阻塞写操作。这种模式也就是典型的无锁编程思想，和CAS自旋的思想一样**。

```java
// StampedLock的用法
class Point {
   private double x, y;
   private final StampedLock sl = new StampedLock();

   // 写锁的使用
   void move(double deltaX, double deltaY) {
     long stamp = sl.writeLock(); // 获取写锁
     try {
       x += deltaX;
       y += deltaY;
     } finally {
       sl.unlockWrite(stamp); // 释放写锁
     }
   }

   // 乐观读锁的使用
   double distanceFromOrigin() {
     long stamp = sl.tryOptimisticRead(); // 获取乐观读锁
     double currentX = x, currentY = y;
     if (!sl.validate(stamp)) { // //检查乐观读锁后是否有其他写锁发生，有则返回false
        stamp = sl.readLock(); // 获取一个悲观读锁
        try {
          currentX = x;
          currentY = y;
        } finally {
           sl.unlockRead(stamp); // 释放悲观读锁
        }
     }
     return Math.sqrt(currentX * currentX + currentY * currentY);
   }

   // 悲观读锁以及读锁升级写锁的使用
   void moveIfAtOrigin(double newX, double newY) {
     long stamp = sl.readLock(); // 悲观读锁
     try {
       while (x == 0.0 && y == 0.0) {
         // 读锁尝试转换为写锁：转换成功后相当于获取了写锁，转换失败相当于有写锁被占用
         long ws = sl.tryConvertToWriteLock(stamp); 

         if (ws != 0L) { // 如果转换成功
           stamp = ws; // 读锁的票据更新为写锁的
           x = newX;
           y = newY;
           break;
         }
         else { // 如果转换失败
           sl.unlockRead(stamp); // 释放读锁
           stamp = sl.writeLock(); // 强制获取写锁
         }
       }
     } finally {
       sl.unlock(stamp); // 释放所有锁
     }
   }
}}
```

### CopyOnWrite

CopyOnWrite容器即**写时复制的容器**，当我们往一个容器中添加元素的时候，不直接往容器中添加，而是将当前容器进行copy，复制出来一个新的容器，然后向新容器中添加我们需要的元素，最后将原容器的引用指向新容器。

**优点：**适用于**读多写少**场景，在读的时候不会加锁，大大提高读的性能。在写的时候加锁，又可以保证数据**最终一致性**。

**缺点：**占用内存比较大，可能会引起频繁GC。在写操作的过程中，读取到的数据可能是老数据，会有短暂的不一致。

```java
// CopyOnWriteArrayList的add方法
public boolean add(E e) {

    // ReentrantLock加锁，保证线程安全
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 拷贝原容器，长度为原容器长度加一
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 在新副本上执行添加操作
        newElements[len] = e;
        // 将原容器引用指向新副本
        setArray(newElements);
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}
```



