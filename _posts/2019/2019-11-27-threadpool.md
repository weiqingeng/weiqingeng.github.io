---
layout: post
title:  深入剖析ThreadPool线程池
category: java
tags: java
excerpt: 资源共享
---

# 前言
  线程池是java人员在工作中经遇到的一个技术，也是一个技术难点，最近遇到一个生产环境线程池使用问题，今天特针对源码和平时的工作经验，对ThreadPoolExecutor进行一个全面的剖析


# 1.问题
## 1.1 业务场景  
业务端直连oss上传的文件结果值，oss会回调到业务服务，然后业务服务处理数据后，将处理后的数据发送到mq供业务端消费以便得到文件上传后的结果值
## 1.2 问题现象
业务端反馈，上传文件后拿不到异步回调结果，请求都卡在等待结果值，全公司一片沸腾。。。 
## 1.3 排查现场
接到反馈后，果断到生产环境上去排查，查看一下日志，我的乖乖，大量的堆栈异常，如下：
```
2019-11-25 14:05:12,722 ERROR ctr.CallbackController - /file-system/callback, error:{} 
java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.FutureTask@4b1c9707 rejected from java.util.concurrent.ThreadPoolExecutor@428d700e[Running, pool size = 4, active threads = 4, queued tasks = 10000, completed tasks = 6618871]
	at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063) ~[?:1.8.0_151]
	at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830) ~[?:1.8.0_151]
	at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1379) ~[?:1.8.0_151]

```
大致的意思是：线程池线程数4，激活线程数4，queued tasks =
1000，目前queue已经满了，新的任务被拒绝丢弃，抛出RejectedExecutionException这个运行时异常...

## 1.4 找原因
问题一出，马上想到线程池，查看源码的线程池配置，核心线程数 = cpu * 2， 最大线程数
= CPU * 2，最大任务堵塞队列10000， 查看了生产环境的cpu个数 = 2，当时该台服务器的
QPS 大概在
300，整体核算，发现确实是线程数设置过小，又加上使用的默认的拒绝策略是AbortPolicy，所以问题基本上可以确认

```java
public class ThreadPoolUtil {
    // 线程总数为 内核数的 2倍
    private static int threadNum = Runtime.getRuntime().availableProcessors() * 2;

    // 创建固定线程数的线程池，堵塞队列的最大容量为1万
    public static ExecutorService executor = new ThreadPoolExecutor(threadNum, threadNum,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(10000));
}



    /**
     * The default rejected execution handler
     */
    private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
    
    
    /**
     * A handler for rejected tasks that throws a
     * {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

```

## 解决问题
先下线服务器，增加cpu，然后上线(初步解决)  
调优线程数  
修改拒绝策略，该业务场景是不允许丢数据，所以拒绝策略变更为CallerRunsPolicy，降低阻塞队列容量  
将线程池的方式更换为mq


# 2. 线程池

## 2.1 Executor

Executor是在JDK1.5的java.util.concurrent 并发包中引入的，作者就是我们的大牛
Doug Lea，该接口中只有一个方法void execute(Runnable
command);因此也可以称为函数式接口，对execute方法的解释：在将来的某个时候执行给定的命令。该命令可以在新线程、池化线程或调用线程中执行，具体由执行程序实现决定。
咋一看，貌似很神秘，执行的时机都是未知的，那么我们该如何使用线程池呢？接下来就一步一步来剖析
```java
/* 
 * @since 1.5
 * @author Doug Lea
 */
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

## 2.2 ExecutorService
ExecutorService也是在JDK1.5的java.util.concurrent
并发包中引入的，是Executor的子接口，对Executor的方法进行扩充，里面新增了关于线程池的关闭诸多方法，后面会详细讲解

```java
/* 
 * @since 1.5
 * @author Doug Lea
 * 代码有整理，去除了大量的注释
 */
public interface ExecutorService extends Executor {
    
    void shutdown();
    
    List<Runnable> shutdownNow();

    boolean isShutdown();
    
    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    
    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);
    
    Future<?> submit(Runnable task);
    
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

   
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

## 2.3 AbstractExecutorService
AbstractExecutorService是ExecutorService的抽象实现类，所谓抽象，也就是不让实例化，要使用它，思路应该是实现子类，进行实例化，接下来就是我们的今天的主角之一(ThreadPoolExecutor)即将登场

