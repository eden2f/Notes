# Java 线程池监控

> 对 ThreadPoolExecutor  各个属性解释得挺全的 [Java并发（六）线程池监控](https://www.cnblogs.com/warehouse/p/10732965.html)

## 背景
业务使用线程池的时候，出现了问题，影响线上业务，由于没有线程池监控，导致问题难以发现和排查。于是需要这么一个线程池监控组件，用来监控线程池执状态，任务执行状态等。
## 实现方式
`ThreadPoolExecutor` 提供了以下几个方法可以监控线程池的使用情况：

| 方法 | 含义 |
| --- | --- |
| getActiveCount() | 线程池中正在执行任务的线程数量 |
| getCompletedTaskCount() | 线程池已完成的任务数量，该值小于等于taskCount |
| getCorePoolSize() | 线程池的核心线程数量 |
| getLargestPoolSize() | 线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize |
| getMaximumPoolSize() | 线程池的最大线程数量 |
| getPoolSize() | 线程池当前的线程数量 |
| getTaskCount() | 线程池已经执行的和未执行的任务总数 |

通过这些方法，可以对线程池进行监控，在 `ThreadPoolExecutor` 类中提供了几个空方法，如 `beforeExecute` 方法， `afterExecute` 方法和 `terminated` 方法，可以扩展这些方法在执行前或执行后增加一些新的操作，例如统计线程池的执行任务的时间等，可以继承自 `ThreadPoolExecutor` 来进行扩展。极简示例如下，hello~~
```java
@Slf4j
public class ThreadPoolMonitor extends ThreadPoolExecutor {
    public ThreadPoolMonitor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
                             TimeUnit unit, BlockingQueue<Runnable> workQueue,
                             ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
        log.info("init");
    }
    @Override
    public void shutdown() {
        log.info("shutdown");
        super.shutdown();
    }
    @Override
    public List<Runnable> shutdownNow() {
        log.info("shutdownNow");
        return super.shutdownNow();
    }
    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        log.info("beforeExecute");
    }
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        log.info("afterExecute");
    }
}
```
## 实战应用
上面是已经说明该组件的实现方式，但是在生产环境中，面对业务的复杂度高、变数大，我们应该如何实现一个高可拓展的线程池监控组件呢？这是这一小节的内容主题。
### 1. 使用示例
先来个 hello world 演示效果
```java
@Test
public void helloWorld() throws InterruptedException {
    //创建线程池对象
    MonitoredThreadPoolExecutor monitoredThreadPoolExecutor = new MonitoredThreadPoolExecutor("被监控的线程池1", 2, 2, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(1), new MonitorConfig().setQueueSlowTime(100).setTaskSlowTimeThreshold(100));
    monitoredThreadPoolExecutor.execute(() -> {
        log.info("任务1开始……");
        try {
            Thread.sleep(RandomUtils.nextInt(1000, 2000));
        } catch (InterruptedException ignore) {
        }
        log.info("任务1完成……");
    });
    monitoredThreadPoolExecutor.execute(() -> {
        log.info("任务2开始……");
        try {
            Thread.sleep(RandomUtils.nextInt(0, 100));
        } catch (InterruptedException ignore) {
        }
        log.info("任务2完成……");
    });
    Thread.currentThread().join(5 * 1000);
}
```
hello world 程序运行日志如下：
```shell
Connected to the target VM, address: '127.0.0.1:62724', transport: 'socket'
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池1, 提交任务数+1
[ThreadPoolMonitor_0] INFO PoolMonitorTask - 线程池名称 = 被监控的线程池1, 活跃线程数峰值 = 0, 队列任务数峰值 = 0, 核心线程数 = 2, 最大线程数 = 2, 执行的任务总数 = 0
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池1, 提交任务数+1
[被监控的线程池1_1] INFO MonitoredThreadPoolExecutorTest - 任务2开始……
[被监控的线程池1_0] INFO MonitoredThreadPoolExecutorTest - 任务1开始……
[被监控的线程池1_1] INFO MonitoredThreadPoolExecutorTest - 任务2完成……
[被监控的线程池1_1] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池1, 任务排队时间 = 1, 任务执行时间 = 86
[ThreadPoolMonitor_0] INFO PoolMonitorTask - 线程池名称 = 被监控的线程池1, 活跃线程数峰值 = 1, 队列任务数峰值 = 0, 核心线程数 = 2, 最大线程数 = 2, 执行的任务总数 = 2
[被监控的线程池1_0] INFO MonitoredThreadPoolExecutorTest - 任务1完成……
[被监控的线程池1_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池1, 任务排队时间 = 2, 任务执行时间 = 1452
[被监控的线程池1_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池1, 执行慢任务数+1
[ThreadPoolMonitor_1] INFO PoolMonitorTask - 线程池名称 = 被监控的线程池1, 活跃线程数峰值 = 0, 队列任务数峰值 = 0, 核心线程数 = 2, 最大线程数 = 2, 执行的任务总数 = 0
[ThreadPoolMonitor_0] INFO PoolMonitorTask - 线程池名称 = 被监控的线程池1, 活跃线程数峰值 = 0, 队列任务数峰值 = 0, 核心线程数 = 2, 最大线程数 = 2, 执行的任务总数 = 0
[ThreadPoolMonitor_2] INFO PoolMonitorTask - 线程池名称 = 被监控的线程池1, 活跃线程数峰值 = 0, 队列任务数峰值 = 0, 核心线程数 = 2, 最大线程数 = 2, 执行的任务总数 = 0
from the target VM, address: '127.0.0.1:62724', transport: 'socket'
```
### 2. 详细使用示例
监控方式
线程池的监控分为2种类型，一种是在执行任务前后全量统计任务排队时间和执行时间，另外一种是通过定时任务，定时获取活跃线程数，队列中的任务数，核心线程数，最大线程数等数据。
MonitoredThreadPoolExecutor 会同时统计这两种类型的数据。如果您不想统计全量任务执行和排队的监控数据，可以使用 ThreadPoolMonitor.monitor(String name, ThreadPoolExecutor threadPoolExecutor) 方法，该方法只使用定时任务来监控线程数据。其中，name 需要唯一，threadPoolExecutor 不能是MonitoredThreadPoolExecutor 类型，否则会抛出异常。
监控参数

-  poolName ：线程池名称。必须为每个线程池创建不同的名称，否则会抛出异常。可以将其作为监控平台的id，通过名称找到对应的监控数据。 
-  monitorConfig ：监控配置参数。其中可以设置两个参数，taskSlowTimeThreshold和 queueSlowTimeThreshold。如果taskSlowTime指定为100，则表示任务执行时间大于100ms的任务会统计为慢任务，在监控中可以看到慢任务的数量。同样的，queueSlowTime指定为100，表示排队时间大于100ms的任务统计为排队慢任务，可以在监控中看到排队慢任务的数量。 

其他参数和JDK中线程池参数意义相同。
使用示例代码
```java
@Slf4j
public class MonitoredThreadPoolExecutorTest {

    @Test
    public void helloWorld() throws InterruptedException {
        //创建线程池对象
        MonitoredThreadPoolExecutor monitoredThreadPoolExecutor = new MonitoredThreadPoolExecutor("被监控的线程池1", 2, 2, 60, TimeUnit.SECONDS, new ArrayBlockingQueue<>(1), new MonitorConfig().setQueueSlowTimeThreshold(100).setTaskSlowTimeThreshold(100));
        monitoredThreadPoolExecutor.execute(() -> {
            log.info("任务1开始……");
            try {
                Thread.sleep(RandomUtils.nextInt(1000, 2000));
            } catch (InterruptedException ignore) {
            }
            log.info("任务1完成……");
        });
        monitoredThreadPoolExecutor.execute(() -> {
            log.info("任务2开始……");
            try {
                Thread.sleep(RandomUtils.nextInt(0, 100));
            } catch (InterruptedException ignore) {
            }
            log.info("任务2完成……");
        });
        Thread.currentThread().join(5 * 1000);
    }

    @Test
    public void ThreadPoolExecutorTest() throws InterruptedException {
        //线程池需要指定唯一的线程池名称，否则会抛出异常
        String uniqPoolName = "被监控的线程池2";
        MonitoredThreadPoolExecutor monitoredThreadPoolExecutor = new MonitoredThreadPoolExecutor(uniqPoolName, 1, 4, 60,
                TimeUnit.SECONDS, new ArrayBlockingQueue<>(256), new MonitorConfig().setQueueSlowTimeThreshold(100).setTaskSlowTimeThreshold(100)) {
            //如果想在任务执行开始或者执行结束时，执行一些操作，覆盖afterExecute0(Runnable r, Throwable t)和beforeExecute0(Thread t, Runnable r)，注意方法名称后面有0
            @Override
            public void afterExecute0(Runnable r, Throwable t) {
                log.info("增强afterExecute0");
            }

            @Override
            public void beforeExecute0(Thread t, Runnable r) {
                log.info("增强beforeExecute0");
            }
        };

        //使用方式和ThreadPoolExecutor完全相同
        for (int i = 0; i < 3; i++) {
            int taskId = i;
            monitoredThreadPoolExecutor.submit(() -> log.info("任务{}", taskId));
        }

        for (int i = 0; i < 3; i++) {
            monitoredThreadPoolExecutor.submit(() -> {
                int id = RandomUtils.nextInt();
                log.info("生成id = {}", id);
                return id;
            });
        }
        //使用结束后，如果需要再创建相同名称的线程池，则需要调用remove方法移除定时任务。
        ThreadPoolMonitor.remove(uniqPoolName);
        //关闭线程池，在一段时间内会关闭所有监控的定时任务
        monitoredThreadPoolExecutor.shutdown();
        Thread.currentThread().join(2 * 1000);
    }

}
```
示例代码运行日志
```shell
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 提交任务数+1
[ThreadPoolMonitor_0] INFO PoolMonitorTask - 线程池名称 = 被监控的线程池2, 活跃线程数峰值 = 0, 队列任务数峰值 = 0, 核心线程数 = 1, 最大线程数 = 4, 执行的任务总数 = 0
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 提交任务数+1
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 提交任务数+1
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强beforeExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 任务0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 任务排队时间 = 0, 任务执行时间 = 0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强afterExecute0
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 提交任务数+1
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强beforeExecute0
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 提交任务数+1
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 任务1
[main] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 提交任务数+1
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 任务排队时间 = 0, 任务执行时间 = 0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强afterExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强beforeExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 任务2
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 任务排队时间 = 0, 任务执行时间 = 0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强afterExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强beforeExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 生成id = 488282372
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 任务排队时间 = 0, 任务执行时间 = 3
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强afterExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强beforeExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 生成id = 1176017668
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 任务排队时间 = 4, 任务执行时间 = 0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强afterExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强beforeExecute0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 生成id = 463520743
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutor - 线程池名称 = 被监控的线程池2, 任务排队时间 = 4, 任务执行时间 = 0
[被监控的线程池2_0] INFO MonitoredThreadPoolExecutorTest - 增强afterExecute0
```
### 3. 主要实现
详情代码看仓库吧~~

- ThreadPoolMonitor 负责线程池与监控方法的管理；
```
public class ThreadPoolMonitor {

    private static final Map<String, FutureWrapper> POOL_TASK_FUTURE_MAP = new ConcurrentHashMap<>();
    private static final ScheduledThreadPoolExecutor SCHEDULE_THREAD_POOL = new ScheduledThreadPoolExecutor(8, new NamedThreadFactory("ThreadPoolMonitor"));

    private static final Long DEFAULT_MONITOR_PERIOD_TIME_MILLS = 1000L;

    public ThreadPoolMonitor() {
    }

    public static void monitor(String name, ThreadPoolExecutor threadPoolExecutor) {
        if (threadPoolExecutor instanceof MonitoredThreadPoolExecutor) {
            throw new IllegalArgumentException("MonitoredThreadPoolExecutor is already monitored.");
        } else {
            monitor0(name, threadPoolExecutor, DEFAULT_MONITOR_PERIOD_TIME_MILLS);
        }
    }

    public static void remove(String name) {
        ThreadPoolMonitor.FutureWrapper futureWrapper = POOL_TASK_FUTURE_MAP.remove(name);
        if (futureWrapper != null) {
            futureWrapper.future.cancel(false);
        }

    }

    public static void remove(String name, ThreadPoolExecutor threadPoolExecutor) {
        ThreadPoolMonitor.FutureWrapper futureWrapper = POOL_TASK_FUTURE_MAP.get(name);
        if (futureWrapper != null && futureWrapper.threadPoolExecutor == threadPoolExecutor) {
            POOL_TASK_FUTURE_MAP.remove(name, futureWrapper);
            futureWrapper.future.cancel(false);
        }

    }

    static void monitor(MonitoredThreadPoolExecutor threadPoolExecutor) {
        monitor0(threadPoolExecutor.poolName(), threadPoolExecutor, DEFAULT_MONITOR_PERIOD_TIME_MILLS);
    }

    private static void monitor0(String name, ThreadPoolExecutor threadPoolExecutor, long monitorPeriodTimeMills) {
        PoolMonitorTask poolMonitorTask = new PoolMonitorTask(threadPoolExecutor, name);
        POOL_TASK_FUTURE_MAP.compute(name, (k, v) -> {
            if (v == null) {
                return new ThreadPoolMonitor.FutureWrapper(SCHEDULE_THREAD_POOL.scheduleWithFixedDelay(poolMonitorTask, 0L, monitorPeriodTimeMills, TimeUnit.MILLISECONDS), threadPoolExecutor);
            } else {
                throw new IllegalStateException("duplicate pool name: " + name);
            }
        });
    }

    static {
        Runtime.getRuntime().addShutdownHook(new Thread(ThreadPoolMonitor.SCHEDULE_THREAD_POOL::shutdown));
    }

    static class FutureWrapper {
        private final Future<?> future;
        private final ThreadPoolExecutor threadPoolExecutor;

        public FutureWrapper(Future<?> future, ThreadPoolExecutor threadPoolExecutor) {
            this.future = future;
            this.threadPoolExecutor = threadPoolExecutor;
        }
    }
}
```

- 给异步任务Runnable套一个壳，让他该任务可监控；
```java
public class MonitoredRunnable implements Runnable, Monitored {

    private final Runnable runnable;
    private final long inQueueNanoTime;

    public MonitoredRunnable(Runnable runnable) {
        this.runnable = runnable;
        this.inQueueNanoTime = System.nanoTime();
    }

    @Override
    public long inQueueNanoTime() {
        return this.inQueueNanoTime;
    }

    @Override
    public void run() {
        this.runnable.run();
    }
}
```

- MonitoredThreadPoolExecutor 继承 ThreadPoolExecutor 覆盖其方法做监控统计增强；
```java
@Slf4j
public class MonitoredThreadPoolExecutor extends ThreadPoolExecutor {

    private final ThreadLocal<Long> executeStartTimeThreadLocal;
    protected String poolName;
    private final int slowTaskThreshold;
    private final int queueTimeThreshold;
    private static final int DEFAULT_SLOW_TASK_TIME = 5000;
    private static final int DEFAULT_QUEUE_TIME = 100;


    @Override
    public void execute(Runnable command) {
        log.info("线程池名称 = {}, 提交任务数+1", this.poolName());
        super.execute(new MonitoredRunnable(command));
    }

    public MonitoredThreadPoolExecutor(String poolName, int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        this(poolName, corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, new NamedThreadFactory(poolName), new AbortPolicy(), new MonitorConfig());
    }

    public MonitoredThreadPoolExecutor(String poolName, int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
        this(poolName, corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler, new MonitorConfig());
    }

    public MonitoredThreadPoolExecutor(String poolName, int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, MonitorConfig monitorConfig) {
        this(poolName, corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, new NamedThreadFactory(poolName), new AbortPolicy(), monitorConfig);
    }

    public MonitoredThreadPoolExecutor(String poolName, int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler, MonitorConfig monitorConfig) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, new MonitorRejectedExecutionHandler(handler, poolName));
        this.executeStartTimeThreadLocal = new ThreadLocal<>();
        this.poolName = poolName;
        this.slowTaskThreshold = monitorConfig.getTaskSlowTimeThreshold() > 0 ? monitorConfig.getTaskSlowTimeThreshold() : DEFAULT_SLOW_TASK_TIME;
        this.queueTimeThreshold = monitorConfig.getQueueSlowTimeThreshold() > 0 ? monitorConfig.getQueueSlowTimeThreshold() : DEFAULT_QUEUE_TIME;
        ThreadPoolMonitor.monitor(this);
    }

    @Override
    protected final void beforeExecute(Thread t, Runnable r) {
        try {
            this.beforeExecute0(t, r);
        } finally {
            this.executeStartTimeThreadLocal.set(System.nanoTime());
        }
    }

    @Override
    protected final void afterExecute(Runnable r, Throwable t) {
        this.afterExecuteMonitor(r, t);
        this.afterExecute0(r, t);
    }

    private void afterExecuteMonitor(Runnable r, Throwable t) {
        try {
            long executeEndNano = System.nanoTime();
            Long executeStartTime = this.executeStartTimeThreadLocal.get();
            Monitored monitored = (Monitored) r;
            long queueNanoTime = monitored.inQueueNanoTime();
            int queueTime = (int) ((executeStartTime - queueNanoTime) / 1000000L);
            int executeTime = (int) ((executeEndNano - executeStartTime) / 1000000L);
            log.info("线程池名称 = {}, 任务排队时间 = {}, 任务执行时间 = {}",
                    this.poolName(), queueTime, executeTime);
            if (executeTime > this.slowTaskThreshold) {
                log.info("线程池名称 = {}, 执行慢任务数+1", this.poolName());
            }
            if (queueTime > this.queueTimeThreshold) {
                log.info("线程池名称 = {}, 排队慢任务数+1", this.poolName());
            }

            if (t != null) {
                log.info("线程池名称 = {}, 执行异常的任务数+1", this.poolName());
            }
        } catch (Exception ignore) {
        } finally {
            executeStartTimeThreadLocal.remove();
        }
    }

    protected void beforeExecute0(Thread t, Runnable r) {
    }

    protected void afterExecute0(Runnable r, Throwable t) {
    }

    @Override
    protected final void terminated() {
        ThreadPoolMonitor.remove(this.poolName(), this);
    }

    public String poolName() {
        return this.poolName;
    }
}
```

- PoolMonitorTask 定时收集线程池监控项的任务实现；
```java
@Slf4j
@Getter
public class PoolMonitorTask implements Runnable {

    private final ThreadPoolExecutor monitoredThreadPool;
    private final String poolName;
    private volatile long lastTaskCount = 0L;

    public PoolMonitorTask(ThreadPoolExecutor monitoredThreadPool, String poolName) {
        this.monitoredThreadPool = monitoredThreadPool;
        this.poolName = poolName;
    }

    @Override
    public void run() {
        int activeCount = this.monitoredThreadPool.getActiveCount();
        int corePoolSize = this.monitoredThreadPool.getCorePoolSize();
        int maximumPoolSize = this.monitoredThreadPool.getMaximumPoolSize();
        int queueTaskSize = this.monitoredThreadPool.getQueue().size();
        long taskCount = this.monitoredThreadPool.getTaskCount();
        int executedTask = (int) (taskCount - this.lastTaskCount);
        log.info("线程池名称 = {}, 活跃线程数峰值 = {}, 队列任务数峰值 = {}, 核心线程数 = {}, 最大线程数 = {}, 执行的任务总数 = {}",
                this.poolName, activeCount, queueTaskSize, corePoolSize, maximumPoolSize, executedTask);
        this.lastTaskCount = taskCount;
        if (this.monitoredThreadPool.isTerminated()) {
            ThreadPoolMonitor.remove(this.poolName, this.monitoredThreadPool);
        }
    }

}
```

- ThreadPoolMonitor 线程池监控者，负责线程池与监控方法的管理，定时采集任务的执行者；
```java
public class ThreadPoolMonitor {

    private static final Map<String, FutureWrapper> POOL_TASK_FUTURE_MAP = new ConcurrentHashMap<>();
    private static final ScheduledThreadPoolExecutor SCHEDULE_THREAD_POOL = new ScheduledThreadPoolExecutor(8, new NamedThreadFactory("ThreadPoolMonitor"));

    private static final Long DEFAULT_MONITOR_PERIOD_TIME_MILLS = 1000L;

    public ThreadPoolMonitor() {
    }

    public static void monitor(String name, ThreadPoolExecutor threadPoolExecutor) {
        if (threadPoolExecutor instanceof MonitoredThreadPoolExecutor) {
            throw new IllegalArgumentException("MonitoredThreadPoolExecutor is already monitored.");
        } else {
            monitor0(name, threadPoolExecutor, DEFAULT_MONITOR_PERIOD_TIME_MILLS);
        }
    }

    public static void remove(String name) {
        ThreadPoolMonitor.FutureWrapper futureWrapper = POOL_TASK_FUTURE_MAP.remove(name);
        if (futureWrapper != null) {
            futureWrapper.future.cancel(false);
        }

    }

    public static void remove(String name, ThreadPoolExecutor threadPoolExecutor) {
        ThreadPoolMonitor.FutureWrapper futureWrapper = POOL_TASK_FUTURE_MAP.get(name);
        if (futureWrapper != null && futureWrapper.threadPoolExecutor == threadPoolExecutor) {
            POOL_TASK_FUTURE_MAP.remove(name, futureWrapper);
            futureWrapper.future.cancel(false);
        }

    }

    static void monitor(MonitoredThreadPoolExecutor threadPoolExecutor) {
        monitor0(threadPoolExecutor.poolName(), threadPoolExecutor, DEFAULT_MONITOR_PERIOD_TIME_MILLS);
    }

    private static void monitor0(String name, ThreadPoolExecutor threadPoolExecutor, long monitorPeriodTimeMills) {
        PoolMonitorTask poolMonitorTask = new PoolMonitorTask(threadPoolExecutor, name);
        POOL_TASK_FUTURE_MAP.compute(name, (k, v) -> {
            if (v == null) {
                return new ThreadPoolMonitor.FutureWrapper(SCHEDULE_THREAD_POOL.scheduleWithFixedDelay(poolMonitorTask, 0L, monitorPeriodTimeMills, TimeUnit.MILLISECONDS), threadPoolExecutor);
            } else {
                throw new IllegalStateException("duplicate pool name: " + name);
            }
        });
    }

    static {
        Runtime.getRuntime().addShutdownHook(new Thread(ThreadPoolMonitor.SCHEDULE_THREAD_POOL::shutdown));
    }

    static class FutureWrapper {
        private final Future<?> future;
        private final ThreadPoolExecutor threadPoolExecutor;

        public FutureWrapper(Future<?> future, ThreadPoolExecutor threadPoolExecutor) {
            this.future = future;
            this.threadPoolExecutor = threadPoolExecutor;
        }
    }
}
```
