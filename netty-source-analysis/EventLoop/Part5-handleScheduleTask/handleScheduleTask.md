### **1. 概述**

本文接 Part4-handleNormalTask，分享【处理**定时任务**】的部分。

### 2. ScheduledFutureTask

`io.netty.util.concurrent.ScheduledFutureTask` ，实现 ScheduledFuture、PriorityQueueNode 接口，继承 PromiseTask 抽象类，Netty 定时任务。

### 2.1 静态属性

```java
/**
 * 任务序号生成器，通过 AtomicLong 实现递增发号
 */
private static final AtomicLong nextTaskId = new AtomicLong();
/**
 * 定时任务时间起点
 */
private static final long START_TIME = System.nanoTime();
```

+ `nextTaskId` 静态属性，任务序号生成器，通过 AtomicLong 实现**递增**发号。
+ `START_TIME` 静态属性，定时任务时间**起点**。在 ScheduledFutureTask 中，定时任务的执行时间，都是基于 `START_TIME` 做**相对**时间。因为是定时调度，我改了系统时间也没关系，存的是距离下次调度还要多长时间
  不受系统时间影响。

### **2.2 nanoTime**

`#nanoTime()` **静态**方法，获得当前时间，这个是相对 `START_TIME` 来算的。代码如下：

```java
static long nanoTime() {
    return System.nanoTime() - START_TIME;
}
```

+ 这是个重要的方法，后续很多方法都会调用到它。

### 2.3 deadlineNanos

`#deadlineNanos(long delay)` **静态**方法，获得任务执行时间，这个也是相对 `START_TIME` 来算的。代码如下：

```java
/**
 * @param delay 延迟时长，单位：纳秒
 * @return 获得任务执行时间，也是相对 {@link #START_TIME} 来算的。
 *          实际上，返回的结果，会用于 {@link #deadlineNanos} 字段
 */
static long deadlineNanos(long delay) {
    long deadlineNanos = nanoTime() + delay;
    // Guard against overflow 防御性编程
    return deadlineNanos < 0 ? Long.MAX_VALUE : deadlineNanos;
}
```

### 2.4 构造方法

```java
/**
 * 任务编号
 */
private final long id = nextTaskId.getAndIncrement();
/**
 * 任务执行时间，即到了该时间，该任务就会被执行
 */
private long deadlineNanos;
/**
 * 任务执行周期
 *
 * =0 - 只执行一次
 * >0 - 按照计划执行时间计算
 * <0 - 按照实际执行时间计算
 *
 * 推荐阅读文章 https://blog.csdn.net/gtuu0123/article/details/6040159
 */
/* 0 - no repeat, >0 - repeat at fixed rate, <0 - repeat with fixed delay */
private final long periodNanos;
/**
 * 队列编号
 */
private int queueIndex = INDEX_NOT_IN_QUEUE;

ScheduledFutureTask(
        AbstractScheduledEventExecutor executor,
        Runnable runnable, V result, long nanoTime) {
    this(executor, toCallable(runnable, result), nanoTime);
}

ScheduledFutureTask(
        AbstractScheduledEventExecutor executor,
        Callable<V> callable, long nanoTime, long period) {
    super(executor, callable);
    if (period == 0) {
        throw new IllegalArgumentException("period: 0 (expected: != 0)");
    }
    deadlineNanos = nanoTime;
    periodNanos = period;
}

ScheduledFutureTask(
        AbstractScheduledEventExecutor executor,
        Callable<V> callable, long nanoTime) {
    super(executor, callable);
    deadlineNanos = nanoTime;
    periodNanos = 0;
}
```

### 2.5 delayNanos

`#delayNanos(...)` 方法，获得距离指定时间，还要多久可执行。代码如下：