```java
/*
 * @since 1.5
 * @author Doug Lea
 */
public abstract class AbstractExecutorService implements ExecutorService {
    
}
```



## 2.4 Executors
为了帮助大家更快的使用线程池，jdk提供了Executors这个工具类，里面封装了很多静态的方法来帮助大家快速的的形成和使用线程池，对于测试环境考验帮助我们更快了解线程池，不过生成环境不推荐，
查看下面的源码，大家可以发现，各个方法都是new ThreadPoolExecutor()不同的构造器，
问题： 
FixedThreadPool和SingleThreadExecutor =>
允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而引起OOM异常  

CachedThreadPool =>
允许创建的线程数为Integer.MAX_VALUE，可能会创建大量的线程，从而引起OOM异常  

所以，生产环境都要求自己手动new
ThreadPoolExecutor()，设置合理的线程数，设置有界LinkedBlockingQueue，设置合理的任务拒绝策略

```java
/**
 *
 * @since 1.5
 * @author Doug Lea
 */
public class Executors {

    
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    
    public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool
            (parallelism,
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }

   
    public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }

    
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }

   
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

   
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }

    
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

   
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }

   
    public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }

   
    public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1, threadFactory));
    }

   
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

   
    public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }

}
```

## 2.5 ThreadPoolExecutor
2.4
中已经看到Executors其实底层就是对ThreadPoolExecutor的封装，接下来我们就来分析下ThreadPoolExecutor，下面摘取了ThreadPoolExecutor里面几个比较核心的constructors，
ThreadPoolExecutor的构造函数共有四个，但最终调用的都是同一个

```java
 /*
  * @since 1.5
  * @author Doug Lea
  **/
 public class ThreadPoolExecutor extends AbstractExecutorService {

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

  
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }

  
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
}
```
上面源码可以看到构造器中几个核心的参数，解释如下： 

corePoolSize  
线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。有些项目很闲，但是也不能把人都撤了，至少要留 corePoolSize 个人坚守阵地。

maximumPoolSize  
线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumPoolSize；当项目很忙时，就需要加人，但是也不能无限制地加，最多就加到 maximumPoolSize 个人。当项目闲下来时，就要撤人了，最多能撤到 corePoolSize 个人。

keepAliveTime  
线程空闲时的存活时间，即当线程没有任务执行时，继续存活的时间；默认情况下，该参数只在线程数大于corePoolSize时才有用；上面提到项目根据忙闲来增减人员，那在编程世界里，如何定义忙和闲呢？很简单，一个线程如果在一段时间内，都没有执行任务，说明很闲，keepAliveTime 和 unit 就是用来定义这个“一段时间”的参数。也就是说，如果一个线程空闲了keepAliveTime & unit这么久，而且线程池的线程数大于 corePoolSize ，那么这个空闲的线程就要被回收了。

unit  
keepAliveTime的单位；

workQueue  
用来保存等待被执行的任务的阻塞队列，且任务必须实现Runnable接口，
在JDK中提供了如下阻塞队列：  
1、ArrayBlockingQueue：基于数组结构的有界阻塞队列，按FIFO排序任务；  
2、LinkedBlockingQueue：基于链表结构的阻塞队列，按FIFO排序任务，
吞吐量通常要高于ArrayBlockingQueue；  
3、SynchronousQueue：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，
吞吐量通常要高于LinkedBlockingQueue；  
4、priorityBlockingQueue：具有优先级的无界阻塞队列；

ThreadFactory threadFactory  
创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名。

