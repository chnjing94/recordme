### **1. 概述**

本文我们分享 EventLoop 的**处理 IO 事件**相关代码的实现。对应如下图的绿条 **process selected keys** 部分：

![](handleIOEvent_files/image-20190213000638382.png)

###**2.SelectorTuple**

SelectorTuple ，Selector 元组。代码如下：

> SelectorTuple 内嵌在 NioEventLoop

```java
private static final class SelectorTuple {

    /**
     * 未包装的 Selector 对象
     */
    final Selector unwrappedSelector;
    /**
     * 未包装的 Selector 对象
     */
    final Selector selector;

    SelectorTuple(Selector unwrappedSelector) {
        this.unwrappedSelector = unwrappedSelector;
        this.selector = unwrappedSelector;
    }

    SelectorTuple(Selector unwrappedSelector, Selector selector) {
        this.unwrappedSelector = unwrappedSelector;
        this.selector = selector;
    }

}
```

### 3. openSelector

`#openSelector()` 方法，创建 Selector 对象。代码如下：

```java
 1: private SelectorTuple openSelector() {
 2:     // 创建 Selector 对象，作为 unwrappedSelector
 3:     final Selector unwrappedSelector;
 4:     try {
 5:         unwrappedSelector = provider.openSelector();
 6:     } catch (IOException e) {
 7:         throw new ChannelException("failed to open a new selector", e);
 8:     }
 9: 
10:     // 禁用 SelectionKey 的优化，则直接返回 SelectorTuple 对象。即，selector 也使用 unwrappedSelector 。
11:     if (DISABLE_KEYSET_OPTIMIZATION) {
12:         return new SelectorTuple(unwrappedSelector);
13:     }
14: 
15:     // 获得 SelectorImpl 类
16:     Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
17:         @Override
18:         public Object run() {
19:             try {
20:                 return Class.forName(
21:                         "sun.nio.ch.SelectorImpl",
22:                         false,
23:                         PlatformDependent.getSystemClassLoader()); // 成功，则返回该类
24:             } catch (Throwable cause) {
25:                 return cause; // 失败，则返回该异常
26:             }
27:         }
28:     });
29: 
30:     // 获得 SelectorImpl 类失败，则直接返回 SelectorTuple 对象。即，selector 也使用 unwrappedSelector 。
31:     if (!(maybeSelectorImplClass instanceof Class) ||
32:             // ensure the current selector implementation is what we can instrument.
33:             !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
34:         if (maybeSelectorImplClass instanceof Throwable) {
35:             Throwable t = (Throwable) maybeSelectorImplClass;
36:             logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
37:         }
38:         return new SelectorTuple(unwrappedSelector);
39:     }
40: 
41:     final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
42: 
43:     // 创建 SelectedSelectionKeySet 对象
44:     final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();
45: 
46:     // 设置 SelectedSelectionKeySet 对象到 unwrappedSelector 中
47:     Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
48:         @Override
49:         public Object run() {
50:             try {
51:                 // 获得 "selectedKeys" "publicSelectedKeys" 的 Field
52:                 Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
53:                 Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");
54: 
55:                 // 设置 Field 可访问
56:                 Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
57:                 if (cause != null) {
58:                     return cause;
59:                 }
60:                 cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
61:                 if (cause != null) {
62:                     return cause;
63:                 }
64: 
65:                 // 设置 SelectedSelectionKeySet 对象到 unwrappedSelector 的 Field 中
66:                 selectedKeysField.set(unwrappedSelector, selectedKeySet);
67:                 publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
68:                 return null;
69:             } catch (NoSuchFieldException e) {
70:                 return e; // 失败，则返回该异常
71:             } catch (IllegalAccessException e) {
72:                 return e; // 失败，则返回该异常
73:             }
74:         }
75:     });
76: 
77:     // 设置 SelectedSelectionKeySet 对象到 unwrappedSelector 中失败，则直接返回 SelectorTuple 对象。即，selector 也使用 unwrappedSelector 。
78:     if (maybeException instanceof Exception) {
79:         selectedKeys = null;
80:         Exception e = (Exception) maybeException;
81:         logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
82:         return new SelectorTuple(unwrappedSelector);
83:     }
84: 
85:     // 设置 SelectedSelectionKeySet 对象到 selectedKeys 中
86:     selectedKeys = selectedKeySet;
87:     logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
88: 
89:     // 创建 SelectedSelectionKeySetSelector 对象
90:     // 创建 SelectorTuple 对象。即，selector 也使用 SelectedSelectionKeySetSelector 对象。
91:     return new SelectorTuple(unwrappedSelector, new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
92: }
```