```java
/**
 * @return 距离当前时间，还要多久可执行。若为负数，直接返回 0
 */
public long delayNanos() {
    return Math.max(0, deadlineNanos() - nanoTime());
}

/**
 * @param currentTimeNanos 指定时间
 * @return 距离指定时间，还要多久可执行。若为负数，直接返回 0
 */
public long delayNanos(long currentTimeNanos) {
    return Math.max(0, deadlineNanos() - (currentTimeNanos - START_TIME));
}

@Override
public long getDelay(TimeUnit unit) {
    return unit.convert(delayNanos(), TimeUnit.NANOSECONDS);
}
```

### 2.6 run

`#run()` 方法，执行定时任务。代码如下：

```java
 1: @Override
 2: public void run() {
 3:     assert executor().inEventLoop();
 4:     try {
 5:         if (periodNanos == 0) {
 6:             // 设置任务不可取消
 7:             if (setUncancellableInternal()) {
 8:                 // 执行任务
 9:                 V result = task.call();
10:                 // 通知任务执行成功
11:                 setSuccessInternal(result);
12:             }
13:         } else {
14:             // 判断任务并未取消
15:             // check if is done as it may was cancelled
16:             if (!isCancelled()) {
17:                 // 执行任务
18:                 task.call();
19:                 if (!executor().isShutdown()) {
20:                     // 计算下次执行时间
21:                     long p = periodNanos;
22:                     if (p > 0) {
23:                         deadlineNanos += p;
24:                     } else {
25:                         deadlineNanos = nanoTime() - p;
26:                     }
27:                     // 判断任务并未取消
28:                     if (!isCancelled()) {
29:                         // 重新添加到任务队列，等待下次定时执行
30:                         // scheduledTaskQueue can never be null as we lazy init it before submit the task!
31:                         Queue<ScheduledFutureTask<?>> scheduledTaskQueue =
32:                                 ((AbstractScheduledEventExecutor) executor()).scheduledTaskQueue;
33:                         assert scheduledTaskQueue != null;
34:                         scheduledTaskQueue.add(this);
35:                     }
36:                 }
37:             }
38:         }
39:     // 发生异常，通知任务执行失败
40:     } catch (Throwable cause) {
41:         setFailureInternal(cause);
42:     }
43: }
```

- 第 3 行：校验，必须在 EventLoop 的线程中。
- 根据不同的任务执行周期 `periodNanos` ，在执行任务会略有不同。当然，大体是相同的。
- 第 5 至 12 行：执行周期为“**只执行一次**”的定时任务。
  - 第 7 行：调用 `PromiseTask#setUncancellableInternal()` 方法，设置任务不可取消。
  - 第 9 行：【重要】调用 `Callable#call()` 方法，执行任务。
  - 第 11 行：调用 `PromiseTask#setSuccessInternal(V result)` 方法，回调通知注册在定时任务上的监听器。为什么能这么做呢？因为 ScheduledFutureTask 继承了 PromiseTask 抽象类。
- 第 13 至 38 行：执行周期为“**固定周期**”的定时任务。
  - 第 16 行：调用 `DefaultPromise#isCancelled()` 方法，判断任务是否已经取消。
  - 第 18 行：【重要】调用 `Callable#call()` 方法，执行任务。
  - 第 19 行：判断 EventExecutor 并未关闭。
  - 第 20 至 26 行：计算下次定时执行的时间。不同的执行 `fixed` 方式，计算方式不同。其中【第 25 行】的 `- p` 的代码，因为 `p` 是负数，所以通过**负负得正**来计算。另外，这块会修改定时任务的 `deadlineNanos` 属性，从而变成新的定时任务执行时间。
  - 第 29 至 34 行：重新添加到定时任务队列 `scheduledTaskQueue` 中，等待下次定时执行。
- 第 39 至 42 行：发生异常，调用 `PromiseTask#setFailureInternal(Throwable cause)` 方法，回调通知注册在定时任务上的监听器。

### **2.7 cancel**

有两个方法，可以取消定时任务。代码如下：