RejectedExecutionHandler handler  
通过这个参数你可以自定义任务的拒绝策略。如果线程池中所有的线程都在忙碌，并且工作队列也满了（前提是工作队列是有界队列），那么此时提交任务，线程池就会拒绝接收。至于拒绝的策略，你可以通过 handler 这个参数来指定。
ThreadPoolExecutor 已经提供了以下 4 种策略。  
* CallerRunsPolicy：提交任务的线程自己去执行该任务。
* AbortPolicy：默认的拒绝策略，会 throws RejectedExecutionException。
* DiscardPolicy：直接丢弃任务，没有任何异常抛出。
* DiscardOldestPolicy：丢弃最老的任务，其实就是把最早进入工作队列的任务丢弃，然后把新任务加入到工作队列。
线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略：
1、AbortPolicy：直接丢弃任务并抛出异常，默认策略； 
```java
     /**
     * A handler for rejected tasks that throws a {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
``` 
2、CallerRunsPolicy：用调用者所在的线程来执行任务；
```java
/**
     * A handler for rejected tasks that runs the rejected task
     * directly in the calling thread of the {@code execute} method,
     * unless the executor has been shut down, in which case the task
     * is discarded.
     */
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```  
3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，在丢弃前尝试线程池执行当前任务；
```java
/**
     * A handler for rejected tasks that discards the oldest unhandled
     * request and then retries {@code execute}, unless the executor
     * is shut down, in which case the task is discarded.
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```
 
4、DiscardPolicy：直接丢弃任务，不抛出异常；  
当然也可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。
```java
/**
     * A handler for rejected tasks that silently discards the
     * rejected task.
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```  

5、自定义一个reject策略；  
比如：如果线程池无法执行更多的任务了，此时建议你可以把这个任务信息持久化写入磁盘或者写入mq中间件等等，后台专门启动一个线程，后续等待你的线程池的工作负载降低了，他可以慢慢的从磁盘里读取之前持久化的任务，重新提交到线程池里去执行


图片展示了线程池执行任务逻辑和线程池参数的关系：
![](/assets/images/2019/threadpool/threadpool.png)

![](/assets/images/2019/threadpool/threadpool-theory.png)

解释说明： 这里以核心线程数=4， 最大线程数=8， 做大阻塞队列=4为例
1. 线程池刚初始化的时候，阻塞队列是空的，线程池的线程数也是空的
2. 当有任务提交过来，线程池就开始创建一个核心线程去处理任务
3. 当有很多任务提交过来，线程池就一直创建核心线程去处理，直到核心线程数达到4(corePoolSize)
4. 如果还有任务过来，而核心线程数达到最大值4，没有空闲的线程来处理当前的任务，因此，就会把当前任务加入阻塞队列
5. 当阻塞队列满了，线程池就开始创建额外的线程(最大线程数-核心线程数)去处理队列里的任务
6. 当线程池的线程数达到最大值8(maximumPoolSize)，线程池就不再创建线程，如果此时还有大量的任务过来，阻塞队列也满了，就会触发丢弃策略，如果线程处理的过来，在空闲时间(keepAliveTime)中，额外线程都没有任务需要处理，额外线程就会被销毁，注意是额外的线程(最大线程 - 核心线程)，不是核心线程

## 2.6 如何正确的关闭线程池
上面讲解了线程池的使用，另外有人可能会问，那么要怎么去优雅关闭一个线程池呢？
要解决这个问题首先让我了解下线程池的几个运行状态，看源码：

### 2.6.1 运行状态
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
}
```
运行状态之间的关系
![](/assets/images/2019/microservice/threadpool/runstate.png)



### 2.6.2 线程池关闭的方法 ThreadPoolExecutor#shutdown()

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
/**
     * Initiates an orderly shutdown in which previously submitted
     * tasks are executed, but no new tasks will be accepted.
     * Invocation has no additional effect if already shut down.
     *
     * <p>This method does not wait for previously submitted tasks to
     * complete execution.  Use {@link #awaitTermination awaitTermination}
     * to do that.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // If there is a security manager, makes sure caller has permission to shut down threads in general (see shutdownPerm). If this passes, additionally makes sure the caller is allowed to interrupt each worker thread. This might not be true even if first check passed, if the SecurityManager treats some threads specially.
            // 如果有安全管理器，请确保调用者通常具有关闭线程的权限(请参阅shutdownPerm)。如果通过了，还要确保调用者可以中断每个工作线程。即使第一次检查通过，如果SecurityManager专门处理一些线程，这也可能不是真的。
            checkShutdownAccess();
            // Transitions runState to given target, or leaves it alone if already at least the given target.
            // 将运行状态转换为给定的目标，或者如果至少已经有了给定的目标，则不进行转换。
            advanceRunState(SHUTDOWN);
            
            // Common form of interruptIdleWorkers, to avoid having to remember what the boolean argument means.
            interruptIdleWorkers();
            
            //Performs any further cleanup following run state transition on invocation of shutdown. A no-op here, but used by ScheduledThreadPoolExecutor to cancel delayed tasks.
            // 在关闭调用时执行运行状态转换后的任何进一步清理，该方法里面没有任何实现，但是子类ScheduledThreadPoolExecutor使用它来取消延迟的任务。
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
}
```
shutdown()首先加锁，其次先检查系统安装状态，接着就会将线程池运行状态变更为
SHUTDOWN，在这之后线程池不再接受提交的新任务。此时如果还继续往线程池提交任务，
将会使用线程池拒绝策略响应，默认情况下将会使用
ThreadPoolExecutor.AbortPolicy，抛出 RejectedExecutionException 异常。
interruptIdleWorkers()
方法只会中断空闲的线程，不会中断正在执行任务的的线程。空闲的线程将会阻塞在线程池的阻塞队列上。


