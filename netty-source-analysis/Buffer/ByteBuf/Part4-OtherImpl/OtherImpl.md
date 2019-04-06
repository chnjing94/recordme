# Buffer 之 ByteBuf（四）其它子类

## 1.概述

在前面三篇文章，我们已经看了 ByteBuf 最最最核心的几个实现类。而剩余的整理起来，用途主要是：

- Swap
- Slice
- Duplicate
- ReadOnly
- Composite

## 2. Composite ByteBuf

因为 Composite ByteBuf 是比较重要的内容，胖友一定要自己去研究下。推荐阅读文章：

- 键走偏锋 [《对于 Netty ByteBuf 的零拷贝（Zero Copy）的理解》](https://my.oschina.net/LucasZhu/blog/1617222)
- [《Netty 学习笔记 —— 类CompositeByteBuf》](https://skyao.gitbooks.io/learning-netty/content/buffer/class_CompositeByteBuf.html)

