

# JAVA并发编程

## 1. JAVA内存模型

 [Java并发编程：volatile关键字解析](https://www.cnblogs.com/dolphin0520/p/3920373.html)

![](./images/concurrency01.png)

- Java内存模型概念
- 缓存一致性协议：最出名的就是Intel 的MESI协议，MESI协议保证了每个缓存中使用的共享变量的副本是一致的。它核心的思想是：当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。

## 2. 线程安全性

### 2.1 原子性

- Atomic包
    - **LongAdder** [《Java并发计数器探秘》](https://www.cnkirito.moe/java-concurrent-counter/) 

    - **AtomicIntegerFieldUpdater** ：修改某一个类的对象的某一个字段，要求是非static, volatile 的字段。

    - **AtomicXXX**的CAS如何保证原子性？[《浅谈CAS(Compare and Swap) 原理》](https://www.cnblogs.com/Leo_wl/p/6899716.html)

    - **AtomicStampedReference**：解决ABA问题

- 锁
    - Synchronized：依赖JVM
    - Lock: JDK提供的接口，依赖特殊的CPU指令，比如可重入锁
    
### 2.2 可见性

- Volatile  [Java并发编程：volatile关键字解析](https://www.cnblogs.com/dolphin0520/p/3920373.html)
- 用作状态标识量，双重检测（例如单例模式）
- 内存屏障功能

### 2.3 有序性

- 8条happens-before原则： [【死磕Java并发】-----Java内存模型之happens-before](https://www.cnblogs.com/chenssy/p/6393321.html)

## 3. 安全发布对象

- 发布对象：使一个对象能够被当前范围之外的代码使用。

  - 用枚举类实现单例模式

    ```java
    public class SingletonExample7 {
    
        // 私有构造函数
        private SingletonExample7() {
    
        }
    
        public static SingletonExample7 getInstance() {
            return Singleton.INSTANCE.getInstance();
        }
    
        private enum Singleton {
            INSTANCE;
    
            private SingletonExample7 singleton;
    
            // JVM保证这个方法绝对只调用一次
            Singleton() {
                singleton = new SingletonExample7();
            }
    
            public SingletonExample7 getInstance() {
                return singleton;
            }
        }
    }
    ```

    - 原理[为什么我墙裂建议大家使用枚举来实现单例。](https://blog.csdn.net/moakun/article/details/80688851)
    - 是一种懒汉模式
    - 对于枚举类实现的单例来说，我有可能只调用枚举类的外部类的其他方法，而不去获取实例，这样内部枚举类不会被初始化。对于饿汉模式来说，不管你或不获取实例，实例都会在初始化阶段被创建。
    - 防止序列化对单例的破坏，双重检测的懒汉单例，可能会被序列化破坏单例。

- 对象逸出：一种错误发布，当一个对象还没有构造完成，就被别的线程看见。

## 4. 线程安全策略

### 4.1 不可变对象

- final 关键字，类，方法，变量
- Collections.unmodiiableXXX: Collection, List, Set, Map
- Guava: ImmutableXXX: Collection, List, Set, Map

### 4.2 线程封闭

- TheadLocal [Java并发编程：深入剖析ThreadLocal](https://www.cnblogs.com/dolphin0520/p/3920407.html)

### 4.3 线程安全/不安全类

- StringBuilder -> StringBuffer
- SimpleDateFormat -> JodaTime
- ArrayList, HashSet, HashMap等Collections

### 4.4 同步容器

- HashMap -> HashTable
- ArrayList -> Vector, Stack
- Collections.synchronizedXXX()

### 4.5 并发容器 J.U.C

- ArrayList -> CopyOnWriteArrayList

  add()操作如下：

  ```java
  public boolean add(E e) {
      final ReentrantLock lock = this.lock;
      lock.lock();
      try {
          Object[] elements = getArray();
          int len = elements.length;
          Object[] newElements = Arrays.copyOf(elements, len + 1);
          newElements[len] = e;
          setArray(newElements);
          return true;
      } finally {
          lock.unlock();
      }
  }
  ```

- HashSet, TreeSet -> CopyOnWriteArraySet, ConcurrentSkipListSet
- HashMap, TreeMap -> ConcurrentHashMap, ConcurrentSkipListMap

J.U.C整体如下图

![](./images/02.png)

## 5. AQS

> [【死磕Java并发】—–J.U.C之AQS：AQS简介](http://cmsblogs.com/?p=2174)

### 5.1 CountDownLatch

继承了AQS, CountDown()时，使用CAS对计数器减一，await()时，循环等待计数器为0，但是使用了LockSupport.park(this)来阻塞线程，并非一直执行for循环。

 [CountDownLatch实现原理](<https://blog.csdn.net/u014653197/article/details/78217571>)

 [LockSupport的park和unpark](https://www.cnblogs.com/moonandstar08/p/5132012.html)

### 5.2 Semaphore

 [【JUC】JDK1.8源码分析之Semaphore（六）](https://www.cnblogs.com/leesf456/p/5414778.html)

### 5.3 CyclicBarrier

 [[死磕Java并发]-----J.U.C之并发工具类：CyclicBarrier](<https://blog.csdn.net/chenssy/article/details/70160595>)

### 5.4 ReentrantLock与锁

 [【死磕Java并发】—–J.U.C之重入锁：ReentrantLock](http://cmsblogs.com/?p=2210)

 [【死磕Java并发】—–J.U.C之读写锁：ReentrantReadWriteLock](http://cmsblogs.com/?p=2213)

[JDK8新增锁StampedLock详解](<http://blog.sina.com.cn/s/blog_6f5e71b30102xfsb.html>)

 [【死磕Java并发】—–J.U.C之Condition](http://cmsblogs.com/?p=2222)

## 6. FutureTask

### 6.1 示例

```java
    public static void main(String[] args) throws Exception {
        FutureTask<String> futureTask = new FutureTask<String>(new Callable<String>() {
            @Override
            public String call() throws Exception {
                log.info("do something in callable");
                Thread.sleep(5000);
                return "Done";
            }
        });

        new Thread(futureTask).start();
        log.info("do something in main");
        Thread.sleep(1000);
        String result = futureTask.get();
        log.info("result：{}", result);
    }
```

### 6.2 Fork/Join

其场景为：如果一个应用程序能够被分解成多个子任务，而且结合多个子任务的结果就能够得到最终的答案，那么它就适合使用FORK/JOIN模式来实现。[JAVA并行框架学习之ForkJoin](https://www.cnblogs.com/jiyuqi/p/4547082.html)

```java
public class ForkJoinTaskExample extends RecursiveTask<Integer> {

    public static final int threshold = 2;
    private int start;
    private int end;

    public ForkJoinTaskExample(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;

        //如果任务足够小就计算任务
        boolean canCompute = (end - start) <= threshold;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            // 如果任务大于阈值，就分裂成两个子任务计算
            int middle = (start + end) / 2;
            ForkJoinTaskExample leftTask = new ForkJoinTaskExample(start, middle);
            ForkJoinTaskExample rightTask = new ForkJoinTaskExample(middle + 1, end);

            // 执行子任务
            leftTask.fork();
            rightTask.fork(); 

            // 等待任务执行结束合并其结果
            int leftResult = leftTask.join();
            int rightResult = rightTask.join();

            // 合并子任务
            sum = leftResult + rightResult;
        }
        return sum;
    }

    public static void main(String[] args) {
        ForkJoinPool forkjoinPool = new ForkJoinPool();

        //生成一个计算任务，计算1+2+3+4
        ForkJoinTaskExample task = new ForkJoinTaskExample(1, 100);

        //执行一个任务
        Future<Integer> result = forkjoinPool.submit(task);

        try {
            log.info("result:{}", result.get());
        } catch (Exception e) {
            log.error("exception", e);
        }
    }
}
```

### 6.3 BlockingQueue

 [BlockingQueue深入解析－BlockingQueue看这一篇就够了](https://www.cnblogs.com/WangHaiMing/p/8798709.html)

#### 6.3.1 DelayQueue

DelayQueue的泛型参数需要实现Delayed接口，Delayed接口继承了Comparable接口，DelayQueue内部使用非线程安全的优先队列（PriorityQueue），并使用Leader/Followers模式，最小化不必要的等待时间。DelayQueue不允许包含null元素。

Leader/Followers模式：

1. 有若干个线程(一般组成线程池)用来处理大量的事件
2. 有一个线程作为领导者，等待事件的发生；其他的线程作为追随者，仅仅是睡眠。
3. 假如有事件需要处理，领导者会从追随者中指定一个新的领导者，自己去处理事件。
4. 唤醒的追随者作为新的领导者等待事件的发生。
5. 处理事件的线程处理完毕以后，就会成为追随者的一员，直到被唤醒成为领导者。
6. 假如需要处理的事件太多，而线程数量不够(能够动态创建线程处理另当别论)，则有的事件可能会得不到处理。

所有线程会有三种身份中的一种：leader和follower，以及一个干活中的状态：proccesser。它的基本原则就是，永远最多只有一个leader。而所有follower都在等待成为leader。线程池启动时会自动产生一个Leader负责等待网络IO事件，当有一个事件产生时，Leader线程首先通知一个Follower线程将其提拔为新的Leader，然后自己就去干活了，去处理这个网络事件，处理完毕后加入Follower线程等待队列，等待下次成为Leader。这种方法可以增强CPU高速缓存相似性，及消除动态内存分配和线程间的数据交换。

#### 6.3.2 ArrayBlockingQueue

#### 6.3.3 LinkedBlockingQueue

