# 1. 基础和应用篇

## 1.1 Redis基础数据结构

Redis有5种基础数据结构，分别为：string（字符串）、list（列表）、hash（字典）、set（集合）和zset（有序集合）

### 1.1.1 String

- Redis字符串是动态字符串，可修改。

- 实现方式类似Java的ArrayList，通过预分配冗余空间来减少频繁的内存分配。
- 字符串长度小于1MB时，扩容是加倍现有空间。长度超过1MB时，每次扩容增加1MB的空间，字符串最大长度512MB。

### 1.1.2 List

- Redis列表相当于Java里面的LinkedList（实际底层是一种“quicklist”的快速列表），插入和删除时间复杂度为O(1)，定位查找时间复杂度O(n)。
- 元素之间使用双向指针连接，支持向前向后遍历。
- 常用来做异步队列，将任务序列化成字符串放入Redis的列表，别的线程从这个列表中轮询数据进行处理。
- ziplist是使用一块连续内存来实现链表，减少前后指针也就减少了内存使用。quicklist是将若干个ziplist双向连接起来。

### 1.1.3 Hash

- 相当于Java里的HashMap，数组+链表二维结构。
- key只能是字符串。
- 采用渐进式rehash策略，在rehash时，保留新旧两个hash，查询时会同时查两个hash，后续会定时将数据从旧的hash里移动到新的hash，迁移完成就会用新的hash代替旧的，然后回收其空间。

### 1.1.4 Set

- Redis集合相当于Java语言里面的HashSet。
- 内部实现是一个value为NULL的字典，所有的键值对是无序的，唯一的。

### 1.1.5 zset

- zset，有序列表，类似于Java的SortedSet和HashMap的结合体。
- 它是一个set，可以保证内部元素(value)的唯一性。
- 可以对set里的value，根据其score进行排序。
- 内部实现是一种叫做“跳表”的数据结构。