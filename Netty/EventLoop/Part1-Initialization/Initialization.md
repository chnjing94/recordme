### 1. 概述
因为EventLoop内容较多，分为几部分来分析。这一部分讲EventLoop的初始化部分。
本章的主要目标如下：
+ 理解 EventLoop 有哪些属性
+ 创建 EventLoop 的过程
+ Channel 注册到 EventLoop 的过程
+ EventLoop 的任务提交

### 2. 类结构图

![](Initialization_files/1.png)
+ 红色框以外的部分为EventLoop相关的类

### 3. EventExecutor

`EventExecutor`，继承 EventExecutorGroup 接口，事件执行器接口。
```java
// ========== 实现自 EventExecutorGroup 接口 ==========
/**
 * 返回自己
 */
@Override
EventExecutor next();

// ========== 自定义接口 ==========
/**
 * 返回所属的 EventExecutorGroup
 */
EventExecutorGroup parent();

/**
 * 当前线程是否在 EventLoop 线程中
 */
boolean inEventLoop();

/**
 * 指定线程是否是 EventLoop 线程
 */
boolean inEventLoop(Thread thread);

/**
 * 创建一个 Promise 对象
 */
<V> Promise<V> newPromise();

/**
 * 创建一个 ProgressivePromise 对象
 */
<V> ProgressivePromise<V> newProgressivePromise();

/**
 * 创建成功结果的 Future 对象
 *
 * Create a new {@link Future} which is marked as succeeded already. So {@link Future#isSuccess()}
 * will return {@code true}. All {@link FutureListener} added to it will be notified directly. Also
 * every call of blocking methods will just return without blocking.
 */
<V> Future<V> newSucceededFuture(V result);

/**
 * 创建异常的 Future 对象
 *
 * Create a new {@link Future} which is marked as failed already. So {@link Future#isSuccess()}
 * will return {@code false}. All {@link FutureListener} added to it will be notified directly. Also
 * every call of blocking methods will just return without blocking.
 */
<V> Future<V> newFailedFuture(Throwable cause);

```

### 4. OrderedEventExecutor
`OrderedEventExecutor` ，继承 EventExecutor 接口，有序的事件执行器接口
```java
/**
 * Marker interface for {@link EventExecutor}s that will process all submitted tasks in an ordered / serial fashion.
 */
public interface OrderedEventExecutor extends EventExecutor {
}
```
+ 没有定义任何方法，仅仅是一个标记接口，表示该执行器会有序 / 串行的方式执行。

### 5. EventLoop
`EventLoop` ，继承 `OrderedEventExecutor` 和 `EventLoopGroup` 接口
```java
/**
 * Will handle all the I/O operations for a {@link Channel} once registered.
 *
 * One {@link EventLoop} instance will usually handle more than one {@link Channel} but this may depend on
 * implementation details and internals.
 *
 */
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {

    @Override
    EventLoopGroup parent();

}

```

+ `parent()` 接口方法，覆写父类`EventExecutor#parent()`方法,新的返回类型为 `EventLoopGroup` 。以前返回类型为`EventExecutorGroup`

### 6. AbstractEventExecutor
`AbstractEventExecutor` ，实现 `EventExecutor` 接口，继承 `AbstractExecutorService` 抽象类，`EventExecutor` 抽象类。

#### 6.1 构造方法
```java
/**
 * 所属 EventExecutorGroup
 */
private final EventExecutorGroup parent;
/**
 * EventExecutor 数组。只包含自己，用于 {@link #iterator()}
 */
private final Collection<EventExecutor> selfCollection = Collections.<EventExecutor>singleton(this);

protected AbstractEventExecutor() {
    this(null);
}

protected AbstractEventExecutor(EventExecutorGroup parent) {
    this.parent = parent;
}
```

#### 6.2 inEventLoop
`inEventLoop()` 判断当前线程是否在EventLoop线程中。
```java
@Override
public boolean inEventLoop() {
    return inEventLoop(Thread.currentThread());
}
```
+ 需要在子类中去实现`inEventLoop(Thread thread)`

