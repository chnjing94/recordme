## EventLoopGroup


### 1.概述

在之前启动客户端或者服务端的时候，我们使用了`new NioEventLoopGroup()`，去得到一个EventLoopGroup对象，本文就详细解析一下NioEventLoopGroup这个类，以及它的上层接口以及父类们。整体的类结构图如下：图@芋道源码，其中红色框中为NioEventLoopGroup的类关系图。![](EventLoopGroup_files/1.jpg)

### 2.EventExecutorGroup

从EventExecutorGroup开始分析，因为再上面就是java自身的 Iterable、ScheduledExecutorService接口了。

```java
// ========== 自定义接口 ==========

boolean isShuttingDown();

Future<?> shutdownGracefully();
Future<?> shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit);

Future<?> terminationFuture();

// 选择下一个 EventExecutor 对象
EventExecutor next(); 

// ========== 实现自 Iterable 接口 ==========

@Override
Iterator<EventExecutor> iterator();

// ========== 实现自 ExecutorService 接口 ==========

@Override
Future<?> submit(Runnable task);
@Override
<T> Future<T> submit(Runnable task, T result);
@Override
<T> Future<T> submit(Callable<T> task);

@Override
@Deprecated
void shutdown();
@Override
@Deprecated
List<Runnable> shutdownNow();

// ========== 实现自 ScheduledExecutorService 接口 ==========

@Override
ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
@Override
<V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);
@Override
ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);
@Override
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);

```

+ submit返回的Future不是java的Future，而是netty自己实现的Future
+ EventExecutorGroup把任务分配给具体的某一个EventExecutor去执行，调用`next()`方法来选择一个EventExecutor

### 3. AbstractEventExecutorGroup

`AbstractEventExecutorGroup` 实现了 `EventExecutorGroup` 接口。

#### 3.1 submit

`submit(...)`提交一个任务到EventExecutor中。
```java
@Override
public Future<?> submit(Runnable task) {
    return next().submit(task);
}

@Override
public <T> Future<T> submit(Runnable task, T result) {
    return next().submit(task, result);
}

@Override
public <T> Future<T> submit(Callable<T> task) {
    return next().submit(task);
}
```
+ 通过`next()`来选择一个EventExecutor

#### 3.2 schedule

`schedule(...)`提交一个定时任务到EventExector
```java
@Override
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
    return next().schedule(command, delay, unit);
}

@Override
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
    return next().schedule(callable, delay, unit);
}

@Override
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
    return next().scheduleAtFixedRate(command, initialDelay, period, unit);
}

@Override
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
    return next().scheduleWithFixedDelay(command, initialDelay, delay, unit);
}
```

#### 3.3 execute

`execute(...)`方法，选择一个EventExecutor执行一个任务
```java
@Override
public void execute(Runnable command) {
    next().execute(command);
}
```
+ `execute(...)`方法和`submit(...)`方法看起来相似，实际区别由EventExecutor决定。

#### 3.4 invokeAll
`invokeAll(...)`在一个EventExecutor里执行多个任务
```java
@Override
public <T> List<java.util.concurrent.Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {
    return next().invokeAll(tasks);
}

@Override
public <T> List<java.util.concurrent.Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
    return next().invokeAll(tasks, timeout, unit);
}
```
+ 这里返回的Future是java原生的Future

#### 3.5 invokeAny
`invokeAny(...)`在EventExecutor中执行多个任务，有一个执行成功即可。
```java
@Override
public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
    return next().invokeAny(tasks);
}

@Override
public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    return next().invokeAny(tasks, timeout, unit);
}
```

#### 3.6 shutdown

`shutdown(...)`方法，关闭EventExecutorGroup
```java
@Override
public Future<?> shutdownGracefully() {
    return shutdownGracefully(DEFAULT_SHUTDOWN_QUIET_PERIOD /* 2 */, DEFAULT_SHUTDOWN_TIMEOUT /* 15 */, TimeUnit.SECONDS);
}

@Override
@Deprecated
public List<Runnable> shutdownNow() {
    shutdown();
    return Collections.emptyList();
}

@Override
@Deprecated
public abstract void shutdown();
```
+ 具体的 `shutdownGracefully(long quietPeriod, long timeout, TimeUnit unit)` 和 `shutdown()` 方法，由子类实现。

### 4. MultithreadEventExecutorGroup

继承 AbstractEventExecutorGroup 抽象类