```java
@Override
public boolean cancel(boolean mayInterruptIfRunning) {
    boolean canceled = super.cancel(mayInterruptIfRunning);
    // 取消成功，移除出定时任务队列
    if (canceled) {
        ((AbstractScheduledEventExecutor) executor()).removeScheduled(this);
    }
    return canceled;
}

// 移除任务
boolean cancelWithoutRemove(boolean mayInterruptIfRunning) {
    return super.cancel(mayInterruptIfRunning);
}

```

- 差别在于，是否 调用 `AbstractScheduledEventExecutor#removeScheduled(ScheduledFutureTask)` 方法，从定时任务队列移除自己。

### 2.8 compareTo

`#compareTo(Delayed o)` 方法，用于队列( ScheduledFutureTask 使用 PriorityQueue 作为**优先级队列** )排序。代码如下：

```java
@Override
public int compareTo(Delayed o) {
    if (this == o) {
        return 0;
    }

    ScheduledFutureTask<?> that = (ScheduledFutureTask<?>) o;
    long d = deadlineNanos() - that.deadlineNanos();
    if (d < 0) {
        return -1;
    } else if (d > 0) {
        return 1;
    } else if (id < that.id) {
        return -1;
    } else if (id == that.id) {
        throw new Error();
    } else {
        return 1;
    }
}
```

- 按照 `deadlineNanos`、`id` 属性**升序**排序。

### 2.9 priorityQueueIndex

`#priorityQueueIndex(...)` 方法，获得或设置 `queueIndex` 属性。代码如下：

```java
@Override
public int priorityQueueIndex(DefaultPriorityQueue<?> queue) { // 获得
    return queueIndex;
}

@Override
public void priorityQueueIndex(DefaultPriorityQueue<?> queue, int i) { // 设置
    queueIndex = i;
}
```

- 因为 ScheduledFutureTask 实现 PriorityQueueNode 接口，所以需要实现这两个方法。

### 3. AbstractScheduledEventExecutor

`io.netty.util.concurrent.AbstractScheduledEventExecutor` ，继承 AbstractEventExecutor 抽象类，**支持定时任务**的 EventExecutor 的抽象类。

### 3.1 构造方法

```java
/**
 * 定时任务队列
 */
PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue;

protected AbstractScheduledEventExecutor() {
}

protected AbstractScheduledEventExecutor(EventExecutorGroup parent) {
    super(parent);
}
```

- `scheduledTaskQueue` 属性，定时任务队列。

### 3.2 scheduledTaskQueue

`#scheduledTaskQueue()` 方法，获得定时任务队列。若未初始化，则进行创建。代码如下：

```java
/**
 * 定时任务排序器
 */
private static final Comparator<ScheduledFutureTask<?>> SCHEDULED_FUTURE_TASK_COMPARATOR =
        new Comparator<ScheduledFutureTask<?>>() {
            @Override
            public int compare(ScheduledFutureTask<?> o1, ScheduledFutureTask<?> o2) {
                return o1.compareTo(o2); //
            }
        };

PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue() {
    if (scheduledTaskQueue == null) {
        scheduledTaskQueue = new DefaultPriorityQueue<ScheduledFutureTask<?>>(
                SCHEDULED_FUTURE_TASK_COMPARATOR,
                // Use same initial capacity as java.util.PriorityQueue
                11);
    }
    return scheduledTaskQueue;
}
```

- 创建的队列是 `io.netty.util.internal.DefaultPriorityQueue` 类型。具体的代码实现，本文先不解析。在这里，我们只要知道它是一个**优先级**队列，通过 `SCHEDULED_FUTURE_TASK_COMPARATOR` 来比较排序 ScheduledFutureTask 的任务优先级( 顺序 )。
- `SCHEDULED_FUTURE_TASK_COMPARATOR` 的具体实现，是调用 「2.8 compareTo」方法来实现，所以队列**首个**任务，就是**第一个**需要执行的定时任务。

### **3.3 nanoTime**

`#nanoTime()` **静态**方法，获得当前时间。代码如下：

```java
protected static long nanoTime() {
    return ScheduledFutureTask.nanoTime();
}
```