#### 6.3 newPromise 和 newProgressivePromise
`newPromise()` 和 `newProgressivePromise()` 方法，分别创建 DefaultPromise 和 DefaultProgressivePromise 对象
```java
@Override
public <V> Promise<V> newPromise() {
    return new DefaultPromise<V>(this);
}

@Override
public <V> ProgressivePromise<V> newProgressivePromise() {
    return new DefaultProgressivePromise<V>(this);
}
```
+ 创建的 Promise 对象，都会传入自身作为 EventExecutor 。Promise相关的内容后面分析

#### 6.3 newPromise 和 newProgressivePromise
`newSucceededFuture(V result)` 和 `newFailedFuture(Throwable cause)` 方法，分别创建成功结果和异常的 Future 对象
```java
@Override
public <V> Future<V> newSucceededFuture(V result) {
    return new SucceededFuture<V>(this, result);
}

@Override
public <V> Future<V> newFailedFuture(Throwable cause) {
    return new FailedFuture<V>(this, cause);
}
```
+ 创建的 Future 对象，会传入自身作为 EventExecutor ，并传入 `result` 或 `cause` 分别作为成功结果和异常

#### 6.4 newTaskFor
`newTaskFor(...)` 方法，创建 PromiseTask 对象。
```java
@Override
protected final <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new PromiseTask<T>(this, runnable, value);
}

@Override
protected final <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new PromiseTask<T>(this, callable);
}
```
+ 创建的 PromiseTask 对象，会传入自身作为 EventExecutor ，并传入 Runnable + Value 或 Callable 作为任务( Task )

#### 6.5 submit
`submit(...)` 方法，提交任务
```java
@Override
public Future<?> submit(Runnable task) {
    return (Future<?>) super.submit(task);
}

@Override
public <T> Future<T> submit(Runnable task, T result) {
    return (Future<T>) super.submit(task, result);
}

@Override
public <T> Future<T> submit(Callable<T> task) {
    return (Future<T>) super.submit(task);
}
```
+ 调用父类 AbstractExecutorService 的实现

#### 6.6 schedule
schedule(...) 方法，都不支持，交给子类 AbstractScheduledEventExecutor 实现
```java
@Override
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
    throw new UnsupportedOperationException();
}
@Override
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
    throw new UnsupportedOperationException();
}

@Override
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
    throw new UnsupportedOperationException();
}
@Override
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
    throw new UnsupportedOperationException();
}
```

#### 6.7 safeExecute
`safeExecute(Runnable task)` 静态方法，安全的执行任务
```java
protected static void safeExecute(Runnable task) {
    try {
        task.run();
    } catch (Throwable t) {
        logger.warn("A task raised an exception. Task: {}", task, t);
    }
}
```
+ 这里的safe是指当，任务执行发生异常时，仅仅打印**警告**日志

#### 6.8 shutdown
`shutdown()` 方法，关闭执行器
```java
@Override
public Future<?> shutdownGracefully() {
    return shutdownGracefully(DEFAULT_SHUTDOWN_QUIET_PERIOD, DEFAULT_SHUTDOWN_TIMEOUT, TimeUnit.SECONDS);
}

@Override
@Deprecated
public List<Runnable> shutdownNow() {
    shutdown();
    return Collections.emptyList();
}
```
+ 具体的 #shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit) 和 #shutdown() 方法的实现，在子类中。

