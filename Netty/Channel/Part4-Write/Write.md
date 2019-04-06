## Channel（四）之 write 操作

### **1. 概述**

本文分享 Netty NioSocketChannel **写入**对端数据的过程。和**写入**相关的，在 Netty Channel 有三种 API 方法：

```java
ChannelFuture write(Object msg)
ChannelFuture write(Object msg, ChannelPromise promise);

ChannelOutboundInvoker flush();

ChannelFuture writeAndFlush(Object msg);
ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
```

原生的 Java NIO SocketChannel 只有一种 write 方法，将数据写到对端。而 Netty Channel 竟然有三种方法，我们来一个个看看：

- write 方法：将数据写到内存队列中。
  - 也就是说，此时数据**并没有**写入到对端。
- flush 方法：刷新内存队列，将其中的数据写入到对端。
  - 也就是说，此时数据才**真正**写到对端。
- writeAndFlush 方法：write + flush 的组合，将数据写到内存队列后，立即刷新内存队列，又将其中的数据写入到对端。
  - 也就是说，此时数据**已经**写到对端。

严格来说，上述的描述不是完全准确。因为 Netty Channel 的 `#write(Object msg, ...)` 和 `#writeAndFlush(Object msg, ...)` 方法，是**异步写入**的过程，需要通过监听返回的 ChannelFuture 来确实是真正写入。例如：

```java
// 方式一：异步监听
channel.write(msg).addListener(new ChannelFutureListener() {
                
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        // ... 相关逻辑，例如是否成功？    
    }
    
});

// 方式二：同步异步写入结果
channel.write(msg).sync();
```

- 所以，实际在使用时，一定要注意。如果感兴趣，可以看看 Dubbo 和 Motan 等等 RPC 框架是怎么使用这个 API 方法的。
-  **有一点一定非常肯定要注意**，`#write(Object msg, ...)` 方法返回的 Promise 对象，只有在数据真正被 `#flush()` 方法调用执行完成后，才会被回调通知。

---

考虑到 Netty NioSocketChannel **写入**对端数据的代码太多，所以笔者拆成 write 和 flush 相关的两篇文章。所以，本文当然是 write 相关的文章。当然，这两个操作相关性很高，所以本文也会包括 flush 部分的内容。

### 2. AbstractChannel

AbstractChannel 对 `#write(Object msg, ...)` 方法的实现，代码如下：

```java
@Override
public ChannelFuture write(Object msg) {
    return pipeline.write(msg);
}

@Override
public ChannelFuture write(Object msg, ChannelPromise promise) {
    return pipeline.write(msg, promise);
}
```

+ 在方法内部，会调用对应的 `ChannelPipeline#write(Object msg, ...)` 方法，将 write 事件在 pipeline 上传播。
+ 最终会传播 write 事件到 `head` 节点，将数据写入到内存队列中。

### 3. DefaultChannelPipeline

`DefaultChannelPipeline#write(Object msg, ...)` 方法，代码如下：

```java
@Override
public final ChannelFuture write(Object msg) {
    return tail.write(msg);
}

@Override
public final ChannelFuture write(Object msg, ChannelPromise promise) {
    return tail.write(msg, promise);
}
```

### 4. TailContext

TailContext 对 `TailContext#write(Object msg, ...)` 方法的实现，是从 AbstractChannelHandlerContext 抽象类继承，代码如下：

```java
1: @Override
 2: public ChannelFuture write(Object msg) {
 3:     return write(msg, newPromise());
 4: }
 5: 
 6: @Override
 7: public ChannelFuture write(final Object msg, final ChannelPromise promise) {
 8:     // 消息( 数据 )为空，抛出异常
 9:     if (msg == null) {
10:         throw new NullPointerException("msg");
11:     }
12: 
13:     try {
14:         // 判断是否为合法的 Promise 对象
15:         if (isNotValidPromise(promise, true)) {
16:             // 释放消息( 数据 )相关的资源
17:             ReferenceCountUtil.release(msg);
18:             // cancelled
19:             return promise;
20:         }
21:     } catch (RuntimeException e) {
22:         // 发生异常，释放消息( 数据 )相关的资源
23:         ReferenceCountUtil.release(msg);
24:         throw e;
25:     }
26: 
27:     // 写入消息( 数据 )到内存队列
28:     write(msg, false, promise);
29:     return promise;
30: }
```