- 第 2 至 8 行：创建 Selector 对象，作为 `unwrappedSelector` 。

- 第 10 至 13 行：禁用 SelectionKey 的优化，则直接返回 SelectorTuple 对象。即，`selector` 也使用 `unwrappedSelector` 。

- 第 15 至 28 行：获得 SelectorImpl 类。在方法内部，调用 `Class#forName(String name, boolean initialize, ClassLoader loader)` 方法，加载 `sun.nio.ch.SelectorImpl` 类。加载成功，则返回该类，否则返回异常。

  - 第 30 至 39 行： 获得 SelectorImpl 类失败，则直接返回 SelectorTuple 对象。即，`selector` 也使用 `unwrappedSelector` 。

- 第 44 行：创建 SelectedSelectionKeySet 对象。这是 Netty 对 Selector 的 `selectionKeys` 的优化。关于 SelectedSelectionKeySet 的详细实现，见 「4. SelectedSelectionKeySet」

  - 第 46 至 75 行： 设置 SelectedSelectionKeySet 对象到 `unwrappedSelector` 中的 `selectedKeys` 和 `publicSelectedKeys` 属性。
  - `selectedKeys` 和 `publicSelectedKeys` 属性在 SelectorImpl 类中，代码如下：

  ```java
  protected HashSet<SelectionKey> keys = new HashSet(); // => publicKeys
  private Set<SelectionKey> publicKeys;
  
  protected Set<SelectionKey> selectedKeys = new HashSet(); // => publicSelectedKeys
  private Set<SelectionKey> publicSelectedKeys;
  
  protected SelectorImpl(SelectorProvider var1) {
      super(var1);
      if (Util.atBugLevel("1.4")) { // 可以无视
          this.publicKeys = this.keys;
          this.publicSelectedKeys = this.selectedKeys;
      } else {
          this.publicKeys = Collections.unmodifiableSet(this.keys);
          this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
      }
  
  }
  ```

  + 可以看到，`selectedKeys` 和 `publicSelectedKeys` 的类型都是 HashSet 。

- 第 77 至 83 行：设置 SelectedSelectionKeySet 对象到 `unwrappedSelector` 中失败，则直接返回 SelectorTuple 对象。即，`selector` 也使用 `unwrappedSelector` 。

- 第 91 行：创建 SelectedSelectionKeySetSelector 对象。这是 Netty 对 Selector 的优化实现类。关于 SelectedSelectionKeySetSelector 的详细实现，见 「5. SelectedSelectionKeySetSelector」

### 4. SelectedSelectionKeySet

`io.netty.channel.nio.SelectedSelectionKeySet` ，继承 AbstractSet 抽象类，已 **select** 的 NIO SelectionKey 集合。代码如下：

```java
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    /**
     * SelectionKey 数组
     */
    SelectionKey[] keys;
    /**
     * 数组可读大小
     */
    int size;

    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024]; // 默认 1024 大小
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        // 添加到数组
        keys[size++] = o;

        // 超过数组大小上限，进行扩容
        if (size == keys.length) {
            increaseCapacity();
        }

        return true;
    }

    @Override
    public int size() {
        return size;
    }

    @Override
    public boolean remove(Object o) {
        return false;
    }

    @Override
    public boolean contains(Object o) {
        return false;
    }

    @Override
    public Iterator<SelectionKey> iterator() {
        throw new UnsupportedOperationException();
    }

    void reset() {
        reset(0);
    }

    void reset(int start) {
        // 重置数组内容为空
        Arrays.fill(keys, start, size, null);
        // 重置可读大小为 0
        size = 0;
    }

    private void increaseCapacity() {
        // 两倍扩容
        SelectionKey[] newKeys = new SelectionKey[keys.length << 1];
        // 复制老数组到新数组
        System.arraycopy(keys, 0, newKeys, 0, size);
        // 赋值给老数组
        keys = newKeys;
    }

}
```