- 在方法内部，会调用 「2.2 nanoTime」方法。

### 3.4 schedule

`#schedule(final ScheduledFutureTask<V> task)` 方法，提交定时任务。代码如下：

```java
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {
        // 添加到定时任务队列
        scheduledTaskQueue().add(task);
    } else {
        // 通过 EventLoop 的线程，添加到定时任务队列
        execute(new Runnable() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task);
            }
        });
    }
    return task;
}
```

- 必须在 EventLoop 的线程中，**才能**添加到定时任务到队列中。

在 ScheduledFutureTask 中，有四个方法，会调用 `#schedule(final ScheduledFutureTask<V> task)` 方法，分别创建 **3** 种不同类型的定时任务。代码如下：

```java
@Override
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit) {
    ObjectUtil.checkNotNull(callable, "callable");
    ObjectUtil.checkNotNull(unit, "unit");
    if (delay < 0) {
        delay = 0;
    }
    // 无视，已经废弃
    validateScheduled0(delay, unit);

    return schedule(new ScheduledFutureTask<V>(
            this, callable, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
}

@Override
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
    ObjectUtil.checkNotNull(command, "command");
    ObjectUtil.checkNotNull(unit, "unit");
    if (initialDelay < 0) {
        throw new IllegalArgumentException(
                String.format("initialDelay: %d (expected: >= 0)", initialDelay));
    }
    if (period <= 0) {
        throw new IllegalArgumentException(
                String.format("period: %d (expected: > 0)", period));
    }
    // 无视，已经废弃
    validateScheduled0(initialDelay, unit);
    validateScheduled0(period, unit);

    return schedule(new ScheduledFutureTask<Void>(
            this, Executors.<Void>callable(command, null), // Runnable => Callable
            ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), unit.toNanos(period)));
}

@Override
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) {
    ObjectUtil.checkNotNull(command, "command");
    ObjectUtil.checkNotNull(unit, "unit");
    if (initialDelay < 0) {
        throw new IllegalArgumentException(
                String.format("initialDelay: %d (expected: >= 0)", initialDelay));
    }
    if (delay <= 0) {
        throw new IllegalArgumentException(
                String.format("delay: %d (expected: > 0)", delay));
    }
    // 无视，已经废弃
    validateScheduled0(initialDelay, unit);
    validateScheduled0(delay, unit);

    return schedule(new ScheduledFutureTask<Void>(
            this, Executors.<Void>callable(command, null), // Runnable => Callable
            ScheduledFutureTask.deadlineNanos(unit.toNanos(initialDelay)), -unit.toNanos(delay)));
}
```

- 每个方法，前面都是校验参数的代码，重点是在最后对 `#schedule(final ScheduledFutureTask<V> task)` 方法的调用。

### 3.5 removeScheduled

`#removeScheduled(final ScheduledFutureTask<?> task)` 方法，移除出定时任务队列。代码如下：

```java
final void removeScheduled(final ScheduledFutureTask<?> task) {
    if (inEventLoop()) {
        // 移除出定时任务队列
        scheduledTaskQueue().removeTyped(task);
    } else {
        // 通过 EventLoop 的线程，移除出定时任务队列
        execute(new Runnable() {
            @Override
            public void run() {
                removeScheduled(task);
            }
        });
    }
}
```

- 必须在 EventLoop 的线程中，**才能**移除出定时任务队列。

### 3.6 hasScheduledTasks

`#hasScheduledTasks()` 方法，判断是否有可执行的定时任务。代码如下：

```java
/**
 * Returns {@code true} if a scheduled task is ready for processing.
 */
protected final boolean hasScheduledTasks() {
    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    // 获得队列首个定时任务。不会从队列中，移除该任务
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    // 判断该任务是否到达可执行的时间
    return scheduledTask != null && scheduledTask.deadlineNanos() <= nanoTime();
}
```

### 3.7 peekScheduledTask

```java
final ScheduledFutureTask<?> peekScheduledTask() {
    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    if (scheduledTaskQueue == null) {
        return null;
    }
    return scheduledTaskQueue.peek();
}
```