### 7. SingleThreadEventExecutor
实现 OrderedEventExecutor 接口，继承 AbstractScheduledEventExecutor 抽象类，基于单线程的 EventExecutor 抽象类，即一个 EventExecutor 对应一个线程。
#### 7.1 构造方法
```java
/**
 * {@link #state} 字段的原子更新器
 */
private static final AtomicIntegerFieldUpdater<SingleThreadEventExecutor> STATE_UPDATER =AtomicIntegerFieldUpdater.newUpdater(SingleThreadEventExecutor.class, "state");
/**
 * {@link #thread} 字段的原子更新器
 */
private static final AtomicReferenceFieldUpdater<SingleThreadEventExecutor, ThreadProperties> PROPERTIES_UPDATER = AtomicReferenceFieldUpdater.newUpdater(SingleThreadEventExecutor.class, ThreadProperties.class, "threadProperties");

/**
 * 任务队列
 *
 * @see #newTaskQueue(int)
 */
private final Queue<Runnable> taskQueue;
/**
 * 线程
 */
private volatile Thread thread;
/**
 * 线程属性
 */
@SuppressWarnings("unused")
private volatile ThreadProperties threadProperties;
/**
 * 执行器
 */
private final Executor executor;
/**
 * 线程是否已经打断
 *
 * @see #interruptThread()
 */
private volatile boolean interrupted;

/**
 * TODO 1006 EventLoop 优雅关闭
 */
private final Semaphore threadLock = new Semaphore(0);
/**
 * TODO 1006 EventLoop 优雅关闭
 */
private final Set<Runnable> shutdownHooks = new LinkedHashSet<Runnable>();
/**
 * 添加任务时，是否唤醒线程{@link #thread}
 */
private final boolean addTaskWakesUp;
/**
 * 最大等待执行任务数量，即 {@link #taskQueue} 的队列大小
 */
private final int maxPendingTasks;
/**
 * 拒绝执行处理器
 *
 * @see #reject()
 * @see #reject(Runnable)
 */
private final RejectedExecutionHandler rejectedExecutionHandler;

/**
 * 最后执行时间
 */
private long lastExecutionTime;

/**
 * 状态
 */
@SuppressWarnings({ "FieldMayBeFinal", "unused" })
private volatile int state = ST_NOT_STARTED;

/**
 * TODO 优雅关闭
 */
private volatile long gracefulShutdownQuietPeriod;
/**
 * 优雅关闭超时时间，单位：毫秒 TODO 1006 EventLoop 优雅关闭
 */
private volatile long gracefulShutdownTimeout;
/**
 * 优雅关闭开始时间，单位：毫秒 TODO 1006 EventLoop 优雅关闭
 */
private long gracefulShutdownStartTime;

/**
 * TODO 1006 EventLoop 优雅关闭
 */
private final Promise<?> terminationFuture = new DefaultPromise<Void>(GlobalEventExecutor.INSTANCE);

protected SingleThreadEventExecutor(
        EventExecutorGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp) {
    this(parent, new ThreadPerTaskExecutor(threadFactory), addTaskWakesUp);
}

protected SingleThreadEventExecutor(
        EventExecutorGroup parent, ThreadFactory threadFactory,
        boolean addTaskWakesUp, int maxPendingTasks, RejectedExecutionHandler rejectedHandler) {
    this(parent, new ThreadPerTaskExecutor(threadFactory), addTaskWakesUp, maxPendingTasks, rejectedHandler);
}

protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
    this(parent, executor, addTaskWakesUp, DEFAULT_MAX_PENDING_EXECUTOR_TASKS, RejectedExecutionHandlers.reject());
}

protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, int maxPendingTasks,
                                    RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = Math.max(16, maxPendingTasks);
    this.executor = ObjectUtil.checkNotNull(executor, "executor");
    taskQueue = newTaskQueue(this.maxPendingTasks);
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```
+ `maxPendingTasks`，最大等待执行任务数量，即 taskQueue 队列大小
+ `rejectedExecutionHandler`，拒绝执行处理器。在 taskQueue 队列超过最大任务数量时，怎么拒绝处理新提交的任务。
+ `thread`，任务提交到`taskQueue`中，由thread执行
+ `executor`，由它创建线程
+ `state`，线程状态，SingleThreadEventExecutor 在实现上，thread 的初始化采用延迟启动的方式，只有在第一个任务时，executor 才会执行并创建该线程，从而节省资源。目前 thread 线程有 5 种状态
```
private static final int ST_NOT_STARTED = 1; // 未开始
private static final int ST_STARTED = 2; // 已开始
private static final int ST_SHUTTING_DOWN = 3; // 正在关闭中
private static final int ST_SHUTDOWN = 4; // 已关闭
private static final int ST_TERMINATED = 5; // 已经终止
```
状态转换图如下：
![](Initialization_files/2.png)