- 通过 `keys` 和 `size` 两个属性，实现**可重用**的数组。
- `#add(SelectionKey o)` 方法，添加新 **select** 到就绪事件的 SelectionKey 到 `keys` 中。当超过数组大小上限时，调用 `#increaseCapacity()` 方法，进行**两倍**扩容。
- `#reset(...)` 方法，每次读取使用完数据，调用该方法，进行重置。
- 因为 `#remove(Object o)`、`#contains(Object o)`、`#iterator()` 不会使用到，索性不进行实现。

### 5. SelectedSelectionKeySetSelector

`io.netty.channel.nio.SelectedSelectionKeySetSelector` ，基于 Netty SelectedSelectionKeySet 作为 `selectionKeys` 的 Selector 实现类。代码如下：

```java
final class SelectedSelectionKeySetSelector extends Selector {

    /**
     * SelectedSelectionKeySet 对象
     */
    private final SelectedSelectionKeySet selectionKeys;
    /**
     * 原始 Java NIO Selector 对象
     */
    private final Selector delegate;

    SelectedSelectionKeySetSelector(Selector delegate, SelectedSelectionKeySet selectionKeys) {
        this.delegate = delegate;
        this.selectionKeys = selectionKeys;
    }

    @Override
    public boolean isOpen() {
        return delegate.isOpen();
    }

    @Override
    public SelectorProvider provider() {
        return delegate.provider();
    }

    @Override
    public Set<SelectionKey> keys() {
        return delegate.keys();
    }

    @Override
    public Set<SelectionKey> selectedKeys() {
        return delegate.selectedKeys();
    }

    @Override
    public int selectNow() throws IOException {
        // 重置 selectionKeys
        selectionKeys.reset();
        // selectNow
        return delegate.selectNow();
    }

    @Override
    public int select(long timeout) throws IOException {
        // 重置 selectionKeys
        selectionKeys.reset();
        // select
        return delegate.select(timeout);
    }

    @Override
    public int select() throws IOException {
        // 重置 selectionKeys
        selectionKeys.reset();
        // select
        return delegate.select();
    }

    @Override
    public Selector wakeup() {
        return delegate.wakeup();
    }

    @Override
    public void close() throws IOException {
        delegate.close();
    }

}
```

### 6. rebuildSelector

`#rebuildSelector()` 方法，重建 Selector 对象。代码如下：

> 该方法用于 NIO Selector 发生 epoll bug 时，重建 Selector 对象。

```java
public void rebuildSelector() {
    // 只允许在 EventLoop 的线程中执行
    if (!inEventLoop()) {
        // 调用execute()，让EventLoop的线程去执行
        execute(new Runnable() {
            @Override
            public void run() {
                rebuildSelector0();
            }
        });
        return;
    }
    rebuildSelector0();
}
```

- 只允许在 EventLoop 的线程中，调用 `#rebuildSelector0()` 方法，重建 Selector 对象。

### 6.1 rebuildSelector0