4.1 构造方法

```java
/**
 * EventExecutor 数组
 */
private final EventExecutor[] children;
/**
 * 不可变( 只读 )的 EventExecutor 数组
 */
private final Set<EventExecutor> readonlyChildren;
/**
 * 已终止的 EventExecutor 数量
 */
private final AtomicInteger terminatedChildren = new AtomicInteger();
/**
 * 用于终止 EventExecutor 的异步 Future
 */
private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
/**
 * EventExecutor 选择器 ！这个chooser实例由后面实例化的方式决定
 */
private final EventExecutorChooserFactory.EventExecutorChooser chooser;

protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    this(nThreads, threadFactory == null ? null : new ThreadPerTaskExecutor(threadFactory), args);
}

protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}

  1: protected MultithreadEventExecutorGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
  2:     if (nThreads <= 0) {
  3:         throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
  4:     }
  5: 
  6:     // 创建执行器
  7:     if (executor == null) {
  8:         executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
  9:     }
 10: 
 11:     // 创建 EventExecutor 数组
 12:     children = new EventExecutor[nThreads];
 13: 
 14:     for (int i = 0; i < nThreads; i ++) {
 15:         boolean success = false; // 是否创建成功
 16:         try {
 17:             // 创建 EventExecutor 对象
 18:             children[i] = newChild(executor, args);
 19:             // 标记创建成功
 20:             success = true;
 21:         } catch (Exception e) {
 22:             // 创建失败，抛出 IllegalStateException 异常
 23:             // TODO: Think about if this is a good exception type
 24:             throw new IllegalStateException("failed to create a child event loop", e);
 25:         } finally {
 26:             // 创建失败，关闭所有已创建的 EventExecutor
 27:             if (!success) {
 28:                 // 关闭所有已创建的 EventExecutor
 29:                 for (int j = 0; j < i; j ++) {
 30:                     children[j].shutdownGracefully();
 31:                 }
 32:                 // 确保所有已创建的 EventExecutor 已关闭
 33:                 for (int j = 0; j < i; j ++) {
 34:                     EventExecutor e = children[j];
 35:                     try {
 36:                         while (!e.isTerminated()) {
 37:                             e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
 38:                         }
 39:                     } catch (InterruptedException interrupted) {
 40:                         // Let the caller handle the interruption.
 41:                         Thread.currentThread().interrupt();
 42:                         break;
 43:                     }
 44:                 }
 45:             }
 46:         }
 47:     }
 48: 
 49:     // 创建 EventExecutor 选择器
 50:     chooser = chooserFactory.newChooser(children);
 51: 
 52:     // 创建监听器，用于 EventExecutor 终止时的监听
 53:     final FutureListener<Object> terminationListener = new FutureListener<Object>() {
 54: 
 55:         @Override
 56:         public void operationComplete(Future<Object> future) throws Exception {
 57:             if (terminatedChildren.incrementAndGet() == children.length) { // 全部关闭
 58:                 terminationFuture.setSuccess(null); // 设置结果，并通知监听器们。
 59:             }
 60:         }
 61: 
 62:     };
 63:     // 设置监听器到每个 EventExecutor 上
 64:     for (EventExecutor e: children) {
 65:         e.terminationFuture().addListener(terminationListener);
 66:     }
 67: 
 68:     // 创建不可变( 只读 )的 EventExecutor 数组
 69:     Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
 70:     Collections.addAll(childrenSet, children);
 71:     readonlyChildren = Collections.unmodifiableSet(childrenSet);
 72: }
```
+ 55-60行，当所有 EventExecutor 都终止完成时，通过调用 Future#setSuccess(V result) 方法，通知监听器们。
+ 68-71行：创建不可变( 只读 )的 EventExecutor 数组。

#### 4.2 ThreadPerTaskExecutor

io.netty.util.concurrent.ThreadPerTaskExecutor 实现了Executor接口，为每个任务创建一个线程。
```java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}

```

#### 4.3 EventExecutorChooserFactory
`EventExecutorChooserFactory`工厂接口

