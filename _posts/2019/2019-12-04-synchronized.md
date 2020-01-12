---
layout: post
title: 深入剖析Java并发之Synchronized锁
category: redis
tags: [redis]
excerpt: 深入剖析Synchronized锁，Java并发，java 锁，Synchronized锁
---

# 前言

并发编程这个技术领域已经发展了半个世纪了，相关的理论和技术纷繁复杂。那有没有一种核心技术可以很方便地解决我们的并发问题呢？  

有，那就是管程技术(Monitor)！

# 定义
所谓**管程，指的是管理共享变量以及对共享变量的操作过程，让他们支持并发。**翻译为 Java 领域的语言，就是管理类的成员变量和成员方法，让这个类是线程安全的。

在管程的发展史上，先后出现过三种不同的管程模型，分别是：Hasen 模型、Hoare 模型和 MESA 模型。其中，现在广泛应用的是 MESA 模型，并且 Java 管程的实现参考的也是 MESA 模型。

在并发编程领域，有两大核心问题：一个是互斥，即同一时刻只允许一个线程访问共享资源；另一个是同步，即线程之间如何通信、协作。这两大问题，管程都是能够解决的。


**管程如何解决互斥问题**:  思路很简单，就是将共享变量及其对共享变量的操作统一封装起来。在下图中，管程 X 将共享变量 queue 这个队列和相关的操作入队 enq()、出队 deq() 都封装起来了；线程 A 和线程 B 如果想访问共享变量 queue，只能通过调用管程提供的 enq()、deq() 方法来实现；enq()、deq() 保证互斥性，只允许一个线程进入管程。不知你有没有发现，管程模型和面向对象高度契合的。估计这也是 Java 选择管程的原因吧。而我在前面章节介绍的互斥锁用法，其背后的模型其实就是它。

![](/assets/images/2019/synchronized/monitor.png)


**管程解决线程间的同步问题呢**？这个就比较复杂了，不过你可以借鉴一下我们曾经提到过的就医流程，它可以帮助你快速地理解这个问题。为进一步便于你理解，在下面，我展示了一幅 MESA 管程模型示意图，
它详细描述了 MESA 模型的主要组成部分。在管程模型里，共享变量和对共享变量的操作是被封装起来的，图中最外层的框就代表封装的意思。框的上面只有一个入口，并且在入口旁边还有一个入口等待队列。
当多个线程同时试图进入管程内部时，只允许一个线程进入，其他线程则在入口等待队列中等待。这个过程类似就医流程的分诊，只允许一个患者就诊，其他患者都在门口等待。管程里还引入了条件变量的概念，
而且每个条件变量都对应有一个等待队列，如下图，条件变量 A 和条件变量 B 分别都有自己的等待队列。那条件变量和等待队列的作用是什么呢？其实就是解决线程同步问题

![](/assets/images/2019/synchronized/mesa.png)

    // 获取锁，想redis发送一段lua脚本
   <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "return redis.call('pttl', KEYS[1]);",
                  // 对应 KEYS[1],ARGV[1],ARGV[2]3个参数的值
                  Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }  
```

**获取锁其实是向redis发送一段lua脚本，封装在lua脚本中发送给redis，保证这段复杂业务逻辑执行的原子性。**

lua脚本解释：   

KEYS[1]代表的是你加锁的那个key，比如：RLock lock =
redisson.getLock("lockKey"); 则“lockKey”就是加锁的key   

ARGV[1]代表的就是锁key的默认生存时间，默认30秒。   

ARGV[2]代表的是加锁的客户端的ID，类似于下面这样：328d90c0-ac3d-4ca1-a366-fd4e80bc4a6d:67

第一个if判断语，就是用“exists
lockKey”命令判断，true代表没有加锁，可以进行加锁操作，则进行如下操作：
1. 设置值 ` hset lockKey 328d90c0-ac3d-4ca1-a366-fd4e80bc4a6d:67 1 ` 
2. 给key设置失效时间，“pexpire lockKey
   30000”命令，设置lockKey这个锁key的生存时间是30秒
3. 加锁成功后redis里面会增加如下的数据结构 

```
lockKey
{
  "328d90c0-ac3d-4ca1-a366-fd4e80bc4a6d:67":1
}
328d90c0-ac3d-4ca1-a366-fd4e80bc4a6d:67   uuid:线程id     67:当前key获取锁的总次数
```
如第一个if是false，则代锁已经存在，则执行第二个if判断(判断当前加锁成功的是否是自己)，true则进行如下操作：
1. 加锁总次数 + 1（可重入锁）
2. 重置失效时间

第二个if的结果值是false，则获取到pttl
lockKey返回的一个数字，这个数字代表了lockKeyke的
剩余生存时间。比如还剩3000毫秒。此时客户端会进入一个while循环，不停的尝试加锁。

对于单个的redis，我们能很好的知道上面的流程，但是如果Redis是master-slave或者cluster呢？锁数据会写到一个redis还是所有的redis，我们还是回到
**evalWriteAsync()源码实现**
```java
 
  @Override
    public <T, R> RFuture<R> evalWriteAsync(String key, Codec codec, RedisCommand<T> evalCommandType, String script, List<Object> keys, Object... params) {
        NodeSource source = getNodeSource(key);
        return evalAsync(source, false, codec, evalCommandType, script, keys, params);
    }