```java
 1: private void rebuildSelector0() {
 2:     final Selector oldSelector = selector;
 3:     if (oldSelector == null) {
 4:         return;
 5:     }
 6: 
 7:     // 创建新的 Selector 对象
 8:     final SelectorTuple newSelectorTuple;
 9:     try {
10:         newSelectorTuple = openSelector();
11:     } catch (Exception e) {
12:         logger.warn("Failed to create a new Selector.", e);
13:         return;
14:     }
15: 
16:     // Register all channels to the new Selector.
17:     // 将注册在 NioEventLoop 上的所有 Channel ，注册到新创建 Selector 对象上
18:     int nChannels = 0; // 计算重新注册成功的 Channel 数量
19:     for (SelectionKey key: oldSelector.keys()) {
20:         Object a = key.attachment();
21:         try {
22:             if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
23:                 continue;
24:             }
25: 
26:             int interestOps = key.interestOps();
27:             // 取消老的 SelectionKey
28:             key.cancel();
29:             // 将 Channel 注册到新的 Selector 对象上
30:             SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
31:             // 修改 Channel 的 selectionKey 指向新的 SelectionKey 上
32:             if (a instanceof AbstractNioChannel) {
33:                 // Update SelectionKey
34:                 ((AbstractNioChannel) a).selectionKey = newKey;
35:             }
36: 
37:             // 计数 ++
38:             nChannels ++;
39:         } catch (Exception e) {
40:             logger.warn("Failed to re-register a Channel to the new Selector.", e);
41:             // 关闭发生异常的 Channel
42:             if (a instanceof AbstractNioChannel) {
43:                 AbstractNioChannel ch = (AbstractNioChannel) a;
44:                 ch.unsafe().close(ch.unsafe().voidPromise());
45:             // 调用 NioTask 的取消注册事件
46:             } else {
47:                 @SuppressWarnings("unchecked")
48:                 NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
49:                 invokeChannelUnregistered(task, key, e);
50:             }
51:         }
52:     }
53: 
54:     // 修改 selector 和 unwrappedSelector 指向新的 Selector 对象
55:     selector = newSelectorTuple.selector;
56:     unwrappedSelector = newSelectorTuple.unwrappedSelector;
57: 
58:     // 关闭老的 Selector 对象
59:     try {
60:         // time to close the old selector as everything else is registered to the new one
61:         oldSelector.close();
62:     } catch (Throwable t) {
63:         if (logger.isWarnEnabled()) {
64:             logger.warn("Failed to close the old Selector.", t);
65:         }
66:     }
67: 
68:     if (logger.isInfoEnabled()) {
69:         logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
70:     }
71: }
```

- 第 7 行：调用 `#openSelector()` 方法，创建新的 Selector 对象。
- 第 16 至 52 行：遍历**老**的 Selector 对象的 `selectionKeys` ，将注册在 NioEventLoop 上的所有 Channel ，注册到**新**创建 Selector 对象上。
  - 第 22 至 24 行：校验 SelectionKey 有效，并且 Java NIO Channel 并未注册在**新**的 Selector 对象上。
  - 第 28 行：调用 `SelectionKey#cancel()` 方法，取消**老**的 SelectionKey 。
  - 第 30 行：将 Java NIO Channel 注册到**新**的 Selector 对象上，返回**新**的 SelectionKey 对象。
  - 第 39 至 51 行：当发生异常时候，根据不同的 SelectionKey 的 `attachment` 来判断处理方式：
    - 第 41 至 44 行：当 `attachment` 是 Netty NIO Channel 时，调用 `Unsafe#close(ChannelPromise promise)` 方法，**关闭**发生异常的 Channel 。
    - 第 45 至 50 行：当 `attachment` 是 Netty NioTask 时，调用 `#invokeChannelUnregistered(NioTask<SelectableChannel> task, SelectionKey k, Throwable cause)` 方法，通知 Channel 取消注册。详细解析，见 「8. NioTask」
- 第 54 至 56 行：修改 `selector` 和 `unwrappedSelector` 指向**新**的 Selector 对象。
- 第 58 至 66 行：调用 `Selector#close()` 方法，关闭**老**的 Selector 对象。

总的来说，`#rebuildSelector()` 方法，相比 `#openSelector()` 方法，主要是需要将老的 Selector 对象的“数据”复制到新的 Selector 对象上，并关闭老的 Selector 对象。

### 7. processSelectedKeys

在 `#run()` 方法中，会调用 `#processSelectedKeys()` 方法，处理 Channel **新增**就绪的 IO 事件。代码如下：`

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

- 当 `selectedKeys` 非空，意味着使用优化的 SelectedSelectionKeySetSelector ，所以调用 `#processSelectedKeysOptimized()` 方法；否则，调用 `#processSelectedKeysPlain()` 方法。

### 7.1 processSelectedKeysOptimized

`#processSelectedKeysOptimized()` 方法，从方法名，我们也可以看出，这是个经过**优化**的实现。基于 Netty SelectedSelectionKeySetSelector ，处理 Channel **新增**就绪的 IO 事件。代码如下：