+ 在【第 2 行】的代码，我们可以看到，`#write(Object msg)` 方法，会调用 `#write(Object msg, ChannelPromise promise)` 方法。

  + 缺少的 `promise` 方法参数，通过调用 `#newPromise()` 方法，进行创建 Promise 对象，代码如下：

    ```java
    @Override
    public ChannelPromise newPromise() {
        return new DefaultChannelPromise(channel(), executor());
    }
    ```

    + 返回 DefaultChannelPromise 对象。

  + 在【第 29 行】的代码，返回的结果就是传入的 `promise` 对象。

+ 第 8 至 11 行：若消息( 消息 )为空，抛出异常。

+ 第 15 行：调用 `#isNotValidPromise(promise, true)` 方法，判断是否为**不合法**的 Promise 对象。

+ 第 17 行：调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。

+ 第 19 行：返回 `promise` 对象。一般情况下，出现这种情况是 `promise` 已经被取消，所以不再有必要写入数据。或者说，**写入数据的操作被取消**。

+ 第 21 至 25 行：若发生异常， 调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。最终，会抛出该异常。

+ 第 28 行：调用 `#write(Object msg, boolean flush, ChannelPromise promise)` 方法，写入消息( 数据 )到内存队列。代码如下：

  ```java
   1: private void write(Object msg, boolean flush, ChannelPromise promise) {
   2:     // 获得下一个 Outbound 节点
   3:     AbstractChannelHandlerContext next = findContextOutbound();
   4:     // 记录 Record 记录
   5:     final Object m = pipeline.touch(msg, next);
   6:     EventExecutor executor = next.executor();
   7:     // 在 EventLoop 的线程中
   8:     if (executor.inEventLoop()) {
   9:         // 执行 writeAndFlush 事件到下一个节点
  10:         if (flush) {
  11:             next.invokeWriteAndFlush(m, promise);
  12:         // 执行 write 事件到下一个节点
  13:         } else {
  14:             next.invokeWrite(m, promise);
  15:         }
  16:     } else {
  17:         AbstractWriteTask task;
  18:         // 创建 writeAndFlush 任务
  19:         if (flush) {
  20:             task = WriteAndFlushTask.newInstance(next, m, promise);
  21:         // 创建 write 任务
  22:         }  else {
  23:             task = WriteTask.newInstance(next, m, promise);
  24:         }
  25:         // 提交到 EventLoop 的线程中，执行该任务
  26:         safeExecute(executor, task, promise, m);
  27:     }
  28: }
  ```

  + 方法参数 `flush` 为 `true` 时，该方法执行的是 write + flush 的组合操作，即将数据写到内存队列后，立即刷新**内存队列**，又将其中的数据写入到对端。

  + 第 3 行：调用 `#findContextOutbound()` 方法，获得**下一个** Outbound 节点。

  + 第 5 行：调用 `DefaultChannelPipeline#touch(Object msg, AbstractChannelHandlerContext next)` 方法，记录 Record 记录。代码如下：

    ```java
    // DefaultChannelPipeline.java
    final Object touch(Object msg, AbstractChannelHandlerContext next) {
        return touch ? ReferenceCountUtil.touch(msg, next) : msg;
    }
    
    // ReferenceCountUtil.java
    /**
     * Tries to call {@link ReferenceCounted#touch(Object)} if the specified message implements
     * {@link ReferenceCounted}.  If the specified message doesn't implement {@link ReferenceCounted},
     * this method does nothing.
     */
    @SuppressWarnings("unchecked")
    public static <T> T touch(T msg, Object hint) {
        if (msg instanceof ReferenceCounted) {
            return (T) ((ReferenceCounted) msg).touch(hint);
        }
        return msg;
    }
    ```

    + 此方法与内存泄漏有关

  + 第 7 行：**在** EventLoop 的线程中。

    + 第 10 至 11 行：如果 `flush = true` 时，调用 `AbstractChannelHandlerContext#invokeWriteAndFlush()` 方法，执行 writeAndFlush 事件到下一个节点。
    + 第 12 至 15 行：如果 `flush = false` 时，调用 `AbstractChannelHandlerContext#invokeWrite()` 方法，执行 write 事件到下一个节点。
    + 后续的逻辑，和 **bind** 事件在 pipeline 中的传播是**基本一致**的。
    + 随着 write 或 writeAndFlush **事件**不断的向下一个节点传播，最终会到达 HeadContext 节点。

  + 第 16 行：**不在** EventLoop 的线程中。

    + 第 19 至 20 行：如果 `flush = true` 时，创建 WriteAndFlushTask 任务。
    + 第 21 至 24 行：如果 `flush = false` 时，创建 WriteTask 任务。
    + 第 26 行：调用 `#safeExecute(executor, task, promise, m)` 方法，提交到 EventLoop 的线程中，执行该任务。从而实现，**在** EventLoop 的线程中，执行 writeAndFlush 或 write 事件到下一个节点。

  + 第 29 行：返回 `promise` 对象。