#### 7.2 newTaskQueue
`newTaskQueue(int maxPendingTasks)` 方法，创建任务队列
```java
/**
 * Create a new {@link Queue} which will holds the tasks to execute. This default implementation will return a
 * {@link LinkedBlockingQueue} but if your sub-class of {@link SingleThreadEventExecutor} will not do any blocking
 * calls on the this {@link Queue} it may make sense to {@code @Override} this and return some more performant
 * implementation that does not support blocking operations at all.
 */
protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
    return new LinkedBlockingQueue<Runnable>(maxPendingTasks);
}
```
+ 这个方法默认返回的是 LinkedBlockingQueue 阻塞队列。如果子类有更好的队列选择( 例如非阻塞队列 )，可以重写该方法

#### 7.3 inEventLoop
`inEventLoop(Thread thread)` 方法，判断指定线程是否是 EventLoop 线程
```java
@Override
public boolean inEventLoop(Thread thread) {
    return thread == this.thread;
}
```

#### 7.4 offerTask
`offerTask(Runnable task)` 方法，添加任务到队列中。若添加失败，则返回 false
```java
final boolean offerTask(Runnable task) {
    // 关闭时，拒绝任务
    if (isShutdown()) {
        reject();
    }
    // 添加任务到队列
    return taskQueue.offer(task);
}
```

#### 7.5 addTask
`addTask(Runnable task)`，在 #offerTask(Runnable task) 的方法的基础上，若添加任务到队列中失败，则进行拒绝任务
```java
protected void addTask(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    // 添加任务到队列
    if (!offerTask(task)) {
        // 添加失败，则拒绝任务
        reject(task);
    }
}
```
+ 调用 #reject(task) 方法，拒绝任务。

#### 7.6 peekTask
`peekTask()` 方法，返回队头的任务，但是不移除
```java
protected Runnable peekTask() {
    assert inEventLoop(); // 仅允许在 EventLoop 线程中执行，这也是体现单线程的地方，不允许在其他线程中消费task队列
    return taskQueue.peek();
}
```

#### 7.7 reject
`reject(Runnable task)` 方法，拒绝任务
```java
protected final void reject(Runnable task) {
    rejectedExecutionHandler.rejected(task, this);
}
```
+ 调用 RejectedExecutionHandler#rejected(Runnable task, SingleThreadEventExecutor executor) 方法，拒绝该任务