```java
 1: private void processSelectedKeysOptimized() {
 2:     // 遍历数组
 3:     for (int i = 0; i < selectedKeys.size; ++i) {
 4:         final SelectionKey k = selectedKeys.keys[i];
 5:         // null out entry in the array to allow to have it GC'ed once the Channel close
 6:         // See https://github.com/netty/netty/issues/2363
 7:         selectedKeys.keys[i] = null;
 8: 
 9:         final Object a = k.attachment();
10: 
11:         // 处理一个 Channel 就绪的 IO 事件
12:         if (a instanceof AbstractNioChannel) {
13:             processSelectedKey(k, (AbstractNioChannel) a);
14:         // 使用 NioTask 处理一个 Channel 就绪的 IO 事件
15:         } else {
16:             @SuppressWarnings("unchecked")
17:             NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
18:             processSelectedKey(k, task);
19:         }
20: 
21:         // TODO 1007 NioEventLoop cancel 方法
22:         if (needsToSelectAgain) {
23:             // null out entries in the array to allow to have it GC'ed once the Channel close
24:             // See https://github.com/netty/netty/issues/2363
25:             selectedKeys.reset(i + 1);
26: 
27:             selectAgain();
28:             i = -1;
29:         }
30:     }
31: }
```

+ 第 3 行：循环 `selectedKeys` 数组。
  + 第 4 至 7 行：置空，让GC回收对象
  + 第 11 至 13 行：当 `attachment` 是 Netty NIO Channel 时，调用 `#processSelectedKey(SelectionKey k, AbstractNioChannel ch)` 方法，处理一个 Channel 就绪的 IO 事件。
  + 第 14 至 19 行：当 `attachment` 是 Netty NioTask 时，调用 `#processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task)` 方法，使用 NioTask 处理一个 Channel 的 IO 事件。详细解析，见 「8. NioTask」。

### 7.2 processSelectedKeysPlain

`#processSelectedKeysOptimized()` 方法，基于 Java NIO 原生 Selecotr ，处理 Channel **新增**就绪的 IO 事件。代码如下：

> 总体和 `#processSelectedKeysOptimized()` 方法**类似**。

```java
 1: private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
 2:     // check if the set is empty and if so just return to not create garbage by
 3:     // creating a new Iterator every time even if there is nothing to process.
 4:     // See https://github.com/netty/netty/issues/597
 5:     if (selectedKeys.isEmpty()) {
 6:         return;
 7:     }
 8: 
 9:     // 遍历 SelectionKey 迭代器
10:     Iterator<SelectionKey> i = selectedKeys.iterator();
11:     for (;;) {
12:         // 获得 SelectionKey 对象
13:         final SelectionKey k = i.next();
14:         // 从迭代器中移除
15:         i.remove();
16: 
17:         final Object a = k.attachment();
18:         // 处理一个 Channel 就绪的 IO 事件
19:         if (a instanceof AbstractNioChannel) {
20:             processSelectedKey(k, (AbstractNioChannel) a);
21:         // 使用 NioTask 处理一个 Channel 就绪的 IO 事件
22:         } else {
23:             @SuppressWarnings("unchecked")
24:             NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
25:             processSelectedKey(k, task);
26:         }
27: 
28:         // 无下一个节点，结束
29:         if (!i.hasNext()) {
30:             break;
31:         }
32: 
33:         // TODO 1007 NioEventLoop cancel 方法
34:         if (needsToSelectAgain) {
35:             selectAgain();
36:             selectedKeys = selector.selectedKeys();
37: 
38:             // Create the iterator again to avoid ConcurrentModificationException
39:             if (selectedKeys.isEmpty()) {
40:                 break;
41:             } else {
42:                 i = selectedKeys.iterator();
43:             }
44:         }
45:     }
46: }
```