### 5. HeadContext

在 pipeline 中，write 事件最终会到达 HeadContext 节点。而 HeadContext 的 `#write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，会处理该事件，代码如下：

```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```

+ 在方法内部，会调用 `AbstractUnsafe#write(Object msg, ChannelPromise promise)` 方法，将数据写到**内存队列**中。

### 6. AbstractUnsafe

`AbstractUnsafe#write(Object msg, ChannelPromise promise)` 方法，将数据写到**内存队列**中。代码如下：

```java
/**
 * 内存队列
 */
private volatile ChannelOutboundBuffer outboundBuffer = new ChannelOutboundBuffer(AbstractChannel.this);

  1: @Override
  2: public final void write(Object msg, ChannelPromise promise) {
  3:     assertEventLoop();
  4: 
  5:     ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
  6:     // 内存队列为空
  7:     if (outboundBuffer == null) {
  8:         // 内存队列为空，一般是 Channel 已经关闭，所以通知 Promise 异常结果
  9:         // If the outboundBuffer is null we know the channel was closed and so
 10:         // need to fail the future right away. If it is not null the handling of the rest
 11:         // will be done in flush0()
 12:         // See https://github.com/netty/netty/issues/2362
 13:         safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
 14:         // 释放消息( 对象 )相关的资源
 15:         // release message now to prevent resource-leak
 16:         ReferenceCountUtil.release(msg);
 17:         return;
 18:     }
 19: 
 20:     int size;
 21:     try {
 22:         // 过滤写入的消息( 数据 )
 23:         msg = filterOutboundMessage(msg);
 24:         // 计算消息的长度
 25:         size = pipeline.estimatorHandle().size(msg);
 26:         if (size < 0) {
 27:             size = 0;
 28:         }
 29:     } catch (Throwable t) {
 30:         // 通知 Promise 异常结果
 31:         safeSetFailure(promise, t);
 32:         // 释放消息( 对象 )相关的资源
 33:         ReferenceCountUtil.release(msg);
 34:         return;
 35:     }
 36: 
 37:     // 写入消息( 数据 )到内存队列
 38:     outboundBuffer.addMessage(msg, size, promise);
 39: }
```

+ `outboundBuffer` 属性，内存队列，用于缓存写入的数据( 消息 )。

+ 第 7 行：内存队列为空，一般是 Channel 已经关闭。

  + 调用 `#safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION)` 方法，通知 Promise 异常结果。
  + 第 16 行：调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。
  + 第 17 行：`return` ，结束执行。

+ 第 23 行：调用 `AbstractNioByteChannel#filterOutboundMessage(msg)` 方法，过滤写入的消息( 数据 )。代码如下：

  ```java
  // AbstractNioByteChannel.java
  
  @Override
  protected final Object filterOutboundMessage(Object msg) {
      // <1> ByteBuf 的情况
      if (msg instanceof ByteBuf) {
          ByteBuf buf = (ByteBuf) msg;
          // 已经是内存 ByteBuf
          if (buf.isDirect()) {
              return msg;
          }
  
          // 非内存 ByteBuf ，需要进行创建封装
          return newDirectBuffer(buf);
      }
  
      // <2> FileRegion 的情况
      if (msg instanceof FileRegion) {
          return msg;
      }
  
      // <3> 不支持其他类型
      throw new UnsupportedOperationException("unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
  }
  ```

  + `<1>` 处，消息( 数据 )是 ByteBuf 类型，如果是非 Direct ByteBuf 对象，需要调用 `#newDirectBuffer(ByteBuf)` 方法，复制封装成 Direct ByteBuf 对象。原因是：在使用 Socket 传递数据时性能很好，由于数据直接在内存中，不存在从 JVM 拷贝数据到直接缓冲区的过程，性能好。
  + `<2>` 处，消息( 数据 )是 FileRegion 类型，直接返回。
  + `<3>` 处，不支持其他数据类型。

+ 第 24 至 28 行：计算消息的长度。

+ 第 29 行：若发生异常时：

  + 第 31 行：调用 `#safeSetFailure(promise, Throwable t)` 方法，通知 Promise 异常结果。
  + 第 33 行：调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。
  + 第 34 行：`return` ，结束执行。