```java
/**
 * Factory that creates new {@link EventExecutorChooser}s.
 */
@UnstableApi
public interface EventExecutorChooserFactory {

    /**
     * Returns a new {@link EventExecutorChooser}.
     */
    EventExecutorChooser newChooser(EventExecutor[] executors);

    /**
     * Chooses the next {@link EventExecutor} to use.
     */
    @UnstableApi
    interface EventExecutorChooser {

        /**
         * Returns the new {@link EventExecutor} to use.
         */
        EventExecutor next();
    }
}
```
+ `newChooser(EventExecutor[] executors)` 方法，创建一个 EventExecutorChooser 对象。
+ `EventExecutorChooser#next()` 返回一个EventExecutor对象

#### 4.3.1 DefaultEventExecutorChooserFactory
`DefaultEventExecutorChooserFactory`实现了`EventExecutorChooserFactory`接口
```java
// 单例模式
public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

private DefaultEventExecutorChooserFactory() { }

@SuppressWarnings("unchecked")
@Override
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) { // 是否为 2 的幂次方
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}

private static boolean isPowerOfTwo(int val) {
    return (val & -val) == val;
}
```

#### 4.3.2 GenericEventExecutorChooser

GenericEventExecutorChooser 实现 EventExecutorChooser 接口，通用的 EventExecutor 选择器实现类。当executors的数量不是2的幂次方的时候使用。
```java
private static final class GenericEventExecutorChooser implements EventExecutorChooser {

    /**
     * 自增序列
     */
    private final AtomicInteger idx = new AtomicInteger();
    /**
     * EventExecutor 数组
     */
    private final EventExecutor[] executors;

    GenericEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[Math.abs(idx.getAndIncrement() % executors.length)];
    }

}
```
+ `next()`方法返回下一个`EventExecutor`,将计数器`idx`加一，再取模executors的数量，这就是简单的循环取`EventExecutor`数组中的对象。

#### 4.3.3 PowerOfTwoEventExecutorChooser

`PowerOfTwoEventExecutorChooser` 实现 `EventExecutorChooser` 接口，当executors数量为2的幂次方时使用,这是一个优化的实现

```java
private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {

    /**
     * 自增序列
     */
    private final AtomicInteger idx = new AtomicInteger();
    /**
     * EventExecutor 数组
     */
    private final EventExecutor[] executors;

    PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[idx.getAndIncrement() & executors.length - 1];
    }

}
```
+ 在`next()`中，`idx`与上`executors.length - 1`这个算法的结果和取模操作的结果一样，但是与操作比取余操作性能高很多。

#### 4.4 newDefaultThreadFactory
`newDefaultThreadFactory`方法创建线程工厂对象
```java 
protected ThreadFactory newDefaultThreadFactory() {
    return new DefaultThreadFactory(getClass());
}
```
+ 创建的对象为 DefaultThreadFactory ，并且使用类名作为 poolType 。

#### 4.5 iterator
`iterator`方法，获得 EventExecutor 数组的迭代器
```java
@Override
public Iterator<EventExecutor> iterator() {
    return readonlyChildren.iterator();
}
```
+ 返回的是不可变的 EventExecutor 数组 `readonlyChildren` 的迭代器。为了防止修改，是只读的。

#### 4.6 newChild
`newChild`这个抽象方法，用来创建EventExecutor 对象
```java
protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;
```
+ 子类去实现该方法，创建对应的EventExecutor实现类对象。

#### 4.7 awaitTermination
等待整个EventExecutorGroup结束，代码如下
```java
public boolean awaitTermination(long timeout, TimeUnit unit)
		throws InterruptedException {
	long deadline = System.nanoTime() + unit.toNanos(timeout);
	loop: for (EventExecutor l: children) {
		for (;;) {
			long timeLeft = deadline - System.nanoTime();
			if (timeLeft <= 0) {
				// 时间到，退出外层循环
				break loop;
			}
			if (l.awaitTermination(timeLeft, TimeUnit.NANOSECONDS)) {
				// 若当前EventExecutor等待结束，并成功，就遍历下一个EventExecutor,若失败，且timeLeft还大于0，继续等待当前线程
				break;
			}
		}
	}
	return isTerminated();
}
```

### 5. EventLoopGroup
`EventExecutorGroup`继承了EventExecutorGroup。
```java
public interface EventLoopGroup extends EventExecutorGroup {
    /**
     * 覆盖父类next()方法，现在返回一个EventLoop
     */
    @Override
    EventLoop next();
	==================自定义的三个注册channel到register的方法==================
    //注册完成后，ChannelFuture会得到通知
    ChannelFuture register(Channel channel);

    ChannelFuture register(ChannelPromise promise);

    @Deprecated
    ChannelFuture register(Channel channel, ChannelPromise promise);
}

```