+ 第 10 至 11 行：遍历 SelectionKey **迭代器**。
  + 第 12 至 15 行：获得下一个 SelectionKey 对象，并从迭代器中移除。
  + 第 18 至 20 行：当 `attachment` 是 Netty NIO Channel 时，调用 `#processSelectedKey(SelectionKey k, AbstractNioChannel ch)` 方法，处理一个 Channel 就绪的 IO 事件。详细解析，见 「7.3 processSelectedKey」 。
  + 第 21 至 26 行：当 `attachment` 是 Netty NioTask 时，调用 `#processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task)` 方法，使用 NioTask 处理一个 Channel 的 IO 事件。详细解析，见 「8. NioTask」。

### 7.3 processSelectedKey

`#processSelectedKey(SelectionKey k, AbstractNioChannel ch)` 方法，处理一个 Channel 就绪的 IO 事件。代码如下：

```java
 1: private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
 2:     // 如果 SelectionKey 是不合法的，则关闭 Channel
 3:     final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
 4:     if (!k.isValid()) {
 5:         final EventLoop eventLoop;
 6:         try {
 7:             eventLoop = ch.eventLoop();
 8:         } catch (Throwable ignored) {
 9:             // If the channel implementation throws an exception because there is no event loop, we ignore this
10:             // because we are only trying to determine if ch is registered to this event loop and thus has authority
11:             // to close ch.
12:             return;
13:         }
14:         // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
15:         // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
16:         // still healthy and should not be closed.
17:         // See https://github.com/netty/netty/issues/5125
18:         if (eventLoop != this) {
19:             return;
20:         }
21:         // close the channel if the key is not valid anymore
22:         unsafe.close(unsafe.voidPromise());
23:         return;
24:     }
25: 
26:     try {
27:         // 获得就绪的 IO 事件的 ops
28:         int readyOps = k.readyOps();
29: 
30:         // OP_CONNECT 事件就绪
31:         // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
32:         // the NIO JDK channel implementation may throw a NotYetConnectedException.
33:         if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
34:             // 移除对 OP_CONNECT 感兴趣
35:             // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
36:             // See https://github.com/netty/netty/issues/924
37:             int ops = k.interestOps();
38:             ops &= ~SelectionKey.OP_CONNECT;
39:             k.interestOps(ops);
40:             // 完成连接
41:             unsafe.finishConnect();
42:         }
43: 
44:         // OP_WRITE 事件就绪
45:         // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
46:         if ((readyOps & SelectionKey.OP_WRITE) != 0) {
47:             // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
48:             // 向 Channel 写入数据
49:             ch.unsafe().forceFlush();
50:         }
51: 
52:         // SelectionKey.OP_READ 或 SelectionKey.OP_ACCEPT 就绪
53:         // readyOps == 0 是对 JDK Bug 的处理，防止空的死循环
54:         // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
55:         // to a spin loop
56:         if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
57:             unsafe.read();
58:         }
59:     } catch (CancelledKeyException ignored) {
60:         // 发生异常，关闭 Channel
61:         unsafe.close(unsafe.voidPromise());
62:     }
63: }
```

- 第 2 至 24 行：如果 SelectionKey 是不合法的，则关闭 Channel 。
- 第 30 至 42 行：如果对 `OP_CONNECT` 事件就绪：
  - 第 34 至 39 行：移除对 `OP_CONNECT` 的感兴趣，即不再监听连接事件。
  - 【重要】第 41 行：调用 `Unsafe#finishConnect()` 方法，完成连接。
- 第 44 至 50 行：如果对 `OP_WRITE` 事件就绪，调用 `Unsafe#forceFlush()` 方法，向 Channel 写入数据。在完成写入数据后，会移除对 `OP_WRITE` 的感兴趣。
- 第 52 至 58 行：如果对 `OP_READ` 或 `OP_ACCEPT` 事件就绪：调用 `Unsafe#read()` 方法，处理读**或者**者接受客户端连接的事件。

### **8. NioTask**

`io.netty.channel.nio.NioTask` ，用于自定义 Nio 事件处理**接口**。对于每个 Nio 事件，可以认为是一个任务( Task )，代码如下：