+ 第 38 行：调用 `ChannelOutboundBuffer#addMessage(msg, size, promise)` 方法，写入消息( 数据 )到内存队列。至此，write 操作，将数据写到**内存队列**中的过程。

### 7. AbstractWriteTask

AbstractWriteTask ，实现 Runnable 接口，写入任务**抽象类**。它有两个子类实现：

- WriteTask ，write 任务实现类。
- WriteAndFlushTask ，write + flush 任务实现类。

它们都是 AbstractChannelHandlerContext 的内部静态类。那么让我们先来 AbstractWriteTask 的代码。

### 7.1 构造方法

```java
/**
 * 提交任务时，是否计算 AbstractWriteTask 对象的自身占用内存大小
 */
private static final boolean ESTIMATE_TASK_SIZE_ON_SUBMIT = SystemPropertyUtil.getBoolean("io.netty.transport.estimateSizeOnSubmit", true);

/**
 * 每个 AbstractWriteTask 对象自身占用内存的大小。
 */
// Assuming a 64-bit JVM, 16 bytes object header, 3 reference fields and one int field, plus alignment
private static final int WRITE_TASK_OVERHEAD = SystemPropertyUtil.getInt("io.netty.transport.writeTaskSizeOverhead", 48);

private final Recycler.Handle<AbstractWriteTask> handle;
/**
 * pipeline 中的节点
 */
private AbstractChannelHandlerContext ctx;
/**
 * 消息( 数据 )
 */
private Object msg;
/**
 * Promise 对象
 */
private ChannelPromise promise;
/**
 * 对象大小
 */
private int size;

@SuppressWarnings("unchecked")
private AbstractWriteTask(Recycler.Handle<? extends AbstractWriteTask> handle) {
    this.handle = (Recycler.Handle<AbstractWriteTask>) handle;
}
```

+ `ESTIMATE_TASK_SIZE_ON_SUBMIT` **静态**字段，提交任务时，是否计算 AbstractWriteTask 对象的自身占用内存大小。
+ `WRITE_TASK_OVERHEAD` **静态**字段，每个 AbstractWriteTask 对象自身占用内存的大小。为什么占用的 48 字节呢？
  + `- 16 bytes object header` ，对象头，16 字节。
  + `- 3 reference fields` ，3 个**对象引用**字段，3 * 8 = 24 字节。
  + `- 1 int fields` ，1 个 `int` 字段，4 字节。
  + `padding` ，补齐 8 字节的整数倍，因此 4 字节。
  + 因此，合计 48 字节( 64 位的 JVM 虚拟机，并且不考虑压缩 )。
+ `handle` 字段，Recycler 处理器。而 Recycler 是 Netty 用来实现对象池的工具类。在网络通信中，写入是非常频繁的操作，因此通过 Recycler 重用 AbstractWriteTask 对象，减少对象的频繁创建，降低 GC 压力，提升性能。

### 7.2 init

`#init(AbstractWriteTask task, AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，初始化 AbstractWriteTask 对象。代码如下:

```java
protected static void init(AbstractWriteTask task, AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    task.ctx = ctx;
    task.msg = msg;
    task.promise = promise;
    // 计算 AbstractWriteTask 对象大小 <1>
    if (ESTIMATE_TASK_SIZE_ON_SUBMIT) {
        task.size = ctx.pipeline.estimatorHandle().size(msg) + WRITE_TASK_OVERHEAD;
        // 增加 ChannelOutboundBuffer 的 totalPendingSize 属性  <2>
        ctx.pipeline.incrementPendingOutboundBytes(task.size);
    } else {
        task.size = 0;
    }
}
```

+ 在下文中，我们会看到 AbstractWriteTask 对象是从 Recycler 中获取，所以获取完成后，需要通过该方法，初始化该对象的属性。

+ `<1>` 处，计算 AbstractWriteTask 对象大小。并且在 `<2>` 处，调用 `ChannelPipeline#incrementPendingOutboundBytes(long size)` 方法，增加 ChannelOutboundBuffer 的 `totalPendingSize` 属性。代码如下：

  ```java
  // DefaultChannelPipeline.java
  @UnstableApi
  protected void incrementPendingOutboundBytes(long size) {
      ChannelOutboundBuffer buffer = channel.unsafe().outboundBuffer();
      if (buffer != null) {
          buffer.incrementPendingOutboundBytes(size);
      }
  }
  ```

  + 内部，会调用 `ChannelOutboundBuffer#incrementPendingOutboundBytes(long size)` 方法，增加 ChannelOutboundBuffer 的 `totalPendingSize` 属性。

### 7.3 run

`#run()` **实现**方法:

