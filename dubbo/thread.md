# 1. 什么是线程池？
通俗来讲是就是装有线程的池子，和我们使用到的各种连接池的概念类似，那么线程池解决了什么问题呢，来看下官方的阐述说明：
````	
  Thread pools address two different problems: they usually provide improved performance when     
  executing large numbers of asynchronous tasks, due to reduced per-task invocation overhead, 
  and they provide a means of bounding and managing the resources, including threads, consumed 
  when executing a collection of tasks.  Each {@code ThreadPoolExecutor} also maintains some basic 
  statistics, such as the number of completed tasks.

  翻译如下，线程池主要解决了两个问题：
  1. 通过减少调用开销，提高大量异步任务执行的性能。
  2. 可以管理和限制线程数量、任务数量等资源。
````
  首先线程池通过长期持有一组线程，避免了线程频繁的创建和销毁带来的额外开销。并且提供了丰富的工具方法来管理和限制线程池状态、资源等。

# 2. 初识线程池
## 2.1 简单的上手例子
````
public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(2, 2
                , 10, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(2),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy());
                
        threadPool.submit(() -> {
        	  // 执行的任务内容仅作为实例使用，没有任何实际意义。
            System.out.println("task has been executing");
        });

}
程序输出：task has been executing
````

第一步我们通过使用````ThreadPoolExecutor````构造方法定义一个拥有两个线程的线程池，并且成功了执行我们提交的任务。

## 2.2 剖析构造方法参数的含义
````ThreadPoolExecutor````的构造方法有以下四个：
![image.png](http://upload-images.jianshu.io/upload_images/3776621-6ba0061dbc5ffeb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
既然是剖析，当然是关注最下面那个参数最多的构造方法啦~，先来看看每个参数的意义以及它们作用。

````
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
  	... 省略中间无关代码
       
}
````
### corePoolSize
线程池里持有的核心线程的数量，即最小线程数量。额，除非设定了````allowCoreThreadTimeOut````为true，那么在指定时间内如果核心线程一直处于空闲状态，则会被终止。

### maximumPoolSize
线程池最大的线程数量。如果新的任务被添加，存放任务的队列已经满了，并且当前的线程数量小于````maximumPoolSize````数量，线程池将会主动的创建一个新的线程。

### keepAliveTime TimeUnit
两个组合使用的参数，用来控制线程池里线程最大的空闲时间上限。如果超出指定时间空闲的线程将会被终止，以释放资源。

### ThreadFactory
用来统一创建线程的工厂类，可以通过接口````ThreadFactory````自定义实现，看下默认使用的线程工厂类内容。

````
static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
````
通过````newThread````方法可以大至得知一下信息：指定了线程名，设定了优先级、设定为非后台线程，如果想修改成自定义值，可以参考使用````com.google.common.util.concurrent.ThreadFactoryBuilder````类。

### BlockingQueue
存放待执行任务队列，特性如下：

1. 任务先进先出
2. 阻塞，具体体现在队列为空时阻塞出队操作；队列满的时候阻塞入队操作。

下面的表格列举了````BlockingQueue````中不同方法在操作不能立刻完成时的行为。

|         | Throws exception | Special value |         Blocks |            Times out |
| ------- | ---------------: | ------------: | -------------: | -------------------: |
| Insert  |           add(e) |      offer(e) |         put(e) | offer(e, time, unit) |
| Remove  |         remove() |        poll() |         take() |     poll(time, unit) |
| Examine |        element() |        peek() | not applicable |       not applicable |
 ````ThreadPoolExecutor````在添加任务的时候使用的是````offer()````方法，如果队列满了则会返回一个false。取任务的时候使用到了````take()、poll()、poll(time, unit)````，具体使用的地方请看下面执行流程的分析。