```java
public interface NioTask<C extends SelectableChannel> {

    /**
     * Invoked when the {@link SelectableChannel} has been selected by the {@link Selector}.
     */
    void channelReady(C ch, SelectionKey key) throws Exception;

    /**
     * Invoked when the {@link SelectionKey} of the specified {@link SelectableChannel} has been cancelled and thus
     * this {@link NioTask} will not be notified anymore.
     *
     * @param cause the cause of the unregistration. {@code null} if a user called {@link SelectionKey#cancel()} or
     *              the event loop has been shut down.
     */
    void channelUnregistered(C ch, Throwable cause) throws Exception;

}
```

- `#channelReady(C ch, SelectionKey key)` 方法，处理 Channel IO 就绪的事件。相当于说，我们可以通过实现该接口方法，实现 「7.3 processSelectedKey」 的逻辑。
- `#channelUnregistered(C ch, Throwable cause)` 方法，Channel 取消注册。一般来说，我们可以通过实现该接口方法，关闭 Channel 。

### 8.1 register

`#register(final SelectableChannel ch, final int interestOps, final NioTask<?> task)` 方法，注册 Java NIO Channel ( 不一定需要通过 Netty 创建的 Channel )到 Selector 上，相当于说，也注册到了 EventLoop 上。代码如下：

```java
/**
 * Registers an arbitrary {@link SelectableChannel}, not necessarily created by Netty, to the {@link Selector}
 * of this event loop.  Once the specified {@link SelectableChannel} is registered, the specified {@code task} will
 * be executed by this event loop when the {@link SelectableChannel} is ready.
 */
public void register(final SelectableChannel ch, final int interestOps, final NioTask<?> task) {
    if (ch == null) {
        throw new NullPointerException("ch");
    }
    if (interestOps == 0) {
        throw new IllegalArgumentException("interestOps must be non-zero.");
    }
    if ((interestOps & ~ch.validOps()) != 0) {
        throw new IllegalArgumentException(
                "invalid interestOps: " + interestOps + "(validOps: " + ch.validOps() + ')');
    }
    if (task == null) {
        throw new NullPointerException("task");
    }

    if (isShutdown()) {
        throw new IllegalStateException("event loop shut down");
    }

    // <1>
    try {
        ch.register(selector, interestOps, task);
    } catch (Exception e) {
        throw new EventLoopException("failed to register a channel", e);
    }
}
```

- `<1>` 处，调用 `SelectableChannel#register(Selector sel, int ops, Object att)` 方法，注册 Java NIO Channel 到 Selector 上。这里我们可以看到，`attachment` 为 NioTask 对象，而不是 Netty Channel 对象。

### 8.2 invokeChannelUnregistered

`#invokeChannelUnregistered(NioTask<SelectableChannel> task, SelectionKey k, Throwable cause)` 方法，执行 Channel 取消注册。代码如下：

```java
private static void invokeChannelUnregistered(NioTask<SelectableChannel> task, SelectionKey k, Throwable cause) {
    try {
        task.channelUnregistered(k.channel(), cause);
    } catch (Exception e) {
        logger.warn("Unexpected exception while running NioTask.channelUnregistered()", e);
    }
}
```

- 在方法内部，调用 `NioTask#channelUnregistered()` 方法，执行 Channel 取消注册。

### 8.3 processSelectedKey

`#processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task)` 方法，使用 NioTask ，自定义实现 Channel 处理 Channel IO 就绪的事件。代码如下：

```java
private static void processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task) {
    int state = 0; // 未执行
    try {
        // 调用 NioTask 的 Channel 就绪事件
        task.channelReady(k.channel(), k);
        state = 1; // 执行成功
    } catch (Exception e) {
        // SelectionKey 取消
        k.cancel();
        // 执行 Channel 取消注册
        invokeChannelUnregistered(task, k, e);
        state = 2; // 执行异常
    } finally {
        switch (state) {
        case 0:
            // SelectionKey 取消
            k.cancel();
            // 执行 Channel 取消注册
            invokeChannelUnregistered(task, k, null);
            break;
        case 1:
            // SelectionKey 不合法，则执行 Channel 取消注册
            if (!k.isValid()) { // Cancelled by channelReady()
                invokeChannelUnregistered(task, k, null);
            }
            break;
        }
    }
}
```