```java
 1: @Override
 2: public final void run() {
 3:     try {
 4:         // 减少 ChannelOutboundBuffer 的 totalPendingSize 属性 <1>
 5:         // Check for null as it may be set to null if the channel is closed already
 6:         if (ESTIMATE_TASK_SIZE_ON_SUBMIT) {
 7:             ctx.pipeline.decrementPendingOutboundBytes(size);
 8:         }
 9:         // 执行 write 事件到下一个节点
10:         write(ctx, msg, promise);
11:     } finally {
12:         // 置空，help gc
13:         // Set to null so the GC can collect them directly
14:         ctx = null;
15:         msg = null;
16:         promise = null;
17:         // 回收对象
18:         handle.recycle(this);
19:     }
20: }
```

+ 在 `<1>` 处， 调用 `ChannelPipeline#decrementPendingOutboundBytes(long size)` 方法，减少 ChannelOutboundBuffer 的 `totalPendingSize` 属性。代码如下：

  ```java
  @UnstableApi
  protected void decrementPendingOutboundBytes(long size) {
      ChannelOutboundBuffer buffer = channel.unsafe().outboundBuffer();
      if (buffer != null) {
          buffer.decrementPendingOutboundBytes(size);
      }
  }
  ```

  + 内部，会调用 `ChannelOutboundBuffer#decrementPendingOutboundBytes(long size)` 方法，减少 ChannelOutboundBuffer 的 `totalPendingSize` 属性。

+ 第 10 行：调用 `#write(ctx, msg, promise)` 方法，执行 write 事件到下一个节点。代码如下：

  ```java
  protected void write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
      ctx.invokeWrite(msg, promise);
  }
  ```

+ 第 11 至 19 行：置空 AbstractWriteTask 对象，并调用 `Recycler.Handle#recycle(this)` 方法，回收该对象。

### 7.4 WriteTask

WriteTask ，实现 SingleThreadEventLoop.NonWakeupRunnable 接口，继承 AbstractWriteTask 抽象类，write 任务实现类。

**为什么会实现 SingleThreadEventLoop.NonWakeupRunnable 接口呢**？write 操作，仅仅将数据写到**内存队列**中，无需唤醒 EventLoop ，从而提升性能。

### 7.4.1 newInstance

`#newInstance(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，创建 WriteTask 对象。代码如下：

```java
private static final Recycler<WriteTask> RECYCLER = new Recycler<WriteTask>() {

    @Override
    protected WriteTask newObject(Handle<WriteTask> handle) {
        return new WriteTask(handle); // 创建 WriteTask 对象
    }

};

private static WriteTask newInstance(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    // 从 Recycler 的对象池中获得 WriteTask 对象
    WriteTask task = RECYCLER.get();
    // 初始化 WriteTask 对象的属性
    init(task, ctx, msg, promise);
    return task;
}
```

### 7.4.2 构造方法

```java
private WriteTask(Recycler.Handle<WriteTask> handle) {
    super(handle);
}
```

### 7.4.3 write

WriteTask 无需实现 `#write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)`方法，直接**重用**父类该方法即可。

### 7.5 WriteAndFlushTask

WriteAndFlushTask ，继承 WriteAndFlushTask 抽象类，write + flush 任务实现类。

#### 7.5.1 newInstance

`#newInstance(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，创建 WriteAndFlushTask 对象。代码如下：

```java
private static final Recycler<WriteAndFlushTask> RECYCLER = new Recycler<WriteAndFlushTask>() {

    @Override
    protected WriteAndFlushTask newObject(Handle<WriteAndFlushTask> handle) {
        return new WriteAndFlushTask(handle); // 创建 WriteAndFlushTask 对象
    }

};

private static WriteAndFlushTask newInstance(AbstractChannelHandlerContext ctx, Object msg,  ChannelPromise promise) {
    // 从 Recycler 的对象池中获得 WriteTask 对象
    WriteAndFlushTask task = RECYCLER.get();
    // 初始化 WriteTask 对象的属性
    init(task, ctx, msg, promise);
    return task;
}
```

#### 7.5.2 构造方法

```java
private WriteAndFlushTask(Recycler.Handle<WriteAndFlushTask> handle) {
    super(handle);
}
```

#### 7.5.3 write

`#write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，在父类的该方法的基础上，增加执行 **flush** 事件到下一个节点。代码如下：

```java
@Override
public void write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    // 执行 write 事件到下一个节点
    super.write(ctx, msg, promise);
    // 执行 flush 事件到下一个节点
    ctx.invokeFlush();
}
```