常用的具体实现类分别是：

	1. ArrayBlockingQueue 有界队列，可以通过限定任务堆积数量
	2. LinkedBlockingQueue 无界队列，任务可以无限堆积，对执行时间不敏感的业务可以使用这个队列。
	3. SynchronousQueue 阻塞队列，不能缓冲任务。新任务进来的时候会一直阻塞到有空闲的线程去处理。

### RejectedExecutionHandler
用来处理不能被线程池执行任务。首先看下接口定义：

````
public interface RejectedExecutionHandler {

    /**
     * Method that may be invoked by a {@link ThreadPoolExecutor} when
     * {@link ThreadPoolExecutor#execute execute} cannot accept a
     * task.  This may occur when no more threads or queue slots are
     * available because their bounds would be exceeded, or upon
     * shutdown of the Executor.
     *
     * <p>In the absence of other alternatives, the method may throw
     * an unchecked {@link RejectedExecutionException}, which will be
     * propagated to the caller of {@code execute}.
     *
     * @param r the runnable task requested to be executed
     * @param executor the executor attempting to execute this task
     * @throws RejectedExecutionException if there is no remedy
     */
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
````
注释里说明了什么情况下任务不能被执行：1.没有多余的线程或者任务队列满了 2.线程池被关闭了

看下其子类实现的具体的拒绝策略有哪些:

1. ````CallerRunsPolicy````直接使用调用submit或者execute的当前线程来执行任务


````
public static void main(String[] args) throws InterruptedException {

    ThreadPoolExecutor threadPool = new ThreadPoolExecutor(1, 1
            , 10, TimeUnit.SECONDS,
            new ArrayBlockingQueue<Runnable>(1),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.CallerRunsPolicy());

    for (int i = 0; i < 3; i++) {
        threadPool.submit(() -> {
            System.out.println(Thread.currentThread().getName());
        });
    }

    Thread.sleep(200000000);
}

输出的结果：
main
pool-1-thread-1
pool-1-thread-1
````
这里定义了固定线程大小的线程池并且队列中只存放一个任务，所以for循环添加的第三个任务会被执行````CallerRunsPolicy````策略，进而会被当前调用submit()方法的线程执行，也就是主线程(main)。

2. ````AbortPolicy````核心代码如下：

````
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
````
比较直白，直接抛出非检查异常。

3. ````DiscardPolicy````直接忽略任务，什么也不干。
4. ````DiscardOldestPolicy````核心代码如下：
````
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
````
因为缓存任务是使用的队列(先进先出)，所以在执行````e.getQueue().poll();````的时候把存放时间最久的任务出队舍弃。下面看下验证示例：
````
public class Demo {

    public static void main(String[] args) throws InterruptedException {

        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(1, 1
                , 10, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(1),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.DiscardOldestPolicy());


        for (int i = 0; i < 5; i++) {
            threadPool.submit(new NamedRunnable(String.valueOf(i)));
        }

        Thread.sleep(10000);
    }
}

class NamedRunnable implements Runnable {

    String name;