`reject()` 方法，拒绝任何任务，用于 SingleThreadEventExecutor 已关闭( #isShutdown() 方法返回的结果为 true )的情况
```java
protected static void reject() {
    throw new RejectedExecutionException("event executor terminated");
}
```

#### 7.7.1 RejectedExecutionHandler
拒绝执行处理器接口
```java
/**
 * Called when someone tried to add a task to {@link SingleThreadEventExecutor} but this failed due capacity
 * restrictions.
 */
void rejected(Runnable task, SingleThreadEventExecutor executor);
```

#### 7.7.2 RejectedExecutionHandlers
有2种实现

**第一种**
```java
private static final RejectedExecutionHandler REJECT = new RejectedExecutionHandler() {

    @Override
    public void rejected(Runnable task, SingleThreadEventExecutor executor) {
        throw new RejectedExecutionException();
    }

};

public static RejectedExecutionHandler reject() {
    return REJECT;
}
```
+ 通过 `reject()` 方法，返回 REJECT 实现类的对象。该实现在拒绝时，直接抛出 RejectedExecutionException 异常。

**第二种**
```java
public static RejectedExecutionHandler backoff(final int retries, long backoffAmount, TimeUnit unit) {
    ObjectUtil.checkPositive(retries, "retries");
    final long backOffNanos = unit.toNanos(backoffAmount);
    return new RejectedExecutionHandler() {
        @Override
        public void rejected(Runnable task, SingleThreadEventExecutor executor) {
            if (!executor.inEventLoop()) { // 非 EventLoop 线程中。如果在 EventLoop 线程中，就无法执行任务，这就导致完全无法重试了。
                // 循环多次尝试添加到队列中
                for (int i = 0; i < retries; i++) {
                    // 唤醒执行器，进行任务执行。这样，就可能执行掉部分任务。
                    // Try to wake up the executor so it will empty its task queue.
                    executor.wakeup(false);

                    // 阻塞等待
                    LockSupport.parkNanos(backOffNanos);
                    // 添加任务
                    if (executor.offerTask(task)) {
                        return;
                    }
                }
            }
            // Either we tried to add the task from within the EventLoop or we was not able to add it even with
            // backoff.
            // 多次尝试添加失败，抛出 RejectedExecutionException 异常
            throw new RejectedExecutionException();
        }
    };
}
```
+ 通过 `backoff(final int retries, long backoffAmount, TimeUnit unit)` 方法，创建带多次尝试添加到任务队列的 RejectedExecutionHandler 实现类

#### 7.8 execute
`execute(Runnable task)` 方法，执行一个任务。
```java
 1: @Override
 2: public void execute(Runnable task) {
 3:     if (task == null) {
 4:         throw new NullPointerException("task");
 5:     }
 6: 
 7:     // 获得当前是否在 EventLoop 的线程中
 8:     boolean inEventLoop = inEventLoop();
 9:     // 添加到任务队列
10:     addTask(task);
11:     if (!inEventLoop) {
12:         // 创建线程
13:         startThread();
14:         // 若已经关闭，移除任务，并进行拒绝
15:         if (isShutdown() && removeTask(task)) {
16:             reject();
17:         }
18:     }
19: 
20:     // 唤醒线程
21:     if (!addTaskWakesUp && wakesUpForTask(task)) {
22:         wakeup(inEventLoop);
23:     }
24: }
```
+ 11行，如果是非EventLoop线程
+ 13行，调用 `startThread()` 方法，启动 EventLoop 独占的线程，即 thread 属性
+ 15-17行，若已经关闭，则移除任务，并拒绝执行
+ 20-23行：调用 `#wakeup(boolean inEventLoop)` 方法，唤醒线程
+ `addTaskWakesUp`的意思是**添加任务后，任务是否会自动导致线程唤醒**
+ Nio 使用的 NioEventLoop ，它的线程执行任务是基于 Selector 监听感兴趣的事件，所以当任务添加到 taskQueue 队列中时，线程是无感知的，所以需要调用 #wakeup(boolean inEventLoop) 方法，进行主动的唤醒，因此在`NioEventLoop`的初始化函数中，将`addTaskWakesUp`设置为了false
![](Initialization_files/3.png)
+ Oio 使用的 ThreadPerChannelEventLoop ，它的线程执行是基于 taskQueue 队列监听( 阻塞拉取 )事件和任务，所以当任务添加到 taskQueue 队列中时，线程是可感知的，相当于说，进行被动的唤醒，因此在`ThreadPerChannelEventLoop`的初始化函数中，将`addTaskWakesUp`设置为了true
![](Initialization_files/4.png)

#### 7.9 startThread
`startThread()` 方法，启动 `EventLoop` 独占的线程，即 `thread` 属性
```java
 1: private void doStartThread() {
 2:     assert thread == null;
 3:     executor.execute(new Runnable() {
 4: 
 5:         @Override
 6:         public void run() {
 7:             // 记录当前线程
 8:             thread = Thread.currentThread();
 9: 
10:             // 如果当前线程已经被标记打断，则进行打断操作。
11:             if (interrupted) {
12:                 thread.interrupt();
13:             }
14: 
15:             boolean success = false; // 是否执行成功
16: 
17:             // 更新最后执行时间
18:             updateLastExecutionTime();
19:             try {
20:                 // 执行任务
21:                 SingleThreadEventExecutor.this.run();
22:                 success = true; // 标记执行成功
23:             } catch (Throwable t) {
24:                 logger.warn("Unexpected exception from an event executor: ", t);
25:             } finally {
26:                 // TODO 1006 EventLoop 优雅关闭
27:                 for (;;) {
28:                     int oldState = state;
29:                     if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
30:                             SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
31:                         break;
32:                     }
33:                 }
34: 
35:                 // TODO 1006 EventLoop 优雅关闭
36:                 // Check if confirmShutdown() was called at the end of the loop.
37:                 if (success && gracefulShutdownStartTime == 0) {
38:                     if (logger.isErrorEnabled()) {
39:                         logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
40:                                 SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
41:                                 "be called before run() implementation terminates.");
42:                     }
43:                 }
44: 
45:                 // TODO 1006 EventLoop 优雅关闭
46:                 try {
47:                     // Run all remaining tasks and shutdown hooks.
48:                     for (;;) {
49:                         if (confirmShutdown()) {
50:                             break;
51:                         }
52:                     }
53:                 } finally {
54:                     try {
55:                         cleanup(); // 清理，释放资源
56:                     } finally {
57:                         STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
58:                         threadLock.release();
59:                         if (!taskQueue.isEmpty()) {
60:                             if (logger.isWarnEnabled()) {
61:                                 logger.warn("An event executor terminated with " +
62:                                         "non-empty task queue (" + taskQueue.size() + ')');
63:                             }
64:                         }
65: 
66:                         terminationFuture.setSuccess(null);
67:                     }
68:                 }
69:             }
70:             
71:         }
72:     });
73: }
```
+ 第 3 行 至 72 行：调用 Executor#execute(Runnable runnable) 方法，执行任务。
+ 第 8 行：赋值当前的线程给 thread 属性。这就是，每个 SingleThreadEventExecutor 独占的线程的创建方式。
+ 第 10 至 13 行：如果当前线程已经被标记打断，则进行打断操作。为什么会有这样的逻辑呢？详细解析，见 「7.11 interruptThread」 。
+ 第 18 行：调用 #updateLastExecutionTime() 方法，更新最后执行时间
```java
/**
 * Updates the internal timestamp that tells when a submitted task was executed most recently.
 * {@link #runAllTasks()} and {@link #runAllTasks(long)} updates this timestamp automatically, and thus there's
 * usually no need to call this method.  However, if you take the tasks manually using {@link #takeTask()} or
 * {@link #pollTask()}, you have to call this method at the end of task execution loop for accurate quiet period
 * checks.
 */
protected void updateLastExecutionTime() {
    lastExecutionTime = ScheduledFutureTask.nanoTime();
}
```
+ 第 21 行：调用 `SingleThreadEventExecutor#run()` 方法，执行任务。详细解析，见 7.13 run 。
+ 第 55 行：调用 `#cleanup()` 方法，清理释放资源，详细解析，见 7.X cleanup 。

#### 7.10 wakeup
wakeup(boolean inEventLoop) 方法，唤醒线程
```java
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop // <1> 
            || state == ST_SHUTTING_DOWN) { // TODO 1006 EventLoop 优雅关闭
        // Use offer as we actually only need this to unblock the thread and if offer fails we do not care as there
        // is already something in the queue.
        taskQueue.offer(WAKEUP_TASK); // <2>
    }
}
```
+ <1> 处的 !inEventLoop 代码段，判断不在 EventLoop 的线程中。因为，如果在 EventLoop 线程中，意味着线程就在执行中，不必要唤醒。代码如下：
```java
private static final Runnable WAKEUP_TASK = new Runnable() {
    @Override
    public void run() {
        // Do nothing.
    }
};
```
+ 这是一个空的 Runnable 实现类。仅仅用于唤醒基于 taskQueue 阻塞拉取的 EventLoop 实现类。
+ 对于 NioEventLoop 会重写该方法，代码如下:
```java
@Override
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
        selector.wakeup();
    }
}
```
+ 通过 NIO Selector 唤醒。

#### 7.11 interruptThread
`interruptThread()` 方法，打断 EventLoop 的线程
```java
protected void interruptThread() {
    Thread currentThread = thread;
    // 线程不存在，则标记线程被打断
    if (currentThread == null) {
        interrupted = true;
    // 打断线程
    } else {
        currentThread.interrupt();
    }
}
```
+ 因为 EventLoop 的线程是延迟启动，所以可能 thread 并未创建，此时通过 interrupted 标记打断。之后在 #startThread() 方法中，创建完线程后，再进行打断，也就是说，“延迟打断”。

#### 7.12 threadProperties
`threadProperties()` 方法，获得 EventLoop 的线程属性。
```java
 1: public final ThreadProperties threadProperties() {
 2:     ThreadProperties threadProperties = this.threadProperties;
 3:     if (threadProperties == null) {
 4:         Thread thread = this.thread;
 5:         if (thread == null) {
 6:             assert !inEventLoop();
 7:             // 提交空任务，促使 execute 方法执行
 8:             submit(NOOP_TASK).syncUninterruptibly();
 9:             // 获得线程
10:             thread = this.thread;
11:             assert thread != null;
12:         }
13: 
14:         // 创建 DefaultThreadProperties 对象
15:         threadProperties = new DefaultThreadProperties(thread);
16:         // CAS 修改 threadProperties 属性
17:         if (!PROPERTIES_UPDATER.compareAndSet(this, null, threadProperties)) {
18:             threadProperties = this.threadProperties;
19:         }
20:     }
21: 
22:     return threadProperties;
23: }
```
+ 第 2 至 3 行：获得 ThreadProperties 对象。若不存在，则进行创建 ThreadProperties 对象。
+ 第 4 至 5 行：获得 EventLoop 的线程。因为线程是延迟启动的，所以会出现线程为空的情况。若线程为空，则需要进行创建
+ 第 8 行：调用 #submit(Runnable) 方法，提交任务，就能促使 #execute(Runnable) 方法执行。
+ 第 8 行：调用 Future#syncUninterruptibly() 方法，保证 execute() 方法中异步创建 thread 完成
+ 第 10 至 11 行：获得线程，并断言保证线程存在。
+ 第 15 行：调用 DefaultThreadProperties 对象。
+ 第 16 至 19 行：CAS 修改 threadProperties 属性。

#### 7.12.1 ThreadProperties
ThreadProperties ，线程属性接口
```java
Thread.State state();

int priority();

boolean isInterrupted();

boolean isDaemon();

String name();

long id();

StackTraceElement[] stackTrace();

boolean isAlive();
```
+ 我们可以看到，每个实现方法，实际上就是对被包装的线程 t 的方法的封装。

#### 7.12.2 DefaultThreadProperties
`DefaultThreadProperties` 实现 ThreadProperties 接口，默认线程属性实现类。DefaultThreadProperties 内嵌在 SingleThreadEventExecutor 中。
```java
private static final class DefaultThreadProperties implements ThreadProperties {

    private final Thread t;

    DefaultThreadProperties(Thread t) {
        this.t = t;
    }

    @Override
    public State state() {
        return t.getState();
    }

    @Override
    public int priority() {
        return t.getPriority();
    }

    @Override
    public boolean isInterrupted() {
        return t.isInterrupted();
    }

    @Override
    public boolean isDaemon() {
        return t.isDaemon();
    }

    @Override
    public String name() {
        return t.getName();
    }

    @Override
    public long id() {
        return t.getId();
    }

    @Override
    public StackTraceElement[] stackTrace() {
        return t.getStackTrace();
    }

    @Override
    public boolean isAlive() {
        return t.isAlive();
    }

}
```
+ 我们可以看到，每个实现方法，实际上就是对被包装的线程 t 的方法的封装。
+ 那为什么 #threadProperties() 方法不直接返回 thread 呢？因为如果直接返回 thread ，调用方可以调用到该变量的其他方法，这个是我们不希望看到的。

#### 7.13 run
`run()` 方法，它是一个抽象方法，由子类实现，如何执行 taskQueue 队列中的任务
```java
protected abstract void run();
```
SingleThreadEventExecutor 提供了很多执行任务的方法，方便子类在实现自定义运行任务的逻辑时：
+ #runAllTasks()
+ #runAllTasks(long timeoutNanos)
+ #runAllTasksFrom(Queue<Runnable> taskQueue)
+ #afterRunningAllTasks()
+ #pollTask()
+ #pollTaskFrom(Queue<Runnable> taskQueue)
+ #takeTask()
+ #fetchFromScheduledTaskQueue()
+ #delayNanos(long currentTimeNanos)

#### 7.14 invokeAll
`invokeAll(...)` 方法，在 EventExecutor 中执行多个普通任务
```java
@Override
public <T> List<java.util.concurrent.Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
    throwIfInEventLoop("invokeAll");
    return super.invokeAll(tasks);
}

@Override
public <T> List<java.util.concurrent.Future<T>> invokeAll(
        Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
    throwIfInEventLoop("invokeAll");
    return super.invokeAll(tasks, timeout, unit);
}
```
+ 调用 `#throwIfInEventLoop(String method)` 方法，判断若在 EventLoop 的线程中调用该方法，抛出 RejectedExecutionException 异常
```java
private void throwIfInEventLoop(String method) {
    if (inEventLoop()) {
        throw new RejectedExecutionException("Calling " + method + " from within the EventLoop is not allowed");
    }
}
```

### 8. SingleThreadEventLoop
`io.netty.channel.SingleThreadEventLoop` ，实现 EventLoop 接口，继承 SingleThreadEventExecutor 抽象类，基于单线程的 EventLoop 抽象类，主要增加了 Channel 注册到 EventLoop 上。

#### 8.1 register
`#register(Channel channel)` 方法，注册 Channel 到 EventLoop 上
```java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```
+ 将 Channel 和 EventLoop 创建一个 DefaultChannelPromise 对象。通过这个 DefaultChannelPromise 对象，我们就能实现对异步注册过程的监听。
+ 调用 #register(final ChannelPromise promise) 方法，注册 Channel 到 EventLoop 上。
```java
@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // 注册 Channel 到 EventLoop 上
    promise.channel().unsafe().register(this, promise);
    // 返回 ChannelPromise 对象
    return promise;
}
```
+ 在方法内部,调用 AbstractUnsafe#register(EventLoop eventLoop, final ChannelPromise promise) 方法，**注册 Channel 到 EventLoop 上**。

#### 8.2 executeAfterEventLoopIteration
`#executeAfterEventLoopIteration(Runnable task)` 方法，执行一个任务。但是方法名无法很完整的体现出具体的方法实现，甚至有一些出入，所以我们直接看源码，代码如下：
```java
 1: @UnstableApi
 2: public final void executeAfterEventLoopIteration(Runnable task) {
 3:     ObjectUtil.checkNotNull(task, "task");
 4:     // 关闭时，拒绝任务
 5:     if (isShutdown()) {
 6:         reject();
 7:     }
 8: 
 9:     // 添加到任务队列
10:     if (!tailTasks.offer(task)) {
11:         // 添加失败，则拒绝任务
12:         reject(task);
13:     }
14: 
15:     // 唤醒线程
16:     if (wakesUpForTask(task)) {
17:         wakeup(inEventLoop());
18:     }
19: }
```
+ 第 4 至 7 行：SingleThreadEventLoop 关闭时，拒绝任务。
+ 第 10 行：调用 Queue#offer(E e) 方法，添加任务到队列中。
+ 第 12 行：若添加失败，调用 #reject(Runnable task) 方法，拒绝任务。
+ 第 15 至 18 行：唤醒线程。
+ 第 16 行：SingleThreadEventLoop 重写了 #wakesUpForTask(Runnable task) 方法。详细解析，见 「8.4 wakesUpForTask」 。

#### 8.3 removeAfterEventLoopIterationTask
`#removeAfterEventLoopIterationTask(Runnable task)` 方法，移除指定任务
```java
@UnstableApi
final boolean removeAfterEventLoopIterationTask(Runnable task) {
    return tailTasks.remove(ObjectUtil.checkNotNull(task, "task"));
}
```

#### 8.4 wakesUpForTask
`#wakesUpForTask(task)` 方法，判断该任务是否需要唤醒线程。
```java
@Override
protected boolean wakesUpForTask(Runnable task) {
    return !(task instanceof NonWakeupRunnable);
}
```
+ 当任务类型为 NonWakeupRunnable ，则不进行唤醒线程。

#### 8.4.1 NonWakeupRunnable
NonWakeupRunnable 实现 Runnable 接口，用于标记不唤醒线程的任务。代码如下：
```java
/**
 * Marker interface for {@link Runnable} that will not trigger an {@link #wakeup(boolean)} in all cases.
 */
interface NonWakeupRunnable extends Runnable { }
```

#### 8.5 afterRunningAllTasks
`#afterRunningAllTasks()` 方法，在运行完所有任务后，执行 tailTasks 队列中的任务。
```java
protected void afterRunningAllTasks() {
    runAllTasksFrom(tailTasks);
}
```
+ 调用 #runAllTasksFrom(queue) 方法，执行 tailTasks 队列中的所有任务。

### 9. NioEventLoop
`NioEventLoop` ，继承 SingleThreadEventLoop 抽象类，NIO EventLoop 实现类，实现对注册到其中的 Channel 的就绪的 IO 事件，和对用户提交的任务进行处理。详细解析见下篇。