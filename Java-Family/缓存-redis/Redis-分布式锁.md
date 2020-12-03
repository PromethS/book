## 特性

- **互斥性**，保证只有一个线程持有锁；
- **避免死锁**，不允许一个锁被永久持有的可能性，即使服务奔溃了，也要保证锁在一定期间内会被安全释放；
- **高可用性**，除非整个分布式系统瘫痪，只要有服务存活，都允许获取和释放锁；
- 谁上的锁由谁释放，不能被**误解锁**。
- **可重入性**，基于redisson可实现相同线程多次加锁。

## 可实现方式

- **基于 MySQL 中的锁**：MySQL 本身有自带的悲观锁 `for update` 关键字，也可以自己实现悲观/乐观锁来达到目的；
- **基于 Zookeeper 有序节点**：Zookeeper 允许临时创建有序的子节点，这样客户端获取节点列表时，就能够当前子节点列表中的序号判断是否能够获得锁；
- **基于 Redis 的单线程**：由于 Redis 是单线程，所以命令会以串行的方式执行，并且本身提供了像 `SETNX(set if not exists)` 这样的指令，本身具有互斥性；

每个方案都有各自的优缺点，例如 MySQL 虽然直观理解容易，但是实现起来却需要额外考虑 **锁超时**、**加事务** 等，并且性能局限于数据库。

## redisson

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还实现了**可重入锁**（Reentrant Lock）、**公平锁**（Fair Lock）、**联锁**（MultiLock）、 **红锁**（RedLock）、 **读写锁**（ReadWriteLock）等，还提供了许多分布式服务。Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离（Separation of Concern），从而让使用者能够将精力更集中地放在处理业务逻辑上。

Redisson分布式锁的实现原理：

![image-20200520185315679](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200520185315679.png)

### watch dog自动延期机制

线程进来获得锁后，线程一切正常并没有宕机，但它的业务逻辑需要执行N秒（大于超时时间），这时锁就自动过期了。如果开启了**watch dog**机制，则会不断的延长锁的时间，避免锁自动失效。

正常情况下是不开启watch dog机制，因为对整体性能有一定的影响。

## 源码分析