    public NamedRunnable(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {

        }
        System.out.print(name + ", ");
    }
}
程序输出：
0,  4,  中间1、2、3任务被丢弃了
````
# 3.核心概念

## 3.1 线程池状态
线程池的线程数量、线程池状态都是通过一个原子类````AtomicInteger````来控制，其中低28位用来表示线程的数量(5亿左右)，高3位用来表示线程的状态(允许有8中状态，实际使用了5种)。

代码如下：

````
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

重点关注下高3位线程池状态的表示：
1110 0000 0000 0000 0000 0000 0000 0000 RUNNING
0000 0000 0000 0000 0000 0000 0000 0000 SHUTDOWN
0010 0000 0000 0000 0000 0000 0000 0000 STOP
0100 0000 0000 0000 0000 0000 0000 0000 TIDYING
0110 0000 0000 0000 0000 0000 0000 0000 TERMINATED
````
线程池状态相互转换关系如下图：
![线程池状态转换](http://upload-images.jianshu.io/upload_images/3776621-8b6e78e2d9bf6eb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.2 Woker
````Worker````继承自AQS实现了简单的非重入锁的功能，并实现了````Runnable````接口，实现了````run()````方法，调用了````runWorker()````方法(ps：runWorker这个方法后续会分析到)。每个worker对象都保存了当前工作线程，和需要执行的任务。代码如下：

````
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
````

## 3.3 提交任务过程
任务提交的方法有：````submit()````和````execute()````两种方法，````submit()````代码如下：

````
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
````

这里````submit()````其实还是调用````execute()````，````execute()````代码如下：


````
 public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    // 当前线程数量如果小于核心线程数量，创建新的线程去执行任务，而不是将任务加入队列。
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 如果线程池是运行状态，并且入队成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查线程池状态，如果不符合条件将任务删除
        if (! isRunning(recheck) && remove(command))
            // rejectHandler处理任务。
            reject(command);
        // 如果线程数量为0，创建新的线程去执行任务。
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 创建线程执行当前任务
    else if (!addWorker(command, false))
        // 失败则执行rejectHandler
        reject(command);
}

````
这里的逻辑还是比较清晰的，下面看下````addWorker()````：

````
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        // 获取线程池状态
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        /**
         * rs > SHUTDOWN || firstTask != null || workQueue.isEmpty()
         */
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            // 线程数量
            int wc = workerCountOf(c);
            /**
             * 线程不能大于最大数量限制。如果core为true则限定线程数量为corePoolSize
             * 大小，否则限定为maximumPoolSize大小
             */
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 线程池线程数量加1
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 如果线程池状态变更则继续loop
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    
    // 添加的线程启动是否成功
    boolean workerStarted = false;
    // 线程是否成功添加
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                /**
                 * 线程池为RUNNING状态时，或者调用了shutdown()方法之后空闲状态的
                 * 线程被清理了，则会主动的添加一个新的工作线程来执行之前的任务。
                 * 如果线程池为SHUTDOWN状态时，新的任务不能被添加进来，以前的任务可
                 * 以继续被执行。
                 */
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 检测Worker中的线程是否已经被启动。
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    // 记录线程池曾经最大的线程数量。
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 启动worker实例中的线程。
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 如果添加失败，则执行清理的工作：线程池线程数量减1、移除刚刚添加的Worker
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
````
到这里任务就已经提交过程就分析完毕了。总结下：

````
	1.当线程池的线程数量小于核心线程数量时，直接创建线程数量执行
	2.当线程池的线程数量大于等于核显线程数来时，先将任务入队等待工作中的线程依次处理。
	3.当线程入队失败时说明队列已经满了，就再次创建新的线程去处理(这里线程数量受maximumPoolSize影响)，如果超过了限制，则会将任务丢给rejectHandler处理。
````

## 3.4 工作线程启动过程
分析完了提交任务的过程，下面关注下启动线程过程。下面代码是Worker被添加之后开始工作的入口，看下相关代码：

````
// 省略无关代码
if (workerAdded) {
    // 启动worker实例中的线程。
    t.start();
    workerStarted = true;
}
// 省略无关代码
````
在上面**3.2````Worker````**中有提到其实现了````Runnable````接口中的````run()````方法

````
/** Delegates main run loop to outer runWorker  */
public void run() {
    runWorker(this);
}
````
那么worker实例中的线程启动的时候是如何调用该````runWorker(this)````方法的呢？这个要从创建Worker实例说起~！

Worker创建代码如下：

````
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
其中threadFactory的默认实现是Executors.DefaultThreadFactory类，代码如下：

class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}

重点关注newThread(Runnable r)的方法，在getThreadFactory().newThread(this)这里偷偷
的传入了this参数，而Worker实现了Runnable接口的run()方法，所以当线程启动的时候会执行
runWorker(this)这个方法。额，有那么一点点绕~，大家明白就好。
````
开始真正的启动过程的分析：