### 6. MultithreadEventLoopGroup
`MultithreadEventLoopGroup`继承`MultithreadEventExecutorGroup`，实现`EventLoopGroup`

#### 6.1 构造方法

```java
/**
 * 默认 EventLoop 线程数
 */
private static final int DEFAULT_EVENT_LOOP_THREADS;

static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}

protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}

protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}

protected MultithreadEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, chooserFactory, args);
}
```
+ 默认线程数为cpu数*2，因为现在的cpu都是每个核对应两个线程

#### 6.2 newDefaultThreadFactory
`newDefaultThreadFactory()`方法创建线程工厂对象
```java
@Override
protected ThreadFactory newDefaultThreadFactory() {
    return new DefaultThreadFactory(getClass(), Thread.MAX_PRIORITY);
}
```
+ 覆盖父类方法，将线程的优先级设置为最大

#### 6.3 next
`next()`方法，返回下一个EventLoop对象
```java
@Override
public EventLoop next() {
    return (EventLoop) super.next();
}
```
+ 覆盖父类方法，将返回值转为EventLoop对象

#### 6.4 newChild
`newChild()`抽象方法，创建 EventExecutor 对象。
```java
@Override
protected abstract EventLoop newChild(Executor executor, Object... args) throws Exception;
```
+ 覆盖父类方法，返回值改为 EventLoop 类

#### 6.5 register
`register()`方法，将channel注册到EventLoopGroup中，具体的，会调用next()去取一个EventLoop，将channel注册到该EventLoop上

### 7. NioEventLoopGroup
`NioEventLoopGroup`继承`MultithreadEventLoopGroup`抽象类

#### 7.1 构造函数

```java
public NioEventLoopGroup() {
    this(0);
}

public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}

public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory) {
    this(nThreads, threadFactory, SelectorProvider.provider());
}

public NioEventLoopGroup(int nThreads, Executor executor) {
    this(nThreads, executor, SelectorProvider.provider());
}

public NioEventLoopGroup(
        int nThreads, ThreadFactory threadFactory, final SelectorProvider selectorProvider) {
    this(nThreads, threadFactory, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}

public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory,
    final SelectorProvider selectorProvider, final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, threadFactory, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}

public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider) {
    this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}

public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}

public NioEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory,
                         final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, chooserFactory, selectorProvider, selectStrategyFactory,
            RejectedExecutionHandlers.reject());
}

public NioEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory,
                         final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory,
                         final RejectedExecutionHandler rejectedExecutionHandler) {
    super(nThreads, executor, chooserFactory, selectorProvider, selectStrategyFactory, rejectedExecutionHandler);
}
```
+ 这里有很多构造函数参数，是用来具体父类的`Object ... args`参数
+ `SelectorProvider selectorProvider` 用于创建 Java NIO Selector 对象。
+ `SelectStrategyFactory selectStrategyFactory`, 选择策略工厂。

#### 7.2 newChild
`newChild()`方法,创建NioEventLoop对象。
```java
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor,
            (SelectorProvider) args[0], ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
```
+ 把args参数传给NioEventLoop的构造方法

#### 7.3 setIoRatio
`setIoRatio(int ioRatio)` 方法，设置所有 EventLoop 的 IO 任务占用执行时间的比例
```java
/**
 * Sets the percentage of the desired amount of time spent for I/O in the child event loops.  The default value is
 * {@code 50}, which means the event loop will try to spend the same amount of time for I/O as for non-I/O tasks.
 */
public void setIoRatio(int ioRatio) {
    for (EventExecutor e: this) {
        ((NioEventLoop) e).setIoRatio(ioRatio);
    }
}
```

#### 7.4 rebuildSelectors
`rebuildSelectors()` 方法，重建所有 EventLoop 的 Selector 对象
```java
/**
 * Replaces the current {@link Selector}s of the child event loops with newly created {@link Selector}s to work
 * around the  infamous epoll 100% CPU bug.
 */
public void rebuildSelectors() {
    for (EventExecutor e: this) {
        ((NioEventLoop) e).rebuildSelector();
    }
}
```
+ 用来处理jdk epoll 100% bug,当NioEventLoop触发该bug，会调用rebuildSelectors(),进行重建 Selector 对象，以修复该问题。