### 2.6.3 线程池关闭的方法 ThreadPoolExecutor#shutdownNow()

```java
 public class ThreadPoolExecutor extends AbstractExecutorService {
 /**
     * Attempts to stop all actively executing tasks, halts the
     * processing of waiting tasks, and returns a list of the tasks
     * that were awaiting execution. These tasks are drained (removed)
     * from the task queue upon return from this method.
     *
     * <p>This method does not wait for actively executing tasks to
     * terminate.  Use {@link #awaitTermination awaitTermination} to
     * do that.
     *
     * <p>There are no guarantees beyond best-effort attempts to stop
     * processing actively executing tasks.  This implementation
     * cancels tasks via {@link Thread#interrupt}, so any task that
     * fails to respond to interrupts may never terminate.
     *
     * @throws SecurityException {@inheritDoc}
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            // 中断所有线程
            interruptWorkers();
            // 丢弃队列中的任务，并将任务放入list并返回
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
}
```
shutdownNow()将会把线程池状态设置为STOP，然后中断所有线程，最后取出工作队列中所有未完成的任务返回给调用者。

对比shutdown 方法，shutdownNow
方法比较粗暴，直接中断工作线程。不过这里需要注意，中断线程并不代表线程立刻结束。这里需要线程主动配合线程中断响应。

### 2.6.4 ThreadPoolExecutor#awaitTermination()
等待指定的时长再中断线程
```java
 public class ThreadPoolExecutor extends AbstractExecutorService {
    
    public boolean awaitTermination(long timeout, TimeUnit unit)throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
 }   
```

线程池 shutdown 与 shutdownNow 方法都不会主动等待执行任务的结束，如果需要等到线程池任务执行结束，需要调用 awaitTermination 主动等待任务调用结束。

调用方法如下： 

```
        threadPool.shutdown();
        try {
            while (!threadPool.awaitTermination(60, TimeUnit.SECONDS)){
                System.out.println("线程池任务还未执行结束");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
```

如果线程池任务执行结束，awaitTermination 方法将会返回
true，否则当等待时间超过指定时间后将会返回
false。如果需要使用这种进制，建议在上面的基础上增加一定重试次数。这个真的很重要！！！


05、优雅关闭线程池
回顾上面线程池状态关系图，我们可以知道处于 SHUTDOWN 的状态下的线程池依旧可以调用 shutdownNow。所以我们可以结合 shutdown ， shutdownNow，awaitTermination ，更加优雅关闭线程池。

        threadPool.shutdown(); // Disable new tasks from being submitted
        // 设定最大重试次数
        try {
            // 等待 60 s
            if (!threadPool.awaitTermination(60, TimeUnit.SECONDS)) {
                // 调用 shutdownNow 取消正在执行的任务
                threadPool.shutdownNow();
                // 再次等待 60 s，如果还未结束，可以再次尝试，或者直接放弃
                if (!threadPool.awaitTermination(60, TimeUnit.SECONDS))
                    System.err.println("线程池任务未正常执行结束");
            }
        } catch (InterruptedException ie) {
            // 重新调用 shutdownNow
            threadPool.shutdownNow();
        }


## 3.线程池的线程数怎么定义?
github的一片博客： 
https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing


