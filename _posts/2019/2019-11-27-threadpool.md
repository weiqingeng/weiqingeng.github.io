---
layout: post
title:  深入剖析ThreadPool线程池
no-post-nav: true
category: it
tags: [it]
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
线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。

maximumPoolSize  
线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumPoolSize；

keepAliveTime  
线程空闲时的存活时间，即当线程没有任务执行时，继续存活的时间；默认情况下，该参数只在线程数大于corePoolSize时才有用；

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

图片展示了线程池执行任务逻辑和线程池参数的关系：
![](/assets/images/2019/microservice/threadpool/threadpool.png)



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
       