[参考](https://www.jianshu.com/p/91b9f7569300)

### 加锁：tryLock

![image-20200520224034452](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200520224034452.png)

```java
    @Override
    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        //取得最大等待时间
        long time = unit.toMillis(waitTime);
        //记录下当前时间
        long current = System.currentTimeMillis();
        //取得当前线程id（判断是否可重入锁的关键）
        long threadId = Thread.currentThread().getId();
        //1.尝试申请锁，返回还剩余的锁过期时间
        // tryAcquire 内部通过调用 tryLockInnerAsync 实现申请锁的逻辑。申请锁并返回锁有效期还剩		    // 余的时间，如果为空说明锁未被其它线程申请则直接获取并返回，如果获取到时间，则进入等待竞争逻辑。
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        //2.如果为空，表示申请锁成功
        if (ttl == null) {
            return true;
        }
        //3.申请锁的耗时如果大于等于最大等待时间，则申请锁失败
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            /**
             * 通过 promise.trySuccess 设置异步执行的结果为null
             * Promise从Uncompleted-->Completed ,通知 Future 异步执行已完成
             */
            acquireFailed(threadId);
            return false;
        }
        
        current = System.currentTimeMillis();

        /**
         * 4.订阅锁释放事件，并通过await方法阻塞等待锁释放，有效的解决了无效的锁申请浪费资源的问题：
         * 基于信号量，当锁被其它资源占用时，当前线程通过 Redis 的 channel 订阅锁的释放事件，一旦锁释放会发消息通知待等待的线程进行竞争
         * 当 this.await返回false，说明等待时间已经超出获取锁最大等待时间，取消订阅并返回获取锁失败
         * 当 this.await返回true，进入循环尝试获取锁
         */
        RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
        //await 方法内部是用Semaphore来实现阻塞，获取subscribe异步执行的结果（应用了Netty 的 Future）
        if (!await(subscribeFuture, time, TimeUnit.MILLISECONDS)) {
            if (!subscribeFuture.cancel(false)) {
                subscribeFuture.onComplete((res, e) -> {
                    if (e == null) {
                        unsubscribe(subscribeFuture, threadId);
                    }
                });
            }
            acquireFailed(threadId);
            return false;
        }

        try {
            //计算获取锁的总耗时，如果大于等于最大等待时间，则获取锁失败
            time -= System.currentTimeMillis() - current;
            if (time <= 0) {
                acquireFailed(threadId);
                return false;
            }

            /**
             * 5.收到锁释放的信号后，在最大等待时间之内，循环一次接着一次的尝试获取锁
             * 获取锁成功，则立马返回true，
             * 若在最大等待时间之内还没获取到锁，则认为获取锁失败，返回false结束循环
             */
            while (true) {
                long currentTime = System.currentTimeMillis();
                // 再次尝试申请锁
                ttl = tryAcquire(leaseTime, unit, threadId);
                // 成功获取锁则直接返回true结束循环
                if (ttl == null) {
                    return true;
                }

                //超过最大等待时间则返回false结束循环，获取锁失败
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(threadId);
                    return false;
                }

                /**
                 * 6.阻塞等待锁（通过信号量(共享锁)阻塞,等待解锁消息）：
                 */
                currentTime = System.currentTimeMillis();
                if (ttl >= 0 && ttl < time) {
                    //如果剩余时间(ttl)小于wait time ,就在 ttl 时间内，从Entry的信号量获取一个许可(除非被中断或者一直没有可用的许可)。
                    getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    //则就在wait time 时间范围内等待可以通过信号量
                    getEntry(threadId).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
                }

                //7.更新剩余的等待时间(最大等待时间-已经消耗的阻塞时间)
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(threadId);
                    return false;
                }
            }
        } finally {
            //7.无论是否获得锁,都要取消订阅解锁消息
            unsubscribe(subscribeFuture, threadId);
        }
    }

```

```java
    // tryAcquire内部调用该方法获取锁
	<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        /**
         * 通过 EVAL 命令执行 Lua 脚本获取锁，保证了原子性
         */
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  // 1.如果缓存中的key不存在，则执行 hset 命令(hset key UUID+threadId 1),然后通过 pexpire 命令设置锁的过期时间(即锁的租约时间)
                  // 返回空值 nil ，表示获取锁成功
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                   // 如果key已经存在，并且value也匹配，表示是当前线程持有的锁，则执行 hincrby 命令，重入次数加1，并且设置失效时间
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                   //如果key已经存在，但是value不匹配，说明锁已经被其他线程持有，通过 pttl 命令获取锁的剩余存活时间并返回，至此获取锁失败
                  "return redis.call('pttl', KEYS[1]);",
                   //这三个参数分别对应KEYS[1]，ARGV[1]和ARGV[2]
                   Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
```

### 解锁：unlock

![image-20200520224447234](https://520li.oss-cn-hangzhou.aliyuncs.com/img/image-20200520224447234.png)

```java
    @Override
    public void unlock() {
        try {
            // get方法利用是 CountDownLatch 在异步调用结果返回前将当前线程阻塞，然后通过 Netty 的 FutureListener 在异步调用完成后解除阻塞，并返回调用结果。
            get(unlockAsync(Thread.currentThread().getId()));
        } catch (RedisException e) {
            if (e.getCause() instanceof IllegalMonitorStateException) {
                throw (IllegalMonitorStateException) e.getCause();
            } else {
                throw e;
            }
        }
    }

    protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        /**
         * 通过 EVAL 命令执行 Lua 脚本获取锁，保证了原子性
         */
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                //如果分布式锁存在，但是value不匹配，表示锁已经被其他线程占用，无权释放锁，那么直接返回空值（解铃还须系铃人）
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                 //如果value匹配，则就是当前线程占有分布式锁，那么将重入次数减1
                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                 //重入次数减1后的值如果大于0，表示分布式锁有重入过，那么只能更新失效时间，还不能删除
                "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                "else " +
                 //重入次数减1后的值如果为0，这时就可以删除这个KEY，并发布解锁消息，返回1
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; "+
                "end; " +
                "return nil;",
                //这5个参数分别对应KEYS[1]，KEYS[2]，ARGV[1]，ARGV[2]和ARGV[3]
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));

    }
```

### 基于Future获取值：get

```java
    @Override
    public <V> V get(RFuture<V> future) {
        if (!future.isDone()) {   //任务还没完成
            // 设置一个单线程的同步控制器
            CountDownLatch l = new CountDownLatch(1);
            future.onComplete((res, e) -> {
                //操作完成时，唤醒在await()方法中等待的线程
                l.countDown();
            });

            boolean interrupted = false;
            while (!future.isDone()) {
                try {
                    //阻塞等待
                    l.await();
                } catch (InterruptedException e) {
                    interrupted = true;
                    break;
                }
            }

            if (interrupted) {
                Thread.currentThread().interrupt();
            }
        }
        
        if (future.isSuccess()) {
            return future.getNow();
        }

        throw convertException(future);
    }
```

### 解锁消息处理

```java
public class LockPubSub extends PublishSubscribe<RedissonLockEntry> {

    public static final Long UNLOCK_MESSAGE = 0L;
    public static final Long READ_UNLOCK_MESSAGE = 1L;

    public LockPubSub(PublishSubscribeService service) {
        super(service);
    }
    
    @Override
    protected RedissonLockEntry createEntry(RPromise<RedissonLockEntry> newPromise) {
        return new RedissonLockEntry(newPromise);
    }

    @Override
    protected void onMessage(RedissonLockEntry value, Long message) {

        /**
         * 判断是否是解锁消息
         */
        if (message.equals(UNLOCK_MESSAGE)) {
            Runnable runnableToExecute = value.getListeners().poll();
            if (runnableToExecute != null) {
                runnableToExecute.run();
            }

            /**
             * 释放一个信号量，唤醒等待的entry.getLatch().tryAcquire去再次尝试申请锁
             */
            value.getLatch().release();
        } else if (message.equals(READ_UNLOCK_MESSAGE)) {
            while (true) {
                /**
                 * 如果还有其他Listeners回调，则也唤醒执行
                 */
                Runnable runnableToExecute = value.getListeners().poll();
                if (runnableToExecute == null) {
                    break;
                }
                runnableToExecute.run();
            }

            value.getLatch().release(value.getLatch().getQueueLength());
        }
    }

}

```

### 总结

通过 Redisson 实现分布式可重入锁，比纯自己通过`set key value px milliseconds nx +lua` 实现的效果更好些，虽然基本原理都一样，因为通过分析源码可知，RedissonLock是可重入的，并且**考虑了失败重试**，可以**设置锁的最大等待时间**， 在实现上也做了一些优化，**减少了无效的锁申请，提升了资源的利用率**。

需要特别注意的是，RedissonLock **同样没有解决节点挂掉的时候，存在丢失锁的风险的问题**。而现实情况是有一些场景无法容忍的，所以 Redisson 提供了实现了redlock算法的 RedissonRedLock，**RedissonRedLock 真正解决了单点失败的问题，代价是需要额外的为 RedissonRedLock 搭建Redis环境**。

**所以，如果业务场景可以容忍这种小概率的错误，则推荐使用 RedissonLock， 如果无法容忍，则推荐使用 RedissonRedLock。**

## redLock算法

在分布式版本的算法里我们假设我们有N个Redis master节点，**这些节点都是完全独立的**，我们不用任何复制或者其他隐含的分布式协调机制。之前我们已经描述了在Redis单实例下怎么安全地获取和释放锁。我们**确保将在每（N)个实例上使用此方法获取和释放锁**。为了取到锁，客户端应该执行以下操作:

- 获取当前Unix时间，以毫秒为单位。
- 依次尝试从N个实例，**使用相同的key和具有唯一性的value（例如UUID）获取锁**。当向Redis请求获取锁时，客户端应该设置一个尝试从某个Reids实例获取锁的**最大等待时间**（超过这个时间，则立马询问下一个实例），这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。
- 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁消耗的时间。**当且仅当从大多数（N/2+1，一半以上）的Redis节点都取到锁，并且使用的总耗时小于锁失效时间时，锁才算获取成功**。
- 如果取到了锁，key的真正有效时间 = 有效时间（获取锁时设置的key的自动超时时间） - 获取锁的总耗时（询问各个Redis实例的总耗时之和）。
- 如果因为某些原因，**最终获取锁失败**（即没有在至少 “N/2+1 ”个Redis实例取到锁或者“获取锁的总耗时”超过了“有效时间”），**客户端应该在所有的Redis实例上进行解锁**（即便某些Redis实例根本就没有加锁成功，这样可以防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）。