### 3.8 nextScheduledTaskNano

`#nextScheduledTaskNano()` 方法，获得定时任务队列，距离当前时间，还要多久可执行。

- 若队列**为空**，则返回 `-1` 。
- 若队列**非空**，若为负数，直接返回 0 。实际等价，ScheduledFutureTask#delayNanos() 方法。

代码如下：

```java
/**
 * Return the nanoseconds when the next scheduled task is ready to be run or {@code -1} if no task is scheduled.
 */
protected final long nextScheduledTaskNano() {
    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    // 获得队列首个定时任务。不会从队列中，移除该任务
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    if (scheduledTask == null) {
        return -1;
    }
    // 距离当前时间，还要多久可执行。若为负数，直接返回 0 。实际等价，ScheduledFutureTask#delayNanos() 方法。
    return Math.max(0, scheduledTask.deadlineNanos() - nanoTime());
}
```

### 3.9 pollScheduledTask

`#pollScheduledTask(...)` 方法，获得指定时间内，定时任务队列**首个**可执行的任务，并且从队列中移除。代码如下：

```java
/**
 * @see #pollScheduledTask(long)
 */
protected final Runnable pollScheduledTask() {
    return pollScheduledTask(nanoTime()); // 当前时间
}

/**
 * Return the {@link Runnable} which is ready to be executed with the given {@code nanoTime}.
 * You should use {@link #nanoTime()} to retrieve the correct {@code nanoTime}.
 */
protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop();

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    // 获得队列首个定时任务。不会从队列中，移除该任务
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    // 直接返回，若获取不到
    if (scheduledTask == null) {
        return null;
    }

    // 在指定时间内，则返回该任务
    if (scheduledTask.deadlineNanos() <= nanoTime) {
        scheduledTaskQueue.remove(); // 移除任务
        return scheduledTask;
    }
    return null;
}
```

### 3.10 cancelScheduledTasks

`#cancelScheduledTasks()` 方法，取消定时任务队列的所有任务。代码如下：

```java
/**
 * Cancel all scheduled tasks.
 * <p>
 * This method MUST be called only when {@link #inEventLoop()} is {@code true}.
 */
protected void cancelScheduledTasks() {
    assert inEventLoop();

    // 若队列为空，直接返回
    PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    if (isNullOrEmpty(scheduledTaskQueue)) {
        return;
    }

    // 循环，取消所有任务
    final ScheduledFutureTask<?>[] scheduledTasks = scheduledTaskQueue.toArray(new ScheduledFutureTask<?>[0]);
    for (ScheduledFutureTask<?> task : scheduledTasks) {
        task.cancelWithoutRemove(false);
    }

    scheduledTaskQueue.clearIgnoringIndexes();
}

private static boolean isNullOrEmpty(Queue<ScheduledFutureTask<?>> queue) {
    return queue == null || queue.isEmpty();
}

```

### 4. SingleThreadEventExecutor

`#fetchFromScheduledTaskQueue()` 方法，将定时任务队列 `scheduledTaskQueue` 到达可执行的任务，添加到任务队列 `taskQueue` 中。代码如下：

```java
private boolean fetchFromScheduledTaskQueue() {
    // 获得当前时间
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    // 获得指定时间内，定时任务队列**首个**可执行的任务，并且从队列中移除。
    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    // 不断从定时任务队列中，获得
    while (scheduledTask != null) {
        // 将定时任务添加到 taskQueue 中。若添加失败，则结束循环，返回 false ，表示未获取完所有课执行的定时任务
        if (!taskQueue.offer(scheduledTask)) {
            // 将定时任务添加回 scheduledTaskQueue 中
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
        // 获得指定时间内，定时任务队列**首个**可执行的任务，并且从队列中移除。
        scheduledTask  = pollScheduledTask(nanoTime);
    }
    // 返回 true ，表示获取完所有可执行的定时任务
    return true;
}

```