创建多少线程合适，要看多线程具体的应用场景。我们的程序一般都是 CPU 计算和 I/O
操作交叉执行的，由于 I/O 设备的速度相对于 CPU 来说都很慢，所以大部分情况下，I/O
操作执行的时间相对于 CPU 计算来说都非常长，这种场景我们一般都称为 I/O
密集型计算；和 I/O 密集型计算相对的就是 CPU 密集型计算了，CPU
密集型计算大部分场景下都是纯 CPU 计算。I/O 密集型程序和 CPU
密集型程序，计算最佳线程数的方法是不同的。    


下面我们对这两个场景分别说明。


对于 CPU密集型计算，多线程本质上是提升多核 CPU 的利用率，所以对于一个 4 核的
CPU，每个核一个线程，理论上创建 4
个线程就可以了，再多创建线程也只是增加线程切换的成本。所以，**对于 CPU
密集型的计算场景，理论上“线程的数量 =CPU
核数”就是最合适的**。不过在工程上，**线程的数量一般会设置为“CPU 核数
+1”**，这样的话，当线程因为偶尔的内存页失效或其他原因导致阻塞时，这个额外的线程可以顶上，从而保证
CPU 的利用率。   
 
对于 I/O 密集型的计算场景，比如前面我们的例子中，如果 CPU 计算和
I/O 操作的耗时是 1:1，那么 2 个线程是最合适的。如果 CPU 计算和 I/O 操作的耗时是
1:2，那多少个线程合适呢？是 3 个线程，如下图所示：CPU 在 A、B、C
三个线程之间切换，对于线程 A，当 CPU 从 B、C 切换回来时，线程 A 正好执行完 I/O
操作。这样 CPU 和 I/O 设备的利用率都达到了 100%。

![](/assets/images/2019/microservice/threadpool/thread.png)
三线程执行示意图 

通过上面这个例子，我们会发现，对于 I/O
密集型计算场景，最佳的线程数是与程序中 CPU 计算和 I/O
操作的耗时比相关的，我们可以总结出这样一个公式：
```java   
最佳线程数 =1 +（I/O 耗时 / CPU 耗时）
```

我们令 R=I/O 耗时 / CPU 耗时，综合上图，可以这样理解：当线程 A 执行 IO
操作时，另外 R 个线程正好执行完各自的 CPU 计算。这样 CPU 的利用率就达到了
100%。  

不过上面这个公式是针对单核 CPU 的，至于多核
CPU，也很简单，只需要等比扩大就可以了，计算公式如下：
```java     
最佳线程数 = CPU 核数 *[ 1 +（I/O 耗时 / CPU 耗时）] 
```

总结：   
很多人都知道线程数不是越多越好，但是设置多少是合适的，却又拿不定主意。其实只要把握住一条原则就可以了，
这条原则就是**将硬件的性能发挥到极致**。上面我们针对 CPU 密集型和 I/O
密集型计算场景都给出了理论上的最佳公式，
这些公式背后的目标其实就是**将硬件的性能发挥到极致**。对于 I/O 密集型计算场景，I/O
耗时和 CPU 耗时的比值是一个关键参数，不幸的是这个参数是未知的，
而且是动态变化的，所以工程上，我们要估算这个参数，然后做各种不同场景下的压测来验证我们的估计。不过工程上，原则还是**将硬件的性能发挥到极致**，所以压测时，我们需要重点关注
CPU、I/O 设备的利用率和性能指标（响应时间、吞吐量）之间的关系。

```
 I/O 耗时 / CPU 耗时可以通过 apm工具进行计算，最终以压测值为准。
 ```

## 常见的问题

1. 如果线上机器突然宕机，线程池的阻塞队列中的请求怎么办？
宕机必然会导致线程池里的积压的任务实际上来说都是会丢失的，如果说你要提交一个任务到线程池里去，在提交之前，
麻烦你先在数据库(Redis)里插入这个任务的信息，更新他的状态：未提交、已提交、已完成。提交成功之后，更新他的状态是已提交状态。
系统重启，后台线程去扫描数据库里的未提交和已提交状态的任务，可以把任务的信息读取出来，重新提交到线程池里去，继续进行执行 