```
 通过源码可以看到，lua脚本在发现redis时做了一个NodeSource source =
 getNodeSource(key);操作，这个操作是干什么，还是看看源码
 
**getNodeSource(String key)**
 ```java
  private NodeSource getNodeSource(String key) {
        int slot = connectionManager.calcSlot(key);
        MasterSlaveEntry entry = connectionManager.getEntry(slot);
        return new NodeSource(entry, slot);
    }
```

**ClusterConnectionManager#calcSlot()**
```java
@Override
    public int calcSlot(String key) {
        if (key == null) {
            return 0;
        }

        int start = key.indexOf('{');
        if (start != -1) {
            int end = key.indexOf('}');
            key = key.substring(start+1, end);
        }

        int result = CRC16.crc16(key.getBytes()) % MAX_SLOT;
        log.debug("slot {} for {}", result, key);
        return result;
    }

```

**MasterSlaveConnectionManager#calcSlot()**
```java
  @Override
    public int calcSlot(String key) {
        return singleSlotRange.getStartSlot();
    }
```

**watch dog看门狗实现**
```java
    private void renewExpiration() {
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ee == null) {
            return;
        }
        
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ent == null) {
                    return;
                }
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                future.onComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getName() + " expiration", e);
                        return;
                    }
                    
                    if (res) {
                        // reschedule itself
                        renewExpiration();
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        
        ee.setTimeout(task);
    }
```
只要客户端加锁成功，就会启动一个watch dog看门狗，他是一个后台线程，会每隔10秒检查一下，如果客户端还持有锁key，那么就会不断的延长锁key的生存时间




* 获取锁重要结论
1. 源码已经很明显的说明了：**redisson分布式锁仅仅只是选择一台机器！**
而且针对Master-slave和cluster两种方式给出了选择node的方式。

2. 加锁方式：可重入加锁机制(redis.call('hincrby', KEYS[1], ARGV[2], 1))

3. watch dog定时(默认10s)给key延长失效时间


到此，获取锁就分析完毕。接下来就是释放锁。


# 释放锁源码分析

**unlock()**
```java
/**
     * Releases the lock.
     *
     * <p><b>Implementation Considerations</b>
     *
     * <p>A {@code Lock} implementation will usually impose
     * restrictions on which thread can release a lock (typically only the
     * holder of the lock can release it) and may throw
     * an (unchecked) exception if the restriction is violated.
     * Any restrictions and the exception
     * type must be documented by that {@code Lock} implementation.
     */
    void unlock();
    
     @Override
    public void unlock() {
        try {
            get(unlockAsync(Thread.currentThread().getId()));
        } catch (RedisException e) {
            if (e.getCause() instanceof IllegalMonitorStateException) {
                throw (IllegalMonitorStateException) e.getCause();
            } else {
                throw e;
            }
        }
    }
 ```
 
**unlockAsync(long threadId)**
 ```java   
    @Override
    public RFuture<Void> unlockAsync(long threadId) {
        RPromise<Void> result = new RedissonPromise<Void>();
        RFuture<Boolean> future = unlockInnerAsync(threadId);

        future.onComplete((opStatus, e) -> {
            if (e != null) {
                cancelExpirationRenewal(threadId);
                result.tryFailure(e);
                return;
            }

            if (opStatus == null) {
                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                        + id + " thread-id: " + threadId);
                result.tryFailure(cause);
                return;
            }
            
            cancelExpirationRenewal(threadId);
            result.trySuccess(null);
        });

        return result;
    }

```

**unlockInnerAsync(long threadId)**
```java 
   protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        // 向所有redis实例都执行如下命令
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                // // 如果分布式锁存在，但是value不匹配，表示锁已经被占用，那么直接返回
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                // 如果就是当前线程占有分布式锁，那么将重入次数减1
                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                // 重入次数减1后的值如果大于0，表示分布式锁有重入过，那么只设置失效时间，还不能删除
                "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                // 重入次数减1后的值如果为0，表示分布式锁只获取过1次，那么删除这个KEY，并发布解锁消息    
                "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; "+
                "end; " +
                "return nil;",
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));

    }
```

通过源码可知，释放锁也是一段lua脚本，和获取锁很类似，现根据key获取对应的redis node节点，然后判断当前持有锁的是否为自己，否，则直接退出，是则做如下操作：
1. 获取锁的次数 -1
2. 如果减1后总次数大于0，则重新设置key的失效时间，否则删除key，并且广播给其他的client节点，这样等待线程就可以尝试获取加锁
说白了，就是每次都对lockKey数据结构中的那个加锁次数减1。如果发现加锁次数是0了，说明这个客户端已经不再持有锁了，此时就会用：
“del lockKey”命令，从redis里删除这个key。广播给其他的节点删除key，因此的等待的客户端就可以尝试完成加锁了。


* Redis分布式锁的缺点

通过源码可以知道，加锁时只会选择一个节点，写入lockKey的value，然后会异步复制给 slave实例，如果在复制给slave之前master挂了，就会导致锁丢失，
接着redis主备切换，slave变为master，这样会导致，其他线程在新的master上完成加锁，而刚才的线程也认为自己加锁成功，导致一个key有两个线程加锁成功，产生各种脏数据。

所以redis cluster或redis master-slave架构的主从异步复制导致的redis分布式锁的最大缺陷：在redis master实例宕机的时候，可能导致多个线程同时完成加锁。