````
final void runWorker(Worker w) {
    // 因为调用当前方法是刚刚启动的新的工作线程，这里自然就是获取到的当前工作线程。
    Thread wt = Thread.currentThread();
    // 获取任务。
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 更新worker状态，允许响应中断。
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
    	 // 循环获取队列中的任务，getTask()下面分析
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 这一大段就说了一件事情，线程池被shutdown()的时候，将当前线程置为
            // 中断状态
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
            	   // 执行钩子函数
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 执行钩子函数
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 记录当前工作线程已经完成的任务数量
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 异常的时候执行清理工作。
        processWorkerExit(w, completedAbruptly);
    }
}
````
至此启动过程就完成了，总结下：

````
1. 通过while循环不断的获取任务，如果获取不到任务就进入finally执行processWorkerExit(w,false)
2. 如果能获取到就执行任务，中间如果报错就抛异常，进入finally执行processWorkerExit(w,true)
````
注意1和2执行````processWorkerExit()````方法时传入的参数是不同，具体的区别后面会分析到。

## 3.5 获取任务的过程
上面介绍了完了worker的启动过程，但是启动之后从任务队列中获取task过程是什么样的呢？

````
private Runnable getTask() {
    // 上次获取任务是否超时
    boolean timedOut = false; // Did the last poll() time out?
    // 死循环
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 线程池状态>=STOP()获取任务队列为空的时候，返回null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        /** 
         * 是否需要剔除工作线程。
         * 如果设定了allowCoreThreadTimeOut为true则超时的核心线程也会被清理，
         * 如果设置为false则只会清理超过corePoolSize数量之外的线程。
         */
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        
        /**
         * (如果线程数量大于maximumPoolSize的限制 || 上次获取任务超时而且需要剔除多余的线程)
         * && (线程数量大于1 或者 队列为空)，满足以上条件则返回null
         * /
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 如果需要剔除线程，则使用poll()限定时间内没有获取到任务，则返回空值
            // 如果不需要剔除线程，则使用take()无限制等待，知道有任务返回
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            // 如果任务为空，则说明超时了。
            timedOut = true;
        } catch (InterruptedException retry) {
            // 线程如果被打断则认为没有超时，继续下次循环
            timedOut = false;
        }
    }
}
````

上面各种判断看着着实有点头大，其实总结下就做了两个方面的工作：返回可执行的任务，或者返回null。返回null的意义在于退出上面**3.4 runWorker()的循环**，进而执行````processWorkerExit(w, false)````，进行线程清理的工作。

那么在执行````getTask()````有哪些情况返回null呢？

````
1. 线程池状态>=STOP或者队列为空的时候，线程池能处于STOP状态(上面有线程池状态转换的图例)是因为调用了shutdownNow()的方法，这时候不管任务队列有没有任务线程都会被清理。
2. 线程池数量超过maximumPoolSize的限制 && (线程数量大于1 或者 任务队列为空)
在设定allowCoreThreadTimeOut为true的情况下：
3. 清理上次获取任务超时的线程(核心线程也会被清理)
在设定allowCoreThreadTimeOut为false的情况下：
4. 清理上次获取任务超时的线程(只会清理大于核心线程数量之外的线程)
````

## 3.6 线程池线程数量的控制

上接**3.5获取任务的过程**，这里看下清理线程的相关代码：

````
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果是因为异常
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 将当前worker完成的任务数量纳入总的数量
        completedTaskCount += w.completedTasks;
        // 移除worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    
    // 尝试将线程池置为TERMINATED状态
    tryTerminate();

    // 假如线程池状态处于RUNNING、SHUTDOWN状态则至少保证有一个线程继续处理已有的任务，
    // 直至任务队列为空
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
````

# 4. 写在最后
希望博客的内容能给广大的Java道友提供一些的帮助和提升。由于笔者水平有限，如果内容有误，希望大家批